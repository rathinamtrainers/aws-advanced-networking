# Lab Guide: Network Configuration with AWS CDK and Terraform

## Topic: Infrastructure as Code for AWS Networking

### Prerequisites
- AWS account with appropriate IAM permissions
- Basic understanding of VPC, subnets, and routing concepts
- Familiarity with Infrastructure as Code concepts
- AWS CLI installed and configured
- Basic knowledge of TypeScript (for CDK) or HCL (for Terraform)

### Learning Objectives
By the end of this lab, you will be able to:
- Understand IaC principles for AWS networking resources
- Compare manual console configuration with automated IaC approaches
- Deploy network infrastructure using AWS Console as a baseline
- Verify network configurations using AWS CLI commands
- Understand the automation preparation for CDK/Terraform implementation

### Architecture Overview
This lab creates a production-ready VPC architecture with:
- VPC with custom CIDR block
- Public and private subnets across 2 Availability Zones
- Internet Gateway for public subnet connectivity
- NAT Gateway for private subnet internet access
- Route tables with appropriate routes
- Security groups for web and application tiers
- VPC Flow Logs for monitoring

### Lab Duration
Estimated time: 60-75 minutes

---

## Part 1: Manual AWS Console Configuration

### Step 1: Create VPC
**Objective**: Create the foundational VPC for our network architecture

**Theory**: A VPC is an isolated virtual network within AWS. When creating infrastructure as code, you'll define this same resource programmatically with version control and repeatability.

