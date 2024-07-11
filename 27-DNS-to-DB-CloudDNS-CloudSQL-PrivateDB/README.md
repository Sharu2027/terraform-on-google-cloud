---
title: GCP Google Cloud Platform - DNS to DB with CloudSQL Private Database
description: Learn to implement DNS to DB with CloudSQL Private Database using Terraform on Google Cloud Platform
---

## Step-01: Introduction
- Implement DNS to DB usecase (Cloud Domains + Cloud DNS)
- We need a registered domain to implement this usecase
- This is a super big demo, lets do it step by step to succeed
- We are going to use Cloud SQL as Private DB (No public access)


## Step-02: COPY from 24-DNS-to-DB-CloudDNS-CloudSQL-PublicDB
- **COPY FROM:** 24-DNS-to-DB-CloudDNS-CloudSQL-PublicDB/p3-https-lb-clouddns
- **COPY TO:** 27-DNS-to-DB-CloudDNS-CloudSQL-PrivateDB/p3-https-lb-clouddns

## Step-03: c1-versions.tf
```hcl
# Terraform Settings Block
terraform {
  required_version = ">= 1.9"
  required_providers {
    google = {
      source = "hashicorp/google"
      version = ">= 5.37.0"
    }
  }
  backend "gcs" {
    bucket = "gcplearn9-tfstate"
    prefix = "myapp1/httpslb-clouddns-privatedb"
  }    
}

# Terraform Provider Block
provider "google" {
  project = var.gcp_project
  region = var.gcp_region1
}
```

## Step-04: c3-remote-state-datasource.tf
- Discuss about Remote State datasource which will be used to 
```hcl
# Terraform Remote State Datasource - Remote Backend GCP Cloud Storage Bucket
data "terraform_remote_state" "project1" {
  backend = "gcs"
  config = {
    bucket = "gcplearn9-tfstate"
    prefix = "cloudsql/privatedb"
  }
}

output "vpc_id" {
  description = "VPC ID"
  value = data.terraform_remote_state.project1.outputs.vpc_id
}

output "mysubnet_id" {
  description = "Subnet ID"
  value = data.terraform_remote_state.project1.outputs.mysubnet_id
}

output "cloudsql_privatedb" {
  value = data.terraform_remote_state.project1.outputs.cloudsql_db_private_ip
}
```

## Step-05: c4-firewallrules.tf
- Add **port 8080** for firewall rules
- Update VPC ID `data.terraform_remote_state.project1.outputs.vpc_id` using Remote State Datasource
```hcl
# Firewall Rule: SSH
resource "google_compute_firewall" "fw_ssh" {
  name = "${local.name}-fwrule-allow-ssh22"
  allow {
    ports    = ["22"]
    protocol = "tcp"
  }
  direction     = "INGRESS"
  network       = data.terraform_remote_state.project1.outputs.vpc_id
  priority      = 1000
  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["ssh-tag"]
}

# Firewall Rule: HTTP Port 80
resource "google_compute_firewall" "fw_http" {
  name = "${local.name}-fwrule-allow-http80"
  allow {
    ports    = ["80", "8080"]
    protocol = "tcp"
  }
  direction     = "INGRESS"
  network       = data.terraform_remote_state.project1.outputs.vpc_id
  priority      = 1000
  source_ranges = ["0.0.0.0/0"]
  target_tags   = ["webserver-tag"]
}

# Firewall rule: Allow Health checks
resource "google_compute_firewall" "fw_health_checks" {
  name    = "fwrule-allow-health-checks"
  network = data.terraform_remote_state.project1.outputs.vpc_id
  allow {
    protocol = "tcp"
    ports    = ["80", "8080"]
  }
  source_ranges = [
    "35.191.0.0/16",
    "130.211.0.0/22"
  ]
  target_tags = ["allow-health-checks"]
}
```

