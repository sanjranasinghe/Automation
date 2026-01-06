# Deployment Pipeline Documentation

## Overview

The Deployment Pipeline automates the process of deploying a Docker container to AWS ECS using CloudFormation change sets. It includes an optional vulnerability scan for the Docker image using Trivy before deployment. The pipeline notifies a Slack channel of the deployment progress and the final status, ensuring full visibility and traceability of all deployments.

## Pipeline Description

The Deployment Pipeline consists of the following stages:

- **Deployment-Pipeline-Started**: Notifies the Slack channel when the deployment pipeline starts with user and timestamp information
- **Retrieve ECR URL**: Retrieves the ECR repository URL for the Docker image from CloudFormation stack outputs
- **Build and Push**: Builds the Docker image from the repository, tags it with a timestamp, and pushes it to ECR
- **Trigger Trivy Scan**: Optionally triggers a Trivy vulnerability scan on the Docker image to identify security issues
- **Deploy to ECS**: Creates a CloudFormation change set and executes it to deploy the Docker container to AWS ECS

## Jenkins Pipeline Code (Jenkinsfile)

```groovy
pipeline {
    agent {
        label "master"
    }
    environment {
        SLACK_CHANNEL = '#cmb-prod-deployments'
        ONCALL_ALERT_CHANNEL = '#oncall-devops'
    }
    parameters {
        choice(
            choices: ['Airline-API'],
            description: 'Systems Manager Parameter Name for App Settings',
            name: 'AppSettingsParameterName'
        )
        booleanParam(name: 'VulnerabilityScan',
            defaultValue: true,
            description: 'Scan image for vulnerabilities')
    }

    stages {
        stage('Deployment-Pipeline-Started') {
            steps {
                script {
                    env.LAST_STAGE = 'Deployment-Pipeline-Started'
                    wrap([$class: 'BuildUser']) {
                        def startTime = new Date().format("yyyy-MM-dd HH:mm:ss", TimeZone.getTimeZone('UTC'))
                        env.BUILD_USER = "${BUILD_USER_ID ?: 'unknown'}"
                        slackSend(
                            channel: env.SLACK_CHANNEL,
                            color: '#439FE0',
                            message: ":rocket: *Deployment Pipeline Started*\n" +
                                     "*User:* ${env.BUILD_USER}\n" +
                                     "*Job:* ${env.JOB_NAME}\n" +
                                     "*Build:* ${env.BUILD_NUMBER}\n" +
                                     "*Start Time:* ${startTime}"
                        )
                    }
                }
            }
        }

        stage('Retrieve ECR URL') {
            steps {
                script {
                    env.LAST_STAGE = 'Retrieve ECR URL'
                    withAWS(credentials: '***') {  // Masked AWS credentials
                        def ecrUrl = sh(
                            script: 'aws cloudformation describe-stacks --stack-name Airline-Api-ECR --query "Stacks[0].Outputs[?OutputKey==\'ECRRepositoryURL\'].OutputValue" --output text',
                            returnStdout: true
                        ).trim()
                        env.ECR_URL = ecrUrl
                        sh "echo $ECR_URL"
                    }
                }
            }
        }

        stage('Build and push') {
            agent {
                label 'aws.ec2.***.jenkins.worker'  // Masked worker label
            }
            steps {
                git 'git@bitbucket.org:***/mos_airline_api.git'  // Masked repository URL
                
                sh "rm -rf src/Airline_API/appsettings.json"
                
                withAWS(credentials: '***') {  // Masked AWS credentials
                    sh "aws ssm get-parameter --name ${params.AppSettingsParameterName} --query 'Parameter.Value' --output text > src/Airline_API/appsettings.json"
                
                    script {
                        env.LAST_STAGE = 'Build and push'
                        def timestamp = new Date().format("yyyyMMddHHmmss")
                        
                        sh "docker build -t ${params.EcrRepoName}:${timestamp} --build-arg APP_SETTINGS=appsettings.json ."
                    
                        sh 'aws ecr get-login-password --region eu-west-2 | docker login --username AWS --password-stdin 397753518402.dkr.ecr.eu-west-2.amazonaws.com'
                        sh "docker tag ${params.EcrRepoName}:${timestamp} $ECR_URL:${timestamp}"
                        sh "docker push $ECR_URL:${timestamp}"
                    
                        env.TIMESTAMP = timestamp
                    }
                }
            }     
        }

        stage('Trigger Trivy Scan') {
            steps {
                script {
                    env.LAST_STAGE = 'Trigger Trivy Scan'
                    if(params.VulnerabilityScan){
                    echo "Running Trivy scan..."
                    build wait: true, propagate: true, job: '/Maintenance/trivy-scan', parameters: [
                            string(name: 'MAXVULCOUNT', value: "5"),
                            string(name: 'REGISTRY', value: "397753518402.dkr.ecr.eu-west-2.amazonaws.com"),
                            string(name: 'ECR_URL', value: "${ECR_URL}"),
                            string(name: 'IMAGE_TAG', value: "${timestamp}"),
                            string(name: 'AWS_ACCOUNT_ID', value: '397753518402'),
                            string(name: 'REGION', value: "eu-west-2")
                        ]
                    } else {
                        echo "Image Scan Skipped."
                    }
                }
            }
        } 
        
        stage('Deploy to ECS') {
            steps {
                script {
                    env.LAST_STAGE = 'Deploy to ECS'
                    withAWS(credentials: '***') {  // Masked AWS credentials
                        echo "🛠️ Creating CloudFormation Change Set..."

                        // Create change set
                        sh """
                        aws cloudformation create-change-set \
                            --stack-name Airline-Api-ECS \
                            --change-set-name my-change-set \
                            --template-body file://cloudformation/Deployment-Airline-Api.yml \
                            --parameters ParameterKey=ECRRepository,ParameterValue=$ECR_URL \
                                         ParameterKey=Service,ParameterValue=Airline-Api \
                                         ParameterKey=ECRRepositoryURL,ParameterValue=$ECR_URL \
                                         ParameterKey=Version,ParameterValue=$TIMESTAMP \
                            --tags Key="map-migrated",Value="***" \
                            --capabilities CAPABILITY_NAMED_IAM
				   

                        """
                        // Wait for the change set to be created
                        echo "⏳ Waiting for change set creation..."
                        timeout(time: 5, unit: 'MINUTES') {
                            waitUntil {
                                def status = sh(
                                    script: "aws cloudformation describe-change-set --stack-name Airline-Api-ECS --change-set-name my-change-set --query 'Status' --output text",
                                    returnStdout: true
                                ).trim()

                                if (status == "FAILED") {
                                    def reason = sh(
                                        script: "aws cloudformation describe-change-set --stack-name Airline-Api-ECS --change-set-name my-change-set --query 'StatusReason' --output text",
                                        returnStdout: true
                                    ).trim()
                                    error "❌ Change set creation failed: ${reason}"
                                }

                                return (status == "CREATE_COMPLETE")
                            }
                          }

                        // Execute the change set
                        echo "🚀 Executing change set..."
                        sh "aws cloudformation execute-change-set --stack-name Airline-Api-ECS --change-set-name my-change-set"

                        // Wait for the stack update to complete
                        echo "⏳ Waiting for CloudFormation stack update to complete..."
                        sh "aws cloudformation wait stack-update-complete --stack-name Airline-Api-ECS"

                        echo "✅ Stack update completed successfully!"
                    }
                }
            }
        }
    }
    
    post {
        always {
            agent {
                label "master"
            }
            script {
                wrap([$class: 'BuildUser']) {
                    def status = currentBuild.currentResult
                    def endTime = new Date().format("yyyy-MM-dd HH:mm:ss", TimeZone.getTimeZone('UTC'))

                    def colorMap = [
                        'SUCCESS' : '#36a64f',
                        'FAILURE' : '#FF0000',
                        'ABORTED' : '#FFA500',
                        'UNSTABLE': '#FFFF00'
                    ]
                    def emojiMap = [
                        'SUCCESS' : ':white_check_mark:',
                        'FAILURE' : ':x:',
                        'ABORTED' : ':warning:',
                        'UNSTABLE': ':warning:'
                    ]

                    def color = colorMap.get(status, '#CCCCCC')
                    def emoji = emojiMap.get(status, ':grey_question:')
                    def logSnippet = ''
                    def logScanLines = 200  // increase scan depth for safety

                    if (status != 'SUCCESS') {
                        try {
                            def lines = currentBuild.rawBuild.getLog(logScanLines)
                            def errorIndex = lines.findIndexOf { line ->
                                def lower = line.toLowerCase()
                                return lower.contains("error") ||
                                       lower.contains("fail") ||
                                       lower.contains("exception") ||
                                       lower.contains("not found") ||
                                       lower.contains("exit code") ||
                                       lower.contains("failed") ||
                                       lower.contains("unable to") ||
                                       lower.contains("cannot") ||
                                       lower.contains("timed out") ||
                                       lower.contains("rejected") ||
                                       lower.contains("denied") ||
                                       lower.contains("missing") ||
                                       lower.contains("invalid") ||
                                       lower.contains("fatal") ||
                                       lower.contains("stack trace") ||
                                       lower.contains("traceback") ||
                                       lower.contains("runtimeexception") ||
                                       lower.contains("nullpointerexception") ||
                                       lower.contains("erroraction") ||
                                       lower.contains("groovy.runtime") ||
                                       lower.contains("cannot get property") ||
                                       lower.contains("on null object") 
                                       
                            }

                            if (errorIndex != -1) {
                                def endIndex = Math.min(errorIndex + 20, lines.size()) // error line + 20 more
                                logSnippet = lines.subList(errorIndex, endIndex).join("\n")
                            } else {
                                logSnippet = "No critical error lines found."
                            }
                        } catch (err) {
                            logSnippet = "Log access failed: ${err.message}"
                        }
                    }

                    def baseMessage = "${emoji} *Deployment Pipeline ${status}*\n" +
                                      "*User:* ${env.BUILD_USER}\n" +
                                      "*Job:* ${env.JOB_NAME}\n" +
                                      "*Build:* ${env.BUILD_NUMBER}\n" +
                                      "*Last Stage:* ${env.LAST_STAGE ?: 'Unknown'}\n" +
                                      "*End Time:* ${endTime}"

                    def failureNote = status != 'SUCCESS' ? "\n*Error Snippet (first match + 20 lines):*\n```${logSnippet}```" : ""


                    if (status != 'FAILURE') {
                     slackSend(
                        channel: env.SLACK_CHANNEL,
                        color: color,
                        message: baseMessage + failureNote 
                     )
                    }
                    
                    if (status == 'FAILURE') {
                        slackSend(
                            channel: env.ONCALL_ALERT_CHANNEL,
                            color: color,
                            message: baseMessage + failureNote
                        )
                    }
                }
             }
        }
    }
    
}
```

