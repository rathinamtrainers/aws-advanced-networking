# Topic 6: Deep Dive into VPC Route Tables

## Prerequisites
- Completed Topics 1-5 (AWS Networking Foundations)
- Understanding of IP routing concepts
- Knowledge of longest prefix matching

## Learning Objectives
By the end of this topic, you will be able to:
- Understand route table hierarchy and route precedence
- Configure complex routing scenarios with multiple route tables
- Implement route propagation and longest prefix matching
- Troubleshoot routing issues using AWS tools

## Theory

### Route Table Fundamentals

#### What are Route Tables?
Route tables contain a set of rules (routes) that determine where network traffic is directed. Every subnet in a VPC must be associated with a route table.

#### Route Table Components
- **Destination**: The CIDR block for the traffic destination
- **Target**: Where to send the traffic (IGW, NAT Gateway, VPC Peering, etc.)
- **Status**: Active or Blackhole
- **Propagated**: Whether the route was propagated automatically

### Route Table Types

#### 1. Main Route Table
- **Default**: Created automatically with VPC
- **Purpose**: Used by subnets without explicit route table associations
- **Best Practice**: Keep main route table for private/isolated subnets
- **Modification**: Can be modified but affects all associated subnets

#### 2. Custom Route Tables
- **Purpose**: Provide specific routing for different subnet types
- **Flexibility**: Can be created, modified, and deleted
- **Association**: Explicitly associated with subnets
- **Best Practice**: Use for public subnets and specialized routing

### Route Precedence and Matching

#### Longest Prefix Matching
Routes are evaluated using longest prefix matching - the most specific route wins.

**Example Route Table**:
```
Destination     Target          Precedence
10.0.0.0/16    Local           1 (VPC CIDR - always most specific for VPC)
10.0.1.0/24    NAT Gateway     2 (More specific than 0.0.0.0/0)
0.0.0.0/0      Internet Gateway 3 (Least specific - default route)
```

#### Route Priority Order
1. **Local routes**: VPC CIDR blocks (cannot be overridden)
2. **Most specific routes**: Longest prefix matches first
3. **Static routes**: Manually configured routes
4. **Propagated routes**: From VPN or Direct Connect
5. **Default route**: 0.0.0.0/0 (least specific)

### Advanced Routing Concepts

#### Route Propagation
- **VPN Route Propagation**: Routes learned from VPN connections
- **Direct Connect Route Propagation**: Routes from Direct Connect virtual interfaces
- **Transit Gateway Route Propagation**: Routes from Transit Gateway attachments
- **Dynamic Updates**: Routes updated automatically when propagation enabled

#### Blackhole Routes
- **Definition**: Routes that drop traffic instead of forwarding
- **Causes**: Target resource unavailable (NAT Gateway stopped, VPN down)
- **Status**: Shows as "Blackhole" in route table
- **Resolution**: Fix or remove the problematic target

## Lab Exercise: Advanced Route Table Configurations

### Lab Duration: 120 minutes

### Step 1: Create Complex Network Architecture
**Objective**: Build multi-tier architecture with different routing requirements

**Architecture Design**:
```
VPC: 10.0.0.0/16
├── Public Subnets (Web Tier)
│   ├── 10.0.1.0/24 (us-east-1a)
│   └── 10.0.2.0/24 (us-east-1b)
├── Private Subnets (App Tier)  
│   ├── 10.0.11.0/24 (us-east-1a)
│   └── 10.0.12.0/24 (us-east-1b)
├── Database Subnets (DB Tier)
│   ├── 10.0.21.0/24 (us-east-1a)
│   └── 10.0.22.0/24 (us-east-1b)
└── Management Subnet
    └── 10.0.100.0/24 (us-east-1a)
```

**CLI Implementation**:
```bash
# Create VPC
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=RouteTable-Lab-VPC}]' \
    --query 'Vpc.VpcId' --output text)

# Enable DNS resolution
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames

# Create subnets
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Public-Web-1a}]'

aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone us-east-1b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Public-Web-1b}]'

aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.11.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Private-App-1a}]'

aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.12.0/24 --availability-zone us-east-1b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Private-App-1b}]'

aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.21.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Isolated-DB-1a}]'

aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.22.0/24 --availability-zone us-east-1b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Isolated-DB-1b}]'

aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.100.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Management}]'
```

