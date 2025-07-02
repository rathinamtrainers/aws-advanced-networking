# Topic 4: Subnets Explained - Public, Private, and Isolated

## Prerequisites
- Completed Topics 1-3 (AWS Networking, Infrastructure, VPC Basics)
- Understanding of IP addressing and routing concepts
- Familiarity with AWS VPC components

## Learning Objectives
By the end of this topic, you will be able to:
- Differentiate between public, private, and isolated subnets
- Design subnet architectures for different application tiers
- Implement proper subnet routing configurations
- Apply security best practices for subnet design

## Theory

### Subnet Overview
Subnets are subdivisions of your VPC that allow you to group resources based on security and operational requirements. Each subnet resides in a single Availability Zone.

### Types of Subnets

#### 1. Public Subnets
**Definition**: Subnets with route to Internet Gateway, allowing direct internet connectivity

**Characteristics**:
- Route table contains route to Internet Gateway (0.0.0.0/0 → IGW)
- Resources can have public IP addresses
- Directly accessible from internet (with proper security groups)
- Used for: Load balancers, bastion hosts, NAT instances, web servers

**Use Cases**:
- Application Load Balancers (ALB)
- Network Load Balancers (NLB) for internet-facing services
- Bastion hosts for administrative access
- NAT Gateways (AWS managed)
- Web servers requiring direct internet access

#### 2. Private Subnets
**Definition**: Subnets without direct route to Internet Gateway but with outbound internet access via NAT

**Characteristics**:
- No route to Internet Gateway in route table
- Route to NAT Gateway/Instance for outbound internet (0.0.0.0/0 → NAT)
- Resources cannot be directly accessed from internet
- Can initiate outbound connections to internet
- Used for: Application servers, internal load balancers, microservices

**Use Cases**:
- Application tier servers
- Internal Application Load Balancers
- Microservices and APIs
- Processing workloads requiring internet access for updates
- Container orchestration nodes

#### 3. Isolated Subnets (Private without NAT)
**Definition**: Subnets with no internet connectivity in either direction

**Characteristics**:
- No route to Internet Gateway or NAT Gateway
- Completely isolated from internet
- Can only communicate within VPC or through VPC endpoints
- Used for: Databases, highly sensitive workloads, compliance requirements

**Use Cases**:
- Database servers (RDS, ElastiCache)
- Data warehouses (Redshift)
- Internal-only services
- Compliance-sensitive workloads
- Backend processing systems

### Subnet Design Patterns

#### 1. Three-Tier Architecture
```
Internet → Public Subnet (Web Tier)
           ↓
       Private Subnet (App Tier)
           ↓
       Isolated Subnet (Database Tier)
```

#### 2. Microservices Architecture
```
Internet → Public Subnet (Load Balancers)
           ↓
       Private Subnet (API Gateway, Services)
           ↓
       Isolated Subnet (Databases, Cache)
```

#### 3. Multi-AZ Pattern
```
AZ-1a: Public + Private + Isolated
AZ-1b: Public + Private + Isolated
AZ-1c: Public + Private + Isolated
```

### Subnet Routing Details

#### Auto-Assign Public IP
- **Public subnets**: Usually enabled
- **Private subnets**: Usually disabled
- **Isolated subnets**: Always disabled
- Can be overridden per instance at launch

#### Route Table Associations
- Each subnet must be associated with a route table
- Subnets without explicit association use main route table
- Multiple subnets can share same route table
- Custom route tables provide better control

## Lab Exercise: Implementing Multi-Tier Subnet Architecture

### Lab Duration: 120 minutes

### Step 1: Design Advanced Subnet Architecture
**Objective**: Plan comprehensive subnet design with multiple tiers

**Architecture Planning**:
```
VPC: 10.0.0.0/16

AZ us-east-1a:
- Public Subnet:    10.0.1.0/24  (Web/LB tier)
- Private Subnet:   10.0.2.0/24  (App tier)
- Isolated Subnet:  10.0.3.0/24  (DB tier)

AZ us-east-1b:
- Public Subnet:    10.0.11.0/24 (Web/LB tier)
- Private Subnet:   10.0.12.0/24 (App tier)
- Isolated Subnet:  10.0.13.0/24 (DB tier)

Management:
- Management Subnet: 10.0.100.0/24 (Bastion hosts)
```

