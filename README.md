Below is a GitHub-ready `README.md`. I’ve deliberately marked **every lab-run-specific value** with `CHANGE THIS` so the documentation remains useful when Qwiklabs changes project IDs, bucket names, regions, or other lab parameters.

 # Create a Landing Zone and Harden It with Security Command Center

 > **Qwiklabs / Google Cloud Challenge Lab**
>
>  Build a secure Google Cloud landing zone using Terraform, Shared VPC, Private Google Access, firewall controls, Cloud NAT, a hardened VM, an internal static IP address, and Security Command Center.

---

 ## Table of Contents

 - Overview
- Architecture
- Prerequisites
- Lab-Specific Values
- Important Rules
- Task 1 — Configure Shared VPC
- Task 2 — Secure the Network
- Task 3 — Deploy the Hardened VM
- Task 3 — Reserve Internal Static IP
- Task 4 — Enable Security Command Center
- Verification Checklist
- Troubleshooting
- Reusable Lab Workflow
- Cleanup
- GitHub Usage

---

 # Overview

 This challenge lab creates a secure Google Cloud landing zone for a fictional financial organization.

 The environment uses:

 - Terraform
- Google Cloud Foundation Fabric / Google Terraform modules
- Shared VPC
- Private Google Access
- Firewall rules
- Identity-Aware Proxy (IAP)
- Cloud NAT
- Private VM instances
- Internal static IP addresses
- Organization policies
- Security Command Center

 The overall goal is to build an environment that is:

 - Centrally managed
- Secure by default
- Reproducible using Infrastructure as Code
- Accessible without exposing VMs directly to the internet
- Ready for application workloads

---

 # Architecture

 The resulting architecture is approximately:

```
                         Google Cloud Organization
                                  |
                                  |
                       +----------------------+
                       |      Host Project    |
                       |                      |
                       |    Shared VPC        |
                       |    shared-network    |
                       |          |           |
                       |     +----+----+      |
                       |     |         |      |
                       |   subnet-01  subnet-02
                       |     |         |      |
                       |     +----+----+      |
                       |          |           |
                       |     Cloud NAT        |
                       |          |           |
                       +----------|-----------+
                                  |
                         Shared VPC attachment
                                  |
                    +-------------+-------------+
                    |                           |
             Service Project              Other Projects
                    |
                    |
             Private VM Instance
          cymbal-service-instance
                    |
              Internal IP only
                    |
              No External IP
```

 The VM uses the Shared VPC subnet from the Host Project while the VM itself belongs to the Service Project.

---

 # Prerequisites

 You need:

 - A Qwiklabs / Google Cloud Skills Boost lab account
- A browser
- Google Cloud Console access
- Cloud Shell
- Terraform
- `gcloud`
- `gsutil`

 The lab provides temporary credentials.

 ## Recommended browser setup

 Use an **Incognito / Private browser window**.

 Do not use your personal Google Cloud account for the challenge lab.

---

 # Lab-Specific Values

 The following values are examples from one lab run.

 **These values can change every time the lab is started.**

 Replace them with the values shown in the current lab instructions.

 ## Configuration variables

```
# ============================================================
# CHANGE THESE VALUES FOR EVERY NEW LAB RUN
# ============================================================

export HOST_PROJECT_ID="CHANGE-THIS-HOST-PROJECT-ID"

export SERVICE_PROJECT_ID="CHANGE-THIS-SERVICE-PROJECT-ID"

export REGION="CHANGE-THIS-REGION"

# Usually the region is also used to determine the VM zone.
# Example:
# export REGION="us-east1"

export NETWORK_NAME="shared-network"

export SUBNET_01="shared-network-subnet-01"

export SUBNET_02="shared-network-subnet-02"
```

 For the lab run documented while creating this README, the values were:

```
HOST_PROJECT_ID="qwiklabs-gcp-02-dd44f8641324"
SERVICE_PROJECT_ID="qwiklabs-gcp-02-5ee4cd1f0d7b"
REGION="us-east1"
NETWORK_NAME="shared-network"
SUBNET_01="shared-network-subnet-01"
SUBNET_02="shared-network-subnet-02"
```

 ### Important

 Do **not** copy the example project IDs into a future lab.

 Always copy the current values from the Qwiklabs Lab Details panel.

---

 # Important Rules

 ## 1\. Never commit temporary credentials

 Do not put these into Git:

```
Qwiklabs username
Qwiklabs password
Temporary access tokens
Service account keys
Private keys
```

 Add sensitive files to `.gitignore`.

 Example:

```
# Terraform
.terraform/
*.tfstate
*.tfstate.*
crash.log
*.tfvars
*.tfvars.json

# Credentials
*.json
*.pem
*.key
credentials/
secrets/

# OS
.DS_Store
```

 ## 2\. Do not assume project IDs

 Project IDs are normally different for every lab session.

 Always use:

```
gcloud config get-value project
```

 and:

```
gcloud projects list
```

 when necessary.

 ## 3\. Do not assume the region

 The region may change between versions of the challenge lab.

 Always use the region specified by the current lab.

---

 # Task 1 — Configure Shared VPC

 ## Objective

 Create a Shared VPC architecture consisting of:

 - Host Project
- Service Project
- VPC network
- Two subnets
- Private Google Access
- IAM permissions allowing the Service Project to use the Shared VPC

---

 ## Step 1 — Configure the Host Project

 Set the current project:

```
gcloud config set project "$HOST_PROJECT_ID"
```

 Verify:

```
gcloud config get-value project
```

 Expected:

```
CHANGE-THIS-HOST-PROJECT-ID
```

---

 # Step 2 — Create the Shared VPC Terraform Directory

 Create the working directory:

```
mkdir -p ~/shared-vpc
```

 Copy the Terraform configuration supplied by the lab.

 > **CHANGE THIS BUCKET NAME when the lab provides a different bucket.**

```
gsutil -m cp -r \
  gs://CHANGE-THIS-HOST-LAB-CONFIG-BUCKET/* \
  ~/shared-vpc/
```

 For the documented lab:

```
gsutil -m cp -r \
  gs://qwiklabs-gcp-02-dd44f8641324-labconfig-bucket/* \
  ~/shared-vpc/
```

 Enter the directory:

```
cd ~/shared-vpc
```

---

 # Step 3 — Configure Terraform Variables

 Open:

```
nano variables.tf
```

 Find:

```
variable "gcp_project_id" {
```

 Set the default to the current Host Project:

```
default = "CHANGE-THIS-HOST-PROJECT-ID"
```

 Example:

```
default = "qwiklabs-gcp-02-dd44f8641324"
```

 Find:

```
variable "gcp_region" {
```

 Set:

```
default = "CHANGE-THIS-REGION"
```

 Example:

```
default = "us-east1"
```

 The network name should normally remain:

```
default = "shared-network"
```

 unless the lab specifies another value.

---

 # Step 4 — Install Terraform

 If Terraform is not installed:

```
wget -O - https://apt.releases.hashicorp.com/gpg \
  | sudo gpg --dearmor \
  -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) \
signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
https://apt.releases.hashicorp.com \
$(grep -oP '(?<=UBUNTU_CODENAME=).*' /etc/os-release || lsb_release -cs) \
main" \
| sudo tee /etc/apt/sources.list.d/hashicorp.list

sudo apt update

sudo apt install terraform
```

 Verify:

```
terraform version
```

---

 # Step 5 — Initialize Terraform

```
cd ~/shared-vpc

terraform init
```

 Validate:

```
terraform validate
```

 A successful validation should show:

```
Success! The configuration is valid.
```

---

 # Step 6 — Deploy the Shared VPC

 Review the Terraform plan:

```
terraform plan
```

 If the plan is correct:

```
terraform apply
```

 or:

```
terraform apply -auto-approve
```

---

 # Step 7 — Verify the Network

 List VPC networks:

```
gcloud compute networks list \
  --project="$HOST_PROJECT_ID"
```

 The network should include:

```
shared-network
```

 List the subnets:

```
gcloud compute networks subnets list \
  --project="$HOST_PROJECT_ID"
```

 The expected subnets are:

```
shared-network-subnet-01
shared-network-subnet-02
```

 The lab specifies:

```
shared-network-subnet-01
172.16.10.0/24

shared-network-subnet-02
172.16.20.0/24
```

---

 # Step 8 — Enable Shared VPC

 Enable Shared VPC on the Host Project:

```
gcloud compute shared-vpc enable "$HOST_PROJECT_ID"
```

 Verify:

```
gcloud compute shared-vpc get-host-project "$HOST_PROJECT_ID"
```

---

 # Step 9 — Attach the Service Project

 **Important:** The project passed to `associated-projects add` must be the **Service Project**, not the Host Project.

 Correct:

```
gcloud compute shared-vpc associated-projects add \
  "$SERVICE_PROJECT_ID" \
  --host-project="$HOST_PROJECT_ID"
```

 Do **not** do this:

```
# WRONG
gcloud compute shared-vpc associated-projects add \
  "$HOST_PROJECT_ID" \
  --host-project="$HOST_PROJECT_ID"
```

 That produces an error similar to:

```
The requested shared VPC resource must be different than the shared VPC host.
```

 Verify the attachment:

```
gcloud compute shared-vpc get-host-project \
  "$SERVICE_PROJECT_ID"
```

 The output should identify the Host Project.

---

 # Step 10 — Grant Network User Permissions

 Get the Service Project number:

```
SERVICE_PROJECT_NUMBER=$(gcloud projects describe "$SERVICE_PROJECT_ID" \
  --format='value(projectNumber)')
```

 The Compute Engine service agent is:

```
service-SERVICE_PROJECT_NUMBER@compute-system.iam.gserviceaccount.com
```

 Grant project-level Network User:

```
gcloud projects add-iam-policy-binding "$HOST_PROJECT_ID" \
  --member="serviceAccount:service-${SERVICE_PROJECT_NUMBER}@compute-system.iam.gserviceaccount.com" \
  --role="roles/compute.networkUser"
```

---

 # Step 11 — Grant Subnet-Level Network User

 Grant access to subnet 01:

```
gcloud compute networks subnets add-iam-policy-binding \
  "$SUBNET_01" \
  --region="$REGION" \
  --project="$HOST_PROJECT_ID" \
  --member="serviceAccount:service-${SERVICE_PROJECT_NUMBER}@compute-system.iam.gserviceaccount.com" \
  --role="roles/compute.networkUser"
```

 Grant access to subnet 02:

```
gcloud compute networks subnets add-iam-policy-binding \
  "$SUBNET_02" \
  --region="$REGION" \
  --project="$HOST_PROJECT_ID" \
  --member="serviceAccount:service-${SERVICE_PROJECT_NUMBER}@compute-system.iam.gserviceaccount.com" \
  --role="roles/compute.networkUser"
```

---

 # Task 1 Verification

 Check:

```
gcloud compute shared-vpc get-host-project \
  "$SERVICE_PROJECT_ID"
```

 Then verify the subnets:

```
gcloud compute networks subnets list \
  --project="$HOST_PROJECT_ID" \
  --regions="$REGION"
```

 Finally click:

 **Check my progress → Configure the Shared VPC and service project**

---

 # Task 2 — Secure the Network

 Task 2 consists of:

 1. Private Google Access
2. Firewall rules
3. Organization policy controls
4. Cloud NAT

---

 # Step 1 — Enable Private Google Access

 Enable Private Google Access on subnet 01:

```
gcloud compute networks subnets update "$SUBNET_01" \
  --project="$HOST_PROJECT_ID" \
  --region="$REGION" \
  --enable-private-ip-google-access
```

 Enable it on subnet 02:

```
gcloud compute networks subnets update "$SUBNET_02" \
  --project="$HOST_PROJECT_ID" \
  --region="$REGION" \
  --enable-private-ip-google-access
```

 Verify:

```
gcloud compute networks subnets describe "$SUBNET_01" \
  --project="$HOST_PROJECT_ID" \
  --region="$REGION" \
  --format="get(privateIpGoogleAccess)"
```

 Expected:

```
True
```

 Repeat for subnet 02:

```
gcloud compute networks subnets describe "$SUBNET_02" \
  --project="$HOST_PROJECT_ID" \
  --region="$REGION" \
  --format="get(privateIpGoogleAccess)"
```

---

 # Step 2 — Create the IAP SSH Firewall Rule

 IAP TCP forwarding uses:

```
35.235.240.0/20
```

 Create the firewall rule:

```
gcloud compute firewall-rules create cymbal-firewall-rule \
  --project="$HOST_PROJECT_ID" \
  --network="$NETWORK_NAME" \
  --direction=INGRESS \
  --action=ALLOW \
  --rules=tcp:22 \
  --source-ranges=35.235.240.0/20
```

 Verify:

```
gcloud compute firewall-rules describe cymbal-firewall-rule \
  --project="$HOST_PROJECT_ID"
```

 The rule should allow:

```
tcp:22
```

 from:

```
35.235.240.0/20
```

---

 # Step 3 — Cloud NAT

 A private VM has no external IP.

 Therefore, it cannot directly use the public internet.

 Cloud NAT provides outbound internet access while keeping the VM private.

 Create the Cloud Router:

```
gcloud compute routers create landing-zone-router \
  --project="$HOST_PROJECT_ID" \
  --network="$NETWORK_NAME" \
  --region="$REGION"
```

 Create Cloud NAT:

```
gcloud compute routers nats create cymbal-gateway \
  --project="$HOST_PROJECT_ID" \
  --router=landing-zone-router \
  --region="$REGION" \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips
```

 ### Important

 When using Public NAT, Google Cloud requires either:

```
--nat-external-ip-pool
```

 or:

```
--auto-allocate-nat-external-ips
```

 Without one of these, you may receive:

```
Either --nat-external-ip-pool or --auto-allocate-nat-external-ips must be specified for Public NAT.
```

---

 # Task 2 Verification

 Check Private Google Access:

```
gcloud compute networks subnets describe "$SUBNET_01" \
  --project="$HOST_PROJECT_ID" \
  --region="$REGION" \
  --format="get(privateIpGoogleAccess)"
```

 Check firewall:

```
gcloud compute firewall-rules describe cymbal-firewall-rule \
  --project="$HOST_PROJECT_ID"
```

 Check Cloud Router:

```
gcloud compute routers describe landing-zone-router \
  --project="$HOST_PROJECT_ID" \
  --region="$REGION"
```

 Check NAT:

```
gcloud compute routers nats describe cymbal-gateway \
  --router=landing-zone-router \
  --project="$HOST_PROJECT_ID" \
  --region="$REGION"
```

 Then click:

 **Check my progress → Secure the network and enforce policies**

 and:

 **Check my progress → Enable Outbound Internet Access for Private VM**

---

 # Task 3 — Deploy the Hardened VM

 The VM must:

 - Use the Service Project
- Use the Shared VPC
- Use subnet 01
- Have no external IP
- Have serial port access disabled
- Use the specified hostname
- Use the specified region

---

 # Step 1 — Clone the Terraform VM Module

 Switch to the Service Project:

```
gcloud config set project "$SERVICE_PROJECT_ID"
```

 Clone the repository:

```
cd ~
git clone https://github.com/terraform-google-modules/terraform-google-vm.git
```

 Enter the simple compute example:

```
cd ~/terraform-google-vm/examples/compute_instance/simple
```

---

 # Step 2 — Configure VM Variables

 Open:

```
nano variables.tf
```

 Set the region:

```
default = "CHANGE-THIS-REGION"
```

 Example:

```
default = "us-east1"
```

 Do not hard-code a zone unless the lab requires one.

 If the module automatically discovers zones, leaving the zone as `null` can be useful.

---

 # Step 3 — Configure `main.tf`

 The VM needs to use the Shared VPC subnet from the Host Project.

 A working structure is:

```
module "instance_template" {
  source  = "terraform-google-modules/vm/google//modules/instance_template"
  version = "~> 13.0"

  region             = var.region
  project_id         = var.project_id
  subnetwork         = var.subnetwork
  subnetwork_project = "CHANGE-THIS-HOST-PROJECT-ID"
  service_account    = var.service_account

  metadata = {
    serial-port-enable = "FALSE"
  }
}

module "compute_instance" {
  source  = "terraform-google-modules/vm/google//modules/compute_instance"
  version = "~> 14.0"

  project_id          = var.project_id
  region              = var.region
  zone                = var.zone
  subnetwork          = var.subnetwork
  subnetwork_project  = "CHANGE-THIS-HOST-PROJECT-ID"
  num_instances       = var.num_instances
  hostname            = "cymbal-service-instance"
  instance_template   = module.instance_template.self_link
  deletion_protection = false
}
```

 ### Important security setting

 Do **not** add:

```
access_config = [{
  nat_ip       = var.nat_ip
  network_tier = var.network_tier
}]
```

 An `access_config` block creates an external/public IP configuration.

 For this challenge, the VM must not have an external IP.

---

 # Step 4 — Configure the Shared VPC Subnet

 The subnet must use the Host Project.

 The required value is:

```
projects/HOST_PROJECT_ID/regions/REGION/subnetworks/SUBNET_NAME
```

 Example:

```
projects/qwiklabs-gcp-02-dd44f8641324/regions/us-east1/subnetworks/shared-network-subnet-01
```

 When creating a new lab, replace:

```
qwiklabs-gcp-02-dd44f8641324
```

 with the current Host Project ID.

---

 # Step 5 — Initialize Terraform

```
terraform init
```

 Validate:

```
terraform validate
```

 Expected:

```
Success! The configuration is valid.
```

---

 # Step 6 — Plan the VM

 Use:

```
terraform plan \
  -var='num_instances=1' \
  -var='project_id=CHANGE-THIS-SERVICE-PROJECT-ID' \
  -var='subnetwork=projects/CHANGE-THIS-HOST-PROJECT-ID/regions/CHANGE-THIS-REGION/subnetworks/shared-network-subnet-01'
```

 Example:

```
terraform plan \
  -var='num_instances=1' \
  -var='project_id=qwiklabs-gcp-02-5ee4cd1f0d7b' \
  -var='subnetwork=projects/qwiklabs-gcp-02-dd44f8641324/regions/us-east1/subnetworks/shared-network-subnet-01'
```

---

 # Important VM Plan Checks

 Before applying, verify the plan.

 ## The project must be the Service Project

```
project = SERVICE_PROJECT_ID
```

 ## The subnet project must be the Host Project

```
subnetwork_project = HOST_PROJECT_ID
```

 ## There must be no `access_config`

 Under:

```
network_interface
```

 you should **not** see an:

```
access_config
```

 block.

 ## Serial port must be disabled

 The instance template should contain:

```
serial-port-enable = FALSE
```

 ## Hostname

 The VM should be named based on:

```
cymbal-service-instance
```

---

 # Step 7 — Apply the VM

 After verifying the plan:

```
terraform apply -auto-approve \
  -var='num_instances=1' \
  -var='project_id=CHANGE-THIS-SERVICE-PROJECT-ID' \
  -var='subnetwork=projects/CHANGE-THIS-HOST-PROJECT-ID/regions/CHANGE-THIS-REGION/subnetworks/shared-network-subnet-01'
```

 Example:

```
terraform apply -auto-approve \
  -var='num_instances=1' \
  -var='project_id=qwiklabs-gcp-02-5ee4cd1f0d7b' \
  -var='subnetwork=projects/qwiklabs-gcp-02-dd44f8641324/regions/us-east1/subnetworks/shared-network-subnet-01'
```

---

 # Step 8 — Verify the VM

```
gcloud compute instances list \
  --project="$SERVICE_PROJECT_ID"
```

 Describe the instance:

```
gcloud compute instances describe cymbal-service-instance-001 \
  --project="$SERVICE_PROJECT_ID" \
  --zone="CHANGE-THIS-ZONE"
```

 If Terraform generated a different instance suffix, use:

```
gcloud compute instances list \
  --project="$SERVICE_PROJECT_ID"
```

 to obtain the exact name.

---

 # Verify No External IP

 Run:

```
gcloud compute instances list \
  --project="$SERVICE_PROJECT_ID" \
  --format="table(name,zone,networkInterfaces[0].networkIP,networkInterfaces[0].accessConfigs[0].natIP)"
```

 The external/NAT IP should be empty.

---

 # Verify Serial Port Is Disabled

 Run:

```
gcloud compute instances describe cymbal-service-instance-001 \
  --project="$SERVICE_PROJECT_ID" \
  --zone="CHANGE-THIS-ZONE" \
  --format="get(metadata.items)"
```

 Look for:

```
serial-port-enable
FALSE
```

---

 # Task 3 — Reserve Internal Static IP

 The lab provides another Cloud Storage bucket containing Terraform configuration.

---

 # Step 1 — Create the Address Directory

```
mkdir -p ~/compute-address
```

 Copy the lab configuration:

```
gsutil -m cp -r \
  gs://CHANGE-THIS-SERVICE-LAB-CONFIG-BUCKET/* \
  ~/compute-address/
```

 Example:

```
gsutil -m cp -r \
  gs://qwiklabs-gcp-02-5ee4cd1f0d7b-labconfig-service-project/* \
  ~/compute-address/
```

 Enter the directory:

```
cd ~/compute-address
```

---

 # Step 2 — Configure the Service Project

 Open:

```
nano variables.tf
```

 Set:

```
variable "gcp_project_id" {
  type        = string
  description = "The GCP Project ID to apply this config to."

  default = "CHANGE-THIS-SERVICE-PROJECT-ID"
}
```

 Example:

```
default = "qwiklabs-gcp-02-5ee4cd1f0d7b"
```

---

 # Step 3 — Set the Correct Region

 This is important.

 The reserved internal IP must be in the **same region as the subnet**.

 For example:

```
default = "us-east1"
```

 Do not accidentally leave:

```
default = "us-central1"
```

 if the subnet is in:

```
us-east1
```

---

 # Step 4 — Configure the Subnetwork

 The subnet variable must contain the complete Shared VPC self-link:

```
default = "projects/CHANGE-THIS-HOST-PROJECT-ID/regions/CHANGE-THIS-REGION/subnetworks/shared-network-subnet-01"
```

 Example:

```
default = "projects/qwiklabs-gcp-02-dd44f8641324/regions/us-east1/subnetworks/shared-network-subnet-01"
```

---

 # Step 5 — Initialize Terraform

```
terraform init
```

 Validate:

```
terraform validate
```

---

 # Step 6 — Plan

```
terraform plan
```

 The plan should create:

```
google_compute_address.example_address
```

 with:

```
name = "internal-web-server-ip"
```

 and:

```
address_type = "INTERNAL"
```

 The region must match the subnet.

---

 # Step 7 — Apply

```
terraform apply -auto-approve
```

---

 # Step 8 — Verify the Internal IP

```
gcloud compute addresses list \
  --project="$SERVICE_PROJECT_ID" \
  --regions="$REGION"
```

 Expected:

```
internal-web-server-ip
```

 The address should be an internal RFC1918 address.

---

 # Common Error — Subnetwork Must Be in the Same Region

 If you see:

```
Invalid value for field 'resource.subnetwork':
Subnetwork must be in the same region.
```

 Check:

```
gcloud compute networks subnets describe "$SUBNET_01" \
  --project="$HOST_PROJECT_ID" \
  --region="$REGION"
```

 Then make sure the address Terraform configuration uses the **same region**.

 For example:

```
Address region:
us-east1

Subnet region:
us-east1
```

 Not:

```
Address region:
us-central1

Subnet region:
us-east1
```

---

 # Task 3 Verification

 Verify:

```
gcloud compute instances list \
  --project="$SERVICE_PROJECT_ID"
```

 Verify the internal address:

```
gcloud compute addresses list \
  --project="$SERVICE_PROJECT_ID" \
  --regions="$REGION"
```

 Then click:

 **Check my progress → Deploy a Secure VM Instance and Reserve an Internal Static IP**

