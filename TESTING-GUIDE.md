# VPC CIDR Expansion Testing Guide

## Overview
This guide walks through testing VPC CIDR expansion with an ECS Fargate cluster that will exhaust IP addresses.

## Initial Setup

### Set your region:
```bash
export AWS_REGION=your-region
```

Note: If you need to use a specific AWS profile, set it with `export AWS_PROFILE=your-profile-name`

### 1. Deploy the CloudFormation Stack

```bash
aws cloudformation create-stack \
  --stack-name vpc-cidr-test \
  --template-body file://vpc-cidr-expansion-test.yaml \
  --capabilities CAPABILITY_IAM \
  --parameters ParameterKey=DesiredTaskCount,ParameterValue=5 \
  --region $AWS_REGION
```

Wait for stack creation:
```bash
aws cloudformation wait stack-create-complete \
  --stack-name vpc-cidr-test \
  --region $AWS_REGION
```

Get stack outputs:
```bash
aws cloudformation describe-stacks \
  --stack-name vpc-cidr-test \
  --query 'Stacks[0].Outputs' \
  --region $AWS_REGION
```

## Step 1: Exhaust IP Addresses

### Check current IP usage:
```bash
# Get VPC ID
VPC_ID=$(aws cloudformation describe-stacks \
  --stack-name vpc-cidr-test \
  --query 'Stacks[0].Outputs[?OutputKey==`VPCId`].OutputValue' \
  --output text \
  --region $AWS_REGION)

# Check available IPs in each subnet
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailableIpAddressCount]' \
  --output table \
  --region $AWS_REGION
```

### Increase task count to exhaust IPs:
```bash
# Update service to run more tasks (will fail when IPs exhausted)
# With 3 subnets of /28 (11 usable IPs each) = 33 total IPs
# Try to run 40 tasks to exceed capacity
aws ecs update-service \
  --cluster TestCluster-IPExhaustion \
  --service test-service \
  --desired-count 40 \
  --region $AWS_REGION
```

### Monitor for IP exhaustion:
```bash
# Check service events for errors
aws ecs describe-services \
  --cluster TestCluster-IPExhaustion \
  --services test-service \
  --query 'services[0].events[0:5]' \
  --region $AWS_REGION
```

You should see errors like: "service test-service was unable to place a task because no container instance met all of its requirements" or network interface allocation failures.

## Step 2: Add Secondary CIDR Block to VPC

### Add new CIDR range:
```bash
aws ec2 associate-vpc-cidr-block \
  --vpc-id $VPC_ID \
  --cidr-block 10.1.0.0/24 \
  --region $AWS_REGION
```

### Wait for CIDR association:
```bash
aws ec2 describe-vpcs \
  --vpc-ids $VPC_ID \
  --query 'Vpcs[0].CidrBlockAssociationSet[*].[CidrBlock,CidrBlockState.State]' \
  --output table \
  --region $AWS_REGION
```

Wait until state shows "associated".

### Get route table and availability zones:
```bash
# Get route table ID
ROUTE_TABLE_ID=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[0].RouteTableId' \
  --output text \
  --region $AWS_REGION)

# Get availability zones
AZ1=$(aws ec2 describe-availability-zones \
  --query 'AvailabilityZones[0].ZoneName' \
  --output text \
  --region $AWS_REGION)
  
AZ2=$(aws ec2 describe-availability-zones \
  --query 'AvailabilityZones[1].ZoneName' \
  --output text \
  --region $AWS_REGION)
  
AZ3=$(aws ec2 describe-availability-zones \
  --query 'AvailabilityZones[2].ZoneName' \
  --output text \
  --region $AWS_REGION)
```

### Create new subnets:
```bash
# Create new subnet in AZ1
SUBNET_NEW_A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.1.0.0/26 \
  --availability-zone $AZ1 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=TestSubnet-New-A}]' \
  --query 'Subnet.SubnetId' \
  --output text \
  --region $AWS_REGION)

# Create new subnet in AZ2
SUBNET_NEW_B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.1.0.64/26 \
  --availability-zone $AZ2 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=TestSubnet-New-B}]' \
  --query 'Subnet.SubnetId' \
  --output text \
  --region $AWS_REGION)

# Create new subnet in AZ3
SUBNET_NEW_C=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 10.1.0.128/26 \
  --availability-zone $AZ3 \
  --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=TestSubnet-New-C}]' \
  --query 'Subnet.SubnetId' \
  --output text \
  --region $AWS_REGION)

# Associate subnets with route table
aws ec2 associate-route-table --subnet-id $SUBNET_NEW_A --route-table-id $ROUTE_TABLE_ID --region $AWS_REGION
aws ec2 associate-route-table --subnet-id $SUBNET_NEW_B --route-table-id $ROUTE_TABLE_ID --region $AWS_REGION
aws ec2 associate-route-table --subnet-id $SUBNET_NEW_C --route-table-id $ROUTE_TABLE_ID --region $AWS_REGION

# Enable auto-assign public IP
aws ec2 modify-subnet-attribute --subnet-id $SUBNET_NEW_A --map-public-ip-on-launch --region $AWS_REGION
aws ec2 modify-subnet-attribute --subnet-id $SUBNET_NEW_B --map-public-ip-on-launch --region $AWS_REGION
aws ec2 modify-subnet-attribute --subnet-id $SUBNET_NEW_C --map-public-ip-on-launch --region $AWS_REGION
```

