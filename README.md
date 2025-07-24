# AWS Health Events Analyzer

A serverless solution that analyzes AWS Health events using Amazon Bedrock, categorizes them by risk level, and sends analysis reports via email with Excel attachments.

## Features

- Retrieves AWS Health events directly from the AWS Health API
- Analyzes events using Amazon Bedrock's Claude 3 Sonnet model
- Categorizes events by risk level and impact
- Generates detailed Excel reports with event analysis
- Sends email notifications with reports attached
- Runs automatically on a weekly schedule (every Tuesday at 5 PM UTC)
- Stores reports in a secure S3 bucket with lifecycle management
- Optional external S3 bucket integration for report storage

## Prerequisites

1. AWS CLI installed and configured
2. AWS SAM CLI installed
3. Python 3.9 or later
4. Verified email address in Amazon SES (for sending reports)
5. Access to Amazon Bedrock and the Claude 3 Sonnet model
6. **AWS Business or Enterprise Support plan** (required for AWS Health API access)
7. **AWS Organizations enabled with AWS Health organizational view activated** (for organization-wide health events)
8. (Optional) Existing S3 bucket for storing reports externally

## Deployment Instructions

### Quick Deployment with Guided Setup (Recommended)

```bash
# Clone the repository
git clone https://github.com/mannylee/aws-health-events-analyzer-v2-sg.git
cd aws-health-events-analyzer-v2-sg

# Build the application
sam build

# Deploy with guided prompts
sam deploy --guided
```

During the guided deployment, you'll be prompted for:
- Stack name
- AWS Region
- Required parameters (SenderEmail, RecipientEmails)
- Optional parameters including S3BucketName and S3KeyPrefix for external S3 storage
- Confirmation of IAM role creation

### Manual Deployment

```bash
# Clone the repository
git clone https://github.com/mannylee/aws-health-events-analyzer-v2-sg.git
cd aws-health-events-analyzer-v2-sg

# Build the application
sam build

# Deploy the application
sam deploy --stack-name health-events-analyzer-v2 \
  --capabilities CAPABILITY_IAM \
  --parameter-overrides \
    SenderEmail=your-email@example.com \
    RecipientEmails=recipient1@example.com,recipient2@example.com \
    S3BucketName=your-existing-bucket \
    S3KeyPrefix=health-events
```

## External S3 Bucket Configuration

The solution supports storing reports in an external S3 bucket in addition to the automatically created internal bucket.

### Important Notes on S3 Configuration

1. **S3BucketName** (Optional):
   - If provided, the solution will attempt to upload reports to this bucket
   - The bucket must already exist before deployment
   - The Lambda function will need permissions to write to this bucket
   - Leave empty to use only the internal bucket created by the solution

2. **S3KeyPrefix** (Optional):
   - Defines the folder structure within the S3 bucket
   - Example: `health-events` will store files as `s3://your-bucket/health-events/filename.xlsx`
   - Leave empty to store files at the root of the bucket

3. **Permissions**:
   - The solution automatically creates the necessary IAM permissions when S3BucketName is provided
   - If the bucket is in a different AWS account, additional bucket policy configuration is required

### Example S3 Configuration

```bash
# Deploy with external S3 bucket configuration
sam deploy --guided

# When prompted for S3BucketName, enter your existing bucket name
# When prompted for S3KeyPrefix, enter your desired prefix (e.g., health-events)
```

## Multiple Installations

If you need to deploy multiple instances of this solution (e.g., for different environments or customers), you should modify your `samconfig.toml` file for each installation:

1. Use a unique `stack_name` for each installation
2. Use a unique `s3_prefix` for each installation
3. The internal S3 bucket for report storage is automatically generated with a unique name
4. Optionally specify different external S3 buckets or prefixes for each installation

Example `samconfig.toml` for multiple installations:

```toml
# First installation
version = 0.1
[default]
[default.deploy]
[default.deploy.parameters]
stack_name = "health-events-analyzer-customer1"
s3_bucket = "aws-sam-cli-managed-default-samclisourcebucket-EXAMPLE"
s3_prefix = "health-events-analyzer-customer1"
region = "ap-southeast-1"
confirm_changeset = true
capabilities = "CAPABILITY_IAM"
parameter_overrides = "SenderEmail=\"customer1@example.com\" RecipientEmails=\"recipient1@example.com\" S3BucketName=\"customer1-bucket\" S3KeyPrefix=\"health-reports\""
```

```toml
# Second installation
version = 0.1
[default]
[default.deploy]
[default.deploy.parameters]
stack_name = "health-events-analyzer-customer2"
s3_bucket = "aws-sam-cli-managed-default-samclisourcebucket-EXAMPLE"
s3_prefix = "health-events-analyzer-customer2"
region = "ap-southeast-1"
confirm_changeset = true
capabilities = "CAPABILITY_IAM"
parameter_overrides = "SenderEmail=\"customer2@example.com\" RecipientEmails=\"recipient2@example.com\" S3BucketName=\"customer2-bucket\" S3KeyPrefix=\"health-reports\""
```

**Note:** The `s3_bucket` and `s3_prefix` parameters in the samconfig.toml refer to the SAM deployment bucket, not the bucket where reports are stored. The internal report storage bucket is automatically created with a unique name for each installation, and the external bucket is specified by the `S3BucketName` parameter.

## Configuration Parameters

