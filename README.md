# Laravel Bref

This is an opiniated guide to deploy Laravel with Bref with several specific requirements. For example we use a separate CDK project to deploy pretty much anything else, from Transit Gateway to RDS.

## Install `serverless` command
Install the `serverless` command via NPM:
```
npm install -g serverless
```

Reference: [Installation - Bref](https://bref.sh/docs/installation.html)

## Install Bref and Laravel Bridge

Install bref/laravel-bridge and create initial serverless.yml
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
To access resources in VPC (e.g., RDS), modify cdk-core to create 2 SSM StringList parameters for subnet IDs and security group IDs respectively.
- `/ken/bref/securityGroupIds`: StringList containing security group IDs for lambda
- `/ken/bref/privateSubnetIds`: StringList containing private subnet IDs for lambda

Then refer to them in serverless.yml by adding the following lines:
```
functions:
  web:
    vpc:
      securityGroupIds: ${ssm:/ken/bref/securityGroupIds}
      subnetIds: ${ssm:/ken/bref/privateSubnetIds}
```

Reference: [Database - Bref](https://bref.sh/docs/environment/database.html)

## RDS
For RDS access, store RDS credentials in SSM. Modify cdk-core projects to also store the created credentials in Parameter Store.
- `/ken/rds/username`: RDS username
- `/ken/rds/password`: RDS password
- `/ken/rds/port`: RDS port
- `/ken/rds/database`: RDS database

For projects with separate read and write connections:
- `/ken/rds/readWriteHost`: RDS writer endpoint (or RDS proxy read/write endpoint if you use it)
- `/ken/rds/readOnlyHost`: RDS reader endpoint (or RDS proxy read-only endpoint if you use it)

Otherwise:
- `/ken/rds/host`: RDS host

## Serverless S3 sync
**TODO: Use OAC**

Sync assets to S3 and exclude them from lambda deployment package.

Exclude public, node\_modules, resources/assets, storage, and tests directories.
```
package:
  exclude:
    - public/**
    - node_modules/**
    - resources/assets/**
    - storage/**
    - tests/**
  # Files and directories to exclude from deployment
  include:
    - public/index.php
    - public/robots.txt
    - public/favicon.ico
    - app/**
    - bootstrap/**
    - config/**
    - routes/**
    - vendor/**
    - artisan
```
Install `serverless-s3-sync` plugin.
```
sls plugin install -n serverless-s3-sync
```
Create mappings for CloudFront managed policies.
```
Mappings: 
  CachePolicyIds:
    # https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-managed-cache-policies.html
    Amplify:
      Id: 2e54312d-136d-493c-8eb9-b001f22f67d2
    CachingDisabled:
      Id: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad
    CachingOptimized:
      Id: 658327ea-f89d-4fab-a63d-7e88639e58f6
    CachingOptimizedForUncompressedObjects:
      Id: b2884449-e4de-46a7-ac36-70bc7f1ddd6d
    Elemental-MediaPackage:
      Id: 08627262-05a9-4f76-9ded-b50ca2e3a84f
  OriginRequestPolicyIds:
    # https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-managed-origin-request-policies.html
    AllViewer:
      Id: 216adef6-5c7f-47e4-b989-5492eafa07d3
    AllViewerAndCloudFrontHeaders-2022-06:
      Id: 33f36d7e-f396-46d9-90e0-52428a34d9dc
    AllViewerExceptHostHeader:
      Id: b689b0a8-53d0-40ab-baf2-68738e2966ac
    CORS-CustomOrigin:
      Id: 88a5eaf4-2fd4-4709-b370-b4c650ea3fcf
    Elemental-MediaTailor-PersonalizedManifests:
      Id: 775133bc-15f2-49f9-abea-afb2e0bf67d2
    UserAgentRefererHeaders:
      Id: acba4595-bd28-49b8-b9fe-13317c0390fa
  ResponseHeadersPolicyIds:
    # https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-managed-response-headers-policies.html
    CORS-and-SecurityHeadersPolicy:
      Id: e61eb60c-9c35-4d20-a928-2b84e02af89c
    CORS-With-Preflight:
      Id: 5cc3b908-e619-4b99-88e5-2cf7f45965bd
    CORS-with-preflight-and-SecurityHeadersPolicy:
      Id: eaab4381-ed33-4a86-88ca-d9558dc6cd63
    SecurityHeadersPolicy:
      Id: 67f7725c-6f97-4210-82d7-5512b31e9d03
    SimpleCORS:
      Id: 60669652-455b-4ae9-85a4-c4c02393f86c
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
            CachePolicyId: !FindInMap [ CachePolicyIds, CachingOptimized , Id ]
            OriginRequestPolicyId: !FindInMap [ OriginRequestPolicyIds, AllViewerExceptHostHeader, Id ]
            ResponseHeadersPolicyId: !FindInMap [ ResponseHeadersPolicyIds, CORS-with-preflight-and-SecurityHeadersPolicy, Id ]
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
Notes: 
- Alternatively, you can use `serverless-lift` as [recommended](https://bref.sh/docs/websites.html) by [Bref](https://bref.sh/docs/frameworks/laravel.html#assets), but we don't use it here because some of our deployments are behind VPN access and some use a different domain for assets.
- For API, skip this step.

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
Use DynamoDB for cache and session.

Modify cdk-core project to create a DynamoDB table with partition key called `key` and TTL attribute called `expires_at`. And modify the following lines in your .env file:
```
CACHE_DRIVER=dynamodb
SESSION_DRIVER=dynamodb
SESSION_STORE=dynamodb
```

## Queue
**TODO: Experiment with `serverless-lift`**

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

## Automatic pruning
Serverless deployment creates deployment package and uploads it to lambda. Each deployment creates a new version without deleting the old ones. This can be problematic for accounts with many lambdas (especially monolithic lambdas like this one) because lambda has a code storage limit of 75GB. Use serverless-prune-plugin plugin delete old versions.

Install `serverless-prune-plugin` plugin.
```
sls plugin install -n serverless-prune-plugin
```
Add plugin configuration to serverless.yml
```
custom:
  prune:
    automatic: true
    number: 3
```

Reference: [serverless-prune-plugin](https://www.serverless.com/plugins/serverless-prune-plugin)

## Notes
- [Serverless offline](https://www.serverless.com/plugins/serverless-offline) doesn't work, for local development refer to [Local development for web apps](https://bref.sh/docs/web-apps/local-development.html).
- `sls dev` also doesn't work.
- You can use [serverless-s3-local](https://github.com/ar90n/serverless-s3-local), [serverless-dynamodb-local](https://www.serverless.com/plugins/serverless-dynamodb-local) and [serverless-offline-sqs](https://www.npmjs.com/package/serverless-offline-sqs) for local development.
