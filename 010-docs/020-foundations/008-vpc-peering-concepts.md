# Topic 8: VPC Peering - Concepts, Use Cases, and Demo

## Prerequisites
- Completed Topics 1-7 (AWS Networking Foundations)
- Understanding of VPC routing and CIDR blocks
- Knowledge of cross-account and cross-region concepts

## Learning Objectives
By the end of this topic, you will be able to:
- Configure VPC peering connections within and across accounts/regions
- Design routing architectures using VPC peering
- Implement security best practices for peered VPCs
- Troubleshoot common VPC peering connectivity issues

## Theory

### VPC Peering Overview

#### Definition
VPC peering connection is a networking connection between two VPCs that enables routing traffic between them using private IPv4 or IPv6 addresses as if they were part of the same network.

#### Key Characteristics
- **Point-to-Point**: Direct connection between two VPCs only
- **Non-Transitive**: No routing through intermediate peered VPCs
- **Private Communication**: Traffic stays within AWS backbone
- **Cross-Account**: Can peer VPCs across different AWS accounts
- **Cross-Region**: Can peer VPCs in different AWS regions
- **No Bandwidth Limits**: Uses AWS backbone network capacity

### VPC Peering Types

#### 1. Intra-Region Peering
- **Same Region**: Both VPCs in same AWS region
- **Lower Latency**: Minimal latency between VPCs
- **No Data Transfer Charges**: Free data transfer within region
- **DNS Resolution**: Can enable DNS resolution across peering

#### 2. Inter-Region Peering
- **Different Regions**: VPCs in different AWS regions
- **Higher Latency**: Latency based on geographic distance
- **Data Transfer Charges**: Standard inter-region data transfer rates
- **Global Connectivity**: Enable global network architectures

#### 3. Cross-Account Peering
- **Different Accounts**: VPCs owned by different AWS accounts
- **Acceptance Required**: Peering request must be accepted
- **Shared Responsibility**: Both accounts manage their side
- **Security Boundaries**: Maintain account-level isolation

### VPC Peering Limitations

#### Technical Limitations
- **CIDR Overlap**: Cannot peer VPCs with overlapping CIDR blocks
- **Transitive Routing**: No transitive routing through peered VPCs
- **Edge-to-Edge**: No edge-to-edge routing via gateways
- **Placement Groups**: Cannot span across peered VPCs
- **DNS**: Limited DNS resolution options

#### Scaling Limitations
- **Point-to-Point Only**: Each peering connects exactly 2 VPCs
- **Management Complexity**: N*(N-1)/2 connections for full mesh
- **Route Table Updates**: Manual route updates required
- **No Central Management**: No single point of control

### VPC Peering vs Alternatives

| Feature | VPC Peering | Transit Gateway | PrivateLink |
|---------|-------------|-----------------|-------------|
| **Connectivity** | Point-to-point | Hub and spoke | Service-to-service |
| **Transitive Routing** | No | Yes | No |
| **Cross-Region** | Yes | Yes | Yes |
| **Bandwidth** | No limits | 50 Gbps per attachment | Depends on endpoint |
| **Cost** | Data transfer only | Hourly + data charges | Hourly + data charges |
| **Management** | Per connection | Centralized | Per endpoint |

## Lab Exercise: Comprehensive VPC Peering Implementation

### Lab Duration: 180 minutes

### Step 1: Create Multiple VPCs for Peering
**Objective**: Set up VPC infrastructure across regions and accounts

