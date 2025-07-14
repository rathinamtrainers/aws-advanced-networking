# Topic 32: VPC Automation and Infrastructure as Code

## Prerequisites
- Completed Topic 31 (VPC Troubleshooting and Advanced Diagnostics)
- Understanding of Infrastructure as Code principles and DevOps practices
- Knowledge of AWS CloudFormation, Terraform, or AWS CDK
- Familiarity with CI/CD pipelines and automation tools

## Learning Objectives
By the end of this topic, you will be able to:
- Design and implement automated VPC provisioning using Infrastructure as Code
- Create reusable VPC templates and modules for standardized deployments
- Implement automated VPC lifecycle management and compliance checking
- Build CI/CD pipelines for network infrastructure deployment and updates

## Theory

### Infrastructure as Code for VPC

#### IaC Benefits for Network Infrastructure
- **Consistency**: Standardized deployments across environments
- **Version Control**: Track changes and rollback capabilities
- **Repeatability**: Reliable recreation of network environments
- **Compliance**: Automated policy enforcement and security standards
- **Documentation**: Self-documenting infrastructure configurations

#### IaC Tools Comparison
```
Tool Comparison Matrix:
┌─────────────────┬─────────────────┬─────────────────┬─────────────────┐
│ Tool            │ Language        │ State Mgmt      │ AWS Integration │
├─────────────────┼─────────────────┼─────────────────┼─────────────────┤
│ CloudFormation  │ JSON/YAML       │ AWS Managed     │ Native          │
│ Terraform       │ HCL             │ State Files     │ Provider-based  │
│ AWS CDK         │ Multiple Lang   │ CloudFormation  │ Native          │
│ Pulumi          │ Multiple Lang   │ State Service   │ Provider-based  │
└─────────────────┴─────────────────┴─────────────────┴─────────────────┘
```

### VPC Template Design Patterns

#### Pattern 1: Modular VPC Architecture
```
VPC Module Structure:
├── vpc-core/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── vpc-subnets/
│   ├── public-subnets.tf
│   ├── private-subnets.tf
│   └── database-subnets.tf
├── vpc-gateways/
│   ├── internet-gateway.tf
│   ├── nat-gateways.tf
│   └── vpn-gateway.tf
└── vpc-routing/
    ├── route-tables.tf
    ├── route-associations.tf
    └── route-propagation.tf
```

#### Pattern 2: Environment-Specific Configurations
```
Environment Layout:
├── environments/
│   ├── dev/
│   │   ├── terraform.tfvars
│   │   └── main.tf
│   ├── staging/
│   │   ├── terraform.tfvars
│   │   └── main.tf
│   └── prod/
│       ├── terraform.tfvars
│       └── main.tf
└── modules/
    └── vpc/
        ├── main.tf
        ├── variables.tf
        └── outputs.tf
```

#### Pattern 3: Multi-Account Network Architecture
```
Organization Structure:
├── shared-network-account/
│   ├── transit-gateway.tf
│   ├── shared-services-vpc.tf
│   └── cross-account-sharing.tf
├── production-account/
│   ├── prod-vpc.tf
│   └── tgw-attachments.tf
└── development-account/
    ├── dev-vpc.tf
    └── tgw-attachments.tf
```

### Automated VPC Provisioning

#### CloudFormation VPC Template
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Enterprise VPC with automated configuration'

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]
    Description: Environment name for resource tagging
  
  VpcCidr:
    Type: String
    Default: '10.0.0.0/16'
    AllowedPattern: '^(([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.){3}([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])(\/(1[6-9]|2[0-8]))$'
    Description: CIDR block for VPC
  
  EnableNatGateway:
    Type: String
    Default: 'true'
    AllowedValues: ['true', 'false']
    Description: Create NAT Gateway for private subnets
  
  EnableVpnGateway:
    Type: String
    Default: 'false'
    AllowedValues: ['true', 'false']
    Description: Create VPN Gateway for hybrid connectivity

Conditions:
  CreateNatGateway: !Equals [!Ref EnableNatGateway, 'true']
  CreateVpnGateway: !Equals [!Ref EnableVpnGateway, 'true']
  IsProduction: !Equals [!Ref Environment, 'prod']

Resources:
  # VPC
  VPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: !Ref VpcCidr
      EnableDnsHostnames: true
      EnableDnsSupport: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-vpc'
        - Key: Environment
          Value: !Ref Environment
        - Key: ManagedBy
          Value: CloudFormation

  # Internet Gateway
  InternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-igw'
        - Key: Environment
          Value: !Ref Environment

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
      CidrBlock: !Select [0, !Cidr [!Ref VpcCidr, 6, 8]]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-subnet-1'
        - Key: Type
          Value: Public
        - Key: Environment
          Value: !Ref Environment

  PublicSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !Select [1, !Cidr [!Ref VpcCidr, 6, 8]]
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-subnet-2'
        - Key: Type
          Value: Public
        - Key: Environment
          Value: !Ref Environment

  # Private Subnets
  PrivateSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !Select [2, !Cidr [!Ref VpcCidr, 6, 8]]
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-subnet-1'
        - Key: Type
          Value: Private
        - Key: Environment
          Value: !Ref Environment

  PrivateSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !Select [3, !Cidr [!Ref VpcCidr, 6, 8]]
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-subnet-2'
        - Key: Type
          Value: Private
        - Key: Environment
          Value: !Ref Environment

  # Database Subnets
  DatabaseSubnet1:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [0, !GetAZs '']
      CidrBlock: !Select [4, !Cidr [!Ref VpcCidr, 6, 8]]
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-database-subnet-1'
        - Key: Type
          Value: Database
        - Key: Environment
          Value: !Ref Environment

  DatabaseSubnet2:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref VPC
      AvailabilityZone: !Select [1, !GetAZs '']
      CidrBlock: !Select [5, !Cidr [!Ref VpcCidr, 6, 8]]
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-database-subnet-2'
        - Key: Type
          Value: Database
        - Key: Environment
          Value: !Ref Environment

  # NAT Gateways
  NatGateway1EIP:
    Type: AWS::EC2::EIP
    Condition: CreateNatGateway
    DependsOn: InternetGatewayAttachment
    Properties:
      Domain: vpc
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-nat-gateway-1-eip'

  NatGateway2EIP:
    Type: AWS::EC2::EIP
    Condition: CreateNatGateway
    DependsOn: InternetGatewayAttachment
    Properties:
      Domain: vpc
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-nat-gateway-2-eip'

  NatGateway1:
    Type: AWS::EC2::NatGateway
    Condition: CreateNatGateway
    Properties:
      AllocationId: !GetAtt NatGateway1EIP.AllocationId
      SubnetId: !Ref PublicSubnet1
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-nat-gateway-1'

  NatGateway2:
    Type: AWS::EC2::NatGateway
    Condition: CreateNatGateway
    Properties:
      AllocationId: !GetAtt NatGateway2EIP.AllocationId
      SubnetId: !Ref PublicSubnet2
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-nat-gateway-2'

  # VPN Gateway
  VpnGateway:
    Type: AWS::EC2::VpnGateway
    Condition: CreateVpnGateway
    Properties:
      Type: ipsec.1
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-vpn-gateway'

  VpnGatewayAttachment:
    Type: AWS::EC2::VPCGatewayAttachment
    Condition: CreateVpnGateway
    Properties:
      VpcId: !Ref VPC
      VpnGatewayId: !Ref VpnGateway

  # Route Tables
  PublicRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-public-route-table'
        - Key: Type
          Value: Public

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

  PrivateRouteTable1:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-private-route-table-1'
        - Key: Type
          Value: Private

  DefaultPrivateRoute1:
    Type: AWS::EC2::Route
    Condition: CreateNatGateway
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
          Value: !Sub '${Environment}-private-route-table-2'
        - Key: Type
          Value: Private

  DefaultPrivateRoute2:
    Type: AWS::EC2::Route
    Condition: CreateNatGateway
    Properties:
      RouteTableId: !Ref PrivateRouteTable2
      DestinationCidrBlock: 0.0.0.0/0
      NatGatewayId: !Ref NatGateway2

  PrivateSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref PrivateRouteTable2
      SubnetId: !Ref PrivateSubnet2

  DatabaseRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId: !Ref VPC
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-database-route-table'
        - Key: Type
          Value: Database

  DatabaseSubnet1RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref DatabaseRouteTable
      SubnetId: !Ref DatabaseSubnet1

  DatabaseSubnet2RouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      RouteTableId: !Ref DatabaseRouteTable
      SubnetId: !Ref DatabaseSubnet2

  # VPC Route Propagation for VPN
  PrivateRouteTable1VpnPropagation:
    Type: AWS::EC2::VPNGatewayRoutePropagation
    Condition: CreateVpnGateway
    DependsOn: VpnGatewayAttachment
    Properties:
      RouteTableId: !Ref PrivateRouteTable1
      VpnGatewayId: !Ref VpnGateway

  PrivateRouteTable2VpnPropagation:
    Type: AWS::EC2::VPNGatewayRoutePropagation
    Condition: CreateVpnGateway
    DependsOn: VpnGatewayAttachment
    Properties:
      RouteTableId: !Ref PrivateRouteTable2
      VpnGatewayId: !Ref VpnGateway

  # Security Groups
  DefaultSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Sub '${Environment}-default-sg'
      GroupDescription: Default security group for VPC
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: -1
          SourceSecurityGroupId: !Ref DefaultSecurityGroup
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-default-sg'
        - Key: Environment
          Value: !Ref Environment

  WebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Sub '${Environment}-web-sg'
      GroupDescription: Security group for web servers
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 10.0.0.0/16
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-web-sg'
        - Key: Environment
          Value: !Ref Environment

  AppSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Sub '${Environment}-app-sg'
      GroupDescription: Security group for application servers
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 8080
          ToPort: 8080
          SourceSecurityGroupId: !Ref WebSecurityGroup
        - IpProtocol: tcp
          FromPort: 22
          ToPort: 22
          CidrIp: 10.0.0.0/16
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-app-sg'
        - Key: Environment
          Value: !Ref Environment

  DatabaseSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupName: !Sub '${Environment}-db-sg'
      GroupDescription: Security group for database servers
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 3306
          ToPort: 3306
          SourceSecurityGroupId: !Ref AppSecurityGroup
        - IpProtocol: tcp
          FromPort: 5432
          ToPort: 5432
          SourceSecurityGroupId: !Ref AppSecurityGroup
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-db-sg'
        - Key: Environment
          Value: !Ref Environment

  # VPC Flow Logs
  VpcFlowLogRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: vpc-flow-logs.amazonaws.com
            Action: sts:AssumeRole
      Policies:
        - PolicyName: flowlogsDeliveryRolePolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - logs:CreateLogGroup
                  - logs:CreateLogStream
                  - logs:PutLogEvents
                  - logs:DescribeLogGroups
                  - logs:DescribeLogStreams
                Resource: !Sub 'arn:aws:logs:${AWS::Region}:${AWS::AccountId}:*'

  VpcFlowLogGroup:
    Type: AWS::Logs::LogGroup
    Properties:
      LogGroupName: !Sub '/aws/vpc/flowlogs/${Environment}'
      RetentionInDays: !If [IsProduction, 90, 30]

  VpcFlowLog:
    Type: AWS::EC2::FlowLog
    Properties:
      ResourceType: VPC
      ResourceId: !Ref VPC
      TrafficType: ALL
      LogDestinationType: cloud-watch-logs
      LogDestination: !GetAtt VpcFlowLogGroup.Arn
      DeliverLogsPermissionArn: !GetAtt VpcFlowLogRole.Arn
      Tags:
        - Key: Name
          Value: !Sub '${Environment}-vpc-flow-log'
        - Key: Environment
          Value: !Ref Environment