**Design Verification**: Document IP allocation and routing requirements

### Step 2: Create Extended VPC Infrastructure
**Objective**: Build comprehensive subnet infrastructure

**AWS Console Steps**:
1. Create VPC (if not exists from previous lab):
   - **Name**: `Multi-Tier-VPC`
   - **CIDR**: `10.0.0.0/16`

2. Create all subnets:
   - Navigate to **VPC Console** → **Subnets**
   - Click **"Create subnet"**
   - Create each subnet with specifications above

**CLI Commands**:
```bash
# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Multi-Tier-VPC}]'

VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=Multi-Tier-VPC" --query 'Vpcs[0].VpcId' --output text)

# Create subnets for AZ-1a
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Public-1a},{Key=Type,Value=Public}]'

aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Private-1a},{Key=Type,Value=Private}]'

aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.3.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Isolated-1a},{Key=Type,Value=Isolated}]'

# Create subnets for AZ-1b
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.11.0/24 --availability-zone us-east-1b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Public-1b},{Key=Type,Value=Public}]'

aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.12.0/24 --availability-zone us-east-1b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Private-1b},{Key=Type,Value=Private}]'

aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.13.0/24 --availability-zone us-east-1b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Isolated-1b},{Key=Type,Value=Isolated}]'

# Management subnet
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.100.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Management},{Key=Type,Value=Management}]'
```

### Step 3: Configure Internet Gateway and NAT Gateways
**Objective**: Set up internet connectivity infrastructure

**AWS Console Steps**:
1. Create and attach Internet Gateway:
   - **Name**: `Multi-Tier-IGW`
   - Attach to Multi-Tier-VPC

2. Create NAT Gateways for high availability:
   - Navigate to **VPC Console** → **NAT Gateways**
   - Click **"Create NAT Gateway"**
   - **Name**: `NAT-Gateway-1a`
   - **Subnet**: Public-1a
   - **Connectivity type**: Public
   - **Elastic IP**: Allocate new
   - Repeat for second NAT Gateway:
     - **Name**: `NAT-Gateway-1b`
     - **Subnet**: Public-1b

**CLI Commands**:
```bash
# Create Internet Gateway
aws ec2 create-internet-gateway \
    --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=Multi-Tier-IGW}]'

IGW_ID=$(aws ec2 describe-internet-gateways --filters "Name=tag:Name,Values=Multi-Tier-IGW" --query 'InternetGateways[0].InternetGatewayId' --output text)

# Attach IGW to VPC
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# Get subnet IDs
PUBLIC_1A=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=Public-1a" --query 'Subnets[0].SubnetId' --output text)
PUBLIC_1B=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=Public-1b" --query 'Subnets[0].SubnetId' --output text)

# Allocate Elastic IPs
EIP_1A=$(aws ec2 allocate-address --domain vpc --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=NAT-EIP-1a}]' --query 'AllocationId' --output text)
EIP_1B=$(aws ec2 allocate-address --domain vpc --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=NAT-EIP-1b}]' --query 'AllocationId' --output text)

# Create NAT Gateways
NAT_1A=$(aws ec2 create-nat-gateway --subnet-id $PUBLIC_1A --allocation-id $EIP_1A \
    --tag-specifications 'ResourceType=nat-gateway,Tags=[{Key=Name,Value=NAT-Gateway-1a}]' --query 'NatGateway.NatGatewayId' --output text)

NAT_1B=$(aws ec2 create-nat-gateway --subnet-id $PUBLIC_1B --allocation-id $EIP_1B \
    --tag-specifications 'ResourceType=nat-gateway,Tags=[{Key=Name,Value=NAT-Gateway-1b}]' --query 'NatGateway.NatGatewayId' --output text)

# Wait for NAT Gateways to be available
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_1A $NAT_1B
```

