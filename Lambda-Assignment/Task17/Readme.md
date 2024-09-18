## Assignment 17: Restore EC2 Instance from Snapshot
### Objective: Automate the process of creating a new EC2 instance from the latest snapshot using a Lambda function.

### Step-by-Step Solution

### 1. **Create a Lambda Function**
   - Go to the **AWS Lambda** dashboard.
   - Click **Create Function** and select **Author from scratch**.
   - Choose **Python 3.x** as the runtime.
   - Assign a role to the Lambda function that has the necessary EC2 permissions. This role should have `AmazonEC2FullAccess` or a more restrictive policy that allows creating instances, volumes, and describing snapshots.

### 2. **Lambda Code to Fetch the Most Recent Snapshot and Create a New EC2 Instance**

Here is the Python code to accomplish this task:

```python
import json
import boto3
from datetime import datetime

def lambda_handler(event, context):
    # Initialize the EC2 client
    ec2 = boto3.client('ec2', region_name='your-region')  # Replace 'your-region' with your AWS region
    
    # Step 1: Fetch the latest snapshot for the given EC2 instance
    instance_id = 'i-XXXXXXXXXXXXXX'  # Replace with the EC2 instance ID
    response = ec2.describe_snapshots(
        Filters=[
            {'Name': 'description', 'Values': [f'Created by CreateImage for {instance_id}*']},
            {'Name': 'status', 'Values': ['completed']}
        ],
        OwnerIds=['self']
    )
    
    # Sort snapshots by StartTime to find the most recent one
    snapshots = sorted(response['Snapshots'], key=lambda x: x['StartTime'], reverse=True)
    
    if not snapshots:
        print(f"No snapshots found for instance {instance_id}")
        return {
            'statusCode': 400,
            'body': json.dumps(f'No snapshots found for instance {instance_id}')
        }
    
    latest_snapshot = snapshots[0]
    snapshot_id = latest_snapshot['SnapshotId']
    print(f"Latest snapshot found: {snapshot_id}, created on {latest_snapshot['StartTime']}")
    
    # Step 2: Create a new volume from the latest snapshot
    availability_zone = 'your-availability-zone'  # Replace with your Availability Zone
    new_volume = ec2.create_volume(
        SnapshotId=snapshot_id,
        AvailabilityZone=availability_zone,
        VolumeType='gp2'
    )
    
    volume_id = new_volume['VolumeId']
    print(f"New volume {volume_id} created from snapshot {snapshot_id}")
    
    # Wait for the volume to become available
    ec2.get_waiter('volume_available').wait(VolumeIds=[volume_id])
    
    # Step 3: Launch a new EC2 instance and attach the new volume
    new_instance = ec2.run_instances(
        BlockDeviceMappings=[
            {
                'DeviceName': '/dev/sdf',  # Change based on your instance configuration
                'Ebs': {
                    'VolumeId': volume_id,
                    'DeleteOnTermination': True
                }
            }
        ],
        ImageId='ami-XXXXXXXXXXXXXX',  # Replace with a valid AMI ID
        MinCount=1,
        MaxCount=1,
        InstanceType='t2.micro',  # Change the instance type as required
        KeyName='your-key-name',  # Replace with your key pair name
        SecurityGroupIds=['sg-XXXXXXXXXXXXXX'],  # Replace with your security group
        SubnetId='subnet-XXXXXXXXXXXXXX'  # Replace with your subnet
    )
    
    new_instance_id = new_instance['Instances'][0]['InstanceId']
    print(f"New EC2 instance {new_instance_id} created and volume {volume_id} attached")
    
    return {
        'statusCode': 200,
        'body': json.dumps(f'New EC2 instance {new_instance_id} created from snapshot {snapshot_id}')
    }
```

### Key Steps in the Code:
1. **Fetching the Latest Snapshot**:  
   The `describe_snapshots()` function fetches snapshots for the given instance ID and filters them by completion status.
   
2. **Creating a New Volume**:  
   The latest snapshot is used to create a new EBS volume with the `create_volume()` method.

3. **Launching a New EC2 Instance**:  
   A new EC2 instance is launched with the `run_instances()` method, and the new volume is attached to it.

### 3. **Trigger Lambda Manually or on a Schedule**

- **Manual Invocation**: After setting up the Lambda function, you can test it manually using the AWS Lambda console.
  
- **Scheduled Invocation**: To trigger this Lambda function periodically (e.g., daily or weekly), use **CloudWatch Events** to schedule the Lambda function based on your recovery requirements.

### Important Configuration Items:
- Replace placeholders such as:
  - `instance_id`: The ID of the EC2 instance whose snapshot you're restoring.
  - `availability_zone`: The availability zone where the new volume will be created.
  - `ami_id`, `key_name`, `security_group_ids`, `subnet_id`: Update these with the appropriate values for your environment.

This Lambda function will automate the process of restoring an EC2 instance from the latest snapshot. Let me know if you need any further customization or guidance!
