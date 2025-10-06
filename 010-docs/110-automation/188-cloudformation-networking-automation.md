# Lab Guide: Automating Networking with AWS CloudFormation

## Topic: Deploy Complete VPC Infrastructure Using Infrastructure as Code

### Prerequisites
- AWS account with permissions for VPC, EC2, CloudFormation
- Basic understanding of YAML/JSON syntax
- Familiarity with VPC concepts (subnets, route tables, gateways)
- AWS CLI installed and configured
- Text editor for creating CloudFormation templates

### Learning Objectives
By the end of this lab, you will be able to:
- Write CloudFormation templates for VPC infrastructure
- Deploy multi-tier network architectures using CloudFormation stacks
- Use CloudFormation parameters and outputs for reusability
- Implement best practices for Infrastructure as Code
- Update and manage network infrastructure through CloudFormation
- Delete and rollback network resources safely

### Architecture Overview
This lab deploys a production-ready VPC architecture with:
- **VPC**: Custom VPC with public and private subnets
- **Public Subnets**: 2 AZs with Internet Gateway access
- **Private Subnets**: 2 AZs with NAT Gateway access
- **NAT Gateway**: High availability across 2 AZs
- **Route Tables**: Separate routing for public and private subnets
- **Security Groups**: Web tier and application tier security
- **Bastion Host**: Secure access to private resources

**Architecture Diagram:**
```
                    Internet Gateway
                           |
                    +--------------+
                    |   VPC        |
                    | 10.0.0.0/16  |
                    +--------------+
                    /              \
        Public Subnet 1         Public Subnet 2
        10.0.1.0/24            10.0.2.0/24
        (us-east-1a)           (us-east-1b)
            |                       |
        NAT GW 1                NAT GW 2
            |                       |
        Private Subnet 1        Private Subnet 2
        10.0.11.0/24           10.0.12.0/24
        (us-east-1a)           (us-east-1b)
```

### Lab Duration
Estimated time: 60-75 minutes

---

## Lab Steps

### Step 1: Create Basic VPC CloudFormation Template

**Objective**: Create a CloudFormation template to define a VPC

**Theory**: CloudFormation uses declarative templates (YAML or JSON) to define AWS resources. The template describes the desired state, and CloudFormation handles the creation, update, and deletion of resources.

**Create Template File**:

Create a file named `vpc-basic.yaml`:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Basic VPC with public and private subnets across 2 AZs'

Parameters:
  EnvironmentName:
    Description: Environment name prefix for resources
    Type: String
    Default: CFN-Lab

  VpcCIDR:
    Description: CIDR block for VPC
    Type: String
    Default: 10.0.0.0/16
    AllowedPattern: '^(([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.){3}([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])(\/([0-9]|[1-2][0-9]|3[0-2]))$'

  PublicSubnet1CIDR:
    Description: CIDR block for public subnet in AZ1
    Type: String
    Default: 10.0.1.0/24

  PublicSubnet2CIDR:
    Description: CIDR block for public subnet in AZ2
    Type: String
    Default: 10.0.2.0/24

  PrivateSubnet1CIDR:
    Description: CIDR block for private subnet in AZ1
    Type: String
    Default: 10.0.11.0/24

  PrivateSubnet2CIDR:
    Description: CIDR block for private subnet in AZ2
    Type: String
    Default: 10.0.12.0/24

Resources:
  # VPC
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-VPC'
        - Key: Environment
          Value: !Ref EnvironmentName

  # Internet Gateway
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-IGW'

  InternetGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      InternetGatewayId: !Ref InternetGateway
      VpcId: !Ref VPC

  # Public Subnets
  PublicSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !Ref PublicSubnet1CIDR
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-Public-Subnet-AZ1'
        - Key: Type
          Value: Public

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !Ref PublicSubnet2CIDR
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-Public-Subnet-AZ2'
        - Key: Type
          Value: Public

  # Private Subnets
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !Ref PrivateSubnet1CIDR
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-Private-Subnet-AZ1'
        - Key: Type
          Value: Private

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !Ref PrivateSubnet2CIDR
      MapPublicIpOnLaunch: false
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-Private-Subnet-AZ2'
        - Key: Type
          Value: Private

  # NAT Gateways
  NatGateway1EIP:
    Type: AWS::EC2::EIP
    DependsOn: InternetGatewayAttachment
    Properties:
      Domain: vpc
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-NAT-GW-EIP-AZ1'

  NatGateway2EIP:
    Type: AWS::EC2::EIP
    DependsOn: InternetGatewayAttachment
    Properties:
      Domain: vpc
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-NAT-GW-EIP-AZ2'

  NatGateway1:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGateway1EIP.AllocationId
      SubnetId: !Ref PublicSubnet1
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-NAT-GW-AZ1'

  NatGateway2:
    Type: AWS::EC2::NatGateway
    Properties:
      AllocationId: !GetAtt NatGateway2EIP.AllocationId
      SubnetId: !Ref PublicSubnet2
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-NAT-GW-AZ2'

  # Public Route Table
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-Public-RT'

  DefaultPublicRoute:
    Type: AWS::EC2::Route
    DependsOn: InternetGatewayAttachment
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway

  PublicSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet1

  PublicSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PublicRouteTable
      SubnetId: !Ref PublicSubnet2

  # Private Route Tables
  PrivateRouteTable1:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-Private-RT-AZ1'

  DefaultPrivateRoute1:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway1

  PrivateSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      SubnetId: !Ref PrivateSubnet1

  PrivateRouteTable2:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-Private-RT-AZ2'

  DefaultPrivateRoute2:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref PrivateRouteTable2
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway2

  PrivateSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable2
      SubnetId: !Ref PrivateSubnet2

