# Topic 20: AWS Transit Gateway - Full Walkthrough

## Prerequisites
- Completed Foundation Module (Topics 1-10)
- Understanding of VPC peering concepts (Topic 8)
- Knowledge of hybrid networking concepts (Topics 11-19)
- Familiarity with routing protocols and network design

## Learning Objectives
By the end of this topic, you will be able to:
- Design and implement comprehensive Transit Gateway architectures
- Configure TGW attachments for VPCs, VPN, and Direct Connect
- Create and manage TGW route tables for network segmentation
- Implement cross-region TGW connectivity and global architectures

## Theory

### AWS Transit Gateway Overview

#### Definition
AWS Transit Gateway is a network transit hub that connects VPCs and on-premises networks through a central hub. It acts as a cloud router, simplifying network architecture and management.

#### Key Characteristics
- **Central Hub**: Single point for connecting multiple networks
- **Scalable**: Supports thousands of VPCs and connections
- **Transitive Routing**: Enables communication between attached networks
- **Route Tables**: Granular control over traffic flow and segmentation
- **Multi-Region**: Cross-region peering for global connectivity

### Transit Gateway Architecture

#### Hub-and-Spoke Model
```
        [Transit Gateway]
         /    |    |    \
    VPC-A   VPC-B  VPN  Direct Connect
                    |         |
              On-Premises  On-Premises
                Network A   Network B
```

#### Components Overview
- **Transit Gateway**: Central routing hub
- **Attachments**: VPCs, VPN connections, Direct Connect gateways
- **Route Tables**: Control traffic flow between attachments
- **Associations**: Link attachments to route tables
- **Propagations**: Automatic route learning from attachments

### Transit Gateway Attachments

#### 1. VPC Attachments
- **Purpose**: Connect VPCs to Transit Gateway
- **Subnet Requirement**: At least one subnet per AZ
- **Route Integration**: Routes to TGW appear in VPC route tables
- **DNS Resolution**: Cross-VPC DNS resolution support

#### 2. VPN Attachments
- **Purpose**: Connect on-premises networks via VPN
- **BGP Support**: Dynamic routing with BGP
- **High Availability**: Multiple VPN connections supported
- **Bandwidth**: Up to 1.25 Gbps per tunnel

#### 3. Direct Connect Gateway Attachments
- **Purpose**: Connect on-premises via dedicated connections
- **Multi-Region**: Single DX location serves multiple regions
- **High Bandwidth**: Up to 100 Gbps per connection
- **Consistent Performance**: Predictable latency and throughput

#### 4. Transit Gateway Peering
- **Purpose**: Connect Transit Gateways across regions
- **Global Reach**: Worldwide connectivity
- **Route Control**: Selective route sharing between regions
- **Bandwidth**: Up to 50 Gbps between regions

### Route Tables and Routing

#### Default Route Table
- **Automatic**: Created with Transit Gateway
- **Default Behavior**: All attachments associated and propagated
- **Full Mesh**: All networks can communicate with each other
- **Simple Setup**: Good for basic connectivity needs

#### Custom Route Tables
- **Segmentation**: Create isolated network segments
- **Selective Routing**: Control which networks communicate
- **Security**: Implement micro-segmentation
- **Compliance**: Meet regulatory requirements

#### Route Propagation
- **Automatic**: Routes learned from BGP
- **Dynamic**: Updates when networks change
- **Selective**: Choose which routes to learn
- **Filtering**: Control route advertisements

## Lab Exercise: Comprehensive Transit Gateway Implementation

### Lab Duration: 300 minutes

### Step 1: Design Transit Gateway Architecture
**Objective**: Plan comprehensive TGW solution for enterprise needs

**Architecture Planning**:
```bash
# Create architecture design script
cat << 'EOF' > design-tgw-architecture.sh
#!/bin/bash

echo "=== Transit Gateway Architecture Design ==="
echo ""

echo "ENTERPRISE NETWORK REQUIREMENTS:"
echo "- Production VPCs: 3 (Web, App, Database tiers)"
echo "- Development VPCs: 2 (Dev, Test environments)"
echo "- Shared Services VPC: 1 (Directory, Monitoring)"
echo "- On-premises connectivity: VPN and Direct Connect"
echo "- Network segmentation: Production isolated from Dev"
echo "- Shared services accessible from all environments"
echo ""

echo "TRANSIT GATEWAY DESIGN:"
cat << 'ARCHITECTURE'

┌─────────────────────────────────────────────────────────────────┐
│                    Transit Gateway Hub                         │
│                                                                 │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │  Production RT  │  │ Development RT  │  │ Shared Svcs RT  │  │
│  │                 │  │                 │  │                 │  │
│  │ • Prod VPCs     │  │ • Dev VPCs      │  │ • Shared VPC    │  │
│  │ • Shared Svcs   │  │ • Shared Svcs   │  │ • All VPCs      │  │
│  │ • On-premises   │  │ • Limited       │  │ • On-premises   │  │
│  │   (full access) │  │   on-premises   │  │   (managed)     │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│           │                     │                     │         │
└─────────────────────────────────────────────────────────────────┘
            │                     │                     │
   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
   │ Production VPCs │   │Development VPCs │   │ Shared Services │
   │ • Web Tier      │   │ • Dev VPC       │   │ • Active Dir    │
   │ • App Tier      │   │ • Test VPC      │   │ • Monitoring    │
   │ • Database      │   └─────────────────┘   │ • Logging       │
   └─────────────────┘                         └─────────────────┘
            │                                           │
   ┌─────────────────┐                         ┌─────────────────┐
   │ On-Premises     │                         │ Management      │
   │ • Data Center   │                         │ • Bastion Hosts │
   │ • Branch Office │                         │ • Admin Tools   │
   └─────────────────┘                         └─────────────────┘

ARCHITECTURE

echo ""
echo "ROUTE TABLE STRATEGY:"
echo "1. Production Route Table:"
echo "   - Production VPCs can communicate with each other"
echo "   - Access to shared services"
echo "   - Full on-premises connectivity"
echo "   - NO access to development environments"
echo ""
echo "2. Development Route Table:"
echo "   - Development VPCs can communicate with each other"
echo "   - Access to shared services"
echo "   - Limited on-premises connectivity (dev/test only)"
echo "   - NO access to production environments"
echo ""
echo "3. Shared Services Route Table:"
echo "   - Can reach all VPCs (production and development)"
echo "   - Full on-premises connectivity for management"
echo "   - Central hub for cross-environment services"

EOF

chmod +x design-tgw-architecture.sh
./design-tgw-architecture.sh
```

