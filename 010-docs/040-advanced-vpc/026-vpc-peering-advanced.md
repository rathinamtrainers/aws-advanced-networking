# Topic 26: Advanced VPC Peering and Multi-VPC Architectures

## Prerequisites
- Completed Hybrid Networking Module (Topics 11-25)
- Understanding of VPC fundamentals and routing concepts
- Knowledge of AWS networking services and inter-VPC connectivity
- Familiarity with DNS resolution and network troubleshooting

## Learning Objectives
By the end of this topic, you will be able to:
- Design complex multi-VPC architectures using VPC peering
- Configure transitive routing solutions and hub-and-spoke topologies
- Implement cross-account and cross-region VPC peering strategies
- Troubleshoot advanced VPC peering connectivity and DNS resolution issues

## Theory

### VPC Peering Advanced Concepts

#### Definition
VPC peering enables network connectivity between VPCs using private IP addresses, allowing resources in different VPCs to communicate as if they were within the same network.

#### Advanced Peering Characteristics
- **Non-transitive**: Peering connections don't support transitive routing
- **Cross-account**: Peering works across different AWS accounts
- **Cross-region**: Peering supports inter-region connectivity
- **CIDR Limitations**: Overlapping CIDR blocks are not supported
- **DNS Resolution**: Cross-VPC DNS resolution requires configuration

### Multi-VPC Architecture Patterns

#### Pattern 1: Hub-and-Spoke with Shared Services
```
        Shared Services VPC (Hub)
               (10.0.0.0/16)
                     |
        ┌────────────┼────────────┐
        |            |            |
   App VPC A    App VPC B    App VPC C
  (10.1.0.0/16) (10.2.0.0/16) (10.3.0.0/16)
```

#### Pattern 2: Mesh Topology for Full Connectivity
```
    VPC A ←→ VPC B
      ↕       ↕
    VPC D ←→ VPC C
```

#### Pattern 3: Segmented Environment Architecture
```
Production VPC ←→ Shared Services VPC ←→ Development VPC
     ↕                    ↕                    ↕
  (10.0.0.0/16)      (10.100.0.0/16)    (10.200.0.0/16)
```

### Cross-Account VPC Peering

#### Security Considerations
- **Least Privilege**: Limit cross-account access to required resources
- **Account Isolation**: Maintain security boundaries between accounts
- **Resource Sharing**: Controlled sharing of common services
- **Audit Trail**: Track cross-account network access

#### Trust Relationships
```
Account A (Requester) → Peering Request → Account B (Accepter)
Account B (Accepter) → Accept Request → Account A (Requester)
```

### DNS Resolution in VPC Peering

#### DNS Resolution Options
- **Requester DNS Resolution**: Enable DNS resolution for peering connection
- **Accepter DNS Resolution**: Allow DNS queries from peering connection
- **Private Hosted Zones**: Route 53 private zones for cross-VPC DNS
- **Custom DNS**: Third-party DNS solutions for complex scenarios

#### DNS Resolution Flow
```
VPC A Instance → DNS Query → VPC B Private IP → Response
                     ↓
             Route 53 Resolver
                     ↓
            Private Hosted Zone
```

## Lab Exercise: Advanced Multi-VPC Architecture Implementation

### Lab Duration: 360 minutes

### Step 1: Design Multi-VPC Architecture
**Objective**: Plan comprehensive multi-VPC connectivity strategy

**Architecture Planning**:
```bash
# Create advanced VPC peering planning assessment
cat << 'EOF' > advanced-vpc-peering-planning.sh
#!/bin/bash

echo "=== Advanced VPC Peering Architecture Planning ==="
echo ""

echo "1. VPC ARCHITECTURE REQUIREMENTS"
echo "   Number of VPCs: _______"
echo "   VPC purpose: Production/Development/Shared Services"
echo "   Cross-account requirements: Yes/No"
echo "   Cross-region requirements: Yes/No"
echo "   Connectivity pattern: Hub-spoke/Mesh/Segmented"
echo "   DNS resolution requirements: Yes/No"
echo ""

echo "2. NETWORK DESIGN CONSIDERATIONS"
echo "   CIDR block allocation strategy:"
echo "   - Production VPC: _______"
echo "   - Development VPC: _______"
echo "   - Shared Services VPC: _______"
echo "   - Management VPC: _______"
echo "   Total IP address space: _______"
echo "   Subnet segmentation: Public/Private/Database"
echo ""

echo "3. SECURITY REQUIREMENTS"
echo "   Network isolation levels: High/Medium/Low"
echo "   Cross-account access control: Yes/No"
echo "   Traffic inspection requirements: Yes/No"
echo "   Compliance requirements: SOC/PCI/HIPAA"
echo "   Audit logging: Yes/No"
echo ""

echo "4. PERFORMANCE REQUIREMENTS"
echo "   Expected traffic volumes: _______"
echo "   Latency requirements: _______"
echo "   Bandwidth requirements: _______"
echo "   High availability: Yes/No"
echo "   Disaster recovery: Yes/No"
echo ""

echo "5. DNS AND SERVICE DISCOVERY"
echo "   Cross-VPC DNS resolution: Yes/No"
echo "   Private hosted zones: Yes/No"
echo "   Service discovery requirements: Yes/No"
echo "   Custom DNS infrastructure: Yes/No"
echo "   External DNS integration: Yes/No"
echo ""

echo "6. COST OPTIMIZATION"
echo "   Data transfer optimization: Yes/No"
echo "   VPC peering vs Transit Gateway: _______"
echo "   Regional vs cross-region peering: _______"
echo "   Expected monthly data transfer: _______"
echo "   Cost monitoring requirements: Yes/No"

EOF

chmod +x advanced-vpc-peering-planning.sh
./advanced-vpc-peering-planning.sh
```

