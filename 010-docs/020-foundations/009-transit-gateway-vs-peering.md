# Topic 9: Transit Gateway vs VPC Peering - Which to Choose?

## Prerequisites
- Completed Topics 1-8 (AWS Networking Foundations)
- Understanding of VPC peering concepts
- Knowledge of network topology design principles

## Learning Objectives
By the end of this topic, you will be able to:
- Compare Transit Gateway and VPC Peering architectures
- Make informed decisions based on network requirements
- Implement migration strategies between peering and Transit Gateway
- Calculate cost implications for different network topologies

## Theory

### Architecture Comparison

#### VPC Peering Architecture
```
VPC-A ←→ VPC-B
  ↓       ↓
VPC-C ←→ VPC-D
  ↓       ↓
VPC-E ←→ VPC-F

Full Mesh: 15 connections for 6 VPCs
Formula: N*(N-1)/2 connections
```

#### Transit Gateway Architecture
```
        [Transit Gateway]
         /    |    \
    VPC-A   VPC-B   VPC-C
       |      |      |
    VPC-D   VPC-E   VPC-F

Hub-and-Spoke: 6 connections for 6 VPCs
Formula: N connections
```

### Feature Comparison Matrix

| Feature | VPC Peering | Transit Gateway |
|---------|-------------|-----------------|
| **Connectivity Model** | Point-to-point | Hub-and-spoke |
| **Transitive Routing** | No | Yes |
| **Maximum VPCs** | 125 per VPC | 5,000 per TGW |
| **Bandwidth Limits** | No limits | 50 Gbps per attachment |
| **Route Propagation** | Manual | Automatic (BGP) |
| **Network Segmentation** | Security groups/NACLs | Route tables + associations |
| **Cross-Region** | Yes | Yes (peering required) |
| **Cross-Account** | Yes | Yes (sharing required) |
| **Pricing Model** | Data transfer only | Hourly + data processing |
| **Management Complexity** | High (mesh) | Low (centralized) |
| **Latency** | Minimal | Slight overhead |
| **Failure Domain** | Per connection | Regional service |

### Detailed Technical Comparison

#### Connectivity and Routing

**VPC Peering**:
- Direct Layer 3 connectivity between two VPCs
- No transitive routing (A→B→C not possible)
- Longest prefix matching for overlapping routes
- Manual route table updates required

**Transit Gateway**:
- Hub-and-spoke connectivity model
- Transitive routing enabled by default
- Dynamic route propagation with BGP
- Centralized route management

#### Scalability Analysis

**VPC Peering Scalability**:
```
VPCs    Connections    Management Overhead
2       1              Low
5       10             Medium
10      45             High
20      190            Very High
50      1,225          Unmanageable
```

**Transit Gateway Scalability**:
```
VPCs    Connections    Management Overhead
2       2              Low
5       5              Low
10      10             Low
20      20             Low
50      50             Medium
5,000   5,000          High (but manageable)
```

#### Cost Analysis Framework

**VPC Peering Costs**:
- **Intra-region**: Free data transfer
- **Inter-region**: Standard data transfer rates
- **Management**: Operational overhead for complex topologies

**Transit Gateway Costs**:
- **Hourly charge**: $0.05 per attachment per hour
- **Data processing**: $0.02 per GB processed
- **Cross-AZ**: Standard data transfer charges
- **Management**: Reduced operational overhead

## Lab Exercise: Migration from VPC Peering to Transit Gateway

### Lab Duration: 200 minutes

### Step 1: Assess Current Peering Architecture
**Objective**: Analyze existing VPC peering setup

**Current State Analysis**:
```bash
# List all VPC peering connections
aws ec2 describe-vpc-peering-connections \
    --query 'VpcPeeringConnections[*].[VpcPeeringConnectionId,RequesterVpcInfo.VpcId,AccepterVpcInfo.VpcId,Status.Code]' \
    --output table

# Analyze route table complexity
aws ec2 describe-route-tables \
    --query 'RouteTables[?Routes[?VpcPeeringConnectionId]].[RouteTableId,VpcId,length(Routes[?VpcPeeringConnectionId])]' \
    --output table

# Calculate current peering connections
VPC_COUNT=$(aws ec2 describe-vpcs --query 'length(Vpcs[?State==`available`])' --output text)
PEERING_COUNT=$(aws ec2 describe-vpc-peering-connections --query 'length(VpcPeeringConnections[?Status.Code==`active`])' --output text)

echo "Current VPCs: $VPC_COUNT"
echo "Active Peering Connections: $PEERING_COUNT"
echo "Theoretical Maximum for Full Mesh: $(( VPC_COUNT * (VPC_COUNT - 1) / 2 ))"
```