## Step-06: ums-install.tmpl
- Review **ums-install.tmpl**
```sh
#! /bin/bash
## SCRIPT-1: Deploy UserMgmt Application ###############
# Step-1: Update package list
sudo apt update

# Step-2: Install telnet (For Troubelshooting)
sudo apt install -y telnet

# Step-3: Install MySQL Client (For Troubelshooting)
sudo apt install -y default-mysql-client

# Step-4: Create directory for the application
mkdir -p /apps/usermgmt && cd /apps/usermgmt

# Step-5: Download Open JDK 11 and Install
wget https://aka.ms/download-jdk/microsoft-jdk-11.0.23-linux-x64.tar.gz -P /apps/usermgmt
sudo tar -xzf microsoft-jdk-11.0.23-linux-x64.tar.gz
sudo mv jdk-11.0.23+9 jdk11
sudo update-alternatives --install /usr/bin/java java /apps/usermgmt/jdk11/bin/java 1
sudo update-alternatives --install /usr/bin/javac javac /apps/usermgmt/jdk11/bin/javac 1

# Step-6: Download the application WAR file
wget https://github.com/stacksimplify/temp1/releases/download/1.0.0/usermgmt-webapp.war -P /apps/usermgmt 

# Step-7: Set environment variables for the database
export DB_HOSTNAME=${cloudsql_db_endpoint}
export DB_PORT=3306
export DB_NAME=webappdb
export DB_USERNAME=umsadmin
export DB_PASSWORD=dbpassword11

# Step-8: Run the application
java -jar /apps/usermgmt/usermgmt-webapp.war > /apps/usermgmt/ums-start.log &

# Step-9: Wait to ensure the webserver setup has completed
sleep 20

## SCRIPT-2: OPS Agent steps ###############
# Step-1: Install Ops Agent
curl -sSO https://dl.google.com/cloudagents/add-google-cloud-ops-agent-repo.sh
sudo bash add-google-cloud-ops-agent-repo.sh --also-install

# Step-2: Backup existing config.yaml file
sudo cp /etc/google-cloud-ops-agent/config.yaml /etc/google-cloud-ops-agent/config.yaml.bak

# Step-3: Write the Ops Agent configuration for Nginx logs
sudo tee /etc/google-cloud-ops-agent/config.yaml > /dev/null << EOF
logging:
  receivers:
    ums_log:
      type: files
      include_paths:
        - /apps/usermgmt/ums-start.log
  service:
    pipelines:
      default_pipeline:
        receivers: [ums_log]
EOF

# Step-4: Restart the Ops Agent to apply the new configuration
sudo service google-cloud-ops-agent restart
```

## Step-07: c6-01-app1-instance-template.tf
- Update **metadata_startup_script** with **ums-install.tmpl** and also pass Cloud SQL DB 
IP address
- Update `cloudsql_db_endpoint = data.terraform_remote_state.project1.outputs.cloudsql_db_private_ip`
- Update `subnetwork = data.terraform_remote_state.project1.outputs.mysubnet_id`
```hcl
# Google Compute Engine: Regional Instance Template
resource "google_compute_region_instance_template" "myapp1" {
  name        = "${local.name}-myapp1-template"
  description = "This template is used to create MyApp1 server instances."
  #tags        = [tolist(google_compute_firewall.fw_ssh.target_tags)[0], tolist(google_compute_firewall.fw_http.target_tags)[0]]
  tags        = [tolist(google_compute_firewall.fw_ssh.target_tags)[0], tolist(google_compute_firewall.fw_http.target_tags)[0], tolist(google_compute_firewall.fw_health_checks.target_tags)[0]]
  instance_description = "MyApp1 VM Instances"
  machine_type         = var.machine_type
  scheduling {
    automatic_restart   = true
    on_host_maintenance = "MIGRATE"
  }
  # Create a new boot disk from an image
  disk {
    #source_image      = "debian-cloud/debian-12"
    source_image      = data.google_compute_image.my_image.self_link
    auto_delete       = true
    boot              = true
  }
  # Network Info
  network_interface {
    subnetwork = data.terraform_remote_state.project1.outputs.mysubnet_id
    /*access_config {
      # Include this section to give the VM an external IP address
    } */ 
  }
  # Install Webserver
  metadata_startup_script = templatefile("ums-install.tmpl",{cloudsql_db_endpoint = data.terraform_remote_state.project1.outputs.cloudsql_db_private_ip})    
  labels = {
    environment = local.environment
  }
  metadata = {
    environment = local.environment
  }
  service_account {
    # Google recommends custom service accounts that have cloud-platform scope and permissions granted via IAM Roles.
    email  = google_service_account.myapp1.email
    scopes = ["cloud-platform"]
  }   
}
```
## Step-08: c6-02-app1-mig-healthcheck.tf
- Update health check `intervals, request_path and port`
```hcl
# Resource: Regional Health Check
resource "google_compute_region_health_check" "myapp1" {
  name                = "${local.name}-myapp1"
  check_interval_sec  = 20
  timeout_sec         = 10
  healthy_threshold   = 3
  unhealthy_threshold = 3
  http_health_check {
    request_path = "/login"
    port         = 8080
  }
}
```
## Step-09: c6-03-app1-mig.tf
- Update `named_port` to `appserver` with value as `8080`
```hcl
# Resource: Managed Instance Group
resource "google_compute_region_instance_group_manager" "myapp1" {
  depends_on = [ google_compute_router_nat.cloud_nat ]
  name                       = "${local.name}-myapp1-mig"
  base_instance_name         = "${local.name}-myapp1"
  region                     = var.gcp_region1
  distribution_policy_zones  = data.google_compute_zones.available.names
  # Instance Template
  version {
    # V1 Version
    instance_template = google_compute_region_instance_template.myapp1.id
  }
  # Named Port
  named_port {
    name = "appserver"
    port = 8080
  }
  # Autosclaing
  auto_healing_policies {
    health_check      = google_compute_region_health_check.myapp1.id
    initial_delay_sec = 300
  }
 
  # Update Policy
  update_policy {
    type                           = "PROACTIVE"
    instance_redistribution_type   = "PROACTIVE"
    minimal_action                 = "REPLACE"
    most_disruptive_allowed_action = "REPLACE"
    max_surge_fixed                = length(data.google_compute_zones.available.names)
    max_unavailable_fixed          = length(data.google_compute_zones.available.names)
    replacement_method             = "SUBSTITUTE"
    # min_ready_sec                  = 50 #   (BETA Parameter)
  } 
}
```