## Explanation of the Jenkins Pipeline

### Environment Variables

- **SLACK_CHANNEL**: `#cmb-prod-deployments` - Channel for deployment notifications
- **ONCALL_ALERT_CHANNEL**: `#oncall-devops` - Channel for failure alerts to on-call team

### Parameters

- **AppSettingsParameterName**: 
  - Type: Choice
  - Options: `Airline-API`
  - Description: Systems Manager Parameter Name for App Settings
  - Used to retrieve application configuration from AWS SSM
  
- **VulnerabilityScan**: 
  - Type: Boolean
  - Default: `true`
  - Description: Enable/disable Trivy vulnerability scanning
  - When enabled, scans Docker image for security vulnerabilities before deployment

### Pipeline Stages

#### 1. Deployment-Pipeline-Started

Initial notification stage that:
- Captures the current stage name in `LAST_STAGE` environment variable for tracking
- Uses BuildUser plugin to identify who triggered the deployment
- Records the start time in UTC format (`yyyy-MM-dd HH:mm:ss`)
- Sends a Slack notification with:
  - Blue color (`#439FE0`)
  - Rocket emoji (`:rocket:`)
  - User who triggered the build
  - Job name and build number
  - Deployment start timestamp

#### 2. Retrieve ECR URL

Fetches the ECR repository URL:
- Updates `LAST_STAGE` for error tracking
- Authenticates with AWS using masked credentials
- Queries CloudFormation stack `Airline-Api-ECR` for outputs
- Extracts the `ECRRepositoryURL` output value
- Stores the URL in `ECR_URL` environment variable
- Echoes the URL for verification in build logs

