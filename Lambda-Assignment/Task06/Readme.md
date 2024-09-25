## Assignment 6: Monitor and Alert High AWS Billing Using AWS Lambda, Boto3, and SNS
- **Objective:**
      Create an automated alerting mechanism for when your AWS billing exceeds a certain threshold.
- **Task:**
     Set up a Lambda function to check your AWS billing amount daily, and if it exceeds a specified threshold, send an alert via SNS..


### 1. **SNS Setup:**
   - **Create an SNS Topic:**
     1. Go to the [SNS dashboard](https://console.aws.amazon.com/sns).
     2. Click "Create Topic".
     3. Choose "Standard" as the type and name your topic (e.g., `billing-alerts`).
     4. Click "Create topic".

   - **Subscribe to the Topic:**
     1. After creating the topic, click on it and go to the "Subscriptions" tab.
     2. Click "Create subscription".
     3. Choose "Email" as the protocol and enter your email.
     4. Confirm the subscription from the email you receive.

### 2. **Lambda IAM Role:**
   - **Create IAM Role for Lambda:**
     1. Go to the [IAM dashboard](https://console.aws.amazon.com/iam).
     2. Click "Roles", then "Create role".
     3. Select "Lambda" as the trusted entity.
     4. Attach the following policies:
        - `CloudWatchReadOnlyAccess`
        - `AmazonSNSFullAccess`

   - **Finalize the Role:**
     1. Name the role (e.g., `BillingAlertLambdaRole`).
     2. Click "Create role".

### 3. **Lambda Function:**
   - **Create a Lambda Function:**
     1. Go to the [Lambda dashboard](https://console.aws.amazon.com/lambda).
     2. Click "Create Function".
     3. Choose "Author from scratch", name your function (e.g., `BillingAlertFunction`), and select Python 3.x as the runtime.
     4. Choose the IAM role you created (`BillingAlertLambdaRole`).

   - **Write the Boto3 Script in Lambda:**
     Here's a Python example for your Lambda function:

     ```python
     import json
     import boto3
     import os
     from datetime import datetime, timedelta

     # Initialize boto3 clients
     cloudwatch = boto3.client('cloudwatch', region_name='us-east-1')
     sns = boto3.client('sns')

     # Set your threshold and SNS topic ARN
     THRESHOLD = float(os.environ['THRESHOLD'])
     SNS_TOPIC_ARN = os.environ['SNS_TOPIC_ARN']

     def lambda_handler(event, context):
         # Get the current billing metric from CloudWatch
         response = cloudwatch.get_metric_statistics(
             Namespace='AWS/Billing',
             MetricName='EstimatedCharges',
             Dimensions=[
                 {
                     'Name': 'Currency',
                     'Value': 'USD'
                 }
             ],
             StartTime=datetime.utcnow() - timedelta(days=1),
             EndTime=datetime.utcnow(),
             Period=86400,
             Statistics=['Maximum']
         )
         
         # Extract the most recent billing data point
         if response['Datapoints']:
             current_billing = response['Datapoints'][0]['Maximum']
             print(f"Current Billing: {current_billing} USD")
             
             # Compare with threshold
             if current_billing > THRESHOLD:
                 print(f"Billing exceeded threshold of {THRESHOLD} USD")
                 
                 # Send SNS notification
                 sns.publish(
                     TopicArn=SNS_TOPIC_ARN,
                     Message=f"Alert! Your current AWS billing is {current_billing} USD, which exceeds the threshold of {THRESHOLD} USD.",
                     Subject="AWS Billing Alert"
                 )
             else:
                 print(f"Billing is below threshold: {current_billing} USD")
         else:
             print("No billing data available")

         return {
             'statusCode': 200,
             'body': json.dumps('Billing Check Complete')
         }
     ```

   - **Environment Variables (Lambda Console):**
     Set the following environment variables in your Lambda function:
     - `THRESHOLD`: `50`
     - `SNS_TOPIC_ARN`: Your SNS topic ARN (from the SNS dashboard)

### 4. **Event Source (Bonus):**
   - **Create a CloudWatch Event to Trigger the Lambda Function Daily:**
     1. Go to the [CloudWatch dashboard](https://console.aws.amazon.com/cloudwatch).
     2. Navigate to "Rules" under "Events" and click "Create Rule".
     3. Set the Event Source to "Event Source" > "Schedule".
     4. Choose a rate expression like `rate(1 day)` for daily execution.
     5. Set the target to your Lambda function.
     6. Click "Create".

### 5. **Testing:**
   - **Manual Test:**
     You can test the Lambda function manually by going to the Lambda dashboard, selecting your function, and clicking "Test". Ensure you create a test event (the content doesn’t matter for this function).

   - **Receive Email Alerts:**
     If your billing exceeds the set threshold (e.g., $50), you should receive an email alert from SNS.

By following these steps, your AWS billing will be monitored daily, and you will receive notifications if it exceeds the threshold.