**Create VPCs in us-east-1**:
```bash
# VPC 1: Production environment
PROD_VPC=$(aws ec2 create-vpc --cidr-block 10.1.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Production-VPC},{Key=Environment,Value=Production}]' \
    --region us-east-1 \
    --query 'Vpc.VpcId' --output text)

# VPC 2: Development environment  
DEV_VPC=$(aws ec2 create-vpc --cidr-block 10.2.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Development-VPC},{Key=Environment,Value=Development}]' \
    --region us-east-1 \
    --query 'Vpc.VpcId' --output text)

# VPC 3: Shared services
SHARED_VPC=$(aws ec2 create-vpc --cidr-block 10.3.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Shared-Services-VPC},{Key=Environment,Value=Shared}]' \
    --region us-east-1 \
    --query 'Vpc.VpcId' --output text)

echo "Production VPC: $PROD_VPC"
echo "Development VPC: $DEV_VPC"
echo "Shared Services VPC: $SHARED_VPC"

# Enable DNS support and hostnames for all VPCs
for vpc in $PROD_VPC $DEV_VPC $SHARED_VPC; do
    aws ec2 modify-vpc-attribute --vpc-id $vpc --enable-dns-support --region us-east-1
    aws ec2 modify-vpc-attribute --vpc-id $vpc --enable-dns-hostnames --region us-east-1
done
```

**Create VPC in different region (us-west-2)**:
```bash
# VPC 4: Disaster recovery in different region
DR_VPC=$(aws ec2 create-vpc --cidr-block 10.4.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=DR-VPC},{Key=Environment,Value=DR}]' \
    --region us-west-2 \
    --query 'Vpc.VpcId' --output text)

aws ec2 modify-vpc-attribute --vpc-id $DR_VPC --enable-dns-support --region us-west-2
aws ec2 modify-vpc-attribute --vpc-id $DR_VPC --enable-dns-hostnames --region us-west-2

echo "DR VPC (us-west-2): $DR_VPC"
```

### Step 2: Create Subnets and Basic Infrastructure
**Objective**: Set up networking components in each VPC

**Create subnets in Production VPC**:
```bash
# Production VPC subnets
PROD_SUBNET=$(aws ec2 create-subnet --vpc-id $PROD_VPC --cidr-block 10.1.1.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Prod-Subnet}]' \
    --region us-east-1 --query 'Subnet.SubnetId' --output text)

# Development VPC subnets
DEV_SUBNET=$(aws ec2 create-subnet --vpc-id $DEV_VPC --cidr-block 10.2.1.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Dev-Subnet}]' \
    --region us-east-1 --query 'Subnet.SubnetId' --output text)

# Shared Services VPC subnets
SHARED_SUBNET=$(aws ec2 create-subnet --vpc-id $SHARED_VPC --cidr-block 10.3.1.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Shared-Subnet}]' \
    --region us-east-1 --query 'Subnet.SubnetId' --output text)

# DR VPC subnets
DR_SUBNET=$(aws ec2 create-subnet --vpc-id $DR_VPC --cidr-block 10.4.1.0/24 --availability-zone us-west-2a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=DR-Subnet}]' \
    --region us-west-2 --query 'Subnet.SubnetId' --output text)
```

**Create Internet Gateways (for testing connectivity)**:
```bash
# Internet Gateway for Production VPC
PROD_IGW=$(aws ec2 create-internet-gateway \
    --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=Prod-IGW}]' \
    --region us-east-1 --query 'InternetGateway.InternetGatewayId' --output text)

aws ec2 attach-internet-gateway --internet-gateway-id $PROD_IGW --vpc-id $PROD_VPC --region us-east-1

# Create route table for Production VPC
PROD_RT=$(aws ec2 create-route-table --vpc-id $PROD_VPC \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Prod-RT}]' \
    --region us-east-1 --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $PROD_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $PROD_IGW --region us-east-1
aws ec2 associate-route-table --route-table-id $PROD_RT --subnet-id $PROD_SUBNET --region us-east-1

# Repeat for other VPCs as needed for testing
```

### Step 3: Create Intra-Region VPC Peering Connections
**Objective**: Establish peering between VPCs in same region