Outputs:
  VpcId:
    Description: ID of the VPC
    Value: !Ref VPC
    Export:
      Name: !Sub '${Environment}-VpcId'

  VpcCidr:
    Description: CIDR block of the VPC
    Value: !Ref VpcCidr
    Export:
      Name: !Sub '${Environment}-VpcCidr'

  PublicSubnets:
    Description: List of public subnet IDs
    Value: !Join
      - ','
      - - !Ref PublicSubnet1
        - !Ref PublicSubnet2
    Export:
      Name: !Sub '${Environment}-PublicSubnets'

  PrivateSubnets:
    Description: List of private subnet IDs
    Value: !Join
      - ','
      - - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
    Export:
      Name: !Sub '${Environment}-PrivateSubnets'

  DatabaseSubnets:
    Description: List of database subnet IDs
    Value: !Join
      - ','
      - - !Ref DatabaseSubnet1
        - !Ref DatabaseSubnet2
    Export:
      Name: !Sub '${Environment}-DatabaseSubnets'

  WebSecurityGroupId:
    Description: Security group ID for web servers
    Value: !Ref WebSecurityGroup
    Export:
      Name: !Sub '${Environment}-WebSecurityGroup'

  AppSecurityGroupId:
    Description: Security group ID for application servers
    Value: !Ref AppSecurityGroup
    Export:
      Name: !Sub '${Environment}-AppSecurityGroup'

  DatabaseSecurityGroupId:
    Description: Security group ID for database servers
    Value: !Ref DatabaseSecurityGroup
    Export:
      Name: !Sub '${Environment}-DatabaseSecurityGroup'

  InternetGatewayId:
    Description: ID of the Internet Gateway
    Value: !Ref InternetGateway
    Export:
      Name: !Sub '${Environment}-InternetGateway'

  NatGateway1Id:
    Condition: CreateNatGateway
    Description: ID of NAT Gateway 1
    Value: !Ref NatGateway1
    Export:
      Name: !Sub '${Environment}-NatGateway1'

  NatGateway2Id:
    Condition: CreateNatGateway
    Description: ID of NAT Gateway 2
    Value: !Ref NatGateway2
    Export:
      Name: !Sub '${Environment}-NatGateway2'

  VpnGatewayId:
    Condition: CreateVpnGateway
    Description: ID of the VPN Gateway
    Value: !Ref VpnGateway
    Export:
      Name: !Sub '${Environment}-VpnGateway'
```

## Lab Exercise: Comprehensive VPC Automation Implementation

### Lab Duration: 360 minutes

### Step 1: CloudFormation VPC Deployment
**Objective**: Deploy enterprise VPC using CloudFormation with automated configuration

**Deploy Development Environment**:
```bash
# Create CloudFormation stack for development environment
aws cloudformation create-stack \
  --stack-name dev-vpc-stack \
  --template-body file://vpc-template.yaml \
  --parameters ParameterKey=Environment,ParameterValue=dev \
               ParameterKey=VpcCidr,ParameterValue=10.0.0.0/16 \
               ParameterKey=EnableNatGateway,ParameterValue=true \
               ParameterKey=EnableVpnGateway,ParameterValue=false \
  --capabilities CAPABILITY_IAM \
  --tags Key=Project,Value=VpcAutomation \
         Key=Owner,Value=NetworkTeam

# Monitor stack creation
aws cloudformation describe-stacks \
  --stack-name dev-vpc-stack \
  --query 'Stacks[0].StackStatus'

# Wait for stack completion
aws cloudformation wait stack-create-complete \
  --stack-name dev-vpc-stack
```

**Deploy Production Environment**:
```bash
# Create CloudFormation stack for production environment
aws cloudformation create-stack \
  --stack-name prod-vpc-stack \
  --template-body file://vpc-template.yaml \
  --parameters ParameterKey=Environment,ParameterValue=prod \
               ParameterKey=VpcCidr,ParameterValue=10.1.0.0/16 \
               ParameterKey=EnableNatGateway,ParameterValue=true \
               ParameterKey=EnableVpnGateway,ParameterValue=true \
  --capabilities CAPABILITY_IAM \
  --tags Key=Project,Value=VpcAutomation \
         Key=Owner,Value=NetworkTeam \
         Key=Environment,Value=production
```

### Step 2: Terraform VPC Module Development
**Objective**: Create reusable Terraform modules for VPC automation

**Create Terraform VPC Module**:
```hcl
# modules/vpc/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Data sources
data "aws_availability_zones" "available" {
  state = "available"
}

data "aws_caller_identity" "current" {}

# Local values
locals {
  azs_count = 2
  azs       = slice(data.aws_availability_zones.available.names, 0, local.azs_count)
  
  common_tags = {
    Project     = var.project_name
    Environment = var.environment
    ManagedBy   = "Terraform"
    Owner       = var.owner
  }
}

# VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_hostnames = var.enable_dns_hostnames
  enable_dns_support   = var.enable_dns_support

  tags = merge(local.common_tags, {
    Name = "${var.environment}-vpc"
    Type = "VPC"
  })
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  count = var.create_igw ? 1 : 0
  
  vpc_id = aws_vpc.main.id

  tags = merge(local.common_tags, {
    Name = "${var.environment}-igw"
    Type = "InternetGateway"
  })
}

# Public Subnets
resource "aws_subnet" "public" {
  count = var.enable_public_subnets ? local.azs_count : 0

  vpc_id                  = aws_vpc.main.id
  cidr_block              = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone       = local.azs[count.index]
  map_public_ip_on_launch = true

  tags = merge(local.common_tags, {
    Name = "${var.environment}-public-subnet-${count.index + 1}"
    Type = "Public"
    AZ   = local.azs[count.index]
  })
}

# Private Subnets
resource "aws_subnet" "private" {
  count = var.enable_private_subnets ? local.azs_count : 0

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 10)
  availability_zone = local.azs[count.index]

  tags = merge(local.common_tags, {
    Name = "${var.environment}-private-subnet-${count.index + 1}"
    Type = "Private"
    AZ   = local.azs[count.index]
  })
}

# Database Subnets
resource "aws_subnet" "database" {
  count = var.enable_database_subnets ? local.azs_count : 0

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index + 20)
  availability_zone = local.azs[count.index]

  tags = merge(local.common_tags, {
    Name = "${var.environment}-database-subnet-${count.index + 1}"
    Type = "Database"
    AZ   = local.azs[count.index]
  })
}

# Elastic IPs for NAT Gateways
resource "aws_eip" "nat" {
  count = var.enable_nat_gateway ? local.azs_count : 0

  domain = "vpc"
  depends_on = [aws_internet_gateway.main]

  tags = merge(local.common_tags, {
    Name = "${var.environment}-nat-eip-${count.index + 1}"
    Type = "NATGatewayEIP"
  })
}

# NAT Gateways
resource "aws_nat_gateway" "main" {
  count = var.enable_nat_gateway ? local.azs_count : 0

  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id
  depends_on    = [aws_internet_gateway.main]

  tags = merge(local.common_tags, {
    Name = "${var.environment}-nat-gateway-${count.index + 1}"
    Type = "NATGateway"
    AZ   = local.azs[count.index]
  })
}

# VPN Gateway
resource "aws_vpn_gateway" "main" {
  count = var.enable_vpn_gateway ? 1 : 0

  vpc_id = aws_vpc.main.id
  
  tags = merge(local.common_tags, {
    Name = "${var.environment}-vpn-gateway"
    Type = "VPNGateway"
  })
}

