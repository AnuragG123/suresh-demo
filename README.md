 To send emails whenever data has been deleted in your AWS Lambda function, you can integrate the AWS Simple Email Service (SES) into your Python code. Here's how you can modify your code to achieve this:

```python
import boto3
import json
from datetime import datetime, timedelta
from botocore.exceptions import ClientError

def lambda_handler(event, context):
    # Extract bucket name and prefix from the event
    bucket_name = event['bucket']
    prefix = event['prefix']
    older_than_years = event.get('older_than_years', 8)

    deleted_objects = delete_old_folders(bucket_name, prefix, older_than_years)
    
    if deleted_objects:
        send_email_notification(deleted_objects)

def delete_old_folders(bucket_name, prefix, older_than_years):
    s3 = boto3.client('s3')
    deleted_objects = []

    # Calculate cutoff date
    cutoff_date = datetime.now() - timedelta(days=older_than_years*365)

    # List objects in the bucket with the given prefix
    response = s3.list_objects_v2(Bucket=bucket_name, Prefix=prefix)

    # Iterate through objects and delete those older than cutoff date
    if 'Contents' in response:
        for obj in response['Contents']:
            key = obj['Key']
            last_modified = obj['LastModified'].replace(tzinfo=None)

            if last_modified < cutoff_date:
                print(f"Deleting object: {key}")
                s3.delete_object(Bucket=bucket_name, Key=key)
                deleted_objects.append(key)
    
    return deleted_objects

def send_email_notification(deleted_objects):
    SENDER = "your_ses_verified_email@example.com"
    RECIPIENTS = ["anurag@gmail.com", "gouthami@gmail.com"]
    SUBJECT = "Data Deletion Notification"
    CHARSET = "UTF-8"

    # Create email body
    email_body = f"The following objects have been deleted:\n\n"
    for obj in deleted_objects:
        email_body += f"- {obj}\n"

    # Create the email message
    email_message = {
        'Subject': {'Data': SUBJECT, 'Charset': CHARSET},
        'Body': {'Text': {'Data': email_body, 'Charset': CHARSET}}
    }

    # Create a new SES client
    ses_client = boto3.client('ses')

    # Try to send the email
    try:
        # Provide the contents of the email
        response = ses_client.send_email(
            Destination={'ToAddresses': RECIPIENTS},
            Message=email_message,
            Source=SENDER
        )
    # Display an error if something goes wrong
    except ClientError as e:
        print(f"Error sending email: {e.response['Error']['Message']}")
    else:
        print("Email sent successfully")
```

Make sure to replace `"your_ses_verified_email@example.com"` with your SES verified email address. Also, ensure that your Lambda function has the necessary permissions to send emails via SES.

This code will send an email notification to the specified recipients whenever data is deleted from the S3 bucket with details of the deleted objects.