### Step 2: Create Multi-VPC Infrastructure
**Objective**: Deploy hub-and-spoke VPC architecture

**Multi-VPC Infrastructure Setup**:
```bash
# Create comprehensive multi-VPC infrastructure
cat << 'EOF' > create-multi-vpc-infrastructure.sh
#!/bin/bash

echo "=== Creating Multi-VPC Infrastructure ==="
echo ""

echo "1. CREATING SHARED SERVICES VPC (HUB)"

# Create shared services VPC
SHARED_VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.0.0.0/16 \
    --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=Shared-Services-VPC},{Key=Environment,Value=Shared}]" \
    --query 'Vpc.VpcId' \
    --output text)

echo "✅ Shared Services VPC created: $SHARED_VPC_ID"

# Create shared services subnets
SHARED_SUBNET_A_ID=$(aws ec2 create-subnet \
    --vpc-id $SHARED_VPC_ID \
    --cidr-block 10.0.1.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=Shared-Services-Subnet-A}]" \
    --query 'Subnet.SubnetId' \
    --output text)

SHARED_SUBNET_B_ID=$(aws ec2 create-subnet \
    --vpc-id $SHARED_VPC_ID \
    --cidr-block 10.0.2.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=Shared-Services-Subnet-B}]" \
    --query 'Subnet.SubnetId' \
    --output text)

echo "✅ Shared Services subnets created: $SHARED_SUBNET_A_ID, $SHARED_SUBNET_B_ID"

echo ""
echo "2. CREATING PRODUCTION VPC (SPOKE)"

# Create production VPC
PROD_VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.1.0.0/16 \
    --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=Production-VPC},{Key=Environment,Value=Production}]" \
    --query 'Vpc.VpcId' \
    --output text)

echo "✅ Production VPC created: $PROD_VPC_ID"

# Create production subnets
PROD_SUBNET_A_ID=$(aws ec2 create-subnet \
    --vpc-id $PROD_VPC_ID \
    --cidr-block 10.1.1.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=Production-Subnet-A}]" \
    --query 'Subnet.SubnetId' \
    --output text)

PROD_SUBNET_B_ID=$(aws ec2 create-subnet \
    --vpc-id $PROD_VPC_ID \
    --cidr-block 10.1.2.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=Production-Subnet-B}]" \
    --query 'Subnet.SubnetId' \
    --output text)

echo "✅ Production subnets created: $PROD_SUBNET_A_ID, $PROD_SUBNET_B_ID"

echo ""
echo "3. CREATING DEVELOPMENT VPC (SPOKE)"

# Create development VPC
DEV_VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.2.0.0/16 \
    --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=Development-VPC},{Key=Environment,Value=Development}]" \
    --query 'Vpc.VpcId' \
    --output text)

echo "✅ Development VPC created: $DEV_VPC_ID"

# Create development subnets
DEV_SUBNET_A_ID=$(aws ec2 create-subnet \
    --vpc-id $DEV_VPC_ID \
    --cidr-block 10.2.1.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=Development-Subnet-A}]" \
    --query 'Subnet.SubnetId' \
    --output text)

DEV_SUBNET_B_ID=$(aws ec2 create-subnet \
    --vpc-id $DEV_VPC_ID \
    --cidr-block 10.2.2.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=Development-Subnet-B}]" \
    --query 'Subnet.SubnetId' \
    --output text)

echo "✅ Development subnets created: $DEV_SUBNET_A_ID, $DEV_SUBNET_B_ID"

echo ""
echo "4. CREATING MANAGEMENT VPC (SPOKE)"

# Create management VPC
MGMT_VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.3.0.0/16 \
    --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=Management-VPC},{Key=Environment,Value=Management}]" \
    --query 'Vpc.VpcId' \
    --output text)

echo "✅ Management VPC created: $MGMT_VPC_ID"

# Create management subnets
MGMT_SUBNET_A_ID=$(aws ec2 create-subnet \
    --vpc-id $MGMT_VPC_ID \
    --cidr-block 10.3.1.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=Management-Subnet-A}]" \
    --query 'Subnet.SubnetId' \
    --output text)

MGMT_SUBNET_B_ID=$(aws ec2 create-subnet \
    --vpc-id $MGMT_VPC_ID \
    --cidr-block 10.3.2.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=Management-Subnet-B}]" \
    --query 'Subnet.SubnetId' \
    --output text)

echo "✅ Management subnets created: $MGMT_SUBNET_A_ID, $MGMT_SUBNET_B_ID"

echo ""
echo "5. CREATING SECURITY GROUPS"

# Create security groups for each VPC
SHARED_SG_ID=$(aws ec2 create-security-group \
    --group-name Shared-Services-SG \
    --description "Security group for shared services VPC" \
    --vpc-id $SHARED_VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Shared-Services-SG}]" \
    --query 'GroupId' \
    --output text)

PROD_SG_ID=$(aws ec2 create-security-group \
    --group-name Production-SG \
    --description "Security group for production VPC" \
    --vpc-id $PROD_VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Production-SG}]" \
    --query 'GroupId' \
    --output text)

DEV_SG_ID=$(aws ec2 create-security-group \
    --group-name Development-SG \
    --description "Security group for development VPC" \
    --vpc-id $DEV_VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Development-SG}]" \
    --query 'GroupId' \
    --output text)

MGMT_SG_ID=$(aws ec2 create-security-group \
    --group-name Management-SG \
    --description "Security group for management VPC" \
    --vpc-id $MGMT_VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Management-SG}]" \
    --query 'GroupId' \
    --output text)

echo "✅ Security groups created"

# Configure security group rules for cross-VPC communication
# Allow traffic from shared services to all spokes
aws ec2 authorize-security-group-ingress \
    --group-id $PROD_SG_ID \
    --protocol tcp \
    --port 22 \
    --cidr 10.0.0.0/16

aws ec2 authorize-security-group-ingress \
    --group-id $DEV_SG_ID \
    --protocol tcp \
    --port 22 \
    --cidr 10.0.0.0/16

aws ec2 authorize-security-group-ingress \
    --group-id $MGMT_SG_ID \
    --protocol tcp \
    --port 22 \
    --cidr 10.0.0.0/16

# Allow management access to all environments
aws ec2 authorize-security-group-ingress \
    --group-id $PROD_SG_ID \
    --protocol tcp \
    --port 22 \
    --cidr 10.3.0.0/16

aws ec2 authorize-security-group-ingress \
    --group-id $DEV_SG_ID \
    --protocol tcp \
    --port 22 \
    --cidr 10.3.0.0/16

aws ec2 authorize-security-group-ingress \
    --group-id $SHARED_SG_ID \
    --protocol tcp \
    --port 22 \
    --cidr 10.3.0.0/16

echo "✅ Security group rules configured"

# Save configuration
cat << CONFIG > multi-vpc-config.env
export SHARED_VPC_ID="$SHARED_VPC_ID"
export SHARED_SUBNET_A_ID="$SHARED_SUBNET_A_ID"
export SHARED_SUBNET_B_ID="$SHARED_SUBNET_B_ID"
export PROD_VPC_ID="$PROD_VPC_ID"
export PROD_SUBNET_A_ID="$PROD_SUBNET_A_ID"
export PROD_SUBNET_B_ID="$PROD_SUBNET_B_ID"
export DEV_VPC_ID="$DEV_VPC_ID"
export DEV_SUBNET_A_ID="$DEV_SUBNET_A_ID"
export DEV_SUBNET_B_ID="$DEV_SUBNET_B_ID"
export MGMT_VPC_ID="$MGMT_VPC_ID"
export MGMT_SUBNET_A_ID="$MGMT_SUBNET_A_ID"
export MGMT_SUBNET_B_ID="$MGMT_SUBNET_B_ID"
export SHARED_SG_ID="$SHARED_SG_ID"
export PROD_SG_ID="$PROD_SG_ID"
export DEV_SG_ID="$DEV_SG_ID"
export MGMT_SG_ID="$MGMT_SG_ID"
CONFIG

echo ""
echo "MULTI-VPC INFRASTRUCTURE SUMMARY:"
echo "Shared Services VPC: $SHARED_VPC_ID (10.0.0.0/16)"
echo "Production VPC: $PROD_VPC_ID (10.1.0.0/16)"
echo "Development VPC: $DEV_VPC_ID (10.2.0.0/16)"
echo "Management VPC: $MGMT_VPC_ID (10.3.0.0/16)"
echo ""
echo "Configuration saved to multi-vpc-config.env"

EOF

chmod +x create-multi-vpc-infrastructure.sh
./create-multi-vpc-infrastructure.sh
```