#### 3. Build and Push

Builds and pushes the Docker image:
- **Agent**: Runs on a dedicated Jenkins worker node
- **Source Control**: Clones the Git repository from Bitbucket
- **Configuration Management**:
  - Removes existing `appsettings.json` to ensure clean state
  - Retrieves fresh configuration from AWS SSM Parameter Store
- **Docker Build Process**:
  - Generates timestamp in format `yyyyMMddHHmmss`
  - Builds Docker image with timestamp tag
  - Passes application settings as build argument
- **ECR Push Process**:
  - Authenticates with ECR using AWS credentials (region: `eu-west-2`)
  - Tags image with ECR repository URL and timestamp
  - Pushes image to ECR
  - Stores timestamp in `TIMESTAMP` environment variable for deployment

#### 4. Trigger Trivy Scan

Optional security scanning stage:
- Updates `LAST_STAGE` for tracking
- **When VulnerabilityScan = true**:
  - Triggers separate Jenkins job `/Maintenance/trivy-scan`
  - Waits for scan completion (`wait: true`)
  - Propagates scan failure to current build (`propagate: true`)
  - Passes parameters:
    - `MAXVULCOUNT`: Maximum allowed vulnerabilities (`5`)
    - `REGISTRY`: ECR registry URL
    - `ECR_URL`: Full repository URL
    - `IMAGE_TAG`: Timestamp tag of the built image
    - `AWS_ACCOUNT_ID`: AWS account identifier
    - `REGION`: AWS region (`eu-west-2`)
