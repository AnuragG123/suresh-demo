Here’s how you can extend your Terraform configuration to include SNS for sending notifications when folders get deleted in your Lambda function. This setup involves creating an SNS topic, adding an SNS policy, and modifying your Lambda function to publish messages to the SNS topic when folders are deleted.

### `provider.tf`

```hcl
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
      version = "~> 3.0"
    }
  }
}

provider "aws" {
  region = "us-east-1"  # Replace with your desired AWS region
}
```

### `variables.tf`

```hcl
variable "lambda_function_name" {
  description = "Name of the Lambda function"
  type        = string
  default     = "my-lambda-function"
}

variable "lambda_source_path" {
  description = "Path to the Lambda function source code directory"
  type        = string
  default     = "path/to/your/lambda/function"
}

variable "lambda_handler" {
  description = "Lambda function handler"
  type        = string
  default     = "lambda_function.lambda_handler"
}

variable "lambda_timeout" {
  description = "Timeout for the Lambda function execution in seconds"
  type        = number
  default     = 300
}

variable "lambda_memory_size" {
  description = "Memory size for the Lambda function in MB"
  type        = number
  default     = 128
}

variable "lambda_environment_variables" {
  description = "Environment variables for the Lambda function"
  type        = map(string)
  default     = {
    LOG_LEVEL = "INFO"
  }
}

variable "sns_topic_name" {
  description = "Name of the SNS topic for notifications"
  type        = string
  default     = "lambda-deletion-notifications"
}

variable "sns_email_subscriptions" {
  description = "List of email addresses to subscribe to the SNS topic"
  type        = list(string)
  default     = ["example@example.com"]
}
```

### `main.tf`

```hcl
# Include the provider configuration from provider.tf

# Define IAM Role for Lambda execution
resource "aws_iam_role" "lambda_role" {
  name = "lambda-execution-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
        Action    = "sts:AssumeRole"
      }
    ]
  })
}

# Define IAM policy for Lambda function
resource "aws_iam_policy" "lambda_policy" {
  name        = "lambda-policy"
  description = "Policy for Lambda function"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect   = "Allow"
        Action   = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "*"
      },
      {
        Effect   = "Allow"
        Action   = [
          "s3:ListBucket",
          "s3:GetObject",
          "s3:DeleteObject"
        ]
        Resource = "*"  # Adjust to specific bucket ARNs if needed
      },
      {
        Effect   = "Allow"
        Action   = [
          "sns:Publish"
        ]
        Resource = "*"
      }
    ]
  })
}

# Attach policy to role
resource "aws_iam_role_policy_attachment" "lambda_attachment" {
  role       = aws_iam_role.lambda_role.name
  policy_arn = aws_iam_policy.lambda_policy.arn
}

# Package Lambda function code
data "archive_file" "lambda_zip" {
  type        = "zip"
  source_dir  = var.lambda_source_path
  output_path = "lambda_function.zip"
}

# Define Lambda function
resource "aws_lambda_function" "my_lambda_function" {
  function_name    = var.lambda_function_name
  filename         = data.archive_file.lambda_zip.output_path
  source_code_hash = data.archive_file.lambda_zip.output_base64sha256
  handler          = var.lambda_handler
  runtime          = "python3.11"
  role             = aws_iam_role.lambda_role.arn
  timeout          = var.lambda_timeout
  memory_size      = var.lambda_memory_size

  environment {
    variables = merge(var.lambda_environment_variables, {
      SNS_TOPIC_ARN = aws_sns_topic.lambda_notifications.arn
    })
  }
}

# Define SNS topic
resource "aws_sns_topic" "lambda_notifications" {
  name = var.sns_topic_name
}

# Subscribe email addresses to SNS topic
resource "aws_sns_topic_subscription" "email_subscriptions" {
  count = length(var.sns_email_subscriptions)
  topic_arn = aws_sns_topic.lambda_notifications.arn
  protocol  = "email"
  endpoint  = element(var.sns_email_subscriptions, count.index)
}

# Optional: Define API Gateway resources and integration
# Define API Gateway integration permissions, etc.
```

### Lambda Function Code Update

Update your Lambda function code to publish messages to the SNS topic when folders are deleted.

