# Infrastructure Pipeline Documentation

## Overview

The Infrastructure Pipeline is responsible for provisioning the necessary AWS Target Group for the microservice using AWS CloudFormation. This pipeline ensures that the target group is created and ready to be used by the application for routing traffic.

This pipeline is implemented in Jenkins and leverages the AWS CLI to interact with AWS CloudFormation, providing a fully automated infrastructure deployment process.

## Pipeline Description

The Infrastructure Pipeline consists of the following steps:

- **Target Group Creation**: Creates a new AWS Elastic Load Balancer Target Group using AWS CloudFormation
- **CloudFormation Stack**: The pipeline uses a CloudFormation template to provision the required target group and configure necessary properties such as health checks and load balancing algorithms
- **Dynamic Configuration**: The pipeline is triggered with a parameter for `targetgroup`, which specifies the name of the target group dynamically

## Jenkins Pipeline Code (Jenkinsfile)

```groovy
pipeline {
    agent {
        label 'master'
    }
    parameters {
        string(description: 'Export variable', name: 'targetgroup')
    }

    stages {
        stage('Create Target Group') {
            steps {
                withAWS(credentials: 'MOS') {
                    sh "aws cloudformation create-stack --stack-name Airline-Api-Target-group --template-body file://cloudformation/target-group-Airline-Api.yml --parameters ParameterKey=Targetgroup,ParameterValue=${params.targetgroup} --capabilities CAPABILITY_NAMED_IAM"
                }
            }
        }
    }
}
```

### Explanation of the Jenkins Pipeline

- **Agent**: The pipeline runs on a master node, indicated by `label 'master'`

- **Parameters**:
  - `targetgroup`: A string parameter that allows you to dynamically specify the name of the target group when triggering the pipeline

- **Stages**:
  - **Create Target Group**:
    - The pipeline uses the `withAWS` Jenkins step to authenticate with AWS using the `MOS` credentials
    - The `aws cloudformation create-stack` command is executed, which creates the target group in AWS using the CloudFormation template (`target-group-Airline-Api.yml`)
    - The `Targetgroup` parameter is passed from the Jenkins UI, allowing you to specify the name of the target group when running the pipeline

## CloudFormation Template (target-group-Airline-Api.yml)

This CloudFormation template defines the Target Group in AWS that is created during the pipeline execution.

```yaml
Parameters:
  Targetgroup:
    Type: String
    Description: Name of the Target group

Resources:
  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: TG-AirLine-Api
      Port: 8702
      Protocol: HTTP
      VpcId: vpc-0c32ed966ff96b4b4  # Replace with the actual VPC ID
      TargetType: ip
      HealthCheckPath: "/dev/health"
      HealthCheckIntervalSeconds: 10
      HealthCheckTimeoutSeconds: 9
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 2
      TargetGroupAttributes:
        - Key: load_balancing.algorithm.type
          Value: "least_outstanding_requests"
      LoadBalancerArns:
        - arn:aws:elasticloadbalancing:eu-west-2:397753518402:loadbalancer/app/test/bca994e6977ad718  # Replace with actual Load Balancer ARN

Outputs:
  target:
    Value: !Ref TargetGroup
    Export:
      Name: !Ref Targetgroup
```

### Explanation of the CloudFormation Template

**Parameters:**
- `Targetgroup`: A string parameter used to define the name of the target group dynamically during stack creation

**Resources:**

The `TargetGroup` resource has the following key properties:
- **Type**: `AWS::ElasticLoadBalancingV2::TargetGroup` - Defines an AWS Elastic Load Balancer target group
- **Name**: `TG-AirLine-Api` - The name of the target group
- **Port**: `8702` - The port on which the target group listens
- **Protocol**: `HTTP` - The protocol used for routing traffic
- **VpcId**: `vpc-0c32ed966ff96b4b4` - The VPC ID (must be replaced with actual VPC ID)
- **TargetType**: `ip` - Specifies that targets are registered by IP address
- **HealthCheckPath**: `/dev/health` - The endpoint used for health checks
- **HealthCheckIntervalSeconds**: `10` - Time between health checks
- **HealthCheckTimeoutSeconds**: `9` - Timeout for each health check
- **HealthyThresholdCount**: `2` - Number of consecutive successful health checks required
- **UnhealthyThresholdCount**: `2` - Number of consecutive failed health checks before marking unhealthy
- **TargetGroupAttributes**:
  - Load balancing algorithm: `least_outstanding_requests`
- **LoadBalancerArns**: The ARN of the load balancer (must be replaced with actual ARN)

**Outputs:**
- **target**: Exports the `TargetGroup` resource so that it can be used by other stacks or services in the AWS environment
- **Export Name**: Tied to the value of the `Targetgroup` parameter

## How to Use the Pipeline

### Prerequisites

- **AWS Credentials**: Ensure that your Jenkins instance has AWS credentials (referred to as `MOS` in the pipeline) configured with appropriate permissions to create CloudFormation stacks and manage AWS resources like load balancers and target groups
- **Jenkins Setup**: Ensure that Jenkins is configured with the necessary plugins (e.g., AWS, CloudFormation, etc.) to interact with AWS services

### Triggering the Pipeline

When triggering the pipeline, you need to provide a value for the `targetgroup` parameter. For example:
- `Airline-Api-TG`

This value will be passed into the CloudFormation template to dynamically set the target group name.

### Pipeline Execution

- The pipeline will automatically execute and create a CloudFormation stack with the target group specified in the `targetgroup` parameter
- The AWS CLI will interact with CloudFormation to provision the target group in your AWS environment

### Verification

After the pipeline finishes, you can verify the creation of the target group in the AWS Management Console:

1. Go to **EC2 > Target Groups**
2. Ensure that the target group is created successfully and is associated with the correct load balancer and VPC

## Troubleshooting

- **Invalid VPC ID**: If the VPC ID is incorrect or doesn't exist in your AWS account, the CloudFormation stack will fail. Verify that the VPC ID in the CloudFormation template is correct

- **IAM Permissions**: Ensure that the AWS credentials (`MOS`) have the appropriate IAM permissions for creating and managing CloudFormation stacks and resources like target groups and load balancers

- **Load Balancer ARN**: If the Load Balancer ARN is incorrect or inaccessible, the target group creation may fail. Verify that the correct ARN is provided for the load balancer

## Conclusion

This Infrastructure Pipeline automates the creation of an AWS Target Group using CloudFormation. It simplifies the infrastructure setup for microservices and ensures that the target group is provisioned with the correct configurations, such as health checks and load balancing. By using this pipeline, you can easily provision AWS resources in a consistent and automated manner.
