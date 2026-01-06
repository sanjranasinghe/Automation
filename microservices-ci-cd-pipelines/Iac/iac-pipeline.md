# IaC Pipeline Documentation

## Overview

The Infrastructure as Code (IaC) Pipeline automates the process of deploying microservices to AWS ECS by building and pushing Docker images to ECR, then deploying those images as tasks on ECS. It uses AWS CloudFormation, SSM for configuration management, and Docker for containerized application builds.

This pipeline is implemented in Jenkins and is structured in several stages, each of which serves a different purpose for building, pushing, and deploying the microservices.

## Pipeline Description

The IaC Pipeline consists of the following stages:

- **Create ECR**: Creates an Amazon Elastic Container Registry (ECR) repository where the Docker image will be stored
- **Retrieve ECR URL**: Retrieves the URL of the newly created ECR repository for use in subsequent stages
- **Build and Push Docker Image**: Builds a Docker image from the source code and pushes it to the ECR repository
- **Deploy to ECS**: Deploys the Docker image to Amazon ECS (Elastic Container Service) and associates it with an ECS service

## Jenkins Pipeline Code (Jenkinsfile)

```groovy
pipeline {
    agent {
        label "master"
    }

    parameters {
        choice(
            choices: ['airline-api'],
            description: 'Ecr repo name',
            name: 'EcrRepoName'
        )
        
        choice(
            choices: ['Airline-API'],
            description: 'Systems Manager Parameter Name for App Settings',
            name: 'AppSettingsParameterName'
        )
    }

    stages {
        stage('Create Ecr') {
            steps {
                withAWS(credentials: '***') {  // Use the appropriate AWS credentials
                    sh "aws cloudformation create-stack --stack-name Airline-Api-ECR --template-body file://cloudformation/ecr.yml --parameters ParameterKey=EcrRepositoryName,ParameterValue=${params.EcrRepoName} --tags Key=\"map-migrated\",Value=\"***\" --capabilities CAPABILITY_NAMED_IAM"
                    sh "aws cloudformation wait stack-create-complete --stack-name Airline-Api-ECR"
                }
            }
        }

        stage('Retrieve ECR URL') {
            steps {
                script {
                    def ecrUrl = sh(
                        script: 'AWS_PROFILE=*** aws cloudformation describe-stacks --stack-name Airline-Api-ECR --query "Stacks[0].Outputs[?OutputKey==\'ECRRepositoryURL\'].OutputValue" --output text',
                        returnStdout: true
                    ).trim()
                    env.ECR_URL = ecrUrl
                }
            }
        }

        stage('Build and push') {
            agent {
                label 'aws.ec2.***.jenkins.worker'  // Masking worker label info for security
            }
            steps {
                git 'git@bitbucket.org:***/mos_airline_api.git'  // Masked repo info for security
                
                sh "rm -rf src/Airline_API/appsettings.json"
                
                withAWS(credentials: '***') {  // Masked credentials for security
                    sh "aws ssm get-parameter --name ${params.AppSettingsParameterName} --query 'Parameter.Value' --output text > src/Airline_API/appsettings.json"
                
                    script {
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

        stage('Deploy to ECS') {
            steps {
                withAWS(credentials: '***') {  // Masked credentials for security
                    sh "aws cloudformation create-stack --stack-name Airline-Api-ECS --template-body file://cloudformation/iac_Airline-Api.yml --parameters ParameterKey=ECRRepository,ParameterValue=$ECR_URL ParameterKey=Service,ParameterValue=Airline-Api ParameterKey=ECRRepositoryURL,ParameterValue=$ECR_URL ParameterKey=Version,ParameterValue=$TIMESTAMP --tags Key=\"map-migrated\",Value=\"***\" --capabilities CAPABILITY_NAMED_IAM"
                }
            }
        }
    }
}
```

## Explanation of the Jenkins Pipeline

### Parameters

- **EcrRepoName**: Specifies the name of the ECR repository where the Docker image will be pushed
  - Options: `airline-api`
  
- **AppSettingsParameterName**: The name of the Systems Manager (SSM) parameter that stores application settings, which are fetched during the build process
  - Options: `Airline-API`

