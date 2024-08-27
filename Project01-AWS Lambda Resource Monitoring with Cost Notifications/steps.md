Python script that can be used in an AWS Lambda function to notify users if their created resources are more than 15 days old and provide the cost estimate to the company using Boto3.

```python
import boto3
import datetime
from botocore.exceptions import ClientError

# Initialize AWS clients
ec2_client = boto3.client('ec2')
sns_client = boto3.client('sns')
ce_client = boto3.client('ce')

# Function to calculate the age of an EC2 instance
def calculate_age(launch_time):
    today = datetime.datetime.now(launch_time.tzinfo)
    age = today - launch_time
    return age.days

# Function to get cost of EC2 instances over the last 15 days
def get_cost(instance_id):
    now = datetime.datetime.utcnow()
    start_date = (now - datetime.timedelta(days=15)).strftime('%Y-%m-%d')
    end_date = now.strftime('%Y-%m-%d')

    response = ce_client.get_cost_and_usage(
        TimePeriod={
            'Start': start_date,
            'End': end_date
        },
        Granularity='DAILY',
        Metrics=['UnblendedCost'],
        Filter={
            'Dimensions': {
                'Key': 'INSTANCE_ID',
                'Values': [instance_id]
            }
        }
    )
    
    # Sum up costs from the past 15 days
    total_cost = 0
    for result in response['ResultsByTime']:
        total_cost += float(result['Total']['UnblendedCost']['Amount'])
    
    return total_cost

def lambda_handler(event, context):
    try:
        # Get all running EC2 instances
        response = ec2_client.describe_instances(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
        
        message = ''
        
        for reservation in response['Reservations']:
            for instance in reservation['Instances']:
                instance_id = instance['InstanceId']
                launch_time = instance['LaunchTime']
                
                # Calculate the age of the instance
                age_in_days = calculate_age(launch_time)
                
                if age_in_days > 15:
                    # Get the cost for this instance over the last 15 days
                    cost = get_cost(instance_id)
                    
                    # Prepare a notification message
                    message += f'Instance {instance_id} is {age_in_days} days old and has cost the company ${cost:.2f} in the last 15 days.\n'
        
        # Check if there is a message to send
        if message:
            sns_client.publish(
                TopicArn='arn:aws:sns:your-region:your-account-id:your-sns-topic',
                Subject='EC2 Instances Cost and Age Notification',
                Message=message
            )
            
        return {
            'statusCode': 200,
            'body': 'Notification sent successfully!'
        }
        
    except ClientError as e:
        return {
            'statusCode': 500,
            'body': f'Error: {e}'
        }
```

### Key Points:
- **Boto3**: The AWS SDK for Python is used to interact with AWS services like EC2, Cost Explorer, and SNS.
- **Cost Calculation**: The `get_cost` function uses AWS Cost Explorer to fetch the unblended cost for the past 15 days for each EC2 instance.
- **Resource Age**: Instances older than 15 days are identified, and their age and cost are included in the notification.
- **SNS**: The Simple Notification Service (SNS) is used to send notifications. You need to configure the SNS topic ARN.

### Steps to Deploy:
1. **Create IAM Role** for Lambda with permissions to describe EC2 instances, use Cost Explorer, and publish to SNS.
2. **Create Lambda Function**: Deploy this code in a Lambda function.
3. **Set Up SNS Topic**: Create an SNS topic and subscribe the user (via email, SMS, etc.) to receive notifications.

