# Topic 27: AWS Transit Gateway - Advanced Architecture and Implementation

## Prerequisites
- Completed Topic 26 (Advanced VPC Peering)
- Understanding of routing protocols and network design principles
- Knowledge of multi-VPC connectivity challenges and solutions
- Familiarity with AWS networking services and hybrid connectivity

## Learning Objectives
By the end of this topic, you will be able to:
- Design complex Transit Gateway architectures for enterprise environments
- Implement advanced routing policies and traffic segmentation
- Configure inter-region Transit Gateway peering and global networks
- Optimize Transit Gateway performance and troubleshoot connectivity issues

## Theory

### AWS Transit Gateway Overview

#### Definition
AWS Transit Gateway is a network transit hub that connects VPCs and on-premises networks through a central gateway, simplifying network architecture and reducing operational complexity.

#### Core Benefits
- **Simplified Connectivity**: Single point of connection for multiple VPCs
- **Scalable Architecture**: Supports thousands of VPC connections
- **Centralized Routing**: Unified route table management
- **Transitive Routing**: Enables full mesh connectivity between attached networks
- **Multi-Account Support**: Seamless cross-account network connectivity

### Transit Gateway Architecture Components

#### Transit Gateway Attachments
```
Transit Gateway
├── VPC Attachments (up to 5,000)
├── VPN Attachments (multiple Site-to-Site VPN)
├── Direct Connect Gateway Attachments
├── Transit Gateway Peering Attachments
└── Connect Attachments (SD-WAN integration)
```

#### Route Tables and Propagation
```
Route Table Structure:
┌─────────────────┬──────────────────┬────────────────┐
│ Destination     │ Target           │ Route Type     │
├─────────────────┼──────────────────┼────────────────┤
│ 10.0.0.0/16    │ vpc-attachment   │ Propagated     │
│ 10.1.0.0/16    │ vpc-attachment   │ Propagated     │
│ 192.168.0.0/16 │ vpn-attachment   │ Static         │
│ 0.0.0.0/0      │ vpc-attachment   │ Static         │
└─────────────────┴──────────────────┴────────────────┘
```

### Advanced Transit Gateway Patterns

#### Pattern 1: Segmented Network Architecture
```
                    Transit Gateway
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   Production RT    Shared Services RT  Development RT
        │                │                │
    Prod VPCs       Shared Svcs VPCs    Dev VPCs
        │                │                │
   (Isolated)      (Hub Services)    (Isolated)
```

#### Pattern 2: Multi-Region Global Network
```
Region A TGW ←──────peering──────→ Region B TGW
     │                                  │
  ┌──┴──┐                           ┌──┴──┐
VPC A1 VPC A2                     VPC B1 VPC B2
```

#### Pattern 3: Hybrid Cloud Integration
```
On-premises Network
         │
    Direct Connect
         │
Transit Gateway Connect
         │
    Transit Gateway
         │
    ┌────┴────┐
  VPC 1    VPC 2
```

### Route Table Design Strategies

#### Isolation Strategy
- **Purpose**: Complete network segregation between environments
- **Implementation**: Separate route tables per environment
- **Use Case**: Security-sensitive workloads, compliance requirements

#### Hub-and-Spoke Strategy
- **Purpose**: Centralized services with controlled access
- **Implementation**: Shared services accessible from all spokes
- **Use Case**: DNS, monitoring, security services

#### Full Mesh Strategy
- **Purpose**: Complete connectivity between all networks
- **Implementation**: Single route table with all propagations
- **Use Case**: Development environments, collaborative workloads

## Lab Exercise: Enterprise Transit Gateway Implementation

### Lab Duration: 360 minutes

### Step 1: Design Enterprise Transit Gateway Architecture
**Objective**: Plan comprehensive Transit Gateway deployment strategy

**Architecture Planning**:
```bash
# Create Transit Gateway architecture planning assessment
cat << 'EOF' > transit-gateway-planning.sh
#!/bin/bash

echo "=== Transit Gateway Architecture Planning ==="
echo ""

echo "1. NETWORK REQUIREMENTS ANALYSIS"
echo "   Number of VPCs to connect: _______"
echo "   Number of on-premises locations: _______"
echo "   Cross-region connectivity: Yes/No"
echo "   Multi-account deployment: Yes/No"
echo "   Expected network traffic volume: _______"
echo "   Compliance requirements: SOC/PCI/HIPAA"
echo ""

echo "2. SEGMENTATION STRATEGY"
echo "   Network isolation requirements:"
echo "   - Production isolation: High/Medium/Low"
echo "   - Development isolation: High/Medium/Low"
echo "   - Shared services access: All/Limited/None"
echo "   - Cross-environment communication: Yes/No"
echo "   - Third-party network access: Yes/No"
echo ""

echo "3. ROUTING DESIGN"
echo "   Routing strategy: Isolation/Hub-Spoke/Full-Mesh"
echo "   Number of route tables needed: _______"
echo "   Static routing requirements: Yes/No"
echo "   BGP route propagation: Yes/No"
echo "   Default route handling: Centralized/Distributed"
echo ""

echo "4. PERFORMANCE REQUIREMENTS"
echo "   Expected bandwidth per attachment: _______"
echo "   Latency requirements: _______"
echo "   High availability requirements: Yes/No"
echo "   Multi-AZ deployment: Yes/No"
echo "   Burst capacity requirements: _______"
echo ""

echo "5. SECURITY CONSIDERATIONS"
echo "   Network ACLs: Yes/No"
echo "   Security group strategies: _______"
echo "   VPC Flow Logs: Yes/No"
echo "   Traffic inspection: Yes/No"
echo "   Encryption requirements: In-transit/At-rest"
echo ""

echo "6. OPERATIONAL REQUIREMENTS"
echo "   Monitoring and alerting: CloudWatch/Third-party"
echo "   Change management process: _______"
echo "   Backup and disaster recovery: Yes/No"
echo "   Cost optimization: Yes/No"
echo "   Automation requirements: Terraform/CloudFormation"

EOF

chmod +x transit-gateway-planning.sh
./transit-gateway-planning.sh
```

### Step 2: Create Transit Gateway and Route Tables
**Objective**: Deploy Transit Gateway with segmented routing architecture