### Step 2: Design Transit Gateway Architecture
**Objective**: Plan TGW replacement for existing peering

**Create Transit Gateway**:
```bash
# Create Transit Gateway
TGW_ID=$(aws ec2 create-transit-gateway \
    --description "Migration from VPC peering to centralized routing" \
    --options DefaultRouteTableAssociation=enable,DefaultRouteTablePropagation=enable \
    --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=Migration-TGW},{Key=Purpose,Value=Peering-Replacement}]' \
    --query 'TransitGateway.TransitGatewayId' --output text)

echo "Transit Gateway created: $TGW_ID"

# Wait for TGW to be available
aws ec2 wait transit-gateway-available --transit-gateway-ids $TGW_ID

# Get default route table
DEFAULT_TGW_RT=$(aws ec2 describe-transit-gateways --transit-gateway-ids $TGW_ID \
    --query 'TransitGateways[0].Options.AssociationDefaultRouteTableId' --output text)

echo "Default TGW Route Table: $DEFAULT_TGW_RT"
```

### Step 3: Create VPCs for Migration Testing
**Objective**: Set up test environment mimicking production

```bash
# Create test VPCs with different purposes
SPOKE1_VPC=$(aws ec2 create-vpc --cidr-block 10.10.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Spoke1-VPC},{Key=Type,Value=Application}]' \
    --query 'Vpc.VpcId' --output text)

SPOKE2_VPC=$(aws ec2 create-vpc --cidr-block 10.20.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Spoke2-VPC},{Key=Type,Value=Database}]' \
    --query 'Vpc.VpcId' --output text)

SPOKE3_VPC=$(aws ec2 create-vpc --cidr-block 10.30.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Spoke3-VPC},{Key=Type,Value=Services}]' \
    --query 'Vpc.VpcId' --output text)

HUB_VPC=$(aws ec2 create-vpc --cidr-block 10.100.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=Hub-VPC},{Key=Type,Value=Management}]' \
    --query 'Vpc.VpcId' --output text)

echo "Created VPCs:"
echo "Spoke1 (App): $SPOKE1_VPC"
echo "Spoke2 (DB): $SPOKE2_VPC" 
echo "Spoke3 (Services): $SPOKE3_VPC"
echo "Hub (Mgmt): $HUB_VPC"

# Enable DNS for all VPCs
for vpc in $SPOKE1_VPC $SPOKE2_VPC $SPOKE3_VPC $HUB_VPC; do
    aws ec2 modify-vpc-attribute --vpc-id $vpc --enable-dns-support
    aws ec2 modify-vpc-attribute --vpc-id $vpc --enable-dns-hostnames
done
```

### Step 4: Implement VPC Peering (Current State)
**Objective**: Set up traditional peering architecture

```bash
# Create full mesh peering between spoke VPCs
PEER_1_2=$(aws ec2 create-vpc-peering-connection --vpc-id $SPOKE1_VPC --peer-vpc-id $SPOKE2_VPC \
    --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

PEER_1_3=$(aws ec2 create-vpc-peering-connection --vpc-id $SPOKE1_VPC --peer-vpc-id $SPOKE3_VPC \
    --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

PEER_2_3=$(aws ec2 create-vpc-peering-connection --vpc-id $SPOKE2_VPC --peer-vpc-id $SPOKE3_VPC \
    --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

# Hub connections
PEER_HUB_1=$(aws ec2 create-vpc-peering-connection --vpc-id $HUB_VPC --peer-vpc-id $SPOKE1_VPC \
    --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

PEER_HUB_2=$(aws ec2 create-vpc-peering-connection --vpc-id $HUB_VPC --peer-vpc-id $SPOKE2_VPC \
    --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

PEER_HUB_3=$(aws ec2 create-vpc-peering-connection --vpc-id $HUB_VPC --peer-vpc-id $SPOKE3_VPC \
    --query 'VpcPeeringConnection.VpcPeeringConnectionId' --output text)

# Accept all peering connections
for peer in $PEER_1_2 $PEER_1_3 $PEER_2_3 $PEER_HUB_1 $PEER_HUB_2 $PEER_HUB_3; do
    aws ec2 accept-vpc-peering-connection --vpc-peering-connection-id $peer
done

echo "Created 6 peering connections for 4-VPC full mesh"
```