### Step 3: Configure VPC Peering Connections
**Objective**: Establish peering connections in hub-and-spoke topology

**VPC Peering Configuration**:
```bash
# Configure VPC peering connections
cat << 'EOF' > configure-vpc-peering.sh
#!/bin/bash

source multi-vpc-config.env

echo "=== Configuring VPC Peering Connections ==="
echo ""

echo "1. CREATING PEERING CONNECTIONS (HUB-AND-SPOKE)"

# Create peering connection: Shared Services ↔ Production
SHARED_PROD_PEER_ID=$(aws ec2 create-vpc-peering-connection \
    --vpc-id $SHARED_VPC_ID \
    --peer-vpc-id $PROD_VPC_ID \
    --tag-specifications "ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=Shared-to-Production-Peering}]" \
    --query 'VpcPeeringConnection.VpcPeeringConnectionId' \
    --output text)

echo "✅ Shared-Production peering created: $SHARED_PROD_PEER_ID"

# Create peering connection: Shared Services ↔ Development
SHARED_DEV_PEER_ID=$(aws ec2 create-vpc-peering-connection \
    --vpc-id $SHARED_VPC_ID \
    --peer-vpc-id $DEV_VPC_ID \
    --tag-specifications "ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=Shared-to-Development-Peering}]" \
    --query 'VpcPeeringConnection.VpcPeeringConnectionId' \
    --output text)

echo "✅ Shared-Development peering created: $SHARED_DEV_PEER_ID"

# Create peering connection: Shared Services ↔ Management
SHARED_MGMT_PEER_ID=$(aws ec2 create-vpc-peering-connection \
    --vpc-id $SHARED_VPC_ID \
    --peer-vpc-id $MGMT_VPC_ID \
    --tag-specifications "ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=Shared-to-Management-Peering}]" \
    --query 'VpcPeeringConnection.VpcPeeringConnectionId' \
    --output text)

echo "✅ Shared-Management peering created: $SHARED_MGMT_PEER_ID"

# Create peering connection: Management ↔ Production (for management access)
MGMT_PROD_PEER_ID=$(aws ec2 create-vpc-peering-connection \
    --vpc-id $MGMT_VPC_ID \
    --peer-vpc-id $PROD_VPC_ID \
    --tag-specifications "ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=Management-to-Production-Peering}]" \
    --query 'VpcPeeringConnection.VpcPeeringConnectionId' \
    --output text)

echo "✅ Management-Production peering created: $MGMT_PROD_PEER_ID"

# Create peering connection: Management ↔ Development (for management access)
MGMT_DEV_PEER_ID=$(aws ec2 create-vpc-peering-connection \
    --vpc-id $MGMT_VPC_ID \
    --peer-vpc-id $DEV_VPC_ID \
    --tag-specifications "ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=Management-to-Development-Peering}]" \
    --query 'VpcPeeringConnection.VpcPeeringConnectionId' \
    --output text)

echo "✅ Management-Development peering created: $MGMT_DEV_PEER_ID"

echo ""
echo "2. ACCEPTING PEERING CONNECTIONS"

# Accept all peering connections
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $SHARED_PROD_PEER_ID
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $SHARED_DEV_PEER_ID
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $SHARED_MGMT_PEER_ID
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $MGMT_PROD_PEER_ID
aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $MGMT_DEV_PEER_ID

echo "✅ All peering connections accepted"

echo ""
echo "3. CONFIGURING DNS RESOLUTION"

# Enable DNS resolution for all peering connections
aws ec2 modify-vpc-peering-connection-options \
    --vpc-peering-connection-id $SHARED_PROD_PEER_ID \
    --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
    --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true

aws ec2 modify-vpc-peering-connection-options \
    --vpc-peering-connection-id $SHARED_DEV_PEER_ID \
    --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
    --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true

aws ec2 modify-vpc-peering-connection-options \
    --vpc-peering-connection-id $SHARED_MGMT_PEER_ID \
    --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
    --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true

aws ec2 modify-vpc-peering-connection-options \
    --vpc-peering-connection-id $MGMT_PROD_PEER_ID \
    --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
    --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true

aws ec2 modify-vpc-peering-connection-options \
    --vpc-peering-connection-id $MGMT_DEV_PEER_ID \
    --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
    --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true

echo "✅ DNS resolution enabled for all peering connections"

# Save peering configuration
cat << CONFIG > vpc-peering-config.env
export SHARED_PROD_PEER_ID="$SHARED_PROD_PEER_ID"
export SHARED_DEV_PEER_ID="$SHARED_DEV_PEER_ID"
export SHARED_MGMT_PEER_ID="$SHARED_MGMT_PEER_ID"
export MGMT_PROD_PEER_ID="$MGMT_PROD_PEER_ID"
export MGMT_DEV_PEER_ID="$MGMT_DEV_PEER_ID"
CONFIG

echo ""
echo "VPC PEERING CONFIGURATION SUMMARY:"
echo "Shared-Production: $SHARED_PROD_PEER_ID"
echo "Shared-Development: $SHARED_DEV_PEER_ID"
echo "Shared-Management: $SHARED_MGMT_PEER_ID"
echo "Management-Production: $MGMT_PROD_PEER_ID"
echo "Management-Development: $MGMT_DEV_PEER_ID"
echo ""
echo "Configuration saved to vpc-peering-config.env"

EOF

chmod +x configure-vpc-peering.sh
./configure-vpc-peering.sh
```