**AWS Console Steps**:
1. Navigate to **VPC Console** → https://console.aws.amazon.com/vpc/
2. Click **Create VPC**
3. Select **VPC only** (we'll create subnets separately)
4. Configure:
   - **Name tag**: `iac-demo-vpc`
   - **IPv4 CIDR block**: `10.0.0.0/16`
   - **IPv6 CIDR block**: No IPv6 CIDR block
   - **Tenancy**: Default
5. Click **Create VPC**

**Verification via AWS CLI**:
```bash
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=iac-demo-vpc" --query 'Vpcs[0].[VpcId,CidrBlock,State]' --output table
```

**Expected Output**:
```
---------------------------------------
|           DescribeVpcs              |
+------------------+------------------+
|  vpc-xxxxxxxxx   |  10.0.0.0/16    |  available  |
+------------------+------------------+
```

---

### Step 2: Create Internet Gateway
**Objective**: Enable internet connectivity for public subnets

**Theory**: Internet Gateways allow resources in public subnets to communicate with the internet. In IaC, this resource will be created and attached to the VPC automatically.

**AWS Console Steps**:
1. In VPC Console, go to **Internet Gateways** → **Create internet gateway**
2. Configure:
   - **Name tag**: `iac-demo-igw`
3. Click **Create internet gateway**
4. Select the newly created IGW → **Actions** → **Attach to VPC**
5. Select `iac-demo-vpc` → **Attach internet gateway**

**Verification via AWS CLI**:
```bash
# Get VPC ID first
export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=iac-demo-vpc" --query 'Vpcs[0].VpcId' --output text)

# Verify IGW attachment
aws ec2 describe-internet-gateways --filters "Name=tag:Name,Values=iac-demo-igw" --query 'InternetGateways[0].[InternetGatewayId,Attachments[0].State,Attachments[0].VpcId]' --output table
```

**Expected Output**:
```
-------------------------------------------------
|        DescribeInternetGateways               |
+------------------+------------+----------------+
|  igw-xxxxxxxxx   | available  | vpc-xxxxxxxxx |
+------------------+------------+----------------+
```

---

### Step 3: Create Subnets
**Objective**: Create public and private subnets across multiple AZs for high availability

**Theory**: Subnet design is crucial for fault tolerance. We'll create subnets in different AZs, which IaC tools can iterate over programmatically.

**AWS Console Steps** (Public Subnet 1):
1. Go to **Subnets** → **Create subnet**
2. Select VPC: `iac-demo-vpc`
3. Click **Add new subnet** and configure:
   - **Subnet name**: `iac-demo-public-1a`
   - **Availability Zone**: Select first AZ (e.g., us-east-1a)
   - **IPv4 CIDR block**: `10.0.1.0/24`
4. Click **Add new subnet** (Public Subnet 2):
   - **Subnet name**: `iac-demo-public-1b`
   - **Availability Zone**: Select second AZ (e.g., us-east-1b)
   - **IPv4 CIDR block**: `10.0.2.0/24`
5. Click **Add new subnet** (Private Subnet 1):
   - **Subnet name**: `iac-demo-private-1a`
   - **Availability Zone**: Same as first AZ (e.g., us-east-1a)
   - **IPv4 CIDR block**: `10.0.11.0/24`
6. Click **Add new subnet** (Private Subnet 2):
   - **Subnet name**: `iac-demo-private-1b`
   - **Availability Zone**: Same as second AZ (e.g., us-east-1b)
   - **IPv4 CIDR block**: `10.0.12.0/24`
7. Click **Create subnet**

**Verification via AWS CLI**:
```bash
# List all subnets in the VPC
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query 'Subnets[].[Tags[?Key==`Name`].Value | [0],SubnetId,CidrBlock,AvailabilityZone]' --output table
```

**Expected Output**:
```
-----------------------------------------------------------------
|                       DescribeSubnets                         |
+------------------------+----------------+----------+-----------+
|  iac-demo-public-1a    | subnet-xxxxx   | 10.0.1.0/24  | us-east-1a |
|  iac-demo-public-1b    | subnet-xxxxx   | 10.0.2.0/24  | us-east-1b |
|  iac-demo-private-1a   | subnet-xxxxx   | 10.0.11.0/24 | us-east-1a |
|  iac-demo-private-1b   | subnet-xxxxx   | 10.0.12.0/24 | us-east-1b |
+------------------------+----------------+----------+-----------+
```

---

### Step 4: Enable Auto-assign Public IP for Public Subnets
**Objective**: Configure public subnets to automatically assign public IPs to instances

**AWS Console Steps**:
1. Select `iac-demo-public-1a` subnet
2. **Actions** → **Edit subnet settings**
3. Enable **Auto-assign public IPv4 address**
4. Click **Save**
5. Repeat for `iac-demo-public-1b`

**Verification via AWS CLI**:
```bash
# Check auto-assign public IP setting
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=iac-demo-public-*" --query 'Subnets[].[Tags[?Key==`Name`].Value | [0],MapPublicIpOnLaunch]' --output table
```

**Expected Output**:
```
-----------------------------------
|      DescribeSubnets            |
+----------------------+-----------+
|  iac-demo-public-1a  |  True     |
|  iac-demo-public-1b  |  True     |
+----------------------+-----------+
```

---

### Step 5: Create and Configure NAT Gateway
**Objective**: Enable private subnet instances to access the internet securely

**Theory**: NAT Gateways allow outbound internet access from private subnets while preventing inbound connections. In IaC, you'll need to consider EIP allocation and proper placement.

**AWS Console Steps**:
1. Go to **Elastic IPs** → **Allocate Elastic IP address**
2. Add tag: `Name` = `iac-demo-natgw-eip`
3. Click **Allocate**
4. Go to **NAT Gateways** → **Create NAT gateway**
5. Configure:
   - **Name**: `iac-demo-natgw-1a`
   - **Subnet**: `iac-demo-public-1a` (must be public subnet)
   - **Elastic IP allocation ID**: Select the EIP created above
6. Click **Create NAT gateway**

**Verification via AWS CLI**:
```bash
# Verify NAT Gateway creation
aws ec2 describe-nat-gateways --filter "Name=tag:Name,Values=iac-demo-natgw-1a" --query 'NatGateways[0].[NatGatewayId,State,SubnetId,NatGatewayAddresses[0].PublicIp]' --output table
```

**Expected Output**:
```
--------------------------------------------------------
|              DescribeNatGateways                     |
+------------------+-----------+-------------+----------+
|  nat-xxxxxxxxx   | available | subnet-xxx  | X.X.X.X  |
+------------------+-----------+-------------+----------+
```

**Note**: Wait 2-3 minutes for NAT Gateway to become available before proceeding.

```bash
# Check NAT Gateway state
aws ec2 describe-nat-gateways --filter "Name=tag:Name,Values=iac-demo-natgw-1a" --query 'NatGateways[0].State' --output text
```

---

### Step 6: Create Route Tables
**Objective**: Configure routing for public and private subnets

**Theory**: Route tables define how traffic flows within and outside the VPC. IaC tools can create and associate these automatically based on subnet type.

**AWS Console Steps - Public Route Table**:
1. Go to **Route Tables** → **Create route table**
2. Configure:
   - **Name**: `iac-demo-public-rt`
   - **VPC**: `iac-demo-vpc`
3. Click **Create route table**
4. Select the route table → **Routes** tab → **Edit routes**
5. Click **Add route**:
   - **Destination**: `0.0.0.0/0`
   - **Target**: Internet Gateway → Select `iac-demo-igw`
6. Click **Save changes**
7. Go to **Subnet associations** tab → **Edit subnet associations**
8. Select `iac-demo-public-1a` and `iac-demo-public-1b`
9. Click **Save associations**

**AWS Console Steps - Private Route Table**:
1. **Create route table**:
   - **Name**: `iac-demo-private-rt`
   - **VPC**: `iac-demo-vpc`
2. Select the route table → **Routes** tab → **Edit routes**
3. Click **Add route**:
   - **Destination**: `0.0.0.0/0`
   - **Target**: NAT Gateway → Select `iac-demo-natgw-1a`
4. Click **Save changes**
5. Go to **Subnet associations** tab → **Edit subnet associations**
6. Select `iac-demo-private-1a` and `iac-demo-private-1b`
7. Click **Save associations**

**Verification via AWS CLI**:
```bash
# Verify public route table
aws ec2 describe-route-tables --filters "Name=tag:Name,Values=iac-demo-public-rt" --query 'RouteTables[0].[RouteTableId,Routes[?DestinationCidrBlock==`0.0.0.0/0`].GatewayId | [0]]' --output table

# Verify private route table
aws ec2 describe-route-tables --filters "Name=tag:Name,Values=iac-demo-private-rt" --query 'RouteTables[0].[RouteTableId,Routes[?DestinationCidrBlock==`0.0.0.0/0`].NatGatewayId | [0]]' --output table

# Verify subnet associations
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" --query 'RouteTables[].[Tags[?Key==`Name`].Value | [0],Associations[].SubnetId]' --output table
```

**Expected Output**:
```
# Public RT should show igw-xxxxxxx
# Private RT should show nat-xxxxxxx
```

---

### Step 7: Create Security Groups
**Objective**: Define security group rules for different application tiers

**Theory**: Security groups are stateful firewalls. In IaC, these are defined as resources with ingress/egress rules as nested properties.

**AWS Console Steps - Web Tier Security Group**:
1. Go to **Security Groups** → **Create security group**
2. Configure:
   - **Name**: `iac-demo-web-sg`
   - **Description**: `Security group for web servers`
   - **VPC**: `iac-demo-vpc`
3. **Inbound rules** → **Add rule**:
   - **Type**: HTTP, **Source**: `0.0.0.0/0`
   - Click **Add rule**: **Type**: HTTPS, **Source**: `0.0.0.0/0`
   - Click **Add rule**: **Type**: SSH, **Source**: `0.0.0.0/0` (use My IP in production)
4. **Outbound rules**: Leave default (all traffic allowed)
5. Click **Create security group**

**AWS Console Steps - App Tier Security Group**:
1. **Create security group**:
   - **Name**: `iac-demo-app-sg`
   - **Description**: `Security group for application servers`
   - **VPC**: `iac-demo-vpc`
2. **Inbound rules** → **Add rule**:
   - **Type**: Custom TCP, **Port**: 8080, **Source**: Custom → Select `iac-demo-web-sg`
3. **Outbound rules**: Leave default
4. Click **Create security group**

**Verification via AWS CLI**:
```bash
# Verify web security group
aws ec2 describe-security-groups --filters "Name=tag:Name,Values=iac-demo-web-sg" "Name=vpc-id,Values=$VPC_ID" --query 'SecurityGroups[0].[GroupId,GroupName,IpPermissions[].{Port:FromPort,Protocol:IpProtocol,CIDR:IpRanges[0].CidrIp}]' --output table

# Verify app security group
aws ec2 describe-security-groups --filters "Name=tag:Name,Values=iac-demo-app-sg" "Name=vpc-id,Values=$VPC_ID" --query 'SecurityGroups[0].[GroupId,GroupName,IpPermissions[].{Port:FromPort,Source:UserIdGroupPairs[0].GroupId}]' --output table
```

---

### Step 8: Enable VPC Flow Logs
**Objective**: Enable network traffic logging for monitoring and troubleshooting

**Theory**: Flow Logs capture IP traffic metadata. IaC can automatically configure this with proper IAM roles and CloudWatch integration.

**AWS Console Steps**:
1. Go to **VPC** → Select `iac-demo-vpc`
2. **Flow logs** tab → **Create flow log**
3. Configure:
   - **Name**: `iac-demo-flowlog`
   - **Filter**: All
   - **Maximum aggregation interval**: 10 minutes
   - **Destination**: Send to CloudWatch Logs
   - **Destination log group**: Create new → `/aws/vpc/flowlogs/iac-demo`
   - **IAM role**: Create new role (or select existing VPC Flow Logs role)
4. Click **Create flow log**

**Verification via AWS CLI**:
```bash
# Verify flow logs
aws ec2 describe-flow-logs --filter "Name=resource-id,Values=$VPC_ID" --query 'FlowLogs[0].[FlowLogId,FlowLogStatus,LogDestinationType,ResourceId]' --output table
```

**Expected Output**:
```
----------------------------------------------------------------
|                     DescribeFlowLogs                         |
+------------------+--------+-------------------+---------------+
|  fl-xxxxxxxxx    | ACTIVE | cloud-watch-logs  | vpc-xxxxxxx  |
+------------------+--------+-------------------+---------------+
```

---

## Part 2: Comprehensive Verification Using AWS CLI

### Verification Script
Create a verification script to audit the entire infrastructure:

```bash
#!/bin/bash
# Save as verify-network-config.sh

echo "=== VPC Configuration Audit ==="
export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=iac-demo-vpc" --query 'Vpcs[0].VpcId' --output text)
echo "VPC ID: $VPC_ID"

echo -e "\n=== Internet Gateway ==="
aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC_ID" --query 'InternetGateways[0].[InternetGatewayId,Attachments[0].State]' --output table

echo -e "\n=== Subnets Summary ==="
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query 'Subnets[].[Tags[?Key==`Name`].Value | [0],SubnetId,CidrBlock,AvailabilityZone,MapPublicIpOnLaunch]' --output table

echo -e "\n=== NAT Gateway ==="
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID" --query 'NatGateways[].[Tags[?Key==`Name`].Value | [0],NatGatewayId,State,NatGatewayAddresses[0].PublicIp]' --output table

echo -e "\n=== Route Tables ==="
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" --query 'RouteTables[].[Tags[?Key==`Name`].Value | [0],RouteTableId,Routes[?GatewayId!=`local`].[DestinationCidrBlock,GatewayId,NatGatewayId] | [0]]' --output table

echo -e "\n=== Security Groups ==="
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=group-name,Values=iac-demo-*" --query 'SecurityGroups[].[GroupName,GroupId,IpPermissions[].{FromPort:FromPort,ToPort:ToPort,Protocol:IpProtocol}]' --output table

echo -e "\n=== VPC Flow Logs ==="
aws ec2 describe-flow-logs --filter "Name=resource-id,Values=$VPC_ID" --query 'FlowLogs[].[FlowLogId,FlowLogStatus,LogDestinationType]' --output table

echo -e "\n=== Resource Count Summary ==="
echo "Public Subnets: $(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=*public*" --query 'length(Subnets)' --output text)"
echo "Private Subnets: $(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" "Name=tag:Name,Values=*private*" --query 'length(Subnets)' --output text)"
echo "NAT Gateways: $(aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID" "Name=state,Values=available" --query 'length(NatGateways)' --output text)"
echo "Security Groups: $(aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=group-name,Values=iac-demo-*" --query 'length(SecurityGroups)' --output text)"
```

**Run the verification**:
```bash
chmod +x verify-network-config.sh
./verify-network-config.sh
```

---

### Advanced Verification Commands

**1. Test Internet Connectivity Route (Public Subnet)**:
```bash
# Get public subnet route table
PUBLIC_RT=$(aws ec2 describe-route-tables --filters "Name=tag:Name,Values=iac-demo-public-rt" --query 'RouteTables[0].RouteTableId' --output text)

# Verify IGW route exists
aws ec2 describe-route-tables --route-table-ids $PUBLIC_RT --query 'RouteTables[0].Routes[?DestinationCidrBlock==`0.0.0.0/0`]' --output table
```

**2. Test NAT Gateway Route (Private Subnet)**:
```bash
# Get private subnet route table
PRIVATE_RT=$(aws ec2 describe-route-tables --filters "Name=tag:Name,Values=iac-demo-private-rt" --query 'RouteTables[0].RouteTableId' --output text)

# Verify NAT Gateway route exists
aws ec2 describe-route-tables --route-table-ids $PRIVATE_RT --query 'RouteTables[0].Routes[?DestinationCidrBlock==`0.0.0.0/0`].NatGatewayId' --output table
```

**3. Verify Security Group References**:
```bash
# Get web SG ID
WEB_SG=$(aws ec2 describe-security-groups --filters "Name=tag:Name,Values=iac-demo-web-sg" --query 'SecurityGroups[0].GroupId' --output text)

# Verify app SG references web SG
aws ec2 describe-security-groups --filters "Name=tag:Name,Values=iac-demo-app-sg" --query "SecurityGroups[0].IpPermissions[?UserIdGroupPairs[?GroupId=='$WEB_SG']]" --output table
```

**4. Check VPC CIDR and Subnet Allocation**:
```bash
# Display CIDR utilization
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query 'Subnets[].[CidrBlock,AvailableIpAddressCount]' --output table
```

**5. Verify Elastic IP Association**:
```bash
# Check EIP is associated with NAT Gateway
aws ec2 describe-addresses --filters "Name=tag:Name,Values=iac-demo-natgw-eip" --query 'Addresses[0].[PublicIp,AssociationId,Domain]' --output table
```

---

## Part 3: Infrastructure Comparison - Manual vs IaC

### What We Built (Manual Console Approach)
✅ VPC with custom CIDR
✅ 4 Subnets (2 public, 2 private) across 2 AZs
✅ Internet Gateway attached to VPC
✅ NAT Gateway in public subnet
✅ 2 Route tables with proper routes
✅ 2 Security groups with rules
✅ VPC Flow Logs enabled

**Challenges with Manual Approach**:
- Time-consuming (60-75 minutes)
- Error-prone (easy to miss configurations)
- Not repeatable across environments (dev/staging/prod)
- No version control
- Difficult to replicate in other regions
- Hard to track changes over time
- Manual cleanup required

### IaC Advantages (CDK/Terraform)
When you implement this with IaC tools:

**AWS CDK Benefits**:
- Define infrastructure in TypeScript/Python/Java
- Type safety and IDE autocomplete
- Reusable constructs and patterns
- Automatic resource dependency management
- Built-in best practices (L2/L3 constructs)
- Deploy entire stack with: `cdk deploy`
- Destroy everything with: `cdk destroy`

**Terraform Benefits**:
- Declarative HCL syntax
- Platform-agnostic (works with multiple clouds)
- State management for tracking changes
- Modules for reusability
- Plan before apply (preview changes)
- Deploy with: `terraform apply`
- Destroy with: `terraform destroy`

**Both provide**:
- Version control with Git
- Automated documentation (code is documentation)
- Environment parity (identical dev/staging/prod)
- Rapid disaster recovery
- Easy rollback capabilities
- Integration with CI/CD pipelines

---

## Part 4: Cleanup

### Manual Cleanup via Console
**IMPORTANT**: Resources must be deleted in specific order to avoid dependency errors.

**Step 1: Delete NAT Gateway** (takes 5-10 minutes):
1. VPC → NAT Gateways → Select → **Actions** → **Delete NAT gateway**
2. Type "delete" → **Delete**
3. Wait for status to change to "Deleted"

**Step 2: Release Elastic IP**:
1. VPC → Elastic IPs → Select EIP → **Actions** → **Release Elastic IP addresses**

**Step 3: Detach and Delete Internet Gateway**:
1. VPC → Internet Gateways → Select IGW → **Actions** → **Detach from VPC**
2. After detached → **Actions** → **Delete internet gateway**

**Step 4: Delete Subnets**:
1. VPC → Subnets → Select all 4 subnets → **Actions** → **Delete subnet**

**Step 5: Delete Route Tables**:
1. VPC → Route Tables → Select custom route tables (not main) → **Actions** → **Delete route table**

**Step 6: Delete Security Groups**:
1. VPC → Security Groups → Select custom SGs → **Actions** → **Delete security groups**

**Step 7: Delete VPC Flow Logs** (automatic when VPC is deleted)

**Step 8: Delete VPC**:
1. VPC → Your VPCs → Select → **Actions** → **Delete VPC**

**Step 9: Delete CloudWatch Log Group**:
1. CloudWatch → Log groups → `/aws/vpc/flowlogs/iac-demo` → **Actions** → **Delete**

### Automated Cleanup via CLI
```bash
#!/bin/bash
# Save as cleanup-network.sh

export VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=iac-demo-vpc" --query 'Vpcs[0].VpcId' --output text)

echo "Deleting NAT Gateway..."
NAT_GW_ID=$(aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID" --query 'NatGateways[0].NatGatewayId' --output text)
aws ec2 delete-nat-gateway --nat-gateway-id $NAT_GW_ID
echo "Waiting for NAT Gateway deletion (this takes 5-10 minutes)..."
aws ec2 wait nat-gateway-deleted --nat-gateway-ids $NAT_GW_ID

echo "Releasing Elastic IP..."
ALLOC_ID=$(aws ec2 describe-addresses --filters "Name=tag:Name,Values=iac-demo-natgw-eip" --query 'Addresses[0].AllocationId' --output text)
aws ec2 release-address --allocation-id $ALLOC_ID

echo "Detaching and deleting Internet Gateway..."
IGW_ID=$(aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC_ID" --query 'InternetGateways[0].InternetGatewayId' --output text)
aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID

echo "Deleting subnets..."
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query 'Subnets[].SubnetId' --output text | xargs -n 1 aws ec2 delete-subnet --subnet-id

echo "Deleting route tables..."
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" --query 'RouteTables[?Associations[0].Main==`false`].RouteTableId' --output text | xargs -n 1 aws ec2 delete-route-table --route-table-id

echo "Deleting security groups..."
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" "Name=group-name,Values=iac-demo-*" --query 'SecurityGroups[].GroupId' --output text | xargs -n 1 aws ec2 delete-security-group --group-id

echo "Deleting VPC..."
aws ec2 delete-vpc --vpc-id $VPC_ID

echo "Deleting CloudWatch Log Group..."
aws logs delete-log-group --log-group-name /aws/vpc/flowlogs/iac-demo

echo "Cleanup complete!"
```

**Run cleanup**:
```bash
chmod +x cleanup-network.sh
./cleanup-network.sh
```

### Verification of Cleanup
```bash
# Verify VPC is deleted
aws ec2 describe-vpcs --filters "Name=tag:Name,Values=iac-demo-vpc" --query 'Vpcs' --output table

# Should return empty table
```

---

## Part 5: AWS CDK Implementation

### CDK Project Setup

**Install AWS CDK**:
```bash
npm install -g aws-cdk
cdk --version
```

**Create CDK Project**:
```bash
mkdir iac-demo-cdk
cd iac-demo-cdk
cdk init app --language typescript
npm install @aws-cdk/aws-ec2 @aws-cdk/aws-logs
```

### CDK Code - TypeScript Implementation

**File: `lib/iac-demo-cdk-stack.ts`**:
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as logs from 'aws-cdk-lib/aws-logs';

export class IacDemoCdkStack extends cdk.Stack {
  constructor(scope: Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Define VPC with high-level construct
    const vpc = new ec2.Vpc(this, 'IacDemoVpc', {
      vpcName: 'iac-demo-vpc',
      ipAddresses: ec2.IpAddresses.cidr('10.0.0.0/16'),
      maxAzs: 2,
      natGateways: 1,

      // Define subnet configuration
      subnetConfiguration: [
        {
          name: 'Public',
          subnetType: ec2.SubnetType.PUBLIC,
          cidrMask: 24,
        },
        {
          name: 'Private',
          subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
          cidrMask: 24,
        },
      ],

      // Enable DNS
      enableDnsHostnames: true,
      enableDnsSupport: true,
    });

    // Add VPC Flow Logs
    const logGroup = new logs.LogGroup(this, 'VpcFlowLogsGroup', {
      logGroupName: '/aws/vpc/flowlogs/iac-demo',
      retention: logs.RetentionDays.ONE_WEEK,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    new ec2.FlowLog(this, 'VpcFlowLog', {
      resourceType: ec2.FlowLogResourceType.fromVpc(vpc),
      destination: ec2.FlowLogDestination.toCloudWatchLogs(logGroup),
      trafficType: ec2.FlowLogTrafficType.ALL,
    });

    // Create Web Tier Security Group
    const webSecurityGroup = new ec2.SecurityGroup(this, 'WebSecurityGroup', {
      vpc,
      securityGroupName: 'iac-demo-web-sg',
      description: 'Security group for web servers',
      allowAllOutbound: true,
    });

    webSecurityGroup.addIngressRule(
      ec2.Peer.anyIpv4(),
      ec2.Port.tcp(80),
      'Allow HTTP from anywhere'
    );

    webSecurityGroup.addIngressRule(
      ec2.Peer.anyIpv4(),
      ec2.Port.tcp(443),
      'Allow HTTPS from anywhere'
    );

    webSecurityGroup.addIngressRule(
      ec2.Peer.anyIpv4(),
      ec2.Port.tcp(22),
      'Allow SSH from anywhere (restrict in production)'
    );

    // Create App Tier Security Group
    const appSecurityGroup = new ec2.SecurityGroup(this, 'AppSecurityGroup', {
      vpc,
      securityGroupName: 'iac-demo-app-sg',
      description: 'Security group for application servers',
      allowAllOutbound: true,
    });

    appSecurityGroup.addIngressRule(
      webSecurityGroup,
      ec2.Port.tcp(8080),
      'Allow traffic from web tier'
    );

    // Add tags
    cdk.Tags.of(vpc).add('Environment', 'Demo');
    cdk.Tags.of(vpc).add('ManagedBy', 'CDK');

    // Outputs
    new cdk.CfnOutput(this, 'VpcId', {
      value: vpc.vpcId,
      description: 'VPC ID',
      exportName: 'IacDemoVpcId',
    });

    new cdk.CfnOutput(this, 'VpcCidr', {
      value: vpc.vpcCidrBlock,
      description: 'VPC CIDR Block',
    });

    new cdk.CfnOutput(this, 'PublicSubnetIds', {
      value: vpc.publicSubnets.map(subnet => subnet.subnetId).join(','),
      description: 'Public Subnet IDs',
    });

    new cdk.CfnOutput(this, 'PrivateSubnetIds', {
      value: vpc.privateSubnets.map(subnet => subnet.subnetId).join(','),
      description: 'Private Subnet IDs',
    });

    new cdk.CfnOutput(this, 'WebSecurityGroupId', {
      value: webSecurityGroup.securityGroupId,
      description: 'Web Security Group ID',
    });

    new cdk.CfnOutput(this, 'AppSecurityGroupId', {
      value: appSecurityGroup.securityGroupId,
      description: 'App Security Group ID',
    });
  }
}
```

### CDK Deployment Commands

**1. Bootstrap CDK (first time only)**:
```bash
cdk bootstrap
```

**2. Synthesize CloudFormation template**:
```bash
cdk synth
```

**3. View changes before deployment**:
```bash
cdk diff
```

**4. Deploy the stack**:
```bash
cdk deploy
```

**Expected output**:
```
IacDemoCdkStack: deploying...
IacDemoCdkStack: creating CloudFormation changeset...

✅  IacDemoCdkStack

Outputs:
IacDemoCdkStack.VpcId = vpc-xxxxxxxxx
IacDemoCdkStack.VpcCidr = 10.0.0.0/16
IacDemoCdkStack.PublicSubnetIds = subnet-xxxxx,subnet-yyyyy
IacDemoCdkStack.PrivateSubnetIds = subnet-aaaaa,subnet-bbbbb
IacDemoCdkStack.WebSecurityGroupId = sg-xxxxxxxxx
IacDemoCdkStack.AppSecurityGroupId = sg-yyyyyyyyy

Stack ARN:
arn:aws:cloudformation:us-east-1:123456789012:stack/IacDemoCdkStack/xxxxxxxx
```

**5. Verify deployment**:
```bash
# Get VPC ID from CDK output
export VPC_ID=$(aws cloudformation describe-stacks --stack-name IacDemoCdkStack --query 'Stacks[0].Outputs[?OutputKey==`VpcId`].OutputValue' --output text)

# Run verification script from Part 2
./verify-network-config.sh
```

**6. Destroy the stack**:
```bash
cdk destroy
```

### Advanced CDK - Parameterized Stack

**File: `lib/iac-demo-parameterized-stack.ts`**:
```typescript
import * as cdk from 'aws-cdk-lib';
import { Construct } from 'constructs';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as logs from 'aws-cdk-lib/aws-logs';

export interface NetworkStackProps extends cdk.StackProps {
  environment: string;
  vpcCidr: string;
  maxAzs: number;
  natGateways: number;
}

export class NetworkStack extends cdk.Stack {
  public readonly vpc: ec2.Vpc;

  constructor(scope: Construct, id: string, props: NetworkStackProps) {
    super(scope, id, props);

    // Create VPC
    this.vpc = new ec2.Vpc(this, 'Vpc', {
      vpcName: `${props.environment}-vpc`,
      ipAddresses: ec2.IpAddresses.cidr(props.vpcCidr),
      maxAzs: props.maxAzs,
      natGateways: props.natGateways,

      subnetConfiguration: [
        {
          name: 'Public',
          subnetType: ec2.SubnetType.PUBLIC,
          cidrMask: 24,
        },
        {
          name: 'Private',
          subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
          cidrMask: 24,
        },
        {
          name: 'Isolated',
          subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
          cidrMask: 24,
        },
      ],
    });

    // Flow Logs
    const logGroup = new logs.LogGroup(this, 'FlowLogs', {
      logGroupName: `/aws/vpc/flowlogs/${props.environment}`,
      retention: logs.RetentionDays.ONE_WEEK,
      removalPolicy: cdk.RemovalPolicy.DESTROY,
    });

    new ec2.FlowLog(this, 'VpcFlowLog', {
      resourceType: ec2.FlowLogResourceType.fromVpc(this.vpc),
      destination: ec2.FlowLogDestination.toCloudWatchLogs(logGroup),
    });

    // Tags
    cdk.Tags.of(this).add('Environment', props.environment);
    cdk.Tags.of(this).add('ManagedBy', 'CDK');
  }
}
```

**File: `bin/iac-demo-cdk.ts`**:
```typescript
#!/usr/bin/env node
import 'source-map-support/register';
import * as cdk from 'aws-cdk-lib';
import { NetworkStack } from '../lib/iac-demo-parameterized-stack';

const app = new cdk.App();

// Development environment
new NetworkStack(app, 'DevNetworkStack', {
  environment: 'dev',
  vpcCidr: '10.0.0.0/16',
  maxAzs: 2,
  natGateways: 1,
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION,
  },
});

// Production environment
new NetworkStack(app, 'ProdNetworkStack', {
  environment: 'prod',
  vpcCidr: '10.1.0.0/16',
  maxAzs: 3,
  natGateways: 3, // One per AZ for HA
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION,
  },
});
```

**Deploy specific environment**:
```bash
# Deploy dev only
cdk deploy DevNetworkStack

# Deploy prod only
cdk deploy ProdNetworkStack

# Deploy all
cdk deploy --all
```

---

## Part 6: Terraform Implementation

### Terraform Project Setup

**Install Terraform**:
```bash
# Download from https://www.terraform.io/downloads
# Or use package manager
# macOS: brew install terraform
# Windows: choco install terraform

terraform --version
```

**Create Terraform Project**:
```bash
mkdir iac-demo-terraform
cd iac-demo-terraform
```

### Terraform Code - HCL Implementation

**File: `main.tf`**:
```hcl
terraform {
  required_version = ">= 1.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-vpc"
    }
  )
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-igw"
    }
  )
}