**Configure Peering Routes**:
```bash
# Function to add peering routes
add_peering_routes() {
    local source_vpc=$1
    local dest_cidr=$2
    local peer_id=$3
    
    # Get main route table for source VPC
    local rt_id=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$source_vpc" "Name=association.main,Values=true" \
        --query 'RouteTables[0].RouteTableId' --output text)
    
    # Add route
    aws ec2 create-route --route-table-id $rt_id --destination-cidr-block $dest_cidr --vpc-peering-connection-id $peer_id
}

# Add all peering routes (complex for full mesh)
add_peering_routes $SPOKE1_VPC "10.20.0.0/16" $PEER_1_2
add_peering_routes $SPOKE2_VPC "10.10.0.0/16" $PEER_1_2
add_peering_routes $SPOKE1_VPC "10.30.0.0/16" $PEER_1_3
add_peering_routes $SPOKE3_VPC "10.10.0.0/16" $PEER_1_3
# ... continue for all peering connections

echo "Peering routes configured (complex management required)"
```

### Step 5: Implement Transit Gateway Solution
**Objective**: Deploy TGW as centralized routing hub

**Attach VPCs to Transit Gateway**:
```bash
# Function to attach VPC to TGW
attach_vpc_to_tgw() {
    local vpc_id=$1
    local vpc_name=$2
    
    # Get a subnet from the VPC
    local subnet_id=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$vpc_id" \
        --query 'Subnets[0].SubnetId' --output text)
    
    # Create subnet if none exists
    if [ "$subnet_id" = "None" ] || [ -z "$subnet_id" ]; then
        subnet_id=$(aws ec2 create-subnet --vpc-id $vpc_id --cidr-block "${vpc_id: -5}.1.0/24" \
            --query 'Subnet.SubnetId' --output text)
    fi
    
    # Attach VPC to TGW
    local attachment_id=$(aws ec2 create-transit-gateway-vpc-attachment \
        --transit-gateway-id $TGW_ID \
        --vpc-id $vpc_id \
        --subnet-ids $subnet_id \
        --tag-specifications "ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=$vpc_name-TGW-Attachment}]" \
        --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' --output text)
    
    echo "Attached $vpc_name ($vpc_id) to TGW: $attachment_id"
    return $attachment_id
}

# Create subnets for TGW attachments
aws ec2 create-subnet --vpc-id $SPOKE1_VPC --cidr-block 10.10.1.0/24 --availability-zone us-east-1a
aws ec2 create-subnet --vpc-id $SPOKE2_VPC --cidr-block 10.20.1.0/24 --availability-zone us-east-1a  
aws ec2 create-subnet --vpc-id $SPOKE3_VPC --cidr-block 10.30.1.0/24 --availability-zone us-east-1a
aws ec2 create-subnet --vpc-id $HUB_VPC --cidr-block 10.100.1.0/24 --availability-zone us-east-1a

# Attach all VPCs to Transit Gateway
TGW_ATTACH_SPOKE1=$(aws ec2 create-transit-gateway-vpc-attachment \
    --transit-gateway-id $TGW_ID \
    --vpc-id $SPOKE1_VPC \
    --subnet-ids $(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$SPOKE1_VPC" --query 'Subnets[0].SubnetId' --output text) \
    --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' --output text)

TGW_ATTACH_SPOKE2=$(aws ec2 create-transit-gateway-vpc-attachment \
    --transit-gateway-id $TGW_ID \
    --vpc-id $SPOKE2_VPC \
    --subnet-ids $(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$SPOKE2_VPC" --query 'Subnets[0].SubnetId' --output text) \
    --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' --output text)

TGW_ATTACH_SPOKE3=$(aws ec2 create-transit-gateway-vpc-attachment \
    --transit-gateway-id $TGW_ID \
    --vpc-id $SPOKE3_VPC \
    --subnet-ids $(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$SPOKE3_VPC" --query 'Subnets[0].SubnetId' --output text) \
    --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' --output text)

TGW_ATTACH_HUB=$(aws ec2 create-transit-gateway-vpc-attachment \
    --transit-gateway-id $TGW_ID \
    --vpc-id $HUB_VPC \
    --subnet-ids $(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$HUB_VPC" --query 'Subnets[0].SubnetId' --output text) \
    --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' --output text)

# Wait for attachments to be available
aws ec2 wait transit-gateway-attachment-available --transit-gateway-attachment-ids \
    $TGW_ATTACH_SPOKE1 $TGW_ATTACH_SPOKE2 $TGW_ATTACH_SPOKE3 $TGW_ATTACH_HUB

echo "All VPC attachments created and available"
```