### Step 2: Create Internet Gateway and NAT Infrastructure
**Objective**: Set up internet connectivity components

```bash
# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
    --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=RouteTable-IGW}]' \
    --query 'InternetGateway.InternetGatewayId' --output text)

aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# Get subnet IDs
PUBLIC_WEB_1A=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=Public-Web-1a" --query 'Subnets[0].SubnetId' --output text)
PUBLIC_WEB_1B=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=Public-Web-1b" --query 'Subnets[0].SubnetId' --output text)

# Create NAT Gateways for high availability
EIP_1A=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
EIP_1B=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)

NAT_GW_1A=$(aws ec2 create-nat-gateway --subnet-id $PUBLIC_WEB_1A --allocation-id $EIP_1A \
    --tag-specifications 'ResourceType=nat-gateway,Tags=[{Key=Name,Value=NAT-1a}]' \
    --query 'NatGateway.NatGatewayId' --output text)

NAT_GW_1B=$(aws ec2 create-nat-gateway --subnet-id $PUBLIC_WEB_1B --allocation-id $EIP_1B \
    --tag-specifications 'ResourceType=nat-gateway,Tags=[{Key=Name,Value=NAT-1b}]' \
    --query 'NatGateway.NatGatewayId' --output text)

# Wait for NAT Gateways
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW_1A $NAT_GW_1B
```

### Step 3: Create Multiple Route Tables with Different Purposes
**Objective**: Implement sophisticated routing strategy

```bash
# 1. Public Route Table (for web tier)
PUBLIC_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Public-RT},{Key=Purpose,Value=Web-Tier}]' \
    --query 'RouteTable.RouteTableId' --output text)

# Add internet route
aws ec2 create-route --route-table-id $PUBLIC_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

# 2. Private Route Table AZ-1a (for app tier)
PRIVATE_RT_1A=$(aws ec2 create-route-table --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Private-RT-1a},{Key=Purpose,Value=App-Tier-1a}]' \
    --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $PRIVATE_RT_1A --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW_1A

# 3. Private Route Table AZ-1b (for app tier)
PRIVATE_RT_1B=$(aws ec2 create-route-table --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Private-RT-1b},{Key=Purpose,Value=App-Tier-1b}]' \
    --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $PRIVATE_RT_1B --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW_1B

# 4. Management Route Table (special routing for admin access)
MGMT_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Management-RT},{Key=Purpose,Value=Admin-Access}]' \
    --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $MGMT_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID

# 5. Database subnets use main route table (no internet access)
MAIN_RT=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" "Name=association.main,Values=true" \
    --query 'RouteTables[0].RouteTableId' --output text)
```

### Step 4: Associate Subnets with Appropriate Route Tables
**Objective**: Implement proper routing associations

```bash
# Get all subnet IDs
PRIVATE_APP_1A=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=Private-App-1a" --query 'Subnets[0].SubnetId' --output text)
PRIVATE_APP_1B=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=Private-App-1b" --query 'Subnets[0].SubnetId' --output text)
MGMT_SUBNET=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=Management" --query 'Subnets[0].SubnetId' --output text)

# Associate public subnets with public route table
aws ec2 associate-route-table --route-table-id $PUBLIC_RT --subnet-id $PUBLIC_WEB_1A
aws ec2 associate-route-table --route-table-id $PUBLIC_RT --subnet-id $PUBLIC_WEB_1B

# Associate private app subnets with their respective route tables
aws ec2 associate-route-table --route-table-id $PRIVATE_RT_1A --subnet-id $PRIVATE_APP_1A
aws ec2 associate-route-table --route-table-id $PRIVATE_RT_1B --subnet-id $PRIVATE_APP_1B

# Associate management subnet
aws ec2 associate-route-table --route-table-id $MGMT_RT --subnet-id $MGMT_SUBNET

# Database subnets automatically use main route table (no explicit association needed)
```