```python
import boto3
from datetime import datetime, timedelta
import re
import logging
import os

# Initialize boto3 clients
s3_client = boto3.client('s3')
sns_client = boto3.client('sns')

# Define constants
BUCKET_NAMES = ['anuragwarangal', 'rameshbuckethm']
SNS_TOPIC_ARN = os.environ['SNS_TOPIC_ARN']

# Define suffixes for office, offc, and region_number
OFFICE_SUFFIXES = [str(i).zfill(2) for i in range(1, 29)]  # 01 to 28
OFFC_SUFFIXES = [str(i).zfill(2) for i in range(1, 29)]  # 01 to 28
REGION_SUFFIXES = [str(i).zfill(2) for i in range(1, 29)]  # 01 to 28

# Set up logging
logger = logging.getLogger()
logger.setLevel(logging.INFO)

def get_folders_to_delete(bucket_name, prefix, date_pattern, date_format):
    folders_to_delete = set()
    suffixes = OFFICE_SUFFIXES if 'OFFICE=' in prefix else OFFC_SUFFIXES if 'OFFC=' in prefix else REGION_SUFFIXES
    for suffix in suffixes:
        full_prefix = f"{prefix}{suffix}/"
        logger.info(f"Listing objects in bucket {bucket_name} with prefix {full_prefix}")
        response = s3_client.list_objects_v2(Bucket=bucket_name, Prefix=full_prefix, Delimiter='/')
        if 'CommonPrefixes' in response:
            for common_prefix in response['CommonPrefixes']:
                sub_prefix = common_prefix['Prefix']
                logger.info(f"Checking sub-prefix: {sub_prefix}")
                match = re.search(date_pattern, sub_prefix)
                if match:
                    date_str = match.group(1)
                    try:
                        if date_format == '%Y%m':  # For YR_MO
                            year = date_str[:4]
                            month = date_str[4:6]
                            folder_date = datetime(int(year), int(month), 1)
                        elif date_format == '%Y%m%d':  # For RPT_DT and FILE_DATE
                            year = date_str[:4]
                            month = date_str[4:6]
                            day = date_str[6:8]
                            folder_date = datetime(int(year), int(month), int(day))
                        logger.info(f"Found folder date: {folder_date}")
                        if (datetime.now() - folder_date).days > 8 * 365:  # Older than 8 years
                            logger.info(f"Folder {sub_prefix} is older than 8 years and will be deleted")
                            folders_to_delete.add(sub_prefix)
                    except ValueError:
                        logger.warning(f"Ignoring invalid date format in folder: {sub_prefix}")
    return list(folders_to_delete)

def delete_folders(bucket_name, folders):
    for folder in folders:
        logger.info(f"Deleting contents of folder: {folder}")
        response = s3_client.list_objects_v2(Bucket=bucket_name, Prefix=folder)
        if 'Contents' in response:
            for obj in response['Contents']:
                logger.info(f"Deleting object: {obj['Key']}")
                s3_client.delete_object(Bucket=bucket_name, Key=obj['Key'])
        logger.info(f"Deleting folder: {folder}")
        s3_client.delete_object(Bucket=bucket_name, Key=folder)

def send_email(folders_deleted):
    message = f"The following folders were deleted:\n" + "\n".join(folders_deleted)
    sns_client.publish(
        TopicArn=SNS_TOPIC_ARN,
        Message=message,
        Subject="S3 Folder Deletion Notification"
    )

def lambda_handler(event, context):
    logger.info("Lambda function started")

    # Define patterns and formats for each bucket
    patterns_formats = {
        'anuragwarangal': [
            (r'YR_MO=(\d{6})/', '%Y%m'),     # For YR_MO folders in on533 and onomc
            (r'RPT_DT=(\d{8})/', '%Y%m%d')   # For RPT_DT folders in oneko and onfpm
        ],
        'rameshbuckethm': [
            (r'YR_MO=(\d{6})/', '%Y%m'),     # For YR_MO folders in oness and onest
            (r'RPT_DT=(\d{

8})/', '%Y%m%d'),  # For RPT_DT folders in onfip
            (r'FILE_DATE=(\d{8})/', '%Y%m%d')  # For FILE_DATE folders in onfld
        ]
    }

    # Combine all folders to delete
    folders_to_delete = []
    
    for bucket_name in BUCKET_NAMES:
        if bucket_name == 'anuragwarangal':
            paths = {
                'converted/loss/on533/region_number=': [
                    (r'YR_MO=(\d{6})/', '%Y%m')
                ],
                'converted/loss/onomc/OFFC=': [
                    (r'YR_MO=(\d{6})/', '%Y%m')
                ],
                'converted/premium/oneko/OFFC=': [
                    (r'RPT_DT=(\d{8})/', '%Y%m%d')
                ],
                'converted/premium/onfpm/OFFC=': [
                    (r'RPT_DT=(\d{8})/', '%Y%m%d')
                ]
            }
        elif bucket_name == 'rameshbuckethm':
            paths = {
                'converted/loss/oness/OFFICE=': [
                    (r'YR_MO=(\d{6})/', '%Y%m')
                ],
                'converted/loss/onest/OFFICE=': [
                    (r'YR_MO=(\d{6})/', '%Y%m')
                ],
                'converted/premium/onfip/RPT_DT=': [
                    (r'RPT_DT=(\d{8})/', '%Y%m%d')
                ],
                'converted/premium/onfld/OFFICE=': [
                    (r'FILE_DATE=(\d{8})/', '%Y%m%d')
                ]
            }
        
        for prefix, patterns in paths.items():
            for pattern, date_format in patterns_formats[bucket_name]:
                folders_to_delete += get_folders_to_delete(bucket_name, prefix, pattern, date_format)

    if folders_to_delete:
        for bucket_name in BUCKET_NAMES:
            delete_folders(bucket_name, folders_to_delete)
            logger.info(f"Deleted folders in {bucket_name}: {folders_to_delete}")
        send_email(folders_to_delete)
    else:
        logger.info("No folders to delete")

    return {
        'statusCode': 200,
        'body': 'Lambda function executed successfully.'
    }
```

### Terraform Commands

After setting up these files, initialize Terraform and apply the configuration:

```bash
terraform init
terraform apply
```

This setup includes the creation of an SNS topic, the necessary IAM policies for publishing to SNS, and the Lambda function code update to publish messages to the SNS topic. Adjust the `path/to/your/lambda/function` to the actual path where your Lambda function code resides. Adjust email addresses in `variables.tf` as needed.