**Configure TGW Route Tables**:
```bash
# With default route table, all VPCs can communicate
# Verify route propagation
aws ec2 get-transit-gateway-route-table-propagations --transit-gateway-route-table-id $DEFAULT_TGW_RT \
    --query 'TransitGatewayRouteTablePropagations[*].[TransitGatewayAttachmentId,State]' \
    --output table

# View TGW route table
aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id $DEFAULT_TGW_RT \
    --filters "Name=state,Values=active" \
    --query 'Routes[*].[DestinationCidrBlock,TransitGatewayAttachments[0].TransitGatewayAttachmentId]' \
    --output table
```

### Step 6: Configure VPC Route Tables for TGW
**Objective**: Update VPC routing to use Transit Gateway

```bash
# Function to update VPC routes for TGW
update_vpc_routes_for_tgw() {
    local vpc_id=$1
    local vpc_name=$2
    
    # Get main route table
    local rt_id=$(aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$vpc_id" "Name=association.main,Values=true" \
        --query 'RouteTables[0].RouteTableId' --output text)
    
    # Add routes to other VPC CIDRs via TGW
    for cidr in "10.10.0.0/16" "10.20.0.0/16" "10.30.0.0/16" "10.100.0.0/16"; do
        # Skip own VPC CIDR
        own_cidr=$(aws ec2 describe-vpcs --vpc-ids $vpc_id --query 'Vpcs[0].CidrBlock' --output text)
        if [ "$cidr" != "$own_cidr" ]; then
            aws ec2 create-route --route-table-id $rt_id --destination-cidr-block $cidr --transit-gateway-id $TGW_ID 2>/dev/null || true
        fi
    done
    
    echo "Updated routes for $vpc_name"
}

# Update all VPC route tables
update_vpc_routes_for_tgw $SPOKE1_VPC "Spoke1"
update_vpc_routes_for_tgw $SPOKE2_VPC "Spoke2"
update_vpc_routes_for_tgw $SPOKE3_VPC "Spoke3"
update_vpc_routes_for_tgw $HUB_VPC "Hub"

echo "All VPC route tables updated for Transit Gateway"
```

### Step 7: Implement Advanced TGW Scenarios
**Objective**: Demonstrate advanced TGW features

**Network Segmentation with Custom Route Tables**:
```bash
# Create custom TGW route table for secure isolation
SECURE_TGW_RT=$(aws ec2 create-transit-gateway-route-table --transit-gateway-id $TGW_ID \
    --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=Secure-RT},{Key=Purpose,Value=Database-Isolation}]' \
    --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' --output text)

# Associate database VPC with secure route table
aws ec2 associate-transit-gateway-route-table \
    --transit-gateway-attachment-id $TGW_ATTACH_SPOKE2 \
    --transit-gateway-route-table-id $SECURE_TGW_RT

# Disassociate from default route table
aws ec2 disassociate-transit-gateway-route-table \
    --transit-gateway-attachment-id $TGW_ATTACH_SPOKE2 \
    --transit-gateway-route-table-id $DEFAULT_TGW_RT

# Add selective routes to secure route table (only Hub access)
aws ec2 create-route --route-table-id $SECURE_TGW_RT \
    --destination-cidr-block 10.100.0.0/16 \
    --transit-gateway-attachment-id $TGW_ATTACH_HUB

echo "Database VPC isolated - only Hub access allowed"
```

