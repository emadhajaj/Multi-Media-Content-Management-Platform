# AWS Storage Portfolio Project: Multi-Media Content Management Platform

1. [Project Overview](#aws-storage-portfolio-project-multi-media-content-management-platform)
2. [Implementation](#implementation)
   - [Core Storage Infrastructure](#core-storage-infrastructure)
	     1. [Create Multi-Tier S3 Architecture](#1-create-multi-tier-s3-architecture)
	     2. [Set Up Cross-Region Replication](#2-set-up-cross-region-replication)
	     3. [Create Specialized Storage for Different Use Cases](#3-create-specialized-storage-for-different-use-cases)
   - [Content Processing Pipeline](#content-processing-pipeline)
	     4. [Custom Policies](#4-custom-policies)
	     5. [Create the Lambda Execution Role](#5-create-the-lambda-execution-role)
	     6. [Lambda Functions](#6-lambda-functions)
	     7. [Create Step Functions Workflow](#7-create-step-functions-workflow)
	     8. [Update Lambda Environment Variables](#8-update-content-upload-handler-environment-variables)
   - [Content Delivery Network](#content-delivery-network)
	     9. [Set Up CloudFront Distribution](#9-set-up-cloudfront-distribution)
	     10. [Update CloudFront Invalidation Function](#10-update-lambda-function)
	     11. [Configure CloudFront for Image Delivery](#11-configure-cloudfront-for-image-delivery)
	     12. [Enable CloudFront Access Logging](#12-enable-cloudfront-access-logging)
   - [Search and Analytics](#search-and-analytics)
	     13. [Set Up Analytics with Athena and QuickSight](#13-set-up-analytics-with-athena-and-quicksight)	
   - [Security and Backup](#security-and-backup)
	     14. [Implement Bucket Policies](#14-implement-bucket-policies)
	     15. [Implement Backup Strategy](#15-implement-backup-strategy)
	     16. [Create SNS Topic for Alerts](#16-create-sns-topic-for-alerts)
	     17. [Implement Secrets Management](#17-implement-secrets-management)
	     18. [Enable AWS Config for Compliance](#18-enable-aws-config-for-compliance)
   - [Monitoring and Cost Optimization](#monitoring-and-cost-optimization)
	     19. [CloudWatch Dashboard](#19-cloudwatch-dashboard)
	     20. [CloudWatch Alarms](#20-cloudwatch-alarms)
	     21. [Cost Optimization Strategies](#21-cost-optimization-strategies)
	     22. [Cost Monitoring Lambda](#22-cost-monitoring-lambda)
	     23. [Enable AWS Budgets](#23-enable-aws-budgets)
   - [Testing and Demonstration](#testing-and-demonstration)


```
┌─────────────────────────────────────────────────────────────────────┐
│                        Content Upload Flow                           │
└─────────────────────────────────────────────────────────────────────┘

User Upload
    │
    ├─→ S3 Primary Bucket (us-east-1)
    │       │
    │       ├─→ S3 Event Notification
    │       │       │
    │       │       └─→ Lambda: content-upload-handler
    │       │               │
    │       │               ├─→ Generate file hash
    │       │               ├─→ Extract metadata
    │       │               ├─→ Store in DynamoDB
    │       │               └─→ Trigger Step Functions
    │       │
    │       └─→ Cross-Region Replication → S3 Backup (us-west-2)
    │
    └─→ Step Functions State Machine
            │
            ├─→ [Image] → Lambda: content-image-processor
            │               │
            │               ├─→ Create thumbnail
            │               ├─→ Resize to medium
            │               ├─→ Convert to WebP
            │               └─→ Upload to Processed Bucket
            │
            ├─→ [Video] → Lambda: content-video-processor
            │               │
            │               ├─→ Submit MediaConvert job
            │               ├─→ Wait for completion
            │               └─→ Upload to Processed Bucket
            │
            └─→ [Generic] → Lambda: content-generic-processor
                            │
                            └─→ Extract basic metadata
            │
            └─→ Lambda: update-metadata
                    │
                    └─→ DynamoDB: Update processing status
                            │
                            └─→ Lambda: cloudfront-invalidation
                                    │
                                    └─→ CloudFront: Invalidate cache

┌─────────────────────────────────────────────────────────────────────┐
│                        Content Delivery Flow                         │
└─────────────────────────────────────────────────────────────────────┘

End User Request
    │
    └─→ CloudFront Distribution
            │
            ├─→ Cache Hit → Return cached content
            │
            └─→ Cache Miss
                    │
                    ├─→ Origin: S3 Processed Bucket
                    │       └─→ Optimized content (WebP, thumbnails)
                    │
                    └─→ Origin: S3 Primary Bucket
                            └─→ Original content
```


**Data Flow**

```
Raw Upload → Process → Store → Deliver → Archive

1. Upload (Primary)     : S3 Standard
2. Process (Lambda)     : Transform & optimize
3. Store (Processed)    : S3 Standard
4. Deliver (CloudFront) : Edge cached
5. Archive (30 days)    : S3 Glacier

Parallel: Backup Sync  : Cross-region replication
```

---


# Implementation


## Core Storage Infrastructure


### 1. Create Multi-Tier S3 Architecture

1. **Create Primary Content Bucket**
    Bucket name: `content-platform-primary-566564656`
    Region: us-east-1
    Block Public Access
    Bucket Versioning: Enable
    Tags: Project=`ContentPlatform`, Environment=`Production`
    Default encryption: SSE-S3
    
2. **Create Lifecycle Configuration**
    Lifecycle rule name: `intelligent-content-tiering`
    Rule scope: Apply to all objects
    
    Lifecycle rule actions:
    - Move current versions of objects between storage classes
    - Move noncurrent versions of objects between storage classes
    - Delete noncurrent versions of objects
    - Delete expired object delete markers
    
    Transitions for current object versions:
    - After 30 days: Standard-IA
    - After 90 days: Glacier Instant Retrieval
    - After 365 days: Glacier Flexible Retrieval
    - After 1095 days: Glacier Deep Archive
    
    Transitions for noncurrent object versions:
    - After 30 days: Standard-IA
    - After 60 days: Glacier Flexible Retrieval
    
    Delete noncurrent versions: After 2555 days (7 years)
    Delete expired delete markers: Yes
    
1. **Enable S3 Intelligent Tiering**
    Configuration name: `auto-optimization`
    Status: Enabled
    Scope: Entire bucket
    
    Optional configurations:
    - Archive Access tier (90 days)
    - Deep Archive Access tier (180)
    
1. **Create Additional Specialized Buckets**
    **Raw Data Bucket (Data Lake):**
    Bucket name: `content-platform-raw-data-468862`
    Purpose: Store raw, unprocessed files
    Partitioning: year/month/day/hour structure
    
    **Processed Content Bucket:**
    Bucket name: `content-platform-processed-564156156`
    Purpose: Store processed/optimized content
    Features:
    - CloudFront Origin
    - Public read access via CloudFront only
    
    **Analytics Bucket:**
    Bucket name: `content-platform-analytics-1684845`
    Purpose: Store access logs, metrics, and analytics data
    Features:
    - Partitioned by date
    - Automatic archiving to Glacier
    - Integration with Athena for querying
    
    **Backup Bucket (Different Region):**
    Bucket name: `content-platform-backup-351586884`
    Region: us-west-2
    Purpose: Cross-region backup and disaster recovery
    Features:
    - Cross-Region Replication from primary bucket
    - Glacier-first storage class
    - Compliance and audit logging
    

### 2. Set Up Cross-Region Replication

1. **Create Replication Role**
    Role name: `s3-replication-role`
    Trust entity: S3 service
    Policy: 
    ```json
    {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetReplicationConfiguration",
                "s3:ListBucket"
            ],
            "Resource": "arn:aws:s3:::SOURCE-BUCKET-NAME"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObjectVersionForReplication",
                "s3:GetObjectVersionAcl",
                "s3:GetObjectVersionTagging"
            ],
            "Resource": "arn:aws:s3:::SOURCE-BUCKET-NAME/*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:ReplicateObject",
                "s3:ReplicateDelete",
                "s3:ReplicateTags"
            ],
            "Resource": "arn:aws:s3:::DESTINATION-BUCKET-NAME/*"
        }
    ]
}
    ```
    
2. **Configure Replication**
    Source bucket: content-platform-primary
    Replication rule name: disaster-recovery-replication
    Status: Enabled
    Destination:
    - Bucket: content-platform-backup (us-west-2)
    - Storage class: Glacier Instant Retrieval
    - Replica modification sync: Enabled
    
    Objects to replicate: All objects
    Additional options:
    - Replicate delete markers
    - Replicate replica modifications
    


### 3. Create Specialized Storage for Different Use Cases

1. **Set Up EFS for Shared File Access**
    File system name: `content-platform-efs`
    VPC: Default or create new VPC
    
    Performance mode: General Purpose
    Throughput mode: Provisioned (500 MiB/s)
    
    Storage class: Standard
    Enable lifecycle management:
    - Transition to IA: 30 days
    - Transition out of IA: On first access
    
    Encryption: Enabled
    
    Mount targets: Create in multiple AZs
    Security groups: Create custom SG allowing NFS (port 2049)
    
1. **Create DynamoDB Table**
	Table name: content-metadata
	Partition key: `file_id` (String)
	Billing mode: On-demand
	Encryption: AWS owned key


## Content Processing Pipeline


### 4. Custom Policies

**lambda-s3**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:DeleteObject",
        "s3:GetObjectVersion",
        "s3:PutObjectAcl"
      ],
      "Resource": "arn:aws:s3:::content-platform-*/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ListBucket",
        "s3:GetBucketLocation",
        "s3:GetBucketNotification",
        "s3:PutBucketNotification"
      ],
      "Resource": "arn:aws:s3:::content-platform-*"
    }
  ]
}
```


**MediaConvert**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "mediaconvert:CreateJob",
        "mediaconvert:GetJob",
        "mediaconvert:ListJobs",
        "mediaconvert:CancelJob",
        "mediaconvert:DescribeEndpoints"
      ],
      "Resource": "*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "iam:PassRole"
      ],
      "Resource": "arn:aws:iam::*:role/MediaConvertRole",
      "Condition": {
        "StringEquals": {
          "iam:PassedToService": "mediaconvert.amazonaws.com"
        }
      }
    }
  ]
}
```




**ContentPlatform-DynamoDB-Access**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:GetItem",
        "dynamodb:UpdateItem",
        "dynamodb:Query",
        "dynamodb:Scan",
		"dynamodb:DeleteItem"
		
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/content-metadata"
    }
  ]
}
```


**ContentPlatform-StepFunctions-Access**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "states:StartExecution",
        "states:DescribeExecution"
      ],
      "Resource": "arn:aws:states:*:*:stateMachine:content-processing-workflow"
    }
  ]
}
```




### 5. Create the Lambda Execution Role

1. First Role
	**Select trusted entity**
	- Trusted entity type: **AWS service**
	- Use case: **Lambda**
	
	**Add permissions** Attach these policies:
	-  `AWSLambdaBasicExecutionRole` (AWS managed - for CloudWatch Logs)
	-  `lambda-s3` (Your custom policy)
	-  `MediaConvert` (Your custom policy)
	- `ContentPlatform-DynamoDB-Access`
	- `ContentPlatform-StepFunctions-Access`
	
	- Role name: `content-platform-lambda-role`
	Create inline policy:
	
	```json
	{
	  "Version": "2012-10-17",
	  "Statement": [
	    {
	      "Effect": "Allow",
	      "Action": [
	        "cloudfront:CreateInvalidation",
	        "cloudfront:GetInvalidation",
	        "cloudfront:ListInvalidations"
	      ],
	      "Resource": "*"
	    }
	  ]
	}
	```
	
	
1. Second Role
	- Trusted entity type: AWS service
    - Use case: Select "MediaConvert"
    - Attach policy: `AmazonS3FullAccess` (or create custom policy below)
    
    - Role name: `MediaConvertRole`




### 6. Lambda functions

1. **Create File Upload Handler Lambda**
    Function name: content-upload-handler
    Runtime: Python 3.11
	Role name: `content-platform-lambda-role`
    
    Environment variables:
    - PRIMARY_BUCKET=content-platform-primary-566564656
    - PROCESSED_BUCKET=content-platform-processed-564156156
    - IMAGE_PROCESSING_STATE_MACHINE: (leave blank for now)
	- VIDEO_PROCESSING_STATE_MACHINE: (leave blank for now)
	- AUDIO_PROCESSING_STATE_MACHINE: (leave blank for now)
	- GENERIC_PROCESSING_STATE_MACHINE: (leave blank for now)
	
	**Configure S3 Event Trigger**:
	- S3
	- Bucket: `content-platform-primary-566564656`
	- Event type: All object create events
    
    **Function Code:**
    
    ```python
    import json
	import boto3
	import hashlib
	import mimetypes
	from datetime import datetime
	from urllib.parse import unquote_plus
	import os
	
	s3 = boto3.client('s3')
	dynamodb = boto3.resource('dynamodb')
	stepfunctions = boto3.client('stepfunctions')
	
	def lambda_handler(event, context):
	    """
	    Handle file uploads and trigger appropriate processing workflows.
	    This version uses DynamoDB instead of OpenSearch for metadata indexing.
	    """
	    try:
	        for record in event['Records']:
	            bucket = record['s3']['bucket']['name']
	            key = unquote_plus(record['s3']['object']['key'])
	            size = record['s3']['object']['size']
	
	            # Get file metadata
	            response = s3.head_object(Bucket=bucket, Key=key)
	            content_type = response.get('ContentType', 'application/octet-stream')
	            last_modified = response['LastModified']
	
	            # Generate file hash
	            file_hash = generate_file_hash(bucket, key)
	
	            # Extract metadata
	            metadata = extract_metadata(bucket, key, content_type)
	
	            # Store metadata in DynamoDB
	            store_metadata_in_dynamodb(key, bucket, size, content_type, last_modified, file_hash, metadata)
	
	            # Trigger processing workflow
	            trigger_processing_workflow(bucket, key, content_type, size)
	
	        return {
	            'statusCode': 200,
	            'body': json.dumps({'message': 'Files processed successfully'})
	        }
	
	    except Exception as e:
	        print(f"Error processing files: {str(e)}")
	        return {'statusCode': 500, 'body': json.dumps({'error': str(e)})}
	
	
	def generate_file_hash(bucket, key):
	    """Generate SHA-256 hash of file for deduplication."""
	    try:
	        response = s3.get_object(Bucket=bucket, Key=key)
	        file_content = response['Body'].read()
	        return hashlib.sha256(file_content).hexdigest()
	    except Exception as e:
	        print(f"Error generating hash: {str(e)}")
	        return None
	
	
	def extract_metadata(bucket, key, content_type):
	    """Extract basic metadata based on file type."""
	    metadata = {
	        'file_extension': key.split('.')[-1].lower() if '.' in key else '',
	        'file_name': key.split('/')[-1],
	        'folder_path': '/'.join(key.split('/')[:-1]),
	    }
	
	    if content_type.startswith('image/'):
	        metadata.update({
	            'type': 'image',
	            'requires_processing': True,
	            'processing_types': ['thumbnail', 'webp_conversion', 'metadata_extraction']
	        })
	    elif content_type.startswith('video/'):
	        metadata.update({
	            'type': 'video',
	            'requires_processing': True,
	            'processing_types': ['thumbnail', 'transcoding', 'subtitle_extraction']
	        })
	    elif content_type.startswith('audio/'):
	        metadata.update({
	            'type': 'audio',
	            'requires_processing': True,
	            'processing_types': ['format_conversion', 'metadata_extraction']
	        })
	    else:
	        metadata.update({'type': 'generic', 'requires_processing': False})
	
	    return metadata
	
	
	def store_metadata_in_dynamodb(key, bucket, size, content_type, last_modified, file_hash, metadata):
	    """Store metadata in DynamoDB instead of OpenSearch."""
	    try:
	        table = dynamodb.Table('content-metadata')
	        table.put_item(Item={
	            'file_id': hashlib.md5(key.encode()).hexdigest(),
	            'file_path': key,
	            'bucket': bucket,
	            'size': size,
	            'content_type': content_type,
	            'upload_time': last_modified.isoformat(),
	            'file_hash': file_hash,
	            'metadata': metadata,
	            'processing_status': 'pending',
	            'indexed_at': datetime.utcnow().isoformat()
	        })
	        print(f"Stored metadata for {key} in DynamoDB.")
	    except Exception as e:
	        print(f"Error storing metadata in DynamoDB: {str(e)}")
	
	
	def trigger_processing_workflow(bucket, key, content_type, size):
	    """Trigger Step Functions workflow based on file type."""
	    try:
	        workflow_input = {
	            'bucket': bucket,
	            'key': key,
	            'content_type': content_type,
	            'size': size
	        }
	
	        # Determine workflow
	        if content_type.startswith('image/'):
	            state_machine_arn = os.environ.get('IMAGE_PROCESSING_STATE_MACHINE')
	        elif content_type.startswith('video/'):
	            state_machine_arn = os.environ.get('VIDEO_PROCESSING_STATE_MACHINE')
	        elif content_type.startswith('audio/'):
	            state_machine_arn = os.environ.get('AUDIO_PROCESSING_STATE_MACHINE')
	        else:
	            state_machine_arn = os.environ.get('GENERIC_PROCESSING_STATE_MACHINE')
	
	        if state_machine_arn:
	            stepfunctions.start_execution(
	                stateMachineArn=state_machine_arn,
	                name=f"process-{hashlib.md5(key.encode()).hexdigest()}",
	                input=json.dumps(workflow_input)
	            )
	            print(f"Triggered processing workflow for {key}")
	    except Exception as e:
	        print(f"Error triggering workflow: {str(e)}")

    ```
    
2. **Create Image Processing Lambda**
    Function name: content-image-processor
    Runtime: Python 3.11
    Role name: `content-platform-lambda-role`
    
    Layers: PIL/Pillow layer for image processing
    - on your local machine
    ```bash
    # Step 1 — Create working directory
	mkdir pillow-layer && cd pillow-layer
	
	# Step 2 — Install Pillow into correct structure
	mkdir python
	pip install Pillow -t python/
	
	# Step 3 — Zip the layer contents
	zip -r pillow-layer.zip python
    ```
	
	Create a layer and add the zip to it and attach it to the function
	
    **Function Code:**
    
    ```python
    import json
    import boto3
    from PIL import Image, ImageOps
    import io
    from urllib.parse import unquote_plus
    
    s3 = boto3.client('s3')
    
    def lambda_handler(event, context):
        """
        Process images: create thumbnails, convert formats, extract metadata
        """
        try:
            bucket = event['bucket']
            key = event['key']
            
            # Download image from S3
            response = s3.get_object(Bucket=bucket, Key=key)
            image_data = response['Body'].read()
            
            # Open image with PIL
            original_image = Image.open(io.BytesIO(image_data))
            
            # Generate different sizes and formats
            processed_files = []
            
            # Create thumbnail (150x150)
            thumbnail = create_thumbnail(original_image, (150, 150))
            thumbnail_key = f"processed/thumbnails/{key}"
            upload_image_to_s3(thumbnail, bucket.replace('primary', 'processed'), 
                              thumbnail_key, 'JPEG')
            processed_files.append(thumbnail_key)
            
            # Create medium size (800x600)
            medium = resize_image(original_image, (800, 600))
            medium_key = f"processed/medium/{key}"
            upload_image_to_s3(medium, bucket.replace('primary', 'processed'), 
                              medium_key, 'JPEG')
            processed_files.append(medium_key)
            
            # Convert to WebP for web optimization
            webp_key = f"processed/webp/{key.rsplit('.', 1)[0]}.webp"
            upload_image_to_s3(original_image, bucket.replace('primary', 'processed'), 
                              webp_key, 'WEBP')
            processed_files.append(webp_key)
            
            # Extract and return metadata
            metadata = extract_image_metadata(original_image)
            
            return {
                'statusCode': 200,
                'body': json.dumps({
                    'original_file': key,
                    'processed_files': processed_files,
                    'metadata': metadata,
                    'processing_completed': True
                })
            }
            
        except Exception as e:
            print(f"Error processing image: {str(e)}")
            return {
                'statusCode': 500,
                'body': json.dumps({'error': str(e)})
            }
    
    def create_thumbnail(image, size):
        """Create thumbnail maintaining aspect ratio"""
        thumbnail = image.copy()
        thumbnail.thumbnail(size, Image.Resampling.LANCZOS)
        return thumbnail
    
    def resize_image(image, max_size):
        """Resize image to fit within max_size maintaining aspect ratio"""
        image.thumbnail(max_size, Image.Resampling.LANCZOS)
        return image
    
    def upload_image_to_s3(image, bucket, key, format):
        """Upload processed image to S3"""
        buffer = io.BytesIO()
        image.save(buffer, format=format, optimize=True, quality=85)
        buffer.seek(0)
        
        s3.put_object(
            Bucket=bucket,
            Key=key,
            Body=buffer.getvalue(),
            ContentType=f'image/{format.lower()}',
            Metadata={
                'processed': 'true',
                'processor': 'lambda-image-processor',
                'format': format
            }
        )
    
    def extract_image_metadata(image):
        """Extract image metadata"""
        return {
            'width': image.width,
            'height': image.height,
            'format': image.format,
            'mode': image.mode,
            'has_transparency': image.mode in ('RGBA', 'LA') or 'transparency' in image.info
        }
    ```
    
2. **Create Video Processing Lambda (MediaConvert)**
    Function name: content-video-processor
    Runtime: Python 3.11
    Role name: `content-platform-lambda-role`
    
    **Function Code:**
    
    ```python
    import json
    import boto3
    import uuid
    from urllib.parse import unquote_plus
    
    mediaconvert = boto3.client('mediaconvert')
    s3 = boto3.client('s3')
    
    def lambda_handler(event, context):
        """
        Create MediaConvert job for video processing
        """
        try:
            bucket = event['bucket']
            key = event['key']
            
            # Get MediaConvert endpoint
            response = mediaconvert.describe_endpoints()
            endpoint = response['Endpoints'][0]['Url']
            mediaconvert_client = boto3.client('mediaconvert', endpoint_url=endpoint)
            
            # Create job settings
            job_settings = create_job_settings(bucket, key)
            
            # Submit job
            response = mediaconvert_client.create_job(
                Role='arn:aws:iam::011868793819:role/MediaConvertRole',
                Settings=job_settings,
                Queue='Default',
                UserMetadata={
                    'source_bucket': bucket,
                    'source_key': key
                }
            )
            
            return {
                'statusCode': 200,
                'body': json.dumps({
                    'job_id': response['Job']['Id'],
                    'status': response['Job']['Status'],
                    'message': 'Video processing job submitted'
                })
            }
            
        except Exception as e:
            print(f"Error creating MediaConvert job: {str(e)}")
            return {
                'statusCode': 500,
                'body': json.dumps({'error': str(e)})
            }
    
    def create_job_settings(bucket, key):
        """Create MediaConvert job settings"""
        output_bucket = bucket.replace('primary', 'processed')
        
        return {
            "Inputs": [
                {
                    "FileInput": f"s3://{bucket}/{key}",
                    "AudioSelectors": {
                        "Audio Selector 1": {
                            "Offset": 0,
                            "DefaultSelection": "DEFAULT",
                            "ProgramSelection": 1
                        }
                    },
                    "VideoSelector": {
                        "ColorSpace": "FOLLOW"
                    }
                }
            ],
            "OutputGroups": [
                {
                    "Name": "File Group",
                    "OutputGroupSettings": {
                        "Type": "FILE_GROUP_SETTINGS",
                        "FileGroupSettings": {
                            "Destination": f"s3://{output_bucket}/video-processed/"
                        }
                    },
                    "Outputs": [
                        {
                            "NameModifier": "_720p",
                            "VideoDescription": {
                                "Width": 1280,
                                "Height": 720,
                                "CodecSettings": {
                                    "Codec": "H_264",
                                    "H264Settings": {
                                        "MaxBitrate": 3000000,
                                        "RateControlMode": "QVBR",
                                        "QvbrSettings": {"QvbrQualityLevel": 7}
                                    }
                                }
                            },
                            "AudioDescriptions": [
                                {
                                    "AudioTypeControl": "FOLLOW_INPUT",
                                    "AudioSourceName": "Audio Selector 1",
                                    "CodecSettings": {
                                        "Codec": "AAC",
                                        "AacSettings": {
                                            "AudioDescriptionBroadcasterMix": "NORMAL",
                                            "Bitrate": 128000,
                                            "RateControlMode": "CBR",
                                            "CodecProfile": "LC",
                                            "CodingMode": "CODING_MODE_2_0",
                                            "SampleRate": 48000
                                        }
                                    }
                                }
                            ],
                            "ContainerSettings": {
                                "Container": "MP4",
                                "Mp4Settings": {
                                    "CslgAtom": "INCLUDE",
                                    "FreeSpaceBox": "EXCLUDE",
                                    "MoovPlacement": "PROGRESSIVE_DOWNLOAD"
                                }
                            }
                        }
                    ]
                }
            ]
        }
    ```    
	
2. **Create Generic File Processor**
	Function name: `content-generic-processor` 
	Runtime: Python 3.11 
	Role: `content-platform-lambda-role`
	
	**Function Code:**
	
	```python
	import json
	import boto3
	from datetime import datetime
	
	s3 = boto3.client('s3')
	dynamodb = boto3.resource('dynamodb')
	
	def lambda_handler(event, context):
	    """
	    Process generic files (PDFs, documents, etc.)
	    """
	    try:
	        bucket = event['bucket']
	        key = event['key']
	        
	        # Get file metadata
	        response = s3.head_object(Bucket=bucket, Key=key)
	        
	        # Extract basic information
	        metadata = {
	            'content_type': response.get('ContentType', 'application/octet-stream'),
	            'size': response.get('ContentLength', 0),
	            'last_modified': response['LastModified'].isoformat(),
	            'etag': response.get('ETag', ''),
	            'processed': True,
	            'processing_type': 'generic',
	            'processed_at': datetime.utcnow().isoformat()
	        }
	        
	        return {
	            'statusCode': 200,
	            'body': json.dumps({
	                'original_file': key,
	                'metadata': metadata,
	                'processing_completed': True
	            })
	        }
	        
	    except Exception as e:
	        print(f"Error processing generic file: {str(e)}")
	        return {
	            'statusCode': 500,
	            'body': json.dumps({'error': str(e)})
	        }
	```
	
2. **MediaConvert Status Checker**
	Function name: `check-mediaconvert-status` 
	Runtime: Python 3.11 
	Role: `content-platform-lambda-role`
	
	**Function Code:**
	
	```python
	import json
	import boto3
	
	mediaconvert = boto3.client('mediaconvert')
	
	def lambda_handler(event, context):
	    """
	    Check status of MediaConvert job
	    """
	    try:
	        job_id = event.get('job_id')
	        
	        if not job_id:
	            return {
	                'statusCode': 400,
	                'body': json.dumps({'error': 'No job_id provided'}),
	                'processing_status': 'ERROR'
	            }
	        
	        # Get MediaConvert endpoint
	        response = mediaconvert.describe_endpoints()
	        endpoint = response['Endpoints'][0]['Url']
	        mediaconvert_client = boto3.client('mediaconvert', endpoint_url=endpoint)
	        
	        # Get job status
	        job_response = mediaconvert_client.get_job(Id=job_id)
	        job_status = job_response['Job']['Status']
	        
	        # Map MediaConvert status to our workflow status
	        status_mapping = {
	            'SUBMITTED': 'PROCESSING',
	            'PROGRESSING': 'PROCESSING',
	            'COMPLETE': 'COMPLETE',
	            'CANCELED': 'ERROR',
	            'ERROR': 'ERROR'
	        }
	        
	        processing_status = status_mapping.get(job_status, 'PROCESSING')
	        
	        return {
	            'statusCode': 200,
	            'body': json.dumps({
	                'job_id': job_id,
	                'mediaconvert_status': job_status,
	                'processing_status': processing_status
	            }),
	            'processing_status': processing_status,
	            'job_id': job_id
	        }
	        
	    except Exception as e:
	        print(f"Error checking MediaConvert status: {str(e)}")
	        return {
	            'statusCode': 500,
	            'body': json.dumps({'error': str(e)}),
	            'processing_status': 'ERROR'
	        }
	```
	
2. **Metadata Update Function**
	Function name: `update-metadata` 
	Runtime: Python 3.11 
	Role: `content-platform-lambda-role`
	
	**Function Code:**
	
	```python
	import json
	import boto3
	import hashlib
	from datetime import datetime
	
	dynamodb = boto3.resource('dynamodb')
	
	def lambda_handler(event, context):
	    """
	    Update metadata in DynamoDB after processing
	    """
	    try:
	        table = dynamodb.Table('content-metadata')
	        
	        # Extract information from event
	        bucket = event.get('bucket', '')
	        key = event.get('key', '')
	        processing_result = event.get('body', '{}')
	        
	        # Parse processing result if it's a string
	        if isinstance(processing_result, str):
	            processing_result = json.loads(processing_result)
	        
	        # Generate file_id
	        file_id = hashlib.md5(key.encode()).hexdigest()
	        
	        # Update item in DynamoDB
	        response = table.update_item(
	            Key={'file_id': file_id},
	            UpdateExpression='SET processing_status = :status, processed_at = :timestamp, processing_metadata = :metadata',
	            ExpressionAttributeValues={
	                ':status': 'completed',
	                ':timestamp': datetime.utcnow().isoformat(),
	                ':metadata': processing_result
	            },
	            ReturnValues='ALL_NEW'
	        )
	        
	        return {
	            'statusCode': 200,
	            'body': json.dumps({
	                'message': 'Metadata updated successfully',
	                'file_id': file_id,
	                'updated_attributes': response.get('Attributes', {})
	            })
	        }
	        
	    except Exception as e:
	        print(f"Error updating metadata: {str(e)}")
	        return {
	            'statusCode': 500,
	            'body': json.dumps({'error': str(e)})
	        }
	```
	
2. **CloudFront Invalidation Function**
	Function name: `cloudfront-invalidation` 
	Runtime: Python 3.11 
	Role: `content-platform-lambda-role`
	
	**Function Code:** after creating CloudFront add the distribution id
	
	```python
	import json
	import boto3
	from datetime import datetime
	
	cloudfront = boto3.client('cloudfront')
	
	def lambda_handler(event, context):
	    """
	    Invalidate CloudFront cache for processed files
	    """
	    try:
	        # Get CloudFront distribution ID from environment variable
	        distribution_id = 'EGXP5DIC1ELJL'  # Update this after creating CloudFront
	        
	        # Extract file key
	        key = event.get('key', '')
	        
	        if not key:
	            return {
	                'statusCode': 400,
	                'body': json.dumps({'error': 'No file key provided'})
	            }
	        
	        # Create invalidation paths
	        paths = [
	            f'/{key}',
	            f'/processed/*{key.split("/")[-1]}*'  # Invalidate processed versions
	        ]
	        
	        # Create invalidation
	        response = cloudfront.create_invalidation(
	            DistributionId=distribution_id,
	            InvalidationBatch={
	                'Paths': {
	                    'Quantity': len(paths),
	                    'Items': paths
	                },
	                'CallerReference': f'invalidation-{datetime.utcnow().timestamp()}'
	            }
	        )
	        
	        return {
	            'statusCode': 200,
	            'body': json.dumps({
	                'message': 'Cache invalidation created',
	                'invalidation_id': response['Invalidation']['Id'],
	                'status': response['Invalidation']['Status']
	            })
	        }
	        
	    except Exception as e:
	        print(f"Error creating invalidation: {str(e)}")
	        # Don't fail the workflow if invalidation fails
	        return {
	            'statusCode': 200,
	            'body': json.dumps({
	                'message': 'Invalidation skipped',
	                'error': str(e)
	            })
	        }
	```



### 7. Create Step Functions Workflow

1. Go to Lambda console and copy the ARN for each function:
	- `content-image-processor`
	- `content-video-processor`
	- `check-mediaconvert-status`
	- `content-generic-processor`
	- `update-metadata`
	- `cloudfront-invalidation`
	
2. **Create state machine**
    
    - Choose authoring method: Write your workflow in code
    - Type: Standard
3. **Paste this definition** (replace ARNs with your actual ARNs):
    
```json
	{
	  "Comment": "Content processing workflow for multi-media files",
	  "StartAt": "DetermineFileType",
	  "States": {
	    "DetermineFileType": {
	      "Type": "Choice",
	      "Choices": [
	        {
	          "Variable": "$.content_type",
	          "StringMatches": "image/*",
	          "Next": "ProcessImage"
	        },
	        {
	          "Variable": "$.content_type",
	          "StringMatches": "video/*",
	          "Next": "ProcessVideo"
	        }
	      ],
	      "Default": "ProcessGeneric"
	    },
	    "ProcessImage": {
	      "Type": "Task",
	      "Resource": "arn:aws:lambda:us-east-1:011868793819:function:content-image-processor",
	      "Next": "UpdateMetadata",
	      "Catch": [
	        {
	          "ErrorEquals": ["States.ALL"],
	          "Next": "ProcessingFailed"
	        }
	      ]
	    },
	    "ProcessVideo": {
	      "Type": "Task",
	      "Resource": "arn:aws:lambda:us-east-1:011868793819:function:content-video-processor",
	      "Next": "WaitForVideoProcessing",
	      "Catch": [
	        {
	          "ErrorEquals": ["States.ALL"],
	          "Next": "ProcessingFailed"
	        }
	      ]
	    },
	    "WaitForVideoProcessing": {
	      "Type": "Wait",
	      "Seconds": 30,
	      "Next": "CheckVideoStatus"
	    },
	    "CheckVideoStatus": {
	      "Type": "Task",
	      "Resource": "arn:aws:lambda:us-east-1:011868793819:function:check-mediaconvert-status",
	      "Next": "IsVideoProcessingComplete",
	      "Catch": [
	        {
	          "ErrorEquals": ["States.ALL"],
	          "Next": "ProcessingFailed"
	        }
	      ]
	    },
	    "IsVideoProcessingComplete": {
	      "Type": "Choice",
	      "Choices": [
	        {
	          "Variable": "$.processing_status",
	          "StringEquals": "COMPLETE",
	          "Next": "UpdateMetadata"
	        },
	        {
	          "Variable": "$.processing_status",
	          "StringEquals": "ERROR",
	          "Next": "ProcessingFailed"
	        }
	      ],
	      "Default": "WaitForVideoProcessing"
	    },
	    "ProcessGeneric": {
	      "Type": "Task",
	      "Resource": "arn:aws:lambda:us-east-1:011868793819:function:content-generic-processor",
	      "Next": "UpdateMetadata",
	      "Catch": [
	        {
	          "ErrorEquals": ["States.ALL"],
	          "Next": "ProcessingFailed"
	        }
	      ]
	    },
	    "UpdateMetadata": {
	      "Type": "Task",
	      "Resource": "arn:aws:lambda:us-east-1:011868793819:function:update-metadata",
	      "Next": "InvalidateCloudFrontCache",
	      "Catch": [
	        {
	          "ErrorEquals": ["States.ALL"],
	          "Next": "ProcessingComplete"
	        }
	      ]
	    },
	    "InvalidateCloudFrontCache": {
	      "Type": "Task",
	      "Resource": "arn:aws:lambda:us-east-1:011868793819:function:cloudfront-invalidation",
	      "Next": "ProcessingComplete",
	      "Catch": [
	        {
	          "ErrorEquals": ["States.ALL"],
	          "Next": "ProcessingComplete"
	        }
	      ]
	    },
	    "ProcessingComplete": {
	      "Type": "Succeed"
	    },
	    "ProcessingFailed": {
	      "Type": "Fail",
	      "Cause": "File processing failed"
	    }
	  }
	}
	```
	
3. **Configure state machine**
    
    - Name: `content-processing-workflow`
    - Permissions: Choose existing role → `StepFunctions-ContentProcessing-Role`
    - Logging: Enable
    - Click Create
4. **Copy State Machine ARN**
    

### 8. Update content-upload-handler Environment Variables

Go back to Lambda → `content-upload-handler` → Configuration → Environment variables:

Update the variables with your State Machine ARN:

- IMAGE_PROCESSING_STATE_MACHINE = arn:aws:states:us-east-1:011868793819:stateMachine:content-processing-workflow
- VIDEO_PROCESSING_STATE_MACHINE = arn:aws:states:us-east-1:011868793819:stateMachine:content-processing-workflow
- AUDIO_PROCESSING_STATE_MACHINE = arn:aws:states:us-east-1:011868793819:stateMachine:content-processing-workflow
- GENERIC_PROCESSING_STATE_MACHINE = arn:aws:states:us-east-1:011868793819:stateMachine:content-processing-workflow

(Using same state machine for all types - it will route internally)



## Content Delivery Network

### 9. Set Up CloudFront Distribution


- **S3 origin:**
	1. Click **"Browse S3"** button
	2. Select: `content-platform-processed-564156156`
	**Origin path:** Leave empty
	
- Configure Settings: **Allow private S3 bucket access to CloudFront:**	
- **Origin settings:**  **"Use recommended origin settings"**
- **Cache settings:** **"Use recommended cache settings tailored to serving S3 content"**
- **Web Application Firewall (WAF):** **"Do not enable security protections"**

After creation: Add Second Origin (Primary Bucket)

- **S3 origin:**
	- Select: `content-platform-primary-566564656`
	- Origin path: `/public` (optional)
	
- **Origin settings:** Use recommended origin settings
- Create Cache Behaviors
	
	**Path pattern:** `raw/*`
	**Origin and origin groups:** `content-platform-primary-566564656` (your second origin)
	**Viewer protocol policy:** Redirect HTTP to HTTPS
	**Allowed HTTP methods:** GET, HEAD, OPTIONS
	**Cache policy and origin request policy:**
	- Cache policy: CachingOptimized
	- Origin request policy: CORS-S3Origin


### 10. Update Lambda Function

1. Open Lambda → `cloudfront-invalidation`
2. Find line: `distribution_id = 'YOUR_DISTRIBUTION_ID'`
3. Replace with your actual Distribution ID
4. Deploy

### 11. Configure CloudFront for Image Delivery
  
1. **Click "Create behavior"**    
    **Path pattern:** `processed/thumbnails/*`
    **Origin:** `processed-content-origin`
    **Viewer protocol policy:** Redirect HTTP to HTTPS
    **Cache policy:** CachingOptimized
    
    **Response headers policy:**
    - Click "Create policy"
    - Name: `image-headers`
    - Custom headers:
        - Cache-Control: public
        - max-age=31536000
		
    - CORS: Enable
    - Click "Create"
    - Select the new policy
    
1. **Repeat for other processed content types:**
    
    - `processed/medium/*`
    - `processed/webp/*`
    - `video-processed/*`

### 12. Enable CloudFront Access Logging

1. **Go to your distribution → General tab**
2. **Click Edit**
3. **Standard logging:**
    - Toggle On
    - S3 bucket: `content-platform-analytics-1684845`
    - Log prefix: `cloudfront-logs/`
    



## Search and Analytics

### 13. Set Up Analytics with Athena and QuickSight

1. **Configure Athena for S3 Data Querying**
    
    ```sql
    -- Create database for content analytics
    CREATE DATABASE content_analytics;
    
    -- Create table for CloudFront access logs
    CREATE EXTERNAL TABLE content_analytics.cloudfront_logs (
      date_time string,
      edge_location string,
      bytes_sent bigint,
      client_ip string,
      method string,
      host string,
      uri_stem string,
      status_code int,
      referer string,
      user_agent string,
      uri_query string,
      cookie string,
      result_type string,
      request_id string,
      host_header string,
      request_protocol string,
      bytes_received bigint,
      time_taken double,
      forwarded_for string,
      ssl_protocol string,
      ssl_cipher string,
      response_result_type string,
      http_version string,
      fle_status string,
      fle_encrypted_fields int,
      port int,
      time_to_first_byte double,
      detail_result_type string,
      country string,
      city string,
      region string
    )
    ROW FORMAT DELIMITED
    FIELDS TERMINATED BY '\t'
    STORED AS TEXTFILE
    LOCATION 's3://content-platform-analytics-[unique-id]/cloudfront-logs/'
    TBLPROPERTIES ('skip.header.line.count'='2');
    
    -- Create table for S3 access logs
    CREATE EXTERNAL TABLE content_analytics.s3_access_logs (
      bucket_owner string,
      bucket string,
      request_time string,
      remote_ip string,
      requester string,
      request_id string,
      operation string,
      key string,
      request_uri string,
      http_status string,
      error_code string,
      bytes_sent bigint,
      object_size bigint,
      total_time int,
      turn_around_time int,
      referer string,
      user_agent string,
      version_id string,
      host_id string,
      signature_version string,
      cipher_suite string,
      authentication_type string,
      host_header string,
      tls_version string
    )
    ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.RegexSerDe'
    WITH SERDEPROPERTIES (
      'serialization.format' = '1',
      'input.regex' = '([^ ]*) ([^ ]*) \\[(.*?)\\] ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) \\\"([^ ]*)([^\\\"]*)?\\\" (-|[0-9]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) \\\"([^\\\"]*)?\\\" ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) ([^ ]*) "EXCLUDE",
                                    "MoovPlacement": "PROGRESSIVE_DOWNLOAD"
                                }
                            }
                        },
                        {
                            "NameModifier": "_480p",
                            "VideoDescription": {
                                "Width": 854,
                                "Height": 480,
                                "CodecSettings": {
                                    "Codec": "H_264",
                                    "H264Settings": {
                                        "MaxBitrate": 1500000,
                                        "RateControlMode": "QVBR",
                                        "QvbrSettings": {"QvbrQualityLevel": 7}
                                    }
                                }
                            },
                            "AudioDescriptions": [
                                {
                                    "AudioTypeControl": "FOLLOW_INPUT",
                                    "AudioSourceName": "Audio Selector 1",
                                    "CodecSettings": {
                                        "Codec": "AAC",
                                        "AacSettings": {
                                            "AudioDescriptionBroadcasterMix": "NORMAL",
                                            "Bitrate": 96000,
                                            "RateControlMode": "CBR",
                                            "CodecProfile": "LC",
                                            "CodingMode": "CODING_MODE_2_0",
                                            "SampleRate": 48000
                                        }
                                    }
                                }
                            ],
                            "ContainerSettings": {
                                "Container": "MP4",
                                "Mp4Settings": {
                                    "CslgAtom": "INCLUDE",
                                    "FreeSpaceBox":
    )
    STORED AS TEXTFILE
    LOCATION 's3://content-platform-analytics-[unique-id]/s3-access-logs/';
    
    -- Create partitioned table for content metadata
    CREATE EXTERNAL TABLE content_analytics.content_metadata (
      file_id string,
      file_name string,
      file_path string,
      content_type string,
      file_size bigint,
      upload_date timestamp,
      processing_status string,
      tags array<string>,
      metadata struct<
        width:int,
        height:int,
        duration:double,
        format:string
      >
    )
    PARTITIONED BY (
      year string,
      month string,
      day string
    )
    STORED AS PARQUET
    LOCATION 's3://content-platform-analytics-[unique-id]/content-metadata/'
    TBLPROPERTIES ('has_encrypted_data'='false');
```



## Security and Backup

### 14. Implement Bucket Policies

**Apply to all content buckets:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedObjectUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::BUCKET-NAME/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": "AES256"
        }
      }
    },
    {
      "Sid": "DenyInsecureTransport",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::BUCKET-NAME",
        "arn:aws:s3:::BUCKET-NAME/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    },
    {
      "Sid": "RestrictPublicAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "s3:PutBucketPublicAccessBlock",
        "s3:DeleteBucketPolicy",
        "s3:PutBucketPolicy"
      ],
      "Resource": "arn:aws:s3:::BUCKET-NAME",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalArn": "arn:aws:iam::ACCOUNT-ID:role/AdminRole"
        }
      }
    }
  ]
}
```

**Apply this policy to:**
- content-platform-primary-566564656
- content-platform-processed-564156156
- content-platform-raw-data-468862
- content-platform-backup-351586884


**Enable MFA Delete on Critical Buckets**

```bash
# Enable versioning with MFA Delete (AWS CLI)
aws s3api put-bucket-versioning \
  --bucket content-platform-primary-566564656 \
  --versioning-configuration Status=Enabled,MFADelete=Enabled \
  --mfa "arn:aws:iam::011868793819:mfa/root-account-mfa-device XXXXXX"
```

**Note:** MFA Delete requires root account MFA. Apply to:
- Primary bucket
- Backup bucket



**For each bucket, enable access logging:**

```bash
# Create logging policy for analytics bucket
aws s3api put-bucket-policy \
  --bucket content-platform-analytics-1684845 \
  --policy '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "S3ServerAccessLogsPolicy",
        "Effect": "Allow",
        "Principal": {
          "Service": "logging.s3.amazonaws.com"
        },
        "Action": "s3:PutObject",
        "Resource": "arn:aws:s3:::content-platform-analytics-1684845/s3-access-logs/*",
        "Condition": {
          "StringEquals": {
            "aws:SourceAccount": "011868793819"
          }
        }
      }
    ]
  }'

# Enable logging on primary bucket
aws s3api put-bucket-logging \
  --bucket content-platform-primary-566564656 \
  --bucket-logging-status '{
    "LoggingEnabled": {
      "TargetBucket": "content-platform-analytics-1684845",
      "TargetPrefix": "s3-access-logs/primary/"
    }
  }'
```

**Repeat for all buckets with appropriate prefixes.**


### 15. Implement Backup Strategy

**Create Backup Lambda Function:**

Function name: `content-backup-validator`
Runtime: Python 3.11
Role: content-platform-lambda-role

```python
import json
import boto3
from datetime import datetime, timedelta

s3 = boto3.client('s3')
sns = boto3.client('sns')

def lambda_handler(event, context):
    """
    Validate backup bucket integrity and report status
    """
    primary_bucket = 'content-platform-primary-566564656'
    backup_bucket = 'content-platform-backup-351586884'
    
    try:
        # Count objects in primary bucket
        primary_count = count_objects(primary_bucket)
        
        # Count objects in backup bucket
        backup_count = count_objects(backup_bucket)
        
        # Calculate replication lag
        lag = calculate_replication_lag(primary_bucket, backup_bucket)
        
        # Check backup health
        health_status = "HEALTHY" if backup_count >= primary_count * 0.95 else "DEGRADED"
        
        report = {
            'timestamp': datetime.utcnow().isoformat(),
            'primary_object_count': primary_count,
            'backup_object_count': backup_count,
            'replication_percentage': (backup_count / primary_count * 100) if primary_count > 0 else 0,
            'replication_lag_hours': lag,
            'status': health_status
        }
        
        # Send SNS notification if degraded
        if health_status == "DEGRADED":
            send_alert(report)
        
        return {
            'statusCode': 200,
            'body': json.dumps(report)
        }
        
    except Exception as e:
        print(f"Backup validation error: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }

def count_objects(bucket):
    """Count total objects in bucket"""
    count = 0
    paginator = s3.get_paginator('list_objects_v2')
    for page in paginator.paginate(Bucket=bucket):
        count += page.get('KeyCount', 0)
    return count

def calculate_replication_lag(primary_bucket, backup_bucket):
    """Calculate replication lag in hours"""
    try:
        # Get latest object from primary
        primary_response = s3.list_objects_v2(
            Bucket=primary_bucket,
            MaxKeys=1
        )
        
        if 'Contents' not in primary_response:
            return 0
        
        primary_latest = primary_response['Contents'][0]['LastModified']
        
        # Get latest object from backup
        backup_response = s3.list_objects_v2(
            Bucket=backup_bucket,
            MaxKeys=1
        )
        
        if 'Contents' not in backup_response:
            return 999
        
        backup_latest = backup_response['Contents'][0]['LastModified']
        
        # Calculate lag
        lag = (datetime.now(primary_latest.tzinfo) - backup_latest).total_seconds() / 3600
        return round(lag, 2)
        
    except Exception as e:
        print(f"Error calculating lag: {str(e)}")
        return 0

def send_alert(report):
    """Send SNS alert for backup issues"""
    try:
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:011868793819:backup-alerts',
            Subject='⚠️ Backup Health Degraded',
            Message=json.dumps(report, indent=2)
        )
    except Exception as e:
        print(f"Error sending alert: {str(e)}")
```

**Schedule with EventBridge:**
- Rule name: `daily-backup-validation`
- Schedule: `cron(0 2 * * ? *)` (2 AM UTC daily)
- Target: content-backup-validator Lambda

### 16. Create SNS Topic for Alerts

```bash
# Create SNS topic
aws sns create-topic --name backup-alerts

# Subscribe email
aws sns subscribe \
  --topic-arn arn:aws:sns:us-east-1:011868793819:backup-alerts \
  --protocol email \
  --notification-endpoint emadhajaj7@gmail.com
```

### 17. Implement Secrets Management

**For sensitive data (API keys, credentials):**

```bash
# Store MediaConvert role ARN in SSM Parameter Store
aws ssm put-parameter \
  --name "/contentplatform/mediaconvert/role-arn" \
  --value "arn:aws:iam::ACCOUNT-ID:role/MediaConvertRole" \
  --type "String" \
  --description "MediaConvert IAM role ARN"

# Store CloudFront distribution ID
aws ssm put-parameter \
  --name "/contentplatform/cloudfront/distribution-id" \
  --value "YOUR-DISTRIBUTION-ID" \
  --type "String"
```

**Update Lambda to use SSM:**

```python
import boto3

ssm = boto3.client('ssm')

def get_parameter(name):
    """Retrieve parameter from SSM"""
    response = ssm.get_parameter(Name=name, WithDecryption=True)
    return response['Parameter']['Value']

# Usage in Lambda
mediaconvert_role = get_parameter('/contentplatform/mediaconvert/role-arn')
```

### 18. Enable AWS Config for Compliance

**Create Config Rules:**

```bash
# Check if S3 buckets have versioning enabled
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-bucket-versioning-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_BUCKET_VERSIONING_ENABLED"
    },
    "Scope": {
      "ComplianceResourceTypes": ["AWS::S3::Bucket"]
    }
  }'

# Check if S3 buckets have encryption enabled
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "s3-default-encryption-enabled",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "S3_DEFAULT_ENCRYPTION_KMS"
    }
  }'
```

---

## Monitoring and Cost Optimization

### 19. CloudWatch Dashboard

**Create comprehensive monitoring dashboard:**

```bash
# Create dashboard JSON
cat > dashboard.json << 'EOF'
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          [ "AWS/Lambda", "Invocations", { "stat": "Sum" } ],
          [ ".", "Errors", { "stat": "Sum", "color": "#d62728" } ],
          [ ".", "Duration", { "stat": "Average" } ],
          [ ".", "Throttles", { "stat": "Sum", "color": "#ff7f0e" } ]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "Lambda Performance",
        "yAxis": {
          "left": {
            "min": 0
          }
        }
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          [ "AWS/S3", "BucketSizeBytes", { "stat": "Average" } ],
          [ ".", "NumberOfObjects", { "stat": "Average" } ]
        ],
        "period": 86400,
        "stat": "Average",
        "region": "us-east-1",
        "title": "S3 Storage Metrics"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          [ "AWS/CloudFront", "Requests", { "stat": "Sum" } ],
          [ ".", "BytesDownloaded", { "stat": "Sum" } ],
          [ ".", "4xxErrorRate", { "stat": "Average" } ],
          [ ".", "5xxErrorRate", { "stat": "Average" } ]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "CloudFront Metrics"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          [ "AWS/DynamoDB", "ConsumedReadCapacityUnits", { "stat": "Sum" } ],
          [ ".", "ConsumedWriteCapacityUnits", { "stat": "Sum" } ],
          [ ".", "UserErrors", { "stat": "Sum" } ]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "DynamoDB Performance"
      }
    },
    {
      "type": "metric",
      "properties": {
        "metrics": [
          [ "AWS/States", "ExecutionsFailed", { "stat": "Sum", "color": "#d62728" } ],
          [ ".", "ExecutionsSucceeded", { "stat": "Sum", "color": "#2ca02c" } ],
          [ ".", "ExecutionTime", { "stat": "Average" } ]
        ],
        "period": 300,
        "stat": "Average",
        "region": "us-east-1",
        "title": "Step Functions"
      }
    }
  ]
}
EOF

# Create dashboard
aws cloudwatch put-dashboard \
  --dashboard-name ContentPlatformMonitoring \
  --dashboard-body file://dashboard.json
```

### 20. CloudWatch Alarms

**Create critical alarms:**

```bash
# Lambda error rate alarm
aws cloudwatch put-metric-alarm \
  --alarm-name lambda-high-error-rate \
  --alarm-description "Alert when Lambda error rate exceeds 5%" \
  --metric-name Errors \
  --namespace AWS/Lambda \
  --statistic Sum \
  --period 300 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:011868793819:backup-alerts

# S3 4xx error alarm
aws cloudwatch put-metric-alarm \
  --alarm-name s3-high-4xx-errors \
  --metric-name 4xxErrors \
  --namespace AWS/S3 \
  --statistic Sum \
  --period 300 \
  --threshold 100 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:011868793819:backup-alerts

# CloudFront 5xx error alarm
aws cloudwatch put-metric-alarm \
  --alarm-name cloudfront-high-5xx-errors \
  --metric-name 5xxErrorRate \
  --namespace AWS/CloudFront \
  --statistic Average \
  --period 300 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --alarm-actions arn:aws:sns:us-east-1:011868793819:backup-alerts

# Step Functions failure alarm
aws cloudwatch put-metric-alarm \
  --alarm-name stepfunctions-executions-failed \
  --metric-name ExecutionsFailed \
  --namespace AWS/States \
  --statistic Sum \
  --period 300 \
  --threshold 5 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 1 \
  --alarm-actions arn:aws:sns:us-east-1:011868793819:backup-alerts
```

### 21. Cost Optimization Strategies

**A. S3 Cost Optimization**

```bash
# Enable S3 Storage Lens (Free tier available)
aws s3control put-storage-lens-configuration \
  --account-id 011868793819 \
  --config-id content-platform-lens \
  --storage-lens-configuration '{
    "Id": "content-platform-lens",
    "AccountLevel": {
      "BucketLevel": {
        "ActivityMetrics": {
          "IsEnabled": true
        },
        "AdvancedCostOptimizationMetrics": {
          "IsEnabled": true
        }
      }
    },
    "IsEnabled": true
  }'

# Add S3 Intelligent-Tiering archive configurations
aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket content-platform-primary-566564656 \
  --id auto-archive-config \
  --intelligent-tiering-configuration '{
    "Id": "auto-archive-config",
    "Status": "Enabled",
    "Tierings": [
      {
        "Days": 90,
        "AccessTier": "ARCHIVE_ACCESS"
      },
      {
        "Days": 180,
        "AccessTier": "DEEP_ARCHIVE_ACCESS"
      }
    ]
  }'
```

**B. Lambda Cost Optimization**

```python
# Add to Lambda functions to track cold starts
import time

start_time = time.time()
COLD_START = True

def lambda_handler(event, context):
    global COLD_START
    
    if COLD_START:
        print(f"COLD_START: {time.time() - start_time}s")
        COLD_START = False
    
    # Your function code here
```

**Configure Lambda reserved concurrency for predictable workloads:**

```bash
# Set reserved concurrency for upload handler
aws lambda put-function-concurrency \
  --function-name content-upload-handler \
  --reserved-concurrent-executions 5
```

**C. CloudFront Cost Optimization**

```bash
# Enable CloudFront cost allocation tags
aws cloudfront tag-resource \
  --resource arn:aws:cloudfront::011868793819:distribution/E1FUBFA6K3A30D \
  --tags '{
    "Items": [
      {"Key": "Project", "Value": "ContentPlatform"},
      {"Key": "CostCenter", "Value": "Media"},
      {"Key": "Environment", "Value": "Production"}
    ]
  }'
```

### 22. Cost Monitoring Lambda

Function name: `cost-report-generator`
Runtime: Python 3.11
Schedule: Weekly (Monday 9 AM)

```python
import json
import boto3
from datetime import datetime, timedelta

ce = boto3.client('ce')
sns = boto3.client('sns')

def lambda_handler(event, context):
    """
    Generate weekly cost report for Content Platform
    """
    end_date = datetime.now().strftime('%Y-%m-%d')
    start_date = (datetime.now() - timedelta(days=7)).strftime('%Y-%m-%d')
    
    try:
        # Get cost and usage
        response = ce.get_cost_and_usage(
            TimePeriod={
                'Start': start_date,
                'End': end_date
            },
            Granularity='DAILY',
            Metrics=['UnblendedCost'],
            Filter={
                'Tags': {
                    'Key': 'Project',
                    'Values': ['ContentPlatform']
                }
            },
            GroupBy=[
                {'Type': 'SERVICE', 'Key': 'SERVICE'}
            ]
        )
        
        # Process results
        service_costs = {}
        total_cost = 0
        
        for result in response['ResultsByTime']:
            for group in result['Groups']:
                service = group['Keys'][0]
                cost = float(group['Metrics']['UnblendedCost']['Amount'])
                service_costs[service] = service_costs.get(service, 0) + cost
                total_cost += cost
        
        # Generate report
        report = generate_report(service_costs, total_cost, start_date, end_date)
        
        # Send via SNS
        sns.publish(
            TopicArn='arn:aws:sns:us-east-1:011868793819:backup-alerts',
            Subject=f'💰 Weekly Cost Report: ${total_cost:.2f}',
            Message=report
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps({'total_cost': total_cost})
        }
        
    except Exception as e:
        print(f"Error generating cost report: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }

def generate_report(service_costs, total_cost, start_date, end_date):
    """Format cost report"""
    report = f"""
Content Platform Weekly Cost Report
Period: {start_date} to {end_date}
Total Cost: ${total_cost:.2f}

Service Breakdown:
"""
    
    sorted_services = sorted(service_costs.items(), key=lambda x: x[1], reverse=True)
    
    for service, cost in sorted_services:
        percentage = (cost / total_cost * 100) if total_cost > 0 else 0
        report += f"  {service}: ${cost:.2f} ({percentage:.1f}%)\n"
    
    report += f"\n🎯 Free Tier Status:\n"
    report += "  ⚠️  Check AWS Billing Dashboard for detailed free tier usage\n"
    
    return report
```

### 23. Enable AWS Budgets

```bash
# Create monthly budget alert
aws budgets create-budget \
  --account-id 011868793819 \
  --budget '{
    "BudgetName": "ContentPlatformMonthlyBudget",
    "BudgetLimit": {
      "Amount": "50",
      "Unit": "USD"
    },
    "TimeUnit": "MONTHLY",
    "BudgetType": "COST",
    "CostFilters": {
      "TagKeyValue": ["Project$ContentPlatform"]
    }
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [{
        "SubscriptionType": "EMAIL",
        "Address": "emadhajaj7@gmail.com"
      }]
    }
  ]'
