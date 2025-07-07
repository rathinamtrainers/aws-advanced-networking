# Topic 28: Advanced VPC Routing and Traffic Engineering

## Prerequisites
- Completed Topic 27 (Transit Gateway Advanced Architecture)
- Understanding of IP routing protocols and network design
- Knowledge of AWS networking services and route table management
- Familiarity with BGP, OSPF, and static routing concepts

## Learning Objectives
By the end of this topic, you will be able to:
- Design complex routing architectures using multiple route tables
- Implement advanced traffic engineering and path selection strategies
- Configure policy-based routing and traffic steering mechanisms
- Troubleshoot complex routing scenarios and optimize network performance

## Theory

### VPC Routing Fundamentals

#### Route Table Hierarchy
```
VPC Route Table Structure:
┌─────────────────┬──────────────────┬─────────────────┬──────────────┐
│ Destination     │ Target           │ Status          │ Propagated   │
├─────────────────┼──────────────────┼─────────────────┼──────────────┤
│ 10.0.0.0/16    │ local            │ Active          │ No           │
│ 0.0.0.0/0      │ igw-xxxxxxxxx    │ Active          │ No           │
│ 192.168.0.0/16 │ vgw-xxxxxxxxx    │ Active          │ Yes          │
│ 172.16.0.0/12  │ tgw-xxxxxxxxx    │ Active          │ No           │
└─────────────────┴──────────────────┴─────────────────┴──────────────┘
```

#### Route Priority and Selection
1. **Longest Prefix Match**: Most specific route wins
2. **Local Routes**: VPC CIDR always takes precedence
3. **Static Routes**: Manual routes override propagated routes
4. **Propagated Routes**: BGP-learned routes (lowest priority)

### Advanced Routing Patterns

#### Pattern 1: Multi-Tier Application Routing
```
Internet Gateway
       │
   Public Subnet (Web Tier)
       │
   Private Subnet (App Tier)
       │
   Database Subnet (Data Tier)
       │
   VPN Gateway (On-premises)
```

#### Pattern 2: Traffic Segmentation by Service Type
```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Route     │    │   Route     │    │   Route     │
│  Table A    │    │  Table B    │    │  Table C    │
│(Critical)   │    │(Standard)   │    │(Development)│
└─────────────┘    └─────────────┘    └─────────────┘
       │                   │                   │
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Subnets    │    │  Subnets    │    │  Subnets    │
│ 10.0.1.0/24 │    │ 10.0.2.0/24 │    │ 10.0.3.0/24 │
└─────────────┘    └─────────────┘    └─────────────┘
```

#### Pattern 3: Geographic Traffic Steering
```
Application Load Balancer (Global)
            │
    ┌───────┼───────┐
    │               │
Region A         Region B
    │               │
Route Table      Route Table
(Local Pref)     (Backup Path)
```

### Traffic Engineering Strategies

#### Equal-Cost Multi-Path (ECMP) Routing
- **Implementation**: Multiple routes with same destination and metric
- **Benefits**: Load distribution and increased bandwidth
- **Use Cases**: High-availability and load balancing scenarios

#### Policy-Based Routing (PBR)
- **Source-based Routing**: Route based on source IP/subnet
- **Application-based Routing**: Different paths for different protocols
- **Time-based Routing**: Dynamic routing based on time of day

#### Quality of Service (QoS) Routing
- **Priority Routing**: Critical traffic gets preferred paths
- **Bandwidth Allocation**: Guaranteed bandwidth for specific flows
- **Latency Optimization**: Shortest path for latency-sensitive traffic

### Advanced Route Table Design

#### Subnet-Specific Routing
```bash
# Design Matrix for Route Table Assignment
Subnet Type        Route Table    Internet    VPN        TGW
Public Web         RT-Public      ✓           ✗          ✗
Private App        RT-App         Via NAT     ✓          ✓
Database           RT-DB          ✗           ✓          Limited
Management         RT-Mgmt        Via NAT     ✓          ✓
```

#### Service-Based Routing
```bash
# Service-specific routing policies
Service Type       Priority    Path Selection    Backup Path
Critical Apps      High        Direct Connect    VPN
Standard Apps      Medium      VPN              Internet
Bulk Data          Low         Internet         VPN
Backup Traffic     Lowest      VPN              Direct Connect
```

## Lab Exercise: Advanced VPC Routing Implementation

### Lab Duration: 360 minutes

### Step 1: Design Advanced Routing Architecture
**Objective**: Plan comprehensive routing strategy for multi-tier application

**Routing Architecture Planning**:
```bash
# Create advanced routing planning assessment
cat << 'EOF' > advanced-routing-planning.sh
#!/bin/bash

echo "=== Advanced VPC Routing Architecture Planning ==="
echo ""

echo "1. APPLICATION ARCHITECTURE ANALYSIS"
echo "   Application tiers: Web/App/Database/Management"
echo "   Traffic patterns: North-South/East-West"
echo "   Performance requirements: Latency/Throughput"
echo "   Security requirements: Isolation/Compliance"
echo "   Scalability requirements: Auto-scaling/Load distribution"
echo ""

echo "2. ROUTING REQUIREMENTS"
echo "   Number of route tables needed: _______"
echo "   Traffic segmentation strategy: Tier-based/Service-based"
echo "   External connectivity: Internet/VPN/Direct Connect"
echo "   Inter-VPC connectivity: Peering/Transit Gateway"
echo "   Redundancy requirements: Primary/Backup paths"
echo ""

echo "3. TRAFFIC ENGINEERING OBJECTIVES"
echo "   Load balancing: ECMP/Weighted routing"
echo "   Path optimization: Latency/Cost/Bandwidth"
echo "   Failover requirements: Automatic/Manual"
echo "   QoS requirements: Priority/Bandwidth guarantee"
echo "   Geographic distribution: Multi-region/Single-region"
echo ""

echo "4. SECURITY AND COMPLIANCE"
echo "   Network isolation requirements: Strict/Moderate/Minimal"
echo "   Traffic inspection: Required/Optional"
echo "   Audit requirements: Full logging/Selective"
echo "   Compliance standards: SOC/PCI/HIPAA"
echo "   Access control: Role-based/Network-based"
echo ""

echo "5. OPERATIONAL CONSIDERATIONS"
echo "   Change management: Automated/Manual"
echo "   Monitoring requirements: Real-time/Batch"
echo "   Troubleshooting: Automated/Manual tools"
echo "   Documentation: Auto-generated/Manual"
echo "   Training requirements: Technical/Operational"

EOF

chmod +x advanced-routing-planning.sh
./advanced-routing-planning.sh
```