**Transit Gateway Infrastructure**:
```bash
# Create Transit Gateway with advanced configuration
cat << 'EOF' > create-transit-gateway.sh
#!/bin/bash

echo "=== Creating Enterprise Transit Gateway ==="
echo ""

echo "1. CREATING TRANSIT GATEWAY"

# Create Transit Gateway with advanced settings
TGW_ID=$(aws ec2 create-transit-gateway \
    --description "Enterprise Transit Gateway for Multi-VPC Connectivity" \
    --options AmazonSideAsn=64512,AutoAcceptSharedAttachments=disable,DefaultRouteTableAssociation=disable,DefaultRouteTablePropagation=disable \
    --tag-specifications "ResourceType=transit-gateway,Tags=[{Key=Name,Value=Enterprise-Transit-Gateway},{Key=Environment,Value=Shared}]" \
    --query 'TransitGateway.TransitGatewayId' \
    --output text)

echo "✅ Transit Gateway created: $TGW_ID"

# Wait for Transit Gateway to be available
echo "Waiting for Transit Gateway to become available..."
aws ec2 wait transit-gateway-available --transit-gateway-ids $TGW_ID
echo "✅ Transit Gateway is available"

echo ""
echo "2. CREATING SEGMENTED ROUTE TABLES"

# Create Production Route Table
PROD_RT_ID=$(aws ec2 create-transit-gateway-route-table \
    --transit-gateway-id $TGW_ID \
    --tag-specifications "ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=Production-Route-Table},{Key=Environment,Value=Production}]" \
    --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' \
    --output text)

echo "✅ Production route table created: $PROD_RT_ID"

# Create Development Route Table
DEV_RT_ID=$(aws ec2 create-transit-gateway-route-table \
    --transit-gateway-id $TGW_ID \
    --tag-specifications "ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=Development-Route-Table},{Key=Environment,Value=Development}]" \
    --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' \
    --output text)

echo "✅ Development route table created: $DEV_RT_ID"

# Create Shared Services Route Table
SHARED_RT_ID=$(aws ec2 create-transit-gateway-route-table \
    --transit-gateway-id $TGW_ID \
    --tag-specifications "ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=Shared-Services-Route-Table},{Key=Environment,Value=Shared}]" \
    --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' \
    --output text)

echo "✅ Shared Services route table created: $SHARED_RT_ID"

# Create On-Premises Route Table
ONPREM_RT_ID=$(aws ec2 create-transit-gateway-route-table \
    --transit-gateway-id $TGW_ID \
    --tag-specifications "ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=OnPremises-Route-Table},{Key=Environment,Value=Hybrid}]" \
    --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' \
    --output text)

echo "✅ On-Premises route table created: $ONPREM_RT_ID"

echo ""
echo "3. CREATING VPCS FOR TESTING"

# Reuse VPCs from previous lab if available, otherwise create new ones
if [ -f "multi-vpc-config.env" ]; then
    source multi-vpc-config.env
    echo "✅ Using existing VPCs from previous lab"
else
    echo "Creating new VPCs for Transit Gateway testing..."
    
    # Create Production VPC
    PROD_VPC_ID=$(aws ec2 create-vpc \
        --cidr-block 10.10.0.0/16 \
        --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=TGW-Production-VPC}]" \
        --query 'Vpc.VpcId' \
        --output text)
    
    # Create Development VPC
    DEV_VPC_ID=$(aws ec2 create-vpc \
        --cidr-block 10.20.0.0/16 \
        --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=TGW-Development-VPC}]" \
        --query 'Vpc.VpcId' \
        --output text)
    
    # Create Shared Services VPC
    SHARED_VPC_ID=$(aws ec2 create-vpc \
        --cidr-block 10.30.0.0/16 \
        --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=TGW-Shared-Services-VPC}]" \
        --query 'Vpc.VpcId' \
        --output text)
    
    echo "✅ New VPCs created for Transit Gateway"
fi

# Save Transit Gateway configuration
cat << CONFIG > transit-gateway-config.env
export TGW_ID="$TGW_ID"
export PROD_RT_ID="$PROD_RT_ID"
export DEV_RT_ID="$DEV_RT_ID"
export SHARED_RT_ID="$SHARED_RT_ID"
export ONPREM_RT_ID="$ONPREM_RT_ID"
export PROD_VPC_ID="$PROD_VPC_ID"
export DEV_VPC_ID="$DEV_VPC_ID"
export SHARED_VPC_ID="$SHARED_VPC_ID"
CONFIG

echo ""
echo "TRANSIT GATEWAY INFRASTRUCTURE SUMMARY:"
echo "Transit Gateway: $TGW_ID"
echo "Route Tables:"
echo "  Production: $PROD_RT_ID"
echo "  Development: $DEV_RT_ID"
echo "  Shared Services: $SHARED_RT_ID"
echo "  On-Premises: $ONPREM_RT_ID"
echo ""
echo "Configuration saved to transit-gateway-config.env"

EOF

chmod +x create-transit-gateway.sh
./create-transit-gateway.sh
```

### Step 3: Attach VPCs to Transit Gateway
**Objective**: Configure VPC attachments with proper route table associations