# Public Subnets
resource "aws_subnet" "public" {
  count = length(var.availability_zones)

  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = var.availability_zones[count.index]
  map_public_ip_on_launch = true

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-public-${var.availability_zones[count.index]}"
      Type = "Public"
    }
  )
}

# Private Subnets
resource "aws_subnet" "private" {
  count = length(var.availability_zones)

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = var.availability_zones[count.index]

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-private-${var.availability_zones[count.index]}"
      Type = "Private"
    }
  )
}

# Elastic IP for NAT Gateway
resource "aws_eip" "nat" {
  count  = var.nat_gateway_count
  domain = "vpc"

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-nat-eip-${count.index + 1}"
    }
  )

  depends_on = [aws_internet_gateway.main]
}

# NAT Gateway
resource "aws_nat_gateway" "main" {
  count = var.nat_gateway_count

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-nat-${var.availability_zones[count.index]}"
    }
  )

  depends_on = [aws_internet_gateway.main]
}

# Public Route Table
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-public-rt"
    }
  )
}

# Public Route Table Association
resource "aws_route_table_association" "public" {
  count = length(aws_subnet.public)

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# Private Route Table
resource "aws_route_table" "private" {
  count = var.nat_gateway_count

  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-private-rt-${count.index + 1}"
    }
  )
}