- **When VulnerabilityScan = false**:
  - Logs "Image Scan Skipped" message
  - Continues to deployment

#### 5. Deploy to ECS

CloudFormation change set deployment:
- Updates `LAST_STAGE` for tracking
- **Create Change Set**:
  - Creates a new change set named `my-change-set`
  - Uses template: `cloudformation/Deployment-Airline-Api.yml`
  - Passes parameters:
    - `ECRRepository`: ECR repository URL
    - `Service`: Service name (`Airline-Api`)
    - `ECRRepositoryURL`: Full ECR URL
    - `Version`: Timestamp tag for the image
  - Applies migration tags
  - Requires `CAPABILITY_NAMED_IAM` for IAM resource creation
- **Wait for Change Set Creation**:
  - Timeout: 5 minutes
  - Polls change set status until `CREATE_COMPLETE`
  - If status is `FAILED`, retrieves failure reason and errors out
- **Execute Change Set**:
  - Executes the approved change set
  - Applies changes to the ECS stack
- **Wait for Stack Update**:
  - Uses CloudFormation wait command
  - Blocks until stack update completes successfully
  - Logs success message with checkmark emoji

### Post-Build Actions

The `post` block handles notifications regardless of build outcome:

#### Always Block

Runs after every build (success or failure):

**Status Determination**:
- Captures final build result: `SUCCESS`, `FAILURE`, `ABORTED`, or `UNSTABLE`
- Records end time in UTC format

**Color and Emoji Mapping**:
- `SUCCESS`: Green (`#36a64f`), checkmark (`:white_check_mark:`)
- `FAILURE`: Red (`#FF0000`), X mark (`:x:`)
- `ABORTED`: Orange (`#FFA500`), warning (`:warning:`)
- `UNSTABLE`: Yellow (`#FFFF00`), warning (`:warning:`)
- Default: Grey (`#CCCCCC`), question mark (`:grey_question:`)

**Error Log Extraction** (for non-SUCCESS builds):
- Scans last 200 lines of build log
- Searches for error indicators including:
  - Common error keywords: "error", "fail", "exception", "fatal"
  - Status messages: "not found", "exit code", "timed out"
  - Permission issues: "rejected", "denied", "unauthorized"
  - Missing resources: "missing", "invalid"
  - Runtime errors: "nullpointerexception", "stack trace"
  - Groovy-specific errors: "cannot get property", "on null object"
- Extracts error line plus 20 following lines for context
- Falls back to "No critical error lines found" if no matches

**Notification Logic**:
- **For Non-Failure Status** (SUCCESS, ABORTED, UNSTABLE):
  - Sends notification to `#cmb-prod-deployments` channel
  - Includes base message with status, user, job details
  - Includes error snippet if not SUCCESS
- **For Failure Status**:
  - Sends notification to `#oncall-devops` channel only
  - Alerts on-call team for immediate action
  - Includes base message with error snippet

