# Introduction to Terraform and Our First AWS Infrastructure
A hands-on introduction to Infrastructure as Code (IaC) with Terraform - covering core concepts, the standard workflow, state management, and provisioning real AWS resources.
## Table of Contents
- [Understand Infrastructure as Code](#1-understand-infrastructure-as-code)
- [Install Terraform and Configure AWS](#2-install-terraform-and-configure-aws)
- [Your First Terraform Config — Create an S3 Bucket](#3-your-first-terraform-config--create-an-s3-bucket)
- [Add an EC2 Instance](#4-add-an-ec2-instance)
- [Understand the State File](#5-understand-the-state-file)
- [Modify, Plan, and Destroy](#6-modify-plan-and-destroy)

## 1. Understand Infrastructure as Code
### What is IaC and why does it matter in DevOps?
Infrastructure as Code means defining our infrastructure - servers, networks, storage, DNS - in version-controlled text files rather than clicking through a cloud console. The code describes the desired state and a tool like Terraform figures out the steps to get there.
 
| Benefit | What it means in practice |
|---|---|
| **Consistency** | The same code produces identical environments every time |
| **Automation** | Infrastructure can be provisioned in a CI/CD pipeline without human intervention |
| **Version control** | Infrastructure changes are reviewed, approved, and rolled back like application code |
| **Documentation** | The `.tf` files are the living documentation of what is deployed and why |
 
### What problems does IaC solve vs the AWS console?
| Manual console | IaC (Terraform) |
|---|---|
| Prone to human error and typos | Reproducible - same result every run |
| Hard to reproduce across environments | Dev, staging, and prod use the same code |
| No audit trail of who changed what | Git history tracks every change |
| Cannot be peer-reviewed | Pull request workflow applies to infra changes |
| Slow and sequential | Parallelizable and pipeline-ready |
 
### How does Terraform compare to other IaC tools?
| Tool | Type | Cloud support | Language | Key distinction |
|---|---|---|---|---|
| **Terraform** | Declarative | Multi-cloud | HCL | Cloud-agnostic, large provider ecosystem |
| **CloudFormation** | Declarative | AWS only | JSON / YAML | Native AWS integration, no extra tooling |
| **Ansible** | Imperative | Multi-cloud | YAML | Configuration management, not just provisioning |
| **Pulumi** | Declarative | Multi-cloud | Python, Go, TS | Uses general-purpose programming languages |
 
### Declarative and Cloud-Agnostic - what do these mean?
**Declarative:** We describe the desired end state (`"I want an EC2 instance and an S3 bucket"`), and Terraform determines the sequence of API calls needed to reach it. We never write step-by-step instructions.
 
**Cloud-Agnostic:** The same Terraform workflow - `init` → `plan` → `apply` - works across AWS, Azure, GCP, Kubernetes, GitHub, Datadog, and hundreds of other providers. One tool manages our entire stack.

## 2. Install Terraform and Configure AWS
### Install Terraform
```bash
# macOS
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
 
# Linux (amd64)
wget -O - https://apt.releases.hashicorp.com/gpg | \
  sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
  https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
  sudo tee /etc/apt/sources.list.d/hashicorp.list
  
sudo apt update && sudo apt install terraform
 
# Windows
choco install terraform
```
 
### Verify
```bash
terraform -version
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1146).png)

### Configure the AWS CLI
```bash
sudo apt install awscli
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1149).png)

```bash
aws configure
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1148).png)
 
### Verify AWS access
```bash
aws sts get-caller-identity
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1151).png)

We should see our **AWS account ID**, **user ID**, and **ARN**. If this command succeeds, Terraform will be able to authenticate with AWS.

## 3. Your First Terraform Config - Create an S3 Bucket
### Project setup
```bash
mkdir terraform-basics && cd terraform-basics
```
 
### `main.tf`
```hcl
terraform {                                         #configures Terraform itself
  required_providers {                              #declares which providers (cloud/service plugins) our configuration needs
    aws = {                                         #defines AWS provider
      source  = "hashicorp/aws"                     #tells Terraform to download AWS provider plugin maintained by HashiCorp
      version = "~> 5.0"                            #pins provider version '~>' means “compatible with 5.x releases” (e.g., 5.0, 5.1, 5.9) ensures stability
    }
  }
}
 
provider "aws" {                                    #configures how Terraform talks to AWS
  region = "ap-south-1"                             #sets AWS region (Mumbai) - all resources will be created in this region unless overridden
}
 
resource "aws_s3_bucket" "terraweek_bucket" {       #declares resource of type 'aws_s3_bucket' - "terraweek_bucket" is local name we use inside Terraform to reference this bucket
  bucket = "terraweek-atul-2026"                    #sets actual S3 bucket name - must be globally unique across all AWS accounts
  tags = {                                          #adds metadata tags
    Name = "TerraWeek-Day1-Bucket"                  #tag key/value pair - Tags help identify and organize resources in AWS
  }
}
```
 
### The Terraform lifecycle
```bash
terraform init     # downloads the AWS provider plugin
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1153).png)

```bash
terraform plan      # previews what will be created
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1154).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1156).png)

