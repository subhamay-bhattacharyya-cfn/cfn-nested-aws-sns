# CloudFormation SNS Topic Template Repository

<!-- Row 1: Status - Most Important -->
[![Release](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns/actions/workflows/release.yaml/badge.svg)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns)&nbsp;[![GitHub Repo](https://img.shields.io/badge/GitHub-Repository-blue?logo=github)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns)&nbsp;[![Issues](https://img.shields.io/github/issues/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns/issues)&nbsp;[![Last Commit](https://img.shields.io/github/last-commit/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns/commits)

<!-- Row 2: Code Quality -->
[![Top Language](https://img.shields.io/github/languages/top/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns)&nbsp;[![Commits](https://img.shields.io/github/commit-activity/t/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns/commits)

<!-- Row 3: Tech Stack -->
[![CloudFormation](https://img.shields.io/badge/CloudFormation-IaC-orange?logo=amazon&logoColor=white)](https://aws.amazon.com/cloudformation/)&nbsp;[![SNS](https://img.shields.io/badge/SNS-Pub%2FSub-brightgreen?logo=amazon&logoColor=white)](https://aws.amazon.com/sns/)&nbsp;[![Built with Claude Code](https://img.shields.io/badge/Built_with-Claude_Code-D97757?logo=anthropic&logoColor=white)](https://claude.ai/)

<!-- Row 4: Repository Info -->
[![Files](https://img.shields.io/github/directory-file-count/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns)&nbsp;[![Repo Size](https://img.shields.io/github/repo-size/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns)&nbsp;[![Release Date](https://img.shields.io/github/release-date/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns)](https://github.com/subhamay-bhattacharyya-cfn/cfn-nested-aws-sns/releases)

<!-- Row 5: Custom Metrics -->
[![Custom Endpoint](https://img.shields.io/endpoint?url=https://gist.githubusercontent.com/bsubhamay/15cfe9b1efbc73710f3f39ffc2d1caab/raw/cfn-nested-aws-sns.json)](https://gist.github.com/subhamay-bhattacharyya/15cfe9b1efbc73710f3f39ffc2d1caab)

This repository contains a nested CloudFormation template for creating **Amazon SNS topics** with standard or FIFO type, consistent naming, optional KMS encryption, and optional cross-account access.

## Overview

This is a **nested stack template** designed to be invoked from a parent/root CloudFormation stack. The template is stored in this repository and should be uploaded to an S3 bucket for reference by parent stacks.

## Template Files

### CloudFormation Template

- **`cloudformation/template.yaml`** — Nested template for SNS topic creation with support for:
  - Standard or FIFO topics
  - Predictable topic naming with optional CI suffix
  - KMS encryption with a customer-managed key
  - Cross-account publish/subscribe through a topic policy

### Configuration Files

- **`cloudformation/parameters.json`** — Parameter values used by CI (`Environment`, `CiSuffix`)
- **`cloudformation/stack-config.json`** — Stack name, template file, and parameter file used by CI

## Template Features

- ✅ **Standard or FIFO Topics** — Selected with `TopicType`
- ✅ **FIFO Content-Based Deduplication** — Optional, FIFO topics only
- ✅ **Smart Topic Naming** — Project prefix, base name, environment, region, with optional CI suffix
- ✅ **KMS Encryption** — Pass a key alias, ARN, or key ID; empty leaves the topic unencrypted
- ✅ **Cross-Account Access** — Topic policy granting `sns:Publish` and/or `sns:Subscribe` to listed account IDs
- ✅ **Export Values** — Topic ARN and name exported for cross-stack references

## Parameters

### Topic Naming & Environment

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `ProjectName` | String | `proj-ztc` | Project name prefix (lowercase letters, numbers, hyphens; max 20 characters) |
| `SnsTopicBaseName` | String | `sns-topic` | Base name for the topic (letters, numbers, hyphens, underscores; max 60 characters) |
| `Environment` | String | `devl` | Deployment environment (lowercase letters, numbers, hyphens) |
| `CiSuffix` | String | `""` | Optional suffix appended to the topic name (e.g., pipeline ID; max 30 characters) |

### Topic Type

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `TopicType` | String | `Standard` | `Standard` or `FIFO`. FIFO topic names get the `.fifo` suffix |
| `ContentBasedDeduplication` | String | `false` | Enable content-based deduplication (`true` or `false`). Only applies to FIFO topics |

### Encryption

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `KmsKeyId` | String | `""` | KMS key to encrypt the topic. Accepts an alias (`alias/my-key`), a key ARN, a key ID, or a multi-Region key ID (`mrk-...`). Empty means no server-side encryption |

### Cross-Account Access

| Parameter | Type | Default | Description |
| ----------- | ------ | --------- | ------------- |
| `CrossAccountIds` | String | `""` | Comma-separated 12-digit AWS account IDs to grant access (e.g., `111122223333,444455556666`). Empty creates no topic policy |
| `CrossAccountActions` | String | `sns:Publish` | Actions granted to `CrossAccountIds`: `sns:Publish`, `sns:Subscribe`, or `sns:Publish,sns:Subscribe` |

## Outputs

| Output | Type | Description |
| ----------- | ------ | ------------- |
| `TopicArn` | String | ARN of the SNS topic (exported as `<StackName>-TopicArn`) |
| `TopicName` | String | Name of the SNS topic (exported as `<StackName>-TopicName`) |
| `CrossAccountPolicyApplied` | String | `true` if a cross-account topic policy was created, otherwise `false` |

## Usage

### 1. Upload Template to S3

```bash
aws s3 cp cloudformation/template.yaml s3://your-cfn-bucket/templates/sns-topic.yaml
```

### 2. Reference from Parent Stack

In your parent/root CloudFormation template:

```yaml
SnsTopicNestedStack:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: https://s3.amazonaws.com/your-cfn-bucket/templates/sns-topic.yaml
    Parameters:
      ProjectName: !Ref ProjectName
      SnsTopicBaseName: alerts
      Environment: !Ref Environment
      TopicType: FIFO
      ContentBasedDeduplication: "true"
      KmsKeyId: alias/my-sns-key
      CrossAccountIds: 111122223333
      CrossAccountActions: sns:Publish
    Tags:
      - Key: Environment
        Value: !Ref Environment

Outputs:
  TopicArn:
    Value: !GetAtt SnsTopicNestedStack.Outputs.TopicArn
  TopicName:
    Value: !GetAtt SnsTopicNestedStack.Outputs.TopicName
```

### 3. Deploy Using AWS CLI

`aws cloudformation deploy` takes parameter overrides as `KEY=VALUE` pairs.

#### Example 1: Basic Standard Topic

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name myapp-sns-dev \
  --parameter-overrides \
    ProjectName=myapp \
    SnsTopicBaseName=alerts \
    Environment=devl \
  --region us-east-1
```

#### Example 2: FIFO Topic with Content-Based Deduplication

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name myapp-orders-prod \
  --parameter-overrides \
    ProjectName=myapp \
    SnsTopicBaseName=orders \
    Environment=prod \
    TopicType=FIFO \
    ContentBasedDeduplication=true \
  --region us-east-1
```

#### Example 3: Customer-Managed KMS Encryption

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name myapp-sns-secure \
  --parameter-overrides \
    ProjectName=myapp \
    SnsTopicBaseName=sensitive-events \
    Environment=prod \
    KmsKeyId=arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012 \
  --region us-east-1
```

#### Example 4: Cross-Account Publish Access

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name myapp-sns-shared \
  --parameter-overrides \
    ProjectName=myapp \
    SnsTopicBaseName=shared-events \
    Environment=prod \
    CrossAccountIds=111122223333,444455556666 \
    CrossAccountActions=sns:Publish \
  --region us-east-1
```

To update cross-account access later, change `CrossAccountIds` or `CrossAccountActions` and redeploy the same stack. The topic policy is updated in place.

#### Example 5: CI Suffix for Ephemeral Deployments

```bash
aws cloudformation deploy \
  --template-file cloudformation/template.yaml \
  --stack-name myapp-sns-ci \
  --parameter-overrides \
    ProjectName=myapp \
    SnsTopicBaseName=test-topic \
    Environment=devl \
    CiSuffix=$CI_PIPELINE_ID \
  --region us-east-1
```

#### Monitoring Stack Creation

```bash
# Get stack outputs
aws cloudformation describe-stacks \
  --stack-name myapp-sns-dev \
  --query 'Stacks[0].Outputs' \
  --output table

# Get topic details
TOPIC_ARN=$(aws cloudformation describe-stacks \
  --stack-name myapp-sns-dev \
  --query 'Stacks[0].Outputs[?OutputKey==`TopicArn`].OutputValue' \
  --output text)

aws sns get-topic-attributes --topic-arn $TOPIC_ARN
```

## Topic Naming Convention

**Without CI Suffix:**

```bash
{ProjectName}-{SnsTopicBaseName}-{Environment}-{Region}
```

Example: `myapp-alerts-devl-us-east-1`

**With CI Suffix:**

```bash
{ProjectName}-{SnsTopicBaseName}-{Environment}-{Region}-{CiSuffix}
```

Example: `myapp-alerts-devl-us-east-1-pipeline-12345`

**FIFO topics** add `.fifo` to the end, as AWS requires for FIFO topic names:

Example: `myapp-orders-prod-us-east-1.fifo`

## Notes on Encryption and Cross-Account Access

- A KMS-encrypted topic needs the publisher to have `kms:GenerateDataKey` on the key, and the subscriber to have `kms:Decrypt`. Cross-account principals need access in the **KMS key policy** as well as the topic policy.
- The cross-account topic policy is created only when `CrossAccountIds` is set. Leaving it empty keeps the default topic policy, which allows only the owning account.

## Best Practices Implemented

- ✅ Predictable topic naming with project prefix, environment, and region
- ✅ Optional CI suffix support for ephemeral test deployments
- ✅ Optional KMS encryption with customer-managed keys
- ✅ FIFO topics with optional content-based deduplication
- ✅ Explicit, parameter-driven cross-account access
- ✅ Automatic tagging for resource management
- ✅ Export values for cross-stack references

## License

MIT