### Step 2: Create Transit Gateway
**Objective**: Set up the central routing hub

```bash
# Create Transit Gateway
cat << 'EOF' > create-transit-gateway.sh
#!/bin/bash

echo "=== Creating Transit Gateway ==="
echo ""

TGW_NAME="Enterprise-Transit-Gateway"
AMAZON_ASN="64512"
DEFAULT_ROUTE_TABLE="enable"
DEFAULT_ROUTE_PROPAGATION="enable"

echo "Creating Transit Gateway..."
echo "Name: $TGW_NAME"
echo "Amazon ASN: $AMAZON_ASN"
echo "Default Route Table Association: $DEFAULT_ROUTE_TABLE"
echo "Default Route Table Propagation: $DEFAULT_ROUTE_PROPAGATION"
echo ""

# Create Transit Gateway
TGW_ID=$(aws ec2 create-transit-gateway \
    --description "Enterprise Transit Gateway for hub-and-spoke architecture" \
    --options AmazonSideAsn=$AMAZON_ASN,DefaultRouteTableAssociation=$DEFAULT_ROUTE_TABLE,DefaultRouteTablePropagation=$DEFAULT_ROUTE_PROPAGATION \
    --tag-specifications "ResourceType=transit-gateway,Tags=[{Key=Name,Value=$TGW_NAME},{Key=Environment,Value=Production}]" \
    --query 'TransitGateway.TransitGatewayId' \
    --output text)

echo "✅ Transit Gateway created: $TGW_ID"
echo ""

# Wait for TGW to be available
echo "Waiting for Transit Gateway to become available..."
aws ec2 wait transit-gateway-available --transit-gateway-ids $TGW_ID

echo "✅ Transit Gateway is now available"
echo ""

# Get default route table ID
DEFAULT_TGW_RT=$(aws ec2 describe-transit-gateways \
    --transit-gateway-ids $TGW_ID \
    --query 'TransitGateways[0].Options.AssociationDefaultRouteTableId' \
    --output text)

echo "Default Route Table ID: $DEFAULT_TGW_RT"

# Save configuration for later use
cat << CONFIG > tgw-config.env
export TGW_ID=$TGW_ID
export DEFAULT_TGW_RT=$DEFAULT_TGW_RT
export TGW_NAME="$TGW_NAME"
CONFIG

echo "Configuration saved to tgw-config.env"

EOF

chmod +x create-transit-gateway.sh
./create-transit-gateway.sh
```

### Step 3: Create Custom Route Tables
**Objective**: Set up network segmentation with custom route tables

```bash
# Create custom route tables for segmentation
cat << 'EOF' > create-tgw-route-tables.sh
#!/bin/bash

source tgw-config.env

echo "=== Creating Transit Gateway Route Tables ==="
echo ""

echo "Creating custom route tables for network segmentation..."
echo ""

# 1. Production Route Table
PROD_RT_NAME="Production-Route-Table"
PROD_TGW_RT=$(aws ec2 create-transit-gateway-route-table \
    --transit-gateway-id $TGW_ID \
    --tag-specifications "ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=$PROD_RT_NAME},{Key=Environment,Value=Production}]" \
    --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' \
    --output text)

echo "✅ Production Route Table created: $PROD_TGW_RT"

# 2. Development Route Table
DEV_RT_NAME="Development-Route-Table"
DEV_TGW_RT=$(aws ec2 create-transit-gateway-route-table \
    --transit-gateway-id $TGW_ID \
    --tag-specifications "ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=$DEV_RT_NAME},{Key=Environment,Value=Development}]" \
    --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' \
    --output text)

echo "✅ Development Route Table created: $DEV_TGW_RT"

# 3. Shared Services Route Table
SHARED_RT_NAME="Shared-Services-Route-Table"
SHARED_TGW_RT=$(aws ec2 create-transit-gateway-route-table \
    --transit-gateway-id $TGW_ID \
    --tag-specifications "ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=$SHARED_RT_NAME},{Key=Environment,Value=Shared}]" \
    --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' \
    --output text)

echo "✅ Shared Services Route Table created: $SHARED_TGW_RT"

# 4. On-Premises Route Table (for hybrid connectivity)
ONPREM_RT_NAME="OnPremises-Route-Table"
ONPREM_TGW_RT=$(aws ec2 create-transit-gateway-route-table \
    --transit-gateway-id $TGW_ID \
    --tag-specifications "ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=$ONPREM_RT_NAME},{Key=Environment,Value=Hybrid}]" \
    --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' \
    --output text)

echo "✅ On-Premises Route Table created: $ONPREM_TGW_RT"

echo ""
echo "Route table summary:"
echo "Production RT: $PROD_TGW_RT"
echo "Development RT: $DEV_TGW_RT"
echo "Shared Services RT: $SHARED_TGW_RT"
echo "On-Premises RT: $ONPREM_TGW_RT"

# Update configuration file
cat << CONFIG >> tgw-config.env
export PROD_TGW_RT=$PROD_TGW_RT
export DEV_TGW_RT=$DEV_TGW_RT
export SHARED_TGW_RT=$SHARED_TGW_RT
export ONPREM_TGW_RT=$ONPREM_TGW_RT
CONFIG

echo ""
echo "Route table IDs saved to tgw-config.env"

EOF

chmod +x create-tgw-route-tables.sh
./create-tgw-route-tables.sh
```