Outputs:
  VPC:
    Description: VPC ID
    Value: !Ref VPC
    Export:
      Name: !Sub '${EnvironmentName}-VPC-ID'

  PublicSubnet1:
    Description: Public Subnet 1 ID
    Value: !Ref PublicSubnet1
    Export:
      Name: !Sub '${EnvironmentName}-Public-Subnet-1'

  PublicSubnet2:
    Description: Public Subnet 2 ID
    Value: !Ref PublicSubnet2
    Export:
      Name: !Sub '${EnvironmentName}-Public-Subnet-2'

  PrivateSubnet1:
    Description: Private Subnet 1 ID
    Value: !Ref PrivateSubnet1
    Export:
      Name: !Sub '${EnvironmentName}-Private-Subnet-1'

  PrivateSubnet2:
    Description: Private Subnet 2 ID
    Value: !Ref PrivateSubnet2
    Export:
      Name: !Sub '${EnvironmentName}-Private-Subnet-2'

  NatGateway1EIP:
    Description: NAT Gateway 1 Elastic IP
    Value: !Ref NatGateway1EIP

  NatGateway2EIP:
    Description: NAT Gateway 2 Elastic IP
    Value: !Ref NatGateway2EIP
```

**AWS Console UI Steps to Deploy:**

1. **Go to CloudFormation Console**: https://console.aws.amazon.com/cloudformation/
2. **Click "Create stack"** → **With new resources (standard)**
3. **Prepare template**: Select **Template is ready**
4. **Template source**: Select **Upload a template file**
5. **Click "Choose file"** → Select your `vpc-basic.yaml` file
6. **Click "Next"**
7. **Stack name**: Enter `networking-lab-vpc`
8. **Parameters**:
   - **EnvironmentName**: `CFN-Lab` (default)
   - **VpcCIDR**: `10.0.0.0/16` (default)
   - Leave other parameters as default
9. **Click "Next"**
10. **Configure stack options**: Leave defaults
11. **Click "Next"**
12. **Review**: Check all settings
13. **Click "Submit"**
14. **Wait for stack creation** (Status: CREATE_IN_PROGRESS → CREATE_COMPLETE, ~5-7 minutes)

**CLI Commands**:
```bash
# Validate template syntax
aws cloudformation validate-template --template-body file://vpc-basic.yaml

# Create stack
aws cloudformation create-stack \
  --stack-name networking-lab-vpc \
  --template-body file://vpc-basic.yaml \
  --parameters \
    ParameterKey=EnvironmentName,ParameterValue=CFN-Lab \
    ParameterKey=VpcCIDR,ParameterValue=10.0.0.0/16

# Monitor stack creation
aws cloudformation describe-stacks \
  --stack-name networking-lab-vpc \
  --query 'Stacks[0].StackStatus' \
  --output text

# Watch stack events (run in separate terminal)
aws cloudformation describe-stack-events \
  --stack-name networking-lab-vpc \
  --query 'StackEvents[*].[Timestamp,ResourceStatus,ResourceType,LogicalResourceId]' \
  --output table
```

**CLI Verification**:
```bash
# 1. Check stack status
aws cloudformation describe-stacks \
  --stack-name networking-lab-vpc \
  --query 'Stacks[0].[StackName,StackStatus,CreationTime]' \
  --output table

# 2. Get stack outputs
aws cloudformation describe-stacks \
  --stack-name networking-lab-vpc \
  --query 'Stacks[0].Outputs' \
  --output table

# 3. List all resources created by the stack
aws cloudformation list-stack-resources \
  --stack-name networking-lab-vpc \
  --query 'StackResourceSummaries[*].[LogicalResourceId,ResourceType,ResourceStatus]' \
  --output table

# 4. Get VPC ID from outputs
VPC_ID=$(aws cloudformation describe-stacks \
  --stack-name networking-lab-vpc \
  --query 'Stacks[0].Outputs[?OutputKey==`VPC`].OutputValue' \
  --output text)
echo "VPC ID: $VPC_ID"

# 5. Verify VPC exists
aws ec2 describe-vpcs --vpc-ids $VPC_ID --query 'Vpcs[0].[VpcId,CidrBlock,State]' --output table

# 6. List subnets in VPC
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' \
  --output table

# 7. Verify Internet Gateway
aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC_ID" \
  --query 'InternetGateways[0].[InternetGatewayId,Attachments[0].State]' \
  --output table

