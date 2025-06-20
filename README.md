<!DOCTYPE html>
<html lang="en">

<body>

<h1>Smart Vault: Automated EC2 Backup System on AWS</h1>

<section>
  <h2>Overview</h2>
  <p><strong>Smart Vault</strong> is a fully automated, tag-based EC2 backup solution using Amazon EBS snapshots. It is designed to be smart, cost-effective, and secure. It includes automated snapshot creation, snapshot lifecycle management, cross-region disaster recovery, alerting, and monitoring — all built using AWS native services.</p>
</section>

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
</section>

<section>
  <h2>EventBridge Rule</h2>
  <ul>
    <li><strong>Type</strong>: Rate-based schedule (e.g., every 12 hours)</li>
    <li><strong>Target</strong>: The Lambda function</li>
    <li><strong>Permissions</strong>: IAM role with <code>ec2:*</code>, <code>sns:Publish</code>, <code>cloudwatch:PutMetricData</code>, <code>logs:*</code></li>
  </ul>
</section>

<section>
  <h2>CloudWatch Monitoring</h2>
  <h3>Metrics</h3>
  <ul>
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
</section>

<section>
  <h2>S3 and Cross-Region Copy</h2>
  <ul>
    <li>Snapshots are copied from <code>SOURCE_REGION</code> to <code>DEST_REGION</code> using <code>ec2.copy_snapshot()</code></li>
    <li>Copied snapshots are tagged with original snapshot ID and date</li>
    <li>Can be extended to use AWS Backup or replicate to S3 via custom scripts</li>
  </ul>
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
</section>

<section>
  <h2>Deployment Steps</h2>
  <ol>
    <li>Launch EC2 instances and attach EBS volumes</li>
    <li>Add the tag <code>backup=true</code> to instances</li>
    <li>Create an IAM role for Lambda with necessary permissions</li>
    <li>Deploy Lambda function with environment variables</li>
    <li>Create an EventBridge schedule to trigger Lambda once a day</li>
    <li>Set up CloudWatch logs and custom metric tracking for monitoring</li>
    <li>Create CloudWatch alarm for missed backups</li>
    <li>Set up cross-region copy by setting <code>DEST_REGION</code></li>
    <li>Subscribe to SNS notifications to be notified when snapshots are made or missed</li>
  </ol>
</section>


</body>
</html>