### Step 4: Create VPCs for Multi-Environment Setup
**Objective**: Build VPC infrastructure for enterprise deployment

```bash
# Create VPCs for different environments
cat << 'EOF' > create-enterprise-vpcs.sh
#!/bin/bash

echo "=== Creating Enterprise VPC Infrastructure ==="
echo ""

# Production VPCs
echo "Creating Production VPCs..."

# Production Web Tier VPC
PROD_WEB_VPC=$(aws ec2 create-vpc --cidr-block 10.10.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Prod-Web-VPC},{Key=Environment,Value=Production},{Key=Tier,Value=Web}]' \
    --query 'Vpc.VpcId' --output text)

aws ec2 modify-vpc-attribute --vpc-id $PROD_WEB_VPC --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $PROD_WEB_VPC --enable-dns-hostnames

# Production App Tier VPC
PROD_APP_VPC=$(aws ec2 create-vpc --cidr-block 10.11.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Prod-App-VPC},{Key=Environment,Value=Production},{Key=Tier,Value=Application}]' \
    --query 'Vpc.VpcId' --output text)

aws ec2 modify-vpc-attribute --vpc-id $PROD_APP_VPC --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $PROD_APP_VPC --enable-dns-hostnames

# Production Database VPC
PROD_DB_VPC=$(aws ec2 create-vpc --cidr-block 10.12.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Prod-DB-VPC},{Key=Environment,Value=Production},{Key=Tier,Value=Database}]' \
    --query 'Vpc.VpcId' --output text)

aws ec2 modify-vpc-attribute --vpc-id $PROD_DB_VPC --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $PROD_DB_VPC --enable-dns-hostnames

echo "✅ Production VPCs created:"
echo "  Web VPC: $PROD_WEB_VPC"
echo "  App VPC: $PROD_APP_VPC"
echo "  DB VPC: $PROD_DB_VPC"
echo ""

# Development VPCs
echo "Creating Development VPCs..."

# Development VPC
DEV_VPC=$(aws ec2 create-vpc --cidr-block 10.20.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Dev-VPC},{Key=Environment,Value=Development}]' \
    --query 'Vpc.VpcId' --output text)

aws ec2 modify-vpc-attribute --vpc-id $DEV_VPC --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $DEV_VPC --enable-dns-hostnames

# Test VPC
TEST_VPC=$(aws ec2 create-vpc --cidr-block 10.21.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Test-VPC},{Key=Environment,Value=Testing}]' \
    --query 'Vpc.VpcId' --output text)

aws ec2 modify-vpc-attribute --vpc-id $TEST_VPC --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $TEST_VPC --enable-dns-hostnames

echo "✅ Development VPCs created:"
echo "  Dev VPC: $DEV_VPC"
echo "  Test VPC: $TEST_VPC"
echo ""

# Shared Services VPC
echo "Creating Shared Services VPC..."

SHARED_VPC=$(aws ec2 create-vpc --cidr-block 10.100.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Shared-Services-VPC},{Key=Environment,Value=Shared}]' \
    --query 'Vpc.VpcId' --output text)

aws ec2 modify-vpc-attribute --vpc-id $SHARED_VPC --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $SHARED_VPC --enable-dns-hostnames

echo "✅ Shared Services VPC created: $SHARED_VPC"
echo ""

# Create subnets for TGW attachments
echo "Creating subnets for Transit Gateway attachments..."

create_tgw_subnet() {
    local vpc_id=$1
    local cidr=$2
    local name=$3
    
    aws ec2 create-subnet --vpc-id $vpc_id --cidr-block $cidr --availability-zone us-east-1a \
        --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=$name}]" \
        --query 'Subnet.SubnetId' --output text
}

# Create TGW attachment subnets
PROD_WEB_SUBNET=$(create_tgw_subnet $PROD_WEB_VPC "10.10.1.0/24" "Prod-Web-TGW-Subnet")
PROD_APP_SUBNET=$(create_tgw_subnet $PROD_APP_VPC "10.11.1.0/24" "Prod-App-TGW-Subnet")
PROD_DB_SUBNET=$(create_tgw_subnet $PROD_DB_VPC "10.12.1.0/24" "Prod-DB-TGW-Subnet")
DEV_SUBNET=$(create_tgw_subnet $DEV_VPC "10.20.1.0/24" "Dev-TGW-Subnet")
TEST_SUBNET=$(create_tgw_subnet $TEST_VPC "10.21.1.0/24" "Test-TGW-Subnet")
SHARED_SUBNET=$(create_tgw_subnet $SHARED_VPC "10.100.1.0/24" "Shared-TGW-Subnet")

echo "✅ TGW attachment subnets created"

# Save VPC configuration
cat << CONFIG > vpc-config.env
export PROD_WEB_VPC=$PROD_WEB_VPC
export PROD_APP_VPC=$PROD_APP_VPC
export PROD_DB_VPC=$PROD_DB_VPC
export DEV_VPC=$DEV_VPC
export TEST_VPC=$TEST_VPC
export SHARED_VPC=$SHARED_VPC
export PROD_WEB_SUBNET=$PROD_WEB_SUBNET
export PROD_APP_SUBNET=$PROD_APP_SUBNET
export PROD_DB_SUBNET=$PROD_DB_SUBNET
export DEV_SUBNET=$DEV_SUBNET
export TEST_SUBNET=$TEST_SUBNET
export SHARED_SUBNET=$SHARED_SUBNET
CONFIG

echo "VPC configuration saved to vpc-config.env"

EOF

chmod +x create-enterprise-vpcs.sh
./create-enterprise-vpcs.sh
```

### Step 5: Attach VPCs to Transit Gateway
**Objective**: Connect all VPCs to the central hub