| Parameter | Description | Default |
|-----------|-------------|---------|
| SenderEmail | Email address to send reports from (must be verified in SES) | - |
| RecipientEmails | Comma-separated list of email recipients | - |
| AnalysisWindowDays | Number of days of historical events to analyze | 8 |
| EventCategories | Optional comma-separated list of event categories to filter | accountNotification |
| ExcludedServices | Optional comma-separated list of services to exclude | - |
| ExcelFilenameTemplate | Template for Excel filenames | AWS_Health_Events_Analysis_{date}_{time}.xlsx |
| ScheduleEnabled | Enable or disable the scheduled execution | true |
| CustomerName | Customer name for report customization | - |
| BedrockTopP | Top-p parameter for Bedrock model (0.0-1.0) | 0.9 |
| S3BucketName | Name of external S3 bucket for report storage (optional) | '' |
| S3KeyPrefix | Prefix for objects in the external S3 bucket (optional) | '' |

## Bedrock Configuration

The Lambda function uses Amazon Bedrock with the following configuration:
- Model: Claude 3.5 Sonnet v1 (`anthropic.claude-3-5-sonnet-20240620-v1:0`)
- Maximum Tokens: 4000
- Temperature: 0.3
- Top-P: 0.9 (configurable)

**Note**: The solution uses Bedrock inference profiles. If deploying in a different region, you may need to update the model ID in the CloudFormation template to match the available inference profiles in that region.

## Architecture

The solution consists of:
1. A Lambda function that retrieves and analyzes AWS Health events
2. An EventBridge rule that triggers the function weekly
3. An internal S3 bucket for storing generated reports
4. Optional external S3 bucket integration
5. IAM roles with least-privilege permissions
6. Integration with Amazon Bedrock for AI analysis
7. Amazon SES for email delivery

## AWS Resources Created

### Lambda Functions
- **HealthEventsAnalyzerFunction** - Main function that analyzes AWS Health events using Bedrock and sends email reports
- **TimestampGeneratorFunction** - Custom resource function to generate unique timestamps for S3 bucket naming

### Lambda Layer
- **DependenciesLayer** - Contains Python dependencies for the main Lambda function

### IAM Role
- **HealthEventsAnalyzerRole** - Execution role with permissions for Bedrock, SES, Health API, Organizations, S3, and CloudWatch

### S3 Bucket
- **ReportsBucket** - Encrypted bucket for temporary report storage with 90-day lifecycle policy

### EventBridge
- **HealthEventsAnalyzerScheduleRule** - Triggers Lambda function every Tuesday at 5 PM UTC

### Lambda Permission
- **HealthEventsAnalyzerPermission** - Allows EventBridge to invoke the main Lambda function

### Custom Resource
- **TimestampGenerator** - CloudFormation custom resource for generating unique S3 bucket names

## Testing

After deployment, you can manually invoke the Lambda function:

```bash
aws lambda invoke --function-name aws-health-events-analyzer-{stack-name} response.json
```

Replace `{stack-name}` with your actual stack name.

## Monitoring

The solution publishes the following CloudWatch metrics:
- TotalEventsProcessed
- CriticalEventsFound
- HighRiskEvents
- MediumRiskEvents
- LowRiskEvents

## Troubleshooting

- **No email received**: Verify SES email sending permissions and check if the sender email is verified
- **Lambda timeout**: Check if you have many events to process; consider filtering by category or increasing the Lambda timeout
- **Bedrock errors**: 
  - Ensure your account has access to the Claude 3 Sonnet model
  - Verify the correct inference profile ID is used for your region
  - Check that inference profiles are enabled in your Bedrock console
- **Missing events**: 
  - Verify your AWS Health API access and check the EventCategories filter
  - **Important**: AWS Health API requires Business or Enterprise Support plan
  - Basic and Developer support plans do not have access to AWS Health API
- **No organization events**: Ensure AWS Health organizational view is enabled in your AWS Organizations management account
- **S3 upload failures**: 
  - If using external S3 bucket: Check if the S3BucketName parameter is correctly specified and the bucket exists
  - If using internal bucket: The solution automatically creates and configures permissions for the internal bucket
  - Verify the Lambda role has permissions to write to external buckets (internal bucket permissions are automatic)
  - Check CloudWatch Logs for specific error messages
  - For cross-account buckets, verify bucket policies allow access
  - **Tip**: Leave S3BucketName empty to use the internal bucket with automatic permissions

## S3 Access Troubleshooting

If you encounter S3 access issues:

1. Verify the S3 bucket exists and is in the same region as the Lambda function
2. Check CloudWatch Logs for specific error messages
3. Ensure the Lambda execution role has the necessary permissions:
   - s3:PutObject
   - s3:GetObject
   - s3:ListBucket
4. For cross-account buckets, add a bucket policy allowing access from the Lambda role

Example bucket policy for cross-account access:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT_ID:role/LAMBDA_ROLE_NAME"
      },
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::YOUR_BUCKET_NAME",
        "arn:aws:s3:::YOUR_BUCKET_NAME/*"
      ]
    }
  ]
}
```

## Security

This solution requires permissions to:
- Call the AWS Health API
- Use Amazon Bedrock
- Send emails via Amazon SES
- Write to S3 buckets
- Write CloudWatch metrics
- Access AWS Organizations data (for organization-wide health events)

All data is processed within your AWS account, and reports are stored in encrypted S3 buckets with a 90-day lifecycle policy.

## License

This project is licensed under the AWS - see the LICENSE file for details.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.