**Create Peering Connections**:
```bash
# 1. Production <-> Shared Services peering
PROD_SHARED_PEER=$(aws ec2 create-vpc-peering-connection \
    --vpc-id $PROD_VPC \
    --peer-vpc-id $SHARED_VPC \
    --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=Prod-Shared-Peering}]' \
    --region us-east-1 \
    --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

# 2. Development <-> Shared Services peering
DEV_SHARED_PEER=$(aws ec2 create-vpc-peering-connection \
    --vpc-id $DEV_VPC \
    --peer-vpc-id $SHARED_VPC \
    --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=Dev-Shared-Peering}]' \
    --region us-east-1 \
    --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

echo "Production-Shared Peering: $PROD_SHARED_PEER"
echo "Development-Shared Peering: $DEV_SHARED_PEER"

# Accept peering connections
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $PROD_SHARED_PEER --region us-east-1
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $DEV_SHARED_PEER --region us-east-1

# Wait for peering connections to be active
aws ec2 wait vpc-peering-connection-exists --vpc-peering-connection-ids $PROD_SHARED_PEER $DEV_SHARED_PEER --region us-east-1
```

**Configure DNS Resolution**:
```bash
# Enable DNS resolution for peering connections
aws ec2 modify-vpc-peering-connection-options \
    --vpc-peering-connection-id $PROD_SHARED_PEER \
    --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
    --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
    --region us-east-1

aws ec2 modify-vpc-peering-connection-options \
    --vpc-peering-connection-id $DEV_SHARED_PEER \
    --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
    --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
    --region us-east-1
```

### Step 4: Configure Routing for Peered VPCs
**Objective**: Set up proper routing between peered VPCs

**Add Routes to Production VPC**:
```bash
# Route from Production to Shared Services
aws ec2 create-route \
    --route-table-id $PROD_RT \
    --destination-cidr-block 10.3.0.0/16 \
    --vpc-peering-connection-id $PROD_SHARED_PEER \
    --region us-east-1

echo "Added route from Production VPC to Shared Services VPC"
```

**Add Routes to Shared Services VPC**:
```bash
# Get main route table for Shared Services VPC
SHARED_RT=$(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$SHARED_VPC" "Name=association.main,Values=true" \
    --region us-east-1 \
    --query 'RouteTables[0].RouteTableId' --output text)

# Route from Shared Services to Production
aws ec2 create-route \
    --route-table-id $SHARED_RT \
    --destination-cidr-block 10.1.0.0/16 \
    --vpc-peering-connection-id $PROD_SHARED_PEER \
    --region us-east-1

# Route from Shared Services to Development
aws ec2 create-route \
    --route-table-id $SHARED_RT \
    --destination-cidr-block 10.2.0.0/16 \
    --vpc-peering-connection-id $DEV_SHARED_PEER \
    --region us-east-1

echo "Added routes from Shared Services VPC to Production and Development VPCs"
```

**Add Routes to Development VPC**:
```bash
# Get main route table for Development VPC
DEV_RT=$(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$DEV_VPC" "Name=association.main,Values=true" \
    --region us-east-1 \
    --query 'RouteTables[0].RouteTableId' --output text)

# Route from Development to Shared Services
aws ec2 create-route \
    --route-table-id $DEV_RT \
    --destination-cidr-block 10.3.0.0/16 \
    --vpc-peering-connection-id $DEV_SHARED_PEER \
    --region us-east-1

echo "Added route from Development VPC to Shared Services VPC"
```

### Step 5: Create Inter-Region Peering Connection
**Objective**: Establish cross-region VPC peering