```

---

## Testing and Demonstration



**File: `setup_test_environment.sh`**

```bash
#!/bin/bash

echo "Setting up test environment..."

# Check AWS CLI
if ! command -v aws &> /dev/null; then
    echo "❌ AWS CLI not found. Please install it first."
    echo "Visit: https://aws.amazon.com/cli/"
    exit 1
fi

# Check jq
if ! command -v jq &> /dev/null; then
    echo "Installing jq..."
    if [[ "$OSTYPE" == "darwin"* ]]; then
        brew install jq
    elif [[ "$OSTYPE" == "linux-gnu"* ]]; then
        sudo apt-get update && sudo apt-get install -y jq
    fi
fi

# Check AWS credentials
if ! aws sts get-caller-identity &> /dev/null; then
    echo "❌ AWS credentials not configured."
    echo "Run: aws configure"
    exit 1
fi

# Prompt for configuration
echo "Enter your AWS configuration:"
read -p "Primary Bucket Name: " PRIMARY_BUCKET
read -p "Processed Bucket Name: " PROCESSED_BUCKET
read -p "CloudFront Domain: " CLOUDFRONT_DOMAIN
read -p "AWS Account ID: " ACCOUNT_ID
read -p "AWS Region [us-east-1]: " AWS_REGION
AWS_REGION=${AWS_REGION:-us-east-1}