# 8. Verify NAT Gateways
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID" \
  --query 'NatGateways[*].[NatGatewayId,State,SubnetId,NatGatewayAddresses[0].PublicIp]' \
  --output table

# 9. Check route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'RouteTables[*].[RouteTableId,Tags[?Key==`Name`].Value|[0],Associations[0].SubnetId]' \
  --output table
```

**Expected Output**:
```
# Stack status
CREATE_COMPLETE

# Outputs should show:
- VPC ID
- 2 Public Subnet IDs
- 2 Private Subnet IDs
- 2 NAT Gateway EIPs

# Resources created: ~20 resources
- 1 VPC
- 1 Internet Gateway
- 2 Public Subnets
- 2 Private Subnets
- 2 NAT Gateways
- 2 Elastic IPs
- 3 Route Tables
- Multiple route table associations
```

**Verification Checklist**:
- ✅ Stack status shows CREATE_COMPLETE
- ✅ All 20+ resources created successfully
- ✅ VPC has correct CIDR (10.0.0.0/16)
- ✅ 2 public subnets in different AZs
- ✅ 2 private subnets in different AZs
- ✅ 2 NAT Gateways with Elastic IPs
- ✅ Internet Gateway attached to VPC
- ✅ Route tables configured correctly

---

### Step 2: Add Security Groups to Template

**Objective**: Extend the template to include security groups for web and app tiers

**Theory**: Security groups act as virtual firewalls. CloudFormation allows you to define security group rules declaratively, including references between security groups.

**Create Enhanced Template**:

Create a file named `vpc-with-security-groups.yaml` (add to the previous template):

```yaml
# Add these resources to the previous template under Resources section

  # Security Groups
  BastionSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Sub '${EnvironmentName}-Bastion-SG'
      GroupDescription: Security group for bastion host
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 0.0.0.0/0
          Description: SSH from anywhere (restrict in production)
      SecurityGroupEgress:
        - IpProtocol: -1
          CidrIp: 0.0.0.0/0
          Description: Allow all outbound
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-Bastion-SG'

  WebServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Sub '${EnvironmentName}-WebServer-SG'
      GroupDescription: Security group for web servers
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
          Description: HTTP from anywhere
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
          Description: HTTPS from anywhere
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          SourceSecurityGroupId: !Ref BastionSecurityGroup
          Description: SSH from bastion
      SecurityGroupEgress:
        - IpProtocol: -1
          CidrIp: 0.0.0.0/0
          Description: Allow all outbound
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-WebServer-SG'

  AppServerSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Sub '${EnvironmentName}-AppServer-SG'
      GroupDescription: Security group for application servers
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 8080
          ToPort: 8080
          SourceSecurityGroupId: !Ref WebServerSecurityGroup
          Description: App port from web tier
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          SourceSecurityGroupId: !Ref BastionSecurityGroup
          Description: SSH from bastion
      SecurityGroupEgress:
        - IpProtocol: -1
          CidrIp: 0.0.0.0/0
          Description: Allow all outbound
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-AppServer-SG'

  DatabaseSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Sub '${EnvironmentName}-Database-SG'
      GroupDescription: Security group for database servers
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          SourceSecurityGroupId: !Ref AppServerSecurityGroup
          Description: MySQL from app tier
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          SourceSecurityGroupId: !Ref AppServerSecurityGroup
          Description: PostgreSQL from app tier
      SecurityGroupEgress:
        - IpProtocol: -1
          CidrIp: 0.0.0.0/0
          Description: Allow all outbound
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-Database-SG'

# Add these to Outputs section:
  BastionSecurityGroup:
    Description: Bastion Security Group ID
    Value: !Ref BastionSecurityGroup
    Export:
      Name: !Sub '${EnvironmentName}-Bastion-SG'

  WebServerSecurityGroup:
    Description: Web Server Security Group ID
    Value: !Ref WebServerSecurityGroup
    Export:
      Name: !Sub '${EnvironmentName}-WebServer-SG'

  AppServerSecurityGroup:
    Description: App Server Security Group ID
    Value: !Ref AppServerSecurityGroup
    Export:
      Name: !Sub '${EnvironmentName}-AppServer-SG'

  DatabaseSecurityGroup:
    Description: Database Security Group ID
    Value: !Ref DatabaseSecurityGroup
    Export:
      Name: !Sub '${EnvironmentName}-Database-SG'
```

**Update the Stack (AWS Console)**:

1. **CloudFormation Console** → Select `networking-lab-vpc` stack
2. **Click "Update"**
3. **Prepare template**: **Replace current template**
4. **Template source**: **Upload a template file**
5. **Choose file** → Select `vpc-with-security-groups.yaml`
6. **Click "Next"** (keep same parameters)
7. **Click "Next"** (keep same options)
8. **Click "Submit"**
9. **Wait for update** (Status: UPDATE_IN_PROGRESS → UPDATE_COMPLETE)

**CLI Commands**:
```bash
# Update stack
aws cloudformation update-stack \
  --stack-name networking-lab-vpc \
  --template-body file://vpc-with-security-groups.yaml \
  --parameters \
    ParameterKey=EnvironmentName,ParameterValue=CFN-Lab