**Create Cross-Region Peering**:
```bash
# Create peering connection from us-east-1 to us-west-2
CROSS_REGION_PEER=$(aws ec2 create-vpc-peering-connection \
    --vpc-id $PROD_VPC \
    --peer-vpc-id $DR_VPC \
    --peer-region us-west-2 \
    --tag-specifications 'ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=Prod-DR-CrossRegion-Peering}]' \
    --region us-east-1 \
    --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

echo "Cross-region peering connection: $CROSS_REGION_PEER"

# Accept peering connection in us-west-2
aws ec2 accept-vpc-peering-connection \
    --vpc-peering-connection-id $CROSS_REGION_PEER \
    --region us-west-2

# Wait for connection to be active
sleep 30

# Verify peering connection status
aws ec2 describe-vpc-peering-connections \
    --vpc-peering-connection-ids $CROSS_REGION_PEER \
    --region us-east-1 \
    --query 'VpcPeeringConnections[0].[VpcPeeringConnectionId,Status.Code]' \
    --output table
```

**Configure Cross-Region Routing**:
```bash
# Add route from Production VPC to DR VPC
aws ec2 create-route \
    --route-table-id $PROD_RT \
    --destination-cidr-block 10.4.0.0/16 \
    --vpc-peering-connection-id $CROSS_REGION_PEER \
    --region us-east-1

# Get DR VPC main route table
DR_RT=$(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$DR_VPC" "Name=association.main,Values=true" \
    --region us-west-2 \
    --query 'RouteTables[0].RouteTableId' --output text)

# Add route from DR VPC to Production VPC
aws ec2 create-route \
    --route-table-id $DR_RT \
    --destination-cidr-block 10.1.0.0/16 \
    --vpc-peering-connection-id $CROSS_REGION_PEER \
    --region us-west-2

echo "Cross-region routing configured"
```

### Step 6: Test VPC Peering Connectivity
**Objective**: Validate peering connections work correctly

**Launch Test Instances**:
```bash
# Create security group allowing ICMP and SSH
PEERING_SG_PROD=$(aws ec2 create-security-group \
    --group-name Peering-Test-SG \
    --description "Security group for peering connectivity testing" \
    --vpc-id $PROD_VPC \
    --region us-east-1 \
    --query 'GroupId' --output text)

# Allow SSH and ICMP from peered VPC CIDRs
aws ec2 authorize-security-group-ingress --group-id $PEERING_SG_PROD --protocol tcp --port 22 --cidr 10.0.0.0/8 --region us-east-1
aws ec2 authorize-security-group-ingress --group-id $PEERING_SG_PROD --protocol icmp --port -1 --cidr 10.0.0.0/8 --region us-east-1

# Launch test instance in Production VPC
PROD_INSTANCE=$(aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --count 1 \
    --instance-type t3.micro \
    --key-name your-key-pair \
    --security-group-ids $PEERING_SG_PROD \
    --subnet-id $PROD_SUBNET \
    --associate-public-ip-address \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Prod-Test-Instance}]' \
    --region us-east-1 \
    --query 'Instances[0].InstanceId' --output text)

# Create similar instances in other VPCs
PEERING_SG_SHARED=$(aws ec2 create-security-group \
    --group-name Peering-Test-SG \
    --description "Security group for peering connectivity testing" \
    --vpc-id $SHARED_VPC \
    --region us-east-1 \
    --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $PEERING_SG_SHARED --protocol tcp --port 22 --cidr 10.0.0.0/8 --region us-east-1
aws ec2 authorize-security-group-ingress --group-id $PEERING_SG_SHARED --protocol icmp --port -1 --cidr 10.0.0.0/8 --region us-east-1

SHARED_INSTANCE=$(aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --count 1 \
    --instance-type t3.micro \
    --key-name your-key-pair \
    --security-group-ids $PEERING_SG_SHARED \
    --subnet-id $SHARED_SUBNET \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Shared-Test-Instance}]' \
    --region us-east-1 \
    --query 'Instances[0].InstanceId' --output text)

# Wait for instances to be running
aws ec2 wait instance-running --instance-ids $PROD_INSTANCE $SHARED_INSTANCE --region us-east-1
```