# Route Tables - Public
resource "aws_route_table" "public" {
  count = var.enable_public_subnets ? 1 : 0

  vpc_id = aws_vpc.main.id

  tags = merge(local.common_tags, {
    Name = "${var.environment}-public-rt"
    Type = "PublicRouteTable"
  })
}

# Route Tables - Private
resource "aws_route_table" "private" {
  count = var.enable_private_subnets ? local.azs_count : 0

  vpc_id = aws_vpc.main.id

  tags = merge(local.common_tags, {
    Name = "${var.environment}-private-rt-${count.index + 1}"
    Type = "PrivateRouteTable"
    AZ   = local.azs[count.index]
  })
}

# Route Tables - Database
resource "aws_route_table" "database" {
  count = var.enable_database_subnets ? 1 : 0

  vpc_id = aws_vpc.main.id

  tags = merge(local.common_tags, {
    Name = "${var.environment}-database-rt"
    Type = "DatabaseRouteTable"
  })
}

# Routes - Public to Internet Gateway
resource "aws_route" "public_internet_gateway" {
  count = var.enable_public_subnets && var.create_igw ? 1 : 0

  route_table_id         = aws_route_table.public[0].id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.main[0].id

  timeouts {
    create = "5m"
  }
}

# Routes - Private to NAT Gateway
resource "aws_route" "private_nat_gateway" {
  count = var.enable_private_subnets && var.enable_nat_gateway ? local.azs_count : 0

  route_table_id         = aws_route_table.private[count.index].id
  destination_cidr_block = "0.0.0.0/0"
  nat_gateway_id         = aws_nat_gateway.main[count.index].id

  timeouts {
    create = "5m"
  }
}

# Route Table Associations - Public
resource "aws_route_table_association" "public" {
  count = var.enable_public_subnets ? local.azs_count : 0

  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public[0].id
}

# Route Table Associations - Private
resource "aws_route_table_association" "private" {
  count = var.enable_private_subnets ? local.azs_count : 0

  subnet_id      = aws_subnet.private[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# Route Table Associations - Database
resource "aws_route_table_association" "database" {
  count = var.enable_database_subnets ? local.azs_count : 0

  subnet_id      = aws_subnet.database[count.index].id
  route_table_id = aws_route_table.database[0].id
}

# VPN Gateway Route Propagation
resource "aws_vpn_gateway_route_propagation" "private" {
  count = var.enable_vpn_gateway && var.enable_private_subnets ? local.azs_count : 0

  vpn_gateway_id = aws_vpn_gateway.main[0].id
  route_table_id = aws_route_table.private[count.index].id
}

resource "aws_vpn_gateway_route_propagation" "database" {
  count = var.enable_vpn_gateway && var.enable_database_subnets ? 1 : 0

  vpn_gateway_id = aws_vpn_gateway.main[0].id
  route_table_id = aws_route_table.database[0].id
}

# Security Groups
resource "aws_security_group" "default" {
  name        = "${var.environment}-default-sg"
  description = "Default security group for ${var.environment} VPC"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
    self      = true
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.common_tags, {
    Name = "${var.environment}-default-sg"
    Type = "DefaultSecurityGroup"
  })
}

resource "aws_security_group" "web" {
  count = var.create_web_sg ? 1 : 0

  name        = "${var.environment}-web-sg"
  description = "Security group for web servers in ${var.environment}"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.common_tags, {
    Name = "${var.environment}-web-sg"
    Type = "WebSecurityGroup"
  })
}

resource "aws_security_group" "app" {
  count = var.create_app_sg ? 1 : 0

  name        = "${var.environment}-app-sg"
  description = "Security group for application servers in ${var.environment}"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = var.create_web_sg ? [aws_security_group.web[0].id] : []
  }

  ingress {
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = [var.vpc_cidr]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.common_tags, {
    Name = "${var.environment}-app-sg"
    Type = "AppSecurityGroup"
  })
}

resource "aws_security_group" "database" {
  count = var.create_db_sg ? 1 : 0

  name        = "${var.environment}-db-sg"
  description = "Security group for database servers in ${var.environment}"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = var.create_app_sg ? [aws_security_group.app[0].id] : []
  }

  ingress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = var.create_app_sg ? [aws_security_group.app[0].id] : []
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(local.common_tags, {
    Name = "${var.environment}-db-sg"
    Type = "DatabaseSecurityGroup"
  })
}

# VPC Flow Logs
resource "aws_cloudwatch_log_group" "vpc_flow_log" {
  count = var.enable_flow_logs ? 1 : 0

  name              = "/aws/vpc/flowlogs/${var.environment}"
  retention_in_days = var.flow_log_retention_days

  tags = merge(local.common_tags, {
    Name = "${var.environment}-vpc-flow-logs"
    Type = "VPCFlowLogs"
  })
}

resource "aws_iam_role" "flow_log" {
  count = var.enable_flow_logs ? 1 : 0

  name = "${var.environment}-vpc-flow-log-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "vpc-flow-logs.amazonaws.com"
        }
      }
    ]
  })

  tags = local.common_tags
}

resource "aws_iam_role_policy" "flow_log" {
  count = var.enable_flow_logs ? 1 : 0

  name = "${var.environment}-vpc-flow-log-policy"
  role = aws_iam_role.flow_log[0].id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents",
          "logs:DescribeLogGroups",
          "logs:DescribeLogStreams"
        ]
        Effect = "Allow"
        Resource = "arn:aws:logs:*:${data.aws_caller_identity.current.account_id}:*"
      }
    ]
  })
}

resource "aws_flow_log" "vpc" {
  count = var.enable_flow_logs ? 1 : 0

  iam_role_arn    = aws_iam_role.flow_log[0].arn
  log_destination = aws_cloudwatch_log_group.vpc_flow_log[0].arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.main.id

  tags = merge(local.common_tags, {
    Name = "${var.environment}-vpc-flow-log"
    Type = "VPCFlowLog"
  })
}
```

**Create Terraform Variables**:
```hcl
# modules/vpc/variables.tf
variable "environment" {
  description = "Environment name"
  type        = string
  
  validation {
    condition = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}

variable "project_name" {
  description = "Name of the project"
  type        = string
  default     = "VPCAutomation"
}

variable "owner" {
  description = "Owner of the resources"
  type        = string
  default     = "NetworkTeam"
}

variable "vpc_cidr" {
  description = "CIDR block for VPC"
  type        = string
  default     = "10.0.0.0/16"
  
  validation {
    condition = can(cidrhost(var.vpc_cidr, 0))
    error_message = "VPC CIDR must be a valid IPv4 CIDR block."
  }
}

variable "enable_dns_hostnames" {
  description = "Enable DNS hostnames in the VPC"
  type        = bool
  default     = true
}

variable "enable_dns_support" {
  description = "Enable DNS support in the VPC"
  type        = bool
  default     = true
}

variable "create_igw" {
  description = "Create Internet Gateway"
  type        = bool
  default     = true
}

variable "enable_public_subnets" {
  description = "Enable creation of public subnets"
  type        = bool
  default     = true
}

variable "enable_private_subnets" {
  description = "Enable creation of private subnets"
  type        = bool
  default     = true
}

variable "enable_database_subnets" {
  description = "Enable creation of database subnets"
  type        = bool
  default     = true
}

variable "enable_nat_gateway" {
  description = "Enable NAT Gateways for private subnets"
  type        = bool
  default     = true
}

variable "enable_vpn_gateway" {
  description = "Enable VPN Gateway"
  type        = bool
  default     = false
}

variable "create_web_sg" {
  description = "Create web security group"
  type        = bool
  default     = true
}

variable "create_app_sg" {
  description = "Create application security group"
  type        = bool
  default     = true
}

variable "create_db_sg" {
  description = "Create database security group"
  type        = bool
  default     = true
}

variable "enable_flow_logs" {
  description = "Enable VPC Flow Logs"
  type        = bool
  default     = true
}

variable "flow_log_retention_days" {
  description = "CloudWatch log retention in days for VPC Flow Logs"
  type        = number
  default     = 30
  
  validation {
    condition = contains([1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365, 400, 545, 731, 1827, 3653], var.flow_log_retention_days)
    error_message = "Flow log retention days must be a valid CloudWatch Logs retention value."
  }
}

variable "additional_tags" {
  description = "Additional tags to apply to all resources"
  type        = map(string)
  default     = {}
}
```

**Create Terraform Outputs**:
```hcl
# modules/vpc/outputs.tf
output "vpc_id" {
  description = "ID of the VPC"
  value       = aws_vpc.main.id
}

output "vpc_arn" {
  description = "ARN of the VPC"
  value       = aws_vpc.main.arn
}

output "vpc_cidr_block" {
  description = "CIDR block of the VPC"
  value       = aws_vpc.main.cidr_block
}

output "internet_gateway_id" {
  description = "ID of the Internet Gateway"
  value       = var.create_igw ? aws_internet_gateway.main[0].id : null
}

output "public_subnet_ids" {
  description = "List of IDs of the public subnets"
  value       = aws_subnet.public[*].id
}

output "public_subnet_arns" {
  description = "List of ARNs of the public subnets"
  value       = aws_subnet.public[*].arn
}

output "public_subnet_cidr_blocks" {
  description = "List of CIDR blocks of the public subnets"
  value       = aws_subnet.public[*].cidr_block
}

output "private_subnet_ids" {
  description = "List of IDs of the private subnets"
  value       = aws_subnet.private[*].id
}

output "private_subnet_arns" {
  description = "List of ARNs of the private subnets"
  value       = aws_subnet.private[*].arn
}

output "private_subnet_cidr_blocks" {
  description = "List of CIDR blocks of the private subnets"
  value       = aws_subnet.private[*].cidr_block
}

output "database_subnet_ids" {
  description = "List of IDs of the database subnets"
  value       = aws_subnet.database[*].id
}

output "database_subnet_arns" {
  description = "List of ARNs of the database subnets"
  value       = aws_subnet.database[*].arn
}

output "database_subnet_cidr_blocks" {
  description = "List of CIDR blocks of the database subnets"
  value       = aws_subnet.database[*].cidr_block
}

output "nat_gateway_ids" {
  description = "List of IDs of the NAT Gateways"
  value       = aws_nat_gateway.main[*].id
}

output "nat_gateway_public_ips" {
  description = "List of public Elastic IPs associated with the NAT Gateways"
  value       = aws_eip.nat[*].public_ip
}

output "vpn_gateway_id" {
  description = "ID of the VPN Gateway"
  value       = var.enable_vpn_gateway ? aws_vpn_gateway.main[0].id : null
}

output "public_route_table_ids" {
  description = "List of IDs of the public route tables"
  value       = aws_route_table.public[*].id
}

output "private_route_table_ids" {
  description = "List of IDs of the private route tables"
  value       = aws_route_table.private[*].id
}

output "database_route_table_ids" {
  description = "List of IDs of the database route tables"
  value       = aws_route_table.database[*].id
}

output "default_security_group_id" {
  description = "ID of the default security group"
  value       = aws_security_group.default.id
}

output "web_security_group_id" {
  description = "ID of the web security group"
  value       = var.create_web_sg ? aws_security_group.web[0].id : null
}

output "app_security_group_id" {
  description = "ID of the application security group"
  value       = var.create_app_sg ? aws_security_group.app[0].id : null
}

output "database_security_group_id" {
  description = "ID of the database security group"
  value       = var.create_db_sg ? aws_security_group.database[0].id : null
}

output "vpc_flow_log_id" {
  description = "ID of the VPC Flow Log"
  value       = var.enable_flow_logs ? aws_flow_log.vpc[0].id : null
}

output "vpc_flow_log_cloudwatch_log_group" {
  description = "CloudWatch Log Group name for VPC Flow Logs"
  value       = var.enable_flow_logs ? aws_cloudwatch_log_group.vpc_flow_log[0].name : null
}

# Computed values
output "availability_zones" {
  description = "List of availability zones used"
  value       = local.azs
}

output "azs_count" {
  description = "Number of availability zones used"
  value       = local.azs_count
}
```

**Create Environment-Specific Configuration**:
```hcl
# environments/dev/main.tf
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
  
  backend "s3" {
    bucket  = "your-terraform-state-bucket"
    key     = "vpc/dev/terraform.tfstate"
    region  = "us-west-2"
    encrypt = true
    
    dynamodb_table = "terraform-lock-table"
  }
}