**Cross-Region TGW Peering Setup**:
```bash
# Create TGW in different region for DR
DR_TGW_ID=$(aws ec2 create-transit-gateway \
    --description "DR Transit Gateway for cross-region connectivity" \
    --region us-west-2 \
    --tag-specifications 'ResourceType=transit-gateway,Tags=[{Key=Name,Value=DR-TGW}]' \
    --query 'TransitGateway.TransitGatewayId' --output text)

# Create peering connection between TGWs
TGW_PEER=$(aws ec2 create-transit-gateway-peering-attachment \
    --transit-gateway-id $TGW_ID \
    --peer-transit-gateway-id $DR_TGW_ID \
    --peer-region us-west-2 \
    --tag-specifications 'ResourceType=transit-gateway-peering-attachment,Tags=[{Key=Name,Value=Cross-Region-TGW-Peer}]' \
    --query 'TransitGatewayPeeringAttachment.TransitGatewayPeeringAttachmentId' --output text)

# Accept peering in DR region
aws ec2 accept-transit-gateway-peering-attachment \
    --transit-gateway-peering-attachment-id $TGW_PEER \
    --region us-west-2

echo "Cross-region TGW peering established: $TGW_PEER"
```

### Step 8: Performance and Cost Analysis
**Objective**: Compare performance and costs between architectures

**Performance Testing**:
```bash
# Launch test instances in each VPC for performance testing
create_test_instance() {
    local vpc_id=$1
    local name=$2
    
    # Get subnet
    local subnet_id=$(aws ec2 describe-subnets --filters "Name=vpc-id,Values=$vpc_id" \
        --query 'Subnets[0].SubnetId' --output text)
    
    # Create security group
    local sg_id=$(aws ec2 create-security-group \
        --group-name "${name}-Test-SG" \
        --description "Test security group for ${name}" \
        --vpc-id $vpc_id \
        --query 'GroupId' --output text)
    
    # Allow ICMP and SSH from all VPCs
    aws ec2 authorize-security-group-ingress --group-id $sg_id --protocol icmp --port -1 --cidr 10.0.0.0/8
    aws ec2 authorize-security-group-ingress --group-id $sg_id --protocol tcp --port 22 --cidr 10.0.0.0/8
    
    # Launch instance
    local instance_id=$(aws ec2 run-instances \
        --image-id ami-0abcdef1234567890 \
        --count 1 \
        --instance-type t3.micro \
        --key-name your-key-pair \
        --security-group-ids $sg_id \
        --subnet-id $subnet_id \
        --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=${name}-Test}]" \
        --query 'Instances[0].InstanceId' --output text)
    
    echo "Created test instance in $name: $instance_id"
    echo $instance_id
}

# Create test instances
SPOKE1_INSTANCE=$(create_test_instance $SPOKE1_VPC "Spoke1")
SPOKE2_INSTANCE=$(create_test_instance $SPOKE2_VPC "Spoke2")
HUB_INSTANCE=$(create_test_instance $HUB_VPC "Hub")

# Wait for instances
aws ec2 wait instance-running --instance-ids $SPOKE1_INSTANCE $SPOKE2_INSTANCE $HUB_INSTANCE
```