### Step 2: Create Multi-Tier VPC with Advanced Routing
**Objective**: Deploy VPC with sophisticated routing architecture

**Advanced VPC Infrastructure**:
```bash
# Create advanced VPC with multi-tier routing
cat << 'EOF' > create-advanced-routing-vpc.sh
#!/bin/bash

echo "=== Creating Advanced VPC with Multi-Tier Routing ==="
echo ""

echo "1. CREATING MULTI-TIER VPC"

# Create VPC with comprehensive CIDR planning
VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.0.0.0/16 \
    --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=Advanced-Routing-VPC},{Key=Purpose,Value=Multi-Tier-Application}]" \
    --query 'Vpc.VpcId' \
    --output text)

echo "✅ VPC created: $VPC_ID"

# Enable DNS hostname and resolution
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support

echo "✅ DNS attributes configured"

echo ""
echo "2. CREATING SUBNETS WITH TIER-BASED DESIGN"

# Create subnets for different tiers across multiple AZs
# Web Tier (Public)
WEB_SUBNET_A=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.1.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=Web-Tier-Subnet-A},{Key=Tier,Value=Web}]" \
    --query 'Subnet.SubnetId' \
    --output text)

WEB_SUBNET_B=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.2.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=Web-Tier-Subnet-B},{Key=Tier,Value=Web}]" \
    --query 'Subnet.SubnetId' \
    --output text)

# Application Tier (Private)
APP_SUBNET_A=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.11.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=App-Tier-Subnet-A},{Key=Tier,Value=Application}]" \
    --query 'Subnet.SubnetId' \
    --output text)

APP_SUBNET_B=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.12.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=App-Tier-Subnet-B},{Key=Tier,Value=Application}]" \
    --query 'Subnet.SubnetId' \
    --output text)

# Database Tier (Private)
DB_SUBNET_A=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.21.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=DB-Tier-Subnet-A},{Key=Tier,Value=Database}]" \
    --query 'Subnet.SubnetId' \
    --output text)

DB_SUBNET_B=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.22.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=DB-Tier-Subnet-B},{Key=Tier,Value=Database}]" \
    --query 'Subnet.SubnetId' \
    --output text)

# Management Tier (Private)
MGMT_SUBNET_A=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.31.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=Management-Subnet-A},{Key=Tier,Value=Management}]" \
    --query 'Subnet.SubnetId' \
    --output text)

MGMT_SUBNET_B=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.32.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=Management-Subnet-B},{Key=Tier,Value=Management}]" \
    --query 'Subnet.SubnetId' \
    --output text)

echo "✅ Subnets created across all tiers"

echo ""
echo "3. CREATING GATEWAYS AND ENDPOINTS"

# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
    --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=Advanced-Routing-IGW}]" \
    --query 'InternetGateway.InternetGatewayId' \
    --output text)

aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID
echo "✅ Internet Gateway created and attached: $IGW_ID"

# Create NAT Gateways for high availability
# Allocate Elastic IPs
EIP_A=$(aws ec2 allocate-address --domain vpc --tag-specifications "ResourceType=elastic-ip,Tags=[{Key=Name,Value=NAT-Gateway-EIP-A}]" --query 'AllocationId' --output text)
EIP_B=$(aws ec2 allocate-address --domain vpc --tag-specifications "ResourceType=elastic-ip,Tags=[{Key=Name,Value=NAT-Gateway-EIP-B}]" --query 'AllocationId' --output text)

# Create NAT Gateways
NAT_GW_A=$(aws ec2 create-nat-gateway \
    --subnet-id $WEB_SUBNET_A \
    --allocation-id $EIP_A \
    --tag-specifications "ResourceType=nat-gateway,Tags=[{Key=Name,Value=NAT-Gateway-A}]" \
    --query 'NatGateway.NatGatewayId' \
    --output text)

NAT_GW_B=$(aws ec2 create-nat-gateway \
    --subnet-id $WEB_SUBNET_B \
    --allocation-id $EIP_B \
    --tag-specifications "ResourceType=nat-gateway,Tags=[{Key=Name,Value=NAT-Gateway-B}]" \
    --query 'NatGateway.NatGatewayId' \
    --output text)

echo "✅ NAT Gateways created: $NAT_GW_A, $NAT_GW_B"

# Wait for NAT Gateways to be available
echo "Waiting for NAT Gateways to become available..."
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW_A $NAT_GW_B
echo "✅ NAT Gateways are available"

# Save infrastructure configuration
cat << CONFIG > advanced-routing-config.env
export VPC_ID="$VPC_ID"
export IGW_ID="$IGW_ID"
export NAT_GW_A="$NAT_GW_A"
export NAT_GW_B="$NAT_GW_B"
export WEB_SUBNET_A="$WEB_SUBNET_A"
export WEB_SUBNET_B="$WEB_SUBNET_B"
export APP_SUBNET_A="$APP_SUBNET_A"
export APP_SUBNET_B="$APP_SUBNET_B"
export DB_SUBNET_A="$DB_SUBNET_A"
export DB_SUBNET_B="$DB_SUBNET_B"
export MGMT_SUBNET_A="$MGMT_SUBNET_A"
export MGMT_SUBNET_B="$MGMT_SUBNET_B"
CONFIG

echo ""
echo "ADVANCED VPC INFRASTRUCTURE SUMMARY:"
echo "VPC: $VPC_ID (10.0.0.0/16)"
echo "Subnets:"
echo "  Web Tier: $WEB_SUBNET_A (10.0.1.0/24), $WEB_SUBNET_B (10.0.2.0/24)"
echo "  App Tier: $APP_SUBNET_A (10.0.11.0/24), $APP_SUBNET_B (10.0.12.0/24)"
echo "  DB Tier: $DB_SUBNET_A (10.0.21.0/24), $DB_SUBNET_B (10.0.22.0/24)"
echo "  Mgmt Tier: $MGMT_SUBNET_A (10.0.31.0/24), $MGMT_SUBNET_B (10.0.32.0/24)"
echo "Gateways:"
echo "  Internet Gateway: $IGW_ID"
echo "  NAT Gateways: $NAT_GW_A, $NAT_GW_B"
echo ""
echo "Configuration saved to advanced-routing-config.env"

EOF

chmod +x create-advanced-routing-vpc.sh
./create-advanced-routing-vpc.sh
```

