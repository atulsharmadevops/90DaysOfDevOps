# Terraform - Providers, Resources and Dependencies
A hands-on guide to configuring the AWS provider, building a VPC from scratch, understanding implicit and explicit dependencies, and controlling resource lifecycle behavior.
 
## Table of Contents
- [Explore the AWS Provider](#1-explore-the-aws-provider)
- [Build a VPC from Scratch](#2-build-a-vpc-from-scratch)
- [Understand Implicit Dependencies](#3-understand-implicit-dependencies)
- [Add a Security Group and EC2 Instance](#4-add-a-security-group-and-ec2-instance)
- [Explicit Dependencies with depends_on](#5-explicit-dependencies-with-depends_on)
- [Lifecycle Rules and Destroy](#6-lifecycle-rules-and-destroy)

## 1. Explore the AWS Provider
### Create project directory
```bash
#Set up a clean workspace for Terraform files.
mkdir terraform-aws-infra && cd terraform-aws-infra
```

### Write `providers.tf`
Define Terraform settings and AWS provider
```hcl
terraform {                       # configures terraform itself, not AWS  
  required_providers {            # declares which external plugins terraform must download
    aws = {                       # defines AWS provider
      source  = "hashicorp/aws"   # tells terraform to fetch AWS provider maintained by HashiCorp from terraform registry
      version = "~> 5.0"          # pins provider version - '~> 5.0' means “any version starting at 5.0 up to but not including 6.0.”
    }
  }
}

provider "aws" {                  # configures how terraform talks to AWS
  region = "ap-south-1"           # sets AWS region (Mumbai) - every resource create will live in this region unless override it
}
```

### Initialize Terraform
```bash
terraform init    # Download provider plugins and set up local state.
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1239).png)

### Inspect lock file
```bash
cat .terraform.lock.hcl     # pins exact provider versions
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1240).png)

### Version constraint syntax
| Constraint | Meaning | Example versions allowed |
|---|---|---|
| `~> 5.0` | Any 5.x version, but not 6.0+ | 5.0, 5.1, 5.99 |
| `>= 5.0` | 5.0 or anything higher | 5.0, 6.0, 7.2 |
| `= 5.0.0` | Only this exact version | 5.0.0 only |

`~> 5.0` (the pessimistic constraint operator) is the standard recommendation - it allows patch and minor updates within the major version while protecting against breaking changes in major version bumps.

## 2. Build a VPC from Scratch
### `main.tf`
```hcl
# VPC
resource "aws_vpc" "main" {                             # declares new AWS VPC resource, named 'main' in terraform
  cidr_block = "10.0.0.0/16"                            # defines IP range for VPC (65,536 addresses)
  tags = {
    Name = "TerraWeek-VPC"                              # adds a human‑readable name in AWS console 'TerraWeek-VPC'
  }
}
 
# Subnet
resource "aws_subnet" "public" {                        # creates a subnet inside VPC
  vpc_id                  = aws_vpc.main.id             # implicit dependency - subnet belongs to VPC
  cidr_block              = "10.0.1.0/24"               # subnet range (256 IPs)
  map_public_ip_on_launch = true                        # ensures EC2 instances launched here get public IPs automatically
  tags = {
    Name = "TerraWeek-Public-Subnet"
  }
}
 
# Internet Gateway
resource "aws_internet_gateway" "gw" {                  # creates an IGW to connect VPC to internet
  vpc_id = aws_vpc.main.id                              # attaches IGW to VPC
  tags = {
    Name = "TerraWeek-IGW"
  }
}
 
# Route Table
resource "aws_route_table" "rt" {                       # creates a route table in VPC
  vpc_id = aws_vpc.main.id
 
  route {                                               # adds a default route
    cidr_block = "0.0.0.0/0"                            # matches all IPs
    gateway_id = aws_internet_gateway.gw.id             # sends traffic to IGW
  }
 
  tags = {
    Name = "TerraWeek-RT"
  }
}
 
# Route Table Association
resource "aws_route_table_association" "assoc" {        # links subnet to route table
  subnet_id      = aws_subnet.public.id                 # associates public subnet
  route_table_id = aws_route_table.rt.id                # associates route table with default internet route - this ensures instances in subnet can reach internet
}
```
 
### What each resource does
| Resource | Purpose |
|---|---|
| `aws_vpc` | Isolated network with a `10.0.0.0/16` address space (65,536 IPs) |
| `aws_subnet` | Public subnet inside the VPC; auto-assigns public IPs to instances |
| `aws_internet_gateway` | Attaches to the VPC and enables internet traffic |
| `aws_route_table` | Defines a default route (`0.0.0.0/0`) pointing to the IGW |
| `aws_route_table_association` | Links the subnet to the route table |
 
### Format, plan, and apply
```bash
terraform fmt           # auto-formats HCL style
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1241).png)

```bash
terraform plan          # should show 5 resources to create
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1242).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1243).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1244).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1245).png)