### Step 4: Configure Routing Tables
**Objective**: Set up routing for multi-VPC communication

**Routing Configuration**:
```bash
# Configure routing tables for VPC peering
cat << 'EOF' > configure-peering-routes.sh
#!/bin/bash

source multi-vpc-config.env
source vpc-peering-config.env

echo "=== Configuring Routing Tables for VPC Peering ==="
echo ""

echo "1. GETTING ROUTE TABLE IDs"

# Get main route table for each VPC
SHARED_RT_ID=$(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$SHARED_VPC_ID" "Name=association.main,Values=true" \
    --query 'RouteTables[0].RouteTableId' \
    --output text)

PROD_RT_ID=$(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$PROD_VPC_ID" "Name=association.main,Values=true" \
    --query 'RouteTables[0].RouteTableId' \
    --output text)

DEV_RT_ID=$(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$DEV_VPC_ID" "Name=association.main,Values=true" \
    --query 'RouteTables[0].RouteTableId' \
    --output text)

MGMT_RT_ID=$(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$MGMT_VPC_ID" "Name=association.main,Values=true" \
    --query 'RouteTables[0].RouteTableId' \
    --output text)

echo "✅ Route table IDs retrieved"

echo ""
echo "2. CONFIGURING SHARED SERVICES VPC ROUTES"

# Shared Services VPC routes to all spokes
aws ec2 create-route \
    --route-table-id $SHARED_RT_ID \
    --destination-cidr-block 10.1.0.0/16 \
    --vpc-peering-connection-id $SHARED_PROD_PEER_ID

aws ec2 create-route \
    --route-table-id $SHARED_RT_ID \
    --destination-cidr-block 10.2.0.0/16 \
    --vpc-peering-connection-id $SHARED_DEV_PEER_ID

aws ec2 create-route \
    --route-table-id $SHARED_RT_ID \
    --destination-cidr-block 10.3.0.0/16 \
    --vpc-peering-connection-id $SHARED_MGMT_PEER_ID

echo "✅ Shared Services VPC routes configured"

echo ""
echo "3. CONFIGURING PRODUCTION VPC ROUTES"

# Production VPC routes
aws ec2 create-route \
    --route-table-id $PROD_RT_ID \
    --destination-cidr-block 10.0.0.0/16 \
    --vpc-peering-connection-id $SHARED_PROD_PEER_ID

aws ec2 create-route \
    --route-table-id $PROD_RT_ID \
    --destination-cidr-block 10.3.0.0/16 \
    --vpc-peering-connection-id $MGMT_PROD_PEER_ID

echo "✅ Production VPC routes configured"

echo ""
echo "4. CONFIGURING DEVELOPMENT VPC ROUTES"

# Development VPC routes
aws ec2 create-route \
    --route-table-id $DEV_RT_ID \
    --destination-cidr-block 10.0.0.0/16 \
    --vpc-peering-connection-id $SHARED_DEV_PEER_ID

aws ec2 create-route \
    --route-table-id $DEV_RT_ID \
    --destination-cidr-block 10.3.0.0/16 \
    --vpc-peering-connection-id $MGMT_DEV_PEER_ID

echo "✅ Development VPC routes configured"

echo ""
echo "5. CONFIGURING MANAGEMENT VPC ROUTES"

# Management VPC routes to all other VPCs
aws ec2 create-route \
    --route-table-id $MGMT_RT_ID \
    --destination-cidr-block 10.0.0.0/16 \
    --vpc-peering-connection-id $SHARED_MGMT_PEER_ID

aws ec2 create-route \
    --route-table-id $MGMT_RT_ID \
    --destination-cidr-block 10.1.0.0/16 \
    --vpc-peering-connection-id $MGMT_PROD_PEER_ID

aws ec2 create-route \
    --route-table-id $MGMT_RT_ID \
    --destination-cidr-block 10.2.0.0/16 \
    --vpc-peering-connection-id $MGMT_DEV_PEER_ID

echo "✅ Management VPC routes configured"

echo ""
echo "6. CREATING ROUTE VALIDATION SCRIPT"

cat << 'ROUTE_VALIDATION' > validate-routes.sh
#!/bin/bash

echo "=== Validating VPC Peering Routes ==="
echo ""

echo "1. SHARED SERVICES VPC ROUTES:"
aws ec2 describe-route-tables \
    --route-table-ids $SHARED_RT_ID \
    --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,VpcPeeringConnectionId,State]' \
    --output table

echo ""
echo "2. PRODUCTION VPC ROUTES:"
aws ec2 describe-route-tables \
    --route-table-ids $PROD_RT_ID \
    --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,VpcPeeringConnectionId,State]' \
    --output table

echo ""
echo "3. DEVELOPMENT VPC ROUTES:"
aws ec2 describe-route-tables \
    --route-table-ids $DEV_RT_ID \
    --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,VpcPeeringConnectionId,State]' \
    --output table

echo ""
echo "4. MANAGEMENT VPC ROUTES:"
aws ec2 describe-route-tables \
    --route-table-ids $MGMT_RT_ID \
    --query 'RouteTables[0].Routes[*].[DestinationCidrBlock,VpcPeeringConnectionId,State]' \
    --output table

echo ""
echo "5. PEERING CONNECTION STATUS:"
aws ec2 describe-vpc-peering-connections \
    --vpc-peering-connection-ids $SHARED_PROD_PEER_ID $SHARED_DEV_PEER_ID $SHARED_MGMT_PEER_ID $MGMT_PROD_PEER_ID $MGMT_DEV_PEER_ID \
    --query 'VpcPeeringConnections[*].[VpcPeeringConnectionId,Status.Code,RequesterVpcInfo.VpcId,AccepterVpcInfo.VpcId]' \
    --output table

ROUTE_VALIDATION

chmod +x validate-routes.sh

# Save route table configuration
cat << CONFIG > route-tables-config.env
export SHARED_RT_ID="$SHARED_RT_ID"
export PROD_RT_ID="$PROD_RT_ID"
export DEV_RT_ID="$DEV_RT_ID"
export MGMT_RT_ID="$MGMT_RT_ID"
CONFIG

echo ""
echo "ROUTING CONFIGURATION COMPLETED"
echo ""
echo "Route table IDs:"
echo "Shared Services: $SHARED_RT_ID"
echo "Production: $PROD_RT_ID"
echo "Development: $DEV_RT_ID"
echo "Management: $MGMT_RT_ID"
echo ""
echo "Run './validate-routes.sh' to verify routing configuration"

EOF

chmod +x configure-peering-routes.sh
./configure-peering-routes.sh
```