# Monitor update
aws cloudformation describe-stacks \
  --stack-name networking-lab-vpc \
  --query 'Stacks[0].StackStatus' \
  --output text
```

**CLI Verification**:
```bash
# 1. Check update status
aws cloudformation describe-stacks \
  --stack-name networking-lab-vpc \
  --query 'Stacks[0].StackStatus' \
  --output text

# 2. List security groups created
aws cloudformation list-stack-resources \
  --stack-name networking-lab-vpc \
  --query 'StackResourceSummaries[?ResourceType==`AWS::EC2::SecurityGroup`].[LogicalResourceId,PhysicalResourceId,ResourceStatus]' \
  --output table

# 3. Get security group IDs
VPC_ID=$(aws cloudformation describe-stacks \
  --stack-name networking-lab-vpc \
  --query 'Stacks[0].Outputs[?OutputKey==`VPC`].OutputValue' \
  --output text)

aws ec2 describe-security-groups \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'SecurityGroups[*].[GroupId,GroupName,Description]' \
  --output table

# 4. View security group rules
BASTION_SG=$(aws cloudformation describe-stacks \
  --stack-name networking-lab-vpc \
  --query 'Stacks[0].Outputs[?OutputKey==`BastionSecurityGroup`].OutputValue' \
  --output text)

aws ec2 describe-security-groups \
  --group-ids $BASTION_SG \
  --query 'SecurityGroups[0].IpPermissions' \
  --output json

# 5. Verify web server security group rules
WEB_SG=$(aws cloudformation describe-stacks \
  --stack-name networking-lab-vpc \
  --query 'Stacks[0].Outputs[?OutputKey==`WebServerSecurityGroup`].OutputValue' \
  --output text)

aws ec2 describe-security-groups \
  --group-ids $WEB_SG \
  --query 'SecurityGroups[0].IpPermissions[*].[IpProtocol,FromPort,ToPort,IpRanges[0].CidrIp]' \
  --output table
```

**Expected Output**:
```
# Stack status
UPDATE_COMPLETE

# New security groups:
- Bastion-SG (SSH from 0.0.0.0/0)
- WebServer-SG (HTTP/HTTPS from 0.0.0.0/0, SSH from Bastion)
- AppServer-SG (Port 8080 from Web tier, SSH from Bastion)
- Database-SG (MySQL/PostgreSQL from App tier)
```

**Verification Checklist**:
- ✅ Stack updated successfully (UPDATE_COMPLETE)
- ✅ 4 security groups created
- ✅ Bastion SG allows SSH from anywhere
- ✅ Web SG allows HTTP/HTTPS and SSH from bastion
- ✅ App SG allows traffic only from web tier
- ✅ Database SG allows traffic only from app tier
- ✅ Security group chaining works correctly

---

### Step 3: Use Nested Stacks for Modularity

**Objective**: Create modular, reusable templates using nested stacks

**Theory**: Nested stacks allow you to break large templates into smaller, reusable components. This follows the DRY (Don't Repeat Yourself) principle and makes templates more maintainable.

**Create Network Module Template**:

Create `modules/network.yaml`:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Reusable VPC network module'

Parameters:
  EnvironmentName:
    Type: String
  VpcCIDR:
    Type: String
    Default: 10.0.0.0/16
  PublicSubnet1CIDR:
    Type: String
    Default: 10.0.1.0/24
  PublicSubnet2CIDR:
    Type: String
    Default: 10.0.2.0/24
  PrivateSubnet1CIDR:
    Type: String
    Default: 10.0.11.0/24
  PrivateSubnet2CIDR:
    Type: String
    Default: 10.0.12.0/24
  EnableNatGateway:
    Type: String
    Default: 'true'
    AllowedValues: ['true', 'false']

Conditions:
  CreateNatGateway: !Equals [!Ref EnableNatGateway, 'true']

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCIDR
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${EnvironmentName}-VPC'

  # Include all VPC resources from Step 1...
  # (InternetGateway, Subnets, NAT Gateways, Route Tables)

Outputs:
  VPCId:
    Value: !Ref VPC
  PublicSubnet1Id:
    Value: !Ref PublicSubnet1
  PublicSubnet2Id:
    Value: !Ref PublicSubnet2
  PrivateSubnet1Id:
    Value: !Ref PrivateSubnet1
  PrivateSubnet2Id:
    Value: !Ref PrivateSubnet2
```

**Create Master Template**:

Create `master-stack.yaml`:

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Master stack with nested stacks'

Parameters:
  EnvironmentName:
    Type: String
    Default: CFN-Lab

  NetworkTemplateURL:
    Type: String
    Description: S3 URL for network template

Resources:
  NetworkStack:
    Type: AWS::CloudFormation::Stack
    Properties:
      TemplateURL: !Ref NetworkTemplateURL
      Parameters:
        EnvironmentName: !Ref EnvironmentName
        VpcCIDR: 10.0.0.0/16
        PublicSubnet1CIDR: 10.0.1.0/24
        PublicSubnet2CIDR: 10.0.2.0/24
        PrivateSubnet1CIDR: 10.0.11.0/24
        PrivateSubnet2CIDR: 10.0.12.0/24
        EnableNatGateway: 'true'
      TimeoutInMinutes: 30
      Tags:
        - Key: Environment
          Value: !Ref EnvironmentName