```bash
terraform apply -auto-approve
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1246).png)

### Verify in the AWS console
- **VPC** → Your VPCs → `TerraWeek-VPC`

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1247).png)

- **Subnets** → `TerraWeek-Public-Subnet`

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1248).png)

- **Internet Gateways** → attached to `TerraWeek-VPC`

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1249).png)

- **Route Tables** → default route (`0.0.0.0/0`) pointing to the IGW
- **Route Table Associations** → subnet linked to the route table

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1250).png)

## 3. Understand Implicit Dependencies
Look at our `main.tf` carefully:
1. The subnet references `aws_vpc.main.id` -- this is an implicit dependency
2. The internet gateway references the VPC ID -- another implicit dependency
3. The route table association references both the route table and the subnet

Terraform analyzes attribute references between resources and automatically builds a **dependency graph (DAG)**. When resource B references an attribute of resource A, Terraform knows A must be created first — without any explicit ordering from you.
 
### Dependency map for this VPC config
```
aws_vpc.main
  ├── aws_subnet.public           (vpc_id = aws_vpc.main.id)
  ├── aws_internet_gateway.gw     (vpc_id = aws_vpc.main.id)
  └── aws_route_table.rt          (vpc_id = aws_vpc.main.id)
        └── [also needs gw]       (gateway_id = aws_internet_gateway.gw.id)
              └── aws_route_table_association.assoc
                    (subnet_id      = aws_subnet.public.id)
                    (route_table_id = aws_route_table.rt.id)