### Step 5: Test Multi-VPC Connectivity
**Objective**: Verify connectivity and troubleshoot issues

**Connectivity Testing**:
```bash
# Test multi-VPC connectivity
cat << 'EOF' > test-multi-vpc-connectivity.sh
#!/bin/bash

source multi-vpc-config.env

echo "=== Testing Multi-VPC Connectivity ==="
echo ""

echo "1. LAUNCHING TEST INSTANCES"

# Get latest Amazon Linux 2 AMI
AMI_ID=$(aws ec2 describe-images \
    --owners amazon \
    --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
              "Name=state,Values=available" \
    --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
    --output text)

echo "Using AMI: $AMI_ID"

# Launch test instances in each VPC
SHARED_INSTANCE=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.micro \
    --subnet-id $SHARED_SUBNET_A_ID \
    --security-group-ids $SHARED_SG_ID \
    --associate-public-ip-address \
    --user-data '#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Shared Services VPC</h1>" > /var/www/html/index.html
echo "<p>Instance: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>" >> /var/www/html/index.html
echo "<p>Private IP: $(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)</p>" >> /var/www/html/index.html' \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=Shared-Services-Test}]" \
    --query 'Instances[0].InstanceId' \
    --output text)

PROD_INSTANCE=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.micro \
    --subnet-id $PROD_SUBNET_A_ID \
    --security-group-ids $PROD_SG_ID \
    --associate-public-ip-address \
    --user-data '#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Production VPC</h1>" > /var/www/html/index.html
echo "<p>Instance: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>" >> /var/www/html/index.html
echo "<p>Private IP: $(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)</p>" >> /var/www/html/index.html' \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=Production-Test}]" \
    --query 'Instances[0].InstanceId' \
    --output text)

DEV_INSTANCE=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.micro \
    --subnet-id $DEV_SUBNET_A_ID \
    --security-group-ids $DEV_SG_ID \
    --associate-public-ip-address \
    --user-data '#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Development VPC</h1>" > /var/www/html/index.html
echo "<p>Instance: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>" >> /var/www/html/index.html
echo "<p>Private IP: $(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)</p>" >> /var/www/html/index.html' \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=Development-Test}]" \
    --query 'Instances[0].InstanceId' \
    --output text)

MGMT_INSTANCE=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.micro \
    --subnet-id $MGMT_SUBNET_A_ID \
    --security-group-ids $MGMT_SG_ID \
    --associate-public-ip-address \
    --user-data '#!/bin/bash
yum update -y
yum install -y httpd tcpdump nmap
systemctl start httpd
systemctl enable httpd
echo "<h1>Management VPC</h1>" > /var/www/html/index.html
echo "<p>Instance: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>" >> /var/www/html/index.html
echo "<p>Private IP: $(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)</p>" >> /var/www/html/index.html' \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=Management-Test}]" \
    --query 'Instances[0].InstanceId' \
    --output text)

echo "✅ Test instances launched:"
echo "  Shared Services: $SHARED_INSTANCE"
echo "  Production: $PROD_INSTANCE"
echo "  Development: $DEV_INSTANCE"
echo "  Management: $MGMT_INSTANCE"

# Wait for instances to be running
echo "Waiting for instances to be running..."
aws ec2 wait instance-running --instance-ids $SHARED_INSTANCE $PROD_INSTANCE $DEV_INSTANCE $MGMT_INSTANCE
echo "✅ All instances are running"

echo ""
echo "2. GETTING INSTANCE PRIVATE IP ADDRESSES"

SHARED_IP=$(aws ec2 describe-instances \
    --instance-ids $SHARED_INSTANCE \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' \
    --output text)

PROD_IP=$(aws ec2 describe-instances \
    --instance-ids $PROD_INSTANCE \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' \
    --output text)

DEV_IP=$(aws ec2 describe-instances \
    --instance-ids $DEV_INSTANCE \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' \
    --output text)

MGMT_IP=$(aws ec2 describe-instances \
    --instance-ids $MGMT_INSTANCE \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' \
    --output text)

echo "✅ Private IP addresses:"
echo "  Shared Services: $SHARED_IP"
echo "  Production: $PROD_IP"
echo "  Development: $DEV_IP"
echo "  Management: $MGMT_IP"

echo ""
echo "3. CREATING CONNECTIVITY TEST SCRIPT"

cat << 'CONNECTIVITY_TEST' > test-connectivity.sh
#!/bin/bash

echo "=== Multi-VPC Connectivity Test ==="
echo ""

SHARED_IP="$SHARED_IP"
PROD_IP="$PROD_IP"
DEV_IP="$DEV_IP"
MGMT_IP="$MGMT_IP"

echo "Testing connectivity between all VPCs..."
echo ""

# Test from Management VPC to all other VPCs
echo "1. TESTING FROM MANAGEMENT VPC:"
echo "   To Shared Services ($SHARED_IP):"
curl -s --connect-timeout 5 http://$SHARED_IP/ > /dev/null && echo "   ✅ SUCCESS" || echo "   ❌ FAILED"

echo "   To Production ($PROD_IP):"
curl -s --connect-timeout 5 http://$PROD_IP/ > /dev/null && echo "   ✅ SUCCESS" || echo "   ❌ FAILED"

echo "   To Development ($DEV_IP):"
curl -s --connect-timeout 5 http://$DEV_IP/ > /dev/null && echo "   ✅ SUCCESS" || echo "   ❌ FAILED"

echo ""
echo "2. TESTING DNS RESOLUTION:"
echo "   Shared Services DNS:"
nslookup $SHARED_IP 2>/dev/null | grep -q "name =" && echo "   ✅ DNS RESOLUTION WORKS" || echo "   ❌ DNS RESOLUTION FAILED"

echo ""
echo "3. TESTING PING CONNECTIVITY:"
echo "   Ping to Shared Services:"
ping -c 3 $SHARED_IP > /dev/null 2>&1 && echo "   ✅ PING SUCCESS" || echo "   ❌ PING FAILED"

echo "   Ping to Production:"
ping -c 3 $PROD_IP > /dev/null 2>&1 && echo "   ✅ PING SUCCESS" || echo "   ❌ PING FAILED"

echo "   Ping to Development:"
ping -c 3 $DEV_IP > /dev/null 2>&1 && echo "   ✅ PING SUCCESS" || echo "   ❌ PING FAILED"

CONNECTIVITY_TEST

chmod +x test-connectivity.sh

# Save test configuration
cat << CONFIG > test-instances-config.env
export SHARED_INSTANCE="$SHARED_INSTANCE"
export PROD_INSTANCE="$PROD_INSTANCE"
export DEV_INSTANCE="$DEV_INSTANCE"
export MGMT_INSTANCE="$MGMT_INSTANCE"
export SHARED_IP="$SHARED_IP"
export PROD_IP="$PROD_IP"
export DEV_IP="$DEV_IP"
export MGMT_IP="$MGMT_IP"
CONFIG

echo ""
echo "CONNECTIVITY TESTING SETUP COMPLETED"
echo ""
echo "Test instances and IPs:"
echo "Shared Services: $SHARED_INSTANCE ($SHARED_IP)"
echo "Production: $PROD_INSTANCE ($PROD_IP)"
echo "Development: $DEV_INSTANCE ($DEV_IP)"
echo "Management: $MGMT_INSTANCE ($MGMT_IP)"
echo ""
echo "Run './test-connectivity.sh' to test connectivity"

EOF

chmod +x test-multi-vpc-connectivity.sh
./test-multi-vpc-connectivity.sh
```