**Message Format**:
```
[emoji] *Deployment Pipeline [STATUS]*
*User:* [username]
*Job:* [job name]
*Build:* [build number]
*Last Stage:* [stage where failure occurred]
*End Time:* [UTC timestamp]
*Error Snippet (first match + 20 lines):*
```[log excerpt]```
```

## Key Features

### CloudFormation Change Sets

The pipeline uses CloudFormation change sets for safer deployments:
- **Preview Changes**: Change sets show what will be modified before applying
- **Approval Workflow**: Can be reviewed before execution (manual step not shown in current pipeline)
- **Rollback Protection**: Safer than direct stack updates
- **Failure Detection**: Validates changes before applying to production

### Slack Integration

Comprehensive Slack notifications provide:
- **Real-time Updates**: Immediate notification when deployments start
- **User Accountability**: Tracks who triggered each deployment
- **Error Context**: Automatically extracts and shares error logs
- **Alert Routing**: 
  - Normal deployments → `#cmb-prod-deployments`
  - Failures → `#oncall-devops` for immediate response

### Security Scanning

Optional Trivy vulnerability scanning:
- **Configurable**: Can be enabled/disabled per deployment
- **Threshold-based**: Fails deployment if vulnerabilities exceed threshold (5)
- **Blocking**: Prevents vulnerable images from reaching production
- **Detailed Reporting**: Trivy provides CVE details and severity levels

### Error Tracking

Robust error handling:
- **Stage Tracking**: `LAST_STAGE` variable identifies exactly where failures occur
- **Log Analysis**: Automatically extracts relevant error context
- **Comprehensive Keywords**: Scans for 20+ error patterns
- **Contextual Logs**: Provides error line plus 20 surrounding lines

## How to Use the Deployment Pipeline

### Prerequisites

- **AWS Credentials**: Jenkins must have AWS credentials configured with permissions for:
  - CloudFormation (create/update stacks and change sets)
  - ECR (pull/push images)
  - ECS (update services and task definitions)
  - SSM (read parameters)
- **Jenkins Plugins**:
  - AWS Steps Plugin
  - Docker Plugin
  - Slack Notification Plugin
  - Build User Vars Plugin
- **Slack Integration**: 
  - Slack webhook configured in Jenkins
  - Access to `#cmb-prod-deployments` and `#oncall-devops` channels
- **Trivy Scan Job**: Separate Jenkins job at `/Maintenance/trivy-scan` must exist if vulnerability scanning is enabled
- **CloudFormation Templates**:
  - `Deployment-Airline-Api.yml` must exist in repository

### Triggering the Pipeline

1. Navigate to the Jenkins job
2. Click **"Build with Parameters"**
3. Configure parameters:
   - **AppSettingsParameterName**: Select `Airline-API` (or appropriate environment)
   - **VulnerabilityScan**: 
     - Check to enable security scanning (recommended)
     - Uncheck to skip scanning (use only for urgent hotfixes)
4. Click **"Build"**
5. Monitor Slack channel for real-time updates

### Pipeline Execution Flow

**Typical Timeline:**

1. **Start Notification** (< 5 seconds):
   - Slack notification sent immediately
   - User and timestamp recorded

2. **Retrieve ECR URL** (5-10 seconds):
   - Queries CloudFormation for repository URL
   - Validates ECR stack exists

3. **Build and Push** (5-15 minutes):
   - Clones source code
   - Retrieves application configuration
   - Builds Docker image
   - Pushes to ECR

4. **Trivy Scan** (1-3 minutes if enabled):
   - Scans image for vulnerabilities
   - Fails if critical vulnerabilities exceed threshold
   - Generates security report

5. **Deploy to ECS** (3-8 minutes):
   - Creates CloudFormation change set (30-60 seconds)
   - Waits for change set validation (30-120 seconds)
   - Executes change set (1-2 minutes)
   - Waits for stack update completion (2-5 minutes)

6. **Completion Notification** (< 5 seconds):
   - Slack notification with final status
   - Error logs if applicable

**Total Time**: 10-30 minutes depending on image size and complexity

### Verification Steps

#### 1. Verify Slack Notifications