## Step 3: Reconfigure ECS Service to Use New Subnets

You have two options for reconfiguring the ECS service:

### Get original subnets and security group:
```bash
SECURITY_GROUP=$(aws cloudformation describe-stacks \
  --stack-name vpc-cidr-test \
  --query 'Stacks[0].Outputs[?OutputKey==`SecurityGroupId`].OutputValue' \
  --output text \
  --region $AWS_REGION)

# Get original subnet IDs (comma-separated)
ORIGINAL_SUBNETS=$(aws cloudformation describe-stacks \
  --stack-name vpc-cidr-test \
  --query 'Stacks[0].Outputs[?OutputKey==`SubnetIds`].OutputValue' \
  --output text \
  --region $AWS_REGION)

# Parse individual subnet IDs
SUBNET_OLD_A=$(echo $ORIGINAL_SUBNETS | cut -d',' -f1)
SUBNET_OLD_B=$(echo $ORIGINAL_SUBNETS | cut -d',' -f2)
SUBNET_OLD_C=$(echo $ORIGINAL_SUBNETS | cut -d',' -f3)
```

### Path 1: Replace old subnets with new subnets only
This approach completely switches to the new CIDR range. Existing tasks will continue running in old subnets, but all new tasks will launch in new subnets only.

```bash
aws ecs update-service \
  --cluster TestCluster-IPExhaustion \
  --service test-service \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_NEW_A,$SUBNET_NEW_B,$SUBNET_NEW_C],securityGroups=[$SECURITY_GROUP],assignPublicIp=ENABLED}" \
  --region $AWS_REGION
```

### Path 2: Add new subnets while keeping old ones (Recommended)
This approach keeps both old and new subnets available. New tasks can launch in either range, providing maximum flexibility and capacity.

```bash
aws ecs update-service \
  --cluster TestCluster-IPExhaustion \
  --service test-service \
  --network-configuration "awsvpcConfiguration={subnets=[$SUBNET_OLD_A,$SUBNET_OLD_B,$SUBNET_OLD_C,$SUBNET_NEW_A,$SUBNET_NEW_B,$SUBNET_NEW_C],securityGroups=[$SECURITY_GROUP],assignPublicIp=ENABLED}" \
  --region $AWS_REGION
```

**Note**: Path 2 is recommended because:
- Provides maximum IP capacity (old + new subnets)
- No disruption to existing tasks
- ECS will distribute tasks across all available subnets
- More resilient to future IP exhaustion

### Verify tasks can now be placed:
```bash
# Check service status
aws ecs describe-services \
  --cluster TestCluster-IPExhaustion \
  --services test-service \
  --query 'services[0].[runningCount,desiredCount,pendingCount]' \
  --region $AWS_REGION

# Check for successful task placement
aws ecs describe-services \
  --cluster TestCluster-IPExhaustion \
  --services test-service \
  --query 'services[0].events[0:3]' \
  --region $AWS_REGION
```

### Verify IP availability in new subnets:
```bash
aws ec2 describe-subnets \
  --subnet-ids $SUBNET_NEW_A $SUBNET_NEW_B $SUBNET_NEW_C \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailableIpAddressCount]' \
  --output table \
  --region $AWS_REGION
```

## Cleanup

**IMPORTANT**: Cleanup must be done in the correct order to avoid dependency errors. Follow these steps sequentially:

### Step 1: Stop All ECS Tasks
```bash
# Scale service to 0 to stop all tasks
aws ecs update-service \
  --cluster TestCluster-IPExhaustion \
  --service test-service \
  --desired-count 0 \
  --region $AWS_REGION

# Wait for all tasks to stop (this may take 1-2 minutes)
aws ecs wait services-stable \
  --cluster TestCluster-IPExhaustion \
  --services test-service \
  --region $AWS_REGION

# Verify no tasks are running
aws ecs describe-services \
  --cluster TestCluster-IPExhaustion \
  --services test-service \
  --query 'services[0].[runningCount,pendingCount]' \
  --region $AWS_REGION
```