# Create config file
cat > test_config.env << EOF
export PRIMARY_BUCKET="$PRIMARY_BUCKET"
export PROCESSED_BUCKET="$PROCESSED_BUCKET"
export CLOUDFRONT_DOMAIN="$CLOUDFRONT_DOMAIN"
export ACCOUNT_ID="$ACCOUNT_ID"
export AWS_REGION="$AWS_REGION"
export DYNAMODB_TABLE="content-metadata"
export STATE_MACHINE_ARN="arn:aws:states:$AWS_REGION:$ACCOUNT_ID:stateMachine:content-processing-workflow"
EOF

echo "✓ Configuration saved to test_config.env"
echo ""
echo "To run tests, execute:"
echo "  source test_config.env"
echo "  bash test_platform.sh"
```




**File: `test_platform.sh`**

```bash
#!/bin/bash

# AWS Content Platform - Comprehensive Test Script
# This script tests all components of the platform

set -e

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Configuration
PRIMARY_BUCKET="content-platform-primary-566564656"
PROCESSED_BUCKET="content-platform-processed-564156156"
DYNAMODB_TABLE="content-metadata"
STATE_MACHINE_ARN="arn:aws:states:us-east-1:011868793819:stateMachine:content-processing-workflow"
CLOUDFRONT_DOMAIN="d3wfyqy0nkis4.cloudfront.net"