```bash
terraform apply     # creates the bucket (type 'yes' to confirm)
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1158).png)

After applying, verify the bucket exists in the AWS S3 console.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1161).png)

### What does `terraform init` do?
`terraform init` downloads the provider plugins specified in `required_providers` and sets up the working directory. It creates:
 
| Path | Contents |
|---|---|
| `.terraform/` | Provider binaries (e.g., `terraform-provider-aws_v5.x.x`) |
| `.terraform.lock.hcl` | Pins exact provider versions for reproducibility |
 
The lock file ensures that every team member and every CI/CD run uses the same provider version - no unexpected behavior from silent upgrades.

## 4. Add an EC2 Instance
Add the following resource block to `main.tf`:
 
```hcl
resource "aws_instance" "terraweek_ec2" {   #declares resource of type 'aws_instance' (an EC2 virtual machine) - "terraweek_ec2" is the local name we use inside Terraform to reference this instance
  ami           = "ami-0f5ee92e2d63afc18"   #specifies OS image for EC2 instance - this AMI ID corresponds to Amazon Linux 2 in ap-south-1 (Mumbai) region - AMI IDs are region-specific so we must use correct one for our chosen region
  instance_type = "t2.micro"                #defines hardware size of instance
 
  tags = {                                  #metadata key/value pairs attached to resources
    Name = "TerraWeek-Day1"                 #this tag makes instance easier to identify in AWS console
  }
}
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1163).png)

```bash
terraform plan    # shows 1 resource to add (EC2 only)
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1166).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1167).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1168).png)

```bash
terraform apply   # creates the instance
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1169).png)
 
Verify in the AWS EC2 console - the instance should appear with the tag `TerraWeek-Day1`.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1173).png)

### Why does the plan show only 1 resource to add?
 
Terraform tracks every resource it manages in a **state file** (`terraform.tfstate`). When we applied the S3 bucket earlier, Terraform recorded its ID, ARN, and all attributes in that file.
 
On the next `terraform plan`, Terraform:
 
1. Reads our `.tf` files (desired state)
2. Reads `terraform.tfstate` (known current state)
3. Queries AWS to detect any drift
4. Computes the diff - only the new EC2 resource is "pending"
Because the bucket already exists in state and matches the config, Terraform leaves it untouched. Only the new resource is created. This is the core of Terraform's **declarative reconciliation model**.

## 5. Understand the State File
Terraform tracks everything it creates in a state file. Time to inspect it.

Open `terraform.tfstate`

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1174).png)

- The state file (`terraform.tfstate`) is Terraform's source of truth. It is a JSON file that records every resource Terraform manages.
- We’ll see entries for each resource (`aws_s3_bucket`, `aws_instance`) with attributes like IDs, ARNs, tags, and metadata.

### Inspect the state
```bash
# Human-readable summary of all resources in state
terraform show
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1175).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1176).png)

```bash
# List all managed resources
terraform state list
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1177).png)

```bash
# Detailed attributes for a specific resource
terraform state show aws_s3_bucket.terraweek_bucket
terraform state show aws_instance.terraweek_ec2
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1178).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1179).png)

### What does the state file store for each resource?
 
| Field | Example |
|---|---|
| Resource type and name | `aws_instance.terraweek_ec2` |
| Unique identifiers | Instance ID, ARN |
| Configuration attributes | AMI, instance type, tags |
| Dependency metadata | Which resources this one depends on |
| Current status | Running, stopped, destroyed |
 
### State file rules - three things to always remember
 
| Rule | Reason |
|---|---|
| **Never manually edit `terraform.tfstate`** | It is Terraform's source of truth. Edits can corrupt state, cause drift, or break future plans. Always use `terraform state` commands. |
| **Never commit it to Git** | It contains sensitive data (resource IDs, sometimes secrets) and changes on every apply, causing merge conflicts. |
| **Use a remote backend in production** | Store state in S3 + DynamoDB (for locking) to enable team collaboration and prevent concurrent apply conflicts. |

## 6. Modify, Plan, and Destroy
### Modify the EC2 tag
In `main.tf`, update the tag:
```hcl
tags = {
  Name = "TerraWeek-Modified"
}
```
 
### Preview the change
```bash
terraform plan
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1180).png)

### Reading plan output symbols
| Symbol | Meaning | Example |
|---|---|---|
| `+` | Create a new resource | New EC2 instance |
| `-` | Destroy an existing resource | Remove S3 bucket |
| `~` | Update in place | Change a tag |
| `-/+` | Destroy and recreate | Change an immutable attribute |
 
For a tag change, we will see `~` - an in-place update. Terraform changes the tag without destroying or recreating the instance.
 
### Apply the change
 
```bash
terraform apply
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1182).png)

Check the AWS EC2 console - the instance tag should now read `TerraWeek-Modified`.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1183).png)

### Destroy all resources
 
```bash
terraform destroy
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1185).png)

Terraform removes both the S3 bucket and EC2 instance. Verify in the AWS console - both resources should be gone.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1188).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/b8987f8c5502981b826f2a874972834df9d8bccf/2026/day-61/Screenshots/Screenshot%20(1190).png)

### The complete Terraform workflow
 
```
terraform init
      │
      ▼
terraform plan        ← always review before applying
      │
      ▼
terraform apply       ← creates or updates resources
      │
      ▼
(make changes to .tf files)
      │
      ▼
terraform plan        ← review the diff
      │
      ▼
terraform apply       ← applies the diff
      │
      ▼
terraform destroy     ← tears down all managed resources
```
