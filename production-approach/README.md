# Production Approach: Git-based AWS AppConfig Configuration Management

## Table of Contents
- [Overview](#overview)
- [Architecture](#architecture)
- [Why This Approach for Production](#why-this-approach-for-production)
- [Asset Naming Convention](#asset-naming-convention)
- [Prerequisites](#prerequisites)
- [Setup Guide](#setup-guide)
- [Usage Instructions](#usage-instructions)
- [Jenkins Pipeline Parameters](#jenkins-pipeline-parameters)
- [Troubleshooting](#troubleshooting)
- [Real-world Usage Scenarios](#real-world-usage-scenarios)

## Overview

This production approach implements a **Git-based configuration management system** for AWS AppConfig, where:

- **Infrastructure as Code**: AppConfig resources (Application, Environment, Configuration Profile, Deployment Strategies) are managed via CloudFormation templates
- **Git as Source of Truth**: All configuration files are stored in Git repository with version control
- **Automated Deployments**: Jenkins pipeline automates the deployment process using AWS CLI
- **Environment Separation**: Each environment (CDC1C, CDC4C, CDC7C, PDC1C) has its own configuration file
- **Audit Trail**: Every deployment is tracked with Git commit SHA for full traceability
- **Safe Rollouts**: Support for both immediate and canary deployment strategies

### Key Benefits

✅ **Version Control**: All configuration changes tracked in Git with full history  
✅ **Code Review**: Configuration changes go through PR review process  
✅ **Rollback Capability**: Easy rollback to any previous configuration version  
✅ **Consistency**: Same configuration file structure across all environments  
✅ **Automation**: Fully automated deployment pipeline  
✅ **Safety**: Validation and verification at every step  

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                         Git Repository                          │
│  ┌───────────────────┐         ┌──────────────────────────┐   │
│  │  Configuration    │         │  CloudFormation          │   │
│  │  Files (JSON)     │         │  Templates (YAML)        │   │
│  │  - cdc1c.json     │         │  - appconfig-service.yml │   │
│  │  - cdc4c.json     │         │                          │   │
│  │  - cdc7c.json     │         │                          │   │
│  │  - pdc1c.json     │         │                          │   │
│  └───────────────────┘         └──────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │   Jenkins Pipeline    │
                │  (Groovy Jenkinsfile) │
                └───────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
    ┌───────────────┐ ┌──────────┐ ┌──────────────┐
    │ CloudFormation│ │ AWS CLI  │ │  AppConfig   │
    │     Stack     │ │ Commands │ │     API      │
    └───────────────┘ └──────────┘ └──────────────┘
            │               │               │
            └───────────────┼───────────────┘
                            ▼
                ┌───────────────────────┐
                │   AWS AppConfig       │
                │  - Application        │
                │  - Environment        │
                │  - Config Profile     │
                │  - Deployments        │
                └───────────────────────┘
                            │
                            ▼
                ┌───────────────────────┐
                │  Application Services │
                │  (ECS, Lambda, EC2)   │
                └───────────────────────┘
```

### Workflow Flow

1. **Developer** makes configuration changes in Git and creates Pull Request
2. **Code Review** team reviews and approves the PR
3. **Merge** to main branch triggers or allows manual Jenkins pipeline execution
4. **Jenkins Pipeline**:
   - Validates JSON syntax
   - Checks if infrastructure exists
   - Retrieves AppConfig resource IDs from CloudFormation
   - Creates new configuration version from Git file
   - Starts deployment with selected strategy
   - Monitors deployment progress
   - Verifies deployed configuration
5. **AppConfig** distributes configuration to application instances
6. **Applications** fetch updated configuration and apply changes

## Why This Approach for Production

### 1. **Separation of Concerns**
- **Infrastructure** (AppConfig resources) managed separately from **Configuration** (feature flags, settings)
- Infrastructure changes are rare and controlled via CloudFormation
- Configuration changes are frequent and automated via pipeline

### 2. **Git as Single Source of Truth**
- All configuration stored in version control
- Full history of all changes with author, timestamp, and reason
- Easy to see what changed, when, and why
- Ability to rollback to any previous version

### 3. **Code Review Process**
- Configuration changes go through PR workflow
- Team can review, discuss, and approve changes before deployment
- Prevents unauthorized or erroneous configuration changes
- Documentation of decisions in PR comments

### 4. **Automated Testing & Validation**
- JSON syntax validation before deployment
- Schema validation via AppConfig validators
- Configuration comparison after deployment
- Automated rollback on failure

### 5. **Environment Consistency**
- Same configuration structure across all environments
- Clear separation of environment-specific values
- Easy to promote configurations from dev → staging → production
- Reduces configuration drift

### 6. **Audit Trail & Compliance**
- Every deployment tagged with Git commit SHA
- Full audit log of who deployed what and when
- CloudFormation change sets show infrastructure modifications
- Meets compliance requirements for change tracking

### 7. **Safety & Reliability**
- Multiple deployment strategies (immediate vs canary)
- Monitoring and verification at each step
- Automatic validation checks prevent bad configurations
- Easy rollback mechanism

## Asset Naming Convention

The naming convention provides a structured way to organize and identify AppConfig resources:

### Components

| Component | Description | Example | Pattern |
|-----------|-------------|---------|---------|
| **AssetId** | Numeric identifier for the asset/service | `10` | `[0-9]+` |
| **AssetName** | Short name for the asset/service | `auth` | `[a-z][a-z0-9-]*` |
| **AssetGroup** | Environment/cluster identifier | `cdc1c`, `pdc1c` | `cdc1c\|cdc4c\|cdc7c\|pdc1c` |
| **AssetService** | Specific service name | `authapi` | `[a-z][a-z0-9-]*` |

### Examples

#### Example 1: Authentication Service
- **AssetId**: `10`
- **AssetName**: `auth`
- **AssetGroup**: `cdc1c` (Dev cluster)
- **AssetService**: `authapi`
- **Stack Name**: `appconfig-auth-10-cdc1c`
- **Application Name**: `auth-10`
- **Config File**: `config/cdc1c.json`

#### Example 2: Payment Service
- **AssetId**: `20`
- **AssetName**: `payment`
- **AssetGroup**: `pdc1c` (Production cluster)
- **AssetService**: `paymentapi`
- **Stack Name**: `appconfig-payment-20-pdc1c`
- **Application Name**: `payment-20`
- **Config File**: `config/pdc1c.json`

#### Example 3: Notification Service
- **AssetId**: `30`
- **AssetName**: `notification`
- **AssetGroup**: `cdc4c` (Test cluster)
- **AssetService**: `notificationapi`
- **Stack Name**: `appconfig-notification-30-cdc4c`
- **Application Name**: `notification-30`
- **Config File**: `config/cdc4c.json`

### Naming Rules

1. **AssetId** must be unique across all services
2. **AssetName** should be short and descriptive (3-20 characters)
3. **AssetGroup** maps directly to environment and configuration file
4. **Stack names** follow pattern: `appconfig-{AssetName}-{AssetId}-{AssetGroup}`
5. **Application names** follow pattern: `{AssetName}-{AssetId}`

## Prerequisites

### Required Tools & Access
- ✅ AWS Account with appropriate permissions
- ✅ Jenkins server with AWS CLI installed
- ✅ Git repository access
- ✅ AWS credentials configured in Jenkins

### Required AWS Permissions

The Jenkins execution role/user needs the following permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "cloudformation:CreateStack",
        "cloudformation:UpdateStack",
        "cloudformation:DescribeStacks",
        "cloudformation:DescribeStackEvents",
        "cloudformation:GetTemplate"
      ],
      "Resource": "arn:aws:cloudformation:*:*:stack/appconfig-*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "appconfig:CreateApplication",
        "appconfig:CreateEnvironment",
        "appconfig:CreateConfigurationProfile",
        "appconfig:CreateDeploymentStrategy",
        "appconfig:CreateHostedConfigurationVersion",
        "appconfig:StartDeployment",
        "appconfig:GetDeployment",
        "appconfig:GetHostedConfigurationVersion",
        "appconfig:ListHostedConfigurationVersions",
        "appconfig:TagResource"
      ],
      "Resource": "*"
    }
  ]
}
```

### Jenkins Configuration

1. **AWS CLI Installation**:
   ```bash
   # On Jenkins server
   curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
   unzip awscliv2.zip
   sudo ./aws/install
   ```

2. **AWS Credentials**:
   - Configure AWS credentials in Jenkins (Manage Jenkins → Credentials)
   - Or use IAM role if Jenkins runs on EC2

3. **Pipeline Setup**:
   - Create new Pipeline job in Jenkins
   - Point to `production-approach/pipelines/appconfig-deploy.jenkinsfile`
   - Configure parameters (or use defaults)

## Setup Guide

### Step 1: Clone Repository

```bash
git clone https://github.com/your-org/aws-appconfig-demo.git
cd aws-appconfig-demo
```

### Step 2: Review Configuration Files

Check the configuration files in `production-approach/config/`:

```bash
# View CDC1C (dev) configuration
cat production-approach/config/cdc1c.json

# View PDC1C (production) configuration
cat production-approach/config/pdc1c.json
```

### Step 3: Customize for Your Service

Edit the configuration files to match your service requirements:

```json
{
  "featureFlags": {
    "enableNewAuth": false,
    "enableRateLimiting": true,
    "maxRetryAttempts": 3
  },
  "settings": {
    "sessionTimeout": 3600,
    "tokenExpiry": 7200,
    "apiTimeout": 30
  }
}
```

### Step 4: Create Infrastructure

Run the Jenkins pipeline with **ACTION = create-infra**:

```groovy
// Jenkins Pipeline Parameters
ACTION: create-infra
ASSET_GROUP: cdc1c
ASSET_ID: 10
ASSET_NAME: auth
ASSET_SERVICE: authapi
DEPLOY_INITIAL_CONFIG: true  // Optional: deploy config during creation
```

This creates:
- AppConfig Application: `auth-10`
- AppConfig Environment: `cdc1c`
- Configuration Profile: `authapi-config`
- Two Deployment Strategies: `auth-10-Immediate` and `auth-10-Canary`

**CloudFormation Stack**: `appconfig-auth-10-cdc1c`

### Step 5: Verify Infrastructure

```bash
# Check CloudFormation stack
aws cloudformation describe-stacks \
  --stack-name appconfig-auth-10-cdc1c \
  --query 'Stacks[0].Outputs'

# List stack outputs
aws cloudformation list-stack-resources \
  --stack-name appconfig-auth-10-cdc1c
```

### Step 6: Deploy Configuration

Run the Jenkins pipeline with **ACTION = deploy-config**:

```groovy
// Jenkins Pipeline Parameters
ACTION: deploy-config
ASSET_GROUP: cdc1c
ASSET_ID: 10
ASSET_NAME: auth
ASSET_SERVICE: authapi
DEPLOYMENT_STRATEGY: Canary  // or Immediate
```

This will:
1. Read `config/cdc1c.json` from Git
2. Create new hosted configuration version
3. Start deployment to `cdc1c` environment
4. Monitor deployment progress
5. Verify deployed configuration

## Usage Instructions

### Deploying Configuration Updates

#### Via Pull Request Workflow (Recommended)

1. **Create feature branch**:
   ```bash
   git checkout -b feature/update-auth-timeout
   ```

2. **Edit configuration file**:
   ```bash
   vim production-approach/config/cdc1c.json
   # Change sessionTimeout from 3600 to 7200
   ```

3. **Commit and push**:
   ```bash
   git add production-approach/config/cdc1c.json
   git commit -m "Increase session timeout to 2 hours for CDC1C"
   git push origin feature/update-auth-timeout
   ```

4. **Create Pull Request**:
   - Open PR on GitHub
   - Add description explaining the change
   - Request review from team

5. **Code Review**:
   - Team reviews the configuration change
   - Discusses impact and timing
   - Approves PR

6. **Merge PR**:
   - Merge to main branch
   - This triggers CI or allows manual Jenkins run

7. **Run Jenkins Pipeline**:
   - Execute pipeline with `ACTION=deploy-config`
   - Select appropriate `ASSET_GROUP`
   - Choose deployment strategy

8. **Monitor Deployment**:
   - Jenkins shows real-time progress
   - Check AWS AppConfig console for deployment status

#### Direct Jenkins Execution

For urgent changes or testing:

1. Open Jenkins job
2. Click "Build with Parameters"
3. Fill in parameters:
   - `ACTION`: `deploy-config`
   - `ASSET_GROUP`: Target environment
   - `ASSET_ID`: Your asset ID (e.g., `10`)
   - `ASSET_NAME`: Your asset name (e.g., `auth`)
   - `ASSET_SERVICE`: Your service name (e.g., `authapi`)
   - `DEPLOYMENT_STRATEGY`: `Immediate` or `Canary`
4. Click "Build"

### Updating Infrastructure

When you need to modify AppConfig infrastructure (rare):

1. **Edit CloudFormation template**:
   ```bash
   vim production-approach/infra/appconfig-service.yml
   ```

2. **Commit changes**:
   ```bash
   git add production-approach/infra/appconfig-service.yml
   git commit -m "Update deployment strategy timings"
   git push
   ```

3. **Run Jenkins pipeline**:
   ```groovy
   ACTION: update-infra
   ASSET_GROUP: cdc1c
   ASSET_ID: 10
   ASSET_NAME: auth
   ASSET_SERVICE: authapi
   ```

### Rollback Procedures

#### Option 1: Rollback via Git Revert

1. **Find the previous working commit**:
   ```bash
   git log --oneline production-approach/config/cdc1c.json
   ```

2. **Revert to previous version**:
   ```bash
   git revert <commit-sha>
   # or
   git checkout <previous-commit-sha> -- production-approach/config/cdc1c.json
   git commit -m "Rollback to previous configuration"
   git push
   ```

3. **Deploy reverted configuration**:
   - Run Jenkins pipeline with `ACTION=deploy-config`

#### Option 2: Rollback via AppConfig Console

1. Go to AWS AppConfig Console
2. Navigate to your Application → Configuration Profile
3. View deployment history
4. Select previous version
5. Click "Start deployment" with previous version number

#### Option 3: Emergency Rollback via AWS CLI

```bash
# List available versions
aws appconfig list-hosted-configuration-versions \
  --application-id <app-id> \
  --configuration-profile-id <profile-id>

# Deploy previous version
aws appconfig start-deployment \
  --application-id <app-id> \
  --environment-id <env-id> \
  --configuration-profile-id <profile-id> \
  --configuration-version <previous-version-number> \
  --deployment-strategy-id <strategy-id> \
  --description "Emergency rollback"
```

## Jenkins Pipeline Parameters

### ACTION
**Type**: Choice  
**Options**: `deploy-config`, `create-infra`, `update-infra`  
**Description**: 
- `deploy-config`: Deploy configuration from Git to AppConfig (most common)
- `create-infra`: Create new CloudFormation stack with AppConfig resources
- `update-infra`: Update existing CloudFormation stack

### ASSET_GROUP
**Type**: Choice  
**Options**: `cdc1c`, `cdc4c`, `cdc7c`, `pdc1c`  
**Description**: Environment/cluster identifier that maps to configuration file
- `cdc1c`: Development cluster 1
- `cdc4c`: Development cluster 4
- `cdc7c`: Test/Staging cluster 7
- `pdc1c`: Production cluster 1

### ASSET_ID
**Type**: String  
**Default**: `10`  
**Pattern**: `^[0-9]+$`  
**Description**: Numeric identifier for the asset/service. Must be unique across all services.

### ASSET_NAME
**Type**: String  
**Default**: `auth`  
**Pattern**: `^[a-z][a-z0-9-]*$`  
**Description**: Short descriptive name for the asset/service (e.g., auth, payment, notification)

### ASSET_SERVICE
**Type**: String  
**Default**: `authapi`  
**Pattern**: `^[a-z][a-z0-9-]*$`  
**Description**: Specific service name (e.g., authapi, paymentapi, notificationapi)

### DEPLOYMENT_STRATEGY
**Type**: Choice  
**Options**: `Immediate`, `Canary`  
**Description**:
- `Immediate`: Instant deployment with 5-minute bake time (use for dev/test)
- `Canary`: Gradual rollout - 10% for 2 minutes, then 100% with 5-minute bake (use for production)

### DEPLOY_INITIAL_CONFIG
**Type**: Boolean  
**Default**: `false`  
**Description**: Whether to deploy initial configuration when creating infrastructure (only for `create-infra` action)

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: "Stack does not exist"

**Error Message**: 
```
CloudFormation stack appconfig-auth-10-cdc1c does not exist
```

**Solution**:
1. Run pipeline with `ACTION=create-infra` first
2. Verify stack name follows naming convention: `appconfig-{AssetName}-{AssetId}-{AssetGroup}`
3. Check AWS region matches your configuration

```bash
# Verify stack exists
aws cloudformation describe-stacks --stack-name appconfig-auth-10-cdc1c
```

#### Issue 2: "Invalid JSON in configuration file"

**Error Message**:
```
Invalid JSON in configuration file: Unexpected character at line 5
```

**Solution**:
1. Validate JSON syntax using online validator or:
   ```bash
   cat production-approach/config/cdc1c.json | python -m json.tool
   ```
2. Check for:
   - Missing commas
   - Trailing commas (not allowed in JSON)
   - Unquoted keys or values
   - Incorrect brackets/braces

#### Issue 3: "Configuration missing required sections"

**Error Message**:
```
Configuration missing required 'featureFlags' section
```

**Solution**:
Ensure your JSON has both required sections:
```json
{
  "featureFlags": { },
  "settings": { }
}
```

#### Issue 4: "Permission denied" or "Access Denied"

**Error Message**:
```
User: arn:aws:iam::123456789012:user/jenkins is not authorized to perform: appconfig:CreateHostedConfigurationVersion
```

**Solution**:
1. Verify Jenkins has correct AWS credentials
2. Check IAM permissions include required AppConfig and CloudFormation actions
3. Review the [Required AWS Permissions](#required-aws-permissions) section

```bash
# Test AWS credentials
aws sts get-caller-identity
```

#### Issue 5: "Deployment rolled back"

**Error Message**:
```
Deployment was rolled back
```

**Solution**:
1. Check AppConfig console for deployment details and rollback reason
2. Common causes:
   - Configuration validation failed
   - CloudWatch alarm triggered
   - Manual rollback initiated
3. Review configuration for errors
4. Check application logs during deployment

```bash
# Get deployment details
aws appconfig get-deployment \
  --application-id <app-id> \
  --environment-id <env-id> \
  --deployment-number <deployment-num>
```

#### Issue 6: "Stack update has no changes"

**Error Message**:
```
No updates are to be performed
```

**Solution**:
- This is informational, not an error
- CloudFormation detected no changes in the template or parameters
- Pipeline continues successfully

#### Issue 7: "Configuration file not found"

**Error Message**:
```
Configuration file not found: production-approach/config/cdc1c.json
```

**Solution**:
1. Verify file exists in repository
2. Check file path is correct
3. Ensure file is committed and pushed to Git
4. Verify Jenkins workspace is up to date

```bash
# Check file exists
ls -la production-approach/config/
```

#### Issue 8: "Version number mismatch"

**Problem**: Deployed configuration doesn't match expected version

**Solution**:
1. Run pipeline again - it creates a new version each time
2. Check AppConfig console for version history
3. Verify no manual changes made in console

```bash
# List all configuration versions
aws appconfig list-hosted-configuration-versions \
  --application-id <app-id> \
  --configuration-profile-id <profile-id>
```

### Debug Tips

1. **Enable verbose logging**:
   - Add `set +x` in Jenkins pipeline for detailed output
   - Check CloudWatch Logs for AppConfig events

2. **Verify AWS CLI works**:
   ```bash
   aws appconfig list-applications
   aws cloudformation list-stacks
   ```

3. **Check CloudFormation events**:
   ```bash
   aws cloudformation describe-stack-events \
     --stack-name appconfig-auth-10-cdc1c \
     --max-items 20
   ```

4. **Inspect AppConfig resources**:
   ```bash
   # List applications
   aws appconfig list-applications
   
   # List environments
   aws appconfig list-environments --application-id <app-id>
   
   # List configuration profiles
   aws appconfig list-configuration-profiles --application-id <app-id>
   ```

5. **Compare configurations**:
   ```bash
   # Get current config
   aws appconfig get-hosted-configuration-version \
     --application-id <app-id> \
     --configuration-profile-id <profile-id> \
     --version-number <version> \
     /tmp/config.json
   
   # Compare with Git
   diff /tmp/config.json production-approach/config/cdc1c.json
   ```

## Real-world Usage Scenarios

### Scenario 1: Enable New Feature for Specific Environment

**Situation**: You've developed a new authentication mechanism and want to test it in CDC1C first.

**Steps**:
1. Update feature flag in `config/cdc1c.json`:
   ```json
   {
     "featureFlags": {
       "enableNewAuth": true
     }
   }
   ```

2. Create PR with description:
   ```
   Title: Enable new authentication in CDC1C for testing
   
   Description:
   - Enabling new OAuth2 authentication flow
   - Testing in CDC1C environment only
   - Expected to improve login time by 30%
   - Rollback plan: Set enableNewAuth to false
   ```

3. After approval and merge, run Jenkins pipeline:
   ```
   ACTION: deploy-config
   ASSET_GROUP: cdc1c
   DEPLOYMENT_STRATEGY: Immediate
   ```

4. Monitor application logs and metrics

5. If successful, repeat for other environments (cdc4c, cdc7c, pdc1c)

### Scenario 2: Gradual Rollout to Production

**Situation**: Feature tested successfully in dev/test, now rolling out to production with extra caution.

**Steps**:
1. Update `config/pdc1c.json`:
   ```json
   {
     "featureFlags": {
       "enableNewAuth": true
     }
   }
   ```

2. Create PR with thorough documentation

3. After approval, run Jenkins pipeline with **Canary strategy**:
   ```
   ACTION: deploy-config
   ASSET_GROUP: pdc1c
   DEPLOYMENT_STRATEGY: Canary
   ```

4. **Canary deployment timeline**:
   - **T+0**: Deployment starts to 10% of instances
   - **T+2min**: If no issues, rolls out to 20%
   - **T+4min**: 30%
   - **T+6min**: 40%
   - **T+8min**: 50%
   - **T+10min**: 100%
   - **T+15min**: 5-minute bake time completes

5. Monitor metrics throughout rollout

6. AppConfig automatically rolls back if issues detected

### Scenario 3: Emergency Configuration Change

**Situation**: Production issue requires immediate configuration change (e.g., increase rate limits due to unexpected traffic spike).

**Steps**:
1. Create hotfix branch:
   ```bash
   git checkout -b hotfix/increase-rate-limits
   ```

2. Update `config/pdc1c.json`:
   ```json
   {
     "settings": {
       "maxConcurrentRequests": 1000
     }
   }
   ```

3. Commit and push:
   ```bash
   git add production-approach/config/pdc1c.json
   git commit -m "HOTFIX: Increase concurrent requests to 1000"
   git push
   ```

4. Create PR with "URGENT" label

5. Get expedited review approval

6. Merge to main

7. Run Jenkins pipeline with **Immediate strategy**:
   ```
   ACTION: deploy-config
   ASSET_GROUP: pdc1c
   DEPLOYMENT_STRATEGY: Immediate
   ```

8. Configuration deployed within minutes

9. Create follow-up ticket for root cause analysis

### Scenario 4: Synchronized Multi-Environment Deployment

**Situation**: Security patch requires updating configuration across all environments simultaneously.

**Steps**:
1. Create branch:
   ```bash
   git checkout -b security/update-token-expiry
   ```

2. Update all config files:
   ```bash
   # Update cdc1c.json, cdc4c.json, cdc7c.json, pdc1c.json
   # Change tokenExpiry from 7200 to 3600
   ```

3. Commit all changes:
   ```bash
   git add production-approach/config/*.json
   git commit -m "Security: Reduce token expiry to 1 hour"
   git push
   ```

4. After PR approval and merge, run Jenkins pipeline for each environment:
   ```bash
   # CDC1C
   ACTION: deploy-config, ASSET_GROUP: cdc1c
   
   # CDC4C
   ACTION: deploy-config, ASSET_GROUP: cdc4c
   
   # CDC7C
   ACTION: deploy-config, ASSET_GROUP: cdc7c
   
   # PDC1C
   ACTION: deploy-config, ASSET_GROUP: pdc1c
   ```

5. Monitor all environments for successful deployment

### Scenario 5: A/B Testing Configuration

**Situation**: Testing two different timeout values to optimize performance.

**Steps**:
1. Set different values in test environments:
   - CDC4C: `"apiTimeout": 30`
   - CDC7C: `"apiTimeout": 60`

2. Deploy to both environments

3. Monitor metrics:
   - Response times
   - Error rates
   - Resource utilization

4. After analysis period (e.g., 1 week):
   - Determine optimal value
   - Update all environments with winning configuration

### Scenario 6: Infrastructure Update

**Situation**: Need to modify deployment strategy timings.

**Steps**:
1. Edit `infra/appconfig-service.yml`:
   ```yaml
   DeploymentStrategyCanary:
     Properties:
       DeploymentDurationInMinutes: 20  # Changed from 10
       FinalBakeTimeInMinutes: 10        # Changed from 5
   ```

2. Commit and push changes

3. Run Jenkins pipeline:
   ```
   ACTION: update-infra
   ASSET_GROUP: cdc1c
   ```

4. Verify CloudFormation update

5. Future deployments will use new strategy timings

### Scenario 7: Disaster Recovery - Complete Rollback

**Situation**: New configuration causing widespread issues, need immediate rollback.

**Steps**:
1. **Identify last working commit**:
   ```bash
   git log --oneline production-approach/config/pdc1c.json
   # Find commit hash before problematic change
   ```

2. **Revert configuration**:
   ```bash
   git revert <bad-commit-sha>
   git push
   ```

3. **Emergency deployment**:
   ```
   ACTION: deploy-config
   ASSET_GROUP: pdc1c
   DEPLOYMENT_STRATEGY: Immediate
   ```

4. **Verify rollback**:
   - Check application health
   - Monitor error rates
   - Verify customer impact resolved

5. **Post-mortem**:
   - Document what went wrong
   - Update validation checks
   - Improve testing procedures

---

## Best Practices

1. **Always use Canary for production deployments** unless emergency
2. **Test configuration changes in dev environments first** (cdc1c → cdc4c → cdc7c → pdc1c)
3. **Use descriptive commit messages** that explain the "why" not just the "what"
4. **Include rollback plan in PR descriptions**
5. **Monitor deployments actively** - don't assume they worked
6. **Keep configuration files simple** - complex logic belongs in code
7. **Document feature flags** - include comments or maintain a feature flag registry
8. **Regular cleanup** - remove unused feature flags
9. **Version naming** - use semantic versioning in commit messages
10. **Automated alerts** - set up CloudWatch alarms for AppConfig deployment failures

## Additional Resources

- [AWS AppConfig Documentation](https://docs.aws.amazon.com/appconfig/)
- [CloudFormation AppConfig Resources](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/AWS_AppConfig.html)
- [AWS AppConfig Best Practices](https://docs.aws.amazon.com/appconfig/latest/userguide/appconfig-best-practices.html)
- [Feature Toggles (Martin Fowler)](https://martinfowler.com/articles/feature-toggles.html)

## Support

For issues or questions:
1. Check [Troubleshooting](#troubleshooting) section
2. Review CloudFormation and AppConfig logs
3. Contact DevOps team
4. Create issue in repository

---

**Last Updated**: 2026-02-18  
**Version**: 1.0.0  
**Maintained By**: DevOps Team
