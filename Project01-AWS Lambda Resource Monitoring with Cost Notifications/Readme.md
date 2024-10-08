# AWS Lambda Resource Monitoring with Cost Notifications

This project implements an AWS Lambda function using Python and Boto3 that monitors EC2 instances and sends notifications if any instance has been running for more than 15 days. The function also calculates the cost of those instances for the last 15 days and includes this information in the notification.

## Problem Statement

Companies often leave resources such as EC2 instances running for long periods of time, resulting in unnecessary costs. There is a need for an automated solution that monitors these resources, calculates the costs they are incurring, and notifies users if their resources have been running for an extended period.

This Lambda function:
- Identifies EC2 instances that are running for more than 15 days.
- Calculates the cost of running these instances over the past 15 days using AWS Cost Explorer.
- Sends a notification via SNS to alert the user about these instances and the associated costs.

## Features

- **EC2 Monitoring**: Automatically identifies EC2 instances that have been running for more than 15 days.
- **Cost Calculation**: Uses AWS Cost Explorer to calculate the cost associated with each instance over the last 15 days.
- **Notifications**: Sends notifications via AWS SNS with details of the EC2 instances, including instance ID, the number of days the instance has been running, and the cost incurred during that time period.

## AWS Services Used

- **Lambda**: For running the automated monitoring script.
- **EC2**: Monitored resources.
- **Cost Explorer**: Used to calculate the cost of resources.
- **SNS (Simple Notification Service)**: Used to send email or SMS notifications about the resources.

## Requirements

- AWS Account with access to the following services:
  - EC2
  - Cost Explorer
  - SNS
  - Lambda
- Python 3.x
- Boto3 (AWS SDK for Python)

### AWS Permissions

The Lambda function will need the following permissions:
- **EC2:** `ec2:DescribeInstances`
- **Cost Explorer:** `ce:GetCostAndUsage`
- **SNS:** `sns:Publish`

Make sure the IAM role associated with your Lambda function has the required permissions.

## Setup and Deployment

### Step 1: Clone the Repository

```bash
git clone https://github.com/yourusername/resource-monitoring-lambda.git
cd resource-monitoring-lambda
```

### Step 2: Install Dependencies

Since Lambda functions are deployed with minimal libraries, ensure that the `boto3` package is included in the deployment package. Install it locally:

```bash
pip install boto3 -t .
```

This will install `boto3` into your project directory.

### Step 3: Deploy to Lambda

1. **Create a Lambda Function**:
   - In the AWS Management Console, create a new Lambda function with Python as the runtime.
   - Upload your deployment package (all files from the project directory, including dependencies).
   
2. **Set up SNS Topic**:
   - Create an SNS Topic in the AWS Management Console.
   - Subscribe to the topic using the desired method (email, SMS, etc.).
   - Note the SNS Topic ARN for use in the Lambda function.

3. **Configure Lambda Environment Variables**:
   Set the following environment variables in the Lambda configuration:
   - `SNS_TOPIC_ARN`: The ARN of the SNS topic you created.

### Step 4: Test the Lambda Function

Trigger the Lambda function manually from the AWS Console or by scheduling it using a CloudWatch event. If any EC2 instances have been running for more than 15 days, a notification will be sent with details about those instances and their cost.

## Code Walkthrough

### lambda_function.py

This script contains the core logic for:
- **Identifying EC2 Instances**: The Lambda function uses `ec2.describe_instances()` to retrieve running instances.
- **Calculating Age**: The age of each instance is calculated based on its launch time.
- **Cost Calculation**: Cost is calculated using `ce.get_cost_and_usage()` from the AWS Cost Explorer service for the last 15 days.
- **Sending Notifications**: If any instance meets the criteria (running for more than 15 days), a notification is sent using SNS.

### Example Notification

The notification sent to the user includes:
- Instance ID
- Number of days the instance has been running
- The total cost incurred over the last 15 days

Sample Message:
```
Instance i-0abcdef1234567890 is 20 days old and has cost the company $15.76 in the last 15 days.
```

## Future Enhancements

- **Additional Resource Types**: Expand the script to monitor other resource types such as RDS, Lambda functions, etc.
- **Cost Alerts**: Add thresholds for cost alerts to notify users if costs exceed a certain limit.
- **Auto-Tagging**: Automatically tag resources that are older than a threshold for better tracking and management.

## Contributing

Feel free to submit issues, fork the repository, and send pull requests! Any contribution is greatly appreciated.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

This `README.md` outlines the purpose of the Lambda function, explains how it works, and provides setup instructions. It also includes future enhancements and information on how to contribute to the project.
