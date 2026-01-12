# AWS SageMaker Logs Pack
----

## About this Pack
This pack is built as a complete SOURCE + DESTINATION solution (identified by the IO suffix). Data collection and delivery happen entirely within the pack's context, eliminating the need to connect it to globally defined Sources and Destinations.

This Pack enables comprehensive ingestion, processing, and routing of AWS SageMaker telemetry into Cribl Stream for normalization, enrichment, and delivery to downstream destinations such as Amazon S3 or Splunk.
It captures two complementary data sources for full visibility into SageMaker usage and security:

* `Invocation Logs (Data Capture)`: Actual prompts, responses, and inference payloads sent to/from SageMaker endpoints.
* `CloudTrail Events`: API-level activity including endpoint creation, model deployment, and administrative actions.

This Pack provides the following benefits:

* Ingests both SageMaker log types from Amazon S3
* Uses IAM AssumeRole for secure, short-lived access from Cribl Cloud
* Supports event-driven SQS-based collection
* Pre-configured routes and pipelines for each log type
* Normalizes events into clean, structured JSON with consistent field naming
* Filters CloudTrail events to SageMaker-only in the pipeline
* Extracts nested prompt/response data from Data Capture format
* Supports routing to Amazon S3, Splunk, or other Cribl-supported destinations

## Use Cases

* **Prompt Injection Detection**: Monitor model inputs for malicious prompt patterns
* **PII/Sensitive Data Monitoring**: Detect sensitive information in prompts or responses
* **Model Usage Analytics**: Track token counts, latency, and usage patterns per endpoint
* **Compliance & Audit**: Maintain records of all model interactions and API activity
* **Cost Attribution**: Correlate invocations with endpoints for chargeback

## Deployment

This section describes all required steps to deploy this Pack, including AWS-side setup and Cribl configuration.

### Prerequisites

* An AWS account with Amazon SageMaker endpoints deployed (or planned)
* Permissions to create IAM roles and attach policies
* Permissions to create and configure S3 buckets and event notifications
* Permissions to create SQS queues
* Permissions to modify SageMaker endpoint configurations
* (Optional) Permissions to configure CloudTrail trails

### Part 1: SageMaker Invocation Logs (Data Capture)

Invocation logs capture the actual model interactions — input payloads (prompts), output payloads (responses), and inference metadata.

### `Step 1.1: Create the S3 Bucket for Invocation Logs (AWS)`

* Create an S3 bucket to store SageMaker Data Capture logs
* Example bucket name: `<YOUR_COMPANY>-sagemaker-invocation-logs`
* Data Capture will write logs under the prefix: `<endpoint-name>/AllTraffic/<year>/<month>/<day>/<hour>/`

```
aws s3 mb s3://<YOUR_COMPANY>-sagemaker-invocation-logs --region <YOUR_REGION>
```

### `Step 1.2: Create the SQS Queue for Invocation Logs (AWS)`

* Create an SQS queue to receive S3 event notifications

```
aws sqs create-queue \
  --queue-name <YOUR_COMPANY>-sagemaker-invocation-notifications \
  --region <YOUR_REGION>
```

### `Step 1.3: Configure the SQS Queue Policy (AWS)`

* Add a policy to allow S3 to send notifications to the queue:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:<REGION>:<AWS_ACCOUNT_ID>:<YOUR_COMPANY>-sagemaker-invocation-notifications",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:::<YOUR_COMPANY>-sagemaker-invocation-logs"
        }
      }
    }
  ]
}
```

### `Step 1.4: Configure S3 Event Notifications (AWS)`

* Navigate to S3 → Your invocation logs bucket → Properties
* Scroll to Event notifications → Create event notification
* Configure:
   * Event name: `invocation-logs-created`
   * Suffix: `.jsonl` (Data Capture writes JSONL files)
   * Event types: `s3:ObjectCreated:*`
   * Destination: SQS queue → Select your invocation notifications queue
* Save

### `Step 1.5: Enable Data Capture on SageMaker Endpoints (AWS)`

Data Capture must be enabled per endpoint. This can be done via Console, CLI, or SDK.

**Option A: Enable via AWS CLI (Recommended)**

* First, get the current endpoint configuration:

```
aws sagemaker describe-endpoint \
  --endpoint-name <YOUR_ENDPOINT_NAME> \
  --region <YOUR_REGION> \
  --query 'EndpointConfigName' \
  --output text
```

* Get the configuration details:

```
aws sagemaker describe-endpoint-config \
  --endpoint-config-name <CONFIG_NAME> \
  --region <YOUR_REGION>