**Cost Calculation**:
```bash
# Calculate VPC Peering costs
cat << 'EOF' > cost-analysis.sh
#!/bin/bash

echo "=== Cost Analysis: VPC Peering vs Transit Gateway ==="
echo ""

# VPC Peering costs (example for 10 VPCs)
VPC_COUNT=10
PEERING_CONNECTIONS=$((VPC_COUNT * (VPC_COUNT - 1) / 2))
MONTHLY_DATA_GB=1000

echo "VPC Peering (Full Mesh for $VPC_COUNT VPCs):"
echo "- Peering connections needed: $PEERING_CONNECTIONS"
echo "- Intra-region data transfer: FREE"
echo "- Inter-region data transfer: \$0.02/GB (example)"
echo "- Management overhead: HIGH"
echo ""

# Transit Gateway costs
TGW_HOURLY=0.05
TGW_DATA_PROCESSING=0.02
HOURS_PER_MONTH=730

TGW_MONTHLY_ATTACHMENT_COST=$(echo "$VPC_COUNT * $TGW_HOURLY * $HOURS_PER_MONTH" | bc -l)
TGW_MONTHLY_DATA_COST=$(echo "$MONTHLY_DATA_GB * $TGW_DATA_PROCESSING" | bc -l)
TGW_TOTAL_MONTHLY=$(echo "$TGW_MONTHLY_ATTACHMENT_COST + $TGW_MONTHLY_DATA_COST" | bc -l)

echo "Transit Gateway for $VPC_COUNT VPCs:"
echo "- Attachment cost: \$${TGW_MONTHLY_ATTACHMENT_COST} per month"
echo "- Data processing: \$${TGW_MONTHLY_DATA_COST} per month (${MONTHLY_DATA_GB}GB)"
echo "- Total monthly cost: \$${TGW_TOTAL_MONTHLY}"
echo "- Management overhead: LOW"
echo ""

# Break-even analysis
echo "Break-even analysis:"
echo "- TGW becomes cost-effective with more complex topologies"
echo "- Consider operational costs of managing many peering connections"
echo "- TGW provides additional features (segmentation, propagation, etc.)"

EOF

chmod +x cost-analysis.sh
./cost-analysis.sh
```

## Decision Framework

### When to Choose VPC Peering
```bash
# Decision criteria script
cat << 'EOF' > decision-framework.sh
#!/bin/bash

echo "=== VPC Peering vs Transit Gateway Decision Framework ==="
echo ""

choose_peering() {
    echo "Choose VPC Peering when:"
    echo "✓ Simple point-to-point connectivity (2-5 VPCs)"
    echo "✓ Minimal latency requirements"
    echo "✓ Cost optimization for simple topologies"
    echo "✓ No need for transitive routing"
    echo "✓ Established architecture with few changes"
    echo ""
}

choose_tgw() {
    echo "Choose Transit Gateway when:"
    echo "✓ Complex network topologies (5+ VPCs)"
    echo "✓ Need for transitive routing"
    echo "✓ Network segmentation requirements"
    echo "✓ Frequent network changes"
    echo "✓ Centralized network management preferred"
    echo "✓ Integration with on-premises networks"
    echo "✓ Cross-region connectivity with hub-and-spoke"
    echo ""
}

hybrid_approach() {
    echo "Hybrid Approach considerations:"
    echo "→ Use TGW for main hub-and-spoke architecture"
    echo "→ Use VPC Peering for high-bandwidth, low-latency pairs"
    echo "→ Migrate gradually from peering to TGW"
    echo "→ Consider workload-specific requirements"
    echo ""
}

choose_peering
choose_tgw
hybrid_approach

EOF

chmod +x decision-framework.sh
./decision-framework.sh
```

### Migration Strategy

**Phased Migration Approach**:
```bash
# Migration planning script
cat << 'EOF' > migration-strategy.sh
#!/bin/bash

echo "=== Migration Strategy: VPC Peering to Transit Gateway ==="
echo ""

echo "Phase 1: Assessment and Planning"
echo "- Inventory existing peering connections"
echo "- Analyze traffic patterns and requirements"
echo "- Design target TGW architecture"
echo "- Plan network segmentation strategy"
echo ""

echo "Phase 2: TGW Deployment"
echo "- Create Transit Gateway"
echo "- Design custom route tables for segmentation"
echo "- Set up monitoring and logging"
echo ""

echo "Phase 3: Gradual Migration"
echo "- Attach VPCs to TGW (maintain peering)"
echo "- Configure TGW routing"
echo "- Test connectivity via TGW"
echo "- Update applications to use TGW routes"
echo ""

echo "Phase 4: Cutover and Cleanup"
echo "- Remove peering routes from VPC route tables"
echo "- Add TGW routes to VPC route tables"
echo "- Validate end-to-end connectivity"
echo "- Delete VPC peering connections"
echo ""

echo "Phase 5: Optimization"
echo "- Implement network segmentation"
echo "- Optimize route table design"
echo "- Set up advanced monitoring"
echo "- Document new architecture"

EOF

chmod +x migration-strategy.sh
./migration-strategy.sh
```