## Step-10: c7-01-loadbalancer.tf
- Update health check `intervals, request_path and port`
```hcl
# Resource: Regional Health Check
resource "google_compute_region_health_check" "mylb" {
  name                = "${local.name}-mylb-myapp1-health-check"
  check_interval_sec  = 20
  timeout_sec         = 10
  healthy_threshold   = 3
  unhealthy_threshold = 3
  http_health_check {
    request_path = "/login"
    port         = 8080
  }
}
```

## Step-11: c7-01-loadbalancer.tf
- Update port_name to `appserver` and also add `session_affinity      = "GENERATED_COOKIE"`
- Session affinity is needed to ensure Login session for application stick to that respective backend (VM Instance)
```hcl
# Resource: Regional Backend Service
resource "google_compute_region_backend_service" "mylb" {
  name                  = "${local.name}-myapp1-backend-service"
  protocol              = "HTTP"
  load_balancing_scheme = "EXTERNAL_MANAGED"
  health_checks         = [google_compute_region_health_check.mylb.self_link]
  port_name             = "appserver"
  session_affinity      = "GENERATED_COOKIE"
  backend {
    group = google_compute_region_instance_group_manager.myapp1.instance_group
    capacity_scaler = 1.0
    balancing_mode = "UTILIZATION"
  }
}
```

## Step-12: c7-01-loadbalancer.tf
- Update `  network = data.terraform_remote_state.project1.outputs.vpc_id`
- Comment `#depends_on = [ google_compute_subnetwork.regional_proxy_subnet ]`
```hcl
# Resource: Regional Forwarding Rule
resource "google_compute_forwarding_rule" "mylb" {
  name        = "${local.name}-mylb-forwarding-rule"
  target      = google_compute_region_target_https_proxy.mylb.self_link
  port_range  = "443"
  ip_protocol = "TCP"
  ip_address = google_compute_address.mylb.address
  load_balancing_scheme = "EXTERNAL_MANAGED" # Creates new GCP LB (not classic)
  network = data.terraform_remote_state.project1.outputs.vpc_id
  # During the destroy process, we need to ensure LB is deleted first, before deleting VPC proxy-only subnet
  #depends_on = [ google_compute_subnetwork.regional_proxy_subnet ]
}
```
## Step-13: c7-02-loadbalancer-http-to-https.tf
- Update `  network = data.terraform_remote_state.project1.outputs.vpc_id`
- Comment `#depends_on = [ google_compute_subnetwork.regional_proxy_subnet ]`
```hcl
# Resource: Regional Forwarding Rule for HTTP to HTTPS redirection
resource "google_compute_forwarding_rule" "http" {
  name        = "${local.name}-myapp1-http-to-https-forwarding-rule"
  target      = google_compute_region_target_http_proxy.http.self_link
  port_range  = "80"
  ip_protocol = "TCP"
  ip_address = google_compute_address.mylb.address
  load_balancing_scheme = "EXTERNAL_MANAGED" # Creates new GCP LB (not classic)
  network = data.terraform_remote_state.project1.outputs.vpc_id
  # During the destroy process, we need to ensure LB is deleted first, before deleting VPC proxy-only subnet
  #depends_on = [ google_compute_subnetwork.regional_proxy_subnet ]
}
```