Outputs:
  VPCId:
    Value: !GetAtt NetworkStack.Outputs.VPCId
  PublicSubnets:
    Value: !Sub '${NetworkStack.Outputs.PublicSubnet1Id},${NetworkStack.Outputs.PublicSubnet2Id}'
```

**Deploy Nested Stack**:

**Upload templates to S3:**
```bash
# Create S3 bucket for templates
aws s3 mb s3://cfn-templates-networking-lab-$(date +%s)

# Upload network module
aws s3 cp modules/network.yaml s3://your-cfn-bucket/network.yaml

# Get S3 URL
NETWORK_URL=$(aws s3 presign s3://your-cfn-bucket/network.yaml --expires-in 3600)
echo $NETWORK_URL
```

**Deploy master stack:**
```bash
aws cloudformation create-stack \
  --stack-name networking-master \
  --template-body file://master-stack.yaml \
  --parameters \
    ParameterKey=EnvironmentName,ParameterValue=CFN-Lab \
    ParameterKey=NetworkTemplateURL,ParameterValue=$NETWORK_URL
```

**CLI Verification**:
```bash
# 1. Check nested stack status
aws cloudformation describe-stacks \
  --stack-name networking-master \
  --query 'Stacks[0].StackStatus' \
  --output text

# 2. List nested stacks
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE \
  --query 'StackSummaries[?contains(StackName, `networking`)].StackName' \
  --output table

# 3. View nested stack resources
aws cloudformation describe-stack-resources \
  --stack-name networking-master \
  --query 'StackResources[?ResourceType==`AWS::CloudFormation::Stack`]' \
  --output table
```

**Verification Checklist**:
- ✅ Master stack created successfully
- ✅ Nested network stack created
- ✅ Can view nested stack independently
- ✅ Outputs from nested stack accessible in master

---

### Step 4: Implement Stack Updates and Change Sets

**Objective**: Learn to safely update stacks using change sets

**Theory**: Change sets allow you to preview changes before applying them. This prevents accidental deletions or modifications of critical resources.

**AWS Console UI Steps**:

1. **CloudFormation Console** → Select your stack
2. **Click "Stack actions"** → **"Create change set for current stack"**
3. **Prepare template**: **Replace current template**
4. **Upload modified template** (e.g., add a new subnet)
5. **Change set name**: `add-database-subnet`
6. **Click "Next"** → **"Next"** → **"Create change set"**
7. **Review changes**:
   - **Add**: Resources to be created (green +)
   - **Modify**: Resources to be updated (blue ~)
   - **Remove**: Resources to be deleted (red -)
8. **If satisfied**, click **"Execute change set"**

**CLI Commands**:
```bash
# Create change set
aws cloudformation create-change-set \
  --stack-name networking-lab-vpc \
  --template-body file://vpc-updated.yaml \
  --change-set-name add-database-subnet \
  --parameters \
    ParameterKey=EnvironmentName,ParameterValue=CFN-Lab

# View change set
aws cloudformation describe-change-set \
  --stack-name networking-lab-vpc \
  --change-set-name add-database-subnet \
  --query 'Changes[*].[Type,ResourceChange.Action,ResourceChange.LogicalResourceId,ResourceChange.ResourceType]' \
  --output table

# Execute change set
aws cloudformation execute-change-set \
  --stack-name networking-lab-vpc \
  --change-set-name add-database-subnet
```

**CLI Verification**:
```bash
# 1. List all change sets for stack
aws cloudformation list-change-sets \
  --stack-name networking-lab-vpc \
  --output table

# 2. View change set details
aws cloudformation describe-change-set \
  --stack-name networking-lab-vpc \
  --change-set-name add-database-subnet \
  --output json

# 3. Check if change set was executed
aws cloudformation describe-stacks \
  --stack-name networking-lab-vpc \
  --query 'Stacks[0].StackStatus' \
  --output text

# 4. View stack events during update
aws cloudformation describe-stack-events \
  --stack-name networking-lab-vpc \
  --max-items 10 \
  --query 'StackEvents[*].[Timestamp,ResourceStatus,LogicalResourceId]' \
  --output table
```

**Expected Output**:
```
# Change set shows:
- Actions: Add/Modify/Remove
- Resource types affected
- Replacement: True/False (important!)

# After execution:
- Stack status: UPDATE_IN_PROGRESS → UPDATE_COMPLETE
- New resources created as specified
```

**Verification Checklist**:
- ✅ Change set created successfully
- ✅ Changes previewed before execution
- ✅ No unexpected resource replacements
- ✅ Stack updated successfully
- ✅ All resources still functioning

---

### Step 5: Implement Stack Policies and Deletion Protection

**Objective**: Protect critical resources from accidental deletion

**Theory**: Stack policies define update permissions for resources. Deletion protection prevents stack deletion. These are essential for production environments.

**Create Stack Policy**:

Create `stack-policy.json`:

```json
{
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "Update:Delete",
        "Update:Replace"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ResourceType": [
            "AWS::EC2::VPC",
            "AWS::EC2::InternetGateway",
            "AWS::EC2::NatGateway"
          ]
        }
      }
    },
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "Update:*",
      "Resource": "*"
    }
  ]
}
```

**Apply Stack Policy (AWS Console)**:

1. **CloudFormation Console** → Select stack
2. **Stack actions** → **"Set stack policy"**
3. **Upload policy file** or paste JSON
4. **Click "Save"**

**Enable Termination Protection (Console)**:

1. **Select stack** → **Stack actions**
2. **"Edit termination protection"**
3. **Select "Activated"**
4. **Click "Save"**

**CLI Commands**:
```bash
# Apply stack policy
aws cloudformation set-stack-policy \
  --stack-name networking-lab-vpc \
  --stack-policy-body file://stack-policy.json

