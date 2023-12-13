Certainly! Below is an example Terraform script for creating an AWS Lambda function using `boto3` in Python to delete data older than 8 years with the 'converted' prefix in an S3 bucket and send a notification using Amazon SNS to an email address (e.g., "anurag@gmail.com") after deleting the data:

```hcl
provider "aws" {
  region = "your_aws_region"
}

resource "aws_lambda_function" "cleanup_function" {
  filename      = "cleanup_lambda.zip"
  function_name = "cleanupLambdaFunction"
  role          = aws_iam_role.lambda_execution_role.arn
  handler       = "lambda_function.lambda_handler"
  runtime       = "python3.8"
}

resource "aws_iam_role" "lambda_execution_role" {
  name = "lambda_execution_role"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      }
    }
  ]
}
EOF
}

data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = "${path.module}/lambda_code"
  output_path = "${path.module}/cleanup_lambda.zip"
}

// Define the Lambda function code in a separate folder
data "aws_s3_bucket_object" "lambda_code" {
  bucket = "your_s3_bucket_name"
  key    = "lambda_code.zip"
}

resource "aws_s3_bucket_object" "upload_lambda_code" {
  bucket = "your_s3_bucket_name"
  key    = "lambda_code.zip"
  source = data.aws_s3_bucket_object.lambda_code.source
}

resource "aws_lambda_permission" "allow_s3" {
  statement_id  = "AllowS3Invocation"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.cleanup_function.function_name
  principal     = "s3.amazonaws.com"
  source_arn    = "arn:aws:s3:::your_s3_bucket_name"
}

resource "aws_s3_bucket_notification" "bucket_notification" {
  bucket = "your_s3_bucket_name"

  lambda_function {
    lambda_function_arn = aws_lambda_function.cleanup_function.arn
    events              = ["s3:ObjectRemoved:*"]
    filter_prefix       = "converted/"
  }
}

resource "aws_sns_topic" "notification_topic" {
  name = "cleanup_notification_topic"
}

resource "aws_lambda_permission" "allow_sns" {
  statement_id  = "AllowSNSInvocation"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.cleanup_function.function_name
  principal     = "sns.amazonaws.com"
  source_arn    = aws_sns_topic.notification_topic.arn
}

resource "aws_sns_subscription" "email_subscription" {
  topic_arn = aws_sns_topic.notification_topic.arn
  protocol  = "email"
  endpoint  = "anurag@gmail.com"
}
```

In this example, the Lambda function is written in Python, and its code should be in a folder named `lambda_code` at the same level as your Terraform script.

Here's an example `lambda_function.py`:

```python
import boto3
from datetime import datetime, timedelta

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    sns = boto3.client('sns')

    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']

        # Check if the object has the 'converted' prefix and is older than 8 years
        if key.startswith('converted/') and is_old_object(bucket, key):
            # Delete the object
            s3.delete_object(Bucket=bucket, Key=key)

            # Send notification to SNS
            send_notification(sns, "Cleanup Notification", f"Deleted old object: s3://{bucket}/{key}")

def is_old_object(bucket, key):
    s3 = boto3.client('s3')
    response = s3.head_object(Bucket=bucket, Key=key)
    last_modified = response['LastModified']
    age_threshold = datetime.now() - timedelta(days=8 * 365)

    return last_modified < age_threshold

def send_notification(sns, subject, message):
    sns.publish(
        TopicArn=YOUR_SNS_TOPIC_ARN,  # Replace with your SNS topic ARN
        Subject=subject,
        Message=message
    )
```

Make sure to replace placeholder values like `"your_s3_bucket_name"` and `"YOUR_SNS_TOPIC_ARN"` with your actual S3 bucket name and SNS topic ARN, respectively. Adjust the code according to your specific needs for sending SNS notifications.
