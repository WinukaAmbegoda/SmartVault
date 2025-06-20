Smart Vault: Automated EC2 Backup System on AWS

Overview

Smart Vault is a fully automated, tag-based EC2 backup solution using Amazon EBS snapshots. It is designed to be smart, cost-effective, and secure. It includes automated snapshot creation, snapshot lifecycle management, cross-region disaster recovery, alerting, and monitoring — all built using AWS native services.

Objectives

Prevent data loss from EC2 failures or deletions

Automate backup scheduling with AWS EventBridge

Retain only the necessary number of snapshots using a retention policy

Enable cross-region disaster recovery by copying snapshots

Provide visibility via CloudWatch metrics and alarms

Notify administrators of backup success/failure using SNS

Architecture Components

Component

Purpose

EC2 + EBS

Target instances with data to back up

Tags

Identifies which instances to include in backup (backup=true)

Lambda Function

Creates snapshots, deletes old ones, copies them cross-region

EventBridge Rule

Schedules Lambda to run periodically (e.g., every 12 hours)

SNS Topic

Sends notifications after backup tasks

CloudWatch Logs

Captures logs for debugging and monitoring

CloudWatch Metric

Tracks how many snapshots were created per run

CloudWatch Alarm

Alerts if no snapshots were created

Lambda Function

The Lambda function performs:

EC2 discovery: Queries for EC2 instances tagged with backup=true

Snapshot creation: For each volume attached, creates a snapshot and tags it

Cross-region copy: Copies the snapshot to a defined second region

Old snapshot cleanup: Deletes snapshots older than RETENTION_DAYS

Metric reporting: Publishes SnapshotsCreated metric to CloudWatch

Notification: Sends an SNS alert with a summary

Required Environment Variables

Variable

Example

SOURCE_REGION

ap-southeast-2

DEST_REGION

ap-southeast-1

RETENTION_DAYS

7

SNS_TOPIC_ARN

arn:aws:sns:ap-southeast-2:...

EventBridge Rule

Type: Rate-based schedule (e.g., every 12 hours)

Target: The Lambda function

Permissions: IAM role with ec2:*, sns:Publish, cloudwatch:PutMetricData, logs:*

CloudWatch Monitoring

Metrics

Custom namespace: SmartVault

Metric name: SnapshotsCreated

Dimension: Service=Backup

Alarms

Name: NoSnapshotsCreated

Condition: SnapshotsCreated metric is 0 for 1 period

Action: Send SNS alert to the same topic used in Lambda

S3 and Cross-Region Copy

Snapshots are copied from SOURCE_REGION to DEST_REGION using ec2.copy_snapshot()

Copied snapshots are tagged with original snapshot ID and date

Can be extended to use AWS Backup or replicate to S3 via custom scripts

IAM Role Permissions

The Lambda execution role must include:

ec2:DescribeInstances

ec2:CreateSnapshot

ec2:DeleteSnapshot

ec2:DescribeSnapshots

ec2:CreateTags

ec2:CopySnapshot

cloudwatch:PutMetricData

logs:*

sns:Publish

Sample EC2 Tagging

aws ec2 create-tags \
  --resources i-1234567890abcdef0 \
  --tags Key=backup,Value=true

Deployment Steps

Launch EC2 instances and attach EBS volumes

Add the tag backup=true to instances

Create an IAM role with necessary permissions

Deploy Lambda function with environment variables

Create an EventBridge schedule to trigger Lambda

Set up CloudWatch logs and custom metric tracking

Create CloudWatch alarm for missed backups

(Optional) Set up cross-region copy by setting DEST_REGION

(Optional) Subscribe to SNS notifications

Future Enhancements

Store snapshots in S3 Glacier for long-term archival

Support for application-consistent snapshots using SSM

Web dashboard for viewing backup history

Tag-based retention customization (e.g., retention=14)

License

MIT License — free to use and modify.

Author

Built with ❤️ using AWS Lambda, EC2, and EventBridge — 2025