**Test Connectivity**:
```bash
# Get instance private IPs
PROD_IP=$(aws ec2 describe-instances --instance-ids $PROD_INSTANCE --region us-east-1 \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

SHARED_IP=$(aws ec2 describe-instances --instance-ids $SHARED_INSTANCE --region us-east-1 \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

PROD_PUBLIC_IP=$(aws ec2 describe-instances --instance-ids $PROD_INSTANCE --region us-east-1 \
    --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

echo "Production Instance Private IP: $PROD_IP"
echo "Shared Instance Private IP: $SHARED_IP"
echo "Production Instance Public IP: $PROD_PUBLIC_IP"

echo "Test connectivity with:"
echo "ssh -i your-key.pem ec2-user@$PROD_PUBLIC_IP"
echo "From production instance, ping shared instance: ping $SHARED_IP"
```

### Step 7: Implement Advanced Peering Scenarios
**Objective**: Demonstrate complex peering architectures

**Hub-and-Spoke with Shared Services**:
```bash
# Verify routing configuration shows hub-and-spoke pattern
echo "=== Routing Analysis ==="
echo "Shared Services VPC acts as hub, connecting to:"
echo "- Production VPC (spoke)"
echo "- Development VPC (spoke)"
echo "- Note: Production and Development cannot communicate directly"

# Display routing tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$SHARED_VPC" --region us-east-1 \
    --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,VpcPeeringConnectionId]' \
    --output table
```

**Security Group Configuration for Peered VPCs**:
```bash
# Create security group referencing another VPC's security group
CROSS_VPC_SG=$(aws ec2 create-security-group \
    --group-name Cross-VPC-Reference-SG \
    --description "Security group with cross-VPC references" \
    --vpc-id $PROD_VPC \
    --region us-east-1 \
    --query 'GroupId' --output text)

# Allow traffic from specific security group in peered VPC
aws ec2 authorize-security-group-ingress \
    --group-id $CROSS_VPC_SG \
    --protocol tcp \
    --port 80 \
    --source-group $PEERING_SG_SHARED \
    --region us-east-1

echo "Created cross-VPC security group reference"
```

## Monitoring and Troubleshooting VPC Peering

### VPC Flow Logs for Peering Analysis
```bash
# Create VPC Flow Logs for peering traffic analysis
aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids $PROD_VPC $SHARED_VPC \
    --traffic-type ALL \
    --log-destination-type cloud-watch-logs \
    --log-group-name /aws/vpc/peering-flowlogs \
    --deliver-logs-permission-arn arn:aws:iam::ACCOUNT:role/flowlogsRole \
    --region us-east-1

# Create CloudWatch insights query for peering traffic
cat << 'EOF' > peering-analysis-query.txt
fields @timestamp, srcaddr, dstaddr, protocol, srcport, dstport, action
| filter srcaddr like /10\.1\./ and dstaddr like /10\.3\./
| filter action = "ACCEPT"
| stats count() by srcaddr, dstaddr
| sort count desc
EOF

echo "Use this CloudWatch Insights query to analyze peering traffic"
```

### Common Troubleshooting Commands
```bash
# Check peering connection status
aws ec2 describe-vpc-peering-connections \
    --filters "Name=status-code,Values=active" \
    --region us-east-1 \
    --query 'VpcPeeringConnections[*].[VpcPeeringConnectionId,RequesterVpcInfo.VpcId,AccepterVpcInfo.VpcId,Status.Code]' \
    --output table

# Verify route table entries
check_routes() {
    local vpc_id=$1
    local region=$2
    echo "=== Routes for VPC $vpc_id ==="
    aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$vpc_id" --region $region \
        --query 'RouteTables[*].Routes[?VpcPeeringConnectionId].[DestinationCidrBlock,VpcPeeringConnectionId,State]' \
        --output table
}

check_routes $PROD_VPC us-east-1
check_routes $SHARED_VPC us-east-1
check_routes $DEV_VPC us-east-1

# Check security group rules
aws ec2 describe-security-groups --group-ids $PEERING_SG_PROD --region us-east-1 \
    --query 'SecurityGroups[0].IpPermissions[*].[IpProtocol,FromPort,ToPort,IpRanges[0].CidrIp]' \
    --output table
```