# Private Route Table Association
resource "aws_route_table_association" "private" {
  count = length(aws_subnet.private)

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index % var.nat_gateway_count].id
}

# Web Security Group
resource "aws_security_group" "web" {
  name        = "${var.environment}-web-sg"
  description = "Security group for web servers"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP from anywhere"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS from anywhere"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "SSH from anywhere"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-web-sg"
    }
  )
}

# App Security Group
resource "aws_security_group" "app" {
  name        = "${var.environment}-app-sg"
  description = "Security group for application servers"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "App port from web tier"
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]
  }

  egress {
    description = "Allow all outbound"
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-app-sg"
    }
  )
}

# VPC Flow Logs
resource "aws_flow_log" "main" {
  vpc_id          = aws_vpc.main.id
  traffic_type    = "ALL"
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn

  tags = merge(
    var.common_tags,
    {
      Name = "${var.environment}-flow-logs"
    }
  )
}

# CloudWatch Log Group for Flow Logs
resource "aws_cloudwatch_log_group" "flow_logs" {
  name              = "/aws/vpc/flowlogs/${var.environment}"
  retention_in_days = 7

  tags = var.common_tags
}

# IAM Role for Flow Logs
resource "aws_iam_role" "flow_logs" {
  name = "${var.environment}-vpc-flow-logs-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          Service = "vpc-flow-logs.amazonaws.com"
        }
        Action = "sts:AssumeRole"
      }
    ]
  })

  tags = var.common_tags
}