- Check `#cmb-prod-deployments` for:
  - Start notification with your username
  - Completion notification with SUCCESS status
- If failure, check `#oncall-devops` for:
  - Error details
  - Log snippet for debugging

#### 2. Verify ECR Image

Navigate to AWS Console → ECR:
```bash
# Or use AWS CLI
aws ecr describe-images \
  --repository-name airline-api \
  --query 'imageDetails[0]' \
  --output json
```
- Confirm image with timestamp tag exists
- Check image scan results if available

#### 3. Verify ECS Deployment

Navigate to AWS Console → ECS → Clusters:
- Select the appropriate cluster
- Check service `Airline-Api`:
  - **Desired tasks**: Should match expected count
  - **Running tasks**: Should equal desired count
  - **Task definition**: Should show new revision with timestamp tag
  - **Events**: Review for successful deployment events

#### 4. Verify Application Health

Check target group health:
```bash
aws elbv2 describe-target-health \
  --target-group-arn <TARGET_GROUP_ARN>
```
- All targets should show `healthy` status
- If unhealthy, check CloudWatch logs

#### 5. Check CloudFormation Stack

Navigate to AWS Console → CloudFormation:
- Stack `Airline-Api-ECS` should show:
  - Status: `UPDATE_COMPLETE`
  - Last updated: Recent timestamp
  - Change set: `my-change-set` executed
- Review **Events** tab for deployment timeline
- Check **Resources** tab to verify updated task definition

#### 6. Monitor Application Logs

Navigate to CloudWatch Logs:
```bash
# Or use AWS CLI
aws logs tail /ecs/Airline-Api --follow
```
- Verify application started successfully
- Check for any startup errors
- Confirm application is processing requests

#### 7. Test Application Endpoints

Test critical endpoints:
```bash
# Health check
curl https://<load-balancer-url>/dev/health

# Application endpoints
curl https://<load-balancer-url>/api/endpoint
```

## Troubleshooting

### Deployment Failures

#### Change Set Creation Fails

**Symptoms**: Error during "Deploy to ECS" stage, change set status shows FAILED

**Common Causes**:
- Invalid CloudFormation template syntax
- Missing or incorrect parameters
- IAM permission issues
- Resource conflicts or dependencies

**Solutions**:
- Review CloudFormation error message in status reason
- Validate template syntax locally:
  ```bash
  aws cloudformation validate-template \
    --template-body file://cloudformation/Deployment-Airline-Api.yml
  ```
- Check IAM permissions for CloudFormation service role
- Verify all referenced resources exist (VPCs, subnets, security groups, target groups)

#### Stack Update Timeout

**Symptoms**: Pipeline hangs at "Waiting for CloudFormation stack update to complete"

**Common Causes**:
- ECS tasks failing health checks
- Task definition issues (memory, CPU, container configuration)
- Network connectivity problems
- Application startup failures

**Solutions**:
- Check ECS service events in AWS Console
- Review CloudWatch logs for application errors
- Verify target group health checks are passing
- Check security group rules allow traffic
- Ensure task has sufficient memory and CPU

#### Change Set Shows No Changes

**Symptoms**: Change set created but shows no modifications

**Common Causes**:
- Image tag hasn't changed (timestamp issue)
- Template parameters identical to current stack
- CloudFormation doesn't detect changes

**Solutions**:
- Verify `TIMESTAMP` environment variable is set correctly
- Check ECR image tag matches the timestamp
- Consider deleting and recreating the change set
- Review CloudFormation template for drift

### Build Failures

#### Docker Build Fails

**Symptoms**: Error during "Build and push" stage

**Common Causes**:
- Dockerfile syntax errors
- Missing dependencies or build context files
- Network issues downloading base images or packages
- Insufficient disk space on Jenkins agent

**Solutions**:
- Test Dockerfile locally
- Review build logs for specific error messages
- Check network connectivity from Jenkins agent
- Verify Jenkins agent has sufficient disk space:
  ```bash
  df -h
  docker system df
  ```
- Clean up old Docker images:
  ```bash
  docker system prune -a
  ```

#### ECR Push Fails

**Symptoms**: Image builds successfully but fails to push to ECR

**Common Causes**:
- ECR authentication expired
- Network issues
- ECR repository doesn't exist
- IAM permissions insufficient