provider "aws" {
  region = var.aws_region
  
  default_tags {
    tags = {
      Environment = "dev"
      Project     = "VPCAutomation"
      ManagedBy   = "Terraform"
      Owner       = "NetworkTeam"
    }
  }
}

module "vpc" {
  source = "../../modules/vpc"
  
  environment                = "dev"
  project_name              = "VPCAutomation"
  owner                     = "NetworkTeam"
  vpc_cidr                  = "10.0.0.0/16"
  enable_dns_hostnames      = true
  enable_dns_support        = true
  create_igw                = true
  enable_public_subnets     = true
  enable_private_subnets    = true
  enable_database_subnets   = true
  enable_nat_gateway        = true
  enable_vpn_gateway        = false
  create_web_sg             = true
  create_app_sg             = true
  create_db_sg              = true
  enable_flow_logs          = true
  flow_log_retention_days   = 7
  
  additional_tags = {
    CostCenter = "Engineering"
    Backup     = "Required"
  }
}

# Optional: Create VPC Endpoints for S3 and DynamoDB
resource "aws_vpc_endpoint" "s3" {
  vpc_id            = module.vpc.vpc_id
  service_name      = "com.amazonaws.us-west-2.s3"
  vpc_endpoint_type = "Gateway"
  
  route_table_ids = concat(
    module.vpc.public_route_table_ids,
    module.vpc.private_route_table_ids,
    module.vpc.database_route_table_ids
  )
  
  tags = {
    Name        = "dev-s3-endpoint"
    Environment = "dev"
    Type        = "VPCEndpoint"
  }
}

resource "aws_vpc_endpoint" "dynamodb" {
  vpc_id            = module.vpc.vpc_id
  service_name      = "com.amazonaws.us-west-2.dynamodb"
  vpc_endpoint_type = "Gateway"
  
  route_table_ids = concat(
    module.vpc.public_route_table_ids,
    module.vpc.private_route_table_ids,
    module.vpc.database_route_table_ids
  )
  
  tags = {
    Name        = "dev-dynamodb-endpoint"
    Environment = "dev"
    Type        = "VPCEndpoint"
  }
}
```

**Create Environment Variables**:
```hcl
# environments/dev/variables.tf
variable "aws_region" {
  description = "AWS region"
  type        = string
  default     = "us-west-2"
}
```

**Create Environment Outputs**:
```hcl
# environments/dev/outputs.tf
output "vpc_details" {
  description = "VPC configuration details"
  value = {
    vpc_id          = module.vpc.vpc_id
    vpc_cidr        = module.vpc.vpc_cidr_block
    vpc_arn         = module.vpc.vpc_arn
    igw_id          = module.vpc.internet_gateway_id
    availability_zones = module.vpc.availability_zones
  }
}

output "subnet_details" {
  description = "Subnet configuration details"
  value = {
    public_subnet_ids    = module.vpc.public_subnet_ids
    private_subnet_ids   = module.vpc.private_subnet_ids
    database_subnet_ids  = module.vpc.database_subnet_ids
    public_subnet_cidrs  = module.vpc.public_subnet_cidr_blocks
    private_subnet_cidrs = module.vpc.private_subnet_cidr_blocks
    database_subnet_cidrs = module.vpc.database_subnet_cidr_blocks
  }
}

output "security_group_details" {
  description = "Security group configuration details"
  value = {
    default_sg_id  = module.vpc.default_security_group_id
    web_sg_id      = module.vpc.web_security_group_id
    app_sg_id      = module.vpc.app_security_group_id
    database_sg_id = module.vpc.database_security_group_id
  }
}

output "routing_details" {
  description = "Routing configuration details"
  value = {
    public_route_table_ids   = module.vpc.public_route_table_ids
    private_route_table_ids  = module.vpc.private_route_table_ids
    database_route_table_ids = module.vpc.database_route_table_ids
    nat_gateway_ids          = module.vpc.nat_gateway_ids
    nat_gateway_public_ips   = module.vpc.nat_gateway_public_ips
  }
}

output "monitoring_details" {
  description = "Monitoring configuration details"
  value = {
    flow_log_id         = module.vpc.vpc_flow_log_id
    flow_log_group_name = module.vpc.vpc_flow_log_cloudwatch_log_group
  }
}
```

**Deploy Terraform Configuration**:
```bash
# Initialize Terraform
cd environments/dev
terraform init

# Plan deployment
terraform plan -out=tfplan

# Apply configuration
terraform apply tfplan

# Show outputs
terraform output
```

### Step 3: AWS CDK VPC Implementation
**Objective**: Create enterprise VPC using AWS CDK with TypeScript

**Create CDK VPC Stack**:
```typescript
// lib/vpc-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as logs from 'aws-cdk-lib/aws-logs';
import * as iam from 'aws-cdk-lib/aws-iam';
import { Construct } from 'constructs';

export interface VpcStackProps extends cdk.StackProps {
  environment: string;
  vpcCidr: string;
  enableNatGateway: boolean;
  enableVpnGateway: boolean;
  enableFlowLogs: boolean;
  flowLogRetentionDays: number;
}

export class VpcStack extends cdk.Stack {
  public readonly vpc: ec2.Vpc;
  public readonly webSecurityGroup: ec2.SecurityGroup;
  public readonly appSecurityGroup: ec2.SecurityGroup;
  public readonly databaseSecurityGroup: ec2.SecurityGroup;

