<!DOCTYPE html>
<html lang="en">

<body>

<h1>Smart Vault: Automated EC2 Backup System on AWS</h1>

<section>
  <h2>Overview</h2>
  <p>When your company system crashes or someone wipes important files, you are not only losing money, but also the trust of your customers. While every company implements backups, this alone is not enough. <strong>Smart Vault</strong> is a fully automated, tag-based EC2 backup solution using Amazon EBS snapshots. It is designed to be smart, cost-effective, and secure. It includes automated snapshot creation, snapshot lifecycle management, cross-region disaster recovery, alerting, and monitoring — all built using AWS native services.</p>
</section>

<br/>
<p align="center">
  <img src="https://imgur.com/tb7fLD7.png" height="80%" width="80%" alt="Architecture Diagram"/>
</p>
<br/>

<section>
  <h2>Objectives</h2>
  <ul>
    <li>Prevent data loss from EC2 failures or deletions</li>
    <li>Automate backup scheduling with AWS EventBridge</li>
    <li>Retain only the necessary number of snapshots using a retention policy</li>
    <li>Enable cross-region disaster recovery by copying snapshots</li>
    <li>Provide visibility via CloudWatch metrics and alarms</li>
    <li>Notify administrators of backup success/failure using SNS</li>
  </ul>
</section>

<section>
  <h2>Architecture Components</h2>
  <table>
    <tr><th>Component</th><th>Purpose</th></tr>
    <tr><td>EC2 + EBS</td><td>Target instances with data to back up</td></tr>
    <tr><td>Tags</td><td>Identifies which instances to include in backup (<code>backup=true</code>)</td></tr>
    <tr><td>Lambda Function</td><td>Creates snapshots, deletes old ones, copies them cross-region</td></tr>
    <tr><td>EventBridge Rule</td><td>Schedules Lambda to run periodically (e.g., every 12 hours)</td></tr>
    <tr><td>SNS Topic</td><td>Sends notifications after backup tasks</td></tr>
    <tr><td>CloudWatch Logs</td><td>Captures logs for debugging and monitoring</td></tr>
    <tr><td>CloudWatch Metric</td><td>Tracks how many snapshots were created per run</td></tr>
    <tr><td>CloudWatch Alarm</td><td>Alerts if no snapshots were created</td></tr>
  </table>
</section>

<section>
  <h2>Deployment Steps</h2>
  <ol>
    <li>Launch EC2 instances and attach EBS volumes</li>
    <li>Add the tag <code>backup=true</code> to instances</li>
    <li>Create an IAM role for Lambda with necessary permissions</li>
    <li>Create an SNS topic and register an email to recieve alerts</li>
    <li>Deploy Lambda function with environment variables</li>
    <li>Create an EventBridge schedule to trigger Lambda once a day</li>
    <li>Set up CloudWatch logs and custom metric tracking for monitoring</li>
    <li>Create CloudWatch alarm for missed backups</li>
    <li>Subscribe to SNS notifications to be notified when snapshots are made or missed</li>
  </ol>
</section>

<section>
  <h2>EC2 Tagging</h2>
  <ul>
    <li><strong>Key</strong>: "backup"</li>
    <li><strong>Value</strong>: "true"</li>
  </ul>
  <p align="center">
  <img src="https://imgur.com/EV0nXVc.png" height="90%" width="90%" alt="EC2 TAGGING"/>
  </p>
</section>

<section>
  <h2>IAM Role Permissions</h2>
  <p>The Lambda execution role includes:</p>
  <ul>
    <li><code>AmazonEC2FullAccess</code></li>
    <li><code>AmazonSNSFullAccess</code></li>
    <li><code>CloudWatchLogsFullAccess</code></li>
    <li><code>AmazonS3FullAccess</code></li>
  </ul>
  <p align="center">
  <img src="https://imgur.com/4pKQGkj.png" height="90%" width="90%" alt="IAM Role"/>
  </p>
</section>