```
 
### All implicit dependencies in `main.tf`
| Resource | Depends on | Via |
|---|---|---|
| `aws_subnet.public` | `aws_vpc.main` | `vpc_id = aws_vpc.main.id` |
| `aws_internet_gateway.gw` | `aws_vpc.main` | `vpc_id = aws_vpc.main.id` |
| `aws_route_table.rt` | `aws_vpc.main` | `vpc_id = aws_vpc.main.id` |
| `aws_route_table.rt` | `aws_internet_gateway.gw` | `gateway_id = aws_internet_gateway.gw.id` |
| `aws_route_table_association.assoc` | `aws_subnet.public` | `subnet_id = aws_subnet.public.id` |
| `aws_route_table_association.assoc` | `aws_route_table.rt` | `route_table_id = aws_route_table.rt.id` |
 
### Why you cannot create the subnet before the VPC
If we attempted to create the subnet first, the AWS API would return:
 
```
InvalidVpcID.NotFound: The vpc ID does not exist
```
 
Terraform prevents this by sequencing resource creation according to the dependency graph - resources with no dependencies are created in parallel, and dependent resources wait until their prerequisites are complete.

## 4. Add a Security Group and EC2 Instance
### Add Security Group
Open our existing `main.tf` and append:
```hcl
# Security Group
resource "aws_security_group" "sg" {          # tells Terraform we’re creating a Security Group in AWS - "sg" - local name inside Terraform - reference it as 'aws_security_group.sg'
  vpc_id = aws_vpc.main.id                    # associates this Security Group with our VPC (aws_vpc.main) - implicit dependency, SG cannot exist until VPC exists

  # Allow SSH
  ingress {                                   # defines inbound traffic rules
    from_port   = 22                          
    to_port     = 22                          # opens port 22 (SSH)
    protocol    = "tcp"                       # only TCP traffic allowed
    cidr_blocks = ["0.0.0.0/0"]               # allows SSH from anywhere (all IPs)
  }

  # Allow HTTP
  ingress {
    from_port   = 80                          # opens port 80 (HTTP) for web traffic
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # Allow all outbound
  egress {                                    # defines outbound traffic rules
    from_port   = 0
    to_port     = 0                           # effectively all ports
    protocol    = "-1"                        # means “all protocols.”
    cidr_blocks = ["0.0.0.0/0"]               # allows outbound traffic to anywhere
  }

  tags = {
    Name = "TerraWeek-SG"                     # adds a name tag so we can easily identify SG in AWS console
  }
}
```
| Rule | Port | Direction | Purpose |
|---|---|---|---|
| Ingress | 22 | Inbound | SSH access |
| Ingress | 80 | Inbound | HTTP traffic |
| Egress | All | Outbound | Unrestricted outbound |

### Add EC2 Instance
Append:
```hcl
# EC2 Instance
resource "aws_instance" "main" {
  ami                         = "ami-0db56f446d44f2f09"       # Amazon Linux 2 (ap-south-1)
  instance_type               = "t2.micro"
  subnet_id                   = aws_subnet.public.id          # places instance inside our public subnet - implicit dependency, Subnet must exist before instance is created
  vpc_security_group_ids      = [aws_security_group.sg.id]    # attaches Security Group we created (TerraWeek-SG) - ensures inbound SSH (22) and HTTP (80) are allowed - implicit dependency, SG must exist before instance
  associate_public_ip_address = true                          # ensures instance gets a public IP so it’s reachable from internet - without this, we’d only have a private IP inside VPC

  tags = {
    Name = "TerraWeek-Server"
  }
}
```

### Format & Plan
```bash
terraform fmt
terraform plan
```
We should now see 7 resources: VPC, Subnet, IGW, Route Table, Association, SG, EC2.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1251).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1257).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1252).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1253).png)

### Apply
```bash
terraform apply -auto-approve
```
Terraform provisions everything in the correct order.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1255).png)

### Verify in AWS Console
- **EC2 → Instances**: Look for `TerraWeek-Server`.
- Confirm:
    - Public IP assigned.

      ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1260).png)

    - Security group attached.

      ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1261).png)

- Test connectivity:
    - SSH:
        ```bash
        ssh -i your-key.pem ec2-user@<public-ip>
        ```
    - HTTP:
        - Visit `http://<public-ip>` (if we install Nginx later).

## 5. Explicit Dependencies with depends_on
Implicit dependencies work when one resource references another's attributes. But sometimes two resources need to be ordered without any attribute reference - this requires `depends_on`.

### Add an S3 Bucket
Open our existing `main.tf` and append:
```hcl
# Random suffix to ensure unique bucket name
resource "random_id" "suffix" {                       # creates a random identifier - ensures S3 bucket name is globally unique
  byte_length = 4                                     # generates 4 random bytes (32 bits) - terraform exposes this as a hex string 'random_id.suffix.hex'
}

# S3 Bucket for logs
resource "aws_s3_bucket" "logs" {                     # creates an S3 bucket named logs in terraform
  bucket = "terraweek-logs-${random_id.suffix.hex}"   # bucket name includes random suffix, e.g. 'terraweek-logs-a1b2c3d4'

  depends_on = [aws_instance.main]                    # explicit dependency: forces terraform to wait until EC2 instance is created before provisioning this bucket - normally terraform would create bucket in parallel since there’s no direct reference

  tags = {
    Name = "TerraWeek-Logs"
  }
}
```
> `random_id` generates a random hex string to avoid S3 bucket name collisions - bucket names are globally unique across all AWS accounts.
 
Without `depends_on`, Terraform would create the S3 bucket and EC2 instance in parallel since neither references the other's attributes. `depends_on` explicitly tells Terraform to wait until `aws_instance.main` is fully provisioned before creating the bucket.

### Run Plan
```bash
terraform plan
```
![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1262).png)

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1264).png)

- Observe the order: EC2 instance is created before the S3 bucket.
- Without `depends_on`, Terraform might create them in parallel since no attribute references exist.