## Architecture Comparison Diagrams

### VPC Peering Full Mesh (Complex)
```
     VPC-A ←────────────→ VPC-B
       ↕                    ↕
       ↕    ╔═══════════════╗
       ↕    ║ 15 Peering    ║
       ↕    ║ Connections   ║
       ↕    ║ for 6 VPCs    ║
       ↕    ╚═══════════════╝
       ↕                    ↕
     VPC-F ←────────────→ VPC-C
       ↕                    ↕
       ↕                    ↕
     VPC-E ←────────────→ VPC-D
```

### Transit Gateway Hub-and-Spoke (Simple)
```
            ╔═══════════════════╗
            ║ Transit Gateway   ║
            ║ (Central Hub)     ║
            ╚═══════════════════╝
                /    |    \
               /     |     \
          VPC-A    VPC-B   VPC-C
            |       |       |
          VPC-D    VPC-E   VPC-F
```

## Key Decision Points

### Technical Considerations
1. **Scalability**: TGW wins for >5 VPCs
2. **Latency**: VPC Peering has slight advantage
3. **Bandwidth**: VPC Peering unlimited, TGW 50 Gbps per attachment
4. **Management**: TGW much simpler for complex topologies
5. **Features**: TGW provides advanced routing and segmentation

### Financial Considerations
1. **Small networks** (2-3 VPCs): VPC Peering more cost-effective
2. **Medium networks** (4-10 VPCs): Break-even point
3. **Large networks** (10+ VPCs): TGW becomes cost-effective
4. **Operational costs**: TGW reduces management overhead significantly

### Migration Timing
- **Immediate**: Critical operational issues with peering mesh
- **Planned**: During network architecture refresh
- **Gradual**: As new VPCs are added to environment
- **Strategic**: Part of cloud-first infrastructure initiative

## Troubleshooting Common Migration Issues

| Issue | Symptoms | Cause | Solution |
|-------|----------|-------|---------|
| Connectivity loss during migration | Traffic drops | Routing misconfiguration | Maintain dual paths during transition |
| TGW attachment fails | Cannot attach VPC | Subnet selection issues | Use subnet in each AZ for TGW |
| Route propagation not working | Routes missing in TGW RT | Propagation disabled | Enable route propagation on attachments |
| Cross-region connectivity issues | Inter-region traffic fails | TGW peering not configured | Set up TGW peering between regions |
| Cost increases unexpectedly | Higher than expected bills | Data processing charges | Monitor and optimize data flows |

## Best Practices Summary

### VPC Peering Best Practices
- Use for simple, stable topologies
- Plan CIDR blocks to avoid overlaps
- Document peering relationships clearly
- Monitor for unused connections

### Transit Gateway Best Practices
- Design route table strategy upfront
- Use custom route tables for segmentation
- Monitor bandwidth utilization per attachment
- Implement proper tagging and documentation
- Plan for cross-region connectivity requirements

### Migration Best Practices
- Test thoroughly in non-production environment
- Maintain service availability during transition
- Have rollback plan ready
- Monitor performance before and after migration
- Train operations team on new architecture

## Additional Resources
- [AWS Transit Gateway Documentation](https://docs.aws.amazon.com/vpc/latest/tgw/)
- [VPC Peering vs Transit Gateway](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-peering.html)
- [Transit Gateway Best Practices](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-best-design-practices.html)

## Exam Tips
- Know scaling limits: VPC Peering (125 per VPC), TGW (5,000 VPCs)
- Understand cost implications: Peering (data only), TGW (hourly + processing)
- Remember transitive routing: Not supported in peering, supported in TGW
- TGW provides centralized management and advanced features
- Consider operational complexity when choosing architecture
- Both support cross-region and cross-account connectivity