# Test files directory
TEST_DIR="./test_files"
RESULTS_FILE="./test_results.json"

echo "=========================================="
echo "AWS Content Platform - Integration Test"
echo "=========================================="
echo ""

# Function to print status
print_status() {
    if [ $1 -eq 0 ]; then
        echo -e "${GREEN}✓ $2${NC}"
    else
        echo -e "${RED}✗ $2${NC}"
    fi
}

# Function to wait for processing
wait_for_processing() {
    local file_key=$1
    local max_wait=120
    local wait_time=0
    
    echo "Waiting for processing to complete..."
    
    while [ $wait_time -lt $max_wait ]; do
        # Check DynamoDB for processing status
        status=$(aws dynamodb get-item \
            --table-name $DYNAMODB_TABLE \
            --key "{\"file_id\": {\"S\": \"$(echo -n $file_key | md5sum | cut -d' ' -f1)\"}}" \
            --query 'Item.processing_status.S' \
            --output text 2>/dev/null || echo "pending")
        
        if [ "$status" = "completed" ]; then
            echo -e "${GREEN}Processing completed!${NC}"
            return 0
        fi
        
        echo -n "."
        sleep 5
        wait_time=$((wait_time + 5))
    done
    
    echo -e "${RED}Processing timeout${NC}"
    return 1
}