# IAM Role Policy for Flow Logs
resource "aws_iam_role_policy" "flow_logs" {
  name = "${var.environment}-vpc-flow-logs-policy"
  role = aws_iam_role.flow_logs.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogGroups",
          "logs:DescribeLogStreams"
        ]
        Resource = "*"
      }
    ]
  })
}
```

**File: `variables.tf`**:
```hcl
variable "aws_region" {
  description = "AWS region for resources"
  type        = string
  default     = "us-east-1"
}

variable "environment" {
  description = "Environment name (dev, staging, prod)"
  type        = string
  default     = "demo"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
}

variable "availability_zones" {
  description = "List of availability zones"
  type        = list(string)
  default     = ["us-east-1a", "us-east-1b"]
}

variable "nat_gateway_count" {
  description = "Number of NAT Gateways (1 for cost, 2+ for HA)"
  type        = number
  default     = 1
}

variable "common_tags" {
  description = "Common tags for all resources"
  type        = map(string)
  default = {
    ManagedBy   = "Terraform"
    Project     = "IaC Demo"
  }
}
```

**File: `outputs.tf`**:
```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "vpc_cidr" {
  description = "VPC CIDR block"
  value       = aws_vpc.main.cidr_block
}

output "public_subnet_ids" {
  description = "List of public subnet IDs"
  value       = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  description = "List of private subnet IDs"
  value       = aws_subnet.private[*].id
}