### Step 2: Delete Manually Created Subnets
```bash
# Delete the new subnets
aws ec2 delete-subnet --subnet-id $SUBNET_NEW_A --region $AWS_REGION
aws ec2 delete-subnet --subnet-id $SUBNET_NEW_B --region $AWS_REGION
aws ec2 delete-subnet --subnet-id $SUBNET_NEW_C --region $AWS_REGION
```

**If subnet deletion fails with "has dependencies":**
```bash
# Check for remaining ENIs in the subnets
aws ec2 describe-network-interfaces \
  --filters "Name=subnet-id,Values=$SUBNET_NEW_A,$SUBNET_NEW_B,$SUBNET_NEW_C" \
  --query 'NetworkInterfaces[*].[NetworkInterfaceId,Status,Description]' \
  --region $AWS_REGION

# Wait a few minutes for AWS to clean up ENIs automatically, then retry subnet deletion
```

### Step 3: Disassociate Secondary CIDR Block
```bash
# Get the association ID for the secondary CIDR
ASSOCIATION_ID=$(aws ec2 describe-vpcs \
  --vpc-ids $VPC_ID \
  --query 'Vpcs[0].CidrBlockAssociationSet[?CidrBlock==`10.1.0.0/24`].AssociationId' \
  --output text \
  --region $AWS_REGION)

# Disassociate the secondary CIDR
aws ec2 disassociate-vpc-cidr-block \
  --association-id $ASSOCIATION_ID \
  --region $AWS_REGION
```

### Step 4: Delete CloudFormation Stack
```bash
# Delete the stack - this will clean up VPC, original subnets, ECS cluster, etc.
aws cloudformation delete-stack \
  --stack-name vpc-cidr-test \
  --region $AWS_REGION

# Monitor deletion progress
aws cloudformation wait stack-delete-complete \
  --stack-name vpc-cidr-test \
  --region $AWS_REGION
```

### Why This Order Matters

1. **ECS Tasks** create ENIs (Elastic Network Interfaces) in subnets
2. **Subnets** can't be deleted while ENIs exist
3. **CIDR blocks** can't be disassociated while subnets exist
4. **Route tables** have associations with subnets
5. **VPC** can't be deleted until all resources are removed

### Alternative: Automated Cleanup Script

Save this as `cleanup.sh` for automated cleanup:

```bash
#!/bin/bash
set -e

echo "Step 1: Scaling ECS service to 0..."
aws ecs update-service \
  --cluster TestCluster-IPExhaustion \
  --service test-service \
  --desired-count 0 \
  --region $AWS_REGION

echo "Waiting for tasks to stop (this may take 2-3 minutes)..."
sleep 120

echo "Step 2: Deleting manually created subnets..."
for subnet in $SUBNET_NEW_A $SUBNET_NEW_B $SUBNET_NEW_C; do
  echo "Attempting to delete $subnet..."
  aws ec2 delete-subnet --subnet-id $subnet --region $AWS_REGION 2>/dev/null || echo "Failed, will retry..."
  sleep 10
done

echo "Step 3: Disassociating secondary CIDR..."
ASSOCIATION_ID=$(aws ec2 describe-vpcs \
  --vpc-ids $VPC_ID \
  --query 'Vpcs[0].CidrBlockAssociationSet[?CidrBlock==`10.1.0.0/24`].AssociationId' \
  --output text \
  --region $AWS_REGION)

aws ec2 disassociate-vpc-cidr-block \
  --association-id $ASSOCIATION_ID \
  --region $AWS_REGION

echo "Step 4: Deleting CloudFormation stack..."
aws cloudformation delete-stack \
  --stack-name vpc-cidr-test \
  --region $AWS_REGION

echo "Cleanup initiated. Monitor with:"
echo "aws cloudformation describe-stacks --stack-name vpc-cidr-test --region $AWS_REGION"
```

Run with: `bash cleanup.sh`

## Key Observations

1. **IP Exhaustion**: With /28 subnets (11 usable IPs each × 3 subnets = 33 total), you'll hit limits around 33 tasks
2. **Non-disruptive**: Existing tasks continue running while new subnets are added
3. **Immediate availability**: New tasks can launch in new subnets right away
4. **No downtime**: Service remains available throughout the process
