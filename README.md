# AWS S3 Automated Backup System
![AutomatedBackUpDiagram](https://github.com/user-attachments/assets/f3e65954-4f85-4c07-a249-53fc58c69373)

An automated backup solution that copies files from a source S3 bucket into three dedicated backup buckets based on folder organization, with scheduled execution and notification alerts.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation](#installation)
  - [1. Create S3 Buckets](#1-create-s3-buckets)
  - [2. Create SNS Topic](#2-create-sns-topic)
  - [3. Create IAM Role](#3-create-iam-role)
  - [4. Deploy Lambda Function](#4-deploy-lambda-function)
  - [5. Configure EventBridge Scheduler](#5-configure-eventbridge-scheduler)
- [Usage](#usage)
- [Configuration](#configuration)
- [Monitoring](#monitoring)
- [Troubleshooting](#troubleshooting)
- [Cost Considerations](#cost-considerations)
- [Security](#security)
- [Contributing](#contributing)
- [License](#license)

---

## Overview

This project implements an automated backup system using AWS serverless services. Files stored in a source S3 bucket are automatically copied to three separate backup buckets based on their folder structure (documents, photos, database). The system runs on a schedule and sends notifications after each backup operation.



**Services Used:**
- **Amazon S3** - Storage for source and backup files
- **AWS Lambda** - Backup execution logic
- **Amazon EventBridge** - Scheduled triggering
- **Amazon SNS** - Notification delivery
- **IAM** - Permission management
- **CloudWatch** - Logging and monitoring

## Features

- ✅ Automated scheduled backups
- ✅ Folder-based routing to separate backup buckets
- ✅ Email/SMS notifications on success or failure
- ✅ S3 versioning enabled for data protection
- ✅ Server-side encryption (SSE-S3)
- ✅ Lifecycle policies for cost optimization
- ✅ CloudWatch logging for audit trails

## Prerequisites

- AWS Account with appropriate permissions
- Basic knowledge of AWS services
- AWS CLI installed and configured (optional)
- Email address or phone number for notifications

## Installation

### 1. Create S3 Buckets

Create four S3 buckets with the following naming pattern:

**Source Bucket:** `your-source-files-bucket`

**Backup Buckets:**
- `your-documents-backup-bucket`
- `your-photos-backup-bucket`
- `your-database-backup-bucket`

**Bucket Configuration:**

| Setting | Value |
|---------|-------|
| Block Public Access | Enabled (all settings) |
| Versioning | Enabled |
| Encryption | SSE-S3 |
| Lifecycle Rule | Move to Standard-IA after 30 days<br>Move to Glacier after 180 days |

**Create folder structure in source bucket:**
```
your-source-files-bucket/
├── documents/
├── photos/
└── database/
```

### 2. Create SNS Topic

1. Navigate to **Amazon SNS** in AWS Console
2. Click **Create topic**
3. Configure:
   - **Type:** Standard
   - **Name:** `backup-status-notifications`
4. Click **Create topic**
5. Create a subscription:
   - **Protocol:** Email or SMS
   - **Endpoint:** Your email address or phone number
6. Confirm the subscription (check your email/SMS)
7. **Save the Topic ARN** for later use

### 3. Create IAM Role

1. Go to **IAM** → **Roles** → **Create role**
2. Select **AWS Service** → **Lambda**
3. Click **Next**
4. Skip the permissions page, click **Next**
5. **Role name:** `lambda-s3-backup-role`
6. Click **Create role**
7. Open the newly created role
8. Click **Add permissions** → **Create inline policy**
9. Switch to **JSON** tab and paste the following policy:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ReadFromSourceBucket",
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetObject"
      ],
      "Resource": [
        "arn:aws:s3:::your-source-files-bucket",
        "arn:aws:s3:::your-source-files-bucket/*"
      ]
    },
    {
      "Sid": "WriteToBackupBuckets",
      "Effect": "Allow",
      "Action": [
        "s3:PutObject"
      ],
      "Resource": [
        "arn:aws:s3:::your-documents-backup-bucket/*",
        "arn:aws:s3:::your-photos-backup-bucket/*",
        "arn:aws:s3:::your-database-backup-bucket/*"
      ]
    },
    {
      "Sid": "PublishToSnsTopic",
      "Effect": "Allow",
      "Action": "sns:Publish",
      "Resource": "arn:aws:sns:REGION:ACCOUNT_ID:backup-status-notifications"
    },
    {
      "Sid": "BasicLambdaLogs",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
    }
  ]
}
```

10. Replace placeholders:
    - `your-source-files-bucket` with your source bucket name
    - `your-documents-backup-bucket` with your documents backup bucket name
    - `your-photos-backup-bucket` with your photos backup bucket name
    - `your-database-backup-bucket` with your database backup bucket name
    - `REGION` with your AWS region (e.g., `us-east-1`)
    - `ACCOUNT_ID` with your AWS account ID

11. Click **Review policy**
12. **Policy name:** `s3-backup-policy`
13. Click **Create policy**

### 4. Deploy Lambda Function

1. Go to **AWS Lambda** → **Create function**
2. Configure:
   - **Function name:** `s3-backup-function`
   - **Runtime:** Python 3.10
   - **Execution role:** Use an existing role → Select `lambda-s3-backup-role`
3. Click **Create function**
4. In the **Code** tab, replace the default code with:

```python
import boto3
import datetime
import os

s3 = boto3.client("s3")
sns = boto3.client("sns")

# Read from environment variables
SOURCE_BUCKET = os.environ.get("SRC_BUCKET")
DEST_DOCUMENTS = os.environ.get("DOC_DEST")
DEST_PHOTOS = os.environ.get("PHOTO_DEST")
DEST_DATABASE = os.environ.get("DB_DEST")
SNS_TOPIC_ARN = os.environ.get("SNS_TOPIC_ARN")

def lambda_handler(event, context):
    try:
        # List all objects in source bucket
        response = s3.list_objects_v2(Bucket=SOURCE_BUCKET)
        
        if "Contents" in response:
            for obj in response["Contents"]:
                key = obj["Key"]
                
                # Decide which destination bucket to write to
                if key.startswith("documents/"):
                    dest_bucket = DEST_DOCUMENTS
                elif key.startswith("photos/"):
                    dest_bucket = DEST_PHOTOS
                elif key.startswith("database/"):
                    dest_bucket = DEST_DATABASE
                else:
                    continue  # skip unrelated files
                
                # Copy the file
                s3.copy_object(
                    Bucket=dest_bucket,
                    Key=key,
                    CopySource={'Bucket': SOURCE_BUCKET, 'Key': key}
                )
        
        # Send SNS success message
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Message=f"Backup completed successfully at {datetime.datetime.now()}."
        )
        
        return {"status": "SUCCESS"}
        
    except Exception as error:
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Message=f"Backup FAILED at {datetime.datetime.now()}.\nError: {str(error)}"
        )
        raise
```

5. Click **Deploy**
6. Go to **Configuration** → **General configuration** → **Edit**
7. Set **Timeout** to `2 minutes`
8. Click **Save**
9. Go to **Configuration** → **Environment variables** → **Edit**
10. Add the following environment variables:

| Key | Value |
|-----|-------|
| `SNS_TOPIC_ARN` | Your SNS topic ARN |
| `SRC_BUCKET` | your-source-files-bucket |
| `DOC_DEST` | your-documents-backup-bucket |
| `PHOTO_DEST` | your-photos-backup-bucket |
| `DB_DEST` | your-database-backup-bucket |

11. Click **Save**

### 5. Configure EventBridge Scheduler

1. Go to **Amazon EventBridge** → **Rules** → **Create rule**
2. Configure:
   - **Name:** `s3-backup-schedule`
   - **Rule type:** Schedule
3. Click **Next**
4. Configure schedule pattern:
   - **For testing:** Select **A schedule that runs at a regular rate** → `10` minutes
   - **For production:** Select **A fine-grained schedule** → `cron(0 2 * * ? *)` (runs daily at 2 AM UTC)
5. Click **Next**
6. Select target:
   - **Target:** AWS service
   - **Select a target:** Lambda function
   - **Function:** `s3-backup-function`
7. Click **Next**
8. Review and click **Create rule**

---

## Usage

### Manual Testing

1. Upload test files to your source bucket:
   ```bash
   aws s3 cp test-doc.pdf s3://your-source-files-bucket/documents/
   aws s3 cp test-photo.jpg s3://your-source-files-bucket/photos/
   aws s3 cp test-db.sql s3://your-source-files-bucket/database/
   ```

2. Manually invoke the Lambda function:
   - Go to Lambda → `s3-backup-function` → **Test** tab
   - Click **Test**

3. Verify:
   - Check backup buckets for copied files
   - Check your email/SMS for notification
   - Review CloudWatch Logs for execution details

### Automatic Backups

Once EventBridge is configured, backups will run automatically on schedule. You'll receive notifications after each run.

## Configuration

### Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `SNS_TOPIC_ARN` | ARN of SNS topic for notifications | `arn:aws:sns:us-east-1:123456789012:backup-status-notifications` |
| `SRC_BUCKET` | Source bucket name | `your-source-files-bucket` |
| `DOC_DEST` | Documents backup bucket | `your-documents-backup-bucket` |
| `PHOTO_DEST` | Photos backup bucket | `your-photos-backup-bucket` |
| `DB_DEST` | Database backup bucket | `your-database-backup-bucket` |

### Schedule Options

| Pattern | Description |
|---------|-------------|
| `rate(10 minutes)` | Every 10 minutes (testing) |
| `rate(1 hour)` | Every hour |
| `cron(0 2 * * ? *)` | Daily at 2 AM UTC |
| `cron(0 0 * * 0 *)` | Weekly on Sunday at midnight UTC |

## Monitoring

### CloudWatch Logs

View execution logs:
1. Go to **CloudWatch** → **Log groups**
2. Find `/aws/lambda/s3-backup-function`
3. View log streams for each execution

### CloudWatch Metrics

Monitor Lambda performance:
- Invocations
- Duration
- Errors
- Throttles

### SNS Notifications

Receive real-time alerts:
- ✅ Success notifications with timestamp
- ❌ Failure notifications with error details

## Troubleshooting

### Common Issues

| Issue | Possible Cause | Solution |
|-------|---------------|----------|
| Lambda timeout | Too many files or slow network | Increase timeout in Lambda configuration |
| Files not copying | Incorrect IAM permissions | Verify IAM policy includes all buckets |
| No SNS notification | Subscription not confirmed | Check email/SMS and confirm subscription |
| Missing files in backup | Wrong folder structure | Ensure files are in `documents/`, `photos/`, or `database/` folders |
| Permission denied errors | Incorrect bucket names in IAM policy | Update IAM policy with correct bucket ARNs |

### Debugging Steps

1. **Check CloudWatch Logs** for error messages
2. **Verify environment variables** in Lambda configuration
3. **Test IAM permissions** using AWS Policy Simulator
4. **Confirm SNS subscription** is active
5. **Check S3 bucket names** match exactly in all configurations

## Cost Considerations

Estimated monthly costs (based on typical usage):

| Service | Usage | Estimated Cost |
|---------|-------|----------------|
| S3 Storage | 100 GB Standard | ~$2.30 |
| Lambda | 720 invocations/month @ 30s each | Free tier |
| SNS | 720 SMS notifications | ~$51.84 |
| EventBridge | 720 events/month | Free |
| Data Transfer | S3 copy within region | Free |

**Cost Optimization Tips:**
- Use lifecycle policies to move old backups to cheaper storage classes
- Consider email instead of SMS for notifications
- Implement incremental backups to reduce data transfer

## Security

### Best Practices

- ✅ **Block Public Access** enabled on all S3 buckets
- ✅ **Encryption at rest** using SSE-S3
- ✅ **Versioning enabled** for data recovery
- ✅ **Least privilege IAM policies**
- ✅ **CloudWatch logging** for audit trails
- ✅ **Private VPC** (optional for enhanced security)

### Recommendations

1. Enable **MFA Delete** on S3 buckets for critical data
2. Use **AWS CloudTrail** to track API calls
3. Set up **CloudWatch Alarms** for failed backups
4. Regularly review and rotate IAM credentials
5. Implement **S3 Object Lock** for compliance requirements

## Contributing

Contributions are welcome! Please follow these steps:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/improvement`)
3. Commit your changes (`git commit -am 'Add new feature'`)
4. Push to the branch (`git push origin feature/improvement`)
5. Create a Pull Request

## License

This project is licensed under the MIT License - see the LICENSE file for details.

---

## Support

For issues, questions, or contributions:
- 📧 Open an issue on GitHub
- 📖 Check AWS documentation
- 💬 Join our community discussions

---

**Built with ❤️ using AWS Serverless Services**# Serverless-Automated-Backup-System