### Step 4: Create Route Tables for Each Subnet Type
**Objective**: Configure proper routing for different subnet types

**AWS Console Steps**:
1. Create Public Route Table:
   - **Name**: `Public-Route-Table`
   - Add route: `0.0.0.0/0` → Internet Gateway
   - Associate: Public-1a, Public-1b, Management subnets

2. Create Private Route Tables (one per AZ for HA):
   - **Name**: `Private-Route-Table-1a`
   - Add route: `0.0.0.0/0` → NAT-Gateway-1a
   - Associate: Private-1a subnet
   
   - **Name**: `Private-Route-Table-1b`
   - Add route: `0.0.0.0/0` → NAT-Gateway-1b
   - Associate: Private-1b subnet

3. Isolated subnets use main route table (no internet routes)

**CLI Commands**:
```bash
# Create Public Route Table
PUBLIC_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Public-Route-Table}]' \
    --query 'RouteTable.RouteTableId' --output text)

# Add internet route to public route table
aws ec2 create-route --route-table-id $PUBLIC_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

# Associate public subnets
MGMT_SUBNET=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=Management" --query 'Subnets[0].SubnetId' --output text)

aws ec2 associate-route-table --route-table-id $PUBLIC_RT --subnet-id $PUBLIC_1A
aws ec2 associate-route-table --route-table-id $PUBLIC_RT --subnet-id $PUBLIC_1B
aws ec2 associate-route-table --route-table-id $PUBLIC_RT --subnet-id $MGMT_SUBNET

# Create Private Route Tables
PRIVATE_RT_1A=$(aws ec2 create-route-table --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Private-Route-Table-1a}]' \
    --query 'RouteTable.RouteTableId' --output text)

PRIVATE_RT_1B=$(aws ec2 create-route-table --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Private-Route-Table-1b}]' \
    --query 'RouteTable.RouteTableId' --output text)

# Add NAT routes
aws ec2 create-route --route-table-id $PRIVATE_RT_1A --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_1A
aws ec2 create-route --route-table-id $PRIVATE_RT_1B --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_1B

# Associate private subnets
PRIVATE_1A=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=Private-1a" --query 'Subnets[0].SubnetId' --output text)
PRIVATE_1B=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=Private-1b" --query 'Subnets[0].SubnetId' --output text)

aws ec2 associate-route-table --route-table-id $PRIVATE_RT_1A --subnet-id $PRIVATE_1A
aws ec2 associate-route-table --route-table-id $PRIVATE_RT_1B --subnet-id $PRIVATE_1B
```

### Step 5: Test Subnet Connectivity
**Objective**: Verify each subnet type behaves correctly

**Testing Steps**:
1. Launch instances in each subnet type:
   - **Bastion Host**: Management subnet (public IP)
   - **App Server**: Private subnet (no public IP)
   - **Database**: Isolated subnet (no public IP)

2. Test connectivity patterns:
   - SSH to bastion host from internet
   - SSH from bastion to app server (private IP)
   - SSH from bastion to database (private IP)
   - Test internet access from app server: `curl https://aws.amazon.com`
   - Verify database has no internet access

**Security Groups Setup**:
```bash
# Create security groups
BASTION_SG=$(aws ec2 create-security-group --group-name Bastion-SG --description "Bastion Host Security Group" --vpc-id $VPC_ID --query 'GroupId' --output text)

APP_SG=$(aws ec2 create-security-group --group-name App-SG --description "Application Tier Security Group" --vpc-id $VPC_ID --query 'GroupId' --output text)

DB_SG=$(aws ec2 create-security-group --group-name DB-SG --description "Database Tier Security Group" --vpc-id $VPC_ID --query 'GroupId' --output text)

# Configure security group rules
# Bastion: SSH from internet
aws ec2 authorize-security-group-ingress --group-id $BASTION_SG --protocol tcp --port 22 --cidr 0.0.0.0/0

# App: SSH from bastion, HTTP/HTTPS from ALB
aws ec2 authorize-security-group-ingress --group-id $APP_SG --protocol tcp --port 22 --source-group $BASTION_SG
aws ec2 authorize-security-group-ingress --group-id $APP_SG --protocol tcp --port 80 --cidr 10.0.0.0/16
aws ec2 authorize-security-group-ingress --group-id $APP_SG --protocol tcp --port 443 --cidr 10.0.0.0/16

# Database: MySQL from app tier only
aws ec2 authorize-security-group-ingress --group-id $DB_SG --protocol tcp --port 3306 --source-group $APP_SG
aws ec2 authorize-security-group-ingress --group-id $DB_SG --protocol tcp --port 22 --source-group $BASTION_SG
```