# Enable termination protection
aws cloudformation update-termination-protection \
  --stack-name networking-lab-vpc \
  --enable-termination-protection
```

**CLI Verification**:
```bash
# 1. View stack policy
aws cloudformation get-stack-policy \
  --stack-name networking-lab-vpc \
  --query 'StackPolicyBody' \
  --output text | jq .

# 2. Check termination protection status
aws cloudformation describe-stacks \
  --stack-name networking-lab-vpc \
  --query 'Stacks[0].EnableTerminationProtection' \
  --output text

# 3. Try to delete VPC (should fail)
# This will be blocked by stack policy
aws cloudformation update-stack \
  --stack-name networking-lab-vpc \
  --use-previous-template \
  --parameters ParameterKey=EnvironmentName,UsePreviousValue=true
  # Then try to remove VPC resource - should be denied

# 4. Try to delete stack (should fail)
aws cloudformation delete-stack --stack-name networking-lab-vpc
# Should return error: "Termination protection is enabled"
```

**Expected Output**:
```
# Stack policy applied
- Critical resources protected from deletion
- Other resources can still be updated

# Termination protection
- EnableTerminationProtection: true
- Delete stack command fails with error
```

**Verification Checklist**:
- ✅ Stack policy applied successfully
- ✅ VPC/IGW/NAT protected from deletion
- ✅ Termination protection enabled
- ✅ Stack cannot be deleted accidentally
- ✅ Non-protected resources can still update

---

### Step 6: Use CloudFormation Macros and Transforms

**Objective**: Use advanced features like AWS::Include and custom macros

**Theory**: Transforms allow you to use preprocessors that can modify templates before deployment. Common examples: AWS::Include, AWS::Serverless, custom macros.

**Example with AWS::Include Transform**:

Create `common-tags.yaml` and upload to S3:
```yaml
- Key: Environment
  Value: Production
- Key: ManagedBy
  Value: CloudFormation
- Key: CostCenter
  Value: Engineering
```

**Template using transform**:
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Include
Description: 'VPC with common tags'

Parameters:
  CommonTagsLocation:
    Type: String
    Default: s3://your-bucket/common-tags.yaml

Resources:
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/16
      Tags:
        Fn::Transform:
          Name: AWS::Include
          Parameters:
            Location: !Ref CommonTagsLocation
```

**CLI Verification**:
```bash
# 1. Create stack with transform
aws cloudformation create-stack \
  --stack-name vpc-with-transform \
  --template-body file://vpc-transform.yaml \
  --parameters ParameterKey=CommonTagsLocation,ParameterValue=s3://your-bucket/common-tags.yaml

# 2. View processed template (after transform)
aws cloudformation get-template \
  --stack-name vpc-with-transform \
  --template-stage Processed \
  --query 'TemplateBody' \
  --output text

# 3. Compare with original template
aws cloudformation get-template \
  --stack-name vpc-with-transform \
  --template-stage Original \
  --query 'TemplateBody' \
  --output text
```

**Verification Checklist**:
- ✅ Stack created with transform
- ✅ Processed template shows expanded tags
- ✅ Tags applied to all resources
- ✅ Transform executed before deployment

---

### Step 7: Monitor and Troubleshoot Stack Operations

**Objective**: Learn to monitor stack operations and troubleshoot failures

**AWS Console UI Steps**:

1. **CloudFormation Console** → Select stack
2. **"Events" tab**: View all resource operations
3. **"Resources" tab**: View all created resources
4. **"Outputs" tab**: View stack outputs
5. **"Parameters" tab**: View input parameters
6. **"Template" tab**: View template used

**Set up CloudWatch Alarms**:

1. **CloudWatch Console** → **Alarms** → **Create alarm**
2. **Select metric**: CloudFormation → Stack Events
3. **Conditions**: When stack status = CREATE_FAILED
4. **Actions**: Send SNS notification