  constructor(scope: Construct, id: string, props: VpcStackProps) {
    super(scope, id, props);

    // Create VPC
    this.vpc = new ec2.Vpc(this, 'VPC', {
      ipAddresses: ec2.IpAddresses.cidr(props.vpcCidr),
      maxAzs: 2,
      enableDnsHostnames: true,
      enableDnsSupport: true,
      
      subnetConfiguration: [
        {
          cidrMask: 24,
          name: 'Public',
          subnetType: ec2.SubnetType.PUBLIC,
        },
        {
          cidrMask: 24,
          name: 'Private',
          subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
        },
        {
          cidrMask: 24,
          name: 'Database',
          subnetType: ec2.SubnetType.PRIVATE_ISOLATED,
        },
      ],
      
      natGateways: props.enableNatGateway ? 2 : 0,
      
      gatewayEndpoints: {
        S3: {
          service: ec2.GatewayVpcEndpointAwsService.S3,
        },
        DynamoDB: {
          service: ec2.GatewayVpcEndpointAwsService.DYNAMODB,
        },
      },
    });

    // Add VPN Gateway if enabled
    if (props.enableVpnGateway) {
      const vpnGateway = new ec2.VpnGateway(this, 'VpnGateway', {
        type: ec2.VpnConnectionType.IPSEC_1,
      });

      new ec2.VpcGatewayAttachment(this, 'VpnGatewayAttachment', {
        vpc: this.vpc,
        vpnGateway: vpnGateway,
      });

      // Enable route propagation for private subnets
      this.vpc.privateSubnets.forEach((subnet, index) => {
        new ec2.VpnGatewayRoutePropagation(this, `VpnRoutePropagation${index}`, {
          vpnGateway: vpnGateway,
          routeTable: subnet.routeTable,
        });
      });

      // Enable route propagation for isolated subnets
      this.vpc.isolatedSubnets.forEach((subnet, index) => {
        new ec2.VpnGatewayRoutePropagation(this, `VpnRoutePropagationIsolated${index}`, {
          vpnGateway: vpnGateway,
          routeTable: subnet.routeTable,
        });
      });
    }

    // Create Security Groups
    this.webSecurityGroup = new ec2.SecurityGroup(this, 'WebSecurityGroup', {
      vpc: this.vpc,
      description: 'Security group for web servers',
      allowAllOutbound: true,
    });

    this.webSecurityGroup.addIngressRule(
      ec2.Peer.anyIpv4(),
      ec2.Port.tcp(80),
      'Allow HTTP traffic'
    );

    this.webSecurityGroup.addIngressRule(
      ec2.Peer.anyIpv4(),
      ec2.Port.tcp(443),
      'Allow HTTPS traffic'
    );

    this.webSecurityGroup.addIngressRule(
      ec2.Peer.ipv4(props.vpcCidr),
      ec2.Port.tcp(22),
      'Allow SSH from VPC'
    );

    this.appSecurityGroup = new ec2.SecurityGroup(this, 'AppSecurityGroup', {
      vpc: this.vpc,
      description: 'Security group for application servers',
      allowAllOutbound: true,
    });

    this.appSecurityGroup.addIngressRule(
      this.webSecurityGroup,
      ec2.Port.tcp(8080),
      'Allow traffic from web tier'
    );

    this.appSecurityGroup.addIngressRule(
      ec2.Peer.ipv4(props.vpcCidr),
      ec2.Port.tcp(22),
      'Allow SSH from VPC'
    );

    this.databaseSecurityGroup = new ec2.SecurityGroup(this, 'DatabaseSecurityGroup', {
      vpc: this.vpc,
      description: 'Security group for database servers',
      allowAllOutbound: false,
    });

    this.databaseSecurityGroup.addIngressRule(
      this.appSecurityGroup,
      ec2.Port.tcp(3306),
      'Allow MySQL from app tier'
    );

    this.databaseSecurityGroup.addIngressRule(
      this.appSecurityGroup,
      ec2.Port.tcp(5432),
      'Allow PostgreSQL from app tier'
    );

    // Create VPC Flow Logs if enabled
    if (props.enableFlowLogs) {
      const flowLogGroup = new logs.LogGroup(this, 'VpcFlowLogGroup', {
        logGroupName: `/aws/vpc/flowlogs/${props.environment}`,
        retention: props.flowLogRetentionDays,
        removalPolicy: cdk.RemovalPolicy.DESTROY,
      });

      const flowLogRole = new iam.Role(this, 'FlowLogRole', {
        assumedBy: new iam.ServicePrincipal('vpc-flow-logs.amazonaws.com'),
        inlinePolicies: {
          FlowLogDeliveryRolePolicy: new iam.PolicyDocument({
            statements: [
              new iam.PolicyStatement({
                effect: iam.Effect.ALLOW,
                actions: [
                  'logs:CreateLogGroup',
                  'logs:CreateLogStream',
                  'logs:PutLogEvents',
                  'logs:DescribeLogGroups',
                  'logs:DescribeLogStreams',
                ],
                resources: [`arn:aws:logs:${this.region}:${this.account}:*`],
              }),
            ],
          }),
        },
      });

      new ec2.FlowLog(this, 'VpcFlowLog', {
        resourceType: ec2.FlowLogResourceType.fromVpc(this.vpc),
        destination: ec2.FlowLogDestination.toCloudWatchLogs(flowLogGroup, flowLogRole),
        trafficType: ec2.FlowLogTrafficType.ALL,
      });
    }

    // Add Interface VPC Endpoints for common AWS services
    this.vpc.addInterfaceEndpoint('EC2Endpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.EC2,
      privateDnsEnabled: true,
      subnets: {
        subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
      },
    });

    this.vpc.addInterfaceEndpoint('ECSEndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.ECS,
      privateDnsEnabled: true,
      subnets: {
        subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
      },
    });

    this.vpc.addInterfaceEndpoint('SecretsManagerEndpoint', {
      service: ec2.InterfaceVpcEndpointAwsService.SECRETS_MANAGER,
      privateDnsEnabled: true,
      subnets: {
        subnetType: ec2.SubnetType.PRIVATE_WITH_EGRESS,
      },
    });

    // Add tags to all resources
    cdk.Tags.of(this).add('Environment', props.environment);
    cdk.Tags.of(this).add('Project', 'VPCAutomation');
    cdk.Tags.of(this).add('ManagedBy', 'CDK');
  }
}
```

**Create CDK Application**:
```typescript
// bin/vpc-automation.ts
#!/usr/bin/env node
import * as cdk from 'aws-cdk-lib';
import { VpcStack } from '../lib/vpc-stack';

const app = new cdk.App();

// Development Environment
new VpcStack(app, 'DevVpcStack', {
  environment: 'dev',
  vpcCidr: '10.0.0.0/16',
  enableNatGateway: true,
  enableVpnGateway: false,
  enableFlowLogs: true,
  flowLogRetentionDays: 7,
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION,
  },
  tags: {
    Environment: 'dev',
    CostCenter: 'Engineering',
  },
});

// Production Environment
new VpcStack(app, 'ProdVpcStack', {
  environment: 'prod',
  vpcCidr: '10.1.0.0/16',
  enableNatGateway: true,
  enableVpnGateway: true,
  enableFlowLogs: true,
  flowLogRetentionDays: 90,
  env: {
    account: process.env.CDK_DEFAULT_ACCOUNT,
    region: process.env.CDK_DEFAULT_REGION,
  },
  tags: {
    Environment: 'prod',
    CostCenter: 'Engineering',
    Backup: 'Required',
  },
});
```

**Deploy CDK Application**:
```bash
# Install dependencies
npm install

# Bootstrap CDK (if not done before)
cdk bootstrap

# Synthesize CloudFormation templates
cdk synth

# Deploy development environment
cdk deploy DevVpcStack

# Deploy production environment
cdk deploy ProdVpcStack

# List all stacks
cdk list

# View differences
cdk diff DevVpcStack
```

### Step 4: CI/CD Pipeline Implementation
**Objective**: Create automated CI/CD pipeline for VPC infrastructure deployment

**Create GitHub Actions Workflow**:
```yaml
# .github/workflows/vpc-infrastructure.yml
name: VPC Infrastructure Deployment

on:
  push:
    branches: [main, develop]
    paths:
      - 'infrastructure/**'
      - '.github/workflows/vpc-infrastructure.yml'
  pull_request:
    branches: [main]
    paths:
      - 'infrastructure/**'

env:
  AWS_REGION: us-west-2
  TERRAFORM_VERSION: 1.5.7

permissions:
  id-token: write
  contents: read
  pull-requests: write