**Solutions**:
- Verify ECR login:
  ```bash
  aws ecr get-login-password --region eu-west-2 | \
    docker login --username AWS --password-stdin \
    397753518402.dkr.ecr.eu-west-2.amazonaws.com
  ```
- Check ECR repository exists:
  ```bash
  aws ecr describe-repositories --repository-names airline-api
  ```
- Verify IAM permissions include `ecr:PutImage`, `ecr:InitiateLayerUpload`, `ecr:CompleteLayerUpload`

#### SSM Parameter Retrieval Fails

**Symptoms**: Error when retrieving appsettings.json from SSM

**Common Causes**:
- Parameter doesn't exist in SSM
- IAM permissions insufficient
- Wrong parameter name
- Wrong AWS region

**Solutions**:
- Verify parameter exists:
  ```bash
  aws ssm get-parameter --name Airline-API --query 'Parameter.Value'
  ```
- Check IAM permissions include `ssm:GetParameter`
- Ensure parameter name matches exactly (case-sensitive)
- Verify region configuration

### Trivy Scan Issues

#### Scan Fails with High Vulnerability Count

**Symptoms**: Build fails during "Trigger Trivy Scan" stage

**Common Causes**:
- Image contains critical vulnerabilities exceeding threshold (5)
- Outdated base images
- Vulnerable dependencies

**Solutions**:
- Review Trivy scan report for specific CVEs
- Update base Docker image to latest stable version
- Update application dependencies
- Temporarily disable scan for urgent deployments (use with caution):
  - Uncheck `VulnerabilityScan` parameter
  - Create ticket to address vulnerabilities

#### Trivy Job Not Found

**Symptoms**: Error: "No item named '/Maintenance/trivy-scan' exists"

**Common Causes**:
- Trivy scan job doesn't exist
- Incorrect job path
- Missing permissions to trigger job

**Solutions**:
- Verify Trivy job exists at `/Maintenance/trivy-scan`
- Check Jenkins job path is correct
- Ensure user has permission to trigger the Trivy job
- Temporarily disable scan if job is unavailable

### Slack Notification Issues

#### No Slack Notifications

**Symptoms**: Pipeline runs but no Slack messages appear

**Common Causes**:
- Slack webhook not configured
- Channel names incorrect
- Slack plugin not installed
- Network connectivity issues