```

* Create a new endpoint configuration with Data Capture enabled:

```
aws sagemaker create-endpoint-config \
  --endpoint-config-name <CONFIG_NAME>-with-data-capture \
  --production-variants '[{
    "VariantName": "AllTraffic",
    "ModelName": "<YOUR_MODEL_NAME>",
    "InitialInstanceCount": 1,
    "InstanceType": "<YOUR_INSTANCE_TYPE>",
    "InitialVariantWeight": 1.0
  }]' \
  --data-capture-config '{
    "EnableCapture": true,
    "InitialSamplingPercentage": 100,
    "DestinationS3Uri": "s3://<YOUR_COMPANY>-sagemaker-invocation-logs/",
    "CaptureOptions": [
      {"CaptureMode": "Input"},
      {"CaptureMode": "Output"}
    ],
    "CaptureContentTypeHeader": {
      "JsonContentTypes": ["application/json"]
    }
  }' \
  --region <YOUR_REGION>
```

* Update the endpoint to use the new configuration:

```
aws sagemaker update-endpoint \
  --endpoint-name <YOUR_ENDPOINT_NAME> \
  --endpoint-config-name <CONFIG_NAME>-with-data-capture \
  --region <YOUR_REGION>
```

**Option B: Enable via Python SDK (boto3)**

```
import boto3

sagemaker = boto3.client('sagemaker', region_name='<YOUR_REGION>')

data_capture_config = {
    'EnableCapture': True,
    'InitialSamplingPercentage': 100,
    'DestinationS3Uri': 's3://<YOUR_COMPANY>-sagemaker-invocation-logs/',
    'CaptureOptions': [
        {'CaptureMode': 'Input'},
        {'CaptureMode': 'Output'}
    ],
    'CaptureContentTypeHeader': {
        'JsonContentTypes': ['application/json']
    }
}

sagemaker.create_endpoint_config(
    EndpointConfigName='<CONFIG_NAME>-with-data-capture',
    ProductionVariants=[{
        'VariantName': 'AllTraffic',
        'ModelName': '<YOUR_MODEL_NAME>',
        'InitialInstanceCount': 1,
        'InstanceType': '<YOUR_INSTANCE_TYPE>',
        'InitialVariantWeight': 1.0
    }],
    DataCaptureConfig=data_capture_config
)

sagemaker.update_endpoint(
    EndpointName='<YOUR_ENDPOINT_NAME>',
    EndpointConfigName='<CONFIG_NAME>-with-data-capture'
)
```

**Data Capture Configuration Options**

* `EnableCapture`: Enable/disable capture — set to `true`
* `InitialSamplingPercentage`: Percentage of requests to capture (1-100) — use `100` for security monitoring
* `DestinationS3Uri`: S3 bucket for captured data — your invocation logs bucket
* `CaptureOptions`: What to capture: Input, Output, or both — use both for full visibility

### `Step 1.6: Verify Data Capture is Working (AWS)`

* Send a test inference request to your endpoint:

```
aws sagemaker-runtime invoke-endpoint \
  --endpoint-name <YOUR_ENDPOINT_NAME> \
  --content-type "application/json" \
  --body fileb://test-request.json \
  --region <YOUR_REGION> \
  response.json
```

* Wait 2-5 minutes, then check S3 for captured data:

```
aws s3 ls s3://<YOUR_COMPANY>-sagemaker-invocation-logs/ --recursive --region <YOUR_REGION>
```

* You should see `.jsonl` files under: `<endpoint-name>/AllTraffic/<year>/<month>/<day>/<hour>/`

### Part 2: CloudTrail Logs (Management Events)

CloudTrail captures API-level activity for security auditing and compliance — endpoint creation, model deployment, configuration changes, and more.

`Note:` CloudTrail management events cannot be filtered by service at the trail level. This Pack filters events to SageMaker-only in the Cribl pipeline.

### `Step 2.1: Create the S3 Bucket for CloudTrail Logs (AWS)`

* Create or identify an S3 bucket to store CloudTrail logs
* Example bucket name: `<YOUR_COMPANY>-sagemaker-cloudtrail-logs`

```
aws s3 mb s3://<YOUR_COMPANY>-sagemaker-cloudtrail-logs --region <YOUR_REGION>
```

### `Step 2.2: Create a CloudTrail Trail (AWS)`

* Navigate to AWS Console → CloudTrail → Create trail
* Trail name: `<YOUR_COMPANY>-sagemaker-cloudtrail`
* Storage location: Select the S3 bucket from Step 2.1
* Management events: Enable (Read + Write)
* Create the trail

Or via CLI:

```
aws cloudtrail create-trail \
  --name <YOUR_COMPANY>-sagemaker-cloudtrail \
  --s3-bucket-name <YOUR_COMPANY>-sagemaker-cloudtrail-logs \
  --is-multi-region-trail \
  --region <YOUR_REGION>

