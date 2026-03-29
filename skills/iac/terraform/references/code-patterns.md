# Terraform Code Patterns

## Conditional Resource Creation

```hcl
variable "create_vpc" {
  description = "Whether to create VPC resources"
  type        = bool
  default     = true
}

resource "aws_vpc" "this" {
  count = var.create_vpc ? 1 : 0

  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(local.common_tags, {
    Name = "${var.project}-vpc"
  })
}

# Reference conditional resource
output "vpc_id" {
  description = "The VPC ID"
  value       = try(aws_vpc.this[0].id, null)
}
```

## Dynamic Blocks

```hcl
variable "ingress_rules" {
  description = "List of ingress rules"
  type = list(object({
    port        = number
    protocol    = string
    cidr_blocks = list(string)
    description = string
  }))
  default = []
}

resource "aws_security_group" "this" {
  name_prefix = "${var.project}-sg-"
  vpc_id      = var.vpc_id

  dynamic "ingress" {
    for_each = var.ingress_rules
    content {
      from_port   = ingress.value.port
      to_port     = ingress.value.port
      protocol    = ingress.value.protocol
      cidr_blocks = ingress.value.cidr_blocks
      description = ingress.value.description
    }
  }

  tags = local.common_tags
}
```

## for_each with Map

```hcl
variable "subnets" {
  description = "Map of subnet configurations"
  type = map(object({
    cidr_block        = string
    availability_zone = string
    public            = bool
  }))
}

resource "aws_subnet" "this" {
  for_each = var.subnets

  vpc_id                  = aws_vpc.this.id
  cidr_block              = each.value.cidr_block
  availability_zone       = each.value.availability_zone
  map_public_ip_on_launch = each.value.public

  tags = merge(local.common_tags, {
    Name = "${var.project}-${each.key}"
    Tier = each.value.public ? "public" : "private"
  })
}
```

## Local Values for Tag Management

```hcl
locals {
  common_tags = {
    Environment = var.environment
    ManagedBy   = "terraform"
    Project     = var.project
    Team        = var.team
  }
}
```

## Data Source for AMI Lookup

```hcl
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"] # Canonical

  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-*-amd64-server-*"]
  }

  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
}
```

## Module Composition

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "${var.project}-vpc"
  cidr = var.vpc_cidr

  azs             = var.availability_zones
  private_subnets = var.private_subnet_cidrs
  public_subnets  = var.public_subnet_cidrs

  enable_nat_gateway = true
  single_nat_gateway = var.environment != "prod"

  tags = local.common_tags
}
```

## Moved Block (Refactoring)

```hcl
# Rename a resource without destroying it
moved {
  from = aws_instance.web_server
  to   = aws_instance.web
}
```

## Import Block (Terraform 1.5+)

```hcl
import {
  to = aws_s3_bucket.this
  id = "existing-bucket-name"
}
```

## Backend Configuration

```hcl
# S3-compatible backend (AWS, MinIO, etc.)
terraform {
  backend "s3" {
    bucket         = "terraform-state"
    key            = "env/prod/terraform.tfstate"
    region         = "us-east-1"
    encrypt        = true
    dynamodb_table = "terraform-locks"
  }
}
```

## Provider Version Constraints

```hcl
terraform {
  required_version = ">= 1.5.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
    kubernetes = {
      source  = "hashicorp/kubernetes"
      version = "~> 2.25"
    }
    helm = {
      source  = "hashicorp/helm"
      version = "~> 2.12"
    }
  }
}
```

## Error Handling with Validation

```hcl
variable "environment" {
  description = "Deployment environment"
  type        = string

  validation {
    condition     = contains(["dev", "qa", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, qa, staging, prod."
  }
}

variable "instance_count" {
  description = "Number of instances to create"
  type        = number
  default     = 1

  validation {
    condition     = var.instance_count > 0 && var.instance_count <= 10
    error_message = "Instance count must be between 1 and 10."
  }
}
```

## Lifecycle Rules

```hcl
resource "aws_instance" "this" {
  # ...

  lifecycle {
    create_before_destroy = true          # Zero-downtime replacement
    prevent_destroy       = true          # Protect critical resources
    ignore_changes        = [tags["Updated"]]  # Ignore external changes
  }
}
```

## Sources

- [HashiCorp Style Guide](https://developer.hashicorp.com/terraform/language/style)
- [terraform-best-practices.com](https://www.terraform-best-practices.com/)
- [Anton Babenko terraform-skill](https://github.com/antonbabenko/terraform-skill)