```bash
# Attach VPCs to Transit Gateway
cat << 'EOF' > attach-vpcs-to-tgw.sh
#!/bin/bash

source tgw-config.env
source vpc-config.env

echo "=== Attaching VPCs to Transit Gateway ==="
echo ""

# Function to create TGW VPC attachment
create_tgw_attachment() {
    local vpc_id=$1
    local subnet_id=$2
    local attachment_name=$3
    local environment=$4
    
    echo "Creating attachment for $attachment_name..."
    
    attachment_id=$(aws ec2 create-transit-gateway-vpc-attachment \
        --transit-gateway-id $TGW_ID \
        --vpc-id $vpc_id \
        --subnet-ids $subnet_id \
        --tag-specifications "ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=$attachment_name},{Key=Environment,Value=$environment}]" \
        --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' \
        --output text)
    
    echo "✅ Attachment created: $attachment_id"
    echo $attachment_id
}

# Create VPC attachments
echo "Creating VPC attachments..."

PROD_WEB_ATTACH=$(create_tgw_attachment $PROD_WEB_VPC $PROD_WEB_SUBNET "Prod-Web-TGW-Attachment" "Production")
PROD_APP_ATTACH=$(create_tgw_attachment $PROD_APP_VPC $PROD_APP_SUBNET "Prod-App-TGW-Attachment" "Production")
PROD_DB_ATTACH=$(create_tgw_attachment $PROD_DB_VPC $PROD_DB_SUBNET "Prod-DB-TGW-Attachment" "Production")
DEV_ATTACH=$(create_tgw_attachment $DEV_VPC $DEV_SUBNET "Dev-TGW-Attachment" "Development")
TEST_ATTACH=$(create_tgw_attachment $TEST_VPC $TEST_SUBNET "Test-TGW-Attachment" "Testing")
SHARED_ATTACH=$(create_tgw_attachment $SHARED_VPC $SHARED_SUBNET "Shared-TGW-Attachment" "Shared")

echo ""
echo "Waiting for all attachments to become available..."

# Wait for all attachments to be available
aws ec2 wait transit-gateway-attachment-available --transit-gateway-attachment-ids \
    $PROD_WEB_ATTACH $PROD_APP_ATTACH $PROD_DB_ATTACH $DEV_ATTACH $TEST_ATTACH $SHARED_ATTACH

echo "✅ All VPC attachments are now available"
echo ""

echo "Attachment Summary:"
echo "Production Web: $PROD_WEB_ATTACH"
echo "Production App: $PROD_APP_ATTACH"
echo "Production DB: $PROD_DB_ATTACH"
echo "Development: $DEV_ATTACH"
echo "Testing: $TEST_ATTACH"
echo "Shared Services: $SHARED_ATTACH"

# Save attachment configuration
cat << CONFIG > attachment-config.env
export PROD_WEB_ATTACH=$PROD_WEB_ATTACH
export PROD_APP_ATTACH=$PROD_APP_ATTACH
export PROD_DB_ATTACH=$PROD_DB_ATTACH
export DEV_ATTACH=$DEV_ATTACH
export TEST_ATTACH=$TEST_ATTACH
export SHARED_ATTACH=$SHARED_ATTACH
CONFIG

echo ""
echo "Attachment configuration saved to attachment-config.env"

EOF

chmod +x attach-vpcs-to-tgw.sh
./attach-vpcs-to-tgw.sh
```

### Step 6: Configure Route Table Associations
**Objective**: Implement network segmentation through route table associations

```bash
# Configure route table associations for segmentation
cat << 'EOF' > configure-tgw-associations.sh
#!/bin/bash

source tgw-config.env
source attachment-config.env

echo "=== Configuring Transit Gateway Route Table Associations ==="
echo ""

echo "Implementing network segmentation strategy..."
echo ""

# Function to associate attachment with route table
associate_attachment() {
    local attachment_id=$1
    local route_table_id=$2
    local description=$3
    
    echo "Associating $description..."
    
    # First disassociate from default route table
    aws ec2 disassociate-transit-gateway-route-table \
        --transit-gateway-attachment-id $attachment_id \
        --transit-gateway-route-table-id $DEFAULT_TGW_RT >/dev/null 2>&1
    
    # Associate with custom route table
    aws ec2 associate-transit-gateway-route-table \
        --transit-gateway-attachment-id $attachment_id \
        --transit-gateway-route-table-id $route_table_id
    
    if [ $? -eq 0 ]; then
        echo "✅ $description associated successfully"
    else
        echo "❌ Failed to associate $description"
    fi
}

echo "1. PRODUCTION ENVIRONMENT ASSOCIATIONS"
echo "Associating production VPCs with Production Route Table..."

associate_attachment $PROD_WEB_ATTACH $PROD_TGW_RT "Production Web VPC"
associate_attachment $PROD_APP_ATTACH $PROD_TGW_RT "Production App VPC"
associate_attachment $PROD_DB_ATTACH $PROD_TGW_RT "Production DB VPC"

echo ""
echo "2. DEVELOPMENT ENVIRONMENT ASSOCIATIONS"
echo "Associating development VPCs with Development Route Table..."

associate_attachment $DEV_ATTACH $DEV_TGW_RT "Development VPC"
associate_attachment $TEST_ATTACH $DEV_TGW_RT "Test VPC"

echo ""
echo "3. SHARED SERVICES ASSOCIATIONS"
echo "Associating shared services with Shared Services Route Table..."

associate_attachment $SHARED_ATTACH $SHARED_TGW_RT "Shared Services VPC"

echo ""
echo "✅ All route table associations completed"
echo ""

echo "Network Segmentation Summary:"
echo "=============================="
echo "Production Route Table ($PROD_TGW_RT):"
echo "  - Production Web VPC"
echo "  - Production App VPC"
echo "  - Production Database VPC"
echo ""
echo "Development Route Table ($DEV_TGW_RT):"
echo "  - Development VPC"
echo "  - Test VPC"
echo ""
echo "Shared Services Route Table ($SHARED_TGW_RT):"
echo "  - Shared Services VPC"
echo ""

# Verify associations
echo "Verifying associations..."
aws ec2 get-transit-gateway-route-table-associations \
    --transit-gateway-route-table-id $PROD_TGW_RT \
    --query 'Associations[*].[TransitGatewayAttachmentId,State]' \
    --output table

EOF

chmod +x configure-tgw-associations.sh
./configure-tgw-associations.sh
```

