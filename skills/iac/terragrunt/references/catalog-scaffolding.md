# Catalog Scaffolding

The `terragrunt catalog` command provides an interactive way to browse available units and stacks from your catalog and scaffold new deployments.

## Browsing the Catalog

Run from your live repository (where `root.hcl` with a `catalog` block exists):

```bash
# Launch interactive catalog browser
terragrunt catalog
```

This displays all available units and stacks from the configured catalog.

## Catalog Configuration in root.hcl

```hcl
catalog {
  urls = [
    "git@github.com:YOUR_ORG/infrastructure-catalog.git",
    "git@github.com:YOUR_ORG/infrastructure-extra-catalog.git"  # Multiple catalogs supported
  ]
}
```

## Using Boilerplate for Scaffolding

When you select a unit or stack from the catalog, Terragrunt uses [Boilerplate](https://github.com/gruntwork-io/boilerplate) to scaffold the configuration. Units and stacks can include a `boilerplate.yml` to prompt for required values:

```yaml
# units/database/boilerplate.yml
variables:
  - name: name
    description: "Name of the database instance"
    type: string

  - name: environment
    description: "Environment (dev, qa, prod)"
    type: string
    default: "dev"

  - name: instance_type
    description: "Database instance type"
    type: string
    default: "db.t3.medium"

  - name: version
    description: "Module version to use"
    type: string
    default: "v1.0.0"
```

## Scaffold a New Deployment

```bash
# Navigate to target directory
cd accounts/ORG_NAME/ORG_NAME-dev/us-east-1/components/my-service

# Browse and scaffold from catalog
terragrunt catalog

# Or scaffold directly by URL
terragrunt scaffold git@github.com:YOUR_ORG/infrastructure-catalog.git//units/database
```

## Boilerplate Templates

The boilerplate template generates the final terragrunt.hcl:

```hcl
# modules/database/.boilerplate/terragrunt.hcl
terraform {
  source = "git::git@github.com:YOUR_ORG/modules/database.git//app?ref={{ .ModuleVersion }}"
}

inputs = {
  name          = "{{ .Name }}"
  instance_type = "{{ .InstanceType }}"
}
```

## Scaffold Directory Structure

When scaffolding, Terragrunt:
1. Clones the catalog repository (if not cached)
2. Presents available units/stacks for selection
3. Prompts for boilerplate variables
4. Generates terragrunt.hcl in the current directory

## References

- [Terragrunt Catalog Documentation](https://terragrunt.gruntwork.io/docs/features/catalog/)
- [Boilerplate Documentation](https://github.com/gruntwork-io/boilerplate)