### Step 5: Test Route Precedence and Longest Prefix Matching
**Objective**: Demonstrate advanced routing concepts

**Create Test Routes**:
```bash
# Add more specific route to test longest prefix matching
# Route specific subnet traffic through different path
aws ec2 create-route --route-table-id $PRIVATE_RT_1A \
    --destination-cidr-block 10.0.21.0/24 \
    --vpc-peering-connection-id "local"  # This will fail - demonstrating

# Add route for specific external service
aws ec2 create-route --route-table-id $PRIVATE_RT_1A \
    --destination-cidr-block 8.8.8.0/24 \
    --nat-gateway-id $NAT_GW_1A

# View route table to see precedence
aws ec2 describe-route-tables --route-table-ids $PRIVATE_RT_1A \
    --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,GatewayId,State]' \
    --output table
```

### Step 6: Implement Route Monitoring and Analysis
**Objective**: Set up route table monitoring and troubleshooting

```bash
# Create VPC Flow Logs for route analysis
aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids $VPC_ID \
    --traffic-type ALL \
    --log-destination-type cloud-watch-logs \
    --log-group-name /aws/vpc/flowlogs \
    --deliver-logs-permission-arn arn:aws:iam::ACCOUNT:role/flowlogsRole

# Create CloudWatch dashboard for route monitoring
aws cloudwatch put-dashboard \
    --dashboard-name "VPC-Route-Monitoring" \
    --dashboard-body '{
        "widgets": [
            {
                "type": "metric",
                "properties": {
                    "metrics": [
                        ["AWS/VPC", "PacketDropCount", "VPC", "'$VPC_ID'"]
                    ],
                    "period": 300,
                    "stat": "Sum",
                    "region": "us-east-1",
                    "title": "Packet Drops"
                }
            }
        ]
    }'
```

### Step 7: Test and Verify Routing Behavior
**Objective**: Validate routing configuration works as expected

```bash
# Launch test instances in different subnets
SECURITY_GROUP=$(aws ec2 create-security-group \
    --group-name RouteTest-SG \
    --description "Route testing security group" \
    --vpc-id $VPC_ID \
    --query 'GroupId' --output text)

# Allow SSH and ICMP for testing
aws ec2 authorize-security-group-ingress --group-id $SECURITY_GROUP --protocol tcp --port 22 --cidr 10.0.0.0/16
aws ec2 authorize-security-group-ingress --group-id $SECURITY_GROUP --protocol icmp --port -1 --cidr 10.0.0.0/16

# Launch instances for testing
aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --count 1 \
    --instance-type t3.micro \
    --key-name your-key-pair \
    --security-group-ids $SECURITY_GROUP \
    --subnet-id $MGMT_SUBNET \
    --associate-public-ip-address \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Management-Test}]'

aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --count 1 \
    --instance-type t3.micro \
    --key-name your-key-pair \
    --security-group-ids $SECURITY_GROUP \
    --subnet-id $PRIVATE_APP_1A \
    --no-associate-public-ip-address \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=App-Test}]'
```

**Routing Tests**:
```bash
# From management instance, test routing
ssh -i your-key.pem ec2-user@[management-instance-ip]

# Test internet connectivity
curl -I https://aws.amazon.com

# Test internal connectivity
ping [app-instance-private-ip]

# From app instance (via management bastion)
ssh -i your-key.pem ec2-user@[app-instance-private-ip]

# Test NAT Gateway routing
curl -I https://aws.amazon.com  # Should work through NAT
```

## Route Table Analysis and Troubleshooting

### Common Routing Issues

#### 1. Route Conflicts
```bash
# Check for conflicting routes
aws ec2 describe-route-tables --route-table-ids $PRIVATE_RT_1A \
    --query 'RouteTables[0].Routes[?State==`blackhole`]'

# Identify overlapping CIDR blocks
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" \
    --query 'RouteTables[*].Routes[*].[DestinationCidrBlock,GatewayId]' \
    --output table
```

#### 2. Missing Routes
```bash
# Verify route table associations
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" \
    --query 'RouteTables[*].[RouteTableId,Associations[*].SubnetId]' \
    --output table

# Check for unassociated subnets (using main route table)
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" \
    --query 'Subnets[?!Associations || length(Associations)==`0`].[SubnetId,CidrBlock]' \
    --output table
```