## Architecture Diagram
```
Internet
    |
[Internet Gateway]
    |
┌─────────────────────────────────────────────────────────────────┐
│ VPC: 10.0.0.0/16                                               │
│                                                                 │
│ ┌─────────────────┐    ┌─────────────────┐                     │
│ │ AZ us-east-1a   │    │ AZ us-east-1b   │                     │
│ │                 │    │                 │                     │
│ │ Public Subnet   │    │ Public Subnet   │ ← [Public Route]    │
│ │ 10.0.1.0/24     │    │ 10.0.11.0/24    │     │               │
│ │ [NAT-GW-1a]     │    │ [NAT-GW-1b]     │     └→ IGW          │
│ │                 │    │                 │                     │
│ │ Private Subnet  │    │ Private Subnet  │ ← [Private Routes]  │
│ │ 10.0.2.0/24     │    │ 10.0.12.0/24    │     │               │
│ │      ↑          │    │      ↑          │     └→ NAT-GWs      │
│ │      │          │    │      │          │                     │
│ │ Isolated Subnet │    │ Isolated Subnet │ ← [Main Route]      │
│ │ 10.0.3.0/24     │    │ 10.0.13.0/24    │   (VPC local only)  │
│ └─────────────────┘    └─────────────────┘                     │
└─────────────────────────────────────────────────────────────────┘
```

## Troubleshooting
| Issue | Cause | Solution |
|-------|-------|----------|
| Private instance no internet | Missing NAT Gateway route | Add 0.0.0.0/0 → NAT Gateway to route table |
| Public instance no internet access | Missing IGW route | Add 0.0.0.0/0 → IGW route |
| Cannot SSH to private instance | No bastion host | Use bastion host in public subnet |
| Isolated instance has internet | Wrong route table | Use main route table with no internet routes |
| High NAT costs | Single NAT Gateway | Deploy NAT Gateway per AZ for HA |
| Cross-AZ data charges | Wrong NAT routing | Route private subnets to NAT in same AZ |

## Key Takeaways
- Public subnets require route to Internet Gateway
- Private subnets use NAT Gateway for outbound internet access
- Isolated subnets have no internet connectivity
- Multi-AZ deployment provides high availability
- Route tables determine subnet behavior, not just naming
- Security groups provide additional access control
- NAT Gateways should be deployed per AZ to avoid cross-AZ charges

## Best Practices
- **Subnet Design**: Use descriptive naming and consistent tagging
- **High Availability**: Deploy resources across multiple AZs
- **Cost Optimization**: Use NAT Gateway per AZ to minimize data transfer charges
- **Security**: Apply defense in depth with subnets, security groups, and NACLs
- **Monitoring**: Set up VPC Flow Logs for traffic analysis
- **Documentation**: Maintain network diagrams and IP allocation records

## Additional Resources
- [VPC Subnets Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html)
- [NAT Gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
- [Route Tables](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)

## Exam Tips
- Public subnet = route to IGW, Private subnet = route to NAT, Isolated = no internet routes
- Each subnet exists in exactly one AZ
- Multiple subnets can share the same route table
- Auto-assign public IP is a subnet setting that can be overridden per instance
- Main route table is used by subnets without explicit associations
- Reserved IPs: 5 per subnet (network, router, DNS, future, broadcast)