### Stages

#### 1. Create ECR

- The `aws cloudformation create-stack` command creates an ECR repository using a CloudFormation template (`ecr.yml`)
- The `aws cloudformation wait stack-create-complete` command ensures that the ECR repository is created before continuing
- Tags are applied for migration tracking

#### 2. Retrieve ECR URL

- The pipeline fetches the ECR repository URL using the `describe-stacks` command
- Stores the URL in the `ECR_URL` environment variable for use in later stages
- Uses AWS profile for authentication

#### 3. Build and Push Docker Image

This stage performs several critical operations:

- **Git Checkout**: Clones the source code from the Bitbucket repository
- **Configuration Management**: 
  - Removes existing `appsettings.json`
  - Retrieves fresh configuration from AWS SSM Parameter Store
- **Docker Build**:
  - Builds a Docker image tagged with a timestamp (format: `yyyyMMddHHmmss`)
  - Uses build argument for application settings
- **ECR Authentication**: Logs in to AWS ECR using AWS credentials
- **Image Tagging**: Tags the Docker image with the ECR repository URL and timestamp
- **Image Push**: Pushes the Docker image to the ECR repository
- **Environment Variable**: Stores the timestamp in `TIMESTAMP` environment variable

#### 4. Deploy to ECS

- Uses the `aws cloudformation create-stack` command to deploy the microservice to ECS
- Passes the following parameters:
  - `ECRRepository`: The ECR repository name
  - `Service`: Service name (Airline-Api)
  - `ECRRepositoryURL`: The full ECR repository URL
  - `Version`: The timestamp version tag
- The ECS service is defined in the CloudFormation template (`iac_Airline-Api.yml`)

## CloudFormation Template (iac_Airline-Api.yml)

This CloudFormation template defines the ECS task definition, service, and autoscaling configuration for the Airline API service.

```yaml
AWSTemplateFormatVersion: '2010-09-09'

Parameters:
  ECRRepository:
    Type: String
    Description: Name of the ECR

  ECRRepositoryURL:
    Type: String
    Description: URL of the ECR repository

  Version:
    Type: String
    Description: Repo Version

  Service:
    Type: String
    Description: Name of the Service

Resources:

  ECSLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: /ecs/Airline-Api
      RetentionInDays: 30

  TaskDefinition:
    Type: AWS::ECS::TaskDefinition
    UpdateReplacePolicy: Retain
    DeletionPolicy: Delete
    Properties:
      Family: Airline-Api
      RequiresCompatibilities:
        - FARGATE
      Memory: '512'
      Cpu: '256'
      NetworkMode: awsvpc
      ExecutionRoleArn: arn:aws:iam::397753518402:role/***  # Masked IAM role
      RuntimePlatform:
        CpuArchitecture: ARM64
        OperatingSystemFamily: LINUX
      ContainerDefinitions:
        - Name: Airline-Api
          Image: !Join
            - ''
            - - !Ref ECRRepositoryURL
              - ':'
              - !Ref Version
          Memory: 512
          PortMappings:
            - ContainerPort: 80
          LogConfiguration:
            LogDriver: awslogs
            Options:
              awslogs-create-group: 'true'
              awslogs-group: !Ref ECSLogGroup
              awslogs-region: !Ref 'AWS::Region'
              awslogs-stream-prefix: ecs
          Environment:
            - Name: TZ
              Value: Europe/London
            - Name: ENV
              Value: PROD

  ECSService:
    Type: AWS::ECS::Service
    Properties:
      Cluster: !ImportValue ECSClusterNameMOS  # Masked cluster name
      ServiceName: Airline-Api
      TaskDefinition: !Ref TaskDefinition
      NetworkConfiguration:
        AwsvpcConfiguration:
          AssignPublicIp: DISABLED
          SecurityGroups:
            - ***  # Masked Security Group ID
          Subnets:
            - ***  # Masked Subnet ID
            - ***  # Masked Subnet ID
      LoadBalancers:
        - ContainerName: Airline-Api
          ContainerPort: 80
          TargetGroupArn: !ImportValue TG-Aireline-API  # Masked Target Group
        - ContainerName: Airline-Api
          ContainerPort: 80
          TargetGroupArn: !ImportValue TG-AirLine-Api-Shared  # Masked Target Group
      DeploymentConfiguration:
        MaximumPercent: 200
        MinimumHealthyPercent: 100

  ScalingPolicyCPU:
    Type: AWS::ApplicationAutoScaling::ScalingPolicy
    Properties:
      PolicyName: ECSServiceScalingPolicyCPU
      PolicyType: TargetTrackingScaling
      ScalingTargetId: !Ref ECSAutoscalingTarget
      TargetTrackingScalingPolicyConfiguration:
        PredefinedMetricSpecification:
          PredefinedMetricType: ECSServiceAverageCPUUtilization
        ScaleInCooldown: 300
        ScaleOutCooldown: 10
        TargetValue: 60

  ECSAutoscalingTarget:
    Type: AWS::ApplicationAutoScaling::ScalableTarget
    Properties:
      MaxCapacity: 20
      MinCapacity: 2
      ResourceId: !Sub
        - 'service/${clusterName}/${serviceName}'
        - clusterName: !ImportValue ECSClusterNameMOS  # Masked cluster name
          serviceName: !GetAtt ECSService.Name
      RoleARN: arn:aws:iam::397753518402:role/ecsAutoscaleRole  # Masked IAM role
      ScalableDimension: ecs:service:DesiredCount
      ServiceNamespace: ecs
```

