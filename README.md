# vpc-via-terraform
terraform vpc=========================vpc===================

resource "aws_vpc" "main_vpc" {
  cidr_block = var.cidr_block
  enable_dns_support = true
  enable_dns_hostnames = true

  tags = {
    Name = "My_VPC"
  }
}

variable "cidr_block" {
  description = "CIDR block for the VPC"
  default     = "10.0.0.0/16"
}

output "vpc_id" {
  value = aws_vpc.main_vpc.id
}

========================================================subnets============
resource "aws_subnet" "public_subnet" {
  vpc_id                  = var.vpc_id
  cidr_block              = var.public_subnet_cidr
  availability_zone       = var.public_subnet_availability_zone
  map_public_ip_on_launch = true
  tags = {
    Name = "Public_Subnet"
  }
}

resource "aws_subnet" "private_subnet" {
  vpc_id                  = var.vpc_id
  cidr_block              = var.private_subnet_cidr
  availability_zone       = var.private_subnet_availability_zone
  tags = {
    Name = "Private_Subnet"
  }
}

variable "vpc_id" {
  description = "VPC ID where subnets will be created"
}

variable "public_subnet_cidr" {
  description = "CIDR block for the public subnet"
  default     = "10.0.1.0/24"
}

variable "private_subnet_cidr" {
  description = "CIDR block for the private subnet"
  default     = "10.0.2.0/24"
}

variable "public_subnet_availability_zone" {
  description = "Availability zone for the public subnet"
  default     = "ap-south-1a"
}

variable "private_subnet_availability_zone" {
  description = "Availability zone for the private subnet"
  default     = "ap-south-1b"

}

output "public_subnet_id" {
  value = aws_subnet.public_subnet.id
}

output "private_subnet_id" {
  value = aws_subnet.private_subnet.id
}


===================================igw====================
resource "aws_internet_gateway" "main" {
  vpc_id = var.vpc_id
}

variable "vpc_id" {
  description = "VPC ID for the IGW association"
}

output "internet_gateway_id" {
  value = aws_internet_gateway.main.id
}


================================NAT=========================

resource "aws_eip" "nat_eip" {
  vpc = true
}

resource "aws_nat_gateway" "main" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = var.public_subnet_id
}

variable "public_subnet_id" {
  description = "ID of the public subnet"
}

output "nat_gateway_id" {
  value = aws_nat_gateway.main.id
}

output "nat_eip" {
  value = aws_eip.nat_eip.public_ip
}

============================route table======================

resource "aws_route_table" "public" {
  vpc_id = var.vpc_id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = var.internet_gateway_id
  }
}

resource "aws_route_table" "private" {
  vpc_id = var.vpc_id

  route {
    cidr_block    = "0.0.0.0/0"
    nat_gateway_id = var.nat_gateway_id
  }
}

resource "aws_route_table_association" "public_association" {
  subnet_id      = var.public_subnet_id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private_association" {
  subnet_id      = var.private_subnet_id
  route_table_id = aws_route_table.private.id
}

variable "vpc_id" {
  description = "VPC ID"
}

variable "internet_gateway_id" {
  description = "Internet Gateway ID"
}

variable "nat_gateway_id" {
  description = "NAT Gateway ID"
}

variable "public_subnet_id" {
  description = "Public Subnet ID"
}

variable "private_subnet_id" {
  description = "Private Subnet ID"
}

==================================ec2===================
resource "aws_instance" "web_server" {
  ami                         = var.ami_id
  instance_type               = var.instance_type
  subnet_id                   = var.private_subnet_id
  associate_public_ip_address = false

  user_data = <<-EOT
              #!/bin/bash
              apt update -y
              apt install -y httpd
              systemctl start httpd
              systemctl enable httpd
              echo "Hello, World!" > /var/www/html/index.html
              EOT

  tags = {
    Name = "Private EC2 Instance"
  }
}


variable "ami_id" {
  description = "AMI ID for EC2 instance"
}

variable "instance_type" {
  description = "EC2 instance type"
  default     = "t2.micro"
}

variable "private_subnet_id" {
  description = "Private subnet where the EC2 instance will be created"
}


output "instance_id" {
  value = aws_instance.web_server.id
}

output "public_ip" {
  value = aws_instance.web_server.public_ip
}


==========================load balancer=========================
resource "aws_lb" "app_lb" {
  name               = "app-lb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = var.security_groups
  subnets            = var.subnets
  enable_deletion_protection = false
  idle_timeout       = 3600  # Timeout in seconds (60 minutes)
  enable_cross_zone_load_balancing = true
}

resource "aws_lb_target_group" "target_group" {
  name     = "target-group"
  port     = 80
  protocol = "HTTP"
  vpc_id   = var.vpc_id
}

resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.app_lb.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type             = "fixed-response"
    fixed_response {
      status_code = "200"
      content_type = "text/plain"
      message_body = "Hello from ALB"
    }
  }
}

resource "aws_lb_target_group_attachment" "web_server" {
  target_group_arn = aws_lb_target_group.target_group.arn
  target_id        = var.ec2_instance_id
  port             = 80
}

variable "subnets" {
  description = "Subnets to deploy the load balancer"
}

variable "security_groups" {
  description = "Security groups to associate with the load balancer"
}

variable "vpc_id" {
  description = "VPC ID"
}

variable "ec2_instance_id" {
  description = "EC2 instance ID"
}

output "load_balancer_dns_name" {
  value = aws_lb.app_lb.dns_name
}



===========================================main module=============================
# Provider configuration
provider "aws" {
  region = "ap-south-1"
}

# VPC Module
module "vpc" {
  source     = "./vpc"
  cidr_block = "10.0.0.0/16"
}

# Subnet Module
module "subnet" {
  source                         = "./subnet"
  vpc_id                         = module.vpc.vpc_id
  public_subnet_cidr             = "10.0.1.0/24"
  private_subnet_cidr            = "10.0.2.0/24"
  public_subnet_availability_zone = "ap-south-1a"
  private_subnet_availability_zone = "ap-south-1b"
}

# Internet Gateway Module
module "internet_gateway" {
  source  = "./igw"
  vpc_id  = module.vpc.vpc_id
}

# NAT Gateway Module
module "nat_gateway" {
  source           = "./nat"
  public_subnet_id = module.subnet.public_subnet_id
}

# Route Table Module
module "route_table" {
  source              = "./route_table"
  vpc_id              = module.vpc.vpc_id
  internet_gateway_id = module.internet_gateway.internet_gateway_id
  nat_gateway_id      = module.nat_gateway.nat_gateway_id
  public_subnet_id    = module.subnet.public_subnet_id
  private_subnet_id   = module.subnet.private_subnet_id
}

# EC2 Instance Module
module "ec2_instance" {
  source           = "./ec2"
  ami_id           = "ami-0dee22c13ea7a9a67" # Example AMI for Amazon Linux 2, update as needed
  instance_type    = "t2.micro"
  private_subnet_id = module.subnet.private_subnet_id
}




# Load Balancer Module
module "load_balancer" {
  source          = "./load_balancer"
  vpc_id          = module.vpc.vpc_id
  subnets         = [module.subnets.public_subnet_id, module.subnets.private_subnet_id]
  security_groups = [] # Provide security group IDs if necessary
  ec2_instance_id = module.ec2_instance.instance_id
}



output "load_balancer_dns_name" {
  value = module.load_balancer.load_balancer_dns_name
}

output "ec2_public_ip" {
  value = module.ec2_instance.public_ip
}