---

 # Task 4 — Enable Security Command Center

 The lab requires Security Command Center to be enabled according to the scope and tier specified in the current lab instructions.

 > **Important:** Google Cloud security products and activation workflows can change. Always follow the current lab instructions for the exact SCC activation method.

 Before making changes, identify the organization:

```
gcloud organizations list
```

 Get the organization ID:

```
gcloud projects get-ancestors "$HOST_PROJECT_ID"
```

 The output should contain the organization resource.

---

 ## Check Security Command Center Access

```
gcloud services list \
  --enabled \
  --project="$HOST_PROJECT_ID"
```

 Look for the Security Command Center API if required by the current lab:

```
securitycenter.googleapis.com
```

 If the current lab explicitly requires enabling the API:

```
gcloud services enable securitycenter.googleapis.com \
  --project="$HOST_PROJECT_ID"
```

 Follow the current challenge lab's instructions for activating the **Standard Tier** at the required scope.

---

 # Verification Checklist

 Before submitting the challenge lab, verify the following.

 ## Shared VPC

```
[ ] Host Project configured
[ ] Shared VPC enabled
[ ] shared-network exists
[ ] shared-network-subnet-01 exists
[ ] shared-network-subnet-02 exists
[ ] Service Project attached
[ ] Service Project has Network User permissions
```

 ## Private Google Access

```
[ ] subnet-01 has Private Google Access
[ ] subnet-02 has Private Google Access
```

 ## Firewall

```
[ ] cymbal-firewall-rule exists
[ ] TCP 22 is allowed
[ ] Source range is 35.235.240.0/20
```

 ## Cloud NAT

```
[ ] landing-zone-router exists
[ ] cymbal-gateway exists
[ ] NAT covers required subnet ranges
[ ] NAT has external IP allocation configured
```

 ## VM

```
[ ] VM exists
[ ] VM uses Service Project
[ ] VM uses Shared VPC
[ ] VM uses subnet-01
[ ] VM has no external IP
[ ] serial-port-enable = FALSE
[ ] hostname is cymbal-service-instance
```

 ## Internal Static IP

```
[ ] internal-web-server-ip exists
[ ] Address type is INTERNAL
[ ] Address region matches subnet region
[ ] Address uses Shared VPC subnet
```

 ## Security Command Center

```
[ ] SCC enabled according to current lab requirements
[ ] Required tier configured
[ ] Required project/organization scope configured
```

---

 # Troubleshooting

 ## Error: Shared VPC resource must be different than host

 ### Error

```
The requested shared VPC resource must be different than the shared VPC host.
```

 ### Cause

 The Host Project was accidentally supplied as the associated Service Project.

 ### Wrong

```
gcloud compute shared-vpc associated-projects add \
  "$HOST_PROJECT_ID" \
  --host-project="$HOST_PROJECT_ID"
```

 ### Correct

```
gcloud compute shared-vpc associated-projects add \
  "$SERVICE_PROJECT_ID" \
  --host-project="$HOST_PROJECT_ID"
```

---

 # Error: Terraform Cannot Find Available Zones

 ### Error

```
Invalid index
```

 with:

```
data.google_compute_zones.available.names
```

 being an empty list.

 ### Fix

 Make sure Terraform has the correct project and region:

```
terraform plan \
  -var='project_id=CHANGE-THIS-SERVICE-PROJECT-ID' \
  -var='num_instances=1'
```

 Also verify:

```
gcloud config get-value project
```

 and:

```
gcloud compute zones list \
  --filter="region:(CHANGE-THIS-REGION)"
```

---

 # Error: Unsupported Argument `metadata`

 ### Error

```
An argument named "metadata" is not expected here.
```

 ### Cause

 The metadata argument was placed in the `compute_instance` module instead of the `instance_template` module.

 ### Correct location

```
module "instance_template" {
  ...

  metadata = {
    serial-port-enable = "FALSE"
  }
}
```

 Do not put:

```
metadata = {
  serial-port-enable = "FALSE"
}
```

 inside the `compute_instance` module if that module version does not expose that argument.

 Check the installed module:

```
grep -R 'variable "metadata"' \
  .terraform/modules/ 2>/dev/null
```

---

 # Error: Subnetwork Must Be in the Same Region

 ### Error

```
Subnetwork must be in the same region.
```

 ### Cause

 The address region and subnet region differ.

 ### Check subnet:

```
gcloud compute networks subnets list \
  --project="$HOST_PROJECT_ID"
```

 ### Check Terraform region:

```
grep -n "gcp_region\|region" variables.tf
```

 Both must match.

 For example:

```
Subnet:
us-east1

Address:
us-east1
```

---

 # Error: Cloud NAT Requires External IP Allocation

 ### Error

```
Either --nat-external-ip-pool or --auto-allocate-nat-external-ips must be specified for Public NAT.
```

 ### Fix

 Use:

```
gcloud compute routers nats create cymbal-gateway \
  --project="$HOST_PROJECT_ID" \
  --router=landing-zone-router \
  --region="$REGION" \
  --nat-all-subnet-ip-ranges \
  --auto-allocate-nat-external-ips
```

---

 # VM Accidentally Has an External IP

 Check:

```
gcloud compute instances list \
  --project="$SERVICE_PROJECT_ID" \
  --format="table(name,networkInterfaces[0].networkIP,networkInterfaces[0].accessConfigs[0].natIP)"
```

 If an external IP appears, inspect `main.tf`.

 Remove:

```
access_config = [{
  nat_ip       = var.nat_ip
  network_tier = var.network_tier
}]
```

 Then recreate/update the VM using the correct Terraform configuration.

---

 # Shared VPC Subnet Permission Error

 If the VM cannot use the Shared VPC subnet, verify the Service Project Compute Engine service agent.

 Get the project number:

```
gcloud projects describe "$SERVICE_PROJECT_ID" \
  --format="value(projectNumber)"
```

 Then:

```
SERVICE_PROJECT_NUMBER=$(gcloud projects describe "$SERVICE_PROJECT_ID" \
  --format='value(projectNumber)')
```

 Grant Network User:

```
gcloud projects add-iam-policy-binding "$HOST_PROJECT_ID" \
  --member="serviceAccount:service-${SERVICE_PROJECT_NUMBER}@compute-system.iam.gserviceaccount.com" \
  --role="roles/compute.networkUser"
```

 And subnet-level access:

```
gcloud compute networks subnets add-iam-policy-binding "$SUBNET_01" \
  --region="$REGION" \
  --project="$HOST_PROJECT_ID" \
  --member="serviceAccount:service-${SERVICE_PROJECT_NUMBER}@compute-system.iam.gserviceaccount.com" \
  --role="roles/compute.networkUser"
```

---

 # Reusable Lab Workflow

 For future versions of this lab, use this workflow.

 ## 1\. Copy values from Lab Details

 Record:

```
HOST_PROJECT_ID
SERVICE_PROJECT_ID
REGION
HOST_CONFIG_BUCKET
SERVICE_CONFIG_BUCKET
```

---

 ## 2\. Export variables

```
export HOST_PROJECT_ID="CHANGE-THIS"
export SERVICE_PROJECT_ID="CHANGE-THIS"
export REGION="CHANGE-THIS"

export NETWORK_NAME="shared-network"
export SUBNET_01="shared-network-subnet-01"
export SUBNET_02="shared-network-subnet-02"
```

---

 ## 3\. Verify authentication

```
gcloud auth list
```

 Verify project:

```
gcloud config set project "$HOST_PROJECT_ID"
gcloud config get-value project
```

---

 ## 4\. Deploy Shared VPC

```
Terraform
    |
    +-- VPC
    |
    +-- subnet-01
    |
    +-- subnet-02
```

---

 ## 5\. Attach Service Project

```
Host Project
     |
     +-- Shared VPC
             |
             +-- Service Project
```

---

 ## 6\. Secure the network

```
Private Google Access
Firewall
Cloud NAT
Organization Policies
```

---

 ## 7\. Deploy VM

```
Service Project
      |
      +-- VM
            |
            +-- Shared VPC subnet
            +-- No external IP
            +-- Serial port disabled
```

---

 ## 8\. Reserve internal IP

```
Service Project
      |
      +-- internal-web-server-ip
```

---

 ## 9\. Enable Security Command Center

 Follow the current lab's SCC activation instructions.

---

 # Useful Verification Commands

 ## Current account

```
gcloud auth list
```

 ## Current project

```
gcloud config get-value project
```

 ## Project details

```
gcloud projects describe "$HOST_PROJECT_ID"
```

 ## Shared VPC

```
gcloud compute shared-vpc get-host-project \
  "$SERVICE_PROJECT_ID"
```

 ## Networks

```
gcloud compute networks list \
  --project="$HOST_PROJECT_ID"
```

 ## Subnets

```
gcloud compute networks subnets list \
  --project="$HOST_PROJECT_ID"
```

 ## Firewall

```
gcloud compute firewall-rules list \
  --project="$HOST_PROJECT_ID"
```

 ## Cloud Routers

```
gcloud compute routers list \
  --project="$HOST_PROJECT_ID"
```

 ## NAT

```
gcloud compute routers nats list \
  --project="$HOST_PROJECT_ID" \
  --region="$REGION"
```

 ## VMs

```
gcloud compute instances list \
  --project="$SERVICE_PROJECT_ID"
```

 ## Internal IPs

```
gcloud compute addresses list \
  --project="$SERVICE_PROJECT_ID" \
  --regions="$REGION"
```

---

 # Terraform Best Practices for This Lab

 Before every `terraform apply`:

```
terraform fmt
terraform validate
terraform plan
```

 Recommended workflow:

```
terraform fmt
terraform validate
terraform plan
terraform apply
```

 For production environments, save the plan:

```
terraform plan -out=tfplan
```

 Then apply the exact saved plan:

```
terraform apply tfplan
```

---

 # Lab-Specific Values Quick Reference

 Use this table when starting a new lab.

 | Variable | Current Lab Example | Replace? |
| --- | --- | --- |
| Host Project ID | `qwiklabs-gcp-02-dd44f8641324` | **YES** |
| Service Project ID | `qwiklabs-gcp-02-5ee4cd1f0d7b` | **YES** |
| Region | `us-east1` | **YES** |
| Network | `shared-network` | Check lab |
| Subnet 01 | `shared-network-subnet-01` | Check lab |
| Subnet 01 CIDR | `172.16.10.0/24` | Check lab |
| Subnet 02 | `shared-network-subnet-02` | Check lab |
| Subnet 02 CIDR | `172.16.20.0/24` | Check lab |
| Firewall | `cymbal-firewall-rule` | Check lab |
| IAP range | `35.235.240.0/20` | Usually fixed |
| Cloud Router | `landing-zone-router` | Check lab |
| Cloud NAT | `cymbal-gateway` | Check lab |
| VM hostname | `cymbal-service-instance` | Check lab |
| Internal IP | `internal-web-server-ip` | Check lab |
| Host config bucket | Lab-provided | **YES** |
| Service config bucket | Lab-provided | **YES** |

---

 # Security Design Summary

 ## Shared VPC

 Centralizes network management in the Host Project.

 ## Private Google Access

 Allows private resources without external IP addresses to access supported Google APIs and services.

 ## IAP Firewall Rule

 Allows SSH through Google's IAP TCP forwarding infrastructure rather than opening SSH to the entire internet.

 Source:

```
35.235.240.0/20
```

 ## No External VM IP

 The VM cannot be directly addressed from the public internet.

 ## Cloud NAT

 Provides outbound internet connectivity for private VMs without assigning public IP addresses to those VMs.

 ## Serial Port Disabled

 Prevents serial console access through the VM metadata setting:

```
serial-port-enable=FALSE
```

 ## Internal Static IP

 Provides a predictable private address for internal application communication.

 ## Security Command Center

 Provides centralized security visibility and findings according to the configured SCC tier and scope.

---

 # Important: What Usually Changes Between Lab Runs?

 Qwiklabs challenge labs commonly generate new resources for each session.

 Expect these to change:

```
Project IDs
Project numbers
Cloud Storage bucket names
Temporary usernames
Temporary passwords
Organization IDs
Region
Zone
Provided Terraform files
Module versions
Security Command Center workflow
```

 Do **not** blindly copy values from an old lab session.

 Instead:

 1. Start the new lab.
2. Read the Lab Details panel.
3. Replace all values marked `CHANGE-THIS`.
4. Check the current task instructions.
5. Run `terraform validate`.
6. Run `terraform plan`.
7. Inspect the plan.
8. Apply only after confirming the configuration.

---

 # GitHub Setup

 After creating this README:

```
git init
```

 Create `.gitignore`:

```
cat > .gitignore <<'EOF'
.terraform/
*.tfstate
*.tfstate.*
*.tfvars
*.tfvars.json
crash.log
*.json
*.pem
*.key
credentials/
secrets/
.DS_Store
EOF
```

 Check what Git sees:

```
git status
```

 Add the README:

```
git add README.md .gitignore
```

 Commit:

```
git commit -m "Add Google Cloud landing zone challenge lab documentation"
```

 Add your GitHub repository:

```
git remote add origin CHANGE-THIS-GITHUB-REPOSITORY-URL
```

 Push:

```
git branch -M main
git push -u origin main
```

---

 # Final Checklist

 Before considering the lab complete:

```
[✓] Shared VPC created
[✓] Service Project attached
[✓] Subnet permissions configured
[✓] Private Google Access enabled
[✓] IAP SSH firewall rule configured
[✓] Cloud NAT configured
[✓] Secure VM deployed
[✓] VM has no external IP
[✓] Serial port disabled
[✓] Internal static IP reserved
[✓] Security Command Center configured
[✓] All lab checkpoints passed
```

---

 ## Disclaimer

 This README is a reusable implementation guide based on the challenge-lab workflow.

 Google Cloud, Terraform modules, Qwiklabs challenge labs, IAM requirements, and Security Command Center workflows can change over time.

 For every new lab session, treat the **current lab instructions as authoritative** and replace all values marked:

```
CHANGE-THIS
```

 before executing commands.

 Never commit temporary Qwiklabs credentials or secrets to GitHub.

 This is ready to save as `README.md` and push to GitHub. For future lab runs, the **Lab-Specific Values** and **Quick Reference** sections are the main places to update.