### Step 6: Implement Cross-Account VPC Peering
**Objective**: Configure peering across AWS accounts

**Cross-Account Peering Setup**:
```bash
# Configure cross-account VPC peering
cat << 'EOF' > configure-cross-account-peering.sh
#!/bin/bash

echo "=== Configuring Cross-Account VPC Peering ==="
echo ""

echo "1. CROSS-ACCOUNT PEERING OVERVIEW"
echo "This script demonstrates cross-account VPC peering configuration"
echo "Prerequisites:"
echo "- Access to both AWS accounts"
echo "- VPC in each account with non-overlapping CIDR blocks"
echo "- Appropriate IAM permissions in both accounts"
echo ""

echo "2. CREATING CROSS-ACCOUNT PEERING CONNECTION"

# Note: This requires coordination between two AWS accounts
cat << 'CROSS_ACCOUNT_SCRIPT' > cross-account-peering-template.sh
#!/bin/bash

# Variables to set
ACCOUNT_A_VPC_ID="vpc-xxxxxxxxx"     # VPC ID in Account A
ACCOUNT_B_VPC_ID="vpc-yyyyyyyyy"     # VPC ID in Account B
ACCOUNT_B_ID="123456789012"          # Account B ID
ACCOUNT_B_REGION="us-east-1"         # Account B Region

echo "STEP 1: Create peering connection from Account A"
echo "Run this command in Account A:"
echo ""
echo "aws ec2 create-vpc-peering-connection \\"
echo "    --vpc-id $ACCOUNT_A_VPC_ID \\"
echo "    --peer-vpc-id $ACCOUNT_B_VPC_ID \\"
echo "    --peer-region $ACCOUNT_B_REGION \\"
echo "    --peer-owner-id $ACCOUNT_B_ID \\"
echo "    --tag-specifications \"ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=Cross-Account-Peering}]\""
echo ""

echo "STEP 2: Note the peering connection ID from the output"
echo "PEERING_CONNECTION_ID=\"pcx-xxxxxxxxx\""
echo ""

echo "STEP 3: Accept peering connection in Account B"
echo "Run this command in Account B:"
echo ""
echo "aws ec2 accept-vpc-peering-connection \\"
echo "    --vpc-peering-connection-id \$PEERING_CONNECTION_ID"
echo ""

echo "STEP 4: Enable DNS resolution (optional)"
echo "Run in Account A:"
echo "aws ec2 modify-vpc-peering-connection-options \\"
echo "    --vpc-peering-connection-id \$PEERING_CONNECTION_ID \\"
echo "    --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true"
echo ""
echo "Run in Account B:"
echo "aws ec2 modify-vpc-peering-connection-options \\"
echo "    --vpc-peering-connection-id \$PEERING_CONNECTION_ID \\"
echo "    --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true"
echo ""

echo "STEP 5: Configure routing tables"
echo "In Account A - add route to Account B VPC:"
echo "aws ec2 create-route \\"
echo "    --route-table-id rtb-xxxxxxxxx \\"
echo "    --destination-cidr-block 10.1.0.0/16 \\"
echo "    --vpc-peering-connection-id \$PEERING_CONNECTION_ID"
echo ""
echo "In Account B - add route to Account A VPC:"
echo "aws ec2 create-route \\"
echo "    --route-table-id rtb-yyyyyyyyy \\"
echo "    --destination-cidr-block 10.0.0.0/16 \\"
echo "    --vpc-peering-connection-id \$PEERING_CONNECTION_ID"
echo ""

echo "STEP 6: Update security groups"
echo "In both accounts, update security groups to allow traffic:"
echo "aws ec2 authorize-security-group-ingress \\"
echo "    --group-id sg-xxxxxxxxx \\"
echo "    --protocol tcp \\"
echo "    --port 22 \\"
echo "    --cidr 10.1.0.0/16"

CROSS_ACCOUNT_SCRIPT

chmod +x cross-account-peering-template.sh

echo "✅ Cross-account peering template created"

echo ""
echo "3. CROSS-ACCOUNT SECURITY CONSIDERATIONS"

cat << 'SECURITY_GUIDE' > cross-account-security-guide.md
# Cross-Account VPC Peering Security Guide

## IAM Permissions Required

### Account A (Requester)
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:CreateVpcPeeringConnection",
                "ec2:DescribeVpcPeeringConnections",
                "ec2:ModifyVpcPeeringConnectionOptions",
                "ec2:CreateRoute",
                "ec2:DescribeRouteTables"
            ],
            "Resource": "*"
        }
    ]
}
```