### Step 3: Implement Tier-Specific Route Tables
**Objective**: Configure sophisticated routing policies for each application tier

**Route Table Configuration**:
```bash
# Create tier-specific route tables with advanced policies
cat << 'EOF' > configure-tier-routing.sh
#!/bin/bash

source advanced-routing-config.env

echo "=== Configuring Tier-Specific Route Tables ==="
echo ""

echo "1. CREATING TIER-SPECIFIC ROUTE TABLES"

# Web Tier Route Table (Public)
WEB_RT_ID=$(aws ec2 create-route-table \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=Web-Tier-Route-Table},{Key=Tier,Value=Web}]" \
    --query 'RouteTable.RouteTableId' \
    --output text)

echo "✅ Web Tier route table created: $WEB_RT_ID"

# Application Tier Route Tables (Private with HA)
APP_RT_A_ID=$(aws ec2 create-route-table \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=App-Tier-Route-Table-A},{Key=Tier,Value=Application},{Key=AZ,Value=us-east-1a}]" \
    --query 'RouteTable.RouteTableId' \
    --output text)

APP_RT_B_ID=$(aws ec2 create-route-table \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=App-Tier-Route-Table-B},{Key=Tier,Value=Application},{Key=AZ,Value=us-east-1b}]" \
    --query 'RouteTable.RouteTableId' \
    --output text)

echo "✅ Application Tier route tables created: $APP_RT_A_ID, $APP_RT_B_ID"

# Database Tier Route Table (Highly Restricted)
DB_RT_ID=$(aws ec2 create-route-table \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=Database-Tier-Route-Table},{Key=Tier,Value=Database}]" \
    --query 'RouteTable.RouteTableId' \
    --output text)

echo "✅ Database Tier route table created: $DB_RT_ID"

# Management Tier Route Table (Controlled Access)
MGMT_RT_ID=$(aws ec2 create-route-table \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=Management-Route-Table},{Key=Tier,Value=Management}]" \
    --query 'RouteTable.RouteTableId' \
    --output text)

echo "✅ Management Tier route table created: $MGMT_RT_ID"

echo ""
echo "2. CONFIGURING ROUTES FOR EACH TIER"

# Web Tier - Direct Internet access
aws ec2 create-route \
    --route-table-id $WEB_RT_ID \
    --destination-cidr-block 0.0.0.0/0 \
    --gateway-id $IGW_ID

echo "✅ Web Tier: Internet access configured"

# Application Tier - NAT Gateway access with HA
aws ec2 create-route \
    --route-table-id $APP_RT_A_ID \
    --destination-cidr-block 0.0.0.0/0 \
    --nat-gateway-id $NAT_GW_A

aws ec2 create-route \
    --route-table-id $APP_RT_B_ID \
    --destination-cidr-block 0.0.0.0/0 \
    --nat-gateway-id $NAT_GW_B

echo "✅ Application Tier: HA NAT Gateway access configured"

# Database Tier - No Internet access (internal only)
echo "✅ Database Tier: Internal-only routing (no Internet access)"

# Management Tier - Controlled Internet access via NAT Gateway A
aws ec2 create-route \
    --route-table-id $MGMT_RT_ID \
    --destination-cidr-block 0.0.0.0/0 \
    --nat-gateway-id $NAT_GW_A

echo "✅ Management Tier: Controlled Internet access configured"

echo ""
echo "3. ASSOCIATING SUBNETS WITH ROUTE TABLES"

# Associate Web Tier subnets
aws ec2 associate-route-table --subnet-id $WEB_SUBNET_A --route-table-id $WEB_RT_ID
aws ec2 associate-route-table --subnet-id $WEB_SUBNET_B --route-table-id $WEB_RT_ID

# Associate Application Tier subnets (AZ-specific for HA)
aws ec2 associate-route-table --subnet-id $APP_SUBNET_A --route-table-id $APP_RT_A_ID
aws ec2 associate-route-table --subnet-id $APP_SUBNET_B --route-table-id $APP_RT_B_ID

# Associate Database Tier subnets
aws ec2 associate-route-table --subnet-id $DB_SUBNET_A --route-table-id $DB_RT_ID
aws ec2 associate-route-table --subnet-id $DB_SUBNET_B --route-table-id $DB_RT_ID

# Associate Management Tier subnets
aws ec2 associate-route-table --subnet-id $MGMT_SUBNET_A --route-table-id $MGMT_RT_ID
aws ec2 associate-route-table --subnet-id $MGMT_SUBNET_B --route-table-id $MGMT_RT_ID

echo "✅ All subnet associations completed"

echo ""
echo "4. IMPLEMENTING ADVANCED ROUTING POLICIES"

# Create conditional routing script
cat << 'CONDITIONAL_ROUTING' > implement-conditional-routing.sh
#!/bin/bash

echo "=== Implementing Conditional Routing Policies ==="
echo ""

echo "1. PRIORITY-BASED ROUTING"
cat << 'PRIORITY_SCRIPT' > priority-routing-policy.py
#!/usr/bin/env python3

import boto3
import json
from datetime import datetime

class PriorityRoutingManager:
    def __init__(self, vpc_id):
        self.vpc_id = vpc_id
        self.ec2 = boto3.client('ec2')
    
    def implement_priority_routing(self):
        """Implement priority-based routing policies"""
        
        # Define priority levels and corresponding routing policies
        priority_policies = {
            'critical': {
                'description': 'Critical applications get dedicated paths',
                'routes': [
                    {'dest': '172.16.0.0/16', 'target': 'direct_connect'},
                    {'dest': '0.0.0.0/0', 'target': 'internet_gateway'}
                ]
            },
            'standard': {
                'description': 'Standard applications use NAT Gateway',
                'routes': [
                    {'dest': '0.0.0.0/0', 'target': 'nat_gateway'}
                ]
            },
            'development': {
                'description': 'Development traffic uses lowest cost path',
                'routes': [
                    {'dest': '0.0.0.0/0', 'target': 'nat_gateway'},
                    {'dest': '192.168.0.0/16', 'target': 'vpn_gateway'}
                ]
            }
        }
        
        print("Priority Routing Policies:")
        for priority, policy in priority_policies.items():
            print(f"\n{priority.upper()} Priority:")
            print(f"  Description: {policy['description']}")
            for route in policy['routes']:
                print(f"  Route: {route['dest']} → {route['target']}")
    
    def implement_load_balancing(self):
        """Implement ECMP load balancing"""
        
        print("\n2. EQUAL-COST MULTI-PATH (ECMP) ROUTING")
        print("ECMP Routes for Load Distribution:")
        print("  Destination: 10.1.0.0/16")
        print("  Path 1: Transit Gateway Attachment A")
        print("  Path 2: Transit Gateway Attachment B")
        print("  Load Distribution: 50/50 (equal cost)")
        
        # In practice, this would create multiple routes with same destination
        # AWS automatically implements ECMP for equal-cost routes
    
    def implement_failover_routing(self):
        """Implement automatic failover routing"""
        
        print("\n3. FAILOVER ROUTING CONFIGURATION")
        print("Primary Path: Direct Connect")
        print("Backup Path: VPN Gateway")
        print("Failover Trigger: BGP route withdrawal")
        print("Recovery Time: < 30 seconds")

def main():
    import sys
    if len(sys.argv) < 2:
        print("Usage: python3 priority-routing-policy.py <vpc-id>")
        sys.exit(1)
    
    vpc_id = sys.argv[1]
    manager = PriorityRoutingManager(vpc_id)
    
    manager.implement_priority_routing()
    manager.implement_load_balancing()
    manager.implement_failover_routing()

if __name__ == "__main__":
    main()

PRIORITY_SCRIPT

chmod +x priority-routing-policy.py

echo "✅ Priority routing policy script created"

echo ""
echo "2. TIME-BASED ROUTING"
cat << 'TIME_ROUTING' > time-based-routing.py
#!/usr/bin/env python3

import boto3
import json
from datetime import datetime, time

class TimeBasedRouting:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
        self.cloudwatch = boto3.client('cloudwatch')
    
    def get_current_time_policy(self):
        """Determine routing policy based on current time"""
        current_hour = datetime.now().hour
        
        if 6 <= current_hour <= 22:  # Business hours
            return 'business_hours'
        else:  # After hours
            return 'after_hours'
    
    def apply_business_hours_routing(self):
        """Apply business hours routing policy"""
        print("BUSINESS HOURS ROUTING POLICY:")
        print("- High-priority applications: Direct Connect")
        print("- Standard applications: NAT Gateway")
        print("- Development traffic: VPN Gateway")
        print("- Backup operations: Throttled via Internet")
    
    def apply_after_hours_routing(self):
        """Apply after hours routing policy"""
        print("AFTER HOURS ROUTING POLICY:")
        print("- Critical systems only: Direct Connect")
        print("- Maintenance traffic: VPN Gateway")
        print("- Backup operations: Full bandwidth via Internet")
        print("- Development access: Restricted")
    
    def schedule_routing_changes(self):
        """Schedule automatic routing changes"""
        print("\nSCHEDULED ROUTING CHANGES:")
        print("06:00 - Switch to business hours policy")
        print("22:00 - Switch to after hours policy")
        print("02:00 - Weekly maintenance window routing")
        print("Implementation: CloudWatch Events + Lambda")

def main():
    router = TimeBasedRouting()
    
    current_policy = router.get_current_time_policy()
    print(f"Current Time Policy: {current_policy}")
    print("=" * 40)
    
    if current_policy == 'business_hours':
        router.apply_business_hours_routing()
    else:
        router.apply_after_hours_routing()
    
    router.schedule_routing_changes()

if __name__ == "__main__":
    main()

TIME_ROUTING

chmod +x time-based-routing.py

echo "✅ Time-based routing script created"

CONDITIONAL_ROUTING

chmod +x implement-conditional-routing.sh

# Save route table configuration
cat << CONFIG > route-tables-config.env
export WEB_RT_ID="$WEB_RT_ID"
export APP_RT_A_ID="$APP_RT_A_ID"
export APP_RT_B_ID="$APP_RT_B_ID"
export DB_RT_ID="$DB_RT_ID"
export MGMT_RT_ID="$MGMT_RT_ID"
CONFIG

echo ""
echo "TIER-SPECIFIC ROUTING CONFIGURATION SUMMARY:"
echo "Route Tables Created:"
echo "  Web Tier: $WEB_RT_ID (Internet Gateway)"
echo "  App Tier A: $APP_RT_A_ID (NAT Gateway A)"
echo "  App Tier B: $APP_RT_B_ID (NAT Gateway B)"
echo "  Database Tier: $DB_RT_ID (Internal only)"
echo "  Management: $MGMT_RT_ID (NAT Gateway A)"
echo ""
echo "Advanced Policies Available:"
echo "  Priority Routing: priority-routing-policy.py"
echo "  Time-based Routing: time-based-routing.py"
echo ""
echo "Configuration saved to route-tables-config.env"

EOF

chmod +x configure-tier-routing.sh
./configure-tier-routing.sh
```

