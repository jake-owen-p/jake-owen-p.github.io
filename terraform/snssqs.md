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