### Route Table Best Practices

#### 1. Naming and Tagging Strategy
```bash
# Consistent tagging for route tables
aws ec2 create-tags --resources $PUBLIC_RT --tags \
    Key=Environment,Value=Production \
    Key=Tier,Value=Web \
    Key=Purpose,Value=Internet-Access \
    Key=Owner,Value=NetworkTeam
```

#### 2. Route Documentation
```bash
# Export route table configuration for documentation
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" \
    --query 'RouteTables[*].[RouteTableId,Tags[?Key==`Name`].Value|[0],Routes[*].[DestinationCidrBlock,GatewayId]]' \
    --output table > route-tables-documentation.txt
```

## Architecture Diagram
```
Internet
    |
[Internet Gateway]
    |
┌─────────────────────────────────────────────────────────────────────┐
│ VPC: 10.0.0.0/16                                                   │
│                                                                     │
│ ┌─────────────────┐    ┌─────────────────┐                         │
│ │ Public Subnets  │    │ Management      │                         │
│ │ 10.0.1.0/24     │    │ 10.0.100.0/24   │                         │
│ │ 10.0.2.0/24     │    │                 │                         │
│ │ [NAT-GW-1a]     │    │                 │                         │
│ │ [NAT-GW-1b]     │    │                 │                         │
│ └─────────────────┘    └─────────────────┘                         │
│         │                       │                                  │
│    [Public-RT]            [Management-RT]                          │
│    0.0.0.0/0→IGW          0.0.0.0/0→IGW                           │
│         │                       │                                  │
│ ┌─────────────────┐    ┌─────────────────┐                         │
│ │ Private Subnets │    │ Database Subnets│                         │
│ │ 10.0.11.0/24    │    │ 10.0.21.0/24    │                         │
│ │ 10.0.12.0/24    │    │ 10.0.22.0/24    │                         │
│ └─────────────────┘    └─────────────────┘                         │
│         │                       │                                  │
│  [Private-RT-1a]           [Main-RT]                               │
│  [Private-RT-1b]           (VPC local only)                        │
│  0.0.0.0/0→NAT-GW                                                  │
└─────────────────────────────────────────────────────────────────────┘
```

## Troubleshooting Guide

| Issue | Symptoms | Diagnosis | Solution |
|-------|----------|-----------|----------|
| No internet access | Connection timeouts | Check route to 0.0.0.0/0 | Add default route to IGW/NAT |
| Blackhole routes | Traffic dropped | Route status shows blackhole | Fix or remove failed target |
| Wrong route taken | Traffic goes unexpected path | Multiple overlapping routes | Use longest prefix matching rules |
| Subnet isolation | Cannot reach other subnets | Missing VPC local routes | Verify VPC CIDR routes exist |
| Route table limits | Cannot add more routes | 50 routes per table limit | Consolidate routes or create new table |

## Key Takeaways
- Route tables control all traffic flow within VPC
- Longest prefix matching determines route selection
- Main route table vs custom route tables serve different purposes
- Route propagation enables dynamic routing updates
- Proper route table design is critical for security and performance
- Multiple route tables enable sophisticated network architectures

## Best Practices
- Use descriptive names and consistent tagging
- Keep main route table for isolated/database subnets
- Create custom route tables for public and private subnets
- Document route table purposes and associations
- Monitor route table changes and blackhole routes
- Plan for route table limits (50 routes per table)
- Use route propagation for dynamic environments

## Additional Resources
- [VPC Route Tables Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
- [Route Table Limits](https://docs.aws.amazon.com/vpc/latest/userguide/amazon-vpc-limits.html#vpc-limits-route-tables)
- [VPC Routing](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)

## Exam Tips
- Understand longest prefix matching - most specific route wins
- Know route table association rules - explicit vs main route table
- Remember VPC local routes cannot be modified or removed
- Route propagation is used for VPN and Direct Connect
- Each subnet must be associated with exactly one route table
- Maximum 50 routes per route table, 200 route tables per VPC