**Solutions**:
- Verify Slack plugin is installed in Jenkins
- Check Slack webhook configuration in Jenkins global settings
- Verify channel names match exactly (including #)
- Test Slack integration:
  ```groovy
  slackSend channel: '#cmb-prod-deployments', message: 'Test message'
  ```

#### Error Logs Not Appearing in Slack

**Symptoms**: Failure notification sent but no error snippet

**Common Causes**:
- Log parsing failed
- No matching error patterns found
- Message too long for Slack

**Solutions**:
- Check Jenkins console output manually
- Review error pattern keywords in post block
- Consider adjusting `logScanLines` value (currently 200)

### Permission Issues

#### IAM Permission Denied

**Symptoms**: AWS CLI commands fail with "Access Denied" or "UnauthorizedOperation"

**Common Causes**:
- IAM role/user lacks required permissions
- Wrong AWS credentials configured
- Cross-account access not configured

**Solutions**:
- Review IAM policy attached to Jenkins credentials
- Required permissions include:
  - CloudFormation: `CreateStack`, `UpdateStack`, `DescribeStacks`, `CreateChangeSet`, `ExecuteChangeSet`, `DescribeChangeSet`
  - ECR: `GetAuthorizationToken`, `BatchCheckLayerAvailability`, `PutImage`, `InitiateLayerUpload`, `CompleteLayerUpload`
  - ECS: `UpdateService`, `DescribeServices`, `DescribeTaskDefinition`
  - SSM: `GetParameter`
  - IAM: `PassRole` (for ECS task execution role)
- Test AWS credentials:
  ```bash
  aws sts get-caller-identity
  ```

## Best Practices

### Change Set Management

- **Review Before Execute**: For production deployments, consider adding a manual approval step before executing change sets
- **Naming Convention**: Use meaningful change set names (current: `my-change-set` - consider adding timestamp or ticket number)
- **Cleanup**: Delete old change sets after successful deployment to avoid clutter

### Versioning Strategy

- **Timestamp Tags**: Current implementation uses `yyyyMMddHHmmss` format
  - Provides unique, sortable version identifiers
  - Easy to correlate with deployment times
- **Semantic Versioning**: Consider adding semantic version tags (v1.0.0) alongside timestamps for releases
- **Tag Immutability**: Never reuse or overwrite tags - each deployment creates a new tag
- **Retention Policy**: Implement ECR lifecycle policies to clean up old images while retaining critical versions

### Security Best Practices

- **Vulnerability Scanning**: Always enable Trivy scanning for production deployments
- **Credential Management**: 
  - Never hardcode credentials in Jenkinsfile
  - Use Jenkins credential store
  - Rotate AWS credentials regularly
- **Least Privilege**: Grant minimum required IAM permissions
- **Secrets Management**: Store sensitive configuration in AWS SSM Parameter Store or Secrets Manager, never in Git
- **Image Scanning**: Review Trivy reports and address critical vulnerabilities promptly

### Monitoring and Observability

- **CloudWatch Logs**: Monitor `/ecs/Airline-Api` log group for application logs
- **ECS Service Metrics**: Track:
  - CPU and memory utilization
  - Task count and health
  - Deployment success rate
- **Application Health**: Configure meaningful health checks at `/dev/health`
- **Slack Alerts**: Ensure on-call team monitors `#oncall-devops` channel
- **CloudFormation Events**: Review stack events for deployment timeline and issues

### Rollback Strategy

If a deployment fails or causes issues:

1. **Immediate Rollback via Console**:
   ```bash
   aws ecs update-service \
     --cluster <cluster-name> \
     --service Airline-Api \
     --task-definition <previous-task-definition-arn>
   ```

2. **Rollback via Pipeline**:
   - Trigger new deployment with previous working timestamp
   - Ensure previous image still exists in ECR

3. **CloudFormation Rollback**:
   - CloudFormation automatically rolls back on failure if configured
   - Manual rollback via AWS Console if needed

### Deployment Windows

- **Schedule**: Deploy during low-traffic periods when possible
- **Communication**: Notify stakeholders via Slack before major deployments
- **Monitoring**: Watch metrics closely for 15-30 minutes post-deployment
- **Rollback Plan**: Have rollback procedure ready before deploying

## Common Deployment Scenarios

### Standard Production Deployment

1. Trigger pipeline with vulnerability scanning enabled
2. Monitor Slack notifications
3. Verify successful deployment in AWS Console
4. Check application health and logs
5. Monitor metrics for 30 minutes

### Emergency Hotfix Deployment

1. Disable vulnerability scanning for speed (if time-critical)
2. Deploy fix immediately
3. Create follow-up ticket to run security scan
4. Document decision in deployment notes

### Blue-Green Deployment

Current pipeline supports rolling updates. For true blue-green:
- Modify CloudFormation template to support two ECS services
- Update target group routing
- Switch traffic after validation

### Rollback Deployment

1. Identify last known good timestamp/version
2. Trigger pipeline with that specific version
3. OR use AWS Console to revert to previous task definition
4. Verify rollback successful

## Pipeline Maintenance

### Regular Tasks

- **Weekly**: Review failed deployments and error patterns
- **Monthly**: 
  - Update base Docker images
  - Review and update IAM permissions
  - Clean up old ECR images
  - Review Trivy scan reports
- **Quarterly**: 
  - Update Jenkins plugins
  - Review and optimize pipeline performance
  - Update documentation

### Improvements to Consider

- **Approval Gates**: Add manual approval before executing change sets in production
- **Automated Testing**: Integrate smoke tests after deployment
- **Parallel Deployments**: Support multiple environments (dev, staging, prod)
- **Metrics Collection**: Track deployment success rate and duration
- **Enhanced Notifications**: Include deployment duration and resource changes in Slack messages

## Conclusion

The Deployment Pipeline provides a robust, automated solution for deploying Docker containers to AWS ECS with built-in security scanning, comprehensive monitoring, and intelligent error handling. By leveraging CloudFormation change sets, the pipeline ensures safe, reviewable deployments while Slack integration provides real-time visibility to all stakeholders. The optional Trivy scanning adds an essential security layer, preventing vulnerable images from reaching production.