### Visualize Dependency Graph
```bash
terraform graph | dot -Tpng > graph.png
```
- If we don’t have Graphviz installed, run:
    ```bash
    terraform graph
    ```
    Copy the DOT output into `webgraphviz.com`.

We’ll see a DAG (directed acyclic graph) showing VPC → Subnet → IGW → Route Table → Association → SG → Instance → S3 bucket.

![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1266).png)

### When to use `depends_on` - real examples
| Scenario | Why `depends_on` is needed |
|---|---|
| Database migration script | Must run only after the RDS instance is fully available |
| CloudWatch alarms | Should be created after the target EC2 or load balancer exists |
| IAM policy attachment | Policy must exist before the role that references it |
 
### Implicit vs explicit dependencies
| Type | How defined | When to use |
|---|---|---|
| **Implicit** | Attribute reference (`resource_type.name.attribute`) | When one resource uses another's output value |
| **Explicit** (`depends_on`) | Direct declaration | When ordering is required but no attribute is shared |

## 6. Lifecycle Rules and Destroy
Lifecycle blocks lets us control how Terraform handles resource replacement, protection, and drift.

### Add Lifecycle Block
Open our `main.tf` and update the EC2 instance resource:
```hcl
# EC2 Instance
resource "aws_instance" "main" {
  ami                         = "ami-0b4ca4c7a210c8dc6" # Amazon Linux 2 (ap-south-1)
  instance_type               = "t2.micro"
  subnet_id                   = aws_subnet.public.id
  vpc_security_group_ids      = [aws_security_group.sg.id]
  associate_public_ip_address = true

  tags = {
    Name = "TerraWeek-Server"
  }

  lifecycle {
    create_before_destroy = true
  }
}
```

### Change AMI
- **Replace the AMI ID with another Amazon Linux 2 AMI for our region** (e.g., a newer patch version).
- Run:
    ```bash
    terraform plan
    ```

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1268).png)

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1269).png)

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1270).png)

- Without `create_before_destroy`, Terraform would destroy the old instance first and then create the new one - causing downtime. With `create_before_destroy = true`, the new instance is fully running before the old one is terminated.

### Destroy Everything
- Run:
    ```bash
    terraform destroy -auto-approve
    ```

    ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1273).png)

- Terraform destroys resources in **reverse dependency order**:
  ```
  S3 bucket → EC2 → Security Group → Route Table Association
    → Route Table → Internet Gateway → Subnet → VPC
  ```
- Verify in the AWS console - all resources should be gone.

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1275).png)

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1276).png)

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1277).png)

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1278).png)

  ![image alt](https://github.com/atulsharmadevops/90DaysOfDevOps/blob/1ca6c7805eb593d0d083e2bb8bb5eaab4bac6e52/2026/day-62/Screenshots/Screenshot%20(1279).png)

### The three lifecycle arguments
| Argument | Behavior | When to use |
|---|---|---|
| `create_before_destroy` | Creates the replacement first, then destroys the old resource | EC2 instances, load balancers - anything where downtime is unacceptable |
| `prevent_destroy` | Blocks `terraform destroy` from removing this resource | Production databases, S3 buckets with critical data |
| `ignore_changes` | Ignores diffs for specific attributes | Tags managed by another system, auto-scaled instance counts, externally managed fields |
 
### `ignore_changes` example
```hcl
lifecycle {
  ignore_changes = [tags, ami]
}
```
Useful when a separate tagging system or auto-scaler modifies attributes outside of Terraform — without `ignore_changes`, Terraform would try to revert those changes on every apply.

### Terraform workflow summary for this project
 
```
terraform init
      │
      ▼
Write providers.tf + main.tf
      │
      ▼
terraform fmt       ← format HCL
terraform plan      ← review 7 resources
terraform apply     ← provision VPC, Subnet, IGW, RT, SG, EC2, S3
      │
      ▼
Modify (e.g., change AMI or tag)
      │
      ▼
terraform plan      ← see ~ (update) or -/+ (replace)
terraform apply     ← apply the diff
      │
      ▼
terraform destroy   ← tear down in reverse dependency order
```