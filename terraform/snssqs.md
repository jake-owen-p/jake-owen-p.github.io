<!-- ## SNS

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
``` -->