jobs:
  validate:
    name: Validate Infrastructure Code
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TERRAFORM_VERSION }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_OIDC_ROLE_ARN }}
          role-session-name: GitHubActions-VPCInfrastructure
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Format Check
        run: |
          cd infrastructure/terraform
          terraform fmt -check -recursive

      - name: Terraform Init
        run: |
          cd infrastructure/terraform/environments/dev
          terraform init -backend=false

      - name: Terraform Validate
        run: |
          cd infrastructure/terraform/environments/dev
          terraform validate

      - name: Terraform Security Scan
        uses: aquasecurity/tfsec-action@v1.0.3
        with:
          working_directory: infrastructure/terraform
          soft_fail: false

      - name: CloudFormation Lint
        run: |
          pip install cfn-lint
          cfn-lint infrastructure/cloudformation/*.yaml

  plan-dev:
    name: Plan Development Environment
    runs-on: ubuntu-latest
    needs: validate
    if: github.event_name == 'pull_request'
    
    outputs:
      terraform-plan-output: ${{ steps.plan.outputs.stdout }}
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TERRAFORM_VERSION }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_OIDC_ROLE_ARN }}
          role-session-name: GitHubActions-VPCInfrastructure
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: |
          cd infrastructure/terraform/environments/dev
          terraform init

      - name: Terraform Plan
        id: plan
        run: |
          cd infrastructure/terraform/environments/dev
          terraform plan -no-color -out=tfplan
        continue-on-error: true

      - name: Update Pull Request
        uses: actions/github-script@v7
        if: github.event_name == 'pull_request'
        env:
          PLAN: "terraform\n${{ steps.plan.outputs.stdout }}"
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const output = `#### Terraform Format and Validation 🤖\`${{ steps.validate.outcome }}\`
            #### Terraform Plan 📖\`${{ steps.plan.outcome }}\`

            <details><summary>Show Plan</summary>

            \`\`\`\n
            ${process.env.PLAN}
            \`\`\`

            </details>

            *Pusher: @${{ github.actor }}, Action: \`${{ github.event_name }}\`*`;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body: output
            })

  deploy-dev:
    name: Deploy to Development
    runs-on: ubuntu-latest
    needs: validate
    if: github.ref == 'refs/heads/develop' && github.event_name == 'push'
    environment: development
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TERRAFORM_VERSION }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_OIDC_ROLE_ARN }}
          role-session-name: GitHubActions-VPCInfrastructure
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: |
          cd infrastructure/terraform/environments/dev
          terraform init

      - name: Terraform Plan
        run: |
          cd infrastructure/terraform/environments/dev
          terraform plan -out=tfplan

      - name: Terraform Apply
        run: |
          cd infrastructure/terraform/environments/dev
          terraform apply -auto-approve tfplan

      - name: Save Terraform Outputs
        run: |
          cd infrastructure/terraform/environments/dev
          terraform output -json > terraform-outputs.json

      - name: Upload Terraform Outputs
        uses: actions/upload-artifact@v4
        with:
          name: dev-terraform-outputs
          path: infrastructure/terraform/environments/dev/terraform-outputs.json

  deploy-prod:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: validate
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    environment: production
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Setup Terraform
        uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: ${{ env.TERRAFORM_VERSION }}

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_PROD_OIDC_ROLE_ARN }}
          role-session-name: GitHubActions-VPCInfrastructure-Prod
          aws-region: ${{ env.AWS_REGION }}

      - name: Terraform Init
        run: |
          cd infrastructure/terraform/environments/prod
          terraform init

      - name: Terraform Plan
        run: |
          cd infrastructure/terraform/environments/prod
          terraform plan -out=tfplan

      - name: Terraform Apply
        run: |
          cd infrastructure/terraform/environments/prod
          terraform apply -auto-approve tfplan

      - name: Save Terraform Outputs
        run: |
          cd infrastructure/terraform/environments/prod
          terraform output -json > terraform-outputs.json

      - name: Upload Terraform Outputs
        uses: actions/upload-artifact@v4
        with:
          name: prod-terraform-outputs
          path: infrastructure/terraform/environments/prod/terraform-outputs.json

  security-compliance:
    name: Security and Compliance Checks
    runs-on: ubuntu-latest
    needs: [deploy-dev]
    if: github.ref == 'refs/heads/develop' && github.event_name == 'push'
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_OIDC_ROLE_ARN }}
          role-session-name: GitHubActions-VPCCompliance
          aws-region: ${{ env.AWS_REGION }}

      - name: Download Terraform Outputs
        uses: actions/download-artifact@v4
        with:
          name: dev-terraform-outputs

      - name: Run VPC Compliance Checks
        run: |
          python infrastructure/scripts/vpc-compliance-check.py \
            --terraform-outputs terraform-outputs.json \
            --environment dev \
            --output-format json > compliance-report.json

      - name: Upload Compliance Report
        uses: actions/upload-artifact@v4
        with:
          name: vpc-compliance-report
          path: compliance-report.json

  notifications:
    name: Send Notifications
    runs-on: ubuntu-latest
    needs: [deploy-dev, deploy-prod]
    if: always()
    
    steps:
      - name: Notify Slack on Success
        if: ${{ needs.deploy-dev.result == 'success' || needs.deploy-prod.result == 'success' }}
        uses: 8398a7/action-slack@v3
        with:
          status: success
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
          text: "✅ VPC Infrastructure deployment completed successfully"

      - name: Notify Slack on Failure
        if: ${{ needs.deploy-dev.result == 'failure' || needs.deploy-prod.result == 'failure' }}
        uses: 8398a7/action-slack@v3
        with:
          status: failure
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
          text: "❌ VPC Infrastructure deployment failed"
```

**Create VPC Compliance Check Script**:
```python
#!/usr/bin/env python3
# infrastructure/scripts/vpc-compliance-check.py

import json
import boto3
import argparse
import sys
from typing import Dict, List, Any, Tuple
from datetime import datetime

class VPCComplianceChecker:
    def __init__(self, region: str = 'us-west-2'):
        self.ec2 = boto3.client('ec2', region_name=region)
        self.logs = boto3.client('logs', region_name=region)
        self.region = region
        self.compliance_results = []

    def check_vpc_flow_logs(self, vpc_id: str) -> bool:
        """Check if VPC Flow Logs are enabled"""
        try:
            response = self.ec2.describe_flow_logs(
                Filters=[
                    {'Name': 'resource-id', 'Values': [vpc_id]},
                    {'Name': 'resource-type', 'Values': ['VPC']}
                ]
            )
            
            active_flow_logs = [
                fl for fl in response['FlowLogs'] 
                if fl['FlowLogStatus'] == 'ACTIVE'
            ]
            
            result = len(active_flow_logs) > 0
            self.compliance_results.append({
                'check': 'VPC Flow Logs Enabled',
                'status': 'PASS' if result else 'FAIL',
                'resource': vpc_id,
                'details': f"Found {len(active_flow_logs)} active flow logs"
            })
            return result
            
        except Exception as e:
            self.compliance_results.append({
                'check': 'VPC Flow Logs Enabled',
                'status': 'ERROR',
                'resource': vpc_id,
                'details': str(e)
            })
            return False

    def check_default_security_group(self, vpc_id: str) -> bool:
        """Check if default security group has restrictive rules"""
        try:
            response = self.ec2.describe_security_groups(
                Filters=[
                    {'Name': 'vpc-id', 'Values': [vpc_id]},
                    {'Name': 'group-name', 'Values': ['default']}
                ]
            )
            
            if not response['SecurityGroups']:
                self.compliance_results.append({
                    'check': 'Default Security Group Rules',
                    'status': 'ERROR',
                    'resource': vpc_id,
                    'details': 'Default security group not found'
                })
                return False
            
            default_sg = response['SecurityGroups'][0]
            
            # Check for overly permissive rules
            permissive_ingress = any(
                rule.get('CidrIp') == '0.0.0.0/0' or rule.get('CidrIpv6') == '::/0'
                for rule in default_sg.get('IpPermissions', [])
            )
            
            permissive_egress = any(
                rule.get('CidrIp') == '0.0.0.0/0' or rule.get('CidrIpv6') == '::/0'
                for rule in default_sg.get('IpPermissionsEgress', [])
                if rule.get('IpProtocol') != '-1' or 'UserIdGroupPairs' not in rule
            )
            
            result = not (permissive_ingress or permissive_egress)
            
            self.compliance_results.append({
                'check': 'Default Security Group Rules',
                'status': 'PASS' if result else 'FAIL',
                'resource': default_sg['GroupId'],
                'details': (
                    'Default security group has restrictive rules' if result 
                    else 'Default security group has overly permissive rules'
                )
            })
            return result
            
        except Exception as e:
            self.compliance_results.append({
                'check': 'Default Security Group Rules',
                'status': 'ERROR',
                'resource': vpc_id,
                'details': str(e)
            })
            return False

    def check_subnet_availability_zones(self, vpc_id: str) -> bool:
        """Check if subnets are distributed across multiple AZs"""
        try:
            response = self.ec2.describe_subnets(
                Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}]
            )
            
            subnets = response['Subnets']
            availability_zones = set(subnet['AvailabilityZone'] for subnet in subnets)
            
            result = len(availability_zones) >= 2
            
            self.compliance_results.append({
                'check': 'Multi-AZ Subnet Distribution',
                'status': 'PASS' if result else 'FAIL',
                'resource': vpc_id,
                'details': f"Subnets distributed across {len(availability_zones)} AZs"
            })
            return result
            
        except Exception as e:
            self.compliance_results.append({
                'check': 'Multi-AZ Subnet Distribution',
                'status': 'ERROR',
                'resource': vpc_id,
                'details': str(e)
            })
            return False

    def check_private_subnet_nat_gateway(self, vpc_id: str) -> bool:
        """Check if private subnets have NAT Gateway for internet access"""
        try:
            # Get private subnets
            subnets_response = self.ec2.describe_subnets(
                Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}]
            )
            
            private_subnets = [
                subnet for subnet in subnets_response['Subnets']
                if not subnet.get('MapPublicIpOnLaunch', False)
            ]
            
            if not private_subnets:
                self.compliance_results.append({
                    'check': 'Private Subnet NAT Gateway',
                    'status': 'SKIP',
                    'resource': vpc_id,
                    'details': 'No private subnets found'
                })
                return True
            
            # Get route tables for private subnets
            route_tables_response = self.ec2.describe_route_tables(
                Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}]
            )
            
            private_subnets_with_nat = 0
            for subnet in private_subnets:
                subnet_route_table = None
                for rt in route_tables_response['RouteTables']:
                    for assoc in rt.get('Associations', []):
                        if assoc.get('SubnetId') == subnet['SubnetId']:
                            subnet_route_table = rt
                            break
                    if subnet_route_table:
                        break
                
                if not subnet_route_table:
                    # Use main route table
                    for rt in route_tables_response['RouteTables']:
                        for assoc in rt.get('Associations', []):
                            if assoc.get('Main', False):
                                subnet_route_table = rt
                                break
                
                if subnet_route_table:
                    has_nat_route = any(
                        route.get('NatGatewayId') for route in subnet_route_table.get('Routes', [])
                    )
                    if has_nat_route:
                        private_subnets_with_nat += 1
            
            result = private_subnets_with_nat > 0
            
            self.compliance_results.append({
                'check': 'Private Subnet NAT Gateway',
                'status': 'PASS' if result else 'FAIL',
                'resource': vpc_id,
                'details': f"{private_subnets_with_nat}/{len(private_subnets)} private subnets have NAT Gateway routes"
            })
            return result
            
        except Exception as e:
            self.compliance_results.append({
                'check': 'Private Subnet NAT Gateway',
                'status': 'ERROR',
                'resource': vpc_id,
                'details': str(e)
            })
            return False

    def check_resource_tagging(self, vpc_id: str) -> bool:
        """Check if VPC and associated resources have proper tags"""
        try:
            required_tags = ['Environment', 'Project', 'Owner']
            
            # Check VPC tags
            vpc_response = self.ec2.describe_vpcs(VpcIds=[vpc_id])
            vpc = vpc_response['Vpcs'][0]
            vpc_tags = {tag['Key']: tag['Value'] for tag in vpc.get('Tags', [])}
            
            missing_tags = [tag for tag in required_tags if tag not in vpc_tags]
            
            result = len(missing_tags) == 0
            
            self.compliance_results.append({
                'check': 'Resource Tagging',
                'status': 'PASS' if result else 'FAIL',
                'resource': vpc_id,
                'details': (
                    'All required tags present' if result 
                    else f"Missing required tags: {', '.join(missing_tags)}"
                )
            })
            return result
            
        except Exception as e:
            self.compliance_results.append({
                'check': 'Resource Tagging',
                'status': 'ERROR',
                'resource': vpc_id,
                'details': str(e)
            })
            return False

    def check_vpc_endpoints(self, vpc_id: str) -> bool:
        """Check if VPC has endpoints for common AWS services"""
        try:
            response = self.ec2.describe_vpc_endpoints(
                Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}]
            )
            
            endpoints = response['VpcEndpoints']
            endpoint_services = set(endpoint['ServiceName'] for endpoint in endpoints)
            
            # Check for common service endpoints
            common_services = [
                f'com.amazonaws.{self.region}.s3',
                f'com.amazonaws.{self.region}.dynamodb'
            ]
            
            found_services = [
                service for service in common_services 
                if service in endpoint_services
            ]
            
            result = len(found_services) >= 1
            
            self.compliance_results.append({
                'check': 'VPC Endpoints Configuration',
                'status': 'PASS' if result else 'FAIL',
                'resource': vpc_id,
                'details': f"Found {len(found_services)} common service endpoints"
            })
            return result
            
        except Exception as e:
            self.compliance_results.append({
                'check': 'VPC Endpoints Configuration',
                'status': 'ERROR',
                'resource': vpc_id,
                'details': str(e)
            })
            return False

    def run_compliance_checks(self, vpc_id: str) -> Dict[str, Any]:
        """Run all compliance checks for a VPC"""
        checks = [
            self.check_vpc_flow_logs,
            self.check_default_security_group,
            self.check_subnet_availability_zones,
            self.check_private_subnet_nat_gateway,
            self.check_resource_tagging,
            self.check_vpc_endpoints
        ]
        
        results = []
        for check in checks:
            try:
                results.append(check(vpc_id))
            except Exception as e:
                print(f"Error running check {check.__name__}: {e}", file=sys.stderr)
                results.append(False)
        
        passed_checks = sum(results)
        total_checks = len(results)
        compliance_score = (passed_checks / total_checks) * 100 if total_checks > 0 else 0
        
        return {
            'vpc_id': vpc_id,
            'timestamp': datetime.utcnow().isoformat(),
            'compliance_score': compliance_score,
            'total_checks': total_checks,
            'passed_checks': passed_checks,
            'failed_checks': total_checks - passed_checks,
            'check_results': self.compliance_results
        }

def main():
    parser = argparse.ArgumentParser(description='VPC Compliance Checker')
    parser.add_argument('--terraform-outputs', required=True, help='Path to Terraform outputs JSON file')
    parser.add_argument('--environment', required=True, help='Environment name')
    parser.add_argument('--region', default='us-west-2', help='AWS region')
    parser.add_argument('--output-format', choices=['json', 'text'], default='text', help='Output format')
    
    args = parser.parse_args()
    
    # Load Terraform outputs
    try:
        with open(args.terraform_outputs, 'r') as f:
            terraform_outputs = json.load(f)
    except FileNotFoundError:
        print(f"Error: Terraform outputs file not found: {args.terraform_outputs}", file=sys.stderr)
        sys.exit(1)
    except json.JSONDecodeError:
        print(f"Error: Invalid JSON in Terraform outputs file: {args.terraform_outputs}", file=sys.stderr)
        sys.exit(1)
    
    # Extract VPC ID from Terraform outputs
    vpc_id = None
    if 'vpc_details' in terraform_outputs and 'value' in terraform_outputs['vpc_details']:
        vpc_id = terraform_outputs['vpc_details']['value'].get('vpc_id')
    elif 'vpc_id' in terraform_outputs and 'value' in terraform_outputs['vpc_id']:
        vpc_id = terraform_outputs['vpc_id']['value']
    
    if not vpc_id:
        print("Error: VPC ID not found in Terraform outputs", file=sys.stderr)
        sys.exit(1)
    
    # Run compliance checks
    checker = VPCComplianceChecker(region=args.region)
    results = checker.run_compliance_checks(vpc_id)
    
    # Output results
    if args.output_format == 'json':
        print(json.dumps(results, indent=2))
    else:
        print(f"\nVPC Compliance Report for {args.environment} environment")
        print(f"VPC ID: {vpc_id}")
        print(f"Compliance Score: {results['compliance_score']:.1f}%")
        print(f"Passed: {results['passed_checks']}/{results['total_checks']}")
        print("\nDetailed Results:")
        
        for check_result in results['check_results']:
            status_symbol = "✅" if check_result['status'] == 'PASS' else "❌" if check_result['status'] == 'FAIL' else "⚠️"
            print(f"{status_symbol} {check_result['check']}: {check_result['status']}")
            print(f"   Details: {check_result['details']}")
    
    # Exit with error code if compliance score is below threshold
    if results['compliance_score'] < 80:
        sys.exit(1)

if __name__ == '__main__':
    main()
```

### Step 5: Infrastructure Testing and Validation
**Objective**: Implement automated testing for VPC infrastructure

**Create Infrastructure Test Suite**:
```python
#!/usr/bin/env python3
# tests/test_vpc_infrastructure.py

import boto3
import pytest
import json
import time
from typing import Dict, List
from moto import mock_ec2, mock_logs, mock_iam

class TestVPCInfrastructure:
    """Test suite for VPC infrastructure validation"""
    
    def __init__(self, region='us-west-2'):
        self.region = region
        self.ec2 = boto3.client('ec2', region_name=region)
        self.logs = boto3.client('logs', region_name=region)
        
    def test_vpc_exists_and_configured(self, vpc_id: str):
        """Test that VPC exists and is properly configured"""
        response = self.ec2.describe_vpcs(VpcIds=[vpc_id])
        vpc = response['Vpcs'][0]
        
        # Verify VPC configuration
        assert vpc['State'] == 'available'
        assert vpc['DhcpOptionsId'] is not None
        assert vpc['InstanceTenancy'] == 'default'
        
        # Verify DNS configuration
        assert vpc['EnableDnsHostnames'] is True
        assert vpc['EnableDnsSupport'] is True
        
        return True

    def test_subnets_configuration(self, vpc_id: str):
        """Test subnet configuration and distribution"""
        response = self.ec2.describe_subnets(
            Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}]
        )
        subnets = response['Subnets']
        
        # Verify minimum subnet count
        assert len(subnets) >= 6, "Should have at least 6 subnets (2 public, 2 private, 2 database)"
        
        # Group subnets by type based on tags
        public_subnets = [s for s in subnets if 'Public' in str(s.get('Tags', []))]
        private_subnets = [s for s in subnets if 'Private' in str(s.get('Tags', []))]
        database_subnets = [s for s in subnets if 'Database' in str(s.get('Tags', []))]
        
        # Verify subnet distribution across AZs
        public_azs = set(s['AvailabilityZone'] for s in public_subnets)
        private_azs = set(s['AvailabilityZone'] for s in private_subnets)
        database_azs = set(s['AvailabilityZone'] for s in database_subnets)
        
        assert len(public_azs) >= 2, "Public subnets should span at least 2 AZs"
        assert len(private_azs) >= 2, "Private subnets should span at least 2 AZs"
        assert len(database_azs) >= 2, "Database subnets should span at least 2 AZs"
        
        # Verify public subnet configuration
        for subnet in public_subnets:
            assert subnet['MapPublicIpOnLaunch'] is True, "Public subnets should auto-assign public IPs"
        
        return True

    def test_internet_gateway_configuration(self, vpc_id: str):
        """Test Internet Gateway configuration"""
        response = self.ec2.describe_internet_gateways(
            Filters=[{'Name': 'attachment.vpc-id', 'Values': [vpc_id]}]
        )
        
        assert len(response['InternetGateways']) == 1, "Should have exactly one Internet Gateway"
        
        igw = response['InternetGateways'][0]
        assert igw['State'] == 'available'
        
        # Verify attachment
        attachments = igw['Attachments']
        assert len(attachments) == 1
        assert attachments[0]['VpcId'] == vpc_id
        assert attachments[0]['State'] == 'attached'
        
        return True

    def test_nat_gateway_configuration(self, vpc_id: str):
        """Test NAT Gateway configuration"""
        response = self.ec2.describe_nat_gateways(
            Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}]
        )
        
        nat_gateways = response['NatGateways']
        available_nat_gateways = [ng for ng in nat_gateways if ng['State'] == 'available']
        
        assert len(available_nat_gateways) >= 1, "Should have at least one NAT Gateway"
        
        # Verify NAT Gateway is in public subnet
        public_subnets_response = self.ec2.describe_subnets(
            Filters=[
                {'Name': 'vpc-id', 'Values': [vpc_id]},
                {'Name': 'map-public-ip-on-launch', 'Values': ['true']}
            ]
        )
        public_subnet_ids = set(s['SubnetId'] for s in public_subnets_response['Subnets'])
        
        for nat_gateway in available_nat_gateways:
            assert nat_gateway['SubnetId'] in public_subnet_ids, "NAT Gateway should be in public subnet"
            assert nat_gateway['ConnectivityType'] == 'public'
        
        return True

    def test_route_table_configuration(self, vpc_id: str):
        """Test route table configuration"""
        response = self.ec2.describe_route_tables(
            Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}]
        )
        route_tables = response['RouteTables']
        
        # Should have multiple route tables (main + custom)
        assert len(route_tables) >= 2, "Should have multiple route tables"
        
        # Check for Internet Gateway routes in public route tables
        igw_response = self.ec2.describe_internet_gateways(
            Filters=[{'Name': 'attachment.vpc-id', 'Values': [vpc_id]}]
        )
        igw_id = igw_response['InternetGateways'][0]['InternetGatewayId']
        
        public_route_found = False
        for rt in route_tables:
            for route in rt['Routes']:
                if route.get('GatewayId') == igw_id and route.get('DestinationCidrBlock') == '0.0.0.0/0':
                    public_route_found = True
                    break
        
        assert public_route_found, "Should have route to Internet Gateway"
        
        return True

    def test_security_groups_configuration(self, vpc_id: str):
        """Test security group configuration"""
        response = self.ec2.describe_security_groups(
            Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}]
        )
        security_groups = response['SecurityGroups']
        
        # Should have default security group plus custom ones
        assert len(security_groups) >= 4, "Should have default + web + app + database security groups"
        
        # Find specific security groups
        web_sg = None
        app_sg = None
        db_sg = None
        
        for sg in security_groups:
            sg_name = sg['GroupName'].lower()
            if 'web' in sg_name:
                web_sg = sg
            elif 'app' in sg_name:
                app_sg = sg
            elif 'db' in sg_name or 'database' in sg_name:
                db_sg = sg
        
        # Test web security group
        if web_sg:
            ingress_rules = web_sg['IpPermissions']
            http_rule = any(rule['FromPort'] == 80 and rule['ToPort'] == 80 for rule in ingress_rules)
            https_rule = any(rule['FromPort'] == 443 and rule['ToPort'] == 443 for rule in ingress_rules)
            assert http_rule, "Web security group should allow HTTP"
            assert https_rule, "Web security group should allow HTTPS"
        
        # Test application security group
        if app_sg and web_sg:
            ingress_rules = app_sg['IpPermissions']
            app_rule = any(
                rule['FromPort'] == 8080 and rule['ToPort'] == 8080 and
                any(ug['GroupId'] == web_sg['GroupId'] for ug in rule.get('UserIdGroupPairs', []))
                for rule in ingress_rules
            )
            assert app_rule, "App security group should allow traffic from web security group"
        
        return True

    def test_vpc_flow_logs(self, vpc_id: str):
        """Test VPC Flow Logs configuration"""
        response = self.ec2.describe_flow_logs(
            Filters=[
                {'Name': 'resource-id', 'Values': [vpc_id]},
                {'Name': 'resource-type', 'Values': ['VPC']}
            ]
        )
        
        flow_logs = response['FlowLogs']
        active_flow_logs = [fl for fl in flow_logs if fl['FlowLogStatus'] == 'ACTIVE']
        
        assert len(active_flow_logs) >= 1, "Should have at least one active VPC Flow Log"
        
        # Verify flow log configuration
        for flow_log in active_flow_logs:
            assert flow_log['TrafficType'] == 'ALL'
            assert flow_log['LogDestinationType'] in ['cloud-watch-logs', 's3']
        
        return True

    def test_vpc_endpoints(self, vpc_id: str):
        """Test VPC Endpoints configuration"""
        response = self.ec2.describe_vpc_endpoints(
            Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}]
        )
        
        endpoints = response['VpcEndpoints']
        endpoint_services = set(endpoint['ServiceName'] for endpoint in endpoints)
        
        # Check for common service endpoints
        s3_endpoint = f'com.amazonaws.{self.region}.s3'
        dynamodb_endpoint = f'com.amazonaws.{self.region}.dynamodb'
        
        assert s3_endpoint in endpoint_services or dynamodb_endpoint in endpoint_services, \
            "Should have at least S3 or DynamoDB endpoint"
        
        return True

    def test_network_connectivity(self, vpc_id: str):
        """Test basic network connectivity within VPC"""
        # This would typically involve launching test instances
        # For this example, we'll test route table connectivity
        
        response = self.ec2.describe_route_tables(
            Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}]
        )
        
        route_tables = response['RouteTables']
        
        # Verify all route tables have local route
        for rt in route_tables:
            local_route = any(
                route.get('GatewayId') == 'local' for route in rt['Routes']
            )
            assert local_route, f"Route table {rt['RouteTableId']} missing local route"
        
        return True

    def run_all_tests(self, vpc_id: str) -> Dict[str, bool]:
        """Run all infrastructure tests"""
        tests = [
            ('VPC Configuration', self.test_vpc_exists_and_configured),
            ('Subnet Configuration', self.test_subnets_configuration),
            ('Internet Gateway', self.test_internet_gateway_configuration),
            ('NAT Gateway', self.test_nat_gateway_configuration),
            ('Route Tables', self.test_route_table_configuration),
            ('Security Groups', self.test_security_groups_configuration),
            ('VPC Flow Logs', self.test_vpc_flow_logs),
            ('VPC Endpoints', self.test_vpc_endpoints),
            ('Network Connectivity', self.test_network_connectivity),
        ]
        
        results = {}
        for test_name, test_func in tests:
            try:
                results[test_name] = test_func(vpc_id)
                print(f"✅ {test_name}: PASSED")
            except Exception as e:
                results[test_name] = False
                print(f"❌ {test_name}: FAILED - {str(e)}")
        
        return results

def main():
    import argparse
    
    parser = argparse.ArgumentParser(description='VPC Infrastructure Testing')
    parser.add_argument('--vpc-id', required=True, help='VPC ID to test')
    parser.add_argument('--region', default='us-west-2', help='AWS region')
    parser.add_argument('--output-format', choices=['json', 'text'], default='text', help='Output format')
    
    args = parser.parse_args()
    
    # Run tests
    tester = TestVPCInfrastructure(region=args.region)
    results = tester.run_all_tests(args.vpc_id)
    
    # Calculate summary
    total_tests = len(results)
    passed_tests = sum(1 for result in results.values() if result)
    success_rate = (passed_tests / total_tests) * 100 if total_tests > 0 else 0
    
    if args.output_format == 'json':
        output = {
            'vpc_id': args.vpc_id,
            'timestamp': time.strftime('%Y-%m-%d %H:%M:%S UTC', time.gmtime()),
            'total_tests': total_tests,
            'passed_tests': passed_tests,
            'failed_tests': total_tests - passed_tests,
            'success_rate': success_rate,
            'test_results': results
        }
        print(json.dumps(output, indent=2))
    else:
        print(f"\nVPC Infrastructure Test Summary")
        print(f"VPC ID: {args.vpc_id}")
        print(f"Success Rate: {success_rate:.1f}%")
        print(f"Tests Passed: {passed_tests}/{total_tests}")
    
    # Exit with error if tests failed
    if success_rate < 100:
        exit(1)

if __name__ == '__main__':
    main()
```

## Best Practices Summary

### Infrastructure as Code Best Practices
1. **Modular Design**: Create reusable modules for common patterns
2. **Environment Separation**: Use separate state files and configurations per environment
3. **Version Control**: Track all infrastructure changes in Git
4. **Automated Testing**: Validate infrastructure before deployment
5. **Security Scanning**: Run security scans on infrastructure code

### Deployment Best Practices
1. **Progressive Deployment**: Deploy to dev → staging → production
2. **Rollback Strategy**: Maintain ability to rollback changes
3. **Change Management**: Use pull requests for infrastructure changes
4. **Monitoring**: Monitor infrastructure deployment and health
5. **Documentation**: Maintain up-to-date documentation

### Security Best Practices
1. **Least Privilege**: Apply minimal required permissions
2. **Encryption**: Enable encryption for all data in transit and at rest
3. **Audit Logging**: Enable comprehensive logging and monitoring
4. **Compliance**: Implement automated compliance checking
5. **Secret Management**: Use proper secret management tools

## Troubleshooting Guide

### Common Issues
1. **State File Conflicts**: Use remote state with locking
2. **Permission Errors**: Verify IAM roles and policies
3. **Resource Limits**: Check AWS service limits
4. **Dependency Issues**: Ensure proper resource dependencies
5. **Network Connectivity**: Verify routing and security groups

### Debugging Commands
```bash
# Terraform debugging
export TF_LOG=DEBUG
terraform plan -detailed-exitcode

# CloudFormation debugging
aws cloudformation describe-stack-events --stack-name <stack-name>

# CDK debugging
cdk synth --verbose
cdk diff --verbose
```

## Additional Resources
- [AWS VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/)
- [Terraform AWS Provider Documentation](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [AWS CDK Developer Guide](https://docs.aws.amazon.com/cdk/latest/guide/)
- [AWS CloudFormation User Guide](https://docs.aws.amazon.com/cloudformation/)
- [Infrastructure as Code Best Practices](https://aws.amazon.com/blogs/devops/)

## Exam Tips
- Understand different IaC tools and their use cases
- Know how to design modular and reusable infrastructure templates
- Understand CI/CD pipeline integration for infrastructure
- Be familiar with infrastructure testing and validation techniques
- Know security best practices for infrastructure automation