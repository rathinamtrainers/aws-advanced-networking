# Topic 3: Understanding Virtual Private Cloud (VPC) Basics

## Prerequisites
- Completed Topics 1-2 (AWS Networking Intro and Global Infrastructure)
- Understanding of IP addressing and CIDR notation
- Basic knowledge of subnetting concepts

## Learning Objectives
By the end of this topic, you will be able to:
- Create and configure custom VPCs with proper CIDR planning
- Understand VPC components and their relationships
- Design VPC architectures following AWS best practices
- Troubleshoot common VPC connectivity issues

## Theory

### What is Amazon VPC?
Amazon Virtual Private Cloud (VPC) enables you to launch AWS resources in a logically isolated virtual network that you define. You have complete control over your virtual networking environment.

### Key VPC Concepts

#### 1. VPC CIDR Blocks
**Definition**: IP address range for your VPC using CIDR notation

**Characteristics**:
- Primary CIDR block: Required, cannot be changed after creation
- Secondary CIDR blocks: Optional, can be added later (up to 5 total)
- Allowed ranges: 10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16 (RFC 1918)
- Size limits: /16 netmask (65,536 IPs) to /28 netmask (16 IPs)

#### 2. Default vs Custom VPC
**Default VPC**:
- Created automatically in each region
- CIDR: 172.31.0.0/16
- Public subnets in each AZ
- Internet Gateway attached
- Ready for immediate use

**Custom VPC**:
- You define CIDR block
- No components created by default
- Complete control over architecture
- Best practice for production workloads

#### 3. VPC Components

**Subnets**:
- Subdivision of VPC CIDR
- Must reside in single AZ
- Can be public, private, or isolated

**Route Tables**:
- Control traffic routing
- Associated with subnets
- Main route table vs custom route tables

**Internet Gateway (IGW)**:
- Allows communication with internet
- Horizontally scaled and redundant
- One per VPC

**NAT Gateway/Instance**:
- Enables outbound internet access for private subnets
- NAT Gateway: Managed service
- NAT Instance: Self-managed EC2 instance

### VPC Networking Rules

#### DNS Resolution
- **DNS Resolution**: Resolves public DNS hostnames to IP addresses
- **DNS Hostnames**: Assigns public DNS hostnames to instances
- Both enabled by default in default VPC

#### Reserved IP Addresses
In each subnet CIDR block, AWS reserves:
- **Network address**: First IP (e.g., 10.0.1.0)
- **VPC router**: Second IP (e.g., 10.0.1.1)  
- **DNS server**: Third IP (e.g., 10.0.1.2)
- **Future use**: Fourth IP (e.g., 10.0.1.3)
- **Broadcast**: Last IP (e.g., 10.0.1.255)

## Lab Exercise: Creating and Configuring Custom VPC

### Lab Duration: 90 minutes

### Step 1: Plan VPC Architecture
**Objective**: Design a proper VPC architecture before implementation

**Planning Exercise**:
1. Define requirements:
   - 2 Availability Zones for high availability
   - Public and private subnets in each AZ
   - Web tier in public subnets
   - Application tier in private subnets
   - Database tier in private subnets

2. CIDR Planning:
   - VPC CIDR: 10.0.0.0/16 (65,536 IP addresses)
   - Subnet allocation:
     - Public Subnet AZ-1: 10.0.1.0/24 (254 usable IPs)
     - Private Subnet AZ-1: 10.0.2.0/24 (254 usable IPs)
     - Public Subnet AZ-2: 10.0.3.0/24 (254 usable IPs)
     - Private Subnet AZ-2: 10.0.4.0/24 (254 usable IPs)
     - Database Subnet AZ-1: 10.0.5.0/24 (254 usable IPs)
     - Database Subnet AZ-2: 10.0.6.0/24 (254 usable IPs)

**Verification**: Complete architecture diagram with IP allocation

### Step 2: Create Custom VPC
**Objective**: Build VPC infrastructure using AWS Console

**AWS Console Steps**:
1. Navigate to **VPC Console**
2. Click **"Create VPC"**
3. Choose **"VPC only"** (not VPC and more)
4. Configure VPC settings:
   - **Name**: `Lab-VPC`
   - **IPv4 CIDR block**: `10.0.0.0/16`
   - **IPv6 CIDR block**: No IPv6 CIDR block
   - **Tenancy**: Default
   - **Tags**: Add `Environment: Lab`