**CLI Monitoring**:
```bash
# 1. Watch stack creation in real-time
aws cloudformation wait stack-create-complete --stack-name networking-lab-vpc

# 2. Get stack events (last 50)
aws cloudformation describe-stack-events \
  --stack-name networking-lab-vpc \
  --max-items 50 \
  --query 'StackEvents[*].[Timestamp,ResourceStatus,ResourceType,LogicalResourceId,ResourceStatusReason]' \
  --output table

# 3. Filter failed events only
aws cloudformation describe-stack-events \
  --stack-name networking-lab-vpc \
  --query 'StackEvents[?ResourceStatus==`CREATE_FAILED`].[Timestamp,LogicalResourceId,ResourceStatusReason]' \
  --output table

# 4. Get drift detection results
aws cloudformation detect-stack-drift --stack-name networking-lab-vpc

DRIFT_ID=$(aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id <drift-id> \
  --query 'StackDriftDetectionId' \
  --output text)

aws cloudformation describe-stack-resource-drifts \
  --stack-name networking-lab-vpc \
  --query 'StackResourceDrifts[?StackResourceDriftStatus==`MODIFIED`]' \
  --output table

# 5. View stack template
aws cloudformation get-template \
  --stack-name networking-lab-vpc \
  --query 'TemplateBody' \
  --output text > deployed-template.yaml

# 6. Validate current template
aws cloudformation validate-template \
  --template-body file://deployed-template.yaml
```

**CLI Verification**:
```bash
# 1. Check overall stack health
aws cloudformation describe-stacks \
  --stack-name networking-lab-vpc \
  --query 'Stacks[0].[StackName,StackStatus,StackStatusReason]' \
  --output table

# 2. Count resources by status
aws cloudformation list-stack-resources \
  --stack-name networking-lab-vpc \
  --query 'StackResourceSummaries[*].ResourceStatus' \
  --output text | sort | uniq -c

# 3. Find resources in CREATE_FAILED state
aws cloudformation list-stack-resources \
  --stack-name networking-lab-vpc \
  --query 'StackResourceSummaries[?ResourceStatus==`CREATE_FAILED`].[LogicalResourceId,ResourceType,ResourceStatusReason]' \
  --output table

# 4. Check stack drift status
aws cloudformation detect-stack-drift \
  --stack-name networking-lab-vpc \
  --output text

# Wait for drift detection to complete
sleep 10

# Get drift results
aws cloudformation describe-stack-drift-detection-status \
  --stack-drift-detection-id <id-from-previous-command>
```

**Expected Output**:
```
# Healthy stack shows:
- StackStatus: CREATE_COMPLETE or UPDATE_COMPLETE
- All resources: CREATE_COMPLETE
- No drift detected (or IN_SYNC)

# Failed stack shows:
- StackStatus: CREATE_FAILED or ROLLBACK_COMPLETE
- Failed resources with reasons
- Drift: MODIFIED or DELETED (if manual changes)
```

**Common Failure Scenarios**:

| Error | Cause | Solution |
|-------|-------|----------|
| **CIDR overlap** | Subnet CIDR conflicts | Fix CIDR blocks in template |
| **Resource limit exceeded** | Too many EIPs/VPCs | Request limit increase |
| **Insufficient permissions** | IAM role lacks permissions | Add required IAM permissions |
| **Dependency violation** | Wrong resource order | Use DependsOn attribute |
| **Resource already exists** | Resource name conflict | Use unique names or Ref/GetAtt |

**Verification Checklist**:
- ✅ Can view stack events and status
- ✅ Can identify failed resources
- ✅ Can detect stack drift
- ✅ Can troubleshoot common errors
- ✅ Can roll back failed stacks

---

### Cleanup

**IMPORTANT**: Delete stacks in reverse order of dependencies!

**AWS Console UI Steps**:

1. **Disable termination protection**:
   - Select stack → **Stack actions** → **Edit termination protection**
   - Select **"Deactivated"** → **Save**

2. **Delete stack**:
   - Select stack → **Delete**
   - **Confirm deletion**
   - Wait for DELETE_IN_PROGRESS → DELETE_COMPLETE (~5-10 minutes)

3. **Delete nested stacks** (if any):
   - Nested stacks auto-delete with parent
   - Or manually delete child stacks first

4. **Delete S3 buckets** (if created):
   - Empty bucket first
   - Then delete bucket

**CLI Cleanup**:
```bash
# 1. Disable termination protection
aws cloudformation update-termination-protection \
  --stack-name networking-lab-vpc \
  --no-enable-termination-protection

# 2. Delete stack
aws cloudformation delete-stack --stack-name networking-lab-vpc

# 3. Wait for deletion to complete
aws cloudformation wait stack-delete-complete --stack-name networking-lab-vpc

# 4. Verify stack is deleted
aws cloudformation describe-stacks --stack-name networking-lab-vpc 2>&1
# Should return: Stack with id networking-lab-vpc does not exist

# 5. Clean up S3 bucket (if created)
aws s3 rm s3://your-cfn-bucket --recursive
aws s3 rb s3://your-cfn-bucket
```

