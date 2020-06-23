<!-- ## Terraform Initial Setup

### Terraform State
Terraform requires a state file to keep track of any changes. It is best to host this in the cloud so it can always be accessed.

**Remember to add your local .terraform to .gitignore**

### S3
- Create an s3 bucket `<company-name>-terraform-state`
- Create a folder for the specific service `<service-name>`

### main.tf
Terraform's entry point is a `main.tf` file that is used for `terraform init`

```terraform
terraform {
  backend "s3" {
    region  = "eu-west-1"
    bucket  = "garmentfinder-terraform-state"
    key     = "garment-finder-api/terraform.tfstate"
  }
}

provider "aws" {
  region = "eu-west-1"
}

data "aws_caller_identity" "current" {}
```

### Directory Structure
Each directory will have its own `main.tf` file. Name each directory as the service it has to create.

module "sns" {
  source = "./sns"
} -->