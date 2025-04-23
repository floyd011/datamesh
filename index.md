# README

# ETL with AWS Step Functions

This project implements an end-to-end ETL (Extract, Transform, Load) pipeline using AWS Step Functions and Lambda functions. The workflow is designed to process CSV files from an S3 bucket, transform the data, and store the output back in another S3 bucket. It also includes success and failure notifications via SNS.

## Architecture

- **S3**: Input and output buckets for CSV files
- **Lambda**: Serverless functions for extract, transform, load, and notifications
- **Step Functions**: Coordinates the ETL workflow
- **SNS**: Sends success or failure notifications
- **IAM**: Roles and policies for secure access

## Workflow

1. **Extract**: Downloads CSV from S3 and parses the data.
2. **Transform**: Processes and modifies the data as needed.
3. **Load**: Writes the transformed data to a new CSV in the output S3 bucket.
4. **Notify**: Sends a success or failure message through SNS.

## Deployment

1. Ensure you have Terraform installed (`>= 1.3.0`).
2. Run `terraform init` to initialize the working directory.
3. Run `terraform apply` to deploy all AWS resources.

## Notes

- Replace the `TopicArn` and bucket names in the Lambda code with actual values from your environment.
- Make sure Lambda roles have permissions to read/write S3 and publish to SNS.
- Modify the `region` variable in `variables.tf` if needed.
# ETL Step Functions Project (Terraform + AWS Lambda)

This is a complete end-to-end ETL workflow on AWS using Step Functions, Lambda, S3, SNS, and Terraform for infrastructure as code.

---
## Folder structure
```
etl-step-functions/
├── main.tf
├── iam.tf
├── s3.tf
├── sns.tf
├── step_function.tf
├── variables.tf
├── outputs.tf
├── lambdas/
│   ├── extract/
│   │   └── main.py
│   ├── transform/
│   │   └── main.py
│   ├── load/
│   │   └── main.py
│   ├── notify_success/
│   │   └── main.py
│   └── notify_failure/
│       └── main.py
└── README.md
```

## Terraform Files

### `main.tf`
```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  required_version = ">= 1.3.0"
}

provider "aws" {
  region = var.region
}

resource "random_id" "suffix" {
  byte_length = 4
}```

### `iam.tf`
```hcl
resource "aws_iam_role" "stepfn_role" {
  name = "etl_step_functions_role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Action = "sts:AssumeRole",
      Effect = "Allow",
      Principal = {
        Service = "states.amazonaws.com"
      }
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic_execution" {
  role       = aws_iam_role.stepfn_role.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_role_policy_attachment" "stepfn_policy" {
  role       = aws_iam_role.stepfn_role.name
  policy_arn = "arn:aws:iam::aws:policy/AWSStepFunctionsFullAccess"
}
```


### `s3.tf`
```hcl
resource "aws_s3_bucket" "input_bucket" {
  bucket = "etl-input-bucket-${random_id.suffix.hex}"
  force_destroy = true

  tags = {
    Project = "ETL-StepFunctions"
    Environment = "Production"
  }
}

resource "aws_s3_bucket" "output_bucket" {
  bucket = "etl-output-bucket-${random_id.suffix.hex}"
  force_destroy = true

  tags = {
    Project = "ETL-StepFunctions"
    Environment = "Production"
  }
}
```

### `sns.tf`
```hcl
resource "aws_sns_topic" "success_topic" {
  name = "etl-success-topic"

  tags = {
    Project = "ETL-StepFunctions"
    Environment = "Production"
  }
}

resource "aws_sns_topic" "failure_topic" {
  name = "etl-failure-topic"

  tags = {
    Project = "ETL-StepFunctions"
    Environment = "Production"
  }
}
```

### `step_function.tf`
```hcl
resource "aws_sfn_state_machine" "etl_state_machine" {
  name     = "etl_step_function"
  role_arn = aws_iam_role.stepfn_role.arn

  definition = jsonencode({
    StartAt = "ExtractData",
    States = {
      ExtractData = {
        Type = "Task",
        Resource = aws_lambda_function.extract.arn,
        Next = "TransformData",
        Catch = [{
          ErrorEquals = ["States.ALL"],
          Next = "NotifyFailure"
        }]
      },
      TransformData = {
        Type = "Task",
        Resource = aws_lambda_function.transform.arn,
        Next = "LoadData",
        Catch = [{
          ErrorEquals = ["States.ALL"],
          Next = "NotifyFailure"
        }]
      },
      LoadData = {
        Type = "Task",
        Resource = aws_lambda_function.load.arn,
        Next = "NotifySuccess",
        Catch = [{
          ErrorEquals = ["States.ALL"],
          Next = "NotifyFailure"
        }]
      },
      NotifySuccess = {
        Type = "Task",
        Resource = aws_lambda_function.notify_success.arn,
        End = true
      },
      NotifyFailure = {
        Type = "Task",
        Resource = aws_lambda_function.notify_failure.arn,
        End = true
      }
    }
  })
}
```

### `variables.tf`
```hcl
variable "region" {
  description = "AWS region"
  type        = string
  default     = "us-east-1"
}
```

### `outputs.tf`
```hcl
output "etl_state_machine_arn" {
  value = aws_sfn_state_machine.etl_state_machine.arn
}
```

## Lambda functions

### `lambdas/extract/main.py`
```python
import json
import boto3
import csv
import io

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    bucket = event['bucket']
    key = event['key']
    
    response = s3.get_object(Bucket=bucket, Key=key)
    body = response['Body'].read().decode('utf-8')
    reader = csv.DictReader(io.StringIO(body))
    
    data = [row for row in reader]
    return {'statusCode': 200, 'data': data}
```

### `lambdas/transform/main.py`
```python
def lambda_handler(event, context):
    data = event['data']
    
    # Example transformation: uppercase all 'name' fields
    for row in data:
        if 'name' in row:
            row['name'] = row['name'].upper()
    
    return {'statusCode': 200, 'data': data}
```

### `lambdas/load/main.py`
```python
import json
import boto3
import csv
import io

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    data = event['data']
    
    output = io.StringIO()
    writer = csv.DictWriter(output, fieldnames=data[0].keys())
    writer.writeheader()
    writer.writerows(data)
    
    key = 'transformed_data.csv'
    s3.put_object(Bucket='etl-output-bucket', Key=key, Body=output.getvalue())
    
    return {'statusCode': 200, 'message': f'Data written to {key}'}
```

### `lambdas/notify_success/main.py`
```python
import boto3

def lambda_handler(event, context):
    sns = boto3.client('sns')
    sns.publish(
        TopicArn='arn:aws:sns:region:account-id:etl-success-topic',
        Message='ETL job completed successfully.',
        Subject='ETL Success'
    )
    return {'statusCode': 200, 'message': 'Success notification sent'}
```

### `lambdas/notify_failure/main.py`
```python
import boto3

def lambda_handler(event, context):
    sns = boto3.client('sns')
    sns.publish(
        TopicArn='arn:aws:sns:region:account-id:etl-failure-topic',
        Message='ETL job failed.',
        Subject='ETL Failure'
    )
    return {'statusCode': 200, 'message': 'Failure notification sent'}
```