## Architecture Diagram
```
┌─────────────────────────────────────────────────────────────────────┐
│ us-east-1                                                           │
│                                                                     │
│ ┌─────────────────┐         ┌─────────────────┐                     │
│ │ Production VPC  │◄────────┤ Shared Services │                     │
│ │ 10.1.0.0/16     │         │ VPC             │                     │
│ │                 │         │ 10.3.0.0/16     │                     │
│ └─────────────────┘         │                 │                     │
│                             │                 │                     │
│ ┌─────────────────┐         │                 │                     │
│ │ Development VPC │◄────────┤                 │                     │
│ │ 10.2.0.0/16     │         └─────────────────┘                     │
│ └─────────────────┘                                                 │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              │ Cross-Region Peering
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│ us-west-2                                                           │
│                                                                     │
│ ┌─────────────────┐                                                 │
│ │ DR VPC          │                                                 │
│ │ 10.4.0.0/16     │                                                 │
│ └─────────────────┘                                                 │
└─────────────────────────────────────────────────────────────────────┘
```

## Best Practices and Limitations

### VPC Peering Best Practices
1. **CIDR Planning**: Ensure non-overlapping CIDR blocks
2. **Security Groups**: Use cross-VPC security group references when possible
3. **DNS Resolution**: Enable DNS resolution for peered VPCs
4. **Monitoring**: Set up VPC Flow Logs for peering traffic analysis
5. **Documentation**: Maintain clear documentation of peering relationships

### When to Use VPC Peering
- **Simple Connectivity**: Direct connection between two VPCs
- **Low Latency**: Minimal network overhead required
- **Cost Efficiency**: No additional charges for data transfer within region
- **Security**: Private connectivity without internet exposure

### When NOT to Use VPC Peering
- **Complex Topologies**: More than a few VPCs (consider Transit Gateway)
- **Transitive Routing**: Need for hub-and-spoke with transitive routing
- **Frequent Changes**: Dynamic network topologies
- **Service Connectivity**: Accessing specific services (consider PrivateLink)

## Troubleshooting Guide

| Issue | Symptoms | Diagnosis | Solution |
|-------|----------|-----------|----------|
| Peering connection failed | Connection in "failed" state | Check CIDR overlap | Use non-overlapping CIDR blocks |
| Cannot reach peered VPC | Connectivity timeout | Check route tables | Add routes for peered VPC CIDR |
| DNS resolution not working | Cannot resolve hostnames | DNS options not enabled | Enable DNS resolution in peering options |
| Security group rules not working | Traffic blocked | Incorrect references | Use VPC-specific security group IDs |
| Cross-region latency high | Slow response times | Geographic distance | Consider regional architecture |

## Key Takeaways
- VPC peering enables private connectivity between VPCs
- Peering is point-to-point and non-transitive
- Cross-region and cross-account peering is supported
- Proper CIDR planning is essential for peering
- Route tables must be updated for each peering connection
- Security groups can reference other VPC security groups
- Consider Transit Gateway for complex topologies

## Additional Resources
- [VPC Peering Guide](https://docs.aws.amazon.com/vpc/latest/peering/)
- [VPC Peering Scenarios](https://docs.aws.amazon.com/vpc/latest/peering/peering-scenarios.html)
- [Cross-Region VPC Peering](https://docs.aws.amazon.com/vpc/latest/peering/cross-region-peering.html)

## Exam Tips
- VPC peering is not transitive - no routing through intermediate VPCs
- Both VPCs must have non-overlapping CIDR blocks
- Route tables must be updated on both sides of peering connection
- Cross-region peering incurs data transfer charges
- DNS resolution must be explicitly enabled for peered VPCs
- Maximum 125 peering connections per VPC
- Edge-to-edge routing is not supported (no routing via IGW, VPN, etc.)