output "nat_gateway_ips" {
  description = "Elastic IPs of NAT Gateways"
  value       = aws_eip.nat[*].public_ip
}

output "web_security_group_id" {
  description = "Web tier security group ID"
  value       = aws_security_group.web.id
}

output "app_security_group_id" {
  description = "App tier security group ID"
  value       = aws_security_group.app.id
}
```

### Terraform Deployment Commands

**1. Initialize Terraform**:
```bash
terraform init
```

**Expected output**:
```
Initializing the backend...
Initializing provider plugins...
- Finding hashicorp/aws versions matching "~> 5.0"...
- Installing hashicorp/aws v5.x.x...

Terraform has been successfully initialized!
```

**2. Format and validate**:
```bash
terraform fmt
terraform validate
```

**3. Plan deployment**:
```bash
terraform plan
```

**Expected output**:
```
Plan: 24 to add, 0 to change, 0 to destroy.
```

**4. Apply configuration**:
```bash
terraform apply
```

Type `yes` when prompted.

**Expected output**:
```
Apply complete! Resources: 24 added, 0 changed, 0 destroyed.

Outputs:

app_security_group_id = "sg-xxxxxxxxx"
nat_gateway_ips = [
  "X.X.X.X",
]
private_subnet_ids = [
  "subnet-xxxxxxxxx",
  "subnet-yyyyyyyyy",
]
public_subnet_ids = [
  "subnet-aaaaaaa",
  "subnet-bbbbbbb",
]
vpc_cidr = "10.0.0.0/16"
vpc_id = "vpc-xxxxxxxxx"
web_security_group_id = "sg-yyyyyyyyy"
```

**5. Verify deployment**:
```bash
# Export VPC ID from Terraform output
export VPC_ID=$(terraform output -raw vpc_id)