**VPC Attachment Configuration**:
```bash
# Attach VPCs to Transit Gateway with segmentation
cat << 'EOF' > attach-vpcs-to-tgw.sh
#!/bin/bash

source transit-gateway-config.env

echo "=== Attaching VPCs to Transit Gateway ==="
echo ""

echo "1. CREATING VPC ATTACHMENTS"

# Get subnet IDs for each VPC (create if needed)
echo "Getting subnet information for VPC attachments..."

# Get or create subnets for Production VPC
PROD_SUBNET_ID=$(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$PROD_VPC_ID" \
    --query 'Subnets[0].SubnetId' \
    --output text)

if [ "$PROD_SUBNET_ID" = "None" ]; then
    PROD_SUBNET_ID=$(aws ec2 create-subnet \
        --vpc-id $PROD_VPC_ID \
        --cidr-block 10.10.1.0/24 \
        --availability-zone us-east-1a \
        --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=TGW-Production-Subnet}]" \
        --query 'Subnet.SubnetId' \
        --output text)
fi

# Get or create subnets for Development VPC
DEV_SUBNET_ID=$(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$DEV_VPC_ID" \
    --query 'Subnets[0].SubnetId' \
    --output text)

if [ "$DEV_SUBNET_ID" = "None" ]; then
    DEV_SUBNET_ID=$(aws ec2 create-subnet \
        --vpc-id $DEV_VPC_ID \
        --cidr-block 10.20.1.0/24 \
        --availability-zone us-east-1a \
        --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=TGW-Development-Subnet}]" \
        --query 'Subnet.SubnetId' \
        --output text)
fi

# Get or create subnets for Shared Services VPC
SHARED_SUBNET_ID=$(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$SHARED_VPC_ID" \
    --query 'Subnets[0].SubnetId' \
    --output text)

if [ "$SHARED_SUBNET_ID" = "None" ]; then
    SHARED_SUBNET_ID=$(aws ec2 create-subnet \
        --vpc-id $SHARED_VPC_ID \
        --cidr-block 10.30.1.0/24 \
        --availability-zone us-east-1a \
        --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=TGW-Shared-Services-Subnet}]" \
        --query 'Subnet.SubnetId' \
        --output text)
fi

echo "✅ Subnet IDs retrieved/created"

echo ""
echo "2. CREATING TRANSIT GATEWAY VPC ATTACHMENTS"

# Attach Production VPC
PROD_ATTACHMENT_ID=$(aws ec2 create-transit-gateway-vpc-attachment \
    --transit-gateway-id $TGW_ID \
    --vpc-id $PROD_VPC_ID \
    --subnet-ids $PROD_SUBNET_ID \
    --tag-specifications "ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=Production-VPC-Attachment}]" \
    --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' \
    --output text)

echo "✅ Production VPC attached: $PROD_ATTACHMENT_ID"

# Attach Development VPC
DEV_ATTACHMENT_ID=$(aws ec2 create-transit-gateway-vpc-attachment \
    --transit-gateway-id $TGW_ID \
    --vpc-id $DEV_VPC_ID \
    --subnet-ids $DEV_SUBNET_ID \
    --tag-specifications "ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=Development-VPC-Attachment}]" \
    --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' \
    --output text)

echo "✅ Development VPC attached: $DEV_ATTACHMENT_ID"

# Attach Shared Services VPC
SHARED_ATTACHMENT_ID=$(aws ec2 create-transit-gateway-vpc-attachment \
    --transit-gateway-id $TGW_ID \
    --vpc-id $SHARED_VPC_ID \
    --subnet-ids $SHARED_SUBNET_ID \
    --tag-specifications "ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=Shared-Services-VPC-Attachment}]" \
    --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' \
    --output text)

echo "✅ Shared Services VPC attached: $SHARED_ATTACHMENT_ID"

# Wait for attachments to be available
echo "Waiting for VPC attachments to become available..."
aws ec2 wait transit-gateway-attachment-available --transit-gateway-attachment-ids $PROD_ATTACHMENT_ID $DEV_ATTACHMENT_ID $SHARED_ATTACHMENT_ID
echo "✅ All VPC attachments are available"

echo ""
echo "3. ASSOCIATING ATTACHMENTS WITH ROUTE TABLES"

# Associate Production VPC with Production Route Table
aws ec2 associate-transit-gateway-route-table \
    --transit-gateway-attachment-id $PROD_ATTACHMENT_ID \
    --transit-gateway-route-table-id $PROD_RT_ID

echo "✅ Production VPC associated with Production route table"

# Associate Development VPC with Development Route Table
aws ec2 associate-transit-gateway-route-table \
    --transit-gateway-attachment-id $DEV_ATTACHMENT_ID \
    --transit-gateway-route-table-id $DEV_RT_ID

echo "✅ Development VPC associated with Development route table"

# Associate Shared Services VPC with Shared Services Route Table
aws ec2 associate-transit-gateway-route-table \
    --transit-gateway-attachment-id $SHARED_ATTACHMENT_ID \
    --transit-gateway-route-table-id $SHARED_RT_ID

echo "✅ Shared Services VPC associated with Shared Services route table"

echo ""
echo "4. CONFIGURING ROUTE PROPAGATIONS"

# Propagate Shared Services routes to Production and Development
aws ec2 enable-transit-gateway-route-table-propagation \
    --transit-gateway-route-table-id $PROD_RT_ID \
    --transit-gateway-attachment-id $SHARED_ATTACHMENT_ID

aws ec2 enable-transit-gateway-route-table-propagation \
    --transit-gateway-route-table-id $DEV_RT_ID \
    --transit-gateway-attachment-id $SHARED_ATTACHMENT_ID

echo "✅ Shared Services routes propagated to Production and Development"

# Propagate Production and Development routes to Shared Services
aws ec2 enable-transit-gateway-route-table-propagation \
    --transit-gateway-route-table-id $SHARED_RT_ID \
    --transit-gateway-attachment-id $PROD_ATTACHMENT_ID

aws ec2 enable-transit-gateway-route-table-propagation \
    --transit-gateway-route-table-id $SHARED_RT_ID \
    --transit-gateway-attachment-id $DEV_ATTACHMENT_ID

echo "✅ Production and Development routes propagated to Shared Services"

# Save attachment configuration
cat << CONFIG > tgw-attachments-config.env
export PROD_ATTACHMENT_ID="$PROD_ATTACHMENT_ID"
export DEV_ATTACHMENT_ID="$DEV_ATTACHMENT_ID"
export SHARED_ATTACHMENT_ID="$SHARED_ATTACHMENT_ID"
export PROD_SUBNET_ID="$PROD_SUBNET_ID"
export DEV_SUBNET_ID="$DEV_SUBNET_ID"
export SHARED_SUBNET_ID="$SHARED_SUBNET_ID"
CONFIG

echo ""
echo "VPC ATTACHMENT CONFIGURATION SUMMARY:"
echo "Production VPC: $PROD_VPC_ID → $PROD_ATTACHMENT_ID"
echo "Development VPC: $DEV_VPC_ID → $DEV_ATTACHMENT_ID"
echo "Shared Services VPC: $SHARED_VPC_ID → $SHARED_ATTACHMENT_ID"
echo ""
echo "Routing Configuration:"
echo "- Shared Services ↔ Production (bidirectional)"
echo "- Shared Services ↔ Development (bidirectional)"
echo "- Production ↔ Development (isolated)"
echo ""
echo "Configuration saved to tgw-attachments-config.env"

EOF

chmod +x attach-vpcs-to-tgw.sh
./attach-vpcs-to-tgw.sh
```

