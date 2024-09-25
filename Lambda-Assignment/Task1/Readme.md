#### Assignment 1: Automated Instance Management Using AWS Lambda and Boto3
- **Objective:**
      In this assignment, you will gain hands-on experience with AWS Lambda and Boto3, Amazon's SDK for Python. You will create a Lambda function that will automatically manage EC2 instances based on their tags.
- **Task:**
     Task: You're tasked to automate the stopping and starting of EC2 instances based on tags.

### 1. **EC2 Setup:**
   - **Create Two EC2 Instances:**
     1. Go to the [EC2 dashboard](https://console.aws.amazon.com/ec2).
     2. Launch two t2.micro (or free-tier) EC2 instances in the same region.
     3. Once the instances are running, go to the "Tags" tab for each instance.
     4. Add a tag with the key `Action` and value `Auto-Stop` for one instance.
     5. For the other instance, add a tag with the key `Action` and value `Auto-Start`.

### 2. **Lambda IAM Role:**
   - **Create an IAM Role for Lambda:**
     1. Go to the [IAM dashboard](https://console.aws.amazon.com/iam).
     2. Click "Roles", then "Create role".
     3. Select "Lambda" as the trusted entity.
     4. Attach the following policy:
        - `AmazonEC2FullAccess` (for simplicity, in real-world scenarios, use more restrictive policies).

   - **Finalize the Role:**
     1. Name the role (e.g., `EC2InstanceManagerLambdaRole`).
     2. Click "Create role".

### 3. **Lambda Function:**
   - **Create a Lambda Function:**
     1. Go to the [Lambda dashboard](https://console.aws.amazon.com/lambda).
     2. Click "Create Function".
     3. Choose "Author from scratch", name your function (e.g., `ManageEC2Instances`), and select Python 3.x as the runtime.
     4. Choose the IAM role you created (`EC2InstanceManagerLambdaRole`).

   - **Write the Boto3 Script in Lambda:**
     Below is the code to stop instances tagged `Auto-Stop` and start instances tagged `Auto-Start`:

     ```python
     import boto3

     # Initialize EC2 client
     ec2 = boto3.client('ec2', region_name='us-east-1')

     def lambda_handler(event, context):
         # Find all instances with the tag Action=Auto-Stop
         stop_instances = ec2.describe_instances(
             Filters=[
                 {'Name': 'tag:Action', 'Values': ['Auto-Stop']},
                 {'Name': 'instance-state-name', 'Values': ['running']}
             ]
         )

         # Stop instances tagged as Auto-Stop
         for reservation in stop_instances['Reservations']:
             for instance in reservation['Instances']:
                 instance_id = instance['InstanceId']
                 ec2.stop_instances(InstanceIds=[instance_id])
                 print(f"Stopped instance: {instance_id}")

         # Find all instances with the tag Action=Auto-Start
         start_instances = ec2.describe_instances(
             Filters=[
                 {'Name': 'tag:Action', 'Values': ['Auto-Start']},
                 {'Name': 'instance-state-name', 'Values': ['stopped']}
             ]
         )

         # Start instances tagged as Auto-Start
         for reservation in start_instances['Reservations']:
             for instance in reservation['Instances']:
                 instance_id = instance['InstanceId']
                 ec2.start_instances(InstanceIds=[instance_id])
                 print(f"Started instance: {instance_id}")

         return {
             'statusCode': 200,
             'body': 'Instance management complete.'
         }
     ```

   - **Test the Code:**
     1. The script stops instances with the `Auto-Stop` tag that are running.
     2. The script starts instances with the `Auto-Start` tag that are stopped.
     3. It prints the instance IDs that are affected for logging purposes.

### 4. **Manual Invocation:**
   - **Test the Lambda Function:**
     1. Go to the Lambda dashboard and select your function.
     2. Click "Test" to manually invoke the function.
     3. Set up a test event (you can leave it as the default).
     4. After executing, check the EC2 dashboard to verify that:
        - The instance with the `Auto-Stop` tag is stopped.
        - The instance with the `Auto-Start` tag is started.