### Step 4: Implement Traffic Engineering and Policy Routing
**Objective**: Configure advanced traffic engineering mechanisms

**Traffic Engineering Implementation**:
```bash
# Implement advanced traffic engineering
cat << 'EOF' > implement-traffic-engineering.sh
#!/bin/bash

source advanced-routing-config.env
source route-tables-config.env

echo "=== Implementing Advanced Traffic Engineering ==="
echo ""

echo "1. IMPLEMENTING ECMP ROUTING"

# Create script for ECMP configuration
cat << 'ECMP_CONFIG' > configure-ecmp-routing.sh
#!/bin/bash

echo "=== Configuring ECMP (Equal-Cost Multi-Path) Routing ==="
echo ""

echo "ECMP Configuration for High Availability:"
echo "- Multiple NAT Gateways for outbound traffic"
echo "- Transit Gateway multi-path routing"
echo "- Application Load Balancer multi-target routing"
echo ""

echo "Current ECMP Implementation:"
echo "1. Application Tier A → NAT Gateway A (AZ-1a)"
echo "2. Application Tier B → NAT Gateway B (AZ-1b)"
echo "3. Load distribution: Automatic per subnet"
echo ""

echo "Benefits:"
echo "- Increased bandwidth (aggregate capacity)"
echo "- Improved fault tolerance"
echo "- Better load distribution"

ECMP_CONFIG

chmod +x configure-ecmp-routing.sh

echo "✅ ECMP routing configuration created"

echo ""
echo "2. IMPLEMENTING WEIGHTED ROUTING"

cat << 'WEIGHTED_ROUTING' > implement-weighted-routing.py
#!/usr/bin/env python3

import boto3
import json

class WeightedRoutingManager:
    def __init__(self):
        self.route53 = boto3.client('route53')
        self.elb = boto3.client('elbv2')
    
    def configure_weighted_dns_routing(self):
        """Configure weighted DNS routing for traffic distribution"""
        
        weighted_routing_config = {
            'primary_region': {
                'weight': 70,
                'endpoint': 'primary-app.example.com',
                'description': 'Primary region handles 70% of traffic'
            },
            'secondary_region': {
                'weight': 30,
                'endpoint': 'secondary-app.example.com',
                'description': 'Secondary region handles 30% of traffic'
            }
        }
        
        print("WEIGHTED DNS ROUTING CONFIGURATION:")
        for region, config in weighted_routing_config.items():
            print(f"\n{region.upper()}:")
            print(f"  Weight: {config['weight']}%")
            print(f"  Endpoint: {config['endpoint']}")
            print(f"  Description: {config['description']}")
    
    def configure_weighted_load_balancer(self):
        """Configure weighted routing at load balancer level"""
        
        print("\nWEIGHTED LOAD BALANCER CONFIGURATION:")
        print("Target Group A: Weight 60 (Primary datacenter)")
        print("Target Group B: Weight 40 (Secondary datacenter)")
        print("Algorithm: Weighted round robin")
        print("Health checks: Enabled with automatic failover")
    
    def implement_gradual_traffic_shifting(self):
        """Implement gradual traffic shifting for deployments"""
        
        print("\nGRADUAL TRAFFIC SHIFTING (Blue/Green Deployment):")
        print("Phase 1: Blue 100%, Green 0% (Current production)")
        print("Phase 2: Blue 90%, Green 10% (Initial green testing)")
        print("Phase 3: Blue 70%, Green 30% (Increased green traffic)")
        print("Phase 4: Blue 50%, Green 50% (Equal distribution)")
        print("Phase 5: Blue 0%, Green 100% (Full green deployment)")

def main():
    manager = WeightedRoutingManager()
    manager.configure_weighted_dns_routing()
    manager.configure_weighted_load_balancer()
    manager.implement_gradual_traffic_shifting()

if __name__ == "__main__":
    main()

WEIGHTED_ROUTING

chmod +x implement-weighted-routing.py

echo "✅ Weighted routing implementation created"

echo ""
echo "3. IMPLEMENTING POLICY-BASED ROUTING"

cat << 'POLICY_ROUTING' > configure-policy-based-routing.sh
#!/bin/bash

echo "=== Configuring Policy-Based Routing ==="
echo ""

echo "1. SOURCE-BASED ROUTING POLICIES"
echo ""
echo "Policy 1: Management traffic routing"
echo "  Source: 10.0.31.0/24 (Management subnet)"
echo "  Destination: 0.0.0.0/0"
echo "  Action: Route via NAT Gateway A"
echo "  Priority: High"
echo ""

echo "Policy 2: Application traffic routing"
echo "  Source: 10.0.11.0/24, 10.0.12.0/24 (App subnets)"
echo "  Destination: 172.16.0.0/16"
echo "  Action: Route via Transit Gateway"
echo "  Priority: Medium"
echo ""

echo "Policy 3: Database backup traffic"
echo "  Source: 10.0.21.0/24, 10.0.22.0/24 (DB subnets)"
echo "  Destination: S3 VPC Endpoint"
echo "  Action: Route via VPC Endpoint"
echo "  Priority: Low (off-peak hours)"
echo ""

echo "2. APPLICATION-BASED ROUTING POLICIES"
echo ""
echo "HTTP/HTTPS Traffic (Ports 80,443):"
echo "  Route: Via Internet Gateway (Web tier)"
echo "  QoS: High priority, low latency"
echo ""
echo "Database Traffic (Port 3306, 5432):"
echo "  Route: Internal VPC only"
echo "  QoS: High priority, guaranteed bandwidth"
echo ""
echo "Backup Traffic (Custom ports):"
echo "  Route: Via VPN Gateway (off-peak)"
echo "  QoS: Low priority, best effort"
echo ""

echo "3. TIME-BASED ROUTING POLICIES"
echo ""
echo "Business Hours (6 AM - 10 PM):"
echo "  - Critical apps: Direct Connect"
echo "  - Standard apps: NAT Gateway"
echo "  - Development: Limited bandwidth"
echo ""
echo "Off Hours (10 PM - 6 AM):"
echo "  - Maintenance: Full bandwidth"
echo "  - Backups: Primary path"
echo "  - Development: Unrestricted"

POLICY_ROUTING

chmod +x configure-policy-based-routing.sh

echo "✅ Policy-based routing configuration created"

echo ""
echo "4. IMPLEMENTING DYNAMIC ROUTING"

cat << 'DYNAMIC_ROUTING' > implement-dynamic-routing.py
#!/usr/bin/env python3

import boto3
import json
from datetime import datetime, timedelta

class DynamicRoutingManager:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
        self.cloudwatch = boto3.client('cloudwatch')
        self.lambda_client = boto3.client('lambda')
    
    def monitor_route_performance(self):
        """Monitor route performance and trigger adjustments"""
        
        print("DYNAMIC ROUTING MONITORING:")
        print("=" * 40)
        
        monitoring_metrics = {
            'latency': {
                'threshold': '100ms',
                'action': 'Switch to low-latency path',
                'frequency': 'Every 5 minutes'
            },
            'bandwidth_utilization': {
                'threshold': '80%',
                'action': 'Enable additional paths',
                'frequency': 'Every 1 minute'
            },
            'packet_loss': {
                'threshold': '0.1%',
                'action': 'Failover to backup path',
                'frequency': 'Every 30 seconds'
            },
            'cost_threshold': {
                'threshold': '$100/hour',
                'action': 'Switch to cost-optimized path',
                'frequency': 'Every 15 minutes'
            }
        }
        
        for metric, config in monitoring_metrics.items():
            print(f"\n{metric.upper()}:")
            print(f"  Threshold: {config['threshold']}")
            print(f"  Action: {config['action']}")
            print(f"  Check Frequency: {config['frequency']}")
    
    def implement_automated_failover(self):
        """Implement automated failover mechanisms"""
        
        print("\nAUTOMATED FAILOVER CONFIGURATION:")
        print("=" * 40)
        
        failover_scenarios = [
            {
                'trigger': 'NAT Gateway failure',
                'detection_time': '30 seconds',
                'action': 'Route to backup NAT Gateway',
                'recovery_time': '60 seconds'
            },
            {
                'trigger': 'High latency (>200ms)',
                'detection_time': '5 minutes',
                'action': 'Switch to Direct Connect',
                'recovery_time': '2 minutes'
            },
            {
                'trigger': 'Bandwidth saturation (>90%)',
                'detection_time': '1 minute',
                'action': 'Enable ECMP paths',
                'recovery_time': '30 seconds'
            }
        ]
        
        for scenario in failover_scenarios:
            print(f"\nScenario: {scenario['trigger']}")
            print(f"  Detection: {scenario['detection_time']}")
            print(f"  Action: {scenario['action']}")
            print(f"  Recovery: {scenario['recovery_time']}")
    
    def create_lambda_function_template(self):
        """Create Lambda function template for dynamic routing"""
        
        lambda_function = '''
import boto3
import json

def lambda_handler(event, context):
    """
    Dynamic routing Lambda function
    Triggered by CloudWatch alarms or scheduled events
    """
    
    ec2 = boto3.client('ec2')
    
    # Parse the trigger event
    trigger_type = event.get('trigger_type', 'unknown')
    
    if trigger_type == 'high_latency':
        # Switch to low-latency path
        response = switch_to_low_latency_path()
        
    elif trigger_type == 'bandwidth_saturation':
        # Enable additional ECMP paths
        response = enable_ecmp_paths()
        
    elif trigger_type == 'cost_optimization':
        # Switch to cost-optimized path
        response = switch_to_cost_optimized_path()
        
    else:
        response = {'status': 'unknown_trigger'}
    
    return {
        'statusCode': 200,
        'body': json.dumps(response)
    }

def switch_to_low_latency_path():
    # Implementation for low-latency path switching
    return {'action': 'switched_to_low_latency'}

def enable_ecmp_paths():
    # Implementation for ECMP path enablement
    return {'action': 'enabled_ecmp_paths'}

def switch_to_cost_optimized_path():
    # Implementation for cost-optimized path switching
    return {'action': 'switched_to_cost_optimized'}
        '''
        
        with open('dynamic-routing-lambda.py', 'w') as f:
            f.write(lambda_function)
        
        print("\nLAMBDA FUNCTION TEMPLATE CREATED:")
        print("File: dynamic-routing-lambda.py")
        print("Purpose: Automated routing decisions based on triggers")
        print("Triggers: CloudWatch alarms, scheduled events")

def main():
    manager = DynamicRoutingManager()
    manager.monitor_route_performance()
    manager.implement_automated_failover()
    manager.create_lambda_function_template()

if __name__ == "__main__":
    main()

DYNAMIC_ROUTING

chmod +x implement-dynamic-routing.py

echo "✅ Dynamic routing implementation created"

echo ""
echo "TRAFFIC ENGINEERING IMPLEMENTATION COMPLETED"
echo ""
echo "Components Created:"
echo "1. ECMP routing configuration"
echo "2. Weighted routing implementation"
echo "3. Policy-based routing configuration"
echo "4. Dynamic routing with automated failover"
echo ""
echo "Available Scripts:"
echo "- configure-ecmp-routing.sh"
echo "- implement-weighted-routing.py"
echo "- configure-policy-based-routing.sh"
echo "- implement-dynamic-routing.py"

EOF

chmod +x implement-traffic-engineering.sh
./implement-traffic-engineering.sh
```