## CloudFormation Template Components

### Parameters

- **ECRRepository**: Name of the ECR repository
- **ECRRepositoryURL**: Full URL of the ECR repository
- **Version**: Version tag for the Docker image (timestamp)
- **Service**: Name of the service being deployed

### Resources

#### ECSLogGroup

- **Type**: `AWS::Logs::LogGroup`
- **LogGroupName**: `/ecs/Airline-Api`
- **RetentionInDays**: 30 days

#### TaskDefinition

Key properties:
- **Family**: `Airline-Api`
- **Launch Type**: FARGATE
- **Memory**: 512 MB
- **CPU**: 256 units (0.25 vCPU)
- **Network Mode**: `awsvpc`
- **Architecture**: ARM64
- **Operating System**: Linux
- **Container Configuration**:
  - Container Name: `Airline-Api`
  - Image: Dynamically constructed from ECR URL and version
  - Memory: 512 MB
  - Port: 80
  - Log Driver: CloudWatch Logs (`awslogs`)
  - Environment Variables:
    - `TZ`: Europe/London
    - `ENV`: PROD

#### ECSService

Key properties:
- **Service Name**: `Airline-Api`
- **Cluster**: Imported from CloudFormation export
- **Task Definition**: References the TaskDefinition resource
- **Network Configuration**:
  - Public IP: Disabled
  - Security Groups: Configured for service access
  - Subnets: Multi-AZ deployment
- **Load Balancers**: 
  - Two target groups configured for traffic distribution
  - Container port 80 mapped to both target groups
- **Deployment Configuration**:
  - Maximum Percent: 200% (allows rolling updates)
  - Minimum Healthy Percent: 100% (ensures zero downtime)

#### ScalingPolicyCPU

Key properties:
- **Policy Type**: Target Tracking Scaling
- **Metric**: ECS Service Average CPU Utilization
- **Target Value**: 60%
- **Scale In Cooldown**: 300 seconds (5 minutes)
- **Scale Out Cooldown**: 10 seconds

#### ECSAutoscalingTarget

Key properties:
- **Maximum Capacity**: 20 tasks
- **Minimum Capacity**: 2 tasks
- **Scalable Dimension**: Desired task count
- **Service Namespace**: ECS

## How to Use the IaC Pipeline

### Prerequisites

- **AWS Credentials**: Ensure the Jenkins instance has AWS credentials configured for accessing services like ECS, ECR, and CloudFormation
- **Docker & ECR**: 
  - Docker must be installed on the Jenkins agent
  - ECR repository must be correctly set up
