# Terraform

## Terraform Initial Setup

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
}


## SNS

```terraform
resource "aws_sns_topic" "garment_scraper" {
  name = "garment-scraper-topic"
}
```

## SQS

```terraform
resource "aws_sqs_queue" "garment_scraper_queue" {
  name = "garment-scraper-queue"
}
```

## Topic Subscription

```terraform
resource "aws_sns_topic_subscription" "user_updates_sqs_target" {
  topic_arn = aws_sns_topic.garment_scraper_topic.arn
  protocol  = "sqs"
  endpoint  = aws_sqs_queue.garment_scraper_queue.arn
}
```

## SQS Topic Policy
To be able to send messages, you have to explicity apply this to the queue with `sqs:sendMessage`

```terraform
resource "aws_sqs_queue_policy" "garment_scraper_queue_policy" {
  queue_url = "${aws_sqs_queue.garment_scraper_queue.id}"

  policy = <<POLICY
{
  "Version": "2012-10-17",
  "Id": "sqspolicy",
  "Statement": [
    {
      "Sid": "First",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "sqs:SendMessage",
      "Resource": "${aws_sqs_queue.garment_scraper_queue.arn}",
      "Condition": {
        "ArnEquals": {
          "aws:SourceArn": "${aws_sns_topic.garment_scraper_topic.arn}"
        }
      }
    }
  ]
}
POLICY
}
```