aws cloudtrail start-logging \
  --name <YOUR_COMPANY>-sagemaker-cloudtrail \
  --region <YOUR_REGION>
```

### `Step 2.3: Create the SQS Queue for CloudTrail Logs (AWS)`

```
aws sqs create-queue \
  --queue-name <YOUR_COMPANY>-sagemaker-cloudtrail-notifications \
  --region <YOUR_REGION>
```

### `Step 2.4: Configure the SQS Queue Policy (AWS)`

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "s3.amazonaws.com"
      },
      "Action": "sqs:SendMessage",
      "Resource": "arn:aws:sqs:<REGION>:<AWS_ACCOUNT_ID>:<YOUR_COMPANY>-sagemaker-cloudtrail-notifications",
      "Condition": {
        "ArnLike": {
          "aws:SourceArn": "arn:aws:s3:::<YOUR_COMPANY>-sagemaker-cloudtrail-logs"
        }
      }
    }
  ]
}
```

### `Step 2.5: Configure S3 Event Notifications (AWS)`

* Navigate to S3 → Your CloudTrail logs bucket → Properties
* Scroll to Event notifications → Create event notification
* Configure:
   * Event name: `cloudtrail-logs-created`
   * Prefix: `AWSLogs/` (CloudTrail standard prefix)
   * Suffix: `.json.gz`
   * Event types: `s3:ObjectCreated:*`
   * Destination: SQS queue → Select your CloudTrail notifications queue
* Save

### Part 3: IAM Role Configuration

Create a single IAM role that Cribl will assume for accessing both S3 buckets and SQS queues.

### `Step 3.1: Create the IAM Role (AWS)`

* Navigate to AWS Console → IAM → Roles → Create role
* Trusted entity type: AWS account
* Select "Another AWS account"
* Enter the Cribl Cloud AWS Account ID: `017895164650`
* Role name: `CriblSageMakerLogsReader`

Or via CLI, create a trust policy file (`trust-policy.json`):

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::017895164650:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {}
    }
  ]
}
```

Create the role:

```
aws iam create-role \
  --role-name CriblSageMakerLogsReader \
  --assume-role-policy-document file://trust-policy.json
```

### `Step 3.2: Create and Attach the IAM Policy (AWS)`

* Create a policy file (`sagemaker-logs-policy.json`):

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3BucketList",
      "Effect": "Allow",
      "Action": "s3:ListBucket",
      "Resource": [
        "arn:aws:s3:::<YOUR_COMPANY>-sagemaker-invocation-logs",
        "arn:aws:s3:::<YOUR_COMPANY>-sagemaker-cloudtrail-logs"
      ]
    },
    {
      "Sid": "S3ObjectRead",
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": [
        "arn:aws:s3:::<YOUR_COMPANY>-sagemaker-invocation-logs/*",
        "arn:aws:s3:::<YOUR_COMPANY>-sagemaker-cloudtrail-logs/AWSLogs/*"
      ]
    },
    {
      "Sid": "SQSAccess",
      "Effect": "Allow",
      "Action": [
        "sqs:ReceiveMessage",
        "sqs:DeleteMessage",
        "sqs:GetQueueAttributes",
        "sqs:GetQueueUrl"
      ],
      "Resource": [
        "arn:aws:sqs:<REGION>:<AWS_ACCOUNT_ID>:<YOUR_COMPANY>-sagemaker-invocation-notifications",
        "arn:aws:sqs:<REGION>:<AWS_ACCOUNT_ID>:<YOUR_COMPANY>-sagemaker-cloudtrail-notifications"
      ]
    },
    {
      "Sid": "CloudWatchLogsAccess",
      "Effect": "Allow",
      "Action": [
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams",
        "logs:GetLogEvents",
        "logs:FilterLogEvents"
      ],
      "Resource": "arn:aws:logs:<REGION>:<AWS_ACCOUNT_ID>:log-group:/aws/sagemaker/*"
    }
  ]
}
```

* Create and attach the policy:

```
aws iam create-policy \
  --policy-name SageMakerLogsAccess \
  --policy-document file://sagemaker-logs-policy.json

aws iam attach-role-policy \
  --role-name CriblSageMakerLogsReader \
  --policy-arn arn:aws:iam::<AWS_ACCOUNT_ID>:policy/SageMakerLogsAccess
```

* Copy the Role ARN: `arn:aws:iam::<AWS_ACCOUNT_ID>:role/CriblSageMakerLogsReader`

### Part 4: Cribl Configuration

### `Step 4.1: Configure Pack Variables (Cribl)`

After installing the Pack, configure the following variables:

* `aws_region`: AWS region for S3/SQS — example: `us-east-1`
* `iam_role_arn`: IAM Role ARN for cross-account access — example: `arn:aws:iam::<AWS_ACCOUNT_ID>:role/CriblSageMakerLogsReader`
* `sagemaker_invocation_sqs_queue`: SQS queue URL for invocation logs — example: `https://sqs.<REGION>.amazonaws.com/<AWS_ACCOUNT_ID>/<YOUR_COMPANY>-sagemaker-invocation-notifications`
* `sagemaker_cloudtrail_sqs_queue`: SQS queue URL for CloudTrail logs — example: `https://sqs.<REGION>.amazonaws.com/<AWS_ACCOUNT_ID>/<YOUR_COMPANY>-sagemaker-cloudtrail-notifications`
* `default_splunk_index`: Default Splunk index — defaults to `aws`
* `default_splunk_sourcetype`: Default Splunk sourcetype — defaults to `aws:sagemaker`

### `Step 4.2: Enable the Sources (Cribl)`

* Navigate to Data → Sources within the Pack
* Enable `in_sagemaker_invocation` (S3 source for invocation logs)
* Enable `in_sagemaker_cloudtrail` (S3 source for CloudTrail logs)
* Verify the SQS queue URLs and IAM role ARN are populated from Pack variables

### `Step 4.3: Configure Destination (Cribl)`

* Either use the Default Destination defined by your Worker Group, or:
* Create a new Destination within the Pack
* Update the Pack routes to point to your destination

### `Step 4.4: Commit and Deploy (Cribl)`

Once everything is configured, perform a **Commit & Deploy** to enable data collection.

## Data Format

### Invocation Logs (Data Capture)

Data Capture creates JSONL files with the following structure:

```
{
  "captureData": {
    "endpointInput": {
      "observedContentType": "application/json",
      "mode": "INPUT",
      "data": "{\"inputs\": \"What is machine learning?\", \"parameters\": {\"max_new_tokens\": 50}}",
      "encoding": "JSON"
    },
    "endpointOutput": {
      "observedContentType": "application/json",
      "mode": "OUTPUT",
      "data": "{\"generated_text\": \"Machine learning is a subset of artificial intelligence...\"}",
      "encoding": "JSON"
    }
  },
  "eventMetadata": {
    "eventId": "4b9e37f4-559b-402d-ae08-016ea84b6f7e",
    "inferenceTime": "2026-01-07T22:16:43Z"
  },
  "eventVersion": "0"
}
```

### CloudTrail Events

CloudTrail logs contain standard AWS CloudTrail format with SageMaker-specific fields:

```
{
  "eventVersion": "1.08",
  "eventSource": "sagemaker.amazonaws.com",
  "eventName": "InvokeEndpoint",
  "awsRegion": "us-east-1",
  "sourceIPAddress": "203.0.113.50",
  "userAgent": "boto3/1.26.0",
  "requestParameters": {
    "endpointName": "my-llm-endpoint"
  },
  "responseElements": null,
  "eventTime": "2026-01-07T22:16:43Z"
}
```

## Pipeline Processing

### Invocation Logs Pipeline

The Pack includes a pipeline that:

* Extracts nested JSON from `captureData.endpointInput.data` and `captureData.endpointOutput.data`
* Parses the inference timestamp from `eventMetadata.inferenceTime`
* Adds endpoint name from S3 path metadata
* Normalizes field names for downstream processing

### CloudTrail Pipeline

The Pack includes a pipeline that:

* Filters events to `eventSource == 'sagemaker.amazonaws.com'`
* Extracts key fields: `eventName`, `userIdentity`, `sourceIPAddress`, `requestParameters`
* Parses `eventTime` for proper timestamp handling

## Troubleshooting

### No Invocation Logs Appearing

* Verify Data Capture is enabled on your endpoint:

```
aws sagemaker describe-endpoint-config \
  --endpoint-config-name <CONFIG_NAME> \
  --region <YOUR_REGION> \
  --query 'DataCaptureConfig'
```

* Check S3 bucket for files (allow 2-5 minutes after inference):

```
aws s3 ls s3://<YOUR_COMPANY>-sagemaker-invocation-logs/ --recursive
```

* Verify SQS is receiving notifications:

```
aws sqs get-queue-attributes \
  --queue-url <YOUR_SQS_URL> \
  --attribute-names ApproximateNumberOfMessages
```

### No CloudTrail Events for SageMaker

* Verify CloudTrail is logging:

```
aws cloudtrail get-trail-status --name <YOUR_TRAIL_NAME>
```

* Check S3 bucket for CloudTrail files:

```
aws s3 ls s3://<YOUR_COMPANY>-sagemaker-cloudtrail-logs/AWSLogs/ --recursive
```

* The Pack filters CloudTrail to SageMaker events — ensure you're making SageMaker API calls to generate events

### IAM Permission Errors

* Verify the trust relationship allows Cribl Cloud account `017895164650`:

```
aws iam get-role --role-name CriblSageMakerLogsReader --query 'Role.AssumeRolePolicyDocument'
```

* Verify the policy grants required S3 and SQS permissions:

```
aws iam list-attached-role-policies --role-name CriblSageMakerLogsReader
```

## Cost Considerations

### SageMaker Endpoint Costs

* Endpoints are billed per hour while running
* Example: `ml.g6.xlarge` ≈ $1.20/hour
* **Remember to delete endpoints when not in use**

### Data Capture Storage Costs

* JSONL files stored in S3 — standard S3 pricing applies
* 100% sampling on high-volume endpoints can generate significant data
* Consider reducing `InitialSamplingPercentage` for cost optimization

### Cleanup Commands

```
# Delete SageMaker endpoint (stops hourly charges)
aws sagemaker delete-endpoint \
  --endpoint-name <YOUR_ENDPOINT_NAME> \
  --region <YOUR_REGION>

# Delete endpoint configuration
aws sagemaker delete-endpoint-config \
  --endpoint-config-name <CONFIG_NAME> \
  --region <YOUR_REGION>
```

## Configure your Destination/Update Pack Routes

To ensure proper data routing, you must make a choice: retain the current setting to use the Default Destination defined by your Worker Group, or define a new Destination directly inside this pack and adjust the pack's route accordingly.

## Enable the required inputs

### Commit and Deploy

Once everything is configured, perform a Commit & Deploy to enable data collection.

## Pack Configurable Items

The following are the in-Pack configurable items - review/update them as needed.

The Pack has the following variables:

* `default_splunk_index`: Default index for the Splunk output - defaults to `aws`.
* `default_sagemaker_cloudtrail_sourcetype`: Default sourcetype for the Splunk output - defaults to `aws:sagemaker:cloudtrail`.
* `default_sagemaker_invocation_sourcetype`: Default sourcetype for the Splunk output - defaults to `aws:sagemaker:invocation`.
* `aws_region`: AWS region where your S3 buckets and SQS queues are located - defaults to us-east-1.
* `iam_role_arn`: IAM Role ARN that Cribl will assume for cross-account access to S3 and SQS.
* `sagemaker_cloudtrail_sqs_queue`: SQS queue URL for CloudTrail log notifications.
* `sagemaker_invocation_sqs_queue`: SQS queue URL for SageMaker invocation log notifications.

## References

* [SageMaker Data Capture Documentation](https://docs.aws.amazon.com/sagemaker/latest/dg/model-monitor-data-capture.html)
* [SageMaker CloudTrail Logging](https://docs.aws.amazon.com/sagemaker/latest/dg/logging-using-cloudtrail.html)
* [Cribl S3 Source Documentation](https://docs.cribl.io/stream/sources-s3-sqs/)
* [Cribl Cross-Account Access](https://docs.cribl.io/stream/usecase-aws-x-account/)

## Upgrades

Upgrading certain Cribl Packs using the same Pack ID can have unintended consequences. See [Upgrading an Existing Pack](https://docs.cribl.io/stream/packs#upgrading) for details.

## Release Notes

### Version 1.0.0

* Initial release

## Contributing to the Pack

To contribute to the Pack, please connect with us on [Cribl Community Slack](https://cribl-community.slack.com/). You can suggest new features or offer to collaborate.

## License

This Pack uses the following license: [Apache 2.0](https://github.com/criblio/appscope/blob/master/LICENSE).
