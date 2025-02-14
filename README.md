# CloudFormation Template for IAM User Management

This CloudFormation template creates IAM users, assigns permissions, and configures event-driven automation using AWS services such as Secrets Manager, IAM, SSM Parameter Store, EventBridge, and Lambda.

## Resources Created

### 1. Store Temporary Password in Secrets Manager

-   **Resource Name**: `TemporaryPassword`
-   **Type**: `AWS::SecretsManager::Secret`
-   **Description**: Stores a temporary password for all IAM users.
-   **Properties**:
    -   `Name`: `TempUserPassword`
    -   `GenerateSecretString`:
        -   `SecretStringTemplate`: `{"password": ""}`
        -   `GenerateStringKey`: `password`
        -   `PasswordLength`: 12
        -   `ExcludeCharacters`: `"@/\`

### 2. IAM Group for S3 Read-Only Access

-   **Resource Name**: `S3UserGroup`
-   **Type**: `AWS::IAM::Group`
-   **Description**: IAM group with read-only access to S3.
-   **Properties**:
    -   `GroupName`: `S3UserGroup`
    -   `Policies`:
        -   `PolicyName`: `S3ReadOnlyPolicy`
        -   `PolicyDocument`:
            -   `Version`: `2012-10-17`
            -   `Statement`:
                -   `Effect`: `Allow`
                -   `Action`:
                    -   `s3:ListBucket`
                    -   `s3:GetObject`
                -   `Resource`: `*`

### 3. IAM Group for EC2 Read-Only Access

-   **Resource Name**: `EC2UserGroup`
-   **Type**: `AWS::IAM::Group`
-   **Description**: IAM group with read-only access to EC2.
-   **Properties**:
    -   `GroupName`: `EC2UserGroup`
    -   `Policies`:
        -   `PolicyName`: `EC2ReadOnlyPolicy`
        -   `PolicyDocument`:
            -   `Version`: `2012-10-17`
            -   `Statement`:
                -   `Effect`: `Allow`
                -   `Action`:
                    -   `ec2:DescribeInstances`
                -   `Resource`: `*`

### 4. Create IAM Users

-   **Resource Names**: `EC2User`, `S3User`
-   **Type**: `AWS::IAM::User`
-   **Description**: Creates IAM users for EC2 and S3 with login profiles.
-   **Properties**:
    -   `UserName`: `ec2-user` / `s3-user`
    -   `Groups`: `!Ref EC2UserGroup` / `!Ref S3UserGroup`
    -   `LoginProfile`:
        -   `Password`: `!Sub "{{resolve:secretsmanager:TempUserPassword:SecretString:password}}"`
        -   `PasswordResetRequired`: `true`

### 5. Store User Emails in Parameter Store

-   **Resource Names**: `EC2UserEmail`, `S3UserEmail`
-   **Type**: `AWS::SSM::Parameter`
-   **Description**: Stores the email addresses of IAM users in the Parameter Store.
-   **Properties**:
    -   `Name`: `/user/emails/ec2-user` / `/user/emails/s3-user`
    -   `Type`: `String`
    -   `Value`: `ec2-user@example.com` / `s3-user@example.com`

### 6. EventBridge Rule to Detect IAM User Creation

-   **Resource Name**: `UserCreationEventRule`
-   **Type**: `AWS::Events::Rule`
-   **Description**: Triggers a Lambda function when a new IAM user is created.
-   **Properties**:
    -   `Name`: `IAMUserCreationRule`
    -   `Description`: `Triggers a Lambda function when a new IAM user is created`
    -   `EventPattern`:
        -   `source`: `aws.iam`
        -   `detail-type`: `AWS API Call via CloudTrail`
        -   `detail`:
            -   `eventSource`: `iam.amazonaws.com`
            -   `eventName`: `CreateUser`
    -   `Targets`:
        -   `Arn`: `!GetAtt IAMUserLoggingLambda.Arn`
        -   `Id`: `IAMUserLoggingLambdaTarget`

### 7. Lambda Function to Log Email & Password

-   **Resource Name**: `IAMUserLoggingLambda`
-   **Type**: `AWS::Lambda::Function`
-   **Description**: Logs the email and temporary password of newly created IAM users.
-   **Properties**:

    -   `FunctionName`: `IAMUserLogger`
    -   `Runtime`: `python3.8`
    -   `Handler`: `index.lambda_handler`
    -   `Role`: `!GetAtt IAMUserLoggingLambdaRole.Arn`
    -   `Code`:

        -   `ZipFile`:

            ```python
            import json
            import boto3
            import os

            ssm = boto3.client('ssm')
            secrets_manager = boto3.client('secretsmanager')

            def lambda_handler(event, context):
                username = event['detail']['requestParameters']['userName']

                # Retrieve email from Parameter Store
                param_name = f"/user/emails/{username}"
                email = ssm.get_parameter(Name=param_name)['Parameter']['Value']

                # Retrieve password from Secrets Manager
                secret = secrets_manager.get_secret_value(SecretId="TempUserPassword")
                password = json.loads(secret['SecretString'])['password']

                # Log the credentials
                print(f"New IAM user created: {username}")
                print(f"Email: {email}")
                print(f"Temporary Password: {password}")
            ```

### 8. IAM Role for Lambda Execution

-   **Resource Name**: `IAMUserLoggingLambdaRole`
-   **Type**: `AWS::IAM::Role`
-   **Description**: IAM role for the Lambda function to log email and password.
-   **Properties**:
    -   `RoleName`: `IAMUserLoggerRole`
    -   `AssumeRolePolicyDocument`:
        -   `Version`: `2012-10-17`
        -   `Statement`:
            -   `Effect`: `Allow`
            -   `Principal`:
                -   `Service`: `lambda.amazonaws.com`
            -   `Action`: `sts:AssumeRole`
    -   `Policies`:
        -   `PolicyName`: `IAMUserLoggerPolicy`
        -   `PolicyDocument`:
            -   `Version`: `2012-10-17`
            -   `Statement`:
                -   `Effect`: `Allow`
                    -   `Action`:
                        -   `ssm:GetParameter`
                        -   `secretsmanager:GetSecretValue`
                    -   `Resource`: `*`
                -   `Effect`: `Allow`
                    -   `Action`:
                        -   `logs:CreateLogGroup`
                        -   `logs:CreateLogStream`
                        -   `logs:PutLogEvents`
                    -   `Resource`: `*`

## Outputs

### IAMUserLoggingLambdaARN

-   **Description**: ARN of the Lambda function that logs IAM user credentials.
-   **Value**: `!GetAtt IAMUserLoggingLambda.Arn`

## Usage

To deploy this CloudFormation stack, use the AWS Management Console, AWS CLI, or any other CloudFormation deployment tool. Ensure that you have the necessary permissions to create the resources defined in the template.

```sh
aws cloudformation create-stack --stack-name iam-user-management --template-body file://index.yaml --capabilities CAPABILITY_NAMED_IAM
```