<section>
  <h2>Lambda Function</h2>
  <p>The Lambda function performs:</p>
  <ol>
    <li><strong>EC2 discovery</strong>: Queries for EC2 instances tagged with <code>backup=true</code></li>
    <li><strong>Snapshot creation</strong>: For each volume attached, creates a snapshot and tags it</li>
    <li><strong>Cross-region copy</strong>: Copies the snapshot to a defined second region</li>
    <li><strong>Old snapshot cleanup</strong>: Deletes snapshots older than <code>RETENTION_DAYS</code></li>
    <li><strong>Metric reporting</strong>: Publishes <code>SnapshotsCreated</code> metric to CloudWatch</li>
    <li><strong>Notification</strong>: Sends an SNS alert with a summary</li>
  </ol>

  <h3>Required Environment Variables</h3>
  <table>
    <tr><th>Variable</th><th>Example</th></tr>
    <tr><td><code>SOURCE_REGION</code></td><td><code>ap-southeast-2</code></td></tr>
    <tr><td><code>DEST_REGION</code></td><td><code>ap-southeast-1</code></td></tr>
    <tr><td><code>RETENTION_DAYS</code></td><td><code>7</code></td></tr>
    <tr><td><code>SNS_TOPIC_ARN</code></td><td><code>arn:aws:sns:ap-southeast-2:...</code></td></tr>
  </table>

```Python
import boto3
import datetime
import os

SOURCE_REGION = os.environ.get('SOURCE_REGION', 'ap-southeast-2')
DEST_REGION = os.environ.get('DEST_REGION', 'ap-southeast-1')
RETENTION_DAYS = int(os.environ.get('RETENTION_DAYS', 7))
SNS_TOPIC_ARN = os.environ.get('SNS_TOPIC_ARN')

ec2 = boto3.client('ec2', region_name=SOURCE_REGION)
ec2_dest = boto3.client('ec2', region_name=DEST_REGION)
sns = boto3.client('sns', region_name=SOURCE_REGION)
cloudwatch = boto3.client('cloudwatch', region_name=SOURCE_REGION)

def lambda_handler(event, context):
    now = datetime.datetime.utcnow()
    snapshots_created = 0

    # Find instances tagged for backup
    reservations = ec2.describe_instances(
        Filters=[{'Name': 'tag:backup', 'Values': ['true']}]
    )['Reservations']

    for res in reservations:
        for instance in res['Instances']:
            instance_id = instance['InstanceId']
            for vol in instance.get('BlockDeviceMappings', []):
                volume_id = vol['Ebs']['VolumeId']
                desc = f"Backup-{instance_id}-{now.date()}"
                
                # Create snapshot
                snap = ec2.create_snapshot(
                    VolumeId=volume_id,
                    Description=desc,
                    TagSpecifications=[{
                        'ResourceType': 'snapshot',
                        'Tags': [
                            {'Key': 'InstanceId', 'Value': instance_id},
                            {'Key': 'Date', 'Value': now.strftime('%Y-%m-%d')}
                        ]
                    }]
                )
                snapshot_id = snap['SnapshotId']
                print(f"Created snapshot {snapshot_id} for {volume_id}")
                snapshots_created += 1

                # Copy to destination region
                try:
                    copy = ec2_dest.copy_snapshot(
                        SourceRegion=SOURCE_REGION,
                        SourceSnapshotId=snapshot_id,
                        Description=f"Copied from {SOURCE_REGION} on {now.date()}"
                    )
                    dest_snapshot_id = copy['SnapshotId']
                    ec2_dest.create_tags(
                        Resources=[dest_snapshot_id],
                        Tags=[
                            {'Key': 'CopiedFrom', 'Value': SOURCE_REGION},
                            {'Key': 'OriginalSnapshotId', 'Value': snapshot_id},
                            {'Key': 'Date', 'Value': now.strftime('%Y-%m-%d')}
                        ]
                    )
                    print(f"Copied snapshot to {DEST_REGION} as {dest_snapshot_id}")
                except Exception as e:
                    print(f"Failed to copy {snapshot_id} to {DEST_REGION}: {e}")

    # Cleanup old snapshots
    for snap in ec2.describe_snapshots(OwnerIds=['self'])['Snapshots']:
        if (now - snap['StartTime'].replace(tzinfo=None)).days >= RETENTION_DAYS:
            ec2.delete_snapshot(SnapshotId=snap['SnapshotId'])
            print(f"Deleted old snapshot {snap['SnapshotId']}")

    # SNS alert
    if SNS_TOPIC_ARN:
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject="Smart Vault Backup Done",
            Message=f"{snapshots_created} snapshots created. Cross-region copy complete."
        )

    # CloudWatch metric
    if snapshots_created > 0:
        cloudwatch.put_metric_data(
            Namespace='SmartVault',
            MetricData=[{
                'MetricName': 'SnapshotsCreated',
                'Dimensions': [{'Name': 'Service', 'Value': 'Backup'}],
                'Value': snapshots_created,
                'Unit': 'Count'
            }]
        )

```
</section>