### Step 5: Implement Advanced Monitoring and Troubleshooting
**Objective**: Set up comprehensive routing monitoring and diagnostics

**Advanced Routing Monitoring**:
```bash
# Set up advanced routing monitoring and troubleshooting
cat << 'EOF' > setup-routing-monitoring.sh
#!/bin/bash

source advanced-routing-config.env

echo "=== Setting Up Advanced Routing Monitoring ==="
echo ""

echo "1. CREATING ROUTE MONITORING DASHBOARD"

# Create comprehensive routing dashboard
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
                    [ "AWS/NatGateway", "BytesInFromDestination", "NatGatewayId", "$NAT_GW_A" ],
                    [ ".", ".", ".", "$NAT_GW_B" ]
                ],
                "period": 300,
                "stat": "Average",
                "region": "us-east-1",
                "title": "NAT Gateway Traffic"
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
                    [ "AWS/NatGateway", "PacketsDropCount", "NatGatewayId", "$NAT_GW_A" ],
                    [ ".", ".", ".", "$NAT_GW_B" ]
                ],
                "period": 300,
                "stat": "Sum",
                "region": "us-east-1",
                "title": "NAT Gateway Packet Drops"
            }
        }
    ]
}
DASHBOARD
)

aws cloudwatch put-dashboard \
    --dashboard-name "Advanced-Routing-Monitoring" \
    --dashboard-body "$DASHBOARD_BODY"

echo "✅ CloudWatch dashboard created: Advanced-Routing-Monitoring"

echo ""
echo "2. CREATING ROUTING ANALYSIS TOOLS"

cat << 'ROUTING_ANALYSIS' > routing-analysis-toolkit.py
#!/usr/bin/env python3

import boto3
import json
import ipaddress
from datetime import datetime, timedelta

class RoutingAnalysisToolkit:
    def __init__(self, vpc_id):
        self.vpc_id = vpc_id
        self.ec2 = boto3.client('ec2')
        self.cloudwatch = boto3.client('cloudwatch')
    
    def analyze_route_tables(self):
        """Analyze all route tables in the VPC"""
        try:
            response = self.ec2.describe_route_tables(
                Filters=[
                    {
                        'Name': 'vpc-id',
                        'Values': [self.vpc_id]
                    }
                ]
            )
            
            print("ROUTE TABLE ANALYSIS:")
            print("=" * 50)
            
            for rt in response['RouteTables']:
                rt_id = rt['RouteTableId']
                
                # Get route table name from tags
                name = 'Unnamed'
                for tag in rt.get('Tags', []):
                    if tag['Key'] == 'Name':
                        name = tag['Value']
                        break
                
                print(f"\nRoute Table: {rt_id} ({name})")
                print(f"Associated Subnets: {len(rt['Associations'])}")
                
                # Analyze routes
                print("Routes:")
                for route in rt['Routes']:
                    dest = route.get('DestinationCidrBlock', 'N/A')
                    state = route.get('State', 'N/A')
                    
                    # Determine target
                    target = 'Unknown'
                    if 'GatewayId' in route:
                        target = route['GatewayId']
                    elif 'NatGatewayId' in route:
                        target = route['NatGatewayId']
                    elif 'TransitGatewayId' in route:
                        target = route['TransitGatewayId']
                    elif 'InstanceId' in route:
                        target = route['InstanceId']
                    elif 'VpcPeeringConnectionId' in route:
                        target = route['VpcPeeringConnectionId']
                    
                    print(f"  {dest} → {target} ({state})")
        
        except Exception as e:
            print(f"Error analyzing route tables: {e}")
    
    def detect_routing_conflicts(self):
        """Detect potential routing conflicts"""
        try:
            response = self.ec2.describe_route_tables(
                Filters=[{'Name': 'vpc-id', 'Values': [self.vpc_id]}]
            )
            
            print("\nROUTING CONFLICT ANALYSIS:")
            print("=" * 50)
            
            all_routes = []
            
            # Collect all routes
            for rt in response['RouteTables']:
                for route in rt['Routes']:
                    if 'DestinationCidrBlock' in route:
                        all_routes.append({
                            'route_table': rt['RouteTableId'],
                            'destination': route['DestinationCidrBlock'],
                            'target': route.get('GatewayId', route.get('NatGatewayId', 'Unknown')),
                            'state': route.get('State', 'Unknown')
                        })
            
            # Check for overlapping routes
            conflicts_found = False
            for i, route1 in enumerate(all_routes):
                for j, route2 in enumerate(all_routes[i+1:], i+1):
                    if self.check_cidr_overlap(route1['destination'], route2['destination']):
                        if route1['target'] != route2['target']:
                            print(f"CONFLICT DETECTED:")
                            print(f"  Route 1: {route1['destination']} → {route1['target']} (RT: {route1['route_table']})")
                            print(f"  Route 2: {route2['destination']} → {route2['target']} (RT: {route2['route_table']})")
                            conflicts_found = True
            
            if not conflicts_found:
                print("No routing conflicts detected.")
        
        except Exception as e:
            print(f"Error detecting conflicts: {e}")
    
    def check_cidr_overlap(self, cidr1, cidr2):
        """Check if two CIDR blocks overlap"""
        try:
            network1 = ipaddress.IPv4Network(cidr1, strict=False)
            network2 = ipaddress.IPv4Network(cidr2, strict=False)
            return network1.overlaps(network2)
        except:
            return False
    
    def analyze_traffic_patterns(self):
        """Analyze traffic patterns using VPC Flow Logs"""
        print("\nTRAFFIC PATTERN ANALYSIS:")
        print("=" * 50)
        
        # This would typically analyze VPC Flow Logs
        print("Traffic Analysis Recommendations:")
        print("1. Enable VPC Flow Logs for detailed traffic analysis")
        print("2. Use CloudWatch Insights for traffic pattern queries")
        print("3. Monitor top talkers and traffic flows")
        print("4. Identify optimization opportunities")
        
        sample_queries = [
            "Top source IPs by bytes transferred",
            "Most common destination ports",
            "Traffic patterns by time of day",
            "Cross-subnet communication patterns"
        ]
        
        print("\nSample VPC Flow Log Queries:")
        for query in sample_queries:
            print(f"- {query}")
    
    def generate_optimization_recommendations(self):
        """Generate routing optimization recommendations"""
        print("\nROUTING OPTIMIZATION RECOMMENDATIONS:")
        print("=" * 50)
        
        recommendations = [
            {
                'category': 'Performance',
                'recommendations': [
                    'Implement ECMP for load distribution',
                    'Use local gateways for same-AZ traffic',
                    'Optimize route table associations',
                    'Consider Transit Gateway for complex topologies'
                ]
            },
            {
                'category': 'Cost',
                'recommendations': [
                    'Use VPC Endpoints for AWS services',
                    'Optimize NAT Gateway placement',
                    'Consider Regional vs AZ-specific routing',
                    'Implement data transfer optimization'
                ]
            },
            {
                'category': 'Security',
                'recommendations': [
                    'Implement least-privilege routing',
                    'Use private subnets for internal resources',
                    'Consider network ACLs for additional security',
                    'Implement traffic segmentation'
                ]
            },
            {
                'category': 'Reliability',
                'recommendations': [
                    'Implement multi-AZ routing redundancy',
                    'Use health checks for route failover',
                    'Monitor route table propagation',
                    'Implement automated recovery procedures'
                ]
            }
        ]
        
        for rec in recommendations:
            print(f"\n{rec['category'].upper()} OPTIMIZATION:")
            for item in rec['recommendations']:
                print(f"  - {item}")

def main():
    import sys
    
    if len(sys.argv) < 2:
        print("Usage: python3 routing-analysis-toolkit.py <vpc-id>")
        sys.exit(1)
    
    vpc_id = sys.argv[1]
    toolkit = RoutingAnalysisToolkit(vpc_id)
    
    toolkit.analyze_route_tables()
    toolkit.detect_routing_conflicts()
    toolkit.analyze_traffic_patterns()
    toolkit.generate_optimization_recommendations()

if __name__ == "__main__":
    main()

ROUTING_ANALYSIS

chmod +x routing-analysis-toolkit.py

echo "✅ Routing analysis toolkit created"

echo ""
echo "3. CREATING TROUBLESHOOTING SCRIPTS"

cat << 'TROUBLESHOOTING' > routing-troubleshooting-toolkit.sh
#!/bin/bash

echo "=== Advanced Routing Troubleshooting Toolkit ==="
echo ""

VPC_ID="$1"
if [ -z "$VPC_ID" ]; then
    echo "Usage: $0 <vpc-id> [source-ip] [destination-ip]"
    exit 1
fi

SOURCE_IP="$2"
DEST_IP="$3"

echo "Troubleshooting VPC: $VPC_ID"
echo ""

echo "1. BASIC ROUTING INFORMATION"
echo "Route Tables in VPC:"
aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query 'RouteTables[*].[RouteTableId,Tags[?Key==`Name`].Value|[0]]' \
    --output table

echo ""
echo "2. SUBNET ASSOCIATIONS"
aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query 'RouteTables[*].{RouteTable:RouteTableId,Subnets:Associations[?SubnetId!=null].SubnetId}' \
    --output table

echo ""
echo "3. ROUTE ANALYSIS"
aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query 'RouteTables[*].Routes[*].[DestinationCidrBlock,GatewayId,NatGatewayId,State]' \
    --output table

if [ -n "$SOURCE_IP" ] && [ -n "$DEST_IP" ]; then
    echo ""
    echo "4. CONNECTIVITY TROUBLESHOOTING"
    echo "Source: $SOURCE_IP"
    echo "Destination: $DEST_IP"
    echo ""
    
    echo "Troubleshooting Steps:"
    echo "1. Check route table for source subnet"
    echo "2. Verify route exists for destination"
    echo "3. Check security group rules"
    echo "4. Verify NACL configurations"
    echo "5. Test with VPC Reachability Analyzer"
fi

echo ""
echo "5. COMMON ISSUES AND SOLUTIONS"
cat << ISSUES

COMMON ROUTING ISSUES:

1. NO ROUTE TO DESTINATION
   - Check route table for destination CIDR
   - Verify route target is available
   - Check for more specific routes

2. ASYMMETRIC ROUTING
   - Verify return path exists
   - Check route table associations
   - Validate security group rules

3. ROUTE CONFLICTS
   - Check for overlapping CIDR blocks
   - Verify route priorities
   - Review propagated vs static routes

4. NAT GATEWAY ISSUES
   - Verify NAT Gateway is in public subnet
   - Check route to Internet Gateway
   - Validate Elastic IP association

5. TRANSIT GATEWAY ROUTING
   - Check attachment associations
   - Verify route propagation
   - Review route table policies

TROUBLESHOOTING COMMANDS:

# Test connectivity with Reachability Analyzer
aws ec2 create-network-insights-path \
    --source <source-resource-id> \
    --destination <destination-resource-id>

# Analyze path
aws ec2 start-network-insights-analysis \
    --network-insights-path-id <path-id>

# Enable VPC Flow Logs
aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids $VPC_ID \
    --traffic-type ALL \
    --log-destination-type cloud-watch-logs \
    --log-group-name VPC-FlowLogs

ISSUES

TROUBLESHOOTING

chmod +x routing-troubleshooting-toolkit.sh

echo "✅ Troubleshooting toolkit created"

echo ""
echo "ROUTING MONITORING SETUP COMPLETED"
echo ""
echo "Monitoring Components:"
echo "1. CloudWatch dashboard: Advanced-Routing-Monitoring"
echo "2. Routing analysis toolkit: routing-analysis-toolkit.py"
echo "3. Troubleshooting toolkit: routing-troubleshooting-toolkit.sh"
echo ""
echo "Usage Examples:"
echo "python3 routing-analysis-toolkit.py $VPC_ID"
echo "./routing-troubleshooting-toolkit.sh $VPC_ID"

EOF

chmod +x setup-routing-monitoring.sh
./setup-routing-monitoring.sh
```