# Run verification script
./verify-network-config.sh
```

**6. Show current state**:
```bash
terraform show
```

**7. Destroy infrastructure**:
```bash
terraform destroy
```

Type `yes` when prompted.

### Multi-Environment Terraform with Workspaces

**File: `terraform.tfvars.dev`**:
```hcl
environment          = "dev"
vpc_cidr             = "10.0.0.0/16"
availability_zones   = ["us-east-1a", "us-east-1b"]
nat_gateway_count    = 1
aws_region          = "us-east-1"

common_tags = {
  ManagedBy   = "Terraform"
  Environment = "Development"
  CostCenter  = "Engineering"
}
```

**File: `terraform.tfvars.prod`**:
```hcl
environment          = "prod"
vpc_cidr             = "10.1.0.0/16"
availability_zones   = ["us-east-1a", "us-east-1b", "us-east-1c"]
nat_gateway_count    = 3  # One per AZ for HA
aws_region          = "us-east-1"

common_tags = {
  ManagedBy   = "Terraform"
  Environment = "Production"
  CostCenter  = "Operations"
}
```

**Deploy environments**:
```bash
# Deploy dev
terraform apply -var-file="terraform.tfvars.dev"

# Deploy prod
terraform apply -var-file="terraform.tfvars.prod"
```

**Or use Terraform Workspaces**:
```bash
# Create workspaces
terraform workspace new dev
terraform workspace new prod

# Switch to dev
terraform workspace select dev
terraform apply -var-file="terraform.tfvars.dev"

# Switch to prod
terraform workspace select prod
terraform apply -var-file="terraform.tfvars.prod"

