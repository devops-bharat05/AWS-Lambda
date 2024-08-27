# S3 Image Compression Lambda Function

This project is an AWS Lambda function that automatically compresses image files when they are uploaded to an S3 bucket (`Upload` bucket) and then stores the compressed images in a different S3 bucket (`Download` bucket). The compressed image retains the original file name, with an added suffix of the current date and time.

## Features

- **Automated Trigger**: The Lambda function is triggered automatically whenever an image file is uploaded to the `Upload` S3 bucket.
- **Image Compression**: Compresses the uploaded image using the `Pillow` library.
- **S3 Integration**: The compressed image is uploaded to the `Download` bucket with the original file name and a timestamp suffix.
- **Scalable and Serverless**: Utilizes AWS Lambda to handle image compression, ensuring a scalable and cost-effective solution.

## Architecture

1. **Upload S3 Bucket**: The bucket where users upload their images.
2. **Lambda Function**: Automatically triggered by an S3 event when a new image is uploaded.
3. **Download S3 Bucket**: The bucket where the compressed image is stored, with a filename that includes the current timestamp.

## File Naming Convention

The compressed image is saved to the `Download` bucket with a file name format:
```
<original_filename>_<yyyyMMdd_HHmmss>.<extension>
```
For example, if the original file is `image.jpg`, and the current time is `2024-08-26 14:30:45`, the compressed file will be saved as:
```
image_20240826_143045.jpg
```

## Prerequisites

Before deploying the Lambda function, ensure you have the following set up:

1. **AWS Account**: You will need an AWS account with access to Lambda and S3 services.
2. **IAM Role**: Create an IAM Role with the necessary permissions to read from the `Upload` S3 bucket and write to the `Download` S3 bucket. The role should have the following policies:
   - `s3:GetObject`
   - `s3:PutObject`
3. **S3 Buckets**:
   - Create an S3 bucket named `Upload` for the source of the image files.
   - Create an S3 bucket named `Download` for storing the compressed image files.
4. **Python Environment**: Lambda function written in Python. Ensure your Lambda runtime environment is set to `Python 3.9` or above.

## Dependencies

This Lambda function uses the following Python packages:

- `boto3`: AWS SDK for Python to interact with AWS services like S3.
- `Pillow`: A Python Imaging Library (PIL fork) to handle image compression.

### Installing Dependencies

Before deploying to AWS Lambda, you need to install the required dependencies and package them together with your Lambda code.

1. Install dependencies in a local directory:
   ```bash
   mkdir package
   pip install Pillow boto3 -t ./package
   ```
   
2. Package your Lambda function:
   ```bash
   cd package
   zip -r ../lambda_function.zip .
   cd ..
   zip -g lambda_function.zip lambda_function.py
   ```

### PIL/Pillow for Lambda

To use the `Pillow` library in AWS Lambda, make sure to package it along with the Lambda code. You need to install `Pillow` in the same directory as the Lambda function file and upload it as a zip file. Alternatively, you can use a Lambda Layer.

## Setup and Deployment

### Step 1: Create Lambda Function

1. **Create a new Lambda function** in the AWS Lambda Console.
   - Runtime: `Python 3.9` (or later).
   - Handler: `lambda_function.lambda_handler`.
   - Execution Role: Use the IAM role with the required S3 permissions.

2. **Upload the Lambda code**:
   - Either upload the zip file you created earlier (`lambda_function.zip`), or paste the `lambda_function.py` file and install dependencies using a Lambda Layer.

### Step 2: S3 Event Notification

1. **Go to your `Upload` S3 bucket** in the AWS Console.
2. **Navigate to the "Properties" tab**.
3. **Scroll down to "Event notifications"** and create a new event notification:
   - Name: `ImageUploadTrigger`.
   - Event Type: `All object create events`.
   - Prefix: (Optional) Set to filter on a particular folder or file type.
   - Destination: `Lambda Function`.
   - Select your newly created Lambda function.

### Step 3: Testing

1. **Upload an image** file (e.g., `.jpg`, `.png`, etc.) to the `Upload` bucket.
2. The Lambda function should trigger, compress the image, and upload the compressed version to the `Download` bucket.
3. Verify the compressed file in the `Download` bucket.

### Step 4: Monitoring

You can monitor the execution of the Lambda function through Amazon CloudWatch. Logs will provide detailed information about the function's execution, including any errors that might occur.

## Code Structure

```
.
├── README.md                # Project documentation
├── lambda_function.py       # Main Lambda function code
└── requirements.txt         # Python dependencies (optional)
```

### `lambda_function.py`

This is the core of the Lambda function, containing the logic to compress the uploaded image and store it in the `Download` S3 bucket.

```python
import boto3
import os
import tempfile
from PIL import Image
from datetime import datetime

s3_client = boto3.client('s3')

def lambda_handler(event, context):
    # Get bucket and file details from the event
    source_bucket = event['Records'][0]['s3']['bucket']['name']
    source_key = event['Records'][0]['s3']['object']['key']
    
    # Define the target bucket (Download)
    target_bucket = 'Download'
    
    # Extract the file extension and base name
    file_name, file_extension = os.path.splitext(source_key)
    
    # Add datetime suffix to the file name
    current_time = datetime.now().strftime('%Y%m%d_%H%M%S')
    target_key = f"{file_name}_{current_time}{file_extension}"
    
    # Temporary file for processing the image
    with tempfile.NamedTemporaryFile() as temp_file:
        # Download the image from the S3 Upload bucket
        s3_client.download_file(source_bucket, source_key, temp_file.name)
        
        # Open the image and compress it
        with Image.open(temp_file.name) as img:
            compressed_temp_file = tempfile.NamedTemporaryFile(delete=False, suffix=file_extension)
            compressed_temp_file.close()
            img.save(compressed_temp_file.name, optimize=True, quality=85)  # Compress with quality 85
            
            # Upload the compressed image to the Download bucket
            s3_client.upload_file(compressed_temp_file.name, target_bucket, target_key)
        
        # Remove temporary file
        os.remove(compressed_temp_file.name)
    
    return {
        'statusCode': 200,
        'body': f"Image {source_key} compressed and uploaded as {target_key} to {target_bucket}."
    }
```

## Customization

You can customize the compression quality and other parameters as per your requirements. For example:
- Change the `quality` parameter in the `img.save()` method to adjust compression levels.
- Add additional logic to handle different image types or filters.

## License

This project is licensed under the MIT License.