### Step 7: Configure Route Propagation
**Objective**: Set up automatic route learning between network segments

```bash
# Configure route propagation for selective connectivity
cat << 'EOF' > configure-tgw-propagation.sh
#!/bin/bash

source tgw-config.env
source attachment-config.env

echo "=== Configuring Transit Gateway Route Propagation ==="
echo ""

echo "Setting up selective route propagation for network segmentation..."
echo ""

# Function to enable route propagation
enable_propagation() {
    local attachment_id=$1
    local route_table_id=$2
    local description=$3
    
    echo "Enabling propagation: $description"
    
    aws ec2 enable-transit-gateway-route-table-propagation \
        --transit-gateway-attachment-id $attachment_id \
        --transit-gateway-route-table-id $route_table_id
    
    if [ $? -eq 0 ]; then
        echo "✅ Propagation enabled for $description"
    else
        echo "❌ Failed to enable propagation for $description"
    fi
}

echo "1. PRODUCTION ROUTE TABLE PROPAGATION"
echo "Production VPCs can communicate with each other and shared services..."

# Production VPCs propagate to production route table
enable_propagation $PROD_WEB_ATTACH $PROD_TGW_RT "Prod Web → Prod RT"
enable_propagation $PROD_APP_ATTACH $PROD_TGW_RT "Prod App → Prod RT"
enable_propagation $PROD_DB_ATTACH $PROD_TGW_RT "Prod DB → Prod RT"

# Shared services propagate to production route table
enable_propagation $SHARED_ATTACH $PROD_TGW_RT "Shared → Prod RT"

echo ""
echo "2. DEVELOPMENT ROUTE TABLE PROPAGATION"
echo "Development VPCs can communicate with each other and shared services..."

# Development VPCs propagate to development route table
enable_propagation $DEV_ATTACH $DEV_TGW_RT "Dev → Dev RT"
enable_propagation $TEST_ATTACH $DEV_TGW_RT "Test → Dev RT"

# Shared services propagate to development route table
enable_propagation $SHARED_ATTACH $DEV_TGW_RT "Shared → Dev RT"

echo ""
echo "3. SHARED SERVICES ROUTE TABLE PROPAGATION"
echo "Shared services can reach all environments..."

# All VPCs propagate to shared services route table
enable_propagation $PROD_WEB_ATTACH $SHARED_TGW_RT "Prod Web → Shared RT"
enable_propagation $PROD_APP_ATTACH $SHARED_TGW_RT "Prod App → Shared RT"
enable_propagation $PROD_DB_ATTACH $SHARED_TGW_RT "Prod DB → Shared RT"
enable_propagation $DEV_ATTACH $SHARED_TGW_RT "Dev → Shared RT"
enable_propagation $TEST_ATTACH $SHARED_TGW_RT "Test → Shared RT"
enable_propagation $SHARED_ATTACH $SHARED_TGW_RT "Shared → Shared RT"

echo ""
echo "✅ Route propagation configuration completed"
echo ""

echo "Propagation Summary:"
echo "==================="
echo "Production Environment:"
echo "  - Can communicate within production VPCs"
echo "  - Can access shared services"
echo "  - CANNOT access development environment"
echo ""
echo "Development Environment:"
echo "  - Can communicate within development VPCs"
echo "  - Can access shared services"
echo "  - CANNOT access production environment"
echo ""
echo "Shared Services:"
echo "  - Can reach all VPCs in all environments"
echo "  - Provides central services (DNS, monitoring, etc.)"

EOF

chmod +x configure-tgw-propagation.sh
./configure-tgw-propagation.sh
```

### Step 8: Update VPC Route Tables
**Objective**: Configure VPC routing to use Transit Gateway

```bash
# Update VPC route tables to use Transit Gateway
cat << 'EOF' > update-vpc-routing.sh
#!/bin/bash

source tgw-config.env
source vpc-config.env

echo "=== Updating VPC Route Tables for Transit Gateway ==="
echo ""

# Function to update VPC route tables
update_vpc_routes() {
    local vpc_id=$1
    local vpc_name=$2
    
    echo "Updating routes for $vpc_name ($vpc_id)..."
    
    # Get all route tables for the VPC
    route_tables=$(aws ec2 describe-route-tables \
        --filters "Name=vpc-id,Values=$vpc_id" \
        --query 'RouteTables[*].RouteTableId' \
        --output text)
    
    for rt_id in $route_tables; do
        echo "  Updating route table: $rt_id"
        
        # Add routes to other VPC CIDRs via Transit Gateway
        # Production VPC CIDRs
        aws ec2 create-route --route-table-id $rt_id --destination-cidr-block 10.10.0.0/16 --transit-gateway-id $TGW_ID 2>/dev/null || true
        aws ec2 create-route --route-table-id $rt_id --destination-cidr-block 10.11.0.0/16 --transit-gateway-id $TGW_ID 2>/dev/null || true
        aws ec2 create-route --route-table-id $rt_id --destination-cidr-block 10.12.0.0/16 --transit-gateway-id $TGW_ID 2>/dev/null || true
        
        # Development VPC CIDRs
        aws ec2 create-route --route-table-id $rt_id --destination-cidr-block 10.20.0.0/16 --transit-gateway-id $TGW_ID 2>/dev/null || true
        aws ec2 create-route --route-table-id $rt_id --destination-cidr-block 10.21.0.0/16 --transit-gateway-id $TGW_ID 2>/dev/null || true
        
        # Shared Services VPC CIDR
        aws ec2 create-route --route-table-id $rt_id --destination-cidr-block 10.100.0.0/16 --transit-gateway-id $TGW_ID 2>/dev/null || true
        
        # On-premises networks (example)
        aws ec2 create-route --route-table-id $rt_id --destination-cidr-block 192.168.0.0/16 --transit-gateway-id $TGW_ID 2>/dev/null || true
    done
    
    echo "✅ Routes updated for $vpc_name"
}

echo "Updating VPC route tables to use Transit Gateway..."
echo ""

# Update all VPC route tables
update_vpc_routes $PROD_WEB_VPC "Production Web VPC"
update_vpc_routes $PROD_APP_VPC "Production App VPC"
update_vpc_routes $PROD_DB_VPC "Production DB VPC"
update_vpc_routes $DEV_VPC "Development VPC"
update_vpc_routes $TEST_VPC "Test VPC"
update_vpc_routes $SHARED_VPC "Shared Services VPC"

echo ""
echo "✅ All VPC route tables updated"
echo ""

echo "Route Configuration Summary:"
echo "============================"
echo "All VPCs now have routes to:"
echo "  - Other VPC CIDRs via Transit Gateway"
echo "  - On-premises networks via Transit Gateway"
echo "  - Actual connectivity controlled by TGW route tables"
echo ""

echo "Testing connectivity (example commands):"
echo "From Production Web VPC:"
echo "  - Can reach: Production App/DB, Shared Services"
echo "  - Cannot reach: Development, Test (blocked by TGW route tables)"
echo ""
echo "From Development VPC:"
echo "  - Can reach: Test VPC, Shared Services"
echo "  - Cannot reach: Production VPCs (blocked by TGW route tables)"

EOF

chmod +x update-vpc-routing.sh
./update-vpc-routing.sh
```