# List workspaces
terraform workspace list
```

---

## Part 7: CDK vs Terraform Comparison

### Feature Comparison

| Feature | AWS CDK | Terraform |
|---------|---------|-----------|
| **Language** | TypeScript, Python, Java, C#, Go | HCL (HashiCorp Configuration Language) |
| **Cloud Support** | AWS only | Multi-cloud (AWS, Azure, GCP, etc.) |
| **Learning Curve** | Steeper (requires programming knowledge) | Moderate (declarative syntax) |
| **Abstraction Level** | High (L1, L2, L3 constructs) | Medium (resources with modules) |
| **State Management** | CloudFormation (AWS managed) | Terraform state file (local or remote) |
| **Preview Changes** | `cdk diff` | `terraform plan` |
| **Deployment** | `cdk deploy` | `terraform apply` |
| **Best For** | AWS-native teams, complex logic | Multi-cloud, existing HCL expertise |
| **IDE Support** | Excellent (IntelliSense, type checking) | Good (syntax highlighting, completion) |
| **Community** | Growing AWS-focused | Large multi-cloud community |
| **Cost** | Free (CloudFormation is free) | Free for basic, paid for enterprise features |

### When to Choose CDK
✅ AWS-only infrastructure
✅ Team has programming expertise (TypeScript, Python, etc.)
✅ Need complex conditional logic
✅ Want AWS best practices built-in (L2/L3 constructs)
✅ Prefer managed state (CloudFormation)
✅ Need type safety and IDE support

### When to Choose Terraform
✅ Multi-cloud infrastructure
✅ Team prefers declarative over imperative
✅ Need provider ecosystem (1000+ providers)
✅ Existing Terraform expertise
✅ Want explicit control over resources
✅ Need state file portability

### Code Comparison Example

**Creating a VPC:**

**CDK (TypeScript) - ~10 lines**:
```typescript
const vpc = new ec2.Vpc(this, 'MyVpc', {
  ipAddresses: ec2.IpAddresses.cidr('10.0.0.0/16'),
  maxAzs: 2,
  natGateways: 1,
});
```
*Automatically creates: VPC, subnets, IGW, NAT GW, route tables, and associations*

**Terraform (HCL) - ~100 lines for equivalent**:
```hcl
resource "aws_vpc" "main" { ... }
resource "aws_internet_gateway" "main" { ... }
resource "aws_subnet" "public" { ... }
resource "aws_subnet" "private" { ... }
resource "aws_eip" "nat" { ... }
resource "aws_nat_gateway" "main" { ... }
resource "aws_route_table" "public" { ... }
resource "aws_route_table" "private" { ... }
# ... plus associations
```
*More explicit control, but more code*

### Deployment Time Comparison

**Manual Console**: 60-75 minutes
**CDK**: 5-7 minutes (first deploy), 2-3 minutes (updates)
**Terraform**: 5-8 minutes (first apply), 2-4 minutes (updates)

---

## Key Takeaways

### Manual Configuration Insights
1. **Order matters**: Resources have dependencies (NAT GW needs subnet, which needs VPC)
2. **Time investment**: Manual setup took 60-75 minutes
3. **Error potential**: Easy to misconfigure routes or security groups
4. **Cleanup complexity**: Reverse order deletion required

### IaC Preparation Lessons
1. **Resource naming**: Consistent naming enables automation (tags are crucial)
2. **Network design**: CIDR planning must happen upfront
3. **Security defaults**: Auto-assign public IP must be explicit
4. **Monitoring setup**: Flow Logs require IAM role creation

### CLI Verification Value
1. **Scriptable audits**: Verification can be automated
2. **State validation**: CLI confirms console actions
3. **Documentation**: CLI commands serve as runbooks
4. **Troubleshooting**: CLI output helps debug misconfigurations

### Transition to IaC
**What you'll code in CDK/Terraform**:
- VPC resource definition
- Subnet loops with CIDR calculation
- Conditional NAT Gateway (per AZ or shared)
- Route table associations using references
- Security group rule sets
- Tags and naming conventions
- Output values for other stacks

**IaC will eliminate**:
- Manual clicking through console
- Human error in configuration
- Time wasted on repetitive tasks
- Environment drift
- Undocumented changes

---

## Troubleshooting

| Issue | Cause | Solution |
|-------|-------|----------|
| NAT Gateway stuck in "Pending" | Normal behavior | Wait 2-3 minutes, check status with CLI |
| Cannot delete VPC | Resources still attached | Delete in order: NAT GW → IGW → Subnets → RT → SG → VPC |
| Route not propagating | Wrong route table association | Verify subnet associations in route table |
| Elastic IP cannot be released | Still associated with NAT GW | Delete NAT Gateway first, wait for deletion |
| Flow Logs not appearing | IAM role permissions | Verify CloudWatch Logs write permissions |
| Security group cannot be deleted | Referenced by another SG | Delete dependent security groups first |
| Public subnet has no internet | No IGW route or IGW not attached | Verify 0.0.0.0/0 → IGW route exists |
| Private subnet has no internet | NAT Gateway not available or no route | Check NAT GW state and 0.0.0.0/0 → NAT route |

---

## Additional Resources

### AWS Documentation
- [VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/)
- [AWS CDK VPC Construct](https://docs.aws.amazon.com/cdk/api/v2/docs/aws-cdk-lib.aws_ec2.Vpc.html)
- [Terraform AWS VPC Module](https://registry.terraform.io/modules/terraform-aws-modules/vpc/aws/latest)
- [VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)

### Next Steps for IaC
- **Topic 188**: Automating Networking with AWS CloudFormation
- **Topic 190**: Network Policy Automation with Firewall Manager
- **Topic 195**: Integrating AWS Networking with CI/CD Pipelines

### Related Topics
- **Topic 6**: Deep Dive into VPC Route Tables
- **Topic 34**: Designing Highly Available VPC Architectures
- **Topic 91**: AWS Network Cost Optimization Strategies

---

## Exam Tips

### AWS Networking Specialty Exam
1. **IaC questions focus on**: Repeatability, version control, automation benefits
2. **Common scenarios**:
   - Multi-environment deployment (dev/staging/prod must be identical)
   - Disaster recovery (quickly rebuild in another region)
   - Compliance requirements (auditable infrastructure changes)
3. **Key differences**:
   - CloudFormation: AWS-native, YAML/JSON, integrates with AWS services
   - CDK: Programming languages, higher-level abstracts, compiles to CFN
   - Terraform: Multi-cloud, HCL syntax, state management

### Gotchas to Remember
- **NAT Gateway placement**: Must be in public subnet with EIP
- **Route table associations**: Each subnet can only associate with ONE route table
- **Security group references**: Can reference other SGs (useful for app tiers)
- **CIDR planning**: Cannot change VPC CIDR after creation (plan carefully)
- **Flow Logs**: Cannot delete log stream, must delete entire log group
- **IGW limit**: One IGW per VPC (cannot attach multiple)
- **Cleanup order**: Always delete dependencies first (NAT GW before EIP, etc.)

### IaC Best Practices for Exam
1. Use **parameters/variables** for CIDR blocks and environment names
2. Implement **tagging strategy** for cost allocation and automation
3. Enable **termination protection** for production resources
4. Use **modules/stacks** for reusable network components
5. Implement **state locking** (Terraform) or **stack policies** (CloudFormation)
6. Always use **version control** for infrastructure code
7. Implement **CI/CD pipelines** for infrastructure deployment