### Step 4: Configure Advanced Routing Policies
**Objective**: Implement sophisticated routing and traffic control

**Advanced Routing Configuration**:
```bash
# Configure advanced routing policies for Transit Gateway
cat << 'EOF' > configure-advanced-routing.sh
#!/bin/bash

source transit-gateway-config.env
source tgw-attachments-config.env

echo "=== Configuring Advanced Transit Gateway Routing ==="
echo ""

echo "1. CREATING STATIC ROUTES FOR TRAFFIC CONTROL"

# Create static route for default traffic from Production to Shared Services
aws ec2 create-transit-gateway-route \
    --route-table-id $PROD_RT_ID \
    --destination-cidr-block 0.0.0.0/0 \
    --transit-gateway-attachment-id $SHARED_ATTACHMENT_ID

echo "✅ Default route configured: Production → Shared Services"

# Create static route for internet access from Development through Shared Services
aws ec2 create-transit-gateway-route \
    --route-table-id $DEV_RT_ID \
    --destination-cidr-block 0.0.0.0/0 \
    --transit-gateway-attachment-id $SHARED_ATTACHMENT_ID

echo "✅ Default route configured: Development → Shared Services"

echo ""
echo "2. CONFIGURING BLACKHOLE ROUTES FOR SECURITY"

# Block direct communication between Production and Development
aws ec2 create-transit-gateway-route \
    --route-table-id $PROD_RT_ID \
    --destination-cidr-block 10.20.0.0/16 \
    --blackhole

aws ec2 create-transit-gateway-route \
    --route-table-id $DEV_RT_ID \
    --destination-cidr-block 10.10.0.0/16 \
    --blackhole

echo "✅ Blackhole routes configured: Production ↔ Development isolation"

echo ""
echo "3. CREATING CONDITIONAL ROUTING POLICIES"

# Create script for conditional routing based on traffic patterns
cat << 'CONDITIONAL_ROUTING' > conditional-routing-policies.sh
#!/bin/bash

echo "=== Conditional Routing Policies ==="
echo ""

echo "1. TIME-BASED ROUTING POLICY"
cat << 'TIME_POLICY' > time-based-routing.py
#!/usr/bin/env python3

import boto3
import datetime
import json

def update_routing_based_on_time():
    """Update Transit Gateway routing based on time of day"""
    ec2 = boto3.client('ec2')
    current_hour = datetime.datetime.now().hour
    
    # Business hours: 8 AM to 6 PM
    if 8 <= current_hour <= 18:
        # Enable full connectivity during business hours
        policy = "business_hours"
        print("Applying business hours routing policy")
    else:
        # Restricted connectivity after hours
        policy = "after_hours"
        print("Applying after hours routing policy")
    
    # Log the policy change
    with open('/var/log/tgw-routing.log', 'a') as f:
        f.write(f"{datetime.datetime.now()}: Applied {policy} routing policy\n")

if __name__ == "__main__":
    update_routing_based_on_time()

TIME_POLICY

chmod +x time-based-routing.py

echo "✅ Time-based routing policy created"

echo ""
echo "2. TRAFFIC-BASED ROUTING POLICY"
cat << 'TRAFFIC_POLICY' > traffic-based-routing.py
#!/usr/bin/env python3

import boto3
import json

def update_routing_based_on_traffic():
    """Update routing based on traffic patterns"""
    cloudwatch = boto3.client('cloudwatch')
    ec2 = boto3.client('ec2')
    
    # Get Transit Gateway metrics
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/TransitGateway',
        MetricName='BytesIn',
        Dimensions=[
            {
                'Name': 'TransitGateway',
                'Value': 'tgw-xxxxxxxxx'  # Replace with actual TGW ID
            }
        ],
        StartTime=datetime.datetime.utcnow() - datetime.timedelta(minutes=5),
        EndTime=datetime.datetime.utcnow(),
        Period=300,
        Statistics=['Average']
    )
    
    # Implement traffic-based routing logic
    if response['Datapoints']:
        avg_traffic = response['Datapoints'][0]['Average']
        if avg_traffic > 1000000:  # 1MB threshold
            print("High traffic detected, implementing load balancing")
            # Add additional routes or modify existing ones
        else:
            print("Normal traffic levels")

if __name__ == "__main__":
    update_routing_based_on_traffic()

TRAFFIC_POLICY

chmod +x traffic-based-routing.py

echo "✅ Traffic-based routing policy created"

CONDITIONAL_ROUTING

chmod +x conditional-routing-policies.sh

echo ""
echo "4. IMPLEMENTING ROUTE PRIORITY POLICIES"

# Create script for route priority management
cat << 'PRIORITY_ROUTING' > route-priority-management.sh
#!/bin/bash

echo "=== Route Priority Management ==="
echo ""

echo "Route Table Priority Order:"
echo "1. Static Routes (Highest Priority)"
echo "2. Propagated Routes (Lower Priority)"
echo "3. Blackhole Routes (Security Override)"
echo ""

echo "Current Route Tables Status:"
aws ec2 describe-transit-gateway-route-tables \
    --transit-gateway-route-table-ids $PROD_RT_ID $DEV_RT_ID $SHARED_RT_ID $ONPREM_RT_ID \
    --query 'TransitGatewayRouteTables[*].[TransitGatewayRouteTableId,State,Tags[?Key==`Name`].Value|[0]]' \
    --output table

echo ""
echo "Route Table Details:"
for rt_id in $PROD_RT_ID $DEV_RT_ID $SHARED_RT_ID $ONPREM_RT_ID; do
    echo "Routes in $rt_id:"
    aws ec2 search-transit-gateway-routes \
        --transit-gateway-route-table-id $rt_id \
        --filters "Name=state,Values=active" \
        --query 'Routes[*].[DestinationCidrBlock,Type,State,TransitGatewayAttachments[0].TransitGatewayAttachmentId]' \
        --output table
    echo ""
done

PRIORITY_ROUTING

chmod +x route-priority-management.sh

echo ""
echo "ADVANCED ROUTING CONFIGURATION COMPLETED"
echo ""
echo "Routing Policies Implemented:"
echo "1. Default routes for internet access through Shared Services"
echo "2. Blackhole routes for Production ↔ Development isolation"
echo "3. Conditional routing scripts for time and traffic-based policies"
echo "4. Route priority management tools"
echo ""
echo "Available scripts:"
echo "- conditional-routing-policies.sh"
echo "- route-priority-management.sh"

EOF

chmod +x configure-advanced-routing.sh
./configure-advanced-routing.sh
```