5. Click **"Create VPC"**
6. Record VPC ID: `vpc-xxxxxxxxx`

**CLI Commands**:
```bash
# Create VPC
aws ec2 create-vpc \
    --cidr-block 10.0.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Lab-VPC},{Key=Environment,Value=Lab}]'

# Enable DNS resolution and hostnames
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=Lab-VPC" --query 'Vpcs[0].VpcId' --output text)

aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
```

**Expected Output**:
```json
{
    "Vpc": {
        "VpcId": "vpc-12345678",
        "State": "available",
        "CidrBlock": "10.0.0.0/16"
    }
}
```

**Verification**: VPC appears in VPC list with "available" state

### Step 3: Create Subnets
**Objective**: Create public and private subnets across multiple AZs

**AWS Console Steps**:
1. In VPC Console, click **"Subnets"**
2. Click **"Create subnet"**
3. Create first subnet:
   - **VPC ID**: Select Lab-VPC
   - **Subnet name**: `Public-Subnet-AZ1`
   - **Availability Zone**: us-east-1a (or first AZ)
   - **IPv4 CIDR block**: `10.0.1.0/24`
4. Click **"Add new subnet"** and repeat for:
   - **Name**: `Private-Subnet-AZ1`, **AZ**: us-east-1a, **CIDR**: `10.0.2.0/24`
   - **Name**: `Public-Subnet-AZ2`, **AZ**: us-east-1b, **CIDR**: `10.0.3.0/24`
   - **Name**: `Private-Subnet-AZ2`, **AZ**: us-east-1b, **CIDR**: `10.0.4.0/24`
5. Click **"Create subnet"**

**CLI Commands**:
```bash
# Get VPC ID
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=Lab-VPC" --query 'Vpcs[0].VpcId' --output text)

# Create subnets
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Public-Subnet-AZ1}]'

aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Private-Subnet-AZ1}]'

aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.3.0/24 --availability-zone us-east-1b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Public-Subnet-AZ2}]'

aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.4.0/24 --availability-zone us-east-1b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Private-Subnet-AZ2}]'
```

**Verification**: All subnets show "Available" state and correct CIDR blocks

### Step 4: Create and Attach Internet Gateway
**Objective**: Enable internet connectivity for public subnets

**AWS Console Steps**:
1. In VPC Console, click **"Internet Gateways"**
2. Click **"Create internet gateway"**
3. Configure:
   - **Name**: `Lab-IGW`
   - **Tags**: Add `Environment: Lab`
4. Click **"Create internet gateway"**
5. Select the new IGW and click **"Actions"** → **"Attach to VPC"**
6. Select Lab-VPC and click **"Attach internet gateway"**

**CLI Commands**:
```bash
# Create Internet Gateway
aws ec2 create-internet-gateway \
    --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=Lab-IGW},{Key=Environment,Value=Lab}]'

# Get IGW ID
IGW_ID=$(aws ec2 describe-internet-gateways --filters "Name=tag:Name,Values=Lab-IGW" --query 'InternetGateways[0].InternetGatewayId' --output text)

# Attach to VPC
aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID
```

**Verification**: IGW shows "Attached" state to correct VPC

### Step 5: Configure Route Tables
**Objective**: Set up routing for public and private subnets

**AWS Console Steps**:
1. Click **"Route Tables"** in VPC Console
2. Identify the main route table for Lab-VPC
3. Create custom route table for public subnets:
   - Click **"Create route table"**
   - **Name**: `Public-Route-Table`
   - **VPC**: Lab-VPC
   - Click **"Create route table"**
4. Edit routes for public route table:
   - Select Public-Route-Table
   - Click **"Routes"** tab
   - Click **"Edit routes"**
   - Click **"Add route"**
   - **Destination**: `0.0.0.0/0`
   - **Target**: Internet Gateway → Lab-IGW
   - Click **"Save changes"**
5. Associate public subnets:
   - Click **"Subnet associations"** tab
   - Click **"Edit subnet associations"**
   - Select both public subnets
   - Click **"Save associations"**

