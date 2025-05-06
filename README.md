# EBS Auto-Resize Automation

This CloudFormation stack deploys an automated solution for monitoring and resizing EBS volumes based on usage thresholds. The system uses tag-based configuration to enable auto-resize on selected volumes and automatically extends both the volume and filesystem when usage exceeds configured thresholds.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Step Functions Workflow](#step-functions-workflow)
- [Tag-Based Configuration](#tag-based-configuration)
- [CloudWatch Agent Metrics](#cloudwatch-agent-metrics)
- [Installation](#installation)
- [Maintenance and Troubleshooting](#maintenance-and-troubleshooting)
- [Limitations and Considerations](#limitations-and-considerations)
- [Recent Updates](#recent-updates)

## Overview

The EBS Auto-Resize Automation provides the following capabilities:

- Monitors EBS volumes with `EBSAutoResize` tag set to "true"
- Triggers automatic resize when volume usage exceeds 90%
- Resizes volumes by configurable increments (defined in tags)
- Respects maximum size limits (defined in tags)
- Automatically extends the filesystem after resize
- Supports both Linux and Windows instances
- Maintains a resize history in volume tags

The solution uses CloudWatch alarms for monitoring, EventBridge rules for event routing, and Step Functions for orchestrating the resize workflow.

## Architecture

The solution consists of the following components:

```mermaid
flowchart TD
    %% Define components
    CloudWatch[CloudWatch Alarms] --> EventBridge[EventBridge Rules]
    EventBridge --> StepFunction[EBS Resize Step Functions]
    StepFunction --> EvaluateVolume[Evaluate Volume Tags\nLambda]
    EvaluateVolume --> ResizeVolume[Resize Volume\nLambda]
    ResizeVolume --> CheckOS[Check OS Type\nLambda]
    CheckOS --> |Linux| LinuxExtend[Extend Linux Filesystem\nLambda with SSM]
    CheckOS --> |Windows| WindowsExtend[Extend Windows Filesystem\nLambda with SSM]
    
    %% Define tagging system
    Tags[Volume Tags] -.- CloudWatch
    Tags -.- EvaluateVolume
    
    %% Define alarm setup
    Stack[CloudFormation Stack] --> SetupLambda[Setup Lambda]
    SetupLambda --> AlarmCreation[Create CloudWatch Alarms]
    SetupLambda --> RuleCreation[Create EventBridge Rules]
    SetupLambda --> PeriodicCheck[Hourly Check for\nNew Tagged Volumes]
    
    %% Define data flow
    CloudWatchAgent[CloudWatch Agent] --> |disk_used_percent| CloudWatch
    
    %% Define styling
    classDef aws fill:#FF9900,stroke:#232F3E,color:black;
    classDef lambda fill:#009900,stroke:#232F3E,color:white;
    classDef setup fill:#3F8624,stroke:#232F3E,color:white;
    
    class CloudWatch,EventBridge,StepFunction,CloudWatchAgent aws;
    class EvaluateVolume,ResizeVolume,CheckOS,LinuxExtend,WindowsExtend lambda;
    class SetupLambda,AlarmCreation,RuleCreation,PeriodicCheck setup;
```

### Key Components

1. **CloudFormation Stack**: Deploys the entire solution with all required components
2. **Alarm Creation Lambda**: Sets up monitoring for tagged volumes
3. **CloudWatch Agent**: Collects disk usage metrics from EC2 instances
4. **CloudWatch Alarms**: Monitor volume usage and trigger when 90% threshold is exceeded
5. **EventBridge Rules**: Route CloudWatch alarm events to the Step Functions state machine
6. **Step Functions State Machine**: Orchestrates the end-to-end resize workflow
7. **Lambda Functions**: Handle specific steps in the workflow:
   - Evaluate Volume Tags: Determines if and how to resize the volume
   - Resize Volume: Performs the actual EBS volume resize
   - Check OS Type: Identifies if volume is attached to Linux or Windows
   - Extend Linux/Windows Filesystem: Extends the filesystem on the respective OS

## Step Functions Workflow

The Step Functions state machine orchestrates the end-to-end resize process:

```mermaid
stateDiagram-v2
    [*] --> EvaluateVolumeTags
    EvaluateVolumeTags --> IsResizeNeeded
    
    IsResizeNeeded --> ResizeNotNeeded: Resize not needed
    IsResizeNeeded --> ResizeVolume: Resize needed
    
    ResizeNotNeeded --> [*]
    
    ResizeVolume --> WaitForResize
    WaitForResize --> CheckOSType
    
    CheckOSType --> IsInstanceAttached
    
    IsInstanceAttached --> NoInstanceAttached: No instance attached
    IsInstanceAttached --> IsWindowsOS: Instance attached
    
    NoInstanceAttached --> [*]
    
    IsWindowsOS --> ExtendWindowsFilesystem: Windows
    IsWindowsOS --> ExtendLinuxFilesystem: Linux
    
    ExtendWindowsFilesystem --> ResizeComplete
    ExtendLinuxFilesystem --> ResizeComplete
    
    ResizeComplete --> [*]
```

### Workflow Steps

1. **EvaluateVolumeTags**: Examines volume tags to determine if resize is needed
2. **IsResizeNeeded**: Decision point based on tag evaluation
3. **ResizeVolume**: Initiates the AWS EBS volume resize operation
4. **WaitForResize**: Waits for the resize operation to complete (30 seconds)
5. **CheckOSType**: Determines if volume is attached to Linux or Windows instance
6. **IsInstanceAttached**: Checks if volume is attached to an instance
7. **IsWindowsOS**: Determines OS type to execute proper filesystem extension
8. **ExtendLinuxFilesystem/ExtendWindowsFilesystem**: Extends the filesystem using SSM Run Command
9. **ResizeComplete**: Successfully completes the workflow

## Tag-Based Configuration

The automation uses the following tags on EBS volumes:

| Tag Name | Value Type | Description | Default |
|----------|------------|-------------|---------|
| `EBSAutoResize` | Boolean (string) | Enable/disable auto-resize | `false` |
| `ResizeAmountGB` | Integer (string) | Amount to resize by in GB | `10` |
| `MaxResizeGB` | Integer (string) | Maximum volume size in GB | `1000` |

Example:

```
EBSAutoResize = true
ResizeAmountGB = 20
MaxResizeGB = 500
```

With this configuration, the volume would resize in 20GB increments when usage exceeds 90%, but would not exceed 500GB total size.

## CloudWatch Agent Metrics

This solution uses the CloudWatch Agent metrics to monitor disk usage. The CloudWatch Agent must be installed and configured on your EC2 instances for the automation to function properly.

### Required Metrics

The solution uses the following CloudWatch Agent metric:

- **disk_used_percent**: The percentage of total disk space that is used
  - **Namespace**: `CWAgent`
  - **Unit**: Percent
  - **Dimensions**:
    - `InstanceId`: The EC2 instance ID
    - `path`: The mount path of the disk (e.g., `/`, `/data`)

### CloudWatch Agent Setup

The CloudWatch Agent should be configured to collect disk usage metrics. This is typically done via the CloudWatch Agent configuration file. Here's a minimal example:

```json
{
  "metrics": {
    "metrics_collected": {
      "disk": {
        "measurement": [
          "used_percent"
        ],
        "resources": [
          "/"
        ],
        "append_dimensions": {
          "InstanceId": "${aws:InstanceId}"
        }
      }
    }
  }
}
```

For complete installation and configuration instructions, refer to the [AWS CloudWatch Agent documentation](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/Install-CloudWatch-Agent.html).

## Installation

### Prerequisites

- AWS CLI configured with appropriate permissions
- Permission to create IAM roles
- Permission to deploy CloudFormation stacks
- CloudWatch Agent installed and configured on target EC2 instances

### Required IAM Permissions

To deploy this CloudFormation template, you'll need the following IAM permissions:

#### Permissions for Deployment

1. **CloudFormation permissions**:
   - `cloudformation:CreateStack`
   - `cloudformation:DescribeStacks`
   - `cloudformation:GetTemplate`

2. **IAM permissions** (since the template creates IAM roles):
   - `iam:CreateRole`
   - `iam:PutRolePolicy`
   - `iam:AttachRolePolicy` 
   - `iam:PassRole`
   - `iam:GetRole`

3. **Service-specific permissions**:
   - `lambda:CreateFunction`
   - `lambda:AddPermission`
   - `states:CreateStateMachine`
   - `events:PutRule`
   - `events:PutTargets`
   - `cloudwatch:PutMetricAlarm`

> **⚠️ Security Note**: While using `AdministratorAccess` policy would work for deploying this template, it is **not recommended** for production environments. Always follow the principle of least privilege by creating a custom policy with only the necessary permissions listed above.

#### IAM Roles Created by the Stack

The CloudFormation template creates all the necessary IAM roles as part of the stack:

1. **EBSResizeStepFunctionRole**: 
   - Allows Step Functions to invoke Lambda functions and perform EC2 operations
   - Provides permissions to describe volumes, modify volumes, and manage tags

2. **EBSResizeLambdaRole**:
   - Grants Lambda functions permissions to interact with EC2, CloudWatch, and SSM
   - Enables the Lambda functions to resize volumes and extend filesystems

3. **EBSAlarmCreationRole**:
   - Gives the alarm creation Lambda permissions to set up CloudWatch alarms
   - Provides access to create EventBridge rules and targets
   - Allows for invoking the Step Functions state machine

The template follows the principle of least privilege, granting each role only the permissions needed for its specific functions within the automation workflow.

### Deployment Steps

1. Save the CloudFormation template to a file (e.g., `ebs-auto-resize.yaml`)
2. Deploy the stack:

```bash
aws cloudformation create-stack \
  --stack-name EBSAutoResize \
  --template-body file://ebs-auto-resize.yaml \
  --capabilities CAPABILITY_IAM
```

3. Wait for stack creation to complete:

```bash
aws cloudformation wait stack-create-complete --stack-name EBSAutoResize
```

### Enabling Auto-Resize on Volumes

To enable auto-resize on an EBS volume, add the following tags:

```bash
aws ec2 create-tags \
  --resources vol-12345abcdef \
  --tags \
    Key=EBSAutoResize,Value=true \
    Key=ResizeAmountGB,Value=20 \
    Key=MaxResizeGB,Value=500
```

## Maintenance and Troubleshooting

### Monitoring the Solution

You can monitor the solution using:

1. **CloudWatch Log Groups**:
   - `/aws/lambda/EvaluateVolumeTagsFunction`
   - `/aws/lambda/ResizeVolumeFunction`
   - `/aws/lambda/CheckOSTypeFunction`
   - `/aws/lambda/ExtendLinuxFilesystemFunction`
   - `/aws/lambda/ExtendWindowsFilesystemFunction`
   - `/aws/lambda/EBSAlarmCreationFunction`

2. **Step Functions Executions**:
   - Navigate to the Step Functions console
   - Select the `EBSResizeStateMachine`
   - View execution history and details

3. **CloudWatch Alarms**:
   - Check the status of `EBSVolumeUsageAlarm-*` alarms

### Common Issues and Resolutions

| Issue | Possible Cause | Resolution |
|-------|----------------|------------|
| Volume not resizing | Tags not applied correctly | Verify tags are spelled correctly with proper values |
| Volume not resizing | CloudWatch alarm not triggered | Check alarm configuration and threshold |
| Volume not resizing | CloudWatch metrics missing | Ensure CloudWatch Agent is installed and collecting disk metrics |
| Volume resized but filesystem not extended | SSM agent not running | Ensure SSM agent is installed and running |
| Filesystem extension failed | Missing permissions | Check IAM roles and SSM permissions |
| Windows filesystem not extending | Partition issue | Check Windows disk management logs |
| Non-ASCII character errors | Mount path contains special characters | Update Lambda function to sanitize strings properly |

### CloudWatch Agent Troubleshooting

If CloudWatch metrics aren't appearing:

1. Check the CloudWatch Agent status:
   ```bash
   sudo systemctl status amazon-cloudwatch-agent
   ```

2. Verify the agent configuration:
   ```bash
   sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a status
   ```

3. Check the agent logs:
   ```bash
   sudo tail -f /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
   ```

4. Manually verify metrics are being sent:
   ```bash
   aws cloudwatch list-metrics --namespace CWAgent --metric-name disk_used_percent
   ```

### Alarm Not Triggering

If alarms aren't triggering when expected:

1. Verify the metrics exist for the instance and path:
   ```bash
   aws cloudwatch get-metric-data \
     --metric-data-queries '[{"Id":"m1","MetricStat":{"Metric":{"Namespace":"CWAgent","MetricName":"disk_used_percent","Dimensions":[{"Name":"InstanceId","Value":"i-12345abcdef"},{"Name":"path","Value":"/"}]},"Period":300,"Stat":"Average"},"ReturnData":true}]' \
     --start-time $(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%SZ) \
     --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ)
   ```

2. Check if the alarm exists:
   ```bash
   aws cloudwatch describe-alarms \
     --alarm-names EBSVolumeUsageAlarm-vol-12345abcdef
   ```

### Updating the Stack

To update the stack with changes:

```bash
aws cloudformation update-stack \
  --stack-name EBSAutoResize \
  --template-body file://ebs-auto-resize.yaml \
  --capabilities CAPABILITY_IAM
```

## Limitations and Considerations

- **Filesystem Types**: The automation supports ext2/3/4 and xfs on Linux, and NTFS on Windows
- **Attached Instances**: For filesystem extension to work, instances must:
  - Have the SSM agent installed and running
  - Have an instance profile allowing SSM operations
  - Be in a running state
- **CloudWatch Agent**: The automation depends on the CloudWatch Agent being installed and configured
- **Resize Timing**: There's a delay between volume resize and filesystem extension (30 seconds)
- **Cost Considerations**: This solution uses:
  - CloudWatch alarms (per alarm pricing)
  - Lambda invocations
  - Step Functions executions
  - SSM Run Command operations
- **Custom Partitions**: The automation may not work with complex disk partitioning schemes
- **Volume Types**: Works with all EBS volume types (gp2, gp3, io1, io2, st1, sc1)

### Security Considerations

- The solution uses IAM roles with least privilege permissions
- SSM operations execute on instances with user-defined permissions
- No sensitive data is stored in permanent storage

## Recent Updates

### CloudWatch Metrics Update

The solution has been updated to use CloudWatch Agent metrics instead of the non-existent `VolumeUsedSize` metric. Key changes include:

1. **Switched to `disk_used_percent` metric**: Now using the standard CloudWatch Agent metric for disk usage
2. **Added dimensions**: Using `InstanceId` and `path` dimensions for proper metric identification
3. **Enhanced string sanitization**: Added robust handling for non-ASCII characters in mount paths
4. **Fixed character encoding issues**: Implemented proper sanitization of CloudWatch parameters

### String Sanitization Improvements

To resolve issues with non-ASCII characters in CloudWatch API calls:

1. **Unicode normalization**: Convert accented characters to their ASCII equivalents
2. **Safe character filtering**: Remove characters that might cause issues with CloudWatch
3. **Fallback paths**: Use default mount path (root "/") when proper path detection fails
4. **Simplified alarm descriptions**: Reduced complexity to avoid character encoding issues