- **SSM Parameters**: Application settings should be stored in AWS Systems Manager Parameter Store
- **Jenkins Plugins**: Required plugins include:
  - AWS Steps Plugin
  - Docker Plugin
  - Git Plugin
- **Network Access**: Jenkins agent must have access to:
  - AWS services (ECR, ECS, CloudFormation, SSM)
  - Bitbucket repository
  - Docker registry

### Triggering the Pipeline

1. Navigate to the Jenkins job
2. Click "Build with Parameters"
3. Select the appropriate values:
   - **EcrRepoName**: Choose `airline-api`
   - **AppSettingsParameterName**: Choose `Airline-API`
4. Click "Build"

### Pipeline Execution Flow

1. **ECR Creation** (1-2 minutes):
   - Creates CloudFormation stack for ECR repository
   - Waits for stack creation to complete

2. **ECR URL Retrieval** (< 1 minute):
   - Queries CloudFormation stack outputs
   - Stores ECR URL for subsequent stages

3. **Build and Push** (5-15 minutes depending on image size):
   - Clones source code
   - Retrieves application configuration
   - Builds Docker image
   - Authenticates with ECR
   - Pushes image to repository

4. **ECS Deployment** (2-5 minutes):
   - Creates CloudFormation stack for ECS service
   - Provisions task definition and service
   - Configures autoscaling policies

### Verification

After the pipeline completes successfully:

1. **Verify ECR Repository**:
   - Go to AWS Console → ECR
   - Confirm the repository exists and contains the new image with timestamp tag

2. **Verify ECS Service**:
   - Go to AWS Console → ECS → Clusters
   - Check that the service is running with desired task count
   - Verify tasks are in RUNNING state

3. **Verify Application Health**:
   - Check target group health in EC2 Load Balancer console
   - Verify application is responding to health checks
   - Test application endpoints

4. **Check CloudWatch Logs**:
   - Navigate to CloudWatch Logs
   - Check `/ecs/Airline-Api` log group for application logs

## Troubleshooting

### ECR Issues

- **Stack Creation Fails**: 
  - Verify ECR repository name doesn't already exist
  - Check IAM permissions for CloudFormation and ECR operations
  
- **Image Push Fails**:
  - Verify ECR authentication is successful
  - Check network connectivity to ECR
  - Ensure sufficient disk space on Jenkins agent

### Build Issues

- **Docker Build Fails**:
  - Check Dockerfile syntax and build context
  - Verify all dependencies are accessible
  - Review build logs for specific errors

- **SSM Parameter Retrieval Fails**:
  - Verify parameter name exists in SSM Parameter Store
  - Check IAM permissions for SSM access
  - Ensure parameter is in the correct region

### ECS Deployment Issues

- **Service Won't Start**:
  - Check task definition configuration
  - Verify IAM execution role has required permissions
  - Review CloudWatch logs for container startup errors

- **Tasks Keep Stopping**:
  - Check application logs in CloudWatch
  - Verify health check configuration
  - Ensure adequate CPU and memory allocation

- **Load Balancer Health Checks Fail**:
  - Verify target group configuration
  - Check security group rules allow traffic
  - Confirm application is listening on correct port

### Autoscaling Issues

- **Service Not Scaling**:
  - Verify autoscaling target and policy are created
  - Check CloudWatch metrics for CPU utilization
  - Ensure IAM role has autoscaling permissions

## Best Practices

- **Versioning**: The pipeline automatically tags images with timestamps for traceability
- **Zero Downtime**: Deployment configuration ensures 100% minimum healthy tasks
- **Monitoring**: CloudWatch Logs retention is set to 30 days
- **Security**: Use IAM roles and policies following least privilege principle
- **Resource Limits**: Configure appropriate CPU and memory based on application requirements
- **Autoscaling**: Tune scaling policies based on actual application load patterns

## Conclusion

This IaC Pipeline automates the complete deployment lifecycle from source code to running ECS service. It uses CloudFormation templates to define infrastructure, AWS SSM for configuration management, and Docker for containerized deployments. The pipeline ensures consistent, repeatable deployments with built-in scaling and monitoring capabilities.
