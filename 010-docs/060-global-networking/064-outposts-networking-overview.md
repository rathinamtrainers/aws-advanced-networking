# Topic 64: AWS Outposts Networking - Hybrid Connectivity and Architecture

## Table of Contents
1. [AWS Outposts Overview](#overview)
2. [Outposts Networking Architecture](#architecture)
3. [Local Gateway Configuration](#local-gateway)
4. [Service Links and Connectivity](#service-links)
5. [Hybrid Networking Patterns](#hybrid-patterns)
6. [VPC Extension to Outposts](#vpc-extension)
7. [Security and Compliance](#security)
8. [Monitoring and Management](#monitoring)
9. [Performance Optimization](#performance)
10. [Troubleshooting](#troubleshooting)
11. [Best Practices](#best-practices)
12. [Implementation Examples](#implementation)

## AWS Outposts Overview {#overview}

### What is AWS Outposts?

AWS Outposts is a fully managed service that extends AWS infrastructure, services, APIs, and tools to customer premises for a truly consistent hybrid experience. Outposts bring native AWS services, infrastructure, and operating models to virtually any data center, co-location space, or on-premises facility.

**Key Components**:

```
AWS Outposts Architecture Overview:

                    ┌─────────────────────────────────┐
                    │         AWS Region              │
                    │                                 │
                    │  ┌─────────────────────────┐   │
                    │  │      AWS Services       │   │
                    │  │  • Control Plane        │   │
                    │  │  • Management APIs      │   │
                    │  │  • Service Catalog      │   │
                    │  │  • CloudFormation       │   │
                    │  │  • Systems Manager      │   │
                    │  └─────────────────────────┘   │
                    └─────────────────┬───────────────┘
                                      │
                              Service Link Connection
                           (Encrypted VPN or Direct Connect)
                                      │
                    ┌─────────────────▼───────────────┐
                    │     Customer Premises           │
                    │                                 │
                    │  ┌─────────────────────────┐   │
                    │  │    AWS Outposts Rack    │   │
                    │  │                         │   │
                    │  │  ┌─────────────────┐   │   │
                    │  │  │  AWS Services   │   │   │
                    │  │  │  • EC2          │   │   │
                    │  │  │  • EBS          │   │   │
                    │  │  │  • S3 on        │   │   │
                    │  │  │    Outposts     │   │   │
                    │  │  │  • ECS/EKS      │   │   │
                    │  │  │  • RDS          │   │   │
                    │  │  └─────────────────┘   │   │
                    │  │                         │   │
                    │  │  ┌─────────────────┐   │   │
                    │  │  │ Local Gateway   │   │   │
                    │  │  │ (Networking)    │   │   │
                    │  │  └─────────────────┘   │   │
                    │  └─────────────────────────┘   │
                    │                                 │
                    │  ┌─────────────────────────┐   │
                    │  │  On-Premises Network    │   │
                    │  │  • Existing Systems     │   │
                    │  │  • Local Resources      │   │
                    │  │  • Corporate Network    │   │
                    │  └─────────────────────────┘   │
                    └─────────────────────────────────┘
```

### Outposts Networking Benefits

**Hybrid Integration**:
- Seamless extension of AWS VPC to on-premises
- Native AWS networking constructs (subnets, security groups, NACLs)
- Direct access to on-premises resources
- Consistent networking APIs and tools

**Low Latency Access**:
- Single-digit millisecond latency to on-premises systems
- Local data processing and storage
- Reduced data transfer costs
- Improved application performance

**Data Residency and Compliance**:
- Keep sensitive data on-premises
- Meet regulatory requirements
- Local data processing capabilities
- Compliance with data sovereignty laws

### Outposts Networking Components

```yaml
Outposts_Networking_Components:
  
  Local_Gateway:
    Purpose: "Bridge between Outposts and on-premises network"
    Functions:
      - Route traffic between Outposts subnets and on-premises
      - Provide connectivity to existing infrastructure
      - Enable hybrid networking patterns
      - Support multiple VPCs on single Outpost
    
    Routing_Tables:
      - Outpost_to_OnPremises: "Routes from Outpost subnets to local network"
      - OnPremises_to_Outpost: "Routes from local network to Outpost subnets"
      - Cross_VPC: "Routes between multiple VPCs on same Outpost"
    
  Service_Link:
    Purpose: "Connection between Outpost and AWS Region"
    Requirements:
      - Bandwidth: "Minimum 1 Gbps, recommended 10 Gbps"
      - Connectivity: "VPN or Direct Connect"
      - Encryption: "Always encrypted in transit"
      - Redundancy: "Multiple connections recommended"
    
    Traffic_Types:
      - Control_Plane: "Management and monitoring traffic"
      - Service_Communication: "API calls to regional services"
      - Data_Sync: "EBS snapshots, AMI updates"
      - Logging_Monitoring: "CloudWatch logs and metrics"
  
  VPC_Extension:
    Purpose: "Extend regional VPC to Outpost subnets"
    Configuration:
      - Subnet_Creation: "Create subnets in Outpost availability zone"
      - Route_Table_Association: "Associate with appropriate route tables"
      - Security_Group_Application: "Apply security groups to resources"
      - NACL_Configuration: "Network ACLs for additional security"
```

## Outposts Networking Architecture {#architecture}

### Network Topology Design

```python
import boto3
import json
from typing import Dict, List, Optional

class OutpostsNetworkingManager:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
        self.outposts = boto3.client('outposts')
        
    def discover_outposts(self) -> Dict:
        """Discover available Outposts and their networking configuration"""
        
        try:
            # Get Outposts information
            outposts_response = self.outposts.list_outposts()
            
            outposts_info = {}
            
            for outpost in outposts_response['Outposts']:
                outpost_id = outpost['OutpostId']
                outpost_arn = outpost['OutpostArn']
                
                # Get availability zone information
                az_info = self._get_outpost_availability_zone(outpost_id)
                
                # Get local gateway information
                lgw_info = self._get_local_gateway_info(outpost_id)
                
                # Get VPC subnet information
                subnet_info = self._get_outpost_subnets(outpost_id)
                
                outposts_info[outpost_id] = {
                    'outpost_id': outpost_id,
                    'outpost_arn': outpost_arn,
                    'name': outpost.get('Name', 'Unnamed'),
                    'site_id': outpost.get('SiteId'),
                    'availability_zone': az_info,
                    'local_gateway': lgw_info,
                    'subnets': subnet_info,
                    'supported_instance_types': outpost.get('SupportedHardwareType'),
                    'lifecycle_status': outpost.get('LifeCycleStatus')
                }
            
            return {
                'total_outposts': len(outposts_info),
                'outposts': outposts_info
            }
            
        except Exception as e:
            return {'error': str(e)}
    
    def _get_outpost_availability_zone(self, outpost_id: str) -> Dict:
        """Get availability zone information for Outpost"""
        
        try:
            # Outposts create their own AZ
            response = self.ec2.describe_availability_zones(
                Filters=[
                    {'Name': 'zone-type', 'Values': ['outpost']}
                ]
            )
            
            for az in response['AvailabilityZones']:
                if outpost_id in az.get('ZoneName', ''):
                    return {
                        'zone_name': az['ZoneName'],
                        'zone_id': az['ZoneId'],
                        'state': az['State'],
                        'region_name': az['RegionName']
                    }
            
            return {'status': 'not_found'}
            
        except Exception as e:
            return {'error': str(e)}
    
    def _get_local_gateway_info(self, outpost_id: str) -> Dict:
        """Get local gateway information for Outpost"""
        
        try:
            # Get local gateways
            response = self.ec2.describe_local_gateways(
                Filters=[
                    {'Name': 'outpost-arn', 'Values': [f'*{outpost_id}*']}
                ]
            )
            
            if response['LocalGateways']:
                lgw = response['LocalGateways'][0]
                
                # Get local gateway route tables
                rt_response = self.ec2.describe_local_gateway_route_tables(
                    LocalGatewayRouteTableIds=[lgw['LocalGatewayRouteTableId']]
                )
                
                return {
                    'local_gateway_id': lgw['LocalGatewayId'],
                    'state': lgw['State'],
                    'outpost_arn': lgw['OutpostArn'],
                    'route_table_id': lgw['LocalGatewayRouteTableId'],
                    'route_table_info': rt_response['LocalGatewayRouteTables'][0] if rt_response['LocalGatewayRouteTables'] else {}
                }
            
            return {'status': 'not_configured'}
            
        except Exception as e:
            return {'error': str(e)}
    
    def _get_outpost_subnets(self, outpost_id: str) -> List[Dict]:
        """Get VPC subnets configured on Outpost"""
        
        try:
            response = self.ec2.describe_subnets(
                Filters=[
                    {'Name': 'outpost-arn', 'Values': [f'*{outpost_id}*']}
                ]
            )
            
            subnets = []
            for subnet in response['Subnets']:
                subnets.append({
                    'subnet_id': subnet['SubnetId'],
                    'vpc_id': subnet['VpcId'],
                    'cidr_block': subnet['CidrBlock'],
                    'availability_zone': subnet['AvailabilityZone'],
                    'state': subnet['State'],
                    'available_ip_address_count': subnet['AvailableIpAddressCount']
                })
            
            return subnets
            
        except Exception as e:
            return [{'error': str(e)}]
    
    def design_outposts_network_architecture(self, 
                                           requirements: Dict) -> Dict:
        """Design comprehensive network architecture for Outposts deployment"""
        
        architecture = {
            'vpc_configuration': {},
            'subnet_design': {},
            'local_gateway_routing': {},
            'security_configuration': {},
            'connectivity_requirements': {},
            'implementation_plan': []
        }
        
        # VPC Configuration
        architecture['vpc_configuration'] = {
            'regional_vpc': {
                'cidr_block': requirements.get('regional_vpc_cidr', '10.0.0.0/16'),
                'enable_dns_hostnames': True,
                'enable_dns_support': True,
                'tenancy': 'default'
            },
            'outpost_extension': {
                'subnet_cidr_blocks': self._calculate_outpost_subnet_cidrs(
                    requirements.get('regional_vpc_cidr', '10.0.0.0/16'),
                    requirements.get('outpost_subnet_count', 2)
                ),
                'availability_zone': requirements.get('outpost_availability_zone')
            }
        }
        
        # Subnet Design
        architecture['subnet_design'] = {
            'regional_subnets': {
                'public_subnets': [
                    {'cidr': '10.0.1.0/24', 'az': 'us-east-1a'},
                    {'cidr': '10.0.2.0/24', 'az': 'us-east-1b'}
                ],
                'private_subnets': [
                    {'cidr': '10.0.10.0/24', 'az': 'us-east-1a'},
                    {'cidr': '10.0.11.0/24', 'az': 'us-east-1b'}
                ]
            },
            'outpost_subnets': [
                {
                    'cidr': '10.0.100.0/24',
                    'type': 'private',
                    'purpose': 'Workload subnet'
                },
                {
                    'cidr': '10.0.101.0/24', 
                    'type': 'private',
                    'purpose': 'Management subnet'
                }
            ]
        }
        
        # Local Gateway Routing
        architecture['local_gateway_routing'] = {
            'outpost_to_onpremises': {
                'destination': requirements.get('onpremises_cidr', '192.168.0.0/16'),
                'target': 'local_gateway',
                'purpose': 'Access to existing on-premises resources'
            },
            'onpremises_to_outpost': {
                'source': requirements.get('onpremises_cidr', '192.168.0.0/16'),
                'destinations': architecture['subnet_design']['outpost_subnets'],
                'routing_method': 'static_routes'
            },
            'cross_vpc_communication': {
                'enabled': requirements.get('enable_cross_vpc', False),
                'vpc_peering': requirements.get('vpc_peering_required', False)
            }
        }
        
        # Security Configuration
        architecture['security_configuration'] = {
            'security_groups': self._design_security_groups(requirements),
            'network_acls': self._design_network_acls(requirements),
            'local_gateway_security': {
                'route_table_associations': 'Restrict access between networks',
                'traffic_filtering': 'Use security groups and NACLs'
            }
        }
        
        # Connectivity Requirements
        architecture['connectivity_requirements'] = {
            'service_link': {
                'bandwidth': requirements.get('service_link_bandwidth', '10 Gbps'),
                'connectivity_type': requirements.get('connectivity_type', 'Direct Connect'),
                'redundancy': requirements.get('redundancy_required', True),
                'encryption': 'Always encrypted'
            },
            'internet_access': {
                'outbound_internet': requirements.get('internet_access', True),
                'nat_gateway_location': 'Regional VPC',
                'proxy_requirements': requirements.get('proxy_required', False)
            }
        }
        
        # Implementation Plan
        architecture['implementation_plan'] = [
            {
                'phase': 1,
                'description': 'Regional VPC and connectivity setup',
                'tasks': [
                    'Create regional VPC',
                    'Configure Direct Connect or VPN',
                    'Set up internet gateway and NAT gateway',
                    'Create regional subnets'
                ]
            },
            {
                'phase': 2,
                'description': 'Outpost installation and network configuration',
                'tasks': [
                    'Install Outpost hardware',
                    'Configure service link connectivity',
                    'Create Outpost subnets',
                    'Configure local gateway routing'
                ]
            },
            {
                'phase': 3,
                'description': 'Security and monitoring setup',
                'tasks': [
                    'Configure security groups and NACLs',
                    'Set up VPC Flow Logs',
                    'Configure CloudWatch monitoring',
                    'Implement backup and disaster recovery'
                ]
            },
            {
                'phase': 4,
                'description': 'Application deployment and testing',
                'tasks': [
                    'Deploy test workloads',
                    'Validate connectivity patterns',
                    'Performance testing',
                    'Production deployment'
                ]
            }
        ]
        
        return architecture
    
    def _calculate_outpost_subnet_cidrs(self, vpc_cidr: str, subnet_count: int) -> List[str]:
        """Calculate CIDR blocks for Outpost subnets"""
        
        # Simple calculation - in production, use proper CIDR calculation
        base_cidr = vpc_cidr.split('/')[0]
        base_octets = base_cidr.split('.')
        
        subnets = []
        for i in range(subnet_count):
            subnet_cidr = f"{base_octets[0]}.{base_octets[1]}.{100 + i}.0/24"
            subnets.append(subnet_cidr)
        
        return subnets
    
    def _design_security_groups(self, requirements: Dict) -> Dict:
        """Design security groups for Outpost deployment"""
        
        return {
            'web_tier_sg': {
                'description': 'Security group for web tier',
                'inbound_rules': [
                    {'protocol': 'tcp', 'port': 80, 'source': '0.0.0.0/0'},
                    {'protocol': 'tcp', 'port': 443, 'source': '0.0.0.0/0'}
                ],
                'outbound_rules': [
                    {'protocol': 'tcp', 'port': 3306, 'destination': 'db_tier_sg'}
                ]
            },
            'app_tier_sg': {
                'description': 'Security group for application tier',
                'inbound_rules': [
                    {'protocol': 'tcp', 'port': 8080, 'source': 'web_tier_sg'},
                    {'protocol': 'tcp', 'port': 22, 'source': requirements.get('management_cidr', '10.0.0.0/8')}
                ],
                'outbound_rules': [
                    {'protocol': 'tcp', 'port': 3306, 'destination': 'db_tier_sg'},
                    {'protocol': 'tcp', 'port': 443, 'destination': '0.0.0.0/0'}
                ]
            },
            'db_tier_sg': {
                'description': 'Security group for database tier',
                'inbound_rules': [
                    {'protocol': 'tcp', 'port': 3306, 'source': 'app_tier_sg'},
                    {'protocol': 'tcp', 'port': 5432, 'source': 'app_tier_sg'}
                ],
                'outbound_rules': []
            },
            'onpremises_access_sg': {
                'description': 'Security group for on-premises connectivity',
                'inbound_rules': [
                    {'protocol': 'tcp', 'port': 22, 'source': requirements.get('onpremises_cidr', '192.168.0.0/16')},
                    {'protocol': 'tcp', 'port': 3389, 'source': requirements.get('onpremises_cidr', '192.168.0.0/16')}
                ],
                'outbound_rules': [
                    {'protocol': 'all', 'destination': requirements.get('onpremises_cidr', '192.168.0.0/16')}
                ]
            }
        }
    
    def _design_network_acls(self, requirements: Dict) -> Dict:
        """Design Network ACLs for additional security"""
        
        return {
            'outpost_subnet_nacl': {
                'description': 'Network ACL for Outpost subnets',
                'inbound_rules': [
                    {'rule_number': 100, 'protocol': 'tcp', 'port_range': '80-80', 'source': '0.0.0.0/0', 'action': 'allow'},
                    {'rule_number': 110, 'protocol': 'tcp', 'port_range': '443-443', 'source': '0.0.0.0/0', 'action': 'allow'},
                    {'rule_number': 120, 'protocol': 'tcp', 'port_range': '22-22', 'source': '10.0.0.0/8', 'action': 'allow'},
                    {'rule_number': 130, 'protocol': 'tcp', 'port_range': '1024-65535', 'source': '0.0.0.0/0', 'action': 'allow'}
                ],
                'outbound_rules': [
                    {'rule_number': 100, 'protocol': 'tcp', 'port_range': '80-80', 'destination': '0.0.0.0/0', 'action': 'allow'},
                    {'rule_number': 110, 'protocol': 'tcp', 'port_range': '443-443', 'destination': '0.0.0.0/0', 'action': 'allow'},
                    {'rule_number': 120, 'protocol': 'tcp', 'port_range': '1024-65535', 'destination': '0.0.0.0/0', 'action': 'allow'}
                ]
            }
        }

# Usage example
outposts_manager = OutpostsNetworkingManager()

# Discover existing Outposts
outposts_info = outposts_manager.discover_outposts()
print("Outposts Discovery:")
print(json.dumps(outposts_info, indent=2))

# Design network architecture
requirements = {
    'regional_vpc_cidr': '10.0.0.0/16',
    'outpost_subnet_count': 3,
    'outpost_availability_zone': 'us-east-1-outpost-1a',
    'onpremises_cidr': '192.168.0.0/16',
    'service_link_bandwidth': '10 Gbps',
    'connectivity_type': 'Direct Connect',
    'redundancy_required': True,
    'internet_access': True,
    'enable_cross_vpc': True,
    'vpc_peering_required': False,
    'management_cidr': '10.0.0.0/16',
    'proxy_required': False
}

network_architecture = outposts_manager.design_outposts_network_architecture(requirements)
print("\nNetwork Architecture Design:")
print(json.dumps(network_architecture, indent=2))
```

## Local Gateway Configuration {#local-gateway}

### Local Gateway Setup and Routing

```bash
#!/bin/bash
# Local Gateway configuration script for AWS Outposts

OUTPOST_ID="op-1234567890abcdef0"
VPC_ID="vpc-12345678"
ONPREMISES_CIDR="192.168.0.0/16"
OUTPOST_SUBNET_CIDR="10.0.100.0/24"

echo "Configuring Local Gateway for Outpost: $OUTPOST_ID"

# Get Local Gateway information
echo "Discovering Local Gateway..."
LOCAL_GATEWAY_ID=$(aws ec2 describe-local-gateways \
    --filters "Name=outpost-arn,Values=*$OUTPOST_ID*" \
    --query 'LocalGateways[0].LocalGatewayId' \
    --output text)

if [ "$LOCAL_GATEWAY_ID" = "None" ] || [ -z "$LOCAL_GATEWAY_ID" ]; then
    echo "Error: No Local Gateway found for Outpost $OUTPOST_ID"
    exit 1
fi

echo "Found Local Gateway: $LOCAL_GATEWAY_ID"

# Get Local Gateway Route Table
LGW_ROUTE_TABLE_ID=$(aws ec2 describe-local-gateway-route-tables \
    --filters "Name=local-gateway-id,Values=$LOCAL_GATEWAY_ID" \
    --query 'LocalGatewayRouteTables[0].LocalGatewayRouteTableId' \
    --output text)

echo "Local Gateway Route Table: $LGW_ROUTE_TABLE_ID"

# Create Outpost subnet if it doesn't exist
echo "Creating Outpost subnet..."
OUTPOST_SUBNET_ID=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block $OUTPOST_SUBNET_CIDR \
    --outpost-arn $(aws outposts list-outposts --query "Outposts[?OutpostId=='$OUTPOST_ID'].OutpostArn" --output text) \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Outpost-Subnet},{Key=Purpose,Value=Hybrid}]' \
    --query 'Subnet.SubnetId' \
    --output text 2>/dev/null)

if [ $? -eq 0 ]; then
    echo "Created Outpost subnet: $OUTPOST_SUBNET_ID"
else
    # Subnet might already exist, try to find it
    OUTPOST_SUBNET_ID=$(aws ec2 describe-subnets \
        --filters "Name=vpc-id,Values=$VPC_ID" "Name=cidr-block,Values=$OUTPOST_SUBNET_CIDR" \
        --query 'Subnets[0].SubnetId' \
        --output text)
    echo "Using existing Outpost subnet: $OUTPOST_SUBNET_ID"
fi

# Create VPC route table for Outpost subnets
echo "Creating VPC route table for Outpost..."
OUTPOST_ROUTE_TABLE_ID=$(aws ec2 create-route-table \
    --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Outpost-RouteTable},{Key=Purpose,Value=LocalGateway}]' \
    --query 'RouteTable.RouteTableId' \
    --output text)

echo "Created VPC route table: $OUTPOST_ROUTE_TABLE_ID"

# Associate subnet with route table
echo "Associating subnet with route table..."
aws ec2 associate-route-table \
    --subnet-id $OUTPOST_SUBNET_ID \
    --route-table-id $OUTPOST_ROUTE_TABLE_ID

# Create route to on-premises network via Local Gateway
echo "Creating route to on-premises network..."
aws ec2 create-route \
    --route-table-id $OUTPOST_ROUTE_TABLE_ID \
    --destination-cidr-block $ONPREMISES_CIDR \
    --local-gateway-id $LOCAL_GATEWAY_ID

# Create Local Gateway Route Table VPC Association
echo "Creating Local Gateway Route Table VPC Association..."
LGW_VPC_ASSOCIATION_ID=$(aws ec2 create-local-gateway-route-table-vpc-association \
    --local-gateway-route-table-id $LGW_ROUTE_TABLE_ID \
    --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=local-gateway-route-table-vpc-association,Tags=[{Key=Name,Value=Outpost-LGW-VPC-Association}]' \
    --query 'LocalGatewayRouteTableVpcAssociation.LocalGatewayRouteTableVpcAssociationId' \
    --output text)

echo "Created LGW-VPC Association: $LGW_VPC_ASSOCIATION_ID"

# Function to create Local Gateway routes
create_lgw_route() {
    local destination_cidr=$1
    local target_vpc_id=$2
    local description=$3
    
    echo "Creating Local Gateway route: $destination_cidr -> $target_vpc_id ($description)"
    
    aws ec2 create-local-gateway-route \
        --destination-cidr-block $destination_cidr \
        --local-gateway-route-table-id $LGW_ROUTE_TABLE_ID \
        --local-gateway-virtual-interface-group-id $(aws ec2 describe-local-gateway-virtual-interface-groups \
            --filters "Name=local-gateway-id,Values=$LOCAL_GATEWAY_ID" \
            --query 'LocalGatewayVirtualInterfaceGroups[0].LocalGatewayVirtualInterfaceGroupId' \
            --output text) \
        || echo "Route may already exist or require different configuration"
}

# Create routes from Local Gateway to VPC subnets
echo "Creating Local Gateway routes to VPC subnets..."
create_lgw_route $OUTPOST_SUBNET_CIDR $VPC_ID "Outpost subnet access"

# Configure security group for Local Gateway access
echo "Creating security group for Local Gateway connectivity..."
LGW_SECURITY_GROUP_ID=$(aws ec2 create-security-group \
    --group-name OutpostLocalGatewayAccess \
    --description "Security group for Local Gateway connectivity" \
    --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=Outpost-LocalGateway-SG}]' \
    --query 'GroupId' \
    --output text)

echo "Created security group: $LGW_SECURITY_GROUP_ID"

# Configure security group rules
echo "Configuring security group rules..."

# Allow SSH from on-premises
aws ec2 authorize-security-group-ingress \
    --group-id $LGW_SECURITY_GROUP_ID \
    --protocol tcp \
    --port 22 \
    --cidr $ONPREMISES_CIDR

# Allow HTTP/HTTPS from on-premises
aws ec2 authorize-security-group-ingress \
    --group-id $LGW_SECURITY_GROUP_ID \
    --protocol tcp \
    --port 80 \
    --cidr $ONPREMISES_CIDR

aws ec2 authorize-security-group-ingress \
    --group-id $LGW_SECURITY_GROUP_ID \
    --protocol tcp \
    --port 443 \
    --cidr $ONPREMISES_CIDR

# Allow all outbound to on-premises
aws ec2 authorize-security-group-egress \
    --group-id $LGW_SECURITY_GROUP_ID \
    --protocol all \
    --cidr $ONPREMISES_CIDR

echo "Local Gateway configuration completed successfully!"
echo ""
echo "Configuration Summary:"
echo "  Local Gateway ID: $LOCAL_GATEWAY_ID"
echo "  Local Gateway Route Table: $LGW_ROUTE_TABLE_ID"
echo "  Outpost Subnet ID: $OUTPOST_SUBNET_ID"
echo "  VPC Route Table ID: $OUTPOST_ROUTE_TABLE_ID"
echo "  LGW-VPC Association ID: $LGW_VPC_ASSOCIATION_ID"
echo "  Security Group ID: $LGW_SECURITY_GROUP_ID"
echo ""
echo "Next steps:"
echo "  1. Configure on-premises routing to point $OUTPOST_SUBNET_CIDR to the Local Gateway"
echo "  2. Test connectivity between Outpost instances and on-premises resources"
echo "  3. Deploy applications and validate hybrid connectivity"

# Create verification script
cat > verify_lgw_connectivity.sh << 'EOF'
#!/bin/bash
# Connectivity verification script

echo "Verifying Local Gateway connectivity..."

# Test 1: Check route table configuration
echo "1. Checking VPC route table..."
aws ec2 describe-route-tables --route-table-ids $OUTPOST_ROUTE_TABLE_ID \
    --query 'RouteTables[0].Routes' --output table

echo "2. Checking Local Gateway route table..."
aws ec2 describe-local-gateway-route-tables --local-gateway-route-table-ids $LGW_ROUTE_TABLE_ID \
    --query 'LocalGatewayRouteTables[0]' --output json

# Test 3: Check VPC association
echo "3. Checking Local Gateway VPC association..."
aws ec2 describe-local-gateway-route-table-vpc-associations \
    --local-gateway-route-table-vpc-association-ids $LGW_VPC_ASSOCIATION_ID \
    --output table

echo "4. Manual connectivity test required:"
echo "   - Launch an EC2 instance in subnet $OUTPOST_SUBNET_ID"
echo "   - Try to ping/connect to an on-premises resource"
echo "   - Verify bidirectional connectivity"
EOF

chmod +x verify_lgw_connectivity.sh
echo "Created verification script: verify_lgw_connectivity.sh"
```

### Advanced Local Gateway Routing

```python
import boto3
import json
from typing import Dict, List, Optional

class LocalGatewayRoutingManager:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
        
    def configure_advanced_lgw_routing(self, 
                                     lgw_config: Dict) -> Dict:
        """Configure advanced Local Gateway routing patterns"""
        
        results = {
            'route_tables_created': [],
            'vpc_associations': [],
            'routes_created': [],
            'configuration_status': 'success'
        }
        
        try:
            local_gateway_id = lgw_config['local_gateway_id']
            
            # Create multiple route tables for different traffic types
            route_tables = self._create_specialized_route_tables(
                local_gateway_id,
                lgw_config['routing_domains']
            )
            results['route_tables_created'] = route_tables
            
            # Configure VPC associations
            vpc_associations = self._configure_vpc_associations(
                route_tables,
                lgw_config['vpc_configurations']
            )
            results['vpc_associations'] = vpc_associations
            
            # Create granular routes
            routes = self._create_granular_routes(
                route_tables,
                lgw_config['routing_rules']
            )
            results['routes_created'] = routes
            
            return results
            
        except Exception as e:
            results['configuration_status'] = 'failed'
            results['error'] = str(e)
            return results
    
    def _create_specialized_route_tables(self, 
                                       local_gateway_id: str,
                                       routing_domains: List[Dict]) -> List[Dict]:
        """Create specialized route tables for different routing domains"""
        
        route_tables = []
        
        for domain in routing_domains:
            try:
                # Create Local Gateway Route Table
                response = self.ec2.create_local_gateway_route_table(
                    LocalGatewayId=local_gateway_id,
                    Mode=domain.get('mode', 'direct-vpc-routing'),
                    TagSpecifications=[{
                        'ResourceType': 'local-gateway-route-table',
                        'Tags': [
                            {'Key': 'Name', 'Value': domain['name']},
                            {'Key': 'Purpose', 'Value': domain['purpose']},
                            {'Key': 'Domain', 'Value': domain['domain']}
                        ]
                    }]
                )
                
                route_table_info = {
                    'route_table_id': response['LocalGatewayRouteTable']['LocalGatewayRouteTableId'],
                    'domain': domain['domain'],
                    'purpose': domain['purpose'],
                    'name': domain['name']
                }
                
                route_tables.append(route_table_info)
                print(f"Created route table {route_table_info['route_table_id']} for {domain['purpose']}")
                
            except Exception as e:
                print(f"Failed to create route table for {domain['name']}: {e}")
        
        return route_tables
    
    def _configure_vpc_associations(self, 
                                  route_tables: List[Dict],
                                  vpc_configurations: List[Dict]) -> List[Dict]:
        """Configure VPC associations with appropriate route tables"""
        
        associations = []
        
        for vpc_config in vpc_configurations:
            # Find appropriate route table for this VPC
            route_table = next(
                (rt for rt in route_tables if rt['domain'] == vpc_config['routing_domain']),
                None
            )
            
            if not route_table:
                print(f"No route table found for domain {vpc_config['routing_domain']}")
                continue
            
            try:
                response = self.ec2.create_local_gateway_route_table_vpc_association(
                    LocalGatewayRouteTableId=route_table['route_table_id'],
                    VpcId=vpc_config['vpc_id'],
                    TagSpecifications=[{
                        'ResourceType': 'local-gateway-route-table-vpc-association',
                        'Tags': [
                            {'Key': 'Name', 'Value': f"LGW-{vpc_config['vpc_name']}-Association"},
                            {'Key': 'VPCName', 'Value': vpc_config['vpc_name']},
                            {'Key': 'Domain', 'Value': vpc_config['routing_domain']}
                        ]
                    }]
                )
                
                association_info = {
                    'association_id': response['LocalGatewayRouteTableVpcAssociation']['LocalGatewayRouteTableVpcAssociationId'],
                    'vpc_id': vpc_config['vpc_id'],
                    'route_table_id': route_table['route_table_id'],
                    'domain': vpc_config['routing_domain']
                }
                
                associations.append(association_info)
                print(f"Created VPC association {association_info['association_id']}")
                
            except Exception as e:
                print(f"Failed to create VPC association for {vpc_config['vpc_name']}: {e}")
        
        return associations
    
    def _create_granular_routes(self, 
                              route_tables: List[Dict],
                              routing_rules: List[Dict]) -> List[Dict]:
        """Create granular routes based on routing rules"""
        
        routes = []
        
        for rule in routing_rules:
            # Find target route table
            route_table = next(
                (rt for rt in route_tables if rt['domain'] == rule['domain']),
                None
            )
            
            if not route_table:
                print(f"No route table found for domain {rule['domain']}")
                continue
            
            try:
                # Get Local Gateway Virtual Interface Group
                vif_response = self.ec2.describe_local_gateway_virtual_interface_groups(
                    Filters=[
                        {'Name': 'local-gateway-id', 'Values': [rule['local_gateway_id']]}
                    ]
                )
                
                if not vif_response['LocalGatewayVirtualInterfaceGroups']:
                    print(f"No virtual interface group found for Local Gateway")
                    continue
                
                vif_group_id = vif_response['LocalGatewayVirtualInterfaceGroups'][0]['LocalGatewayVirtualInterfaceGroupId']
                
                # Create route
                self.ec2.create_local_gateway_route(
                    DestinationCidrBlock=rule['destination_cidr'],
                    LocalGatewayRouteTableId=route_table['route_table_id'],
                    LocalGatewayVirtualInterfaceGroupId=vif_group_id
                )
                
                route_info = {
                    'destination_cidr': rule['destination_cidr'],
                    'route_table_id': route_table['route_table_id'],
                    'domain': rule['domain'],
                    'purpose': rule.get('purpose', 'Custom route')
                }
                
                routes.append(route_info)
                print(f"Created route {rule['destination_cidr']} in {route_table['route_table_id']}")
                
            except Exception as e:
                print(f"Failed to create route {rule['destination_cidr']}: {e}")
        
        return routes
    
    def implement_traffic_segmentation(self, 
                                     segmentation_config: Dict) -> Dict:
        """Implement traffic segmentation using multiple route tables"""
        
        segmentation_results = {
            'production_routing': {},
            'development_routing': {},
            'management_routing': {},
            'security_policies': {}
        }
        
        # Production traffic segmentation
        segmentation_results['production_routing'] = {
            'route_table_purpose': 'Production workloads',
            'allowed_destinations': segmentation_config.get('production_cidrs', []),
            'security_requirements': 'High - restricted access',
            'monitoring': 'Enhanced monitoring enabled'
        }
        
        # Development traffic segmentation
        segmentation_results['development_routing'] = {
            'route_table_purpose': 'Development and testing',
            'allowed_destinations': segmentation_config.get('development_cidrs', []),
            'security_requirements': 'Medium - controlled access',
            'monitoring': 'Standard monitoring'
        }
        
        # Management traffic segmentation
        segmentation_results['management_routing'] = {
            'route_table_purpose': 'Management and monitoring',
            'allowed_destinations': segmentation_config.get('management_cidrs', []),
            'security_requirements': 'High - administrative access only',
            'monitoring': 'Full audit logging'
        }
        
        # Security policies
        segmentation_results['security_policies'] = {
            'cross_segment_communication': 'Blocked by default',
            'internet_access': 'Via regional NAT gateway only',
            'on_premises_access': 'Segmented by route table',
            'compliance': segmentation_config.get('compliance_requirements', [])
        }
        
        return segmentation_results

# Usage example
lgw_routing_manager = LocalGatewayRoutingManager()

# Advanced Local Gateway routing configuration
lgw_config = {
    'local_gateway_id': 'lgw-12345678',
    'routing_domains': [
        {
            'name': 'Production-Routing',
            'purpose': 'Production workload routing',
            'domain': 'production',
            'mode': 'direct-vpc-routing'
        },
        {
            'name': 'Development-Routing', 
            'purpose': 'Development environment routing',
            'domain': 'development',
            'mode': 'direct-vpc-routing'
        },
        {
            'name': 'Management-Routing',
            'purpose': 'Management and monitoring',
            'domain': 'management',
            'mode': 'direct-vpc-routing'
        }
    ],
    'vpc_configurations': [
        {
            'vpc_id': 'vpc-prod123',
            'vpc_name': 'Production-VPC',
            'routing_domain': 'production'
        },
        {
            'vpc_id': 'vpc-dev456',
            'vpc_name': 'Development-VPC', 
            'routing_domain': 'development'
        },
        {
            'vpc_id': 'vpc-mgmt789',
            'vpc_name': 'Management-VPC',
            'routing_domain': 'management'
        }
    ],
    'routing_rules': [
        {
            'destination_cidr': '192.168.10.0/24',
            'domain': 'production',
            'purpose': 'Production database servers',
            'local_gateway_id': 'lgw-12345678'
        },
        {
            'destination_cidr': '192.168.20.0/24',
            'domain': 'development',
            'purpose': 'Development resources',
            'local_gateway_id': 'lgw-12345678'
        },
        {
            'destination_cidr': '192.168.1.0/24',
            'domain': 'management', 
            'purpose': 'Management network',
            'local_gateway_id': 'lgw-12345678'
        }
    ]
}

# Configure advanced routing
routing_results = lgw_routing_manager.configure_advanced_lgw_routing(lgw_config)
print("Advanced Local Gateway Routing Configuration:")
print(json.dumps(routing_results, indent=2))

# Implement traffic segmentation
segmentation_config = {
    'production_cidrs': ['192.168.10.0/24', '192.168.11.0/24'],
    'development_cidrs': ['192.168.20.0/24', '192.168.21.0/24'],
    'management_cidrs': ['192.168.1.0/24', '192.168.2.0/24'],
    'compliance_requirements': ['SOX', 'PCI-DSS', 'HIPAA']
}

segmentation_results = lgw_routing_manager.implement_traffic_segmentation(segmentation_config)
print("\nTraffic Segmentation Configuration:")
print(json.dumps(segmentation_results, indent=2))
```

This comprehensive guide provides detailed implementations for AWS Outposts networking, including Local Gateway configuration, advanced routing patterns, and traffic segmentation strategies. The examples can be adapted to specific organizational requirements while maintaining best practices for hybrid cloud networking.