**CLI Verification**:
```bash
# 1. Check stack deletion status
aws cloudformation list-stacks \
  --stack-status-filter DELETE_IN_PROGRESS DELETE_COMPLETE \
  --query 'StackSummaries[?StackName==`networking-lab-vpc`].[StackName,StackStatus,DeletionTime]' \
  --output table

# 2. Verify no resources remain
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=tag:Name,Values=CFN-Lab-VPC" \
  --query 'Vpcs[0].VpcId' \
  --output text 2>/dev/null)

if [ "$VPC_ID" != "None" ] && [ -n "$VPC_ID" ]; then
  echo "WARNING: VPC still exists: $VPC_ID"
else
  echo "SUCCESS: VPC deleted"
fi

# 3. Check for orphaned resources
aws ec2 describe-nat-gateways \
  --filter "Name=tag:Name,Values=CFN-Lab*" \
  --query 'NatGateways[?State!=`deleted`].[NatGatewayId,State]' \
  --output table

# 4. List any remaining stacks
aws cloudformation list-stacks \
  --stack-status-filter CREATE_COMPLETE UPDATE_COMPLETE \
  --query 'StackSummaries[*].StackName' \
  --output table
```

**Expected Output**:
```
# Stack deleted
StackStatus: DELETE_COMPLETE

# No VPC found
SUCCESS: VPC deleted

# No orphaned resources
(empty results)
```

**Verification Checklist**:
- ✅ Stack deleted successfully
- ✅ All VPC resources removed
- ✅ NAT Gateways deleted (can take 5-10 min)
- ✅ Elastic IPs released
- ✅ S3 buckets deleted (if created)
- ✅ No charges in billing after 24 hours

---

### Key Takeaways

- **Infrastructure as Code**: CloudFormation enables version-controlled, repeatable infrastructure deployments
- **Declarative Templates**: Define desired state; CloudFormation handles resource creation order
- **Parameters & Outputs**: Make templates reusable and composable across environments
- **Stack Updates**: Change sets provide safe preview before applying changes
- **Stack Policies**: Protect critical resources from accidental deletion or replacement
- **Nested Stacks**: Break large templates into modular, reusable components
- **Drift Detection**: Identify manual changes that deviate from template
- **Rollback on Failure**: Automatic rollback ensures consistent state
- **Cross-Stack References**: Export/Import enables sharing values between stacks
- **Best Practices**:
  - Use parameters for values that change between environments
  - Always use change sets for production updates
  - Tag all resources for cost tracking
  - Version control templates in Git
  - Use nested stacks for complex architectures
  - Enable termination protection for production stacks

---

### Additional Resources

- [AWS CloudFormation User Guide](https://docs.aws.amazon.com/cloudformation/)
- [CloudFormation Template Reference](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/template-reference.html)
- [CloudFormation Best Practices](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html)
- [CloudFormation Sample Templates](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/cfn-sample-templates.html)
- [AWS CloudFormation Registry](https://docs.aws.amazon.com/cloudformation-cli/)
- [CloudFormation Linter (cfn-lint)](https://github.com/aws-cloudformation/cfn-lint)
- [TaskCat - CloudFormation Testing Tool](https://github.com/aws-quickstart/taskcat)

---

### Exam Tips

**For AWS Networking Specialty Exam**:

- **Stack vs StackSet**: Stack deploys to one account/region; StackSet deploys to multiple accounts/regions
- **DeletionPolicy**: Retain, Delete, Snapshot - controls what happens to resources when stack is deleted
- **DependsOn**: Explicit dependencies when CloudFormation can't infer resource order
- **CreationPolicy**: Wait for resource signals before marking as CREATE_COMPLETE
- **UpdatePolicy**: Controls how resources are updated (e.g., AutoScalingRollingUpdate)
- **Conditions**: Create resources conditionally based on parameter values
- **Mappings**: Key-value lookup tables (e.g., AMI IDs by region)
- **Intrinsic Functions**:
  - `!Ref`: Reference parameter or resource
  - `!GetAtt`: Get attribute from resource
  - `!Sub`: String substitution with variables
  - `!Join`: Concatenate strings
  - `!Select`: Select item from list
  - `!GetAZs`: Get availability zones in region
- **Pseudo Parameters**: AWS::Region, AWS::AccountId, AWS::StackName, etc.
- **Stack Drift**: Manual changes cause drift; use drift detection to identify
- **Change Sets**: Preview changes before applying; required for production best practice
- **Nested Stacks**: Use for modular templates; parent stack manages lifecycle
- **Cross-Stack References**: Export from one stack, ImportValue in another
- **CloudFormation Registry**: Custom resource types and modules
- **Helper Scripts**: cfn-init, cfn-signal, cfn-hup for EC2 instance configuration

**Common Exam Scenarios**:
- **Multi-region VPC deployment** → Use StackSets
- **Protect production resources** → Stack policy + termination protection
- **Conditional resource creation** → Use Conditions section
- **Reference resources across stacks** → Export/Import values
- **Wait for application to start** → CreationPolicy with cfn-signal
- **Update without downtime** → UpdatePolicy with AutoScalingRollingUpdate
- **Deploy to multiple AWS accounts** → CloudFormation StackSets
- **Custom resource logic** → Lambda-backed custom resources
- **Template too large** → Use nested stacks or AWS::Include transform
- **Manual changes detected** → Run drift detection and remediate