<section>
  <h2>EventBridge Schedule</h2>
  <ul>
    <li><strong>Type</strong>: CRON-based schedule (e.g., every day at 3am)</li>
    <li><strong>Target</strong>: The Lambda function</li>
  </ul>
  <p align="center">
  <img src="https://imgur.com/sbrZ37C.png" height="90%" width="90%" alt=EventBridge Schedule"/>
  </p>
</section>

<section>
  <h2>CloudWatch Monitoring</h2>
  <h3>Log Event from Lamda Function</h3>
  <ul>
    <li>Lambda function started.</li>
    <li>Created a snapshot for volume of <code>SmartVaultEC2Instance1</code>.</li>
    <li>Copied that snapshot to region <code>ap-southeast-1</code>.</li>
    <li>Created a snapshot for volume of <code>SmartVaultEC2Instance2</code>.</li>
    <li>Copied that snapshot to region <code>ap-southeast-1</code>.</li>
    <li>Lambda function completed successfully.</li>
  </ul>
  <p align="center">
  <img src="https://imgur.com/c3nBb2N.png" height="90%" width="90%" alt="log event"/>
  </p>

  <h3>Metrics</h3>
  <ul>
    <li>Metric Creation: Created in Lambda function</li>
    <li>Custom namespace: <code>SmartVault</code></li>
    <li>Metric name: <code>SnapshotsCreated</code></li>
    <li>Dimension: <code>Service=Backup</code></li>
  </ul>

  <h3>Alarms</h3>
  <ul>
    <li><strong>Name</strong>: <code>NoSnapshotsCreated</code></li>
    <li><strong>Condition</strong>: <code>SnapshotsCreated</code> metric is <code>0</code> for 1 period</li>
    <li><strong>Action</strong>: Send SNS alert to the same topic used in Lambda</li>
  </ul>
  <p align="center">
  <img src="https://imgur.com/tM1G7Qk.png" height="90%" width="90%" alt="CloudWatch Alarm"/>
  </p>
</section>

<section>
  <h2>Cross-Region Snapshot Copy</h2>
  <ul>
    <li>Snapshots are copied from <code>SOURCE_REGION</code> to <code>DEST_REGION</code> using <code>ec2.copy_snapshot()</code></li>
    <li>Copied snapshots are tagged with source region, original snapshot ID and date</li>
  </ul>
</section>

<section>
  <h2>Issues I Faced</h2>
  <ul>
    <li>Upon checking the <code>ap-southeast-1</code> region for the copied snapshots, I realised that each snapshot had an "error" status and "0%" Progess.</li>
    <p align="center">
    <img src="https://imgur.com/17i5wL2.png" height="100%" width="90%" alt="Copied Snapshot error"/>
    </p>
    <li>I reinvoked the lambda function and noticed that the snapshots were being copied to ap-southeast-1 within seconds. But this is not an expected behaviour from copying EBS volumes as it should take a few minutes to create an EBS snapshot and copy EBS volumes to another region.</li>
    <li> After some research I realised that the Lambda function is copying snapshots before it had finished created the snapshot in the ap-southeast-2 region. </li>
    <li>
          I implemented a <strong>waiter</strong> in the Python code to wait for the snapshot to complete before copying:
          <pre><code class="language-python">
          # Wait until snapshot is completed before copying
          waiter = ec2.get_waiter('snapshot_completed')
          waiter.wait(SnapshotIds=[snapshot_id])
          </code></pre>
    </li>
    <li> As to my expectations, this fixed the issue and each copied snapshot had a status of "Completed" and progress of "100%" </li>
    <p align="center">
    <img src="https://imgur.com/e6hw9kn.png" height="100%" width="90%" alt="Copied Snapshot error FIXED"/>
    </p>
  </ul>
</section>





</body>
</html>