### Step 5: Implement Inter-Region Transit Gateway Peering
**Objective**: Configure global network connectivity across regions

**Inter-Region Peering Setup**:
```bash
# Configure inter-region Transit Gateway peering
cat << 'EOF' > configure-inter-region-peering.sh
#!/bin/bash

source transit-gateway-config.env

echo "=== Configuring Inter-Region Transit Gateway Peering ==="
echo ""

echo "1. INTER-REGION PEERING OVERVIEW"
echo "This configuration demonstrates cross-region TGW connectivity"
echo "Primary Region: us-east-1 (current)"
echo "Secondary Region: us-west-2 (to be configured)"
echo ""

echo "2. CREATING SECONDARY REGION TRANSIT GATEWAY"

# Note: This would typically be run in the secondary region
cat << 'SECONDARY_REGION_SETUP' > setup-secondary-region-tgw.sh
#!/bin/bash

# Switch to secondary region
export AWS_DEFAULT_REGION=us-west-2

echo "Setting up Transit Gateway in us-west-2..."

# Create Transit Gateway in secondary region
SECONDARY_TGW_ID=$(aws ec2 create-transit-gateway \
    --description "Secondary Region Transit Gateway" \
    --options AmazonSideAsn=64513 \
    --tag-specifications "ResourceType=transit-gateway,Tags=[{Key=Name,Value=Secondary-Transit-Gateway},{Key=Region,Value=us-west-2}]" \
    --query 'TransitGateway.TransitGatewayId' \
    --output text)

echo "Secondary Transit Gateway created: $SECONDARY_TGW_ID"

# Create VPC in secondary region for testing
SECONDARY_VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.100.0.0/16 \
    --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=Secondary-Region-VPC}]" \
    --query 'Vpc.VpcId' \
    --output text)

SECONDARY_SUBNET_ID=$(aws ec2 create-subnet \
    --vpc-id $SECONDARY_VPC_ID \
    --cidr-block 10.100.1.0/24 \
    --availability-zone us-west-2a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=Secondary-Region-Subnet}]" \
    --query 'Subnet.SubnetId' \
    --output text)

# Attach VPC to secondary TGW
SECONDARY_ATTACHMENT_ID=$(aws ec2 create-transit-gateway-vpc-attachment \
    --transit-gateway-id $SECONDARY_TGW_ID \
    --vpc-id $SECONDARY_VPC_ID \
    --subnet-ids $SECONDARY_SUBNET_ID \
    --tag-specifications "ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=Secondary-VPC-Attachment}]" \
    --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' \
    --output text)

echo "Secondary region setup completed"
echo "Secondary TGW: $SECONDARY_TGW_ID"
echo "Secondary VPC: $SECONDARY_VPC_ID"
echo "Secondary Attachment: $SECONDARY_ATTACHMENT_ID"

# Save secondary region config
cat << CONFIG > secondary-region-config.env
export SECONDARY_TGW_ID="$SECONDARY_TGW_ID"
export SECONDARY_VPC_ID="$SECONDARY_VPC_ID"
export SECONDARY_SUBNET_ID="$SECONDARY_SUBNET_ID"
export SECONDARY_ATTACHMENT_ID="$SECONDARY_ATTACHMENT_ID"
CONFIG

SECONDARY_REGION_SETUP

chmod +x setup-secondary-region-tgw.sh

echo "✅ Secondary region setup script created"

echo ""
echo "3. CREATING TRANSIT GATEWAY PEERING CONNECTION"

# Note: This creates the peering connection from primary to secondary region
cat << 'PEERING_SETUP' > create-tgw-peering.sh
#!/bin/bash

# Assuming secondary region TGW is already created
SECONDARY_TGW_ID="tgw-xxxxxxxxx"  # Replace with actual secondary TGW ID
SECONDARY_REGION="us-west-2"

echo "Creating peering connection between regions..."

# Create peering attachment
PEERING_ATTACHMENT_ID=$(aws ec2 create-transit-gateway-peering-attachment \
    --transit-gateway-id $TGW_ID \
    --peer-transit-gateway-id $SECONDARY_TGW_ID \
    --peer-region $SECONDARY_REGION \
    --tag-specifications "ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=Inter-Region-Peering}]" \
    --query 'TransitGatewayPeeringAttachment.TransitGatewayAttachmentId' \
    --output text)

echo "Peering attachment created: $PEERING_ATTACHMENT_ID"

echo ""
echo "IMPORTANT: Accept the peering connection in the secondary region:"
echo "aws ec2 accept-transit-gateway-peering-attachment \\"
echo "    --transit-gateway-attachment-id $PEERING_ATTACHMENT_ID \\"
echo "    --region $SECONDARY_REGION"

echo ""
echo "After acceptance, configure cross-region routes:"
echo ""
echo "In Primary Region (us-east-1):"
echo "aws ec2 create-transit-gateway-route \\"
echo "    --route-table-id $SHARED_RT_ID \\"
echo "    --destination-cidr-block 10.100.0.0/16 \\"
echo "    --transit-gateway-attachment-id $PEERING_ATTACHMENT_ID"
echo ""
echo "In Secondary Region (us-west-2):"
echo "aws ec2 create-transit-gateway-route \\"
echo "    --route-table-id <SECONDARY_RT_ID> \\"
echo "    --destination-cidr-block 10.0.0.0/8 \\"
echo "    --transit-gateway-attachment-id $PEERING_ATTACHMENT_ID"

PEERING_SETUP

chmod +x create-tgw-peering.sh

echo "✅ TGW peering setup script created"

echo ""
echo "4. CREATING GLOBAL NETWORK MONITORING"

cat << 'GLOBAL_MONITORING' > monitor-global-network.py
#!/usr/bin/env python3

import boto3
import json
from datetime import datetime, timedelta

class GlobalNetworkMonitor:
    def __init__(self):
        self.regions = ['us-east-1', 'us-west-2']
        self.tgw_ids = {}
    
    def get_tgw_status(self, region):
        """Get Transit Gateway status in a region"""
        ec2 = boto3.client('ec2', region_name=region)
        
        try:
            response = ec2.describe_transit_gateways()
            tgws = response['TransitGateways']
            
            status_info = []
            for tgw in tgws:
                status_info.append({
                    'id': tgw['TransitGatewayId'],
                    'state': tgw['State'],
                    'asn': tgw['Options']['AmazonSideAsn']
                })
            
            return status_info
        except Exception as e:
            print(f"Error getting TGW status in {region}: {e}")
            return []
    
    def get_peering_status(self, region):
        """Get peering connection status"""
        ec2 = boto3.client('ec2', region_name=region)
        
        try:
            response = ec2.describe_transit-gateway-peering-attachments()
            peerings = response['TransitGatewayPeeringAttachments']
            
            peering_info = []
            for peering in peerings:
                peering_info.append({
                    'id': peering['TransitGatewayAttachmentId'],
                    'state': peering['State'],
                    'peer_region': peering['PeerRegion'],
                    'peer_tgw': peering['PeerTransitGatewayId']
                })
            
            return peering_info
        except Exception as e:
            print(f"Error getting peering status in {region}: {e}")
            return []
    
    def monitor_global_network(self):
        """Monitor the global network status"""
        print("=== Global Network Status ===")
        print(f"Timestamp: {datetime.now()}")
        print("")
        
        for region in self.regions:
            print(f"Region: {region}")
            print("-" * 30)
            
            # TGW Status
            tgw_status = self.get_tgw_status(region)
            print("Transit Gateways:")
            for tgw in tgw_status:
                print(f"  {tgw['id']}: {tgw['state']} (ASN: {tgw['asn']})")
            
            # Peering Status
            peering_status = self.get_peering_status(region)
            print("Peering Connections:")
            for peering in peering_status:
                print(f"  {peering['id']}: {peering['state']} → {peering['peer_region']}")
            
            print("")

def main():
    monitor = GlobalNetworkMonitor()
    monitor.monitor_global_network()

if __name__ == "__main__":
    main()

GLOBAL_MONITORING

chmod +x monitor-global-network.py

echo "✅ Global network monitoring script created"

echo ""
echo "INTER-REGION PEERING CONFIGURATION COMPLETED"
echo ""
echo "Setup Steps:"
echo "1. Run setup-secondary-region-tgw.sh in us-west-2"
echo "2. Run create-tgw-peering.sh to establish peering"
echo "3. Accept peering connection in secondary region"
echo "4. Configure cross-region routes"
echo "5. Use monitor-global-network.py for ongoing monitoring"

EOF

chmod +x configure-inter-region-peering.sh
./configure-inter-region-peering.sh
```

### Step 6: Implement Advanced Monitoring and Troubleshooting
**Objective**: Set up comprehensive Transit Gateway monitoring and diagnostics

**Advanced Monitoring Setup**:
```bash
# Configure comprehensive Transit Gateway monitoring
cat << 'EOF' > setup-tgw-monitoring.sh
#!/bin/bash

source transit-gateway-config.env

echo "=== Setting Up Transit Gateway Advanced Monitoring ==="
echo ""

echo "1. CREATING CLOUDWATCH DASHBOARD FOR TRANSIT GATEWAY"

# Create comprehensive TGW dashboard
DASHBOARD_BODY=$(cat << DASHBOARD
{
    "widgets": [
        {
            "type": "metric",
            "x": 0,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "metrics": [
                    [ "AWS/TransitGateway", "BytesIn", "TransitGateway", "$TGW_ID" ],
                    [ ".", "BytesOut", ".", "." ]
                ],
                "period": 300,
                "stat": "Average",
                "region": "us-east-1",
                "title": "Transit Gateway Data Transfer"
            }
        },
        {
            "type": "metric",
            "x": 12,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "metrics": [
                    [ "AWS/TransitGateway", "PacketsIn", "TransitGateway", "$TGW_ID" ],
                    [ ".", "PacketsOut", ".", "." ]
                ],
                "period": 300,
                "stat": "Average",
                "region": "us-east-1",
                "title": "Transit Gateway Packet Count"
            }
        },
        {
            "type": "metric",
            "x": 0,
            "y": 6,
            "width": 24,
            "height": 6,
            "properties": {
                "metrics": [
                    [ "AWS/TransitGateway", "PacketDropCount", "TransitGateway", "$TGW_ID" ]
                ],
                "period": 300,
                "stat": "Sum",
                "region": "us-east-1",
                "title": "Transit Gateway Packet Drops"
            }
        }
    ]
}
DASHBOARD
)

aws cloudwatch put-dashboard \
    --dashboard-name "Transit-Gateway-Monitoring" \
    --dashboard-body "$DASHBOARD_BODY"

echo "✅ CloudWatch dashboard created: Transit-Gateway-Monitoring"

echo ""
echo "2. CREATING CLOUDWATCH ALARMS"

# High packet drop alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "TGW-High-Packet-Drops" \
    --alarm-description "Alert when TGW packet drops are high" \
    --metric-name PacketDropCount \
    --namespace AWS/TransitGateway \
    --statistic Sum \
    --period 300 \
    --threshold 1000 \
    --comparison-operator GreaterThanThreshold \
    --dimensions Name=TransitGateway,Value=$TGW_ID \
    --evaluation-periods 2

# High bandwidth utilization alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "TGW-High-Bandwidth" \
    --alarm-description "Alert when TGW bandwidth is high" \
    --metric-name BytesIn \
    --namespace AWS/TransitGateway \
    --statistic Average \
    --period 300 \
    --threshold 1000000000 \
    --comparison-operator GreaterThanThreshold \
    --dimensions Name=TransitGateway,Value=$TGW_ID \
    --evaluation-periods 3

echo "✅ CloudWatch alarms created"

echo ""
echo "3. CREATING COMPREHENSIVE MONITORING SCRIPT"

cat << 'TGW_MONITOR' > comprehensive-tgw-monitor.py
#!/usr/bin/env python3

import boto3
import json
import time
from datetime import datetime, timedelta

class TransitGatewayMonitor:
    def __init__(self, tgw_id):
        self.tgw_id = tgw_id
        self.ec2 = boto3.client('ec2')
        self.cloudwatch = boto3.client('cloudwatch')
    
    def get_tgw_status(self):
        """Get Transit Gateway overall status"""
        try:
            response = self.ec2.describe_transit_gateways(
                TransitGatewayIds=[self.tgw_id]
            )
            
            if response['TransitGateways']:
                tgw = response['TransitGateways'][0]
                return {
                    'id': tgw['TransitGatewayId'],
                    'state': tgw['State'],
                    'asn': tgw['Options']['AmazonSideAsn'],
                    'creation_time': tgw['CreationTime']
                }
            return None
        except Exception as e:
            print(f"Error getting TGW status: {e}")
            return None
    
    def get_attachments_status(self):
        """Get status of all TGW attachments"""
        try:
            response = self.ec2.describe_transit_gateway_attachments(
                Filters=[
                    {
                        'Name': 'transit-gateway-id',
                        'Values': [self.tgw_id]
                    }
                ]
            )
            
            attachments = []
            for attachment in response['TransitGatewayAttachments']:
                attachments.append({
                    'id': attachment['TransitGatewayAttachmentId'],
                    'type': attachment['ResourceType'],
                    'state': attachment['State'],
                    'resource_id': attachment.get('ResourceId', 'N/A')
                })
            
            return attachments
        except Exception as e:
            print(f"Error getting attachments: {e}")
            return []
    
    def get_route_tables_status(self):
        """Get status of all route tables"""
        try:
            response = self.ec2.describe_transit_gateway_route_tables(
                Filters=[
                    {
                        'Name': 'transit-gateway-id',
                        'Values': [self.tgw_id]
                    }
                ]
            )
            
            route_tables = []
            for rt in response['TransitGatewayRouteTables']:
                route_tables.append({
                    'id': rt['TransitGatewayRouteTableId'],
                    'state': rt['State'],
                    'default': rt['DefaultAssociationRouteTable'],
                    'propagation_default': rt['DefaultPropagationRouteTable']
                })
            
            return route_tables
        except Exception as e:
            print(f"Error getting route tables: {e}")
            return []
    
    def get_metrics(self, metric_name, hours=1):
        """Get CloudWatch metrics for TGW"""
        try:
            end_time = datetime.utcnow()
            start_time = end_time - timedelta(hours=hours)
            
            response = self.cloudwatch.get_metric_statistics(
                Namespace='AWS/TransitGateway',
                MetricName=metric_name,
                Dimensions=[
                    {
                        'Name': 'TransitGateway',
                        'Value': self.tgw_id
                    }
                ],
                StartTime=start_time,
                EndTime=end_time,
                Period=300,
                Statistics=['Average', 'Maximum']
            )
            
            return response['Datapoints']
        except Exception as e:
            print(f"Error getting metrics for {metric_name}: {e}")
            return []
    
    def generate_health_report(self):
        """Generate comprehensive health report"""
        print("=" * 60)
        print(f"TRANSIT GATEWAY HEALTH REPORT")
        print(f"Generated: {datetime.now()}")
        print("=" * 60)
        
        # Overall status
        tgw_status = self.get_tgw_status()
        if tgw_status:
            print(f"\nTRANSIT GATEWAY STATUS:")
            print(f"  ID: {tgw_status['id']}")
            print(f"  State: {tgw_status['state']}")
            print(f"  ASN: {tgw_status['asn']}")
            print(f"  Created: {tgw_status['creation_time']}")
        
        # Attachments status
        attachments = self.get_attachments_status()
        print(f"\nATTACHMENTS ({len(attachments)} total):")
        for att in attachments:
            print(f"  {att['id']}: {att['type']} - {att['state']}")
        
        # Route tables status
        route_tables = self.get_route_tables_status()
        print(f"\nROUTE TABLES ({len(route_tables)} total):")
        for rt in route_tables:
            print(f"  {rt['id']}: {rt['state']}")
        
        # Metrics summary
        print(f"\nMETRICS SUMMARY (Last Hour):")
        
        metrics = ['BytesIn', 'BytesOut', 'PacketsIn', 'PacketsOut', 'PacketDropCount']
        for metric in metrics:
            data = self.get_metrics(metric)
            if data:
                avg_val = sum([d['Average'] for d in data]) / len(data)
                max_val = max([d['Maximum'] for d in data])
                print(f"  {metric}: Avg={avg_val:.2f}, Max={max_val:.2f}")
        
        print("\n" + "=" * 60)
    
    def troubleshoot_connectivity(self, source_vpc, dest_vpc):
        """Troubleshoot connectivity between VPCs"""
        print(f"\nTROUBLESHOOTING CONNECTIVITY: {source_vpc} → {dest_vpc}")
        print("-" * 50)
        
        # Check if both VPCs are attached
        attachments = self.get_attachments_status()
        vpc_attachments = [a for a in attachments if a['resource_id'] in [source_vpc, dest_vpc]]
        
        print(f"VPC Attachments Found: {len(vpc_attachments)}")
        for att in vpc_attachments:
            print(f"  {att['resource_id']}: {att['state']}")
        
        # Additional troubleshooting steps would go here
        print("\nTroubleshooting Steps:")
        print("1. Verify both VPCs are attached and in 'available' state")
        print("2. Check route table associations and propagations")
        print("3. Verify security group rules allow traffic")
        print("4. Check NACL rules in both VPCs")
        print("5. Validate instance routing tables")

def main():
    import sys
    
    if len(sys.argv) < 2:
        print("Usage: python3 comprehensive-tgw-monitor.py <tgw-id>")
        sys.exit(1)
    
    tgw_id = sys.argv[1]
    monitor = TransitGatewayMonitor(tgw_id)
    
    # Generate health report
    monitor.generate_health_report()
    
    # Example troubleshooting
    if len(sys.argv) == 4:
        source_vpc = sys.argv[2]
        dest_vpc = sys.argv[3]
        monitor.troubleshoot_connectivity(source_vpc, dest_vpc)

if __name__ == "__main__":
    main()

TGW_MONITOR

chmod +x comprehensive-tgw-monitor.py

echo "✅ Comprehensive monitoring script created"

echo ""
echo "4. CREATING TROUBLESHOOTING TOOLKIT"

cat << 'TROUBLESHOOTING' > tgw-troubleshooting-toolkit.sh
#!/bin/bash

echo "=== Transit Gateway Troubleshooting Toolkit ==="
echo ""

TGW_ID="$1"
if [ -z "$TGW_ID" ]; then
    echo "Usage: $0 <tgw-id>"
    exit 1
fi

echo "Troubleshooting Transit Gateway: $TGW_ID"
echo ""

echo "1. BASIC CONNECTIVITY CHECKS"
echo "Checking Transit Gateway state..."
aws ec2 describe-transit-gateways \
    --transit-gateway-ids $TGW_ID \
    --query 'TransitGateways[0].[TransitGatewayId,State,Options.AmazonSideAsn]' \
    --output table

echo ""
echo "2. ATTACHMENT STATUS"
aws ec2 describe-transit-gateway-attachments \
    --filters "Name=transit-gateway-id,Values=$TGW_ID" \
    --query 'TransitGatewayAttachments[*].[TransitGatewayAttachmentId,ResourceType,State,ResourceId]' \
    --output table

echo ""
echo "3. ROUTE TABLE ANALYSIS"
aws ec2 describe-transit-gateway-route-tables \
    --filters "Name=transit-gateway-id,Values=$TGW_ID" \
    --query 'TransitGatewayRouteTables[*].[TransitGatewayRouteTableId,State,DefaultAssociationRouteTable]' \
    --output table

echo ""
echo "4. COMMON ISSUES AND SOLUTIONS"
cat << ISSUES
Common Transit Gateway Issues:

1. ATTACHMENT STUCK IN 'PENDING'
   - Check VPC subnet availability
   - Verify IAM permissions
   - Ensure no CIDR conflicts

2. NO CONNECTIVITY BETWEEN VPCS
   - Verify route table associations
   - Check route propagations
   - Validate security group rules
   - Check NACL configurations

3. INTERMITTENT CONNECTIVITY
   - Check for route conflicts
   - Verify BGP route advertisements
   - Monitor CloudWatch metrics

4. HIGH LATENCY
   - Check for routing loops
   - Verify optimal route selection
   - Monitor bandwidth utilization

5. PACKET DROPS
   - Check security group rules
   - Verify NACL configurations
   - Monitor for overloaded instances

ISSUES

echo ""
echo "5. DIAGNOSTIC COMMANDS"
echo "Get detailed route information:"
echo "aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id <rt-id> --filters Name=state,Values=active"
echo ""
echo "Get association and propagation details:"
echo "aws ec2 get-transit-gateway-route-table-associations --transit-gateway-route-table-id <rt-id>"
echo "aws ec2 get-transit-gateway-route-table-propagations --transit-gateway-route-table-id <rt-id>"

TROUBLESHOOTING

chmod +x tgw-troubleshooting-toolkit.sh

echo "✅ Troubleshooting toolkit created"

echo ""
echo "TRANSIT GATEWAY MONITORING SETUP COMPLETED"
echo ""
echo "Monitoring Components:"
echo "1. CloudWatch dashboard: Transit-Gateway-Monitoring"
echo "2. CloudWatch alarms for packet drops and bandwidth"
echo "3. Comprehensive monitoring: comprehensive-tgw-monitor.py"
echo "4. Troubleshooting toolkit: tgw-troubleshooting-toolkit.sh"
echo ""
echo "Usage Examples:"
echo "python3 comprehensive-tgw-monitor.py $TGW_ID"
echo "./tgw-troubleshooting-toolkit.sh $TGW_ID"

EOF

chmod +x setup-tgw-monitoring.sh
./setup-tgw-monitoring.sh
```

