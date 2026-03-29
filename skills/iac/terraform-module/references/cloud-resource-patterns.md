# Cloud Resource Patterns

Common patterns for cloud resources managed by Terraform across providers.

## Provider-Managed Tags Lifecycle

Most cloud providers auto-populate certain tags or labels on resources. Use `lifecycle.ignore_changes` to prevent Terraform from drifting on these.

### AWS

```hcl
lifecycle {
  ignore_changes = [
    tags["CreatedBy"],
    tags["CreatedOn"],
    tags["aws:cloudformation:stack-name"],
    tags["aws:cloudformation:stack-id"],
    tags["aws:cloudformation:logical-id"],
  ]
}
```

### Azure

```hcl
lifecycle {
  ignore_changes = [
    tags["createdBy"],
    tags["createdDate"],
    tags["hidden-link:*"],
  ]
}
```

### GCP

```hcl
lifecycle {
  ignore_changes = [
    labels["goog-terraform-provisioned"],
  ]
}
```

## DNS Zone (Public)

### AWS (Route 53)

```hcl
resource "aws_route53_zone" "this" {
  name = var.zone_name

  tags = var.tags
}
```

### Azure

```hcl
resource "azurerm_dns_zone" "this" {
  name                = var.zone_name
  resource_group_name = var.resource_group_name

  tags = var.tags
}
```

### GCP

```hcl
resource "google_dns_managed_zone" "this" {
  name        = var.zone_name
  dns_name    = "${var.zone_name}."
  description = var.description
  visibility  = "public"

  labels = var.labels
}
```

## DNS Record

### AWS (Route 53)

```hcl
resource "aws_route53_record" "this" {
  for_each = var.records

  zone_id = aws_route53_zone.this.zone_id
  name    = each.value.name
  type    = each.value.type
  ttl     = each.value.ttl
  records = each.value.values
}
```

### Azure

```hcl
resource "azurerm_dns_a_record" "this" {
  for_each = var.records

  name                = each.value.name
  zone_name           = azurerm_dns_zone.this.name
  resource_group_name = var.resource_group_name
  ttl                 = each.value.ttl
  records             = each.value.values

  tags = var.tags
}
```

### GCP

```hcl
resource "google_dns_record_set" "this" {
  for_each = var.records

  managed_zone = google_dns_managed_zone.this.name
  name         = "${each.value.name}.${google_dns_managed_zone.this.dns_name}"
  type         = each.value.type
  ttl          = each.value.ttl
  rrdatas      = each.value.values
}
```

## Secret / Vault

### AWS (Secrets Manager)

```hcl
resource "aws_secretsmanager_secret" "this" {
  name = var.secret_name

  tags = var.tags

  lifecycle {
    prevent_destroy = true
  }
}
```

### Azure (Key Vault)

```hcl
resource "azurerm_key_vault" "this" {
  name                = var.vault_name
  location            = var.location
  resource_group_name = var.resource_group_name
  tenant_id           = var.tenant_id
  sku_name            = "standard"

  tags = var.tags

  lifecycle {
    prevent_destroy = true
  }
}
```

### GCP (Secret Manager)

```hcl
resource "google_secret_manager_secret" "this" {
  secret_id = var.secret_name

  replication {
    auto {}
  }

  labels = var.labels

  lifecycle {
    prevent_destroy = true
  }
}
```

## Object Storage

### AWS (S3)

```hcl
resource "aws_s3_bucket" "this" {
  for_each = var.buckets

  bucket = each.value.name

  tags = var.tags

  lifecycle {
    prevent_destroy = true
  }
}
```

### Azure (Blob Storage)

```hcl
resource "azurerm_storage_account" "this" {
  name                     = var.storage_account_name
  resource_group_name      = var.resource_group_name
  location                 = var.location
  account_tier             = "Standard"
  account_replication_type = "LRS"

  tags = var.tags

  lifecycle {
    prevent_destroy = true
  }
}
```

### GCP (GCS)

```hcl
resource "google_storage_bucket" "this" {
  for_each = var.buckets

  name     = each.value.name
  location = var.location

  labels = var.labels

  lifecycle {
    prevent_destroy = true
  }
}
```

## Container Registry

### AWS (ECR)

```hcl
resource "aws_ecr_repository" "this" {
  for_each = var.repositories

  name                 = each.value.name
  image_tag_mutability = "MUTABLE"

  tags = var.tags
}
```

### Azure (ACR)

```hcl
resource "azurerm_container_registry" "this" {
  name                = var.registry_name
  resource_group_name = var.resource_group_name
  location            = var.location
  sku                 = "Standard"

  tags = var.tags
}
```

### GCP (Artifact Registry)

```hcl
resource "google_artifact_registry_repository" "this" {
  for_each = var.repositories

  repository_id = each.value.name
  location      = var.location
  format        = "DOCKER"

  labels = var.labels
}
```

## IAM Policy

### AWS

```hcl
resource "aws_iam_policy" "this" {
  for_each = var.policies

  name        = each.key
  description = each.value.description
  policy      = each.value.policy_json

  tags = var.tags
}
```

### Azure (Role Assignment)

```hcl
resource "azurerm_role_assignment" "this" {
  for_each = var.role_assignments

  scope                = each.value.scope
  role_definition_name = each.value.role
  principal_id         = each.value.principal_id
}
```

### GCP

```hcl
resource "google_project_iam_member" "this" {
  for_each = var.iam_bindings

  project = var.project_id
  role    = each.value.role
  member  = each.value.member
}
```

## Reading Secrets (Data Sources)

### AWS

```hcl
data "aws_secretsmanager_secret_version" "this" {
  secret_id = var.secret_id
}

locals {
  secret_value = data.aws_secretsmanager_secret_version.this.secret_string
}
```

### Azure

```hcl
data "azurerm_key_vault_secret" "this" {
  name         = var.secret_name
  key_vault_id = var.key_vault_id
}

locals {
  secret_value = data.azurerm_key_vault_secret.this.value
}
```

### GCP

```hcl
data "google_secret_manager_secret_version" "this" {
  secret = var.secret_id
}

locals {
  secret_value = data.google_secret_manager_secret_version.this.secret_data
}
```

## Sources

- AWS Provider: https://registry.terraform.io/providers/hashicorp/aws/latest/docs
- Azure Provider: https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs
- GCP Provider: https://registry.terraform.io/providers/hashicorp/google/latest/docs
- Terraform Module Development: https://developer.hashicorp.com/terraform/language/modules/develop