### Step 9: Add Hybrid Connectivity
**Objective**: Connect on-premises networks via VPN and Direct Connect

```bash
# Add hybrid connectivity to Transit Gateway
cat << 'EOF' > add-hybrid-connectivity.sh
#!/bin/bash

source tgw-config.env

echo "=== Adding Hybrid Connectivity to Transit Gateway ==="
echo ""

echo "1. CREATING VPN CONNECTION TO TRANSIT GATEWAY"
echo ""

# Create Customer Gateway for VPN
CGW_NAME="HQ-Customer-Gateway"
CUSTOMER_IP="203.0.113.10"  # Replace with actual public IP
CUSTOMER_ASN="65001"

echo "Creating Customer Gateway..."
CGW_ID=$(aws ec2 create-customer-gateway \
    --type ipsec.1 \
    --public-ip $CUSTOMER_IP \
    --bgp-asn $CUSTOMER_ASN \
    --tag-specifications "ResourceType=customer-gateway,Tags=[{Key=Name,Value=$CGW_NAME}]" \
    --query 'CustomerGateway.CustomerGatewayId' \
    --output text)

echo "✅ Customer Gateway created: $CGW_ID"

# Create VPN connection to Transit Gateway
VPN_NAME="HQ-to-TGW-VPN"
echo "Creating VPN connection to Transit Gateway..."

VPN_ID=$(aws ec2 create-vpn-connection \
    --type ipsec.1 \
    --customer-gateway-id $CGW_ID \
    --transit-gateway-id $TGW_ID \
    --options TunnelInsideIpVersion=ipv4,TunnelOptions='[{TunnelInsideCidr=169.254.10.0/30},{TunnelInsideCidr=169.254.11.0/30}]' \
    --tag-specifications "ResourceType=vpn-connection,Tags=[{Key=Name,Value=$VPN_NAME}]" \
    --query 'VpnConnection.VpnConnectionId' \
    --output text)

echo "✅ VPN connection created: $VPN_ID"
echo ""

echo "2. SIMULATING DIRECT CONNECT GATEWAY ATTACHMENT"
echo ""

# In production, you would:
# 1. Create Direct Connect Gateway
# 2. Associate DX Gateway with Transit Gateway
# 3. Create Virtual Interface to DX Gateway

echo "Direct Connect Gateway setup (production steps):"
cat << 'DX_STEPS'
# Create Direct Connect Gateway
aws directconnect create-direct-connect-gateway \
    --name "Enterprise-DX-Gateway"

# Associate DX Gateway with Transit Gateway
aws ec2 associate-transit-gateway-direct-connect-gateway \
    --transit-gateway-id <tgw-id> \
    --direct-connect-gateway-id <dx-gateway-id> \
    --tags Key=Name,Value=TGW-DX-Association

# Create Transit Virtual Interface
aws directconnect create-transit-virtual-interface \
    --connection-id <dx-connection-id> \
    --new-transit-virtual-interface \
        virtualInterfaceName=Enterprise-Transit-VIF,\
        vlan=100,\
        asn=65001,\
        directConnectGatewayId=<dx-gateway-id>
DX_STEPS

echo ""
echo "3. CONFIGURING HYBRID ROUTING"
echo ""

# Get VPN attachment ID (wait for VPN to be available first)
echo "Waiting for VPN connection to be available..."
aws ec2 wait vpn-connection-available --vpn-connection-ids $VPN_ID

VPN_ATTACH_ID=$(aws ec2 describe-transit-gateway-attachments \
    --filters "Name=resource-id,Values=$VPN_ID" \
    --query 'TransitGatewayAttachments[0].TransitGatewayAttachmentId' \
    --output text)

echo "VPN attachment ID: $VPN_ATTACH_ID"

# Associate VPN with on-premises route table
echo "Associating VPN with on-premises route table..."
aws ec2 associate-transit-gateway-route-table \
    --transit-gateway-attachment-id $VPN_ATTACH_ID \
    --transit-gateway-route-table-id $ONPREM_TGW_RT

# Configure route propagation from on-premises to relevant environments
echo "Configuring route propagation from on-premises..."

# On-premises routes propagate to production (full access)
aws ec2 enable-transit-gateway-route-table-propagation \
    --transit-gateway-attachment-id $VPN_ATTACH_ID \
    --transit-gateway-route-table-id $PROD_TGW_RT

# On-premises routes propagate to shared services
aws ec2 enable-transit-gateway-route-table-propagation \
    --transit-gateway-attachment-id $VPN_ATTACH_ID \
    --transit-gateway-route-table-id $SHARED_TGW_RT

# Limited on-premises access to development (optional)
# aws ec2 enable-transit-gateway-route-table-propagation \
#     --transit-gateway-attachment-id $VPN_ATTACH_ID \
#     --transit-gateway-route-table-id $DEV_TGW_RT

echo "✅ Hybrid connectivity configured"
echo ""

echo "Hybrid Connectivity Summary:"
echo "============================"
echo "VPN Connection: $VPN_ID"
echo "Customer Gateway: $CGW_ID"
echo "VPN Attachment: $VPN_ATTACH_ID"
echo ""
echo "Connectivity Matrix:"
echo "  Production ←→ On-premises (Full access)"
echo "  Shared Services ←→ On-premises (Management access)"
echo "  Development ←→ On-premises (No access - secure)"

# Save hybrid configuration
cat << CONFIG > hybrid-config.env
export CGW_ID=$CGW_ID
export VPN_ID=$VPN_ID
export VPN_ATTACH_ID=$VPN_ATTACH_ID
CONFIG

echo ""
echo "Hybrid configuration saved to hybrid-config.env"

EOF

chmod +x add-hybrid-connectivity.sh
./add-hybrid-connectivity.sh
```