### Account B (Accepter)
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:AcceptVpcPeeringConnection",
                "ec2:DescribeVpcPeeringConnections",
                "ec2:ModifyVpcPeeringConnectionOptions",
                "ec2:CreateRoute",
                "ec2:DescribeRouteTables"
            ],
            "Resource": "*"
        }
    ]
}
```

## Security Best Practices

### 1. Network Segmentation
- Use specific CIDR blocks for cross-account communication
- Implement security groups with least privilege principles
- Consider using NACLs for additional network-level security

### 2. Access Control
- Limit cross-account access to specific resources
- Use IAM roles for cross-account access
- Implement resource-based policies where applicable

### 3. Monitoring and Auditing
- Enable VPC Flow Logs for both VPCs
- Monitor CloudTrail for peering-related API calls
- Set up CloudWatch alarms for unusual traffic patterns

### 4. Resource Tagging
- Use consistent tagging across accounts
- Tag peering connections for cost allocation
- Implement tag-based access control

## Troubleshooting Common Issues

### 1. Peering Connection Stuck in Pending
- Check account ID and region are correct
- Verify IAM permissions in accepter account
- Ensure VPC CIDR blocks don't overlap

### 2. DNS Resolution Not Working
- Verify DNS resolution options are enabled
- Check VPC DNS settings (enableDnsHostnames, enableDnsSupport)
- Validate Route 53 private hosted zone configuration

### 3. Connectivity Issues
- Verify routing tables have correct routes
- Check security group rules allow traffic
- Ensure NACLs don't block traffic
- Validate that instances are in correct subnets

SECURITY_GUIDE

echo "✅ Security guide created: cross-account-security-guide.md"

echo ""
echo "CROSS-ACCOUNT VPC PEERING SETUP COMPLETED"
echo ""
echo "Files created:"
echo "1. cross-account-peering-template.sh - Step-by-step peering setup"
echo "2. cross-account-security-guide.md - Security best practices"
echo ""
echo "Next steps:"
echo "1. Review the security guide"
echo "2. Execute the template script with proper account details"
echo "3. Test connectivity between cross-account VPCs"

EOF

chmod +x configure-cross-account-peering.sh
./configure-cross-account-peering.sh
```