## Advanced VPC Routing Summary

### Route Table Design Principles

#### Hierarchical Routing Structure
- **Tier-based Routing**: Separate route tables per application tier
- **Security-focused Design**: Least-privilege routing principles
- **High Availability**: Multi-AZ routing redundancy
- **Performance Optimization**: Direct paths for critical traffic

#### Route Priority and Selection
1. **Local Routes**: VPC CIDR (highest priority)
2. **Longest Prefix Match**: Most specific route wins
3. **Static Routes**: Manual routes override propagated
4. **Propagated Routes**: BGP-learned routes (lowest priority)

### Traffic Engineering Strategies

#### Equal-Cost Multi-Path (ECMP)
- **Load Distribution**: Multiple paths with same cost
- **Bandwidth Aggregation**: Combined capacity utilization
- **High Availability**: Automatic failover between paths

#### Policy-Based Routing
- **Source-based**: Route based on source IP/subnet
- **Application-based**: Different paths per protocol/port
- **Time-based**: Dynamic routing based on schedules

#### Weighted Routing
- **Traffic Distribution**: Percentage-based traffic splitting
- **Blue/Green Deployments**: Gradual traffic migration
- **Performance Testing**: Controlled traffic distribution

### Advanced Routing Patterns

#### Multi-Tier Application Architecture
```
Internet → Web Tier → Application Tier → Database Tier
   ↓         ↓              ↓              ↓
  IGW    NAT Gateway    No Internet    Internal Only
```