### Step 10: Monitor and Validate Transit Gateway
**Objective**: Set up monitoring and validate connectivity

```bash
# Monitor and validate Transit Gateway configuration
cat << 'EOF' > monitor-validate-tgw.sh
#!/bin/bash

source tgw-config.env
source attachment-config.env

echo "=== Monitoring and Validating Transit Gateway ==="
echo ""

echo "1. TRANSIT GATEWAY STATUS"
echo "Checking Transit Gateway health..."

aws ec2 describe-transit-gateways \
    --transit-gateway-ids $TGW_ID \
    --query 'TransitGateways[0].[TransitGatewayId,State,Description]' \
    --output table

echo ""
echo "2. ATTACHMENT STATUS"
echo "Checking all attachment states..."

aws ec2 describe-transit-gateway-attachments \
    --filters "Name=transit-gateway-id,Values=$TGW_ID" \
    --query 'TransitGatewayAttachments[*].[TransitGatewayAttachmentId,ResourceType,ResourceId,State]' \
    --output table

echo ""
echo "3. ROUTE TABLE ANALYSIS"
echo "Analyzing route table configurations..."

# Function to analyze route table
analyze_route_table() {
    local rt_id=$1
    local rt_name=$2
    
    echo "Route Table: $rt_name ($rt_id)"
    echo "Associations:"
    aws ec2 get-transit-gateway-route-table-associations \
        --transit-gateway-route-table-id $rt_id \
        --query 'Associations[*].[TransitGatewayAttachmentId,State]' \
        --output table
    
    echo "Propagations:"
    aws ec2 get-transit-gateway-route-table-propagations \
        --transit-gateway-route-table-id $rt_id \
        --query 'TransitGatewayRouteTablePropagations[*].[TransitGatewayAttachmentId,State]' \
        --output table
    
    echo "Routes:"
    aws ec2 search-transit-gateway-routes \
        --transit-gateway-route-table-id $rt_id \
        --filters "Name=state,Values=active" \
        --query 'Routes[*].[DestinationCidrBlock,TransitGatewayAttachments[0].TransitGatewayAttachmentId,State]' \
        --output table
    
    echo "---"
}

analyze_route_table $PROD_TGW_RT "Production"
analyze_route_table $DEV_TGW_RT "Development"
analyze_route_table $SHARED_TGW_RT "Shared Services"

echo ""
echo "4. CONNECTIVITY TESTING"
echo "Preparing connectivity test scenarios..."

cat << 'TEST_SCENARIOS'
Connectivity Test Matrix:
========================

TEST 1: Production to Production (Should work)
- From Prod Web VPC (10.10.0.0/16)
- To Prod App VPC (10.11.0.0/16)
- Expected: SUCCESS

TEST 2: Production to Development (Should fail)
- From Prod Web VPC (10.10.0.0/16)
- To Dev VPC (10.20.0.0/16)
- Expected: FAILURE (blocked by route tables)

TEST 3: Development to Shared Services (Should work)
- From Dev VPC (10.20.0.0/16)
- To Shared VPC (10.100.0.0/16)
- Expected: SUCCESS

TEST 4: Any VPC to On-premises (Depends on routing)
- From any VPC
- To on-premises network (192.168.0.0/16)
- Expected: Based on route table configuration

To test connectivity:
1. Launch EC2 instances in test VPCs
2. Use ping, telnet, or application-specific tests
3. Monitor VPC Flow Logs for traffic analysis
4. Check Transit Gateway route tables for path verification

TEST_SCENARIOS

echo ""
echo "5. CLOUDWATCH MONITORING SETUP"
echo "Setting up monitoring and alerting..."

# Create CloudWatch dashboard for Transit Gateway
cat << 'DASHBOARD' > tgw-dashboard.json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["AWS/TransitGateway", "BytesIn", "TransitGateway", "TGW_ID_PLACEHOLDER"],
          [".", "BytesOut", ".", "."],
          [".", "PacketDropCount", ".", "."]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "us-east-1",
        "title": "Transit Gateway Traffic"
      }
    }
  ]
}
DASHBOARD

# Replace placeholder with actual TGW ID
sed -i "s/TGW_ID_PLACEHOLDER/$TGW_ID/g" tgw-dashboard.json

# Create dashboard
aws cloudwatch put-dashboard \
    --dashboard-name "Transit-Gateway-Monitoring" \
    --dashboard-body file://tgw-dashboard.json

echo "✅ CloudWatch dashboard created: Transit-Gateway-Monitoring"

# Create alarm for high packet drops
aws cloudwatch put-metric-alarm \
    --alarm-name "TGW-High-Packet-Drops" \
    --alarm-description "Alert when Transit Gateway packet drops are high" \
    --metric-name PacketDropCount \
    --namespace AWS/TransitGateway \
    --statistic Sum \
    --period 300 \
    --threshold 100 \
    --comparison-operator GreaterThanThreshold \
    --dimensions Name=TransitGateway,Value=$TGW_ID \
    --evaluation-periods 2

echo "✅ CloudWatch alarm created for packet drop monitoring"

echo ""
echo "6. COST OPTIMIZATION ANALYSIS"
echo ""

# Calculate estimated costs
cat << 'COST_ANALYSIS'
Transit Gateway Cost Analysis:
=============================

Hourly Charges:
- Transit Gateway: $0.05/hour
- VPC Attachments: 6 × $0.05/hour = $0.30/hour
- VPN Attachments: 1 × $0.05/hour = $0.05/hour
Total Hourly: $0.40/hour

Monthly Estimate (730 hours):
- Base charges: $292/month
- Data processing: $0.02/GB processed
- Cross-AZ data transfer: Standard rates apply

Optimization Recommendations:
1. Monitor data transfer patterns
2. Optimize cross-AZ traffic
3. Use VPC endpoints for AWS services
4. Consider data compression where applicable
5. Regular review of attachment utilization

COST_ANALYSIS

echo ""
echo "✅ Transit Gateway monitoring and validation completed"
echo ""
echo "Summary of Created Resources:"
echo "============================"
echo "Transit Gateway: $TGW_ID"
echo "Route Tables: 4 custom route tables"
echo "VPC Attachments: 6 attachments"
echo "Hybrid Connectivity: VPN connection configured"
echo "Monitoring: CloudWatch dashboard and alarms"

EOF

chmod +x monitor-validate-tgw.sh
./monitor-validate-tgw.sh
```