**CLI Commands**:
```bash
# Create public route table
aws ec2 create-route-table --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Public-Route-Table}]'

# Get route table ID
RTB_ID=$(aws ec2 describe-route-tables --filters "Name=tag:Name,Values=Public-Route-Table" --query 'RouteTables[0].RouteTableId' --output text)

# Add internet route
aws ec2 create-route --route-table-id $RTB_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

# Get subnet IDs
PUBLIC_SUBNET_1=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=Public-Subnet-AZ1" --query 'Subnets[0].SubnetId' --output text)
PUBLIC_SUBNET_2=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=Public-Subnet-AZ2" --query 'Subnets[0].SubnetId' --output text)

# Associate subnets with route table
aws ec2 associate-route-table --route-table-id $RTB_ID --subnet-id $PUBLIC_SUBNET_1
aws ec2 associate-route-table --route-table-id $RTB_ID --subnet-id $PUBLIC_SUBNET_2
```

**Verification**: Public subnets have route to 0.0.0.0/0 via IGW

### Step 6: Test VPC Connectivity
**Objective**: Verify VPC networking is working correctly

**AWS Console Steps**:
1. Launch test EC2 instance in public subnet:
   - Navigate to **EC2 Console**
   - Click **"Launch Instance"**
   - **Name**: `Test-Instance`
   - **AMI**: Amazon Linux 2023
   - **Instance type**: t3.micro
   - **Key pair**: Create or select existing
   - **Network settings**:
     - **VPC**: Lab-VPC
     - **Subnet**: Public-Subnet-AZ1
     - **Auto-assign public IP**: Enable
     - **Security group**: Create new allowing SSH (port 22)
   - Click **"Launch instance"**

2. Test connectivity:
   - Wait for instance to reach "running" state
   - Note public and private IP addresses
   - SSH to instance using public IP
   - From instance, test internet connectivity: `ping google.com`

**Expected Results**:
- Instance gets both public and private IP
- SSH connection successful
- Internet ping successful

## Architecture Diagram
```
Internet
    |
[Internet Gateway]
    |
[Custom VPC - 10.0.0.0/16]
    |
    ├─[Public Route Table]────[Internet Gateway Route: 0.0.0.0/0]
    │   │
    │   ├─[Public Subnet AZ1: 10.0.1.0/24]
    │   └─[Public Subnet AZ2: 10.0.3.0/24]
    │
    └─[Main Route Table]
        │
        ├─[Private Subnet AZ1: 10.0.2.0/24]
        └─[Private Subnet AZ2: 10.0.4.0/24]
```

## Troubleshooting
| Issue | Cause | Solution |
|-------|-------|----------|
| Instance no internet access | Missing IGW or route | Verify IGW attached and 0.0.0.0/0 route exists |
| Cannot SSH to instance | Security group rules | Allow inbound SSH (port 22) from your IP |
| Instance no public IP | Auto-assign disabled | Enable auto-assign public IP or use Elastic IP |
| Wrong subnet CIDR | Overlapping ranges | Plan non-overlapping CIDR blocks |
| DNS resolution fails | DNS settings disabled | Enable DNS resolution and hostnames |

## Key Takeaways
- VPC provides isolated network environment with complete control
- Proper CIDR planning is essential before creating VPC
- Subnets are AZ-specific and inherit VPC CIDR
- Internet Gateway enables public internet connectivity
- Route tables control traffic flow between subnets and external networks
- Public subnets need explicit route to Internet Gateway

## Best Practices
- **CIDR Planning**: Choose non-overlapping CIDR blocks considering future expansion
- **Multi-AZ Design**: Always create subnets in multiple AZs
- **Subnet Types**: Use public for internet-facing resources, private for internal resources
- **Route Table Design**: Use custom route tables for better control
- **Security**: Follow principle of least privilege for routing and security groups
- **Documentation**: Tag all resources consistently for management

## Additional Resources
- [VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/)
- [VPC Sizing](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Subnets.html#vpc-sizing-ipv4)
- [VPC Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-best-practices.html)

## Exam Tips
- Know VPC CIDR limitations (/16 to /28)
- Understand reserved IP addresses in each subnet (5 total)
- Remember main route table vs custom route tables
- Public subnet = subnet with route to IGW via route table
- Private subnet = subnet without route to IGW
- IGW is horizontally scaled and highly available by design