## Step-14: c8-Cloud-NAT-Cloud-Router.tf
- Update `  network = data.terraform_remote_state.project1.outputs.vpc_id`
```hcl
# Resource: Cloud Router
resource "google_compute_router" "cloud_router" {
  name    = "${local.name}-${var.gcp_region1}-cloud-router"
  network = data.terraform_remote_state.project1.outputs.vpc_id
  region  = var.gcp_region1
}

```

## Step-15: c11-monitoring-uptime-checks.tf
- Update `path = "/login"` and `content = "Username"`
```hcl
# Resource: Uptime check
resource "google_monitoring_uptime_check_config" "https" {
  display_name = "${local.name}-myapp1-lb-https-uptime-check"
  timeout = "60s"

  http_check {
    path = "/login"
    port = "443"
    use_ssl = true
    #validate_ssl = true # We are using self-signed, so don't use this
  }
  monitored_resource {
    type = "uptime_url"
    labels = {
      project_id = var.gcp_project_id # Provide Project ID here, my project name and ID is same used var.gcp_project
      host = google_compute_address.mylb.address
    }
  }
  content_matchers {
    content = "Username"
  }
}
```

## Step-16: Execute Terraform Commands
```t
# Terraform Initialize
terraform init

# Terraform Validate
terraform validate

# Terraform Plan
terraform plan

# Terraform Apply
terraform apply
```

# Step-17: Verify Resources
1. Verify Load Balancer
2. Verify Certificate Manager SSL Certificate
3. Verify MIG
4. Verify Service Account
5. Verify Cloud Logging Logs
6. Verify Cloud Monitoring - Uptime checks
7. Verify Cloud Monitoring - Alert Policy
8. Connect to VM Instance and verify logs
```t
# Connect to VM instance and verify logs
1. SSH to vm
2. cd /apps/usermgmt
3. tail -100f /apps/usermgmt/ums-start.log

# Access HTTPS URL
http://<DOMAIN-NAME>
http://app1.kalyanreddydaida.com
Username: admin101
Password: password101

## CREATE NEW USER IN APPLICATION
Username: admin103
Password: password103
first name: fname103
last name: lname103
email: admin103@stacksimplify.com
ssn: ssn103
Click on **CREATE USER**

## Login with new user
https://app1.kalyanreddydaida.com
Username: admin103
Password: password103

## Verify the new user in Cloud SQL Database
- Goto Cloud SQL -> hr-dev-mysql -> Cloud SQL Studio
- **Database:** webappdb
- **User:** umsadmin
- **Password:** dbpassword11
# SQL Query
select * from user;

# Cloud Logging
resource.type="gce_instance"  log_id("ums_log") 

# Cloud Monitoring
Goto Cloud Monitoring -> Detect -> Uptime checks

# Alert Policy and Incidents
Goto Cloud Monitoring -> Detect -> Alerting -> hr-dev-myapp1-lb-https-uptime-check
1. Verify Incidents (if any)
```

## Step-18: Clean-Up
```t
# Terraform Destroy
terraform destroy -auto-approve
rm -rf .terraform*

```

## Step-19: Delete Cloud SQL Private DB
```t
# Change Directory
cd p1-cloudsql-privatedb

# Terraform Initialize
terraform init

# Terraform State list
terraform state list

# Terraform Destory
terraform destroy -auto-approve
```