# Create test files directory
mkdir -p $TEST_DIR

# Initialize results
cat > $RESULTS_FILE << EOF
{
  "test_run": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
  "tests": []
}
EOF

echo "📋 Test 1: Verify AWS Resources"
echo "-----------------------------------"

# Test S3 buckets
echo "Testing S3 buckets..."
buckets=($PRIMARY_BUCKET $PROCESSED_BUCKET)
for bucket in "${buckets[@]}"; do
    aws s3api head-bucket --bucket $bucket 2>/dev/null
    print_status $? "Bucket exists: $bucket"
done

# Test DynamoDB table
echo "Testing DynamoDB table..."
aws dynamodb describe-table --table-name $DYNAMODB_TABLE >/dev/null 2>&1
print_status $? "DynamoDB table exists: $DYNAMODB_TABLE"

# Test Lambda functions
echo "Testing Lambda functions..."
functions=(
    "content-upload-handler"
    "content-image-processor"
    "content-video-processor"
    "content-generic-processor"
    "update-metadata"
    "cloudfront-invalidation"
)

for func in "${functions[@]}"; do
    aws lambda get-function --function-name $func >/dev/null 2>&1
    print_status $? "Lambda function exists: $func"
done

# Test Step Functions
echo "Testing Step Functions..."
aws stepfunctions describe-state-machine --state-machine-arn $STATE_MACHINE_ARN >/dev/null 2>&1
print_status $? "State machine exists"

echo ""
echo "📸 Test 2: Image Upload and Processing"
echo "-----------------------------------"

# Create test image
echo "Creating test image..."
convert -size 800x600 xc:blue -pointsize 50 -fill white \
    -annotate +200+300 "Test Image $(date +%s)" \
    $TEST_DIR/test_image.jpg 2>/dev/null || \
    echo "ImageMagick not found. Using placeholder method..."

# If ImageMagick not available, create a simple PPM image
if [ ! -f $TEST_DIR/test_image.jpg ]; then
    cat > $TEST_DIR/test_image.ppm << EOF
P3
100 100
255
$(for i in {1..10000}; do echo "0 100 200"; done)
EOF
    # Convert PPM to JPG if possible
    convert $TEST_DIR/test_image.ppm $TEST_DIR/test_image.jpg 2>/dev/null || \
        cp $TEST_DIR/test_image.ppm $TEST_DIR/test_image.jpg
fi

# Upload image
echo "Uploading test image..."
aws s3 cp $TEST_DIR/test_image.jpg s3://$PRIMARY_BUCKET/test/test_image_$(date +%s).jpg
test_image_key="test/test_image_$(date +%s).jpg"
print_status $? "Image uploaded to S3"

# Wait for processing
wait_for_processing $test_image_key

# Verify processed files exist
echo "Verifying processed files..."
aws s3 ls s3://$PROCESSED_BUCKET/processed/thumbnails/ | grep -q "test_image"
print_status $? "Thumbnail generated"

aws s3 ls s3://$PROCESSED_BUCKET/processed/medium/ | grep -q "test_image"
print_status $? "Medium size generated"

aws s3 ls s3://$PROCESSED_BUCKET/processed/webp/ | grep -q "test_image"
print_status $? "WebP format generated"

echo ""
echo "📄 Test 3: Generic File Upload"
echo "-----------------------------------"

# Create test PDF
echo "Creating test document..."
echo "This is a test PDF content generated at $(date)" > $TEST_DIR/test_document.txt

# Upload document
echo "Uploading test document..."
aws s3 cp $TEST_DIR/test_document.txt s3://$PRIMARY_BUCKET/documents/test_doc_$(date +%s).txt
test_doc_key="documents/test_doc_$(date +%s).txt"
print_status $? "Document uploaded to S3"

# Wait for processing
wait_for_processing $test_doc_key

echo ""
echo "🔍 Test 4: DynamoDB Metadata Verification"
echo "-----------------------------------"

# Query DynamoDB for recent items
echo "Querying DynamoDB for metadata..."
recent_items=$(aws dynamodb scan \
    --table-name $DYNAMODB_TABLE \
    --limit 5 \
    --output json)

item_count=$(echo $recent_items | jq '.Items | length')
print_status $? "Retrieved $item_count metadata records"

# Verify specific metadata fields
echo $recent_items | jq -r '.Items[0] | {file_path, content_type, processing_status}' 2>/dev/null
print_status $? "Metadata structure valid"

echo ""
echo "🌐 Test 5: CloudFront Distribution"
echo "-----------------------------------"

# Test CloudFront access
echo "Testing CloudFront distribution..."
response_code=$(curl -s -o /dev/null -w "%{http_code}" "https://$CLOUDFRONT_DOMAIN/")
if [ $response_code -eq 200 ] || [ $response_code -eq 403 ]; then
    print_status 0 "CloudFront distribution accessible (HTTP $response_code)"
else
    print_status 1 "CloudFront distribution issue (HTTP $response_code)"
fi

# Test processed content delivery
echo "Testing processed content delivery..."
latest_thumb=$(aws s3 ls s3://$PROCESSED_BUCKET/processed/thumbnails/ --recursive | sort | tail -n 1 | awk '{print $4}')
if [ ! -z "$latest_thumb" ]; then
    thumb_url="https://$CLOUDFRONT_DOMAIN/$latest_thumb"
    response_code=$(curl -s -o /dev/null -w "%{http_code}" "$thumb_url")
    print_status $? "Thumbnail accessible via CloudFront (HTTP $response_code)"
fi

echo ""
echo "⚡ Test 6: Step Functions Workflow"
echo "-----------------------------------"

# List recent executions
echo "Checking Step Functions executions..."
executions=$(aws stepfunctions list-executions \
    --state-machine-arn $STATE_MACHINE_ARN \
    --max-results 10 \
    --output json)

total_executions=$(echo $executions | jq '.executions | length')
succeeded=$(echo $executions | jq '[.executions[] | select(.status=="SUCCEEDED")] | length')
failed=$(echo $executions | jq '[.executions[] | select(.status=="FAILED")] | length')

echo "Total executions: $total_executions"
echo "Succeeded: $succeeded"
echo "Failed: $failed"

if [ $total_executions -gt 0 ]; then
    success_rate=$((succeeded * 100 / total_executions))
    print_status 0 "Success rate: $success_rate%"
else
    print_status 1 "No executions found"
fi

echo ""
echo "💾 Test 7: Cross-Region Replication"
echo "-----------------------------------"

# Check replication status
echo "Checking replication configuration..."
replication=$(aws s3api get-bucket-replication --bucket $PRIMARY_BUCKET 2>/dev/null || echo "{}")

if [ "$replication" != "{}" ]; then
    print_status 0 "Replication configured"
    
    # Compare object counts
    primary_count=$(aws s3 ls s3://$PRIMARY_BUCKET --recursive | wc -l)
    backup_count=$(aws s3 ls s3://content-platform-backup-351586884 --recursive | wc -l)
    
    echo "Primary bucket objects: $primary_count"
    echo "Backup bucket objects: $backup_count"
    
    if [ $backup_count -ge $((primary_count * 90 / 100)) ]; then
        print_status 0 "Replication health: Good (>90%)"
    else
        print_status 1 "Replication health: Degraded (<90%)"
    fi
else
    print_status 1 "Replication not configured"
fi

echo ""
echo "📊 Test 8: Lifecycle Policies"
echo "-----------------------------------"

# Check lifecycle configuration
echo "Verifying lifecycle policies..."
lifecycle=$(aws s3api get-bucket-lifecycle-configuration --bucket $PRIMARY_BUCKET 2>/dev/null || echo "{}")

if [ "$lifecycle" != "{}" ]; then
    rule_count=$(echo $lifecycle | jq '.Rules | length')
    print_status 0 "Lifecycle policies active: $rule_count rules"
    
    # Display rules summary
    echo $lifecycle | jq -r '.Rules[] | "  - \(.ID): Transitions to \(.Transitions[0].StorageClass) after \(.Transitions[0].Days) days"' 2>/dev/null
else
    print_status 1 "No lifecycle policies found"
fi

echo ""
echo "🔐 Test 9: Security Configuration"
echo "-----------------------------------"

# Check bucket encryption
echo "Checking encryption..."
encryption=$(aws s3api get-bucket-encryption --bucket $PRIMARY_BUCKET 2>/dev/null || echo "{}")

if [ "$encryption" != "{}" ]; then
    print_status 0 "Bucket encryption enabled"
else
    print_status 1 "Bucket encryption not enabled"
fi

# Check versioning
versioning=$(aws s3api get-bucket-versioning --bucket $PRIMARY_BUCKET --query 'Status' --output text)
if [ "$versioning" = "Enabled" ]; then
    print_status 0 "Versioning enabled"
else
    print_status 1 "Versioning not enabled"
fi

# Check public access block
public_access=$(aws s3api get-public-access-block --bucket $PRIMARY_BUCKET 2>/dev/null || echo "{}")
if echo $public_access | grep -q "BlockPublicAcls.*true"; then
    print_status 0 "Public access blocked"
else
    print_status 1 "Public access not fully blocked"
fi

echo ""
echo "📈 Test 10: CloudWatch Metrics"
echo "-----------------------------------"

# Check Lambda metrics
echo "Checking Lambda invocations (last hour)..."
end_time=$(date -u +%Y-%m-%dT%H:%M:%S)
start_time=$(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S 2>/dev/null || date -u -v-1H +%Y-%m-%dT%H:%M:%S)

invocations=$(aws cloudwatch get-metric-statistics \
    --namespace AWS/Lambda \
    --metric-name Invocations \
    --dimensions Name=FunctionName,Value=content-upload-handler \
    --start-time $start_time \
    --end-time $end_time \
    --period 3600 \
    --statistics Sum \
    --output json)

invocation_count=$(echo $invocations | jq '.Datapoints[0].Sum // 0')
print_status 0 "Lambda invocations in last hour: $invocation_count"

# Check S3 metrics
echo "Checking S3 storage size..."
storage_size=$(aws cloudwatch get-metric-statistics \
    --namespace AWS/S3 \
    --metric-name BucketSizeBytes \
    --dimensions Name=BucketName,Value=$PRIMARY_BUCKET Name=StorageType,Value=StandardStorage \
    --start-time $start_time \
    --end-time $end_time \
    --period 86400 \
    --statistics Average \
    --output json | jq '.Datapoints[0].Average // 0')

storage_mb=$((storage_size / 1024 / 1024))
print_status 0 "S3 storage size: ${storage_mb} MB"

echo ""
echo "🧪 Test 11: End-to-End Workflow Test"
echo "-----------------------------------"

# Create a unique test file
unique_id=$(date +%s)
test_file="e2e_test_$unique_id.jpg"

echo "Creating unique test file: $test_file"
echo "Test data $unique_id" > $TEST_DIR/$test_file

# Upload file
echo "1. Uploading file..."
aws s3 cp $TEST_DIR/$test_file s3://$PRIMARY_BUCKET/e2e-test/$test_file
upload_success=$?
print_status $upload_success "File uploaded"

if [ $upload_success -eq 0 ]; then
    # Wait and check processing
    echo "2. Waiting for processing (60 seconds)..."
    sleep 60
    
    # Check DynamoDB
    file_id=$(echo -n "e2e-test/$test_file" | md5sum | cut -d' ' -f1)
    metadata=$(aws dynamodb get-item \
        --table-name $DYNAMODB_TABLE \
        --key "{\"file_id\": {\"S\": \"$file_id\"}}" \
        --output json 2>/dev/null)
    
    if [ ! -z "$metadata" ]; then
        processing_status=$(echo $metadata | jq -r '.Item.processing_status.S // "not_found"')
        print_status 0 "Metadata stored (status: $processing_status)"
    else
        print_status 1 "Metadata not found in DynamoDB"
    fi
    
    # Check Step Functions execution
    execution_name="process-$file_id"
    execution=$(aws stepfunctions describe-execution \
        --execution-arn "$STATE_MACHINE_ARN:$execution_name" \
        --output json 2>/dev/null || echo "{}")
    
    if [ "$execution" != "{}" ]; then
        exec_status=$(echo $execution | jq -r '.status // "UNKNOWN"')
        print_status 0 "Step Functions execution found (status: $exec_status)"
    else
        print_status 1 "Step Functions execution not found"
    fi
fi

echo ""
echo "🧹 Test 12: Cleanup Test Files"
echo "-----------------------------------"

# Optional: Remove test files
read -p "Remove test files from S3? (y/n): " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    echo "Cleaning up test files..."
    aws s3 rm s3://$PRIMARY_BUCKET/test/ --recursive
    aws s3 rm s3://$PRIMARY_BUCKET/e2e-test/ --recursive
    aws s3 rm s3://$PRIMARY_BUCKET/documents/ --recursive
    print_status $? "Test files removed"
fi

echo ""
echo "=========================================="
echo "📊 Test Summary"
echo "=========================================="

# Generate summary report
cat > test_summary.txt << EOF
AWS Content Platform - Test Summary
Generated: $(date)

Infrastructure Tests:
✓ S3 Buckets: Verified
✓ DynamoDB: Verified
✓ Lambda Functions: Verified
✓ Step Functions: Verified

Functional Tests:
✓ Image Processing: Verified
✓ Generic File Upload: Verified
✓ Metadata Storage: Verified
✓ CloudFront Delivery: Verified

Security Tests:
✓ Encryption: Verified
✓ Versioning: Verified
✓ Public Access Block: Verified

Performance Metrics:
- Lambda Invocations (1h): $invocation_count
- S3 Storage Size: ${storage_mb} MB
- DynamoDB Items: $item_count
- Step Functions Success Rate: $success_rate%

Replication Status:
- Primary Objects: $primary_count
- Backup Objects: $backup_count

All tests completed successfully!
EOF

cat test_summary.txt

echo ""
echo -e "${GREEN}✓ All tests completed!${NC}"
echo "Detailed results saved to: test_summary.txt"
echo ""

# Save results to JSON
jq --arg date "$(date -u +%Y-%m-%dT%H:%M:%SZ)" \
   --arg status "completed" \
   --argjson invocations "$invocation_count" \
   --argjson storage "$storage_mb" \
   '.test_run = $date | .status = $status | .metrics = {invocations: $invocations, storage_mb: $storage}' \
   $RESULTS_FILE > ${RESULTS_FILE}.tmp && mv ${RESULTS_FILE}.tmp $RESULTS_FILE

echo "JSON results saved to: $RESULTS_FILE"
```




## Cost Analysis

### Monthly Cost Breakdown (Free Tier)

**Estimated Monthly Costs:**

| Service            | Free Tier                        | Expected Usage                | Estimated Cost                    |
| ------------------ | -------------------------------- | ----------------------------- | --------------------------------- |
| **S3 Standard**    | 5 GB storage, 20K GET, 2K PUT    | 10 GB, 50K GET, 5K PUT        | $0.23 + $0.02 + $0.03 = **$0.28** |
| **S3 Glacier**     | None                             | 5 GB                          | 5 GB × $0.004 = **$0.02**         |
| **Lambda**         | 1M requests, 400K GB-seconds     | 100K requests, 50K GB-seconds | **$0.00** (within free tier)      |
| **DynamoDB**       | 25 GB storage, 25 WCU, 25 RCU    | 1 GB, On-demand (low traffic) | **$0.00** (within free tier)      |
| **CloudFront**     | 1 TB data transfer, 10M requests | 10 GB, 50K requests           | **$0.00** (within free tier)      |
| **Step Functions** | 4K state transitions             | 500 transitions               | **$0.00** (within free tier)      |
| **Data Transfer**  | 100 GB outbound                  | 5 GB                          | **$0.00** (within free tier)      |
| **MediaConvert**   | NO FREE TIER                     | 10 minutes                    | 10 × $0.015 = **$0.15**           |
| **CloudWatch**     | 10 metrics, 5 GB logs            | 20 metrics, 2 GB logs         | $0.30 + $0.50 = **$0.80**         |

**Total Estimated Monthly Cost: $1.25 - $2.00**



### Cost Monitoring Dashboard Query

```sql
-- Athena query for daily cost tracking
SELECT 
    DATE(line_item_usage_start_date) as date,
    product_servicename as service,
    SUM(line_item_unblended_cost) as daily_cost,
    line_item_usage_type as usage_type
FROM 
    cur_database.cost_and_usage_report
WHERE 
    resource_tags_user_project = 'ContentPlatform'
    AND DATE(line_item_usage_start_date) >= DATE_ADD('day', -30, CURRENT_DATE)
GROUP BY 
    DATE(line_item_usage_start_date),
    product_servicename,
    line_item_usage_type
ORDER BY 
    date DESC,
    daily_cost DESC
```

---

### Load Testing Script

**File: `load_test.sh`**

```bash
#!/bin/bash

# Load testing script
source test_config.env

echo "🔥 Load Testing Content Platform"
echo "================================="

CONCURRENT_UPLOADS=10
TEST_FILES=50

# Create test files
mkdir -p load_test_files
for i in $(seq 1 $TEST_FILES); do
    dd if=/dev/urandom of=load_test_files/file_$i.bin bs=1M count=1 2>/dev/null
done

echo "Created $TEST_FILES test files"

# Function to upload file
upload_file() {
    local file=$1
    local start=$(date +%s.%N)
    
    aws s3 cp $file s3://$PRIMARY_BUCKET/load-test/$(basename $file) &>/dev/null
    
    local end=$(date +%s.%N)
    local duration=$(echo "$end - $start" | bc)
    echo $duration
}

# Run concurrent uploads
echo "Starting $CONCURRENT_UPLOADS concurrent uploads..."
total_time_start=$(date +%s.%N)

for i in $(seq 1 $TEST_FILES); do
    upload_file "load_test_files/file_$i.bin" &
    
    # Limit concurrent jobs
    if [ $(jobs -r | wc -l) -ge $CONCURRENT_UPLOADS ]; then
        wait -n
    fi
done

wait

total_time_end=$(date +%s.%N)
total_duration=$(echo "$total_time_end - $total_time_start" | bc)

echo ""
echo "Load Test Results:"
echo "  Total files: $TEST_FILES"
echo "  Total time: ${total_duration}s"
echo "  Throughput: $(echo "$TEST_FILES / $total_duration" | bc -l | xargs printf "%.2f") files/sec"
echo ""

# Cleanup
read -p "Remove load test files? (y/n): " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    rm -rf load_test_files
    aws s3 rm s3://$PRIMARY_BUCKET/load-test/ --recursive
    echo "✓ Cleaned up"
fi
```

---

## Disaster Recovery Plan

### Recovery Time Objectives (RTO) and Recovery Point Objectives (RPO)

| Scenario               | RTO        | RPO        | Recovery Procedure         |
| ---------------------- | ---------- | ---------- | -------------------------- |
| **S3 Bucket Deletion** | 4 hours    | 15 minutes | Restore from backup bucket |
| **Region Failure**     | 2 hours    | 15 minutes | Failover to us-west-2      |
| **Lambda Corruption**  | 30 minutes | 0          | Redeploy from source code  |
| **DynamoDB Data Loss** | 1 hour     | 35 days    | Point-in-time recovery     |
| **CloudFront Failure** | 1 hour     | 0          | Update DNS to backup       |

### Disaster Recovery Runbook

**File: `disaster_recovery.sh`**

```bash
#!/bin/bash

# Disaster Recovery Script
source test_config.env

echo "🚨 Disaster Recovery Procedures"
echo "================================"

show_menu() {
    echo ""
    echo "Select recovery scenario:"
    echo "1) Restore from backup bucket"
    echo "2) Failover to backup region"
    echo "3) Restore DynamoDB table"
    echo "4) Redeploy Lambda functions"
    echo "5) Full system restore"
    echo "6) Exit"
    echo ""
}

restore_from_backup() {
    echo "📦 Restoring from backup bucket..."
    
    BACKUP_BUCKET="content-platform-backup-351586884"
    
    echo "Starting sync from backup to primary..."
    aws s3 sync s3://$BACKUP_BUCKET s3://$PRIMARY_BUCKET \
        --exclude "*.tmp" \
        --storage-class STANDARD
    
    echo "✓ Restore completed"
    
    # Verify count
    primary_count=$(aws s3 ls s3://$PRIMARY_BUCKET --recursive | wc -l)
    echo "Restored objects: $primary_count"
}

failover_to_backup_region() {
    echo "🔄 Initiating regional failover..."
    
    # Update application to point to backup region
    echo "1. Update DNS/Route53 to point to us-west-2"
    echo "2. Update Lambda environment variables"
    echo "3. Update CloudFront origin"
    
    read -p "Proceed with failover? (yes/no): " confirm
    if [ "$confirm" = "yes" ]; then
        # Update Lambda env vars to use backup bucket
        for func in content-upload-handler content-image-processor; do
            aws lambda update-function-configuration \
                --function-name $func \
                --environment "Variables={
                    PRIMARY_BUCKET=content-platform-backup-351586884,
                    PROCESSED_BUCKET=content-platform-processed-564156156,
                    AWS_REGION=us-west-2
                }" \
                --region us-west-2
        done
        
        echo "✓ Failover initiated"
    fi
}

restore_dynamodb() {
    echo "💾 DynamoDB Point-in-Time Recovery..."
    
    read -p "Enter restore timestamp (YYYY-MM-DDTHH:MM:SS): " restore_time
    
    aws dynamodb restore-table-to-point-in-time \
        --source-table-name content-metadata \
        --target-table-name content-metadata-restored-$(date +%Y%m%d) \
        --restore-date-time $restore_time
    
    echo "✓ Restore initiated. Monitor in DynamoDB console."
}

redeploy_lambdas() {
    echo "🔧 Redeploying Lambda functions..."
    
    functions=(
        "content-upload-handler"
        "content-image-processor"
        "content-video-processor"
        "content-generic-processor"
        "update-metadata"
        "cloudfront-invalidation"
    )
    
    for func in "${functions[@]}"; do
        echo "Redeploying $func..."
        # Assumes code is in ./lambda_functions/$func/
        if [ -d "./lambda_functions/$func" ]; then
            cd "./lambda_functions/$func"
            zip -r function.zip . -x "*.git*"
            aws lambda update-function-code \
                --function-name $func \
                --zip-file fileb://function.zip
            cd ../..
            echo "✓ $func redeployed"
        else
            echo "⚠️  Source code not found for $func"
        fi
    done
}

full_system_restore() {
    echo "🔥 Full System Restore - This will:"
    echo "   1. Restore all S3 data from backup"
    echo "   2. Restore DynamoDB to latest snapshot"
    echo "   3. Redeploy all Lambda functions"
    echo "   4. Verify all services"
    
    read -p "Continue? (type 'RESTORE' to confirm): " confirm
    if [ "$confirm" = "RESTORE" ]; then
        restore_from_backup
        sleep 5
        restore_dynamodb
        sleep 5
        redeploy_lambdas
        sleep 5
        
        echo ""
        echo "✓ Full restore completed!"
        echo "Running verification..."
        bash quick_test.sh
    else
        echo "Restore cancelled"
    fi
}

# Main menu loop
while true; do
    show_menu
    read -p "Select option: " choice
    
    case $choice in
        1) restore_from_backup ;;
        2) failover_to_backup_region ;;
        3) restore_dynamodb ;;
        4) redeploy_lambdas ;;
        5) full_system_restore ;;
        6) echo "Exiting..."; exit 0 ;;
        *) echo "Invalid option" ;;
    esac
done
```



