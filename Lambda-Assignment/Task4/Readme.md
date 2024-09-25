## Assignment 4: Automatic EBS Snapshot and Cleanup Using AWS Lambda and Boto3
- **Objective:**
      To automate the backup process for your EBS volumes and ensure that backups older than a specified retention period are cleaned up to save costs. Automate the creation of snapshots for specified EBS volumes and 
      clean up snapshots older than 30 days.
- **Task:**
     Task: Automate the creation of snapshots for specified EBS volumes and clean up snapshots older than 30 days.

### Steps:

### 1. **EBS Setup**
   - Go to the **EC2 Dashboard** and identify or create the EBS volume you want to back up.
   - Note the **Volume ID** of the EBS volume for use in the Lambda function.

### 2. **Lambda IAM Role**
   - In the **IAM Dashboard**, create a new role for Lambda.
   - Attach the following policies:
     - **AmazonEC2FullAccess** (for simplicity), or use a custom policy with the following actions:
       - `ec2:CreateSnapshot`
       - `ec2:DescribeSnapshots`
       - `ec2:DeleteSnapshot`
       - `ec2:DescribeVolumes`

### 3. **Create the Lambda Function**
   - Go to the **AWS Lambda Dashboard** and create a new function.
   - Select **Python 3.x** as the runtime.
   - Assign the role created in Step 2 to this Lambda function.

### 4. **Lambda Code to Create Snapshots and Clean Up Old Snapshots**

Here’s the code to automate both the creation of snapshots and the cleanup of snapshots older than 30 days:

```python
import json
import boto3
from datetime import datetime, timedelta, timezone

def lambda_handler(event, context):
    # Initialize EC2 client
    ec2 = boto3.client('ec2', region_name='us-west-2')  # Replace 'your-region' with your AWS region
    volume_id = 'vol-0faaf073e338794ac'  				        # Replace with your EBS volume ID
    retention_days = 30  						        # Number of days to keep snapshots
    
    # Step 1: Create a snapshot for the specified volume
    snapshot = ec2.create_snapshot(
        VolumeId=volume_id,
        Description=f"Snapshot of volume {volume_id} created by Lambda"
    )
    snapshot_id = snapshot['SnapshotId']
    print(f"Snapshot {snapshot_id} created for volume {volume_id}")
    
    # Step 2: List and delete snapshots older than retention period
    snapshots = ec2.describe_snapshots(
        Filters=[
            {'Name': 'volume-id', 'Values': [volume_id]},
            {'Name': 'status', 'Values': ['completed']}
        ],
        OwnerIds=['self']
    )
    
    # Calculate the cutoff date for deletion (30 days ago)
    cutoff_date = datetime.now(timezone.utc) - timedelta(days=retention_days)
    
    for snap in snapshots['Snapshots']:
        snapshot_time = snap['StartTime']
        if snapshot_time < cutoff_date:
            snap_id = snap['SnapshotId']
            ec2.delete_snapshot(SnapshotId=snap_id)
            print(f"Deleted snapshot {snap_id} created on {snapshot_time}")
    
    return {
        'statusCode': 200,
        'body': json.dumps(f"Snapshot {snapshot_id} created and old snapshots cleaned up.")
    }

```

### 5. **How the Code Works**
1. **Creating a Snapshot**:  
   The `create_snapshot()` method creates a new snapshot for the specified EBS volume and logs its ID.
   
2. **Listing and Cleaning Up Snapshots**:  
   The `describe_snapshots()` method lists all snapshots for the volume, and snapshots older than 30 days are deleted using the `delete_snapshot()` method.

### 6. **Bonus: Attach an Event Source**
   - You can automate the execution of this Lambda function using **Amazon CloudWatch Events** (or **EventBridge**).
   - Create a **Rule** in CloudWatch that triggers the Lambda function at your desired frequency (e.g., weekly).

### 7. **Manual Invocation**
   - You can test the Lambda function manually by invoking it from the AWS Lambda console.
   - Go to the **EC2 Dashboard** and confirm that the snapshot has been created and old snapshots have been deleted.

### Notes:
- Replace the placeholders like `volume_id` and `region_name` with your actual AWS values.
- Modify the `retention_days` variable if you want to change the retention period.
- Ensure the Lambda function has the necessary permissions to manage EBS snapshots.