## Advanced Transit Gateway Features

### Cross-Region Connectivity
```bash
# Set up cross-region Transit Gateway peering
cat << 'EOF' > setup-cross-region-tgw.sh
#!/bin/bash

echo "=== Cross-Region Transit Gateway Setup ==="
echo ""

# Create Transit Gateway in second region (us-west-2)
echo "Creating Transit Gateway in us-west-2..."
WEST_TGW_ID=$(aws ec2 create-transit-gateway \
    --description "West Coast Transit Gateway" \
    --region us-west-2 \
    --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=West-TGW}]' \
    --query 'TransitGateway.TransitGatewayId' \
    --output text)

echo "West TGW created: $WEST_TGW_ID"

# Create peering connection between regions
echo "Creating cross-region peering..."
PEER_ATTACH_ID=$(aws ec2 create-transit-gateway-peering-attachment \
    --transit-gateway-id $TGW_ID \
    --peer-transit-gateway-id $WEST_TGW_ID \
    --peer-region us-west-2 \
    --tag-specifications 'ResourceType=transit-gateway-peering-attachment,Tags=[{Key=Name,Value=East-West-Peering}]' \
    --query 'TransitGatewayPeeringAttachment.TransitGatewayPeeringAttachmentId' \
    --output text)

echo "Peering attachment created: $PEER_ATTACH_ID"

# Accept peering in west region
aws ec2 accept-transit-gateway-peering-attachment \
    --transit-gateway-peering-attachment-id $PEER_ATTACH_ID \
    --region us-west-2

echo "✅ Cross-region peering established"

# Configure selective routing between regions
echo "Configuring cross-region routing..."

# Add static routes for cross-region communication
aws ec2 create-route \
    --route-table-id $PROD_TGW_RT \
    --destination-cidr-block 10.200.0.0/16 \
    --transit-gateway-attachment-id $PEER_ATTACH_ID

echo "✅ Cross-region routing configured"

EOF

chmod +x setup-cross-region-tgw.sh
# Note: This script shows the concept - actual execution requires west region setup
```

## Best Practices Summary

### Design Best Practices
- **Segmentation**: Use custom route tables for network isolation
- **Scalability**: Plan for growth with proper CIDR allocation
- **High Availability**: Deploy across multiple AZs and regions
- **Security**: Implement defense in depth with route table controls

### Operational Best Practices
- **Monitoring**: Comprehensive CloudWatch monitoring and alerting
- **Documentation**: Maintain detailed network diagrams and procedures
- **Testing**: Regular connectivity and failover testing
- **Cost Management**: Monitor usage patterns and optimize accordingly

### Security Best Practices
- **Route Control**: Use route tables for micro-segmentation
- **Least Privilege**: Grant minimum required connectivity
- **Monitoring**: Track all traffic flows and route changes
- **Compliance**: Meet regulatory requirements with proper isolation

## Key Takeaways
- Transit Gateway provides centralized hub-and-spoke connectivity
- Custom route tables enable sophisticated network segmentation
- Route associations and propagations control traffic flow
- Hybrid connectivity integrates on-premises networks seamlessly
- Cross-region peering enables global network architectures
- Comprehensive monitoring ensures operational visibility
- Proper design balances connectivity, security, and cost

## Additional Resources
- [AWS Transit Gateway Documentation](https://docs.aws.amazon.com/vpc/latest/tgw/)
- [Transit Gateway Best Practices](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-best-design-practices.html)
- [Transit Gateway Monitoring](https://docs.aws.amazon.com/vpc/latest/tgw/transit-gateway-cloudwatch-metrics.html)

## Exam Tips
- Transit Gateway uses hub-and-spoke model, not full mesh like VPC peering
- Maximum 5,000 VPCs per Transit Gateway
- Route tables control connectivity - associations determine which table, propagations determine learned routes
- Cross-region peering enables global connectivity with bandwidth limits
- BGP is used for dynamic routing with on-premises connections
- Each attachment has bandwidth limits (50 Gbps VPC, varies for others)
- Route propagation is automatic for BGP-learned routes
- Static routes can be added to route tables for specific routing needs