#### Hub-and-Spoke with Shared Services
```
Shared Services ← → Production
      ↕               ↕
  Management    ←→  Development
```

#### Geographic Traffic Distribution
```
Global Load Balancer
    ↓        ↓
Region A   Region B
    ↓        ↓
Local RT   Local RT
```

## Key Takeaways
- Route table design is critical for security and performance
- Multi-tier routing enables application segmentation
- Traffic engineering optimizes path selection and load distribution
- Monitoring and troubleshooting are essential for complex routing
- Policy-based routing enables sophisticated traffic control
- ECMP provides load balancing and high availability
- Dynamic routing adapts to changing network conditions

## Additional Resources
- [VPC Route Tables User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html)
- [VPC Routing Best Practices](https://aws.amazon.com/blogs/networking-and-content-delivery/vpc-routing-best-practices/)
- [Advanced VPC Design Patterns](https://docs.aws.amazon.com/whitepapers/latest/building-scalable-secure-multi-vpc-network-infrastructure/welcome.html)

## Exam Tips
- Local routes always have highest priority (VPC CIDR)
- Longest prefix match determines route selection
- Static routes override propagated routes
- Each subnet can only be associated with one route table
- Route propagation is automatic for supported gateways
- NACL and security groups work together with routing
- VPC Reachability Analyzer helps troubleshoot connectivity
- Route tables are regional resources within a VPC