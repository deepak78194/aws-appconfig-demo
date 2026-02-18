# Complete AWS CLI Commands Reference for AWS AppConfig

## Table of Contents

1. [Introduction](#1-introduction)
2. [CloudFormation Commands](#2-cloudformation-commands)
3. [AppConfig Resource Commands](#3-appconfig-resource-commands)
4. [Configuration Version Commands](#4-configuration-version-commands)
5. [Deployment Commands](#5-deployment-commands)
6. [Retrieve Configuration Commands](#6-retrieve-configuration-commands)
7. [Complete Workflow Scripts](#7-complete-workflow-scripts)
8. [Troubleshooting Commands](#8-troubleshooting-commands)
9. [Advanced Commands](#9-advanced-commands)
10. [Quick Reference](#10-quick-reference)
11. [Tips and Best Practices](#11-tips-and-best-practices)
12. [Real-World Examples](#12-real-world-examples)

---

## 1. Introduction

### Purpose

This document provides a comprehensive reference for all AWS CLI commands related to AWS AppConfig management. It serves as a complete guide for developers and DevOps engineers who need to work with AppConfig infrastructure and configuration deployments.

### Prerequisites

Before using these commands, ensure you have:

✅ **AWS CLI v2** installed and configured
```bash
# Check AWS CLI version
aws --version

# Install AWS CLI v2 (Linux)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
```

✅ **AWS Credentials** configured
```bash
# Configure AWS credentials
aws configure

# Or verify existing credentials
aws sts get-caller-identity
```

✅ **jq utility** for JSON processing
```bash
# Install jq (Ubuntu/Debian)
sudo apt-get install jq

# Install jq (macOS)
brew install jq

# Install jq (Amazon Linux/RHEL/CentOS)
sudo yum install jq

# Verify installation
jq --version
```

✅ **Required AWS Permissions**
- CloudFormation: CreateStack, UpdateStack, DescribeStacks, DeleteStack
- AppConfig: All operations (Create*, Start*, Get*, List*, Delete*)
- IAM: PassRole (if using service roles)

### Variable Conventions

Throughout this document, the following variable conventions are used:

| Variable | Description | Example Value |
|----------|-------------|---------------|
| `ASSET_ID` | Numeric identifier for the service | `10` |
| `ASSET_NAME` | Short name for the service | `auth` |
| `ASSET_GROUP` | Environment identifier | `cdc1c`, `pdc1c` |
| `ASSET_SERVICE` | Service name | `authapi` |
| `STACK_NAME` | CloudFormation stack name | `appconfig-auth-10-cdc1c` |
| `APP_ID` | AppConfig Application ID | `abc123` |
| `ENV_ID` | AppConfig Environment ID | `def456` |
| `PROFILE_ID` | Configuration Profile ID | `ghi789` |
| `STRATEGY_ID` | Deployment Strategy ID | `jkl012` |
| `VERSION_NUMBER` | Configuration version number | `1`, `2`, `3` |
| `DEPLOYMENT_NUMBER` | Deployment number | `1`, `2`, `3` |
| `AWS_REGION` | AWS Region | `us-east-1` |

**Note**: Replace these placeholder values with your actual resource IDs when running commands.

---

## 2. CloudFormation Commands

### 2.1 Create Stack (First Time)

🔥 **Create new AppConfig infrastructure for the first time**

```bash
# Full command with all parameters
aws cloudformation create-stack \
  --stack-name appconfig-auth-10-cdc1c \
  --template-body file://production-approach/infra/appconfig-service.yml \
  --parameters \
    ParameterKey=AssetId,ParameterValue=10 \
    ParameterKey=AssetName,ParameterValue=auth \
    ParameterKey=AssetGroup,ParameterValue=cdc1c \
    ParameterKey=AssetService,ParameterValue=authapi \
    ParameterKey=DeployInitialConfig,ParameterValue=no \
  --region us-east-1 \
  --tags \
    Key=AssetId,Value=10 \
    Key=AssetName,Value=auth \
    Key=AssetGroup,Value=cdc1c \
    Key=Service,Value=authapi \
    Key=ManagedBy,Value=CLI \
    Key=Environment,Value=Development
```

**Parameter Explanations:**

- **AssetId**: Unique numeric identifier for your service (e.g., `10`)
- **AssetName**: Short descriptive name (e.g., `auth`, `payment`)
- **AssetGroup**: Environment/cluster identifier (e.g., `cdc1c` for dev, `pdc1c` for prod)
- **AssetService**: Specific service name (e.g., `authapi`, `paymentapi`)
- **DeployInitialConfig**: Whether to deploy initial configuration (`yes` or `no`)

**What This Creates:**

- AppConfig Application: `auth-10`
- AppConfig Environment: `cdc1c`
- Configuration Profile: `authapi-config`
- Deployment Strategy (Immediate): `auth-10-Immediate`
- Deployment Strategy (Canary): `auth-10-Canary`

### 2.2 Create Stack with Initial Configuration

Create infrastructure AND deploy initial configuration together:

```bash
# Read configuration file first
CONFIG_CONTENT=$(cat production-approach/config/cdc1c.json | jq -c .)

# Create stack with initial configuration
aws cloudformation create-stack \
  --stack-name appconfig-auth-10-cdc1c \
  --template-body file://production-approach/infra/appconfig-service.yml \
  --parameters \
    ParameterKey=AssetId,ParameterValue=10 \
    ParameterKey=AssetName,ParameterValue=auth \
    ParameterKey=AssetGroup,ParameterValue=cdc1c \
    ParameterKey=AssetService,ParameterValue=authapi \
    ParameterKey=DeployInitialConfig,ParameterValue=yes \
    ParameterKey=InitialConfigContent,ParameterValue="${CONFIG_CONTENT}" \
  --region us-east-1 \
  --tags \
    Key=AssetId,Value=10 \
    Key=AssetName,Value=auth \
    Key=ManagedBy,Value=CLI
```

**Use Case**: When you want to create infrastructure and have a default configuration ready immediately, without needing a separate deployment step.

### 2.3 Update Stack

⚠️ **Modify existing infrastructure (use with caution)**

```bash
# Update stack with new parameter values
aws cloudformation update-stack \
  --stack-name appconfig-auth-10-cdc1c \
  --template-body file://production-approach/infra/appconfig-service.yml \
  --parameters \
    ParameterKey=AssetId,UsePreviousValue=true \
    ParameterKey=AssetName,UsePreviousValue=true \
    ParameterKey=AssetGroup,UsePreviousValue=true \
    ParameterKey=AssetService,UsePreviousValue=true \
    ParameterKey=DeployInitialConfig,ParameterValue=no \
  --region us-east-1
```

**When to Use**:
- Modifying deployment strategy settings in the CloudFormation template
- Updating resource configurations or tags
- Changing validators or other AppConfig settings

**Note**: Use `UsePreviousValue=true` to keep existing parameter values unchanged.

### 2.4 Deploy Stack (Create or Update - Recommended)

✓ **The preferred command for idempotent stack operations**

```bash
# Deploy command - creates if not exists, updates if exists
aws cloudformation deploy \
  --stack-name appconfig-auth-10-cdc1c \
  --template-file production-approach/infra/appconfig-service.yml \
  --parameter-overrides \
    AssetId=10 \
    AssetName=auth \
    AssetGroup=cdc1c \
    AssetService=authapi \
    DeployInitialConfig=no \
  --tags \
    AssetId=10 \
    AssetName=auth \
    AssetGroup=cdc1c \
    ManagedBy=CLI \
  --region us-east-1 \
  --no-fail-on-empty-changeset
```

**Why This is Preferred**:
- **Idempotent**: Safe to run multiple times
- **Automatic**: Creates or updates as needed
- **No-fail flag**: Won't fail if there are no changes
- **Change set preview**: Shows what will change before applying
- **Simpler syntax**: Uses `--parameter-overrides` instead of complex `ParameterKey=X,ParameterValue=Y` format

### 2.5 Describe Stack

#### Get Stack Status

```bash
# Get overall stack status
aws cloudformation describe-stacks \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --query 'Stacks[0].StackStatus' \
  --output text

# Output: CREATE_COMPLETE, UPDATE_COMPLETE, etc.
```

#### Get All Stack Outputs

```bash
# Get all outputs as JSON
aws cloudformation describe-stacks \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --query 'Stacks[0].Outputs' \
  --output json

# Get all outputs as table (human-readable)
aws cloudformation describe-stacks \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --query 'Stacks[0].Outputs[*].[OutputKey,OutputValue,Description]' \
  --output table
```

#### Get Specific Output by Name

```bash
# Get Application ID
aws cloudformation describe-stacks \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --query 'Stacks[0].Outputs[?OutputKey==`ApplicationId`].OutputValue' \
  --output text

# Get Environment ID
aws cloudformation describe-stacks \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --query 'Stacks[0].Outputs[?OutputKey==`EnvironmentId`].OutputValue' \
  --output text

# Get Configuration Profile ID
aws cloudformation describe-stacks \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --query 'Stacks[0].Outputs[?OutputKey==`ConfigurationProfileId`].OutputValue' \
  --output text

# Get Immediate Deployment Strategy ID
aws cloudformation describe-stacks \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --query 'Stacks[0].Outputs[?OutputKey==`DeploymentStrategyImmediateId`].OutputValue' \
  --output text

# Get Canary Deployment Strategy ID
aws cloudformation describe-stacks \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --query 'Stacks[0].Outputs[?OutputKey==`DeploymentStrategyCanaryId`].OutputValue' \
  --output text
```

#### Query Examples with Different Formats

```bash
# Get outputs with jq (alternative method)
aws cloudformation describe-stacks \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --output json | jq -r '.Stacks[0].Outputs'

# Get outputs as key-value pairs
aws cloudformation describe-stacks \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --query 'Stacks[0].Outputs[*].[OutputKey,OutputValue]' \
  --output text
```

### 2.6 Delete Stack

⚠️ **DANGER: This permanently deletes all AppConfig resources**

```bash
# Delete stack
aws cloudformation delete-stack \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1

# Wait for deletion to complete
aws cloudformation wait stack-delete-complete \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1
```

**⚠️ Warning**:
- This deletes the Application, Environment, Configuration Profile, and Deployment Strategies
- Configuration versions and deployment history will be lost
- This action cannot be undone
- Applications will no longer be able to fetch configurations

**When to Use**:
- Decommissioning a service completely
- Cleaning up test/dev environments
- Starting fresh after testing

### 2.7 Wait Commands

Wait for CloudFormation operations to complete:

#### Wait for Stack Creation

```bash
# Wait for stack creation (blocks until complete or timeout)
aws cloudformation wait stack-create-complete \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1

# Returns when stack is CREATE_COMPLETE or fails after timeout
```

#### Wait for Stack Update

```bash
# Wait for stack update
aws cloudformation wait stack-update-complete \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1
```

#### Wait for Stack Deletion

```bash
# Wait for stack deletion
aws cloudformation wait stack-delete-complete \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1
```

**Timeout Considerations**:
- Default timeout: 120 attempts × 30 seconds = 60 minutes
- Stack operations typically complete within 5-10 minutes
- Use `--no-paginate` to avoid excessive API calls in scripts

### 2.8 List and Query Stacks

#### List All Stacks

```bash
# List all stacks (active and deleted)
aws cloudformation list-stacks \
  --region us-east-1 \
  --output table

# List only active stacks
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --region us-east-1 \
  --output table
```

#### Filter Stacks by Name Pattern

```bash
# Find all appconfig stacks
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --region us-east-1 \
  --query 'StackSummaries[?contains(StackName, `appconfig`)].[StackName,StackStatus,CreationTime]' \
  --output table

# Find stacks for specific asset (e.g., auth-10)
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --region us-east-1 \
  --query 'StackSummaries[?contains(StackName, `auth-10`)].[StackName,StackStatus]' \
  --output table
```

#### Get Stack Events

```bash
# Get recent stack events (last 20)
aws cloudformation describe-stack-events \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --max-items 20 \
  --query 'StackEvents[*].[Timestamp,ResourceStatus,ResourceType,LogicalResourceId]' \
  --output table

# Get only failed events
aws cloudformation describe-stack-events \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --query 'StackEvents[?contains(ResourceStatus, `FAILED`)].[Timestamp,ResourceStatus,ResourceStatusReason,LogicalResourceId]' \
  --output table
```

---

## 3. AppConfig Resource Commands

### 3.1 Applications

#### List All Applications

```bash
# List all AppConfig applications
aws appconfig list-applications \
  --region us-east-1 \
  --output table

# List with JSON output
aws appconfig list-applications \
  --region us-east-1 \
  --output json | jq -r '.Items[] | "\(.Id) - \(.Name)"'
```

#### Get Application Details

```bash
# Get details of a specific application
aws appconfig get-application \
  --application-id abc123 \
  --region us-east-1 \
  --output json

# Get just the application name
aws appconfig get-application \
  --application-id abc123 \
  --region us-east-1 \
  --query 'Name' \
  --output text
```

#### Create Application (Manual - Not Using CloudFormation)

```bash
# Create application manually (if not using CloudFormation)
aws appconfig create-application \
  --name "auth-10" \
  --description "Authentication service AppConfig application" \
  --tags AssetId=10,AssetName=auth,ManagedBy=Manual \
  --region us-east-1 \
  --output json
```

**Note**: In production, use CloudFormation instead for infrastructure as code.

#### Delete Application

```bash
# Delete application (this will fail if environments exist)
aws appconfig delete-application \
  --application-id abc123 \
  --region us-east-1
```

⚠️ **Warning**: You must delete all environments first before deleting an application.

### 3.2 Environments

#### List Environments for an Application

```bash
# List all environments in an application
aws appconfig list-environments \
  --application-id abc123 \
  --region us-east-1 \
  --output table

# List with formatted output
aws appconfig list-environments \
  --application-id abc123 \
  --region us-east-1 \
  --output json | jq -r '.Items[] | "\(.Id) - \(.Name) (\(.State))"'
```

#### Get Environment Details

```bash
# Get details of a specific environment
aws appconfig get-environment \
  --application-id abc123 \
  --environment-id def456 \
  --region us-east-1 \
  --output json
```

#### Create Environment (Manual)

```bash
# Create environment manually
aws appconfig create-environment \
  --application-id abc123 \
  --name "cdc1c" \
  --description "Development environment CDC1C" \
  --tags AssetGroup=cdc1c,Environment=Development \
  --region us-east-1 \
  --output json
```

#### Delete Environment

```bash
# Delete environment
aws appconfig delete-environment \
  --application-id abc123 \
  --environment-id def456 \
  --region us-east-1
```

### 3.3 Configuration Profiles

#### List Configuration Profiles

```bash
# List all configuration profiles for an application
aws appconfig list-configuration-profiles \
  --application-id abc123 \
  --region us-east-1 \
  --output table

# List with details
aws appconfig list-configuration-profiles \
  --application-id abc123 \
  --region us-east-1 \
  --output json | jq -r '.Items[] | "\(.Id) - \(.Name) (\(.Type))"'
```

#### Get Configuration Profile Details

```bash
# Get configuration profile details
aws appconfig get-configuration-profile \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --region us-east-1 \
  --output json

# View validators
aws appconfig get-configuration-profile \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --region us-east-1 \
  --query 'Validators' \
  --output json | jq .
```

#### Create Configuration Profile (Manual)

```bash
# Create hosted configuration profile
aws appconfig create-configuration-profile \
  --application-id abc123 \
  --name "authapi-config" \
  --description "Configuration for authapi service" \
  --location-uri "hosted" \
  --type "AWS.AppConfig.FeatureFlags" \
  --region us-east-1 \
  --output json

# Create with JSON schema validator
aws appconfig create-configuration-profile \
  --application-id abc123 \
  --name "authapi-config" \
  --location-uri "hosted" \
  --type "AWS.AppConfig.FeatureFlags" \
  --validators '[{
    "Type": "JSON_SCHEMA",
    "Content": "{\"$schema\":\"http://json-schema.org/draft-07/schema#\",\"type\":\"object\",\"required\":[\"featureFlags\",\"settings\"]}"
  }]' \
  --region us-east-1 \
  --output json
```

#### Delete Configuration Profile

```bash
# Delete configuration profile
aws appconfig delete-configuration-profile \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --region us-east-1
```

### 3.4 Deployment Strategies

#### List Deployment Strategies

```bash
# List all deployment strategies (AWS managed + custom)
aws appconfig list-deployment-strategies \
  --region us-east-1 \
  --output table

# List only custom strategies (filter out AWS managed)
aws appconfig list-deployment-strategies \
  --region us-east-1 \
  --output json | jq -r '.Items[] | select(.ReplicateTo == "NONE") | "\(.Id) - \(.Name)"'
```

#### Get Deployment Strategy Details

```bash
# Get strategy details
aws appconfig get-deployment-strategy \
  --deployment-strategy-id jkl012 \
  --region us-east-1 \
  --output json

# View strategy configuration
aws appconfig get-deployment-strategy \
  --deployment-strategy-id jkl012 \
  --region us-east-1 \
  --query '[DeploymentDurationInMinutes,GrowthFactor,FinalBakeTimeInMinutes,GrowthType]' \
  --output table
```

#### List AWS Managed vs Custom Strategies

```bash
# List AWS managed strategies (ReplicateTo = SSM_DOCUMENT)
aws appconfig list-deployment-strategies \
  --region us-east-1 \
  --output json | jq -r '.Items[] | select(.ReplicateTo != "NONE") | "\(.Id) - \(.Name) (AWS Managed)"'

# List custom strategies (ReplicateTo = NONE)
aws appconfig list-deployment-strategies \
  --region us-east-1 \
  --output json | jq -r '.Items[] | select(.ReplicateTo == "NONE") | "\(.Id) - \(.Name) (Custom)"'
```

---

## 4. Configuration Version Commands

### 4.1 Create Hosted Configuration Version

🔥 **PRIMARY COMMAND - Create configuration version from file**

```bash
# Create version from file (MOST COMMON)
aws appconfig create-hosted-configuration-version \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --content file://production-approach/config/cdc1c.json \
  --content-type "application/json" \
  --description "Deployed from Git commit a1b2c3d" \
  --region us-east-1 \
  --output json

# Store version number in variable
VERSION_NUMBER=$(aws appconfig create-hosted-configuration-version \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --content file://production-approach/config/cdc1c.json \
  --content-type "application/json" \
  --description "Deployed from Git" \
  --region us-east-1 \
  --output json | jq -r '.VersionNumber')

echo "Created version: $VERSION_NUMBER"
```

#### Create Version with Inline Content

```bash
# Create version with inline JSON (useful for small configs)
aws appconfig create-hosted-configuration-version \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --content '{"featureFlags":{"enableNewAuth":true},"settings":{"timeout":3600}}' \
  --content-type "application/json" \
  --description "Quick config update" \
  --region us-east-1 \
  --output json
```

#### Add Description with Metadata

```bash
# Include build number, git commit, and timestamp
GIT_COMMIT=$(git rev-parse --short HEAD)
BUILD_NUMBER="123"
TIMESTAMP=$(date -u +"%Y-%m-%d %H:%M:%S UTC")

aws appconfig create-hosted-configuration-version \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --content file://production-approach/config/cdc1c.json \
  --content-type "application/json" \
  --description "Build: $BUILD_NUMBER | Commit: $GIT_COMMIT | Deployed: $TIMESTAMP" \
  --region us-east-1 \
  --output json
```

#### Best Practices for Descriptions

```bash
# Include deployment metadata for audit trail
DESCRIPTION="Env: cdc1c | Commit: $(git rev-parse --short HEAD) | Author: $(git log -1 --pretty=format:'%an') | Date: $(date -u +%Y-%m-%d)"

aws appconfig create-hosted-configuration-version \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --content file://production-approach/config/cdc1c.json \
  --content-type "application/json" \
  --description "$DESCRIPTION" \
  --region us-east-1 \
  --output json
```

### 4.2 List Configuration Versions

```bash
# List all versions for a configuration profile
aws appconfig list-hosted-configuration-versions \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --region us-east-1 \
  --output table

# List versions with details (formatted)
aws appconfig list-hosted-configuration-versions \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --region us-east-1 \
  --output json | jq -r '.Items[] | "Version \(.VersionNumber): \(.Description)"'
```

#### Sort by Version Number

```bash
# Get latest version number
aws appconfig list-hosted-configuration-versions \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --region us-east-1 \
  --query 'Items[0].VersionNumber' \
  --output text

# Get all version numbers sorted
aws appconfig list-hosted-configuration-versions \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --region us-east-1 \
  --query 'Items[*].VersionNumber' \
  --output json | jq -r '.[]' | sort -nr
```

#### Filter by Date Range

```bash
# List versions created in last 7 days
SEVEN_DAYS_AGO=$(date -d '7 days ago' -u +"%Y-%m-%dT%H:%M:%SZ")

aws appconfig list-hosted-configuration-versions \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --region us-east-1 \
  --output json | jq --arg date "$SEVEN_DAYS_AGO" '.Items[] | select(.CreationTime >= $date)'
```

### 4.3 Get Configuration Version

#### Download Specific Version Content

```bash
# Download version to file
aws appconfig get-hosted-configuration-version \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --version-number 5 \
  --region us-east-1 \
  /tmp/config-version-5.json

# View content
cat /tmp/config-version-5.json | jq .
```

#### View Version Metadata

```bash
# Get version metadata (without content)
aws appconfig list-hosted-configuration-versions \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --region us-east-1 \
  --output json | jq '.Items[] | select(.VersionNumber == 5)'
```

#### Compare Versions

```bash
# Compare two configuration versions
aws appconfig get-hosted-configuration-version \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --version-number 4 \
  --region us-east-1 \
  /tmp/config-v4.json

aws appconfig get-hosted-configuration-version \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --version-number 5 \
  --region us-east-1 \
  /tmp/config-v5.json

# Show differences
diff /tmp/config-v4.json /tmp/config-v5.json

# Or use jq for better formatting
diff <(jq -S . /tmp/config-v4.json) <(jq -S . /tmp/config-v5.json)
```

### 4.4 Delete Configuration Version

```bash
# Delete a specific configuration version
aws appconfig delete-hosted-configuration-version \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --version-number 3 \
  --region us-east-1
```

#### Cleanup Old Versions (Script)

```bash
# Delete all versions except the latest 10
VERSIONS=$(aws appconfig list-hosted-configuration-versions \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --region us-east-1 \
  --query 'Items[*].VersionNumber' \
  --output json | jq -r '.[]' | sort -nr)

KEEP=10
COUNT=0

for VERSION in $VERSIONS; do
  COUNT=$((COUNT + 1))
  if [ $COUNT -gt $KEEP ]; then
    echo "Deleting version $VERSION"
    aws appconfig delete-hosted-configuration-version \
      --application-id abc123 \
      --configuration-profile-id ghi789 \
      --version-number $VERSION \
      --region us-east-1
  fi
done
```

---

## 5. Deployment Commands

### 5.1 Start Deployment

🔥 **PRIMARY COMMAND - Deploy configuration to environment**

```bash
# Start deployment with all parameters
aws appconfig start-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --configuration-profile-id ghi789 \
  --configuration-version 5 \
  --deployment-strategy-id jkl012 \
  --description "Deploying version 5 with canary strategy" \
  --region us-east-1 \
  --output json

# Store deployment number in variable
DEPLOYMENT_NUMBER=$(aws appconfig start-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --configuration-profile-id ghi789 \
  --configuration-version 5 \
  --deployment-strategy-id jkl012 \
  --description "Deployment from CI/CD" \
  --region us-east-1 \
  --output json | jq -r '.DeploymentNumber')

echo "Started deployment: $DEPLOYMENT_NUMBER"
```

#### Different Deployment Strategy Examples

```bash
# Immediate deployment (for dev/test)
aws appconfig start-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --configuration-profile-id ghi789 \
  --configuration-version 5 \
  --deployment-strategy-id immediate-strategy-id \
  --description "Immediate deployment to dev" \
  --region us-east-1 \
  --output json

# Canary deployment (for production)
aws appconfig start-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --configuration-profile-id ghi789 \
  --configuration-version 5 \
  --deployment-strategy-id canary-strategy-id \
  --description "Canary deployment to production" \
  --region us-east-1 \
  --output json

# Use AWS managed strategy (AllAtOnce)
aws appconfig start-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --configuration-profile-id ghi789 \
  --configuration-version 5 \
  --deployment-strategy-id "AppConfig.AllAtOnce" \
  --description "Quick deployment" \
  --region us-east-1 \
  --output json
```

#### Add Description with Tracking Info

```bash
# Include detailed tracking information
GIT_COMMIT=$(git rev-parse HEAD)
GIT_SHORT=$(git rev-parse --short HEAD)
GIT_AUTHOR=$(git log -1 --pretty=format:'%an')
TIMESTAMP=$(date -u +"%Y-%m-%d %H:%M:%S UTC")

DESCRIPTION="Version: 5 | Commit: $GIT_SHORT | Author: $GIT_AUTHOR | Time: $TIMESTAMP"

aws appconfig start-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --configuration-profile-id ghi789 \
  --configuration-version 5 \
  --deployment-strategy-id jkl012 \
  --description "$DESCRIPTION" \
  --region us-east-1 \
  --output json
```

### 5.2 Get Deployment Status

```bash
# Get full deployment details
aws appconfig get-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --deployment-number 7 \
  --region us-east-1 \
  --output json

# Get just the status field
aws appconfig get-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --deployment-number 7 \
  --region us-east-1 \
  --query 'State' \
  --output text

# Get status and percentage complete
aws appconfig get-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --deployment-number 7 \
  --region us-east-1 \
  --query '[State,PercentageComplete]' \
  --output table
```

#### JSON vs Table Output Examples

```bash
# Table format (human-readable)
aws appconfig get-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --deployment-number 7 \
  --region us-east-1 \
  --query '[DeploymentNumber,State,PercentageComplete,ConfigurationVersion]' \
  --output table

# JSON format (for parsing in scripts)
aws appconfig get-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --deployment-number 7 \
  --region us-east-1 \
  --output json | jq -r '{state: .State, progress: .PercentageComplete, version: .ConfigurationVersion}'
```

### 5.3 Monitor Deployment

🔥 **Complete monitoring loop script**

```bash
#!/bin/bash

APP_ID="abc123"
ENV_ID="def456"
DEPLOYMENT_NUMBER="7"
AWS_REGION="us-east-1"

echo "=== Monitoring Deployment $DEPLOYMENT_NUMBER ==="
echo "Started at: $(date)"

MAX_ATTEMPTS=60  # 60 attempts × 10 seconds = 10 minutes max
ATTEMPT=0
DEPLOYMENT_COMPLETE=false

while [ $ATTEMPT -lt $MAX_ATTEMPTS ] && [ "$DEPLOYMENT_COMPLETE" = false ]; do
  ATTEMPT=$((ATTEMPT + 1))
  
  # Get deployment status
  DEPLOYMENT_JSON=$(aws appconfig get-deployment \
    --application-id $APP_ID \
    --environment-id $ENV_ID \
    --deployment-number $DEPLOYMENT_NUMBER \
    --region $AWS_REGION \
    --output json)
  
  STATE=$(echo $DEPLOYMENT_JSON | jq -r '.State')
  PERCENTAGE=$(echo $DEPLOYMENT_JSON | jq -r '.PercentageComplete')
  
  TIMESTAMP=$(date "+%Y-%m-%d %H:%M:%S")
  echo "[$TIMESTAMP] Attempt $ATTEMPT/$MAX_ATTEMPTS: Status=$STATE, Progress=${PERCENTAGE}%"
  
  case $STATE in
    COMPLETE)
      echo "✓ Deployment completed successfully!"
      DEPLOYMENT_COMPLETE=true
      exit 0
      ;;
    ROLLED_BACK)
      echo "✗ Deployment was rolled back!"
      echo "Check AppConfig console for rollback reason"
      exit 1
      ;;
    BAKING)
      echo "⏰ Deployment is in baking phase (waiting for bake time to complete)..."
      ;;
    DEPLOYING)
      echo "🚀 Deployment in progress..."
      ;;
    *)
      echo "⚠ Unknown state: $STATE"
      ;;
  esac
  
  if [ "$DEPLOYMENT_COMPLETE" = false ]; then
    sleep 10
  fi
done

if [ "$DEPLOYMENT_COMPLETE" = false ]; then
  echo "✗ Deployment monitoring timeout after $MAX_ATTEMPTS attempts"
  exit 1
fi
```

### 5.4 List Deployments

```bash
# List all deployments for an environment
aws appconfig list-deployments \
  --application-id abc123 \
  --environment-id def456 \
  --region us-east-1 \
  --output table

# Get latest deployment
aws appconfig list-deployments \
  --application-id abc123 \
  --environment-id def456 \
  --region us-east-1 \
  --query 'Items[0]' \
  --output json

# Get latest deployment number
aws appconfig list-deployments \
  --application-id abc123 \
  --environment-id def456 \
  --region us-east-1 \
  --query 'Items[0].DeploymentNumber' \
  --output text
```

#### Filter by Status

```bash
# Get only completed deployments
aws appconfig list-deployments \
  --application-id abc123 \
  --environment-id def456 \
  --region us-east-1 \
  --output json | jq '.Items[] | select(.State == "COMPLETE")'

# Get failed/rolled back deployments
aws appconfig list-deployments \
  --application-id abc123 \
  --environment-id def456 \
  --region us-east-1 \
  --output json | jq '.Items[] | select(.State == "ROLLED_BACK")'
```

#### Sort Deployments

```bash
# List deployments sorted by deployment number (newest first)
aws appconfig list-deployments \
  --application-id abc123 \
  --environment-id def456 \
  --region us-east-1 \
  --query 'Items | sort_by(@, &DeploymentNumber) | reverse(@)[*].[DeploymentNumber,State,ConfigurationVersion]' \
  --output table
```

### 5.5 Stop Deployment (Rollback)

⚠️ **Stop an in-progress deployment**

```bash
# Stop deployment (triggers rollback)
aws appconfig stop-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --deployment-number 7 \
  --region us-east-1

echo "Deployment stopped. Rolling back to previous configuration..."
```

**When to Use**:
- Critical issue detected during deployment
- Need to abort canary rollout immediately
- Emergency situation requiring immediate rollback

**Implications**:
- Immediately stops the deployment
- Rolls back to the previously deployed configuration
- Applications will receive the old configuration on next fetch
- Deployment state will change to `ROLLED_BACK`

---

## 6. Retrieve Configuration Commands

### 6.1 Get Configuration

🔥 **Get currently deployed configuration - what applications call**

```bash
# Get current configuration for an application
aws appconfig get-configuration \
  --application auth-10 \
  --environment cdc1c \
  --configuration authapi-config \
  --client-id "my-application-instance-1" \
  /tmp/current-config.json

# View the configuration
cat /tmp/current-config.json | jq .
```

**Important Notes**:
- This retrieves the **currently deployed** configuration
- Applications use this command (or SDK equivalent) to fetch config
- Requires application name, environment name, configuration profile name (not IDs)
- `client-id` should be unique per application instance for tracking

#### Different Client-ID Examples

```bash
# Use hostname as client-id
CLIENT_ID=$(hostname)
aws appconfig get-configuration \
  --application auth-10 \
  --environment cdc1c \
  --configuration authapi-config \
  --client-id "$CLIENT_ID" \
  /tmp/config.json

# Use container ID in ECS
CLIENT_ID=$(cat /proc/self/cgroup | grep -o 'docker/[^/]*' | cut -d'/' -f2)
aws appconfig get-configuration \
  --application auth-10 \
  --environment cdc1c \
  --configuration authapi-config \
  --client-id "$CLIENT_ID" \
  /tmp/config.json

# Use Lambda request ID
aws appconfig get-configuration \
  --application auth-10 \
  --environment cdc1c \
  --configuration authapi-config \
  --client-id "$AWS_REQUEST_ID" \
  /tmp/config.json
```

#### View with jq Formatting

```bash
# Pretty print configuration
aws appconfig get-configuration \
  --application auth-10 \
  --environment cdc1c \
  --configuration authapi-config \
  --client-id "cli-test" \
  /tmp/config.json && jq . /tmp/config.json

# Extract specific feature flag
aws appconfig get-configuration \
  --application auth-10 \
  --environment cdc1c \
  --configuration authapi-config \
  --client-id "cli-test" \
  /tmp/config.json && jq '.featureFlags.enableNewAuth' /tmp/config.json

# Extract all settings
aws appconfig get-configuration \
  --application auth-10 \
  --environment cdc1c \
  --configuration authapi-config \
  --client-id "cli-test" \
  /tmp/config.json && jq '.settings' /tmp/config.json
```

#### Use Cases

**Development Testing**:
```bash
# Test configuration retrieval during development
aws appconfig get-configuration \
  --application auth-10 \
  --environment cdc1c \
  --configuration authapi-config \
  --client-id "dev-test-$(date +%s)" \
  /tmp/test-config.json
```

**CI/CD Validation**:
```bash
# Validate deployed configuration in pipeline
aws appconfig get-configuration \
  --application auth-10 \
  --environment pdc1c \
  --configuration authapi-config \
  --client-id "cicd-validation" \
  /tmp/prod-config.json

# Check specific feature flag value
FEATURE_VALUE=$(jq -r '.featureFlags.enableNewAuth' /tmp/prod-config.json)
if [ "$FEATURE_VALUE" = "true" ]; then
  echo "✓ Feature is enabled in production"
else
  echo "✗ Feature is disabled in production"
fi
```

### 6.2 Verify Deployment

#### Compare Deployed Config with Source File

```bash
# Get currently deployed configuration
aws appconfig get-configuration \
  --application auth-10 \
  --environment cdc1c \
  --configuration authapi-config \
  --client-id "verification" \
  /tmp/deployed.json

# Compare with source file
diff <(jq -S . production-approach/config/cdc1c.json) <(jq -S . /tmp/deployed.json)

# If no output, they match (success)
if diff <(jq -S . production-approach/config/cdc1c.json) <(jq -S . /tmp/deployed.json) > /dev/null; then
  echo "✓ Deployed configuration matches source file"
else
  echo "✗ Configuration mismatch detected"
  exit 1
fi
```

#### Diff Commands

```bash
# Colored diff output
diff --color=always <(jq -S . /tmp/source.json) <(jq -S . /tmp/deployed.json)

# Side-by-side comparison
diff -y <(jq -S . /tmp/source.json) <(jq -S . /tmp/deployed.json)

# Unified diff format
diff -u <(jq -S . /tmp/source.json) <(jq -S . /tmp/deployed.json)
```

#### Validation Script

```bash
#!/bin/bash

STACK_NAME="appconfig-auth-10-cdc1c"
CONFIG_FILE="production-approach/config/cdc1c.json"
AWS_REGION="us-east-1"

echo "=== Verifying Deployment ==="

# Get Application name from stack
APP_NAME=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`ApplicationId`].OutputValue' \
  --output text | xargs aws appconfig get-application \
  --application-id $APP_ID \
  --region $AWS_REGION \
  --query 'Name' \
  --output text)

# Get Environment name
ENV_NAME=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`EnvironmentId`].OutputValue' \
  --output text | xargs aws appconfig get-environment \
  --application-id $APP_ID \
  --environment-id $ENV_ID \
  --region $AWS_REGION \
  --query 'Name' \
  --output text)

# Fetch deployed config
aws appconfig get-configuration \
  --application "$APP_NAME" \
  --environment "$ENV_NAME" \
  --configuration "authapi-config" \
  --client-id "verification-$(date +%s)" \
  /tmp/deployed.json

# Compare
if diff <(jq -S . $CONFIG_FILE) <(jq -S . /tmp/deployed.json) > /dev/null; then
  echo "✓ Verification passed: Deployed configuration matches source"
  exit 0
else
  echo "✗ Verification failed: Configuration mismatch"
  echo "Differences:"
  diff -u <(jq -S . $CONFIG_FILE) <(jq -S . /tmp/deployed.json)
  exit 1
fi
```

---

## 7. Complete Workflow Scripts

### 7.1 Scenario 1: First-Time Infrastructure Setup

```bash
#!/bin/bash
# first-time-setup.sh
# Creates AppConfig infrastructure for a new service

set -e  # Exit on error

# Variables
ASSET_ID="10"
ASSET_NAME="auth"
ASSET_GROUP="cdc1c"
ASSET_SERVICE="authapi"
AWS_REGION="us-east-1"
TEMPLATE_FILE="production-approach/infra/appconfig-service.yml"

# Construct stack name
STACK_NAME="appconfig-${ASSET_NAME}-${ASSET_ID}-${ASSET_GROUP}"

echo "=== Creating AppConfig Infrastructure ==="
echo "Stack Name: $STACK_NAME"
echo "Asset: $ASSET_NAME-$ASSET_ID"
echo "Environment: $ASSET_GROUP"
echo "Service: $ASSET_SERVICE"
echo "Region: $AWS_REGION"
echo ""

# Deploy CloudFormation stack
aws cloudformation deploy \
  --stack-name $STACK_NAME \
  --template-file $TEMPLATE_FILE \
  --parameter-overrides \
    AssetId=$ASSET_ID \
    AssetName=$ASSET_NAME \
    AssetGroup=$ASSET_GROUP \
    AssetService=$ASSET_SERVICE \
    DeployInitialConfig=no \
  --tags \
    AssetId=$ASSET_ID \
    AssetName=$ASSET_NAME \
    AssetGroup=$ASSET_GROUP \
    Service=$ASSET_SERVICE \
    ManagedBy=Script \
  --region $AWS_REGION \
  --no-fail-on-empty-changeset

echo ""
echo "✓ Infrastructure created successfully!"
echo ""
echo "Stack Outputs:"
aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query 'Stacks[0].Outputs[*].[OutputKey,OutputValue]' \
  --output table

echo ""
echo "Next steps:"
echo "1. Create/update configuration file: production-approach/config/${ASSET_GROUP}.json"
echo "2. Run deployment script to deploy configuration"
```

### 7.2 Scenario 2: Infrastructure + Initial Configuration

```bash
#!/bin/bash
# create-with-initial-config.sh
# Creates infrastructure AND deploys initial configuration

set -e

# Variables
ASSET_ID="10"
ASSET_NAME="auth"
ASSET_GROUP="cdc1c"
ASSET_SERVICE="authapi"
AWS_REGION="us-east-1"
TEMPLATE_FILE="production-approach/infra/appconfig-service.yml"
CONFIG_FILE="production-approach/config/${ASSET_GROUP}.json"

STACK_NAME="appconfig-${ASSET_NAME}-${ASSET_ID}-${ASSET_GROUP}"

echo "=== Creating Infrastructure with Initial Configuration ==="

# Validate config file exists
if [ ! -f "$CONFIG_FILE" ]; then
  echo "✗ Configuration file not found: $CONFIG_FILE"
  exit 1
fi

echo "✓ Configuration file found"

# Validate JSON syntax
if ! jq empty "$CONFIG_FILE" 2>/dev/null; then
  echo "✗ Invalid JSON in configuration file"
  exit 1
fi

echo "✓ JSON syntax valid"

# Read and compact JSON
CONFIG_CONTENT=$(jq -c . "$CONFIG_FILE")

echo ""
echo "Creating stack with initial configuration..."

# Deploy with initial configuration
aws cloudformation deploy \
  --stack-name $STACK_NAME \
  --template-file $TEMPLATE_FILE \
  --parameter-overrides \
    AssetId=$ASSET_ID \
    AssetName=$ASSET_NAME \
    AssetGroup=$ASSET_GROUP \
    AssetService=$ASSET_SERVICE \
    DeployInitialConfig=yes \
    InitialConfigContent="$CONFIG_CONTENT" \
  --tags \
    AssetId=$ASSET_ID \
    AssetName=$ASSET_NAME \
    AssetGroup=$ASSET_GROUP \
  --region $AWS_REGION \
  --no-fail-on-empty-changeset

echo ""
echo "✓ Infrastructure created with initial configuration!"

# Verify configuration
echo ""
echo "Verifying deployed configuration..."

APP_NAME="${ASSET_NAME}-${ASSET_ID}"

sleep 5  # Wait for configuration to be available

aws appconfig get-configuration \
  --application "$APP_NAME" \
  --environment "$ASSET_GROUP" \
  --configuration "${ASSET_SERVICE}-config" \
  --client-id "verification-$(date +%s)" \
  /tmp/deployed-config.json

if diff <(jq -S . "$CONFIG_FILE") <(jq -S . /tmp/deployed-config.json) > /dev/null; then
  echo "✓ Configuration deployed successfully and matches source"
else
  echo "⚠ Configuration may differ from source (this is normal for CloudFormation-managed initial config)"
fi

rm -f /tmp/deployed-config.json

echo ""
echo "Setup complete!"
```

### 7.3 Scenario 3: Deploy Configuration Only (Most Common Use Case)

🔥 **COMPLETE PRODUCTION-READY SCRIPT**

```bash
#!/bin/bash
# deploy-config.sh
# Production-ready script to deploy configuration to AppConfig
# This is the MOST COMMONLY USED script in day-to-day operations

set -e  # Exit on any error

# ============================================================================
# CONFIGURATION
# ============================================================================

ASSET_ID="${ASSET_ID:-10}"
ASSET_NAME="${ASSET_NAME:-auth}"
ASSET_GROUP="${ASSET_GROUP:-cdc1c}"
ASSET_SERVICE="${ASSET_SERVICE:-authapi}"
DEPLOYMENT_STRATEGY="${DEPLOYMENT_STRATEGY:-Immediate}"  # Immediate or Canary
AWS_REGION="${AWS_REGION:-us-east-1}"

STACK_NAME="appconfig-${ASSET_NAME}-${ASSET_ID}-${ASSET_GROUP}"
CONFIG_FILE="production-approach/config/${ASSET_GROUP}.json"

# ============================================================================
# HELPER FUNCTIONS
# ============================================================================

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

error() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] ✗ ERROR: $1" >&2
  exit 1
}

success() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] ✓ $1"
}

# ============================================================================
# STEP 1: VALIDATE CONFIGURATION FILE EXISTS
# ============================================================================

log "Step 1: Validating configuration file..."

if [ ! -f "$CONFIG_FILE" ]; then
  error "Configuration file not found: $CONFIG_FILE"
fi

success "Configuration file exists: $CONFIG_FILE"

# ============================================================================
# STEP 2: VALIDATE JSON SYNTAX
# ============================================================================

log "Step 2: Validating JSON syntax..."

if ! jq empty "$CONFIG_FILE" 2>/dev/null; then
  error "Invalid JSON in configuration file: $CONFIG_FILE"
fi

# Check required sections
if ! jq -e '.featureFlags' "$CONFIG_FILE" > /dev/null; then
  error "Configuration missing required 'featureFlags' section"
fi

if ! jq -e '.settings' "$CONFIG_FILE" > /dev/null; then
  error "Configuration missing required 'settings' section"
fi

success "JSON syntax valid and required sections present"

# ============================================================================
# STEP 3: CHECK IF CLOUDFORMATION STACK EXISTS
# ============================================================================

log "Step 3: Checking CloudFormation stack..."

STACK_STATUS=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query 'Stacks[0].StackStatus' \
  --output text 2>/dev/null || echo "NOT_FOUND")

if [ "$STACK_STATUS" = "NOT_FOUND" ]; then
  error "CloudFormation stack '$STACK_NAME' does not exist. Create infrastructure first."
fi

if [[ ! "$STACK_STATUS" =~ (CREATE_COMPLETE|UPDATE_COMPLETE) ]]; then
  error "Stack is in invalid state: $STACK_STATUS"
fi

success "CloudFormation stack exists and is ready: $STACK_STATUS"

# ============================================================================
# STEP 4: GET APPCONFIG IDS FROM STACK OUTPUTS
# ============================================================================

log "Step 4: Retrieving AppConfig resource IDs from stack outputs..."

APP_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`ApplicationId`].OutputValue' \
  --output text)

ENV_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`EnvironmentId`].OutputValue' \
  --output text)

PROFILE_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`ConfigurationProfileId`].OutputValue' \
  --output text)

# Get strategy ID based on deployment type
if [ "$DEPLOYMENT_STRATEGY" = "Canary" ]; then
  STRATEGY_OUTPUT="DeploymentStrategyCanaryId"
else
  STRATEGY_OUTPUT="DeploymentStrategyImmediateId"
fi

STRATEGY_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query "Stacks[0].Outputs[?OutputKey==\`${STRATEGY_OUTPUT}\`].OutputValue" \
  --output text)

# Validate we got all IDs
if [ -z "$APP_ID" ] || [ -z "$ENV_ID" ] || [ -z "$PROFILE_ID" ] || [ -z "$STRATEGY_ID" ]; then
  error "Failed to retrieve AppConfig resource IDs from stack"
fi

success "Resource IDs retrieved successfully"
log "  Application ID: $APP_ID"
log "  Environment ID: $ENV_ID"
log "  Profile ID: $PROFILE_ID"
log "  Strategy ID: $STRATEGY_ID ($DEPLOYMENT_STRATEGY)"

# ============================================================================
# STEP 5: CREATE NEW CONFIGURATION VERSION FROM FILE
# ============================================================================

log "Step 5: Creating new configuration version..."

# Get Git info for description
GIT_COMMIT=$(git rev-parse --short HEAD 2>/dev/null || echo "unknown")
GIT_AUTHOR=$(git log -1 --pretty=format:'%an' 2>/dev/null || echo "unknown")
TIMESTAMP=$(date -u '+%Y-%m-%d %H:%M:%S UTC')

DESCRIPTION="Env: $ASSET_GROUP | Commit: $GIT_COMMIT | Author: $GIT_AUTHOR | Time: $TIMESTAMP"

# Create hosted configuration version
VERSION_RESPONSE=$(aws appconfig create-hosted-configuration-version \
  --application-id $APP_ID \
  --configuration-profile-id $PROFILE_ID \
  --content file://$CONFIG_FILE \
  --content-type "application/json" \
  --description "$DESCRIPTION" \
  --region $AWS_REGION \
  --output json)

VERSION_NUMBER=$(echo $VERSION_RESPONSE | jq -r '.VersionNumber')

if [ -z "$VERSION_NUMBER" ] || [ "$VERSION_NUMBER" = "null" ]; then
  error "Failed to create configuration version"
fi

success "Configuration version $VERSION_NUMBER created"

# ============================================================================
# STEP 6: START DEPLOYMENT
# ============================================================================

log "Step 6: Starting deployment..."

DEPLOY_DESCRIPTION="Version $VERSION_NUMBER | Commit: $GIT_COMMIT | Strategy: $DEPLOYMENT_STRATEGY | Time: $TIMESTAMP"

DEPLOYMENT_RESPONSE=$(aws appconfig start-deployment \
  --application-id $APP_ID \
  --environment-id $ENV_ID \
  --configuration-profile-id $PROFILE_ID \
  --configuration-version $VERSION_NUMBER \
  --deployment-strategy-id $STRATEGY_ID \
  --description "$DEPLOY_DESCRIPTION" \
  --region $AWS_REGION \
  --output json)

DEPLOYMENT_NUMBER=$(echo $DEPLOYMENT_RESPONSE | jq -r '.DeploymentNumber')

if [ -z "$DEPLOYMENT_NUMBER" ] || [ "$DEPLOYMENT_NUMBER" = "null" ]; then
  error "Failed to start deployment"
fi

success "Deployment $DEPLOYMENT_NUMBER started"
log "  Version: $VERSION_NUMBER"
log "  Strategy: $DEPLOYMENT_STRATEGY"

# ============================================================================
# STEP 7: MONITOR DEPLOYMENT WITH REAL-TIME STATUS
# ============================================================================

log "Step 7: Monitoring deployment progress..."
echo ""

MAX_ATTEMPTS=60  # 60 × 10 seconds = 10 minutes
ATTEMPT=0
DEPLOYMENT_COMPLETE=false

while [ $ATTEMPT -lt $MAX_ATTEMPTS ] && [ "$DEPLOYMENT_COMPLETE" = "false" ]; do
  ATTEMPT=$((ATTEMPT + 1))
  
  # Get deployment status
  DEPLOYMENT_STATUS=$(aws appconfig get-deployment \
    --application-id $APP_ID \
    --environment-id $ENV_ID \
    --deployment-number $DEPLOYMENT_NUMBER \
    --region $AWS_REGION \
    --output json)
  
  STATE=$(echo $DEPLOYMENT_STATUS | jq -r '.State')
  PERCENTAGE=$(echo $DEPLOYMENT_STATUS | jq -r '.PercentageComplete')
  
  TIMESTAMP_NOW=$(date '+%H:%M:%S')
  
  # Progress indicator
  PROGRESS_BAR=""
  FILLED=$((PERCENTAGE / 5))
  for i in $(seq 1 20); do
    if [ $i -le $FILLED ]; then
      PROGRESS_BAR="${PROGRESS_BAR}█"
    else
      PROGRESS_BAR="${PROGRESS_BAR}░"
    fi
  done
  
  printf "\r[$TIMESTAMP_NOW] Status: %-12s Progress: [%s] %3d%%" "$STATE" "$PROGRESS_BAR" "$PERCENTAGE"
  
  case $STATE in
    COMPLETE)
      echo ""
      success "Deployment completed successfully!"
      DEPLOYMENT_COMPLETE=true
      ;;
    ROLLED_BACK)
      echo ""
      error "Deployment was rolled back. Check AppConfig console for details."
      ;;
    BAKING)
      # Continue monitoring during bake time
      ;;
    DEPLOYING)
      # Continue monitoring
      ;;
    *)
      echo ""
      log "Unknown deployment state: $STATE"
      ;;
  esac
  
  if [ "$DEPLOYMENT_COMPLETE" = "false" ]; then
    sleep 10
  fi
done

if [ "$DEPLOYMENT_COMPLETE" = "false" ]; then
  echo ""
  error "Deployment monitoring timeout after $MAX_ATTEMPTS attempts"
fi

# ============================================================================
# STEP 8: VERIFY DEPLOYED CONFIGURATION MATCHES SOURCE
# ============================================================================

log "Step 8: Verifying deployed configuration..."

# Wait a moment for configuration to be available
sleep 3

# Get deployed configuration
aws appconfig get-hosted-configuration-version \
  --application-id $APP_ID \
  --configuration-profile-id $PROFILE_ID \
  --version-number $VERSION_NUMBER \
  --region $AWS_REGION \
  /tmp/deployed-verification.json

# Compare with source
if diff <(jq -S . $CONFIG_FILE) <(jq -S . /tmp/deployed-verification.json) > /dev/null; then
  success "Verification passed: Deployed configuration matches source file"
else
  log "⚠ Warning: Deployed configuration differs from source"
  diff -u <(jq -S . $CONFIG_FILE) <(jq -S . /tmp/deployed-verification.json) || true
fi

# Cleanup
rm -f /tmp/deployed-verification.json

# ============================================================================
# SUCCESS SUMMARY
# ============================================================================

echo ""
echo "╔════════════════════════════════════════════════════════════════╗"
echo "║           DEPLOYMENT COMPLETED SUCCESSFULLY                    ║"
echo "╚════════════════════════════════════════════════════════════════╝"
echo ""
echo "  Stack:           $STACK_NAME"
echo "  Environment:     $ASSET_GROUP"
echo "  Config File:     $CONFIG_FILE"
echo "  Version:         $VERSION_NUMBER"
echo "  Deployment:      $DEPLOYMENT_NUMBER"
echo "  Strategy:        $DEPLOYMENT_STRATEGY"
echo "  Git Commit:      $GIT_COMMIT"
echo ""
echo "  Applications can now fetch the updated configuration using:"
echo "    Application:   ${ASSET_NAME}-${ASSET_ID}"
echo "    Environment:   $ASSET_GROUP"
echo "    Configuration: ${ASSET_SERVICE}-config"
echo ""

exit 0
```

**Usage Examples**:

```bash
# Deploy to CDC1C (dev) with immediate strategy
./deploy-config.sh

# Deploy to PDC1C (prod) with canary strategy
ASSET_GROUP=pdc1c DEPLOYMENT_STRATEGY=Canary ./deploy-config.sh

# Deploy different service
ASSET_ID=20 ASSET_NAME=payment ASSET_SERVICE=paymentapi ASSET_GROUP=cdc4c ./deploy-config.sh
```

### 7.4 Scenario 4: Rollback to Previous Version

```bash
#!/bin/bash
# rollback-config.sh
# Rollback to a previous configuration version

set -e

# Variables
ASSET_ID="10"
ASSET_NAME="auth"
ASSET_GROUP="cdc1c"
ROLLBACK_VERSION="${1}"  # Version number to rollback to
AWS_REGION="us-east-1"

STACK_NAME="appconfig-${ASSET_NAME}-${ASSET_ID}-${ASSET_GROUP}"

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

if [ -z "$ROLLBACK_VERSION" ]; then
  echo "Usage: $0 <version-number>"
  echo ""
  echo "Available versions:"
  
  # Get resource IDs
  APP_ID=$(aws cloudformation describe-stacks \
    --stack-name $STACK_NAME \
    --region $AWS_REGION \
    --query 'Stacks[0].Outputs[?OutputKey==`ApplicationId`].OutputValue' \
    --output text)
  
  PROFILE_ID=$(aws cloudformation describe-stacks \
    --stack-name $STACK_NAME \
    --region $AWS_REGION \
    --query 'Stacks[0].Outputs[?OutputKey==`ConfigurationProfileId`].OutputValue' \
    --output text)
  
  # List versions
  aws appconfig list-hosted-configuration-versions \
    --application-id $APP_ID \
    --configuration-profile-id $PROFILE_ID \
    --region $AWS_REGION \
    --output json | jq -r '.Items[] | "Version \(.VersionNumber): \(.Description)"'
  
  exit 1
fi

log "Starting rollback to version $ROLLBACK_VERSION..."

# Get resource IDs from stack
APP_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`ApplicationId`].OutputValue' \
  --output text)

ENV_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`EnvironmentId`].OutputValue' \
  --output text)

PROFILE_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`ConfigurationProfileId`].OutputValue' \
  --output text)

STRATEGY_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`DeploymentStrategyImmediateId`].OutputValue' \
  --output text)

log "Resource IDs retrieved from stack"

# Verify version exists
VERSION_EXISTS=$(aws appconfig list-hosted-configuration-versions \
  --application-id $APP_ID \
  --configuration-profile-id $PROFILE_ID \
  --region $AWS_REGION \
  --output json | jq ".Items[] | select(.VersionNumber == $ROLLBACK_VERSION)")

if [ -z "$VERSION_EXISTS" ]; then
  echo "✗ Version $ROLLBACK_VERSION does not exist"
  exit 1
fi

log "Version $ROLLBACK_VERSION found"

# Start rollback deployment
DESCRIPTION="ROLLBACK to version $ROLLBACK_VERSION | Time: $(date -u '+%Y-%m-%d %H:%M:%S UTC')"

DEPLOYMENT_RESPONSE=$(aws appconfig start-deployment \
  --application-id $APP_ID \
  --environment-id $ENV_ID \
  --configuration-profile-id $PROFILE_ID \
  --configuration-version $ROLLBACK_VERSION \
  --deployment-strategy-id $STRATEGY_ID \
  --description "$DESCRIPTION" \
  --region $AWS_REGION \
  --output json)

DEPLOYMENT_NUMBER=$(echo $DEPLOYMENT_RESPONSE | jq -r '.DeploymentNumber')

log "Rollback deployment $DEPLOYMENT_NUMBER started"

# Monitor deployment
MAX_ATTEMPTS=30
ATTEMPT=0

while [ $ATTEMPT -lt $MAX_ATTEMPTS ]; do
  ATTEMPT=$((ATTEMPT + 1))
  sleep 10
  
  STATE=$(aws appconfig get-deployment \
    --application-id $APP_ID \
    --environment-id $ENV_ID \
    --deployment-number $DEPLOYMENT_NUMBER \
    --region $AWS_REGION \
    --query 'State' \
    --output text)
  
  log "Deployment status: $STATE"
  
  if [ "$STATE" = "COMPLETE" ]; then
    echo "✓ Rollback completed successfully!"
    exit 0
  elif [ "$STATE" = "ROLLED_BACK" ]; then
    echo "✗ Rollback deployment failed"
    exit 1
  fi
done

echo "✗ Rollback monitoring timeout"
exit 1
```

### 7.5 Scenario 5: Multi-Environment Deployment

```bash
#!/bin/bash
# multi-env-deploy.sh
# Deploy same configuration to multiple environments in sequence

set -e

ASSET_ID="10"
ASSET_NAME="auth"
ASSET_SERVICE="authapi"
ENVIRONMENTS=("cdc1c" "cdc4c" "cdc7c" "pdc1c")
AWS_REGION="us-east-1"

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

echo "╔════════════════════════════════════════════════════════════╗"
echo "║       Multi-Environment Configuration Deployment           ║"
echo "╚════════════════════════════════════════════════════════════╝"
echo ""
echo "This will deploy to: ${ENVIRONMENTS[@]}"
echo ""
read -p "Continue? (yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
  echo "Deployment cancelled"
  exit 0
fi

FAILED_ENVS=()
SUCCESS_ENVS=()

for ENV in "${ENVIRONMENTS[@]}"; do
  echo ""
  echo "======================================================================"
  log "Deploying to environment: $ENV"
  echo "======================================================================"
  
  # Set environment variables and run deploy script
  if ASSET_GROUP=$ENV DEPLOYMENT_STRATEGY=Immediate ./deploy-config.sh; then
    SUCCESS_ENVS+=($ENV)
    log "✓ Successfully deployed to $ENV"
  else
    FAILED_ENVS+=($ENV)
    log "✗ Failed to deploy to $ENV"
    
    # Ask to continue or abort
    read -p "Continue with remaining environments? (yes/no): " CONTINUE
    if [ "$CONTINUE" != "yes" ]; then
      break
    fi
  fi
  
  # Wait between deployments
  if [ "$ENV" != "${ENVIRONMENTS[-1]}" ]; then
    log "Waiting 30 seconds before next deployment..."
    sleep 30
  fi
done

# Summary
echo ""
echo "╔════════════════════════════════════════════════════════════╗"
echo "║              Deployment Summary                            ║"
echo "╚════════════════════════════════════════════════════════════╝"
echo ""
echo "Successful: ${SUCCESS_ENVS[@]:-none}"
echo "Failed:     ${FAILED_ENVS[@]:-none}"
echo ""

if [ ${#FAILED_ENVS[@]} -eq 0 ]; then
  echo "✓ All environments deployed successfully!"
  exit 0
else
  echo "✗ Some deployments failed"
  exit 1
fi
```

---

## 8. Troubleshooting Commands

### 8.1 Debug Stack Issues

#### Get Stack Events with Errors

```bash
# Get all failed events
aws cloudformation describe-stack-events \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --query 'StackEvents[?contains(ResourceStatus, `FAILED`)]' \
  --output table

# Get events with failure reasons
aws cloudformation describe-stack-events \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --query 'StackEvents[?ResourceStatusReason!=`null`].[Timestamp,LogicalResourceId,ResourceStatus,ResourceStatusReason]' \
  --output table

# Get most recent 10 events
aws cloudformation describe-stack-events \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --max-items 10 \
  --output table
```

#### Describe Stack Resources

```bash
# List all resources in stack
aws cloudformation describe-stack-resources \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --output table

# Get specific resource details
aws cloudformation describe-stack-resources \
  --stack-name appconfig-auth-10-cdc1c \
  --logical-resource-id AppConfigApplication \
  --region us-east-1 \
  --output json

# Find resources by type
aws cloudformation describe-stack-resources \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --query 'StackResources[?ResourceType==`AWS::AppConfig::Application`]' \
  --output table
```

#### Get Failed Resource Reasons

```bash
# Get detailed failure information
aws cloudformation describe-stack-events \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --query 'StackEvents[?contains(ResourceStatus, `FAILED`)].[Timestamp,LogicalResourceId,ResourceStatusReason]' \
  --output table

# Export to file for analysis
aws cloudformation describe-stack-events \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --output json > /tmp/stack-events.json

# Analyze with jq
jq '.StackEvents[] | select(.ResourceStatus | contains("FAILED"))' /tmp/stack-events.json
```

### 8.2 Debug Deployment Issues

#### Check Deployment State

```bash
# Get full deployment details
aws appconfig get-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --deployment-number 7 \
  --region us-east-1 \
  --output json | jq .

# Check if deployment is stuck
aws appconfig get-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --deployment-number 7 \
  --region us-east-1 \
  --query '[State,PercentageComplete,StartedAt]' \
  --output table
```

#### View Deployment Events

```bash
# Get deployment event log
aws appconfig get-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --deployment-number 7 \
  --region us-east-1 \
  --query 'EventLog' \
  --output json | jq .

# Check for errors in event log
aws appconfig get-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --deployment-number 7 \
  --region us-east-1 \
  --output json | jq '.EventLog[] | select(.EventType == "ERROR")'
```

#### Get Validator Errors

```bash
# Check configuration profile validators
aws appconfig get-configuration-profile \
  --application-id abc123 \
  --configuration-profile-id ghi789 \
  --region us-east-1 \
  --query 'Validators' \
  --output json | jq .

# Validate configuration against schema manually
CONFIG_FILE="production-approach/config/cdc1c.json"

# Check JSON is valid
jq empty $CONFIG_FILE && echo "✓ Valid JSON" || echo "✗ Invalid JSON"

# Check required fields exist
jq -e '.featureFlags' $CONFIG_FILE > /dev/null && echo "✓ featureFlags exists"
jq -e '.settings' $CONFIG_FILE > /dev/null && echo "✓ settings exists"
```

### 8.3 Debug Configuration Issues

#### Validate JSON Schema

```bash
# Manual JSON schema validation
CONFIG_FILE="production-approach/config/cdc1c.json"

# Validate structure
cat $CONFIG_FILE | jq '
  if has("featureFlags") and has("settings") then
    "✓ Valid structure"
  else
    "✗ Missing required sections" | halt_error(1)
  end
'

# Check for common issues
cat $CONFIG_FILE | jq '
  if .featureFlags | type != "object" then
    "✗ featureFlags must be an object" | halt_error(1)
  elif .settings | type != "object" then
    "✗ settings must be an object" | halt_error(1)
  else
    "✓ All checks passed"
  end
'
```

#### Test Configuration Retrieval

```bash
# Test if configuration can be retrieved
APP_NAME="auth-10"
ENV_NAME="cdc1c"
CONFIG_NAME="authapi-config"

aws appconfig get-configuration \
  --application $APP_NAME \
  --environment $ENV_NAME \
  --configuration $CONFIG_NAME \
  --client-id "test-$(date +%s)" \
  /tmp/test-config.json

if [ $? -eq 0 ]; then
  echo "✓ Configuration retrieved successfully"
  cat /tmp/test-config.json | jq .
else
  echo "✗ Failed to retrieve configuration"
fi
```

#### Check IAM Permissions

```bash
# Check current identity
aws sts get-caller-identity

# Test CloudFormation permissions
aws cloudformation list-stacks --max-items 1 > /dev/null && \
  echo "✓ CloudFormation permissions OK" || \
  echo "✗ CloudFormation permissions issue"

# Test AppConfig permissions
aws appconfig list-applications --max-items 1 > /dev/null && \
  echo "✓ AppConfig permissions OK" || \
  echo "✗ AppConfig permissions issue"

# Simulate policy evaluation (requires additional permissions)
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/myuser \
  --action-names appconfig:CreateHostedConfigurationVersion appconfig:StartDeployment \
  --resource-arns "*"
```

---

## 9. Advanced Commands

### 9.1 Tagging

#### Add Tags to Resources

```bash
# Tag AppConfig application
aws appconfig tag-resource \
  --resource-arn arn:aws:appconfig:us-east-1:123456789012:application/abc123 \
  --tags Environment=Production,Team=Platform,CostCenter=Engineering \
  --region us-east-1

# Tag configuration profile
aws appconfig tag-resource \
  --resource-arn arn:aws:appconfig:us-east-1:123456789012:application/abc123/configurationprofile/ghi789 \
  --tags Service=authapi,Owner=TeamA \
  --region us-east-1

# Tag CloudFormation stack
aws cloudformation update-stack \
  --stack-name appconfig-auth-10-cdc1c \
  --use-previous-template \
  --tags \
    Key=Project,Value=AppConfig \
    Key=Compliance,Value=HIPAA \
  --region us-east-1
```

#### Query Resources by Tags

```bash
# List applications with specific tag
aws appconfig list-applications \
  --region us-east-1 \
  --output json | jq -r '.Items[] | select(.Tags.Environment == "Production") | .Name'

# Find stacks by tag
aws cloudformation describe-stacks \
  --region us-east-1 \
  --output json | jq -r '.Stacks[] | select(.Tags[] | select(.Key == "ManagedBy" and .Value == "Jenkins")) | .StackName'
```

#### Tag-Based Automation

```bash
# Deploy to all environments tagged as "Development"
for STACK in $(aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --region us-east-1 \
  --output json | jq -r '.StackSummaries[] | select(.Tags[] | select(.Key == "Environment" and .Value == "Development")) | .StackName'); do
  
  echo "Deploying to $STACK..."
  # Deployment logic here
done
```

### 9.2 Cross-Account / Cross-Region

#### Commands with --profile Flag

```bash
# Use named AWS profile
aws cloudformation describe-stacks \
  --stack-name appconfig-auth-10-cdc1c \
  --region us-east-1 \
  --profile production-account \
  --output table

# Deploy to different account
aws appconfig start-deployment \
  --application-id abc123 \
  --environment-id def456 \
  --configuration-profile-id ghi789 \
  --configuration-version 5 \
  --deployment-strategy-id jkl012 \
  --region us-east-1 \
  --profile dev-account \
  --output json
```

#### Different Region Examples

```bash
# List applications in different regions
for REGION in us-east-1 us-west-2 eu-west-1; do
  echo "=== $REGION ==="
  aws appconfig list-applications \
    --region $REGION \
    --output table
done

# Deploy same config to multiple regions
CONFIG_FILE="production-approach/config/cdc1c.json"
REGIONS=("us-east-1" "us-west-2" "eu-west-1")

for REGION in "${REGIONS[@]}"; do
  echo "Deploying to $REGION..."
  
  # Get region-specific resource IDs
  STACK_NAME="appconfig-auth-10-cdc1c-${REGION}"
  
  # Deployment logic for each region
done
```

#### Assume Role Examples

```bash
# Assume cross-account role
ASSUMED_ROLE=$(aws sts assume-role \
  --role-arn arn:aws:iam::123456789012:role/CrossAccountAppConfigRole \
  --role-session-name appconfig-deployment \
  --output json)

# Extract temporary credentials
export AWS_ACCESS_KEY_ID=$(echo $ASSUMED_ROLE | jq -r '.Credentials.AccessKeyId')
export AWS_SECRET_ACCESS_KEY=$(echo $ASSUMED_ROLE | jq -r '.Credentials.SecretAccessKey')
export AWS_SESSION_TOKEN=$(echo $ASSUMED_ROLE | jq -r '.Credentials.SessionToken')

# Now use AWS CLI with assumed role
aws appconfig list-applications --region us-east-1

# Cleanup
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN
```

### 9.3 Automation & Scripting

#### Using AWS CLI in Scripts

```bash
#!/bin/bash
# automation-example.sh

set -e  # Exit on error
set -u  # Exit on undefined variable
set -o pipefail  # Exit on pipe failure

# Enable debug mode
# set -x

# Function definitions
log_info() {
  echo "[INFO] $(date '+%Y-%m-%d %H:%M:%S') - $1"
}

log_error() {
  echo "[ERROR] $(date '+%Y-%m-%d %H:%M:%S') - $1" >&2
}

# Trap errors
trap 'log_error "Script failed at line $LINENO"' ERR

# Your automation logic here
log_info "Starting automation..."
```

#### Error Handling Patterns

```bash
# Pattern 1: Check command exit code
if aws appconfig list-applications --region us-east-1 > /dev/null 2>&1; then
  echo "✓ AppConfig accessible"
else
  echo "✗ AppConfig not accessible"
  exit 1
fi

# Pattern 2: Capture output and error
OUTPUT=$(aws appconfig get-application \
  --application-id abc123 \
  --region us-east-1 2>&1) || {
  echo "Failed to get application: $OUTPUT"
  exit 1
}

# Pattern 3: Retry logic
MAX_RETRIES=3
RETRY_COUNT=0

while [ $RETRY_COUNT -lt $MAX_RETRIES ]; do
  if aws appconfig start-deployment ...; then
    echo "✓ Deployment started"
    break
  else
    RETRY_COUNT=$((RETRY_COUNT + 1))
    echo "⚠ Attempt $RETRY_COUNT failed, retrying..."
    sleep 5
  fi
done

# Pattern 4: Timeout handling
timeout 300 aws cloudformation wait stack-create-complete \
  --stack-name appconfig-auth-10-cdc1c || {
  echo "✗ Stack creation timeout"
  exit 1
}
```

#### Logging Best Practices

```bash
# Setup logging
LOG_FILE="/var/log/appconfig-deploy-$(date +%Y%m%d-%H%M%S).log"
exec 1> >(tee -a "$LOG_FILE")
exec 2>&1

log() {
  echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1"
}

# Log all AWS CLI commands (debug mode)
export AWS_CLI_AUTO_PROMPT=on-partial
export AWS_PAGER=""

# Capture AWS CLI output
aws_cmd() {
  log "Executing: aws $@"
  aws "$@" 2>&1 | tee -a "$LOG_FILE"
  local EXIT_CODE=${PIPESTATUS[0]}
  log "Exit code: $EXIT_CODE"
  return $EXIT_CODE
}

# Usage
aws_cmd appconfig list-applications --region us-east-1
```

#### Idempotent Operations

```bash
# Idempotent stack deployment
deploy_stack_idempotent() {
  local STACK_NAME=$1
  local TEMPLATE_FILE=$2
  
  # Check if stack exists
  if aws cloudformation describe-stacks --stack-name $STACK_NAME > /dev/null 2>&1; then
    echo "Stack exists, updating..."
    aws cloudformation deploy \
      --stack-name $STACK_NAME \
      --template-file $TEMPLATE_FILE \
      --no-fail-on-empty-changeset
  else
    echo "Stack doesn't exist, creating..."
    aws cloudformation deploy \
      --stack-name $STACK_NAME \
      --template-file $TEMPLATE_FILE
  fi
}

# Idempotent configuration deployment
deploy_config_idempotent() {
  local APP_ID=$1
  local PROFILE_ID=$2
  local CONFIG_FILE=$3
  
  # Always create new version (idempotent by design)
  VERSION=$(aws appconfig create-hosted-configuration-version \
    --application-id $APP_ID \
    --configuration-profile-id $PROFILE_ID \
    --content file://$CONFIG_FILE \
    --content-type "application/json" \
    --output json | jq -r '.VersionNumber')
  
  echo "Created version: $VERSION"
  return 0
}
```

---

## 10. Quick Reference

### 10.1 One-Liners for Common Tasks

#### Get All Stack Outputs as Table

```bash
aws cloudformation describe-stacks --stack-name appconfig-auth-10-cdc1c --query 'Stacks[0].Outputs[*].[OutputKey,OutputValue]' --output table
```

#### Get Current Deployed Version

```bash
aws appconfig list-hosted-configuration-versions --application-id abc123 --configuration-profile-id ghi789 --query 'Items[0].VersionNumber' --output text
```

#### List All Versions

```bash
aws appconfig list-hosted-configuration-versions --application-id abc123 --configuration-profile-id ghi789 --query 'Items[*].[VersionNumber,Description]' --output table
```

#### Check Deployment Status

```bash
aws appconfig get-deployment --application-id abc123 --environment-id def456 --deployment-number 7 --query '[State,PercentageComplete]' --output text
```

#### Get Latest Deployment Info

```bash
aws appconfig list-deployments --application-id abc123 --environment-id def456 --query 'Items[0]' --output json | jq .
```

#### Deploy Config (Single Command Chain)

```bash
VERSION=$(aws appconfig create-hosted-configuration-version --application-id abc123 --configuration-profile-id ghi789 --content file://config.json --content-type "application/json" --output json | jq -r '.VersionNumber') && aws appconfig start-deployment --application-id abc123 --environment-id def456 --configuration-profile-id ghi789 --configuration-version $VERSION --deployment-strategy-id jkl012 --output json | jq -r '.DeploymentNumber'
```

#### Get Application ID from Stack

```bash
aws cloudformation describe-stacks --stack-name appconfig-auth-10-cdc1c --query 'Stacks[0].Outputs[?OutputKey==`ApplicationId`].OutputValue' --output text
```

#### Validate JSON Config File

```bash
jq empty production-approach/config/cdc1c.json && echo "✓ Valid" || echo "✗ Invalid"
```

#### Compare Two Config Versions

```bash
diff <(aws appconfig get-hosted-configuration-version --application-id abc123 --configuration-profile-id ghi789 --version-number 4 /dev/stdout | jq -S .) <(aws appconfig get-hosted-configuration-version --application-id abc123 --configuration-profile-id ghi789 --version-number 5 /dev/stdout | jq -S .)
```

#### Get All AppConfig Applications

```bash
aws appconfig list-applications --output json | jq -r '.Items[] | "\(.Id) - \(.Name)"'
```

### 10.2 Environment Variables Setup

```bash
# Export common variables
export AWS_REGION="us-east-1"
export AWS_DEFAULT_REGION="us-east-1"
export AWS_PAGER=""  # Disable pager for scripting

# Export asset-specific variables
export ASSET_ID="10"
export ASSET_NAME="auth"
export ASSET_GROUP="cdc1c"
export ASSET_SERVICE="authapi"

# Export constructed names
export STACK_NAME="appconfig-${ASSET_NAME}-${ASSET_ID}-${ASSET_GROUP}"
export CONFIG_FILE="production-approach/config/${ASSET_GROUP}.json"

# Get and export resource IDs
export APP_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --query 'Stacks[0].Outputs[?OutputKey==`ApplicationId`].OutputValue' \
  --output text)

export ENV_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --query 'Stacks[0].Outputs[?OutputKey==`EnvironmentId`].OutputValue' \
  --output text)

export PROFILE_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --query 'Stacks[0].Outputs[?OutputKey==`ConfigurationProfileId`].OutputValue' \
  --output text)

export STRATEGY_ID_IMMEDIATE=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --query 'Stacks[0].Outputs[?OutputKey==`DeploymentStrategyImmediateId`].OutputValue' \
  --output text)

export STRATEGY_ID_CANARY=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --query 'Stacks[0].Outputs[?OutputKey==`DeploymentStrategyCanaryId`].OutputValue' \
  --output text)

# Verify
echo "Environment variables set:"
echo "  STACK_NAME=$STACK_NAME"
echo "  APP_ID=$APP_ID"
echo "  ENV_ID=$ENV_ID"
echo "  PROFILE_ID=$PROFILE_ID"
```

**Usage**:
```bash
# Source the environment setup
source ./setup-env.sh

# Now use variables in commands
aws appconfig create-hosted-configuration-version \
  --application-id $APP_ID \
  --configuration-profile-id $PROFILE_ID \
  --content file://$CONFIG_FILE \
  --content-type "application/json"
```

---

## 11. Tips and Best Practices

### General Best Practices

1. **Always use --region flag for consistency**
   ```bash
   # Explicitly specify region to avoid confusion
   aws appconfig list-applications --region us-east-1
   
   # Or set default region
   export AWS_DEFAULT_REGION=us-east-1
   ```

2. **Use --output json for scripting, table for humans**
   ```bash
   # For scripts - parse with jq
   aws appconfig list-applications --output json | jq .
   
   # For viewing - use table format
   aws appconfig list-applications --output table
   ```

3. **Store resource IDs in variables**
   ```bash
   # Don't repeat queries
   APP_ID=$(aws cloudformation describe-stacks ...)
   
   # Reuse the variable
   aws appconfig create-hosted-configuration-version --application-id $APP_ID ...
   ```

4. **Add meaningful descriptions to versions and deployments**
   ```bash
   # Include context in descriptions
   --description "Build: 123 | Commit: a1b2c3d | Author: john | Date: 2026-02-18"
   ```

5. **Use --no-fail-on-empty-changeset for idempotent CFT deploys**
   ```bash
   # Won't fail if no changes
   aws cloudformation deploy \
     --stack-name my-stack \
     --template-file template.yml \
     --no-fail-on-empty-changeset
   ```

### Configuration Management

6. **Validate JSON before creating versions**
   ```bash
   # Always validate first
   jq empty config.json || exit 1
   
   # Then create version
   aws appconfig create-hosted-configuration-version ...
   ```

7. **Monitor deployments in CI/CD pipelines**
   ```bash
   # Don't start and forget - monitor until complete
   while [ "$STATE" != "COMPLETE" ]; do
     # Check status
     sleep 10
   done
   ```

8. **Keep audit trail with git commit info in descriptions**
   ```bash
   COMMIT=$(git rev-parse --short HEAD)
   --description "Git commit: $COMMIT"
   ```

### Deployment Strategies

9. **Use jq for JSON processing**
   ```bash
   # Parse complex JSON responses
   aws appconfig get-deployment ... --output json | jq '.State'
   
   # Format for readability
   cat config.json | jq .
   ```

10. **Error handling in scripts**
    ```bash
    # Exit on error
    set -e
    
    # Check exit codes
    if ! aws appconfig ...; then
      echo "Failed"
      exit 1
    fi
    
    # Use trap for cleanup
    trap 'cleanup_on_error' ERR
    ```

### Performance and Reliability

11. **Disable pager for CI/CD**
    ```bash
    export AWS_PAGER=""
    # Or
    aws appconfig list-applications --no-cli-pager
    ```

12. **Use --query to reduce data transfer**
    ```bash
    # Only get what you need
    aws cloudformation describe-stacks \
      --query 'Stacks[0].Outputs[?OutputKey==`ApplicationId`].OutputValue' \
      --output text
    ```

13. **Implement retry logic for transient failures**
    ```bash
    for i in {1..3}; do
      if aws appconfig start-deployment ...; then
        break
      fi
      sleep 5
    done
    ```

14. **Use CloudFormation for infrastructure (not manual creation)**
    - Keeps infrastructure as code
    - Version controlled
    - Repeatable and consistent

15. **Test in lower environments first**
    - Always test: dev → test → staging → prod
    - Use immediate strategy in dev/test
    - Use canary strategy in production

### Security

16. **Use IAM roles instead of access keys where possible**
    ```bash
    # On EC2/ECS/Lambda - use IAM roles (automatically used)
    # No need to configure credentials
    ```

17. **Rotate deployment strategies based on risk**
    - Critical production changes: Canary
    - Non-critical or dev changes: Immediate
    - Emergency hotfixes: Immediate

18. **Review deployed configuration**
    ```bash
    # Always verify after deployment
    aws appconfig get-configuration ... /tmp/deployed.json
    diff source.json /tmp/deployed.json
    ```

### Documentation

19. **Document your configuration schema**
    - Add comments to config files (use separate documentation)
    - Maintain feature flag registry
    - Track deprecation timeline

20. **Log all deployments**
    ```bash
    # Keep deployment logs
    ./deploy-config.sh 2>&1 | tee deploy-$(date +%Y%m%d-%H%M%S).log
    ```

---

## 12. Real-World Examples

### 12.1 Dev/Test/Prod Deployment Pipeline

```bash
#!/bin/bash
# pipeline-deploy.sh
# Complete CI/CD pipeline for deploying configurations

set -e

ASSET_ID="10"
ASSET_NAME="auth"
ASSET_SERVICE="authapi"
AWS_REGION="us-east-1"

# Configuration file (same for all environments, but content differs)
ENVIRONMENTS=("cdc1c" "cdc4c" "cdc7c" "pdc1c")
ENV_NAMES=("Dev" "Test" "Staging" "Production")
STRATEGIES=("Immediate" "Immediate" "Canary" "Canary")

echo "╔════════════════════════════════════════════════════════╗"
echo "║     Multi-Stage Configuration Deployment Pipeline     ║"
echo "╚════════════════════════════════════════════════════════╝"
echo ""

# Step 1: Validate all configuration files
echo "Step 1: Validating configuration files..."
for ENV in "${ENVIRONMENTS[@]}"; do
  CONFIG_FILE="production-approach/config/${ENV}.json"
  
  if [ ! -f "$CONFIG_FILE" ]; then
    echo "✗ Missing config file: $CONFIG_FILE"
    exit 1
  fi
  
  if ! jq empty "$CONFIG_FILE" 2>/dev/null; then
    echo "✗ Invalid JSON in $CONFIG_FILE"
    exit 1
  fi
  
  echo "✓ $ENV configuration valid"
done

echo ""

# Step 2: Deploy to each environment with approval gates
for i in "${!ENVIRONMENTS[@]}"; do
  ENV="${ENVIRONMENTS[$i]}"
  ENV_NAME="${ENV_NAMES[$i]}"
  STRATEGY="${STRATEGIES[$i]}"
  
  echo "======================================================================"
  echo "Deploying to $ENV_NAME ($ENV) with $STRATEGY strategy"
  echo "======================================================================"
  
  # Production requires approval
  if [ "$ENV" = "pdc1c" ]; then
    echo ""
    echo "⚠️  PRODUCTION DEPLOYMENT"
    echo "This will deploy to PRODUCTION environment"
    read -p "Type 'deploy-to-production' to continue: " APPROVAL
    
    if [ "$APPROVAL" != "deploy-to-production" ]; then
      echo "✗ Production deployment cancelled"
      exit 1
    fi
  fi
  
  # Run deployment
  if ASSET_GROUP=$ENV DEPLOYMENT_STRATEGY=$STRATEGY ./deploy-config.sh; then
    echo "✓ Successfully deployed to $ENV_NAME"
    
    # Post-deployment verification
    echo "Running post-deployment checks..."
    
    # Wait for application to pick up config (30 seconds)
    echo "Waiting 30 seconds for applications to refresh..."
    sleep 30
    
    # Add health checks here
    echo "✓ Post-deployment verification passed"
  else
    echo "✗ Deployment to $ENV_NAME failed"
    exit 1
  fi
  
  # Wait between stages (except after last)
  if [ $i -lt $((${#ENVIRONMENTS[@]} - 1)) ]; then
    NEXT_ENV="${ENV_NAMES[$((i + 1))]}"
    echo ""
    echo "Pausing before deploying to $NEXT_ENV..."
    read -p "Press Enter to continue or Ctrl+C to abort..."
  fi
  
  echo ""
done

echo "╔════════════════════════════════════════════════════════╗"
echo "║       All Environments Deployed Successfully!         ║"
echo "╚════════════════════════════════════════════════════════╝"
```

### 12.2 Feature Flag Toggle Workflow

```bash
#!/bin/bash
# toggle-feature-flag.sh
# Quick script to toggle a feature flag on/off

ASSET_ID="10"
ASSET_NAME="auth"
ASSET_GROUP="${1:-cdc1c}"
FEATURE_FLAG="${2}"
NEW_VALUE="${3}"

if [ -z "$FEATURE_FLAG" ] || [ -z "$NEW_VALUE" ]; then
  echo "Usage: $0 <environment> <feature-flag-name> <true|false>"
  echo ""
  echo "Example: $0 cdc1c enableNewAuth true"
  exit 1
fi

CONFIG_FILE="production-approach/config/${ASSET_GROUP}.json"

echo "=== Feature Flag Toggle ==="
echo "Environment: $ASSET_GROUP"
echo "Feature Flag: $FEATURE_FLAG"
echo "New Value: $NEW_VALUE"
echo ""

# Backup current config
cp "$CONFIG_FILE" "${CONFIG_FILE}.backup"

# Get current value
CURRENT_VALUE=$(jq -r ".featureFlags.${FEATURE_FLAG}" "$CONFIG_FILE")
echo "Current value: $CURRENT_VALUE"

if [ "$CURRENT_VALUE" = "$NEW_VALUE" ]; then
  echo "✓ Feature flag already set to $NEW_VALUE"
  rm "${CONFIG_FILE}.backup"
  exit 0
fi

# Update feature flag
jq ".featureFlags.${FEATURE_FLAG} = ${NEW_VALUE}" "$CONFIG_FILE" > "${CONFIG_FILE}.tmp"
mv "${CONFIG_FILE}.tmp" "$CONFIG_FILE"

NEW_CHECK=$(jq -r ".featureFlags.${FEATURE_FLAG}" "$CONFIG_FILE")
echo "Updated value: $NEW_CHECK"

# Commit change
git add "$CONFIG_FILE"
git commit -m "Toggle $FEATURE_FLAG to $NEW_VALUE in $ASSET_GROUP"

echo ""
read -p "Deploy this change now? (yes/no): " DEPLOY

if [ "$DEPLOY" = "yes" ]; then
  # Deploy with immediate strategy (for quick toggle)
  ASSET_GROUP=$ASSET_GROUP DEPLOYMENT_STRATEGY=Immediate ./deploy-config.sh
else
  echo "Change committed but not deployed"
  echo "Run: ASSET_GROUP=$ASSET_GROUP ./deploy-config.sh"
fi

rm "${CONFIG_FILE}.backup"
```

### 12.3 Emergency Rollback Procedure

```bash
#!/bin/bash
# emergency-rollback.sh
# Emergency procedure to rollback to last known good configuration

set -e

ASSET_ID="10"
ASSET_NAME="auth"
ASSET_GROUP="${1:-cdc1c}"
AWS_REGION="us-east-1"

STACK_NAME="appconfig-${ASSET_NAME}-${ASSET_ID}-${ASSET_GROUP}"

echo "╔════════════════════════════════════════════════════════╗"
echo "║            EMERGENCY ROLLBACK PROCEDURE                ║"
echo "╚════════════════════════════════════════════════════════╝"
echo ""
echo "⚠️  WARNING: This will immediately rollback to the previous version"
echo "Environment: $ASSET_GROUP"
echo ""
read -p "Type 'ROLLBACK' to confirm: " CONFIRM

if [ "$CONFIRM" != "ROLLBACK" ]; then
  echo "Rollback cancelled"
  exit 0
fi

# Get resource IDs
echo "Retrieving resource IDs..."
APP_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`ApplicationId`].OutputValue' \
  --output text)

ENV_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`EnvironmentId`].OutputValue' \
  --output text)

PROFILE_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`ConfigurationProfileId`].OutputValue' \
  --output text)

STRATEGY_ID=$(aws cloudformation describe-stacks \
  --stack-name $STACK_NAME \
  --region $AWS_REGION \
  --query 'Stacks[0].Outputs[?OutputKey==`DeploymentStrategyImmediateId`].OutputValue' \
  --output text)

# Get last 5 versions
echo ""
echo "Available versions:"
aws appconfig list-hosted-configuration-versions \
  --application-id $APP_ID \
  --configuration-profile-id $PROFILE_ID \
  --region $AWS_REGION \
  --output json | jq -r '.Items[0:5][] | "Version \(.VersionNumber): \(.Description)"'

echo ""
read -p "Enter version number to rollback to (or press Enter for previous version): " VERSION

if [ -z "$VERSION" ]; then
  # Get second most recent version (skip latest which is current)
  VERSION=$(aws appconfig list-hosted-configuration-versions \
    --application-id $APP_ID \
    --configuration-profile-id $PROFILE_ID \
    --region $AWS_REGION \
    --query 'Items[1].VersionNumber' \
    --output text)
fi

echo ""
echo "Rolling back to version $VERSION..."

# Start rollback deployment
DEPLOYMENT_NUMBER=$(aws appconfig start-deployment \
  --application-id $APP_ID \
  --environment-id $ENV_ID \
  --configuration-profile-id $PROFILE_ID \
  --configuration-version $VERSION \
  --deployment-strategy-id $STRATEGY_ID \
  --description "EMERGENCY ROLLBACK to version $VERSION at $(date -u '+%Y-%m-%d %H:%M:%S UTC')" \
  --region $AWS_REGION \
  --output json | jq -r '.DeploymentNumber')

echo "✓ Rollback deployment $DEPLOYMENT_NUMBER started"

# Monitor with progress
MAX_ATTEMPTS=30
ATTEMPT=0

echo "Monitoring rollback progress..."
while [ $ATTEMPT -lt $MAX_ATTEMPTS ]; do
  ATTEMPT=$((ATTEMPT + 1))
  sleep 10
  
  DEPLOYMENT=$(aws appconfig get-deployment \
    --application-id $APP_ID \
    --environment-id $ENV_ID \
    --deployment-number $DEPLOYMENT_NUMBER \
    --region $AWS_REGION \
    --output json)
  
  STATE=$(echo $DEPLOYMENT | jq -r '.State')
  PERCENTAGE=$(echo $DEPLOYMENT | jq -r '.PercentageComplete')
  
  printf "\r[%s] Status: %-12s Progress: %3d%%" "$(date '+%H:%M:%S')" "$STATE" "$PERCENTAGE"
  
  if [ "$STATE" = "COMPLETE" ]; then
    echo ""
    echo "✓ ROLLBACK COMPLETE"
    break
  elif [ "$STATE" = "ROLLED_BACK" ]; then
    echo ""
    echo "✗ ROLLBACK FAILED"
    exit 1
  fi
done

if [ "$STATE" != "COMPLETE" ]; then
  echo ""
  echo "✗ Rollback monitoring timeout"
  exit 1
fi

# Verify rollback
echo ""
echo "Verifying rollback..."
aws appconfig get-hosted-configuration-version \
  --application-id $APP_ID \
  --configuration-profile-id $PROFILE_ID \
  --version-number $VERSION \
  --region $AWS_REGION \
  /tmp/rollback-config.json

echo "Rolled back configuration:"
jq . /tmp/rollback-config.json

rm /tmp/rollback-config.json

echo ""
echo "╔════════════════════════════════════════════════════════╗"
echo "║      Emergency Rollback Completed Successfully        ║"
echo "╚════════════════════════════════════════════════════════╝"
echo ""
echo "Next steps:"
echo "1. Verify application health"
echo "2. Investigate root cause of issue"
echo "3. Update Git repository to match rolled-back config"
echo "4. Create incident report"
```

### 12.4 Scheduled Configuration Updates

```bash
#!/bin/bash
# scheduled-config-update.sh
# For use in cron jobs or scheduled tasks

# This script is designed to be run via cron for scheduled config updates
# Example crontab entry:
# 0 2 * * * /path/to/scheduled-config-update.sh >> /var/log/appconfig-updates.log 2>&1

set -e

TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
LOG_PREFIX="[$TIMESTAMP]"

echo "$LOG_PREFIX Starting scheduled configuration update"

# Configuration
ASSET_ID="10"
ASSET_NAME="auth"
ASSET_GROUP="cdc1c"
CONFIG_FILE="production-approach/config/${ASSET_GROUP}.json"
GIT_REPO_PATH="/opt/appconfig/aws-appconfig-demo"

# Change to repo directory
cd "$GIT_REPO_PATH"

# Pull latest changes from Git
echo "$LOG_PREFIX Pulling latest changes from Git..."
git fetch origin main
git reset --hard origin/main

# Check if config file changed
CHANGES=$(git diff HEAD@{1} HEAD -- "$CONFIG_FILE" | wc -l)

if [ $CHANGES -eq 0 ]; then
  echo "$LOG_PREFIX No changes detected in $CONFIG_FILE"
  exit 0
fi

echo "$LOG_PREFIX Changes detected, deploying..."

# Deploy the changes
if ASSET_GROUP=$ASSET_GROUP DEPLOYMENT_STRATEGY=Immediate ./deploy-config.sh; then
  echo "$LOG_PREFIX ✓ Deployment successful"
  
  # Send success notification (optional)
  # ./send-notification.sh "success" "Scheduled config update completed"
else
  echo "$LOG_PREFIX ✗ Deployment failed"
  
  # Send failure notification
  # ./send-notification.sh "failure" "Scheduled config update failed"
  
  exit 1
fi

echo "$LOG_PREFIX Scheduled update completed"
```

### 12.5 Blue-Green Deployment Pattern

```bash
#!/bin/bash
# blue-green-deploy.sh
# Blue-green deployment for zero-downtime configuration updates

set -e

ASSET_ID="10"
ASSET_NAME="auth"
ASSET_SERVICE="authapi"
CONFIG_FILE="$1"
AWS_REGION="us-east-1"

if [ -z "$CONFIG_FILE" ]; then
  echo "Usage: $0 <config-file>"
  exit 1
fi

echo "╔════════════════════════════════════════════════════════╗"
echo "║         Blue-Green Configuration Deployment           ║"
echo "╚════════════════════════════════════════════════════════╝"
echo ""

# Blue environment: cdc7c (staging/pre-prod)
# Green environment: pdc1c (production)

BLUE_ENV="cdc7c"
GREEN_ENV="pdc1c"

# Step 1: Deploy to Blue environment
echo "Step 1: Deploying to BLUE environment ($BLUE_ENV)..."
if ASSET_GROUP=$BLUE_ENV DEPLOYMENT_STRATEGY=Immediate CONFIG_FILE=$CONFIG_FILE ./deploy-config.sh; then
  echo "✓ Blue deployment successful"
else
  echo "✗ Blue deployment failed"
  exit 1
fi

# Step 2: Run smoke tests on Blue
echo ""
echo "Step 2: Running smoke tests on Blue environment..."
sleep 30  # Wait for applications to pick up config

# Add your smoke tests here
# Example: curl health endpoints, check metrics, run integration tests
echo "✓ Smoke tests passed"

# Step 3: Get approval for Green deployment
echo ""
echo "Step 3: Blue environment validated successfully"
read -p "Deploy to GREEN (production) environment? (yes/no): " APPROVE

if [ "$APPROVE" != "yes" ]; then
  echo "✗ Green deployment cancelled"
  exit 0
fi

# Step 4: Deploy to Green with canary strategy
echo ""
echo "Step 4: Deploying to GREEN environment ($GREEN_ENV) with Canary strategy..."
if ASSET_GROUP=$GREEN_ENV DEPLOYMENT_STRATEGY=Canary CONFIG_FILE=$CONFIG_FILE ./deploy-config.sh; then
  echo "✓ Green deployment successful"
else
  echo "✗ Green deployment failed"
  echo "Blue environment still has the new configuration"
  exit 1
fi

# Step 5: Monitor production metrics
echo ""
echo "Step 5: Monitoring production metrics..."
echo "Check the following:"
echo "  - Error rates"
echo "  - Response times"
echo "  - Application logs"
echo "  - Business metrics"
echo ""
read -p "Are production metrics healthy? (yes/no): " HEALTHY

if [ "$HEALTHY" != "yes" ]; then
  echo "⚠️  Production issues detected!"
  read -p "Rollback production? (yes/no): " ROLLBACK
  
  if [ "$ROLLBACK" = "yes" ]; then
    echo "Initiating rollback..."
    ASSET_GROUP=$GREEN_ENV ./emergency-rollback.sh
  fi
  
  exit 1
fi

echo ""
echo "╔════════════════════════════════════════════════════════╗"
echo "║    Blue-Green Deployment Completed Successfully       ║"
echo "╚════════════════════════════════════════════════════════╝"
echo ""
echo "Both Blue and Green environments now have the new configuration"
echo "Production is stable and healthy"
```

---

## Additional Resources

### Official Documentation

- [AWS AppConfig Documentation](https://docs.aws.amazon.com/appconfig/)
- [AWS CLI AppConfig Reference](https://awscli.amazonaws.com/v2/documentation/api/latest/reference/appconfig/index.html)
- [CloudFormation AppConfig Resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/AWS_AppConfig.html)
- [AWS AppConfig Best Practices](https://docs.aws.amazon.com/appconfig/latest/userguide/appconfig-best-practices.html)

### Related Tools

- [jq JSON Processor](https://stedolan.github.io/jq/)
- [AWS CLI v2](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html)
- [Git Version Control](https://git-scm.com/)

### Community Resources

- [AWS AppConfig GitHub](https://github.com/aws/aws-appconfig)
- [AWS Samples Repository](https://github.com/aws-samples/)
- [Feature Toggles Pattern (Martin Fowler)](https://martinfowler.com/articles/feature-toggles.html)

---

## Support and Feedback

For issues, questions, or suggestions:

1. Check this documentation first
2. Review CloudFormation stack events
3. Check AppConfig deployment logs
4. Create an issue in the repository
5. Contact the DevOps team

---

**Document Version**: 1.0.0  
**Last Updated**: 2026-02-18  
**Maintained By**: DevOps Team  
**Repository**: [aws-appconfig-demo](https://github.com/deepak78194/aws-appconfig-demo)

---

## Changelog

### Version 1.0.0 (2026-02-18)
- Initial comprehensive documentation
- All 12 sections completed
- Production-ready scripts included
- Real-world examples added
- Best practices documented
