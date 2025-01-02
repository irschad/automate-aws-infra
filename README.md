# Automate AWS Infrastructure

## Project Overview
This project demonstrates the automation of AWS infrastructure provisioning and Docker container deployment using Terraform. It simplifies the creation and management of essential AWS resources like VPCs, subnets, route tables, internet gateways, security groups, and EC2 instances, with the added functionality of deploying a containerized application.

## Technologies
- **Terraform**: Infrastructure as Code (IaC) tool for automating AWS resource creation.
- **AWS**: Cloud platform for deploying and hosting the infrastructure.
- **Docker**: Containerization technology for deploying the application.
- **Linux**: Operating system for managing servers and scripting.
- **Git**: Version control for managing project files.

## Project Description
The project consists of two main parts:
1. **Automate AWS Infrastructure**:
   - Provision essential AWS components: VPC, Subnet, Route Table, Internet Gateway, Security Group, and EC2 instances.
   - Configure a secure environment by managing access rules.

2. **Automate Docker Deployment**:
   - Deploy a Docker container running an NGINX server to the provisioned EC2 instance.
   - Use Terraform scripts to execute custom startup scripts on the instance.


## Getting Started

### Prerequisites
Ensure the following tools are installed on the local machine:
- Terraform (>= 1.0.0)
- AWS CLI
- Git
- A valid AWS account with programmatic access


## Key Components

### Terraform Configuration
The main configuration is defined in `main.tf`, which includes the following resources:
- **VPC and Subnet**:
  ```hcl
  resource "aws_vpc" "myapp-vpc" {
      cidr_block = var.vpc_cidr_block
      tags = {
          Name = "${var.env_prefix}-vpc"
      }
  }

  resource "aws_subnet" "myapp-subnet-1" {
      vpc_id = aws_vpc.myapp-vpc.id
      cidr_block = var.subnet_cidr_block
      availability_zone = var.avail_zone
      tags = {
          Name = "${var.env_prefix}-subnet-1"
      }
  }
  ```

- **Internet Gateway and Route Table**:
  ```hcl
  resource "aws_internet_gateway" "myapp-igw" {
      vpc_id = aws_vpc.myapp-vpc.id
      tags = {
          Name = "${var.env_prefix}-igw"
      }
  }

  resource "aws_default_route_table" "main-rtb" {
      default_route_table_id = aws_vpc.myapp-vpc.default_route_table_id

      route {
          cidr_block = "0.0.0.0/0"
          gateway_id = aws_internet_gateway.myapp-igw.id
      }
      tags = {
          Name = "${var.env_prefix}-main-rtb"
      }
  }
  ```

- **Security Group**:
  ```hcl
  resource "aws_default_security_group" "default-sg" {
      vpc_id = aws_vpc.myapp-vpc.id

      ingress {
          from_port = 22
          to_port = 22
          protocol = "tcp"
          cidr_blocks = [var.my_ip]
      }

      ingress {
          from_port = 8080
          to_port = 8080
          protocol = "tcp"
          cidr_blocks = ["0.0.0.0/0"]
      }

      egress {
          from_port = 0
          to_port = 0
          protocol = "-1"
          cidr_blocks = ["0.0.0.0/0"]
      }

      tags = {
          Name = "${var.env_prefix}-default-sg"
      }
  }
  ```

- **EC2 Instance and Docker Deployment**:
  ```hcl
  resource "aws_instance" "myapp-server" {
      ami = data.aws_ami.latest-amazon-linux-image.id
      instance_type = var.instance_type

      subnet_id = aws_subnet.myapp-subnet-1.id
      vpc_security_group_ids = [aws_default_security_group.default-sg.id]
      availability_zone = var.avail_zone

      associate_public_ip_address = true
      key_name = aws_key_pair.ssh-key.key_name

      user_data = file("entry-script.sh")
      tags = {
          Name = "${var.env_prefix}-server"
      }
  }
  ```

### Docker Deployment Script
The `entry-script.sh` file initializes Docker and deploys an NGINX container:
```bash
#!/bin/bash

# Install and start docker
sudo yum update -y && sudo yum install -y docker
sudo systemctl start docker

# Add ec2-user to docker group to allow it to call docker commands
sudo usermod -aG docker ec2-user

# Start a docker container running nginx
docker run -p 8080:80 nginx
```

### Setup Instructions

1. **Clone the Repository**:
   ```bash
   git clone https://github.com/irschad/automate-aws-infra.git
   cd automate-aws-infra
   ```

2. **Prepare Terraform Variables**:
   Create a `terraform.tfvars` file and specify the following variables:
   ```hcl
   env_prefix = "dev"
   avail_zone = "us-east-1a"
   vpc_cidr_block = "10.0.0.0/16"
   subnet_cidr_block = "10.0.10.0/24"
   my_ip = "<your-ip-address/32>"
   instance_type = "t2.micro"
   public_key_location = "<path-to-your-public-key>"
   ```

3. **Initialize Terraform**:
   ```bash
   terraform init
   ```

4. **Plan and Apply Changes**:
   To preview changes:
   ```bash
   terraform plan
   ```
   To apply changes:
   ```bash
   terraform apply --auto-approve
   ```

5. **Access the Application**:
   Once the EC2 instance is running, retrieve its public IP:
   ```bash
   terraform output ec2_public_ip
   ```
   
## Outputs
The project provides the following outputs:
- **AMI ID**:
  ```hcl
  output "aws_ami_id" {
      value = data.aws_ami.latest-amazon-linux-image.id
  }
  ```
- **EC2 Public IP**:
  ```hcl
  output "ec2_public_ip" {
      value = aws_instance.myapp-server.public_ip
  }
  ```


## Validate the Setup

- Open the browser and navigate to http://<public-ip>:8080 to see the NGINX welcome page.

  Replace <public-ip> with the IP address displayed in the Terraform output.

- SSH into the EC2 instance and verify the Docker container:
  ```bash
  docker ps
  ```