## Transit Gateway Advanced Concepts Summary

### Transit Gateway vs VPC Peering Comparison

| Feature | Transit Gateway | VPC Peering |
|---------|-----------------|-------------|
| **Connectivity** | Transitive (hub-spoke) | Non-transitive (point-to-point) |
| **Scalability** | Up to 5,000 VPC attachments | 50 peering connections per VPC |
| **Routing** | Centralized route tables | Distributed routing |
| **Multi-Account** | Native support | Requires acceptance |
| **On-Premises** | Direct integration | Requires additional setup |
| **Cost** | Hourly + data transfer | Data transfer only |

### Advanced Routing Strategies

#### Segmentation Patterns
- **Environment Isolation**: Separate route tables per environment
- **Service-based Routing**: Route tables based on service tiers
- **Compliance Separation**: Isolated routing for regulatory requirements

#### Traffic Engineering
- **Static Routes**: Override propagated routes for traffic control
- **Blackhole Routes**: Block specific traffic patterns
- **Route Priorities**: Static routes override propagated routes

### Multi-Region Architecture Benefits

#### Global Network Connectivity
- **Inter-region Peering**: Connect TGWs across regions
- **Global Route Tables**: Centralized routing across regions
- **Disaster Recovery**: Cross-region failover capabilities

#### Performance Considerations
- **Latency**: Additional hop through TGW peering
- **Bandwidth**: Shared bandwidth across peering connection
- **Costs**: Cross-region data transfer charges

## Key Takeaways
- Transit Gateway provides transitive routing between connected networks
- Route table design is critical for proper network segmentation
- Advanced routing policies enable sophisticated traffic control
- Inter-region peering enables global network architectures
- Comprehensive monitoring is essential for troubleshooting
- Cost optimization requires careful planning of attachments and data transfer

## Additional Resources
- [AWS Transit Gateway User Guide](https://docs.aws.amazon.com/vpc/latest/tgw/)
- [Transit Gateway Route Tables](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-route-tables.html)
- [Inter-Region Peering](https://docs.aws.amazon.com/vpc/latest/tgw/tgw-peering.html)

## Exam Tips
- Transit Gateway enables transitive routing unlike VPC peering
- Route tables control traffic flow between attachments
- Static routes have higher priority than propagated routes
- Blackhole routes can block specific traffic patterns
- Cross-region peering enables global connectivity
- Monitoring is essential for troubleshooting connectivity issues
- ASN must be unique for BGP operations
- Maximum bandwidth is 50 Gbps per attachment