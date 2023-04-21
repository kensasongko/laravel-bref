# How to
## Install Bref and Laravel Bridge

Install bref/laravel-bridge and setup serverless.yml
```
composer require bref/bref bref/laravel-bridge --update-with-dependencies
php artisan vendor:publish --tag=serverless-config
```
Reference: [Serverless Laravel applications - Bref](https://bref.sh/docs/frameworks/laravel.html)

## ARM architecture
Use ARM, cheaper and better performance.
```
functions:
  web:
    architecture: arm64
```

Reference: [PHP runtimes for AWS Lambda - Bref](https://bref.sh/docs/runtimes/#arm-runtimes)

## VPC
To access resources in VPC (e.g., RDS), create 2 SSM StringList parameters for subnet IDs and security group IDs respectively, then add refer to them in serverless.yml by adding the following lines:
```
functions:
  web:
    vpc:
      securityGroupIds: ${ssm:/ken/bref/securityGroupIds}
      subnetIds: ${ssm:/ken/bref/privateSubnetIds}
```

Reference: [Database - Bref](https://bref.sh/docs/environment/database.html)

## Serverless S3 sync
Install Serverless S3 sync
```
npm install --save serverless-s3-sync
```
Add Serverless S3 Sync plugin to serverless.yml
```
plugins:
  - serverless-s3-sync
```
Add S3 bucket and CloudFront distribution resources to serverless.yml
```
resources:
  Resources:
    AssetsBucket:
      Type: AWS::S3::Bucket
      Properties:
        BucketName: ken-laravel-static-assets
    AssetsCdn:
      Type: AWS::CloudFront::Distribution
      Properties:
        DistributionConfig:
          Enabled: true
          PriceClass: PriceClass_100
          HttpVersion: http2
          Origins:
            - Id: AssetsBucket
              DomainName: !GetAtt AssetsBucket.RegionalDomainName
              S3OriginConfig: {}
          DefaultCacheBehavior:
            AllowedMethods: [GET, HEAD]
            TargetOriginId: AssetsBucket
            CachePolicyId: 658327ea-f89d-4fab-a63d-7e88639e58f6
            # https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-managed-cache-policies.html
            OriginRequestPolicyId: b689b0a8-53d0-40ab-baf2-68738e2966ac
            # https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-cloudfront-distribution-cachebehavior.html
            ResponseHeadersPolicyId: 5cc3b908-e619-4b99-88e5-2cf7f45965bd
            # https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-managed-response-headers-policies.html
            ViewerProtocolPolicy: redirect-to-https
```
Sync public directory to the assets S3 bucket
```
custom:
  s3Sync:
    # Sync public directory to the assets S3 bucket
    - bucketName: ken-laravel-static-assets
      localDir: public
      deleteRemoved: true
      acl: public-read
```
Configure `ASSET_URL` environment variables
```
provider:
  environment:
    ASSET_URL: !Sub 'https://${AssetsCdn.DomainName}'
```
Note: Alternatively, you can use `serverless-lift` as [recommended](https://bref.sh/docs/websites.html) by [Bref](https://bref.sh/docs/frameworks/laravel.html#assets), but we don't use it here because some of our deployments are behind VPN access.

Reference: [Serverless S3 Sync](https://www.serverless.com/plugins/serverless-s3-sync)

## IAM
Create role named `lambda-laravel-role` with the following permission (this is not the minimum required permission):
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "cloudfront:CreateInvalidation",
                "cloudwatch:GetMetricStatistics",
                "ec2:CreateNetworkInterface",
                "ec2:DeleteNetworkInterface",
                "ec2:DescribeNetworkInterfaces",
                "ec2:AssignPrivateIpAddresses",
                "ec2:UnassignPrivateIpAddresses"
                "logs:CreateLogGroup",
                "logs:CreateLogStream",
                "logs:FilterLogEvents",
                "logs:PutLogEvents",
                "kms:Decrypt",
                "secretsmanager:GetSecretValue",
                "ssm:GetParameters",
                "ssm:GetParameter",
                "lambda:invokeFunction",
                "s3:*",
                "ses:*",
                "sqs:*",
                "dynamodb:*",
                "route53domains:*"
            ],
            "Effect": "Allow",
            "Resource": "*"
        }
    ]
}
```
Use the newly created role by adding the following lines:
```
provider:
  iam:
    role: !Sub 'arn:aws:iam::${AWS::AccountId}:role/lambda-laravel-role'
```

Reference: [Serverless Framework - IAM Permissions For Functions](https://www.serverless.com/framework/docs/providers/aws/guide/iam)

## Cache and Session
Create a DynamoDB table with partition key called `key` and TTL attribute called `expires_at`. And modify the following lines in your .env file:
```
CACHE_DRIVER=dynamodb
SESSION_DRIVER=dynamodb
SESSION_STORE=dynamodb
```

## Queue
Create scheduler:
```
functions:
  artisan:
      handler: artisan
      runtime: php-81-console
      architecture: arm64
      timeout: 720 # in seconds
      # Uncomment to also run the scheduler every minute
      vpc:
        securityGroupIds: ${ssm:/ken/bref/securityGroupIds}
        subnetIds: ${ssm:/ken/bref/privateSubnetIds}
      events:
        - schedule:
            rate: rate(30 minutes)
            input: '"schedule:run"'
```

Create SQS queue resource:
```
resources:
  Resources:
    Queue:
      Type: AWS::SQS::Queue
      Properties:
        QueueName: ken-laravel-queue
        ReceiveMessageWaitTimeSeconds: 20
        VisibilityTimeout: 120
```

## Notes:
- [Serverless offline](https://www.serverless.com/plugins/serverless-offline) doesn't work, for local development refer to [Local development for web apps](https://bref.sh/docs/web-apps/local-development.html).
- Serverless dev also doesn't work.