## Advanced VPC Peering Patterns Summary

### Hub-and-Spoke Architecture
- **Central Hub**: Shared services VPC provides common resources
- **Spokes**: Individual application or environment VPCs
- **Benefits**: Centralized management, reduced complexity, cost efficiency
- **Limitations**: No spoke-to-spoke communication without hub

### Mesh Architecture
- **Full Connectivity**: Every VPC peers with every other VPC
- **Benefits**: Direct communication between all VPCs
- **Limitations**: Complex management, scales as n(n-1)/2 connections

### Segmented Architecture
- **Environment Separation**: Production, development, staging isolation
- **Selective Connectivity**: Management access to all environments
- **Benefits**: Security through isolation, controlled access

## DNS Resolution in VPC Peering

### DNS Configuration Options
- **Requester DNS Resolution**: Allow requests from peering connection
- **Accepter DNS Resolution**: Allow requests to peering connection
- **Private Hosted Zones**: Route 53 zones for cross-VPC resolution
- **Custom DNS**: Third-party DNS for complex requirements

### DNS Resolution Flow
1. Instance makes DNS query
2. VPC DNS resolver processes query
3. If cross-VPC, forwards to peered VPC
4. Returns private IP if resolution enabled

## Cross-Account Peering Best Practices

### Security Considerations
- **IAM Permissions**: Minimize required permissions
- **Resource Policies**: Use resource-based access control
- **Network Segmentation**: Implement security groups and NACLs
- **Monitoring**: Enable logging and alerting

### Operational Considerations
- **Change Management**: Coordinate changes across accounts
- **Cost Allocation**: Use tagging for cost tracking
- **Troubleshooting**: Maintain access for support teams

## Key Takeaways
- VPC peering enables private connectivity between VPCs
- Non-transitive routing requires direct peering for multi-VPC communication
- DNS resolution must be explicitly enabled for cross-VPC name resolution
- Cross-account peering requires coordination and proper IAM permissions
- Security groups and routing tables must be configured for connectivity
- Monitoring and troubleshooting are essential for production deployments

## Additional Resources
- [AWS VPC Peering User Guide](https://docs.aws.amazon.com/vpc/latest/peering/)
- [VPC Peering Scenarios](https://docs.aws.amazon.com/vpc/latest/peering/peering-scenarios.html)
- [Cross-Account VPC Peering](https://docs.aws.amazon.com/vpc/latest/peering/cross-account-peering-overview.html)

## Exam Tips
- VPC peering is not transitive - requires direct connections
- CIDR blocks cannot overlap between peered VPCs
- DNS resolution must be enabled for cross-VPC name resolution
- Security groups can reference security groups in peered VPCs
- Route tables must contain routes for peered VPC CIDR blocks
- Cross-account peering requires acceptance from both accounts
- VPC peering works across regions with additional data transfer costs
- Maximum of 50 VPC peering connections per VPC (soft limit)