# Topic 59: Inter-Region VPC Peering - Setup, Routing, and Optimization

## Table of Contents
1. [Inter-Region VPC Peering Overview](#overview)
2. [Architecture and Design Considerations](#architecture)
3. [Setup and Configuration](#setup)
4. [Routing Configuration](#routing)
5. [Security Considerations](#security)
6. [Data Transfer Costs](#costs)
7. [Performance Optimization](#performance)
8. [Monitoring and Troubleshooting](#monitoring)
9. [Advanced Patterns](#advanced-patterns)
10. [Automation and Infrastructure as Code](#automation)
11. [Best Practices](#best-practices)
12. [Migration and Maintenance](#migration)

## Inter-Region VPC Peering Overview {#overview}

### What is Inter-Region VPC Peering?

Inter-region VPC peering enables private communication between VPCs in different AWS regions using AWS's global backbone network. This networking connection allows resources in separate regions to communicate as if they were within the same network, while maintaining the security and isolation benefits of separate VPCs.

**Key Characteristics**:
- Private communication over AWS backbone network
- No internet gateway or VPN connection required
- Encrypted in transit automatically
- No single point of failure
- No bandwidth bottlenecks
- Supports IPv4 and IPv6 traffic

### Use Cases for Inter-Region VPC Peering

```
Common Use Cases:

1. Disaster Recovery
   ┌─────────────────┐    VPC Peering    ┌─────────────────┐
   │ Primary Region  │◄─────────────────►│ Backup Region   │
   │   (us-east-1)   │                   │   (us-west-2)   │
   │                 │                   │                 │
   │ ┌─────────────┐ │                   │ ┌─────────────┐ │
   │ │ Production  │ │   Data Replication│ │ Warm Standby│ │
   │ │ Workloads   │ │◄─────────────────►│ │ Environment │ │
   │ └─────────────┘ │                   │ └─────────────┘ │
   └─────────────────┘                   └─────────────────┘

2. Multi-Region Application Architecture
   ┌─────────────────┐                   ┌─────────────────┐
   │ Web Tier Region │                   │ Data Tier Region│
   │   (us-east-1)   │                   │   (us-west-2)   │
   │                 │                   │                 │
   │ ┌─────────────┐ │    API Calls     │ ┌─────────────┐ │
   │ │   ALB       │ │◄─────────────────►│ │  Database   │ │
   │ │   Web Apps  │ │                   │ │  Services   │ │
   │ └─────────────┘ │                   │ └─────────────┘ │
   └─────────────────┘                   └─────────────────┘

3. Global Content Distribution
   ┌─────────────────┐                   ┌─────────────────┐
   │ Content Origin  │                   │ Processing Hub  │
   │   (us-east-1)   │                   │   (eu-west-1)   │
   │                 │                   │                 │
   │ ┌─────────────┐ │   Content Sync   │ ┌─────────────┐ │
   │ │ Media Files │ │◄─────────────────►│ │ Transcoding │ │
   │ │ Storage     │ │                   │ │ Services    │ │
   │ └─────────────┘ │                   │ └─────────────┘ │
   └─────────────────┘                   └─────────────────┘

4. Regulatory Compliance
   ┌─────────────────┐                   ┌─────────────────┐
   │ EU Operations   │                   │ US Operations   │
   │   (eu-west-1)   │                   │   (us-east-1)   │
   │                 │                   │                 │
   │ ┌─────────────┐ │  Limited Access  │ ┌─────────────┐ │
   │ │ EU Customer │ │◄─────────────────►│ │ Shared      │ │
   │ │ Data        │ │                   │ │ Services    │ │
   │ └─────────────┘ │                   │ └─────────────┘ │
   └─────────────────┘                   └─────────────────┘
```

### Benefits and Limitations

**Benefits**:
```yaml
Benefits:
  Performance:
    - Low latency communication via AWS backbone
    - High bandwidth availability
    - No internet routing delays
    - Consistent network performance
    
  Security:
    - Traffic never traverses public internet
    - Automatic encryption in transit
    - Private IP communication
    - Network isolation maintained
    
  Reliability:
    - No single point of failure
    - AWS managed infrastructure
    - Multiple availability zones support
    - Built-in redundancy
    
  Cost_Efficiency:
    - Lower data transfer costs than internet routing
    - No VPN gateway charges
    - Predictable pricing model
    - No bandwidth limitations
```

**Limitations**:
```yaml
Limitations:
  Network_Design:
    - No transitive routing (A→B→C not possible)
    - CIDR blocks cannot overlap
    - DNS resolution limitations
    - Maximum 125 peering connections per VPC
    
  Regional_Constraints:
    - Both regions must support VPC peering
    - Different pricing in different regions
    - Latency depends on physical distance
    - Regulatory considerations for data movement
    
  Operational_Complexity:
    - Route table management required
    - Security group rule complexity
    - Monitoring across regions needed
    - Troubleshooting can be complex
```

## Architecture and Design Considerations {#architecture}

### Network Topology Design

**Hub and Spoke with Inter-Region Connectivity**:

```
Hub and Spoke Inter-Region Architecture:

Region A (us-east-1) - Hub
┌─────────────────────────────────────────────────────────────────┐
│                          Hub VPC                                │
│                       10.0.0.0/16                              │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │   Shared    │  │   Security  │  │  Management │            │
│  │  Services   │  │  Services   │  │   Services  │            │
│  │   Subnet    │  │   Subnet    │  │   Subnet    │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
└─────────────────────────┬───────────────────────────────────────┘
                          │
         ┌────────────────┼────────────────┐
         │                │                │
         ▼                ▼                ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│ Spoke VPC A     │ │ Spoke VPC B     │ │ Region B Hub    │
│  10.1.0.0/16    │ │  10.2.0.0/16    │ │  10.10.0.0/16   │
│  (us-east-1)    │ │  (us-east-1)    │ │  (eu-west-1)    │
│                 │ │                 │ │                 │
│ ┌─────────────┐ │ │ ┌─────────────┐ │ │ ┌─────────────┐ │
│ │ Application │ │ │ │ Development │ │ │ │ EU Services │ │
│ │ Workloads   │ │ │ │ Environment │ │ │ │ & Workloads │ │
│ └─────────────┘ │ │ └─────────────┘ │ │ └─────────────┘ │
└─────────────────┘ └─────────────────┘ └─────────┬───────┘
                                                  │
                                     ┌────────────┼────────────┐
                                     │            │            │
                                     ▼            ▼            ▼
                              ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
                              │ EU Spoke A  │ │ EU Spoke B  │ │ EU Spoke C  │
                              │10.11.0.0/16 │ │10.12.0.0/16 │ │10.13.0.0/16 │
                              │(eu-west-1)  │ │(eu-west-1)  │ │(eu-west-1)  │
                              └─────────────┘ └─────────────┘ └─────────────┘
```

### CIDR Block Planning

Proper CIDR block planning is crucial for inter-region VPC peering to avoid overlapping address spaces.

```bash
#!/bin/bash
# CIDR block planning script for multi-region deployment

declare -A REGIONS
REGIONS[us-east-1]="10.0.0.0/8"
REGIONS[us-west-2]="10.1.0.0/8"
REGIONS[eu-west-1]="10.2.0.0/8"
REGIONS[ap-south-1]="10.3.0.0/8"

echo "Regional CIDR Block Allocation:"
echo "================================"

for region in "${!REGIONS[@]}"; do
    cidr=${REGIONS[$region]}
    echo "Region: $region"
    echo "  Primary CIDR: $cidr"
    
    # Calculate subnet allocations
    base_octet=$(echo $cidr | cut -d'.' -f2)
    
    echo "  Subnet Allocations:"
    echo "    Public Subnets:  10.$base_octet.0.0/19   (10.$base_octet.0.0   - 10.$base_octet.31.255)"
    echo "    Private Subnets: 10.$base_octet.32.0/19  (10.$base_octet.32.0  - 10.$base_octet.63.255)"
    echo "    Database Subnets: 10.$base_octet.64.0/19 (10.$base_octet.64.0  - 10.$base_octet.95.255)"
    echo "    Reserved:        10.$base_octet.96.0/19  (10.$base_octet.96.0  - 10.$base_octet.127.255)"
    echo ""
done

# Validate no CIDR overlaps
echo "CIDR Overlap Validation:"
echo "========================"

python3 << EOF
import ipaddress

cidrs = {
    'us-east-1': '10.0.0.0/8',
    'us-west-2': '10.1.0.0/8', 
    'eu-west-1': '10.2.0.0/8',
    'ap-south-1': '10.3.0.0/8'
}

regions = list(cidrs.keys())
overlaps_found = False

for i, region1 in enumerate(regions):
    for region2 in regions[i+1:]:
        network1 = ipaddress.IPv4Network(cidrs[region1])
        network2 = ipaddress.IPv4Network(cidrs[region2])
        
        if network1.overlaps(network2):
            print(f"❌ OVERLAP: {region1} ({cidrs[region1]}) overlaps with {region2} ({cidrs[region2]})")
            overlaps_found = True

if not overlaps_found:
    print("✅ No CIDR block overlaps detected")
    print("✅ All regions can be safely peered")
EOF
```

### DNS Resolution Strategy

```yaml
# DNS resolution configuration for inter-region VPC peering
DNSResolutionStrategy:
  
  Option1_EnableDNSResolution:
    Description: "Enable DNS resolution for VPC peering connection"
    Configuration:
      AccepterVPCDNSResolution: true
      RequesterVPCDNSResolution: true
    
    Benefits:
      - Instances can resolve DNS names across peered VPCs
      - Private IP addresses returned for cross-region queries
      - Simplified application configuration
    
    Considerations:
      - DNS queries traverse the peering connection
      - May increase latency for DNS resolution
      - Requires proper security group configuration
  
  Option2_Route53PrivateHostedZones:
    Description: "Use Route 53 Private Hosted Zones for cross-region DNS"
    Configuration:
      SharedPrivateZone: internal.company.com
      RegionalSubzones:
        - us-east-1.internal.company.com
        - eu-west-1.internal.company.com
        - ap-south-1.internal.company.com
    
    Benefits:
      - Centralized DNS management
      - Better performance and caching
      - Advanced routing policies available
      - Health check integration
    
    Implementation:
      - Create private hosted zone in one region
      - Associate zone with VPCs in all regions
      - Create regional records for services
      - Use health checks for failover
  
  Option3_HybridApproach:
    Description: "Combine VPC DNS resolution with Route 53"
    Configuration:
      VPCDNSResolution: true
      PrivateHostedZones: true
      DNSForwarding: conditional
    
    Use_Cases:
      - Legacy applications requiring DNS resolution
      - New applications using Route 53 features
      - Gradual migration from VPC DNS to Route 53
```

## Setup and Configuration {#setup}

### Step-by-Step Peering Setup

```bash
#!/bin/bash
# Complete inter-region VPC peering setup script

# Configuration variables
REQUESTER_REGION="us-east-1"
ACCEPTER_REGION="eu-west-1"
REQUESTER_VPC_ID="vpc-12345678"
ACCEPTER_VPC_ID="vpc-87654321"
PEERING_NAME="us-east-1-to-eu-west-1"

echo "Setting up inter-region VPC peering between $REQUESTER_REGION and $ACCEPTER_REGION"

# Step 1: Create VPC peering connection
echo "Step 1: Creating VPC peering connection..."

PEERING_CONNECTION_ID=$(aws ec2 create-vpc-peering-connection \
  --vpc-id $REQUESTER_VPC_ID \
  --peer-vpc-id $ACCEPTER_VPC_ID \
  --peer-region $ACCEPTER_REGION \
  --region $REQUESTER_REGION \
  --tag-specifications "ResourceType=vpc-peering-connection,Tags=[{Key=Name,Value=$PEERING_NAME}]" \
  --query 'VpcPeeringConnection.VpcPeeringConnectionId' \
  --output text)

echo "Created peering connection: $PEERING_CONNECTION_ID"

# Step 2: Accept the peering connection in the accepter region
echo "Step 2: Accepting peering connection in $ACCEPTER_REGION..."

aws ec2 accept-vpc-peering-connection \
  --vpc-peering-connection-id $PEERING_CONNECTION_ID \
  --region $ACCEPTER_REGION

echo "Peering connection accepted"

# Step 3: Wait for peering connection to be active
echo "Step 3: Waiting for peering connection to become active..."

aws ec2 wait vpc-peering-connection-exists \
  --vpc-peering-connection-ids $PEERING_CONNECTION_ID \
  --region $REQUESTER_REGION

# Check status
STATUS=$(aws ec2 describe-vpc-peering-connections \
  --vpc-peering-connection-ids $PEERING_CONNECTION_ID \
  --region $REQUESTER_REGION \
  --query 'VpcPeeringConnections[0].Status.Code' \
  --output text)

echo "Peering connection status: $STATUS"

# Step 4: Enable DNS resolution (optional)
echo "Step 4: Enabling DNS resolution..."

aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id $PEERING_CONNECTION_ID \
  --requester-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
  --region $REQUESTER_REGION

aws ec2 modify-vpc-peering-connection-options \
  --vpc-peering-connection-id $PEERING_CONNECTION_ID \
  --accepter-peering-connection-options AllowDnsResolutionFromRemoteVpc=true \
  --region $ACCEPTER_REGION

echo "DNS resolution enabled for both VPCs"

# Step 5: Get VPC CIDR blocks for routing
echo "Step 5: Getting VPC CIDR blocks..."

REQUESTER_CIDR=$(aws ec2 describe-vpcs \
  --vpc-ids $REQUESTER_VPC_ID \
  --region $REQUESTER_REGION \
  --query 'Vpcs[0].CidrBlock' \
  --output text)

ACCEPTER_CIDR=$(aws ec2 describe-vpcs \
  --vpc-ids $ACCEPTER_VPC_ID \
  --region $ACCEPTER_REGION \
  --query 'Vpcs[0].CidrBlock' \
  --output text)

echo "Requester VPC CIDR: $REQUESTER_CIDR"
echo "Accepter VPC CIDR: $ACCEPTER_CIDR"

# Step 6: Update route tables
echo "Step 6: Updating route tables..."

# Get route table IDs for requester VPC
REQUESTER_ROUTE_TABLES=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$REQUESTER_VPC_ID" \
  --region $REQUESTER_REGION \
  --query 'RouteTables[].RouteTableId' \
  --output text)

# Get route table IDs for accepter VPC
ACCEPTER_ROUTE_TABLES=$(aws ec2 describe-route-tables \
  --filters "Name=vpc-id,Values=$ACCEPTER_VPC_ID" \
  --region $ACCEPTER_REGION \
  --query 'RouteTables[].RouteTableId' \
  --output text)

# Add routes in requester region
for RT_ID in $REQUESTER_ROUTE_TABLES; do
    echo "Adding route to $ACCEPTER_CIDR in route table $RT_ID"
    aws ec2 create-route \
      --route-table-id $RT_ID \
      --destination-cidr-block $ACCEPTER_CIDR \
      --vpc-peering-connection-id $PEERING_CONNECTION_ID \
      --region $REQUESTER_REGION || echo "Route may already exist"
done

# Add routes in accepter region
for RT_ID in $ACCEPTER_ROUTE_TABLES; do
    echo "Adding route to $REQUESTER_CIDR in route table $RT_ID"
    aws ec2 create-route \
      --route-table-id $RT_ID \
      --destination-cidr-block $REQUESTER_CIDR \
      --vpc-peering-connection-id $PEERING_CONNECTION_ID \
      --region $ACCEPTER_REGION || echo "Route may already exist"
done

echo "Inter-region VPC peering setup completed successfully!"
echo "Peering Connection ID: $PEERING_CONNECTION_ID"
echo "You may need to update security groups to allow cross-region traffic"
```

### CloudFormation Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Inter-region VPC peering setup with routing'

Parameters:
  RequesterVPCId:
    Type: String
    Description: 'VPC ID in the requester region'
    
  AccepterVPCId:
    Type: String
    Description: 'VPC ID in the accepter region'
    
  AccepterRegion:
    Type: String
    Description: 'Region where the accepter VPC exists'
    
  RequesterVPCCIDR:
    Type: String
    Description: 'CIDR block of the requester VPC'
    Default: '10.0.0.0/16'
    
  AccepterVPCCIDR:
    Type: String
    Description: 'CIDR block of the accepter VPC'
    Default: '10.1.0.0/16'

Resources:
  # VPC Peering Connection
  VPCPeeringConnection:
    Type: AWS::EC2::VPCPeeringConnection
    Properties:
      VpcId: !Ref RequesterVPCId
      PeerVpcId: !Ref AccepterVPCId
      PeerRegion: !Ref AccepterRegion
      Tags:
        - Key: Name
          Value: !Sub '${AWS::StackName}-peering-connection'
        - Key: Purpose
          Value: 'Inter-region connectivity'

  # Custom resource to accept peering connection
  AcceptPeeringConnection:
    Type: AWS::CloudFormation::CustomResource
    Properties:
      ServiceToken: !GetAtt AcceptPeeringFunction.Arn
      PeeringConnectionId: !Ref VPCPeeringConnection
      AccepterRegion: !Ref AccepterRegion

  # Lambda function to accept peering connection
  AcceptPeeringFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub '${AWS::StackName}-accept-peering'
      Runtime: python3.9
      Handler: index.handler
      Timeout: 60
      Role: !GetAtt AcceptPeeringRole.Arn
      Code:
        ZipFile: |
          import boto3
          import cfnresponse
          import logging
          
          logger = logging.getLogger()
          logger.setLevel(logging.INFO)
          
          def handler(event, context):
              try:
                  if event['RequestType'] == 'Create':
                      peering_id = event['ResourceProperties']['PeeringConnectionId']
                      region = event['ResourceProperties']['AccepterRegion']
                      
                      ec2 = boto3.client('ec2', region_name=region)
                      
                      # Accept the peering connection
                      response = ec2.accept_vpc_peering_connection(
                          VpcPeeringConnectionId=peering_id
                      )
                      
                      logger.info(f"Accepted peering connection {peering_id}")
                      
                      # Enable DNS resolution
                      ec2.modify_vpc_peering_connection_options(
                          VpcPeeringConnectionId=peering_id,
                          AccepterPeeringConnectionOptions={
                              'AllowDnsResolutionFromRemoteVpc': True
                          }
                      )
                      
                      cfnresponse.send(event, context, cfnresponse.SUCCESS, {})
                      
                  else:
                      cfnresponse.send(event, context, cfnresponse.SUCCESS, {})
                      
              except Exception as e:
                  logger.error(f"Error: {e}")
                  cfnresponse.send(event, context, cfnresponse.FAILED, {})

  # IAM role for Lambda function
  AcceptPeeringRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: lambda.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
      Policies:
        - PolicyName: VPCPeeringPolicy
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - ec2:AcceptVpcPeeringConnection
                  - ec2:DescribeVpcPeeringConnections
                  - ec2:ModifyVpcPeeringConnectionOptions
                Resource: '*'

  # Route table updates for requester VPC
  RequesterRoute:
    Type: AWS::EC2::Route
    DependsOn: AcceptPeeringConnection
    Properties:
      RouteTableId: !Ref RequesterRouteTableId
      DestinationCidrBlock: !Ref AccepterVPCCIDR
      VpcPeeringConnectionId: !Ref VPCPeeringConnection

Outputs:
  VPCPeeringConnectionId:
    Description: 'VPC Peering Connection ID'
    Value: !Ref VPCPeeringConnection
    Export:
      Name: !Sub '${AWS::StackName}-peering-connection-id'
      
  PeeringConnectionStatus:
    Description: 'Peering connection status'
    Value: !GetAtt VPCPeeringConnection.Status
```

### Terraform Configuration

```hcl
# Inter-region VPC peering with Terraform
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Configure providers for both regions
provider "aws" {
  alias  = "requester"
  region = var.requester_region
}

provider "aws" {
  alias  = "accepter"
  region = var.accepter_region
}

# Variables
variable "requester_region" {
  description = "Requester region"
  type        = string
  default     = "us-east-1"
}

variable "accepter_region" {
  description = "Accepter region"
  type        = string
  default     = "eu-west-1"
}

variable "requester_vpc_id" {
  description = "Requester VPC ID"
  type        = string
}

variable "accepter_vpc_id" {
  description = "Accepter VPC ID"
  type        = string
}

# Data sources to get VPC information
data "aws_vpc" "requester" {
  provider = aws.requester
  id       = var.requester_vpc_id
}

data "aws_vpc" "accepter" {
  provider = aws.accepter
  id       = var.accepter_vpc_id
}

# Create VPC peering connection
resource "aws_vpc_peering_connection" "main" {
  provider    = aws.requester
  vpc_id      = var.requester_vpc_id
  peer_vpc_id = var.accepter_vpc_id
  peer_region = var.accepter_region
  auto_accept = false

  tags = {
    Name = "inter-region-peering"
    Side = "Requester"
  }
}

# Accept the peering connection
resource "aws_vpc_peering_connection_accepter" "main" {
  provider                  = aws.accepter
  vpc_peering_connection_id = aws_vpc_peering_connection.main.id
  auto_accept               = true

  tags = {
    Name = "inter-region-peering"
    Side = "Accepter"
  }
}

# Enable DNS resolution for requester
resource "aws_vpc_peering_connection_options" "requester" {
  provider                = aws.requester
  vpc_peering_connection_id = aws_vpc_peering_connection.main.id

  requester {
    allow_remote_vpc_dns_resolution = true
  }

  depends_on = [aws_vpc_peering_connection_accepter.main]
}

# Enable DNS resolution for accepter
resource "aws_vpc_peering_connection_options" "accepter" {
  provider                = aws.accepter
  vpc_peering_connection_id = aws_vpc_peering_connection.main.id

  accepter {
    allow_remote_vpc_dns_resolution = true
  }

  depends_on = [aws_vpc_peering_connection_accepter.main]
}

# Get route tables for requester VPC
data "aws_route_tables" "requester" {
  provider = aws.requester
  vpc_id   = var.requester_vpc_id
}

# Get route tables for accepter VPC
data "aws_route_tables" "accepter" {
  provider = aws.accepter
  vpc_id   = var.accepter_vpc_id
}

# Create routes in requester VPC route tables
resource "aws_route" "requester_to_accepter" {
  provider               = aws.requester
  count                  = length(data.aws_route_tables.requester.ids)
  route_table_id         = data.aws_route_tables.requester.ids[count.index]
  destination_cidr_block = data.aws_vpc.accepter.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.main.id

  depends_on = [aws_vpc_peering_connection_accepter.main]
}

# Create routes in accepter VPC route tables
resource "aws_route" "accepter_to_requester" {
  provider               = aws.accepter
  count                  = length(data.aws_route_tables.accepter.ids)
  route_table_id         = data.aws_route_tables.accepter.ids[count.index]
  destination_cidr_block = data.aws_vpc.requester.cidr_block
  vpc_peering_connection_id = aws_vpc_peering_connection.main.id

  depends_on = [aws_vpc_peering_connection_accepter.main]
}

# Outputs
output "vpc_peering_connection_id" {
  description = "VPC peering connection ID"
  value       = aws_vpc_peering_connection.main.id
}

output "vpc_peering_connection_status" {
  description = "VPC peering connection status"
  value       = aws_vpc_peering_connection.main.accept_status
}

output "requester_vpc_cidr" {
  description = "Requester VPC CIDR block"
  value       = data.aws_vpc.requester.cidr_block
}

output "accepter_vpc_cidr" {
  description = "Accepter VPC CIDR block"
  value       = data.aws_vpc.accepter.cidr_block
}
```

## Routing Configuration {#routing}

### Advanced Routing Scenarios

```python
import boto3
import json
from typing import Dict, List, Tuple

class InterRegionRoutingManager:
    def __init__(self, regions: List[str]):
        self.regions = regions
        self.ec2_clients = {
            region: boto3.client('ec2', region_name=region)
            for region in regions
        }
        
    def analyze_routing_topology(self, vpc_mappings: Dict[str, str]) -> Dict:
        """Analyze current routing topology across regions"""
        
        topology = {
            'vpcs': {},
            'peering_connections': {},
            'routing_paths': {},
            'potential_issues': []
        }
        
        # Get VPC information
        for region, vpc_id in vpc_mappings.items():
            try:
                vpc_info = self.ec2_clients[region].describe_vpcs(
                    VpcIds=[vpc_id]
                )['Vpcs'][0]
                
                topology['vpcs'][region] = {
                    'vpc_id': vpc_id,
                    'cidr_block': vpc_info['CidrBlock'],
                    'state': vpc_info['State']
                }
                
                # Get route tables
                route_tables = self.ec2_clients[region].describe_route_tables(
                    Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}]
                )['RouteTables']
                
                topology['vpcs'][region]['route_tables'] = [
                    {
                        'route_table_id': rt['RouteTableId'],
                        'routes': rt['Routes']
                    }
                    for rt in route_tables
                ]
                
            except Exception as e:
                topology['potential_issues'].append(
                    f"Failed to analyze VPC {vpc_id} in {region}: {e}"
                )
        
        # Analyze peering connections
        for region in self.regions:
            try:
                peering_connections = self.ec2_clients[region].describe_vpc_peering_connections(
                    Filters=[
                        {'Name': 'status-code', 'Values': ['active']},
                        {'Name': 'requester-vpc-info.vpc-id', 'Values': list(vpc_mappings.values())}
                    ]
                )['VpcPeeringConnections']
                
                for pc in peering_connections:
                    pc_id = pc['VpcPeeringConnectionId']
                    topology['peering_connections'][pc_id] = {
                        'requester_vpc': pc['RequesterVpcInfo']['VpcId'],
                        'requester_region': pc['RequesterVpcInfo']['Region'],
                        'accepter_vpc': pc['AccepterVpcInfo']['VpcId'],
                        'accepter_region': pc['AccepterVpcInfo']['Region'],
                        'status': pc['Status']['Code']
                    }
                    
            except Exception as e:
                topology['potential_issues'].append(
                    f"Failed to analyze peering in {region}: {e}"
                )
        
        return topology
    
    def validate_routing_configuration(self, topology: Dict) -> List[Dict]:
        """Validate routing configuration and identify issues"""
        
        issues = []
        
        # Check for CIDR overlaps
        cidrs = {}
        for region, vpc_info in topology['vpcs'].items():
            cidr = vpc_info['cidr_block']
            if cidr in cidrs:
                issues.append({
                    'type': 'cidr_overlap',
                    'severity': 'high',
                    'description': f"CIDR {cidr} overlaps between {region} and {cidrs[cidr]}",
                    'regions': [region, cidrs[cidr]]
                })
            else:
                cidrs[cidr] = region
        
        # Check for missing routes
        for pc_id, pc_info in topology['peering_connections'].items():
            requester_region = pc_info['requester_region']
            accepter_region = pc_info['accepter_region']
            
            # Check if requester has route to accepter
            requester_routes = self._get_vpc_routes(
                topology, requester_region, pc_info['requester_vpc']
            )
            accepter_cidr = topology['vpcs'][accepter_region]['cidr_block']
            
            if not self._has_route_to_cidr(requester_routes, accepter_cidr, pc_id):
                issues.append({
                    'type': 'missing_route',
                    'severity': 'medium',
                    'description': f"Missing route from {requester_region} to {accepter_cidr}",
                    'peering_connection': pc_id
                })
        
        return issues
    
    def _get_vpc_routes(self, topology: Dict, region: str, vpc_id: str) -> List[Dict]:
        """Get all routes for a VPC"""
        routes = []
        vpc_info = topology['vpcs'].get(region, {})
        for rt in vpc_info.get('route_tables', []):
            routes.extend(rt['routes'])
        return routes
    
    def _has_route_to_cidr(self, routes: List[Dict], target_cidr: str, pc_id: str) -> bool:
        """Check if routes contain path to target CIDR via peering connection"""
        for route in routes:
            if (route.get('DestinationCidrBlock') == target_cidr and
                route.get('VpcPeeringConnectionId') == pc_id):
                return True
        return False
    
    def optimize_routing_paths(self, topology: Dict) -> Dict:
        """Suggest routing optimizations"""
        
        optimizations = {
            'recommendations': [],
            'estimated_savings': 0,
            'performance_improvements': []
        }
        
        # Analyze routing efficiency
        for region, vpc_info in topology['vpcs'].items():
            route_tables = vpc_info.get('route_tables', [])
            
            # Check for redundant route tables
            if len(route_tables) > 3:
                optimizations['recommendations'].append({
                    'type': 'route_table_consolidation',
                    'region': region,
                    'description': f"Consider consolidating {len(route_tables)} route tables",
                    'impact': 'Simplified management'
                })
            
            # Check for overly broad routes
            for rt in route_tables:
                for route in rt['routes']:
                    if route.get('DestinationCidrBlock') == '0.0.0.0/0':
                        optimizations['recommendations'].append({
                            'type': 'broad_route',
                            'region': region,
                            'route_table': rt['route_table_id'],
                            'description': 'Default route may cause unintended traffic flow',
                            'impact': 'Security and cost implications'
                        })
        
        return optimizations
    
    def create_routing_diagram(self, topology: Dict) -> str:
        """Create ASCII diagram of routing topology"""
        
        diagram = []
        diagram.append("Inter-Region Routing Topology")
        diagram.append("=" * 40)
        diagram.append("")
        
        # VPC information
        for region, vpc_info in topology['vpcs'].items():
            diagram.append(f"Region: {region}")
            diagram.append(f"  VPC: {vpc_info['vpc_id']} ({vpc_info['cidr_block']})")
            diagram.append(f"  Route Tables: {len(vpc_info.get('route_tables', []))}")
            diagram.append("")
        
        # Peering connections
        diagram.append("Peering Connections:")
        for pc_id, pc_info in topology['peering_connections'].items():
            diagram.append(f"  {pc_id}:")
            diagram.append(f"    {pc_info['requester_region']} ↔ {pc_info['accepter_region']}")
            diagram.append(f"    Status: {pc_info['status']}")
            diagram.append("")
        
        return "\n".join(diagram)

# Usage example
regions = ['us-east-1', 'eu-west-1', 'ap-south-1']
vpc_mappings = {
    'us-east-1': 'vpc-12345678',
    'eu-west-1': 'vpc-87654321',
    'ap-south-1': 'vpc-abcdef12'
}

routing_manager = InterRegionRoutingManager(regions)

# Analyze current topology
topology = routing_manager.analyze_routing_topology(vpc_mappings)
print("Topology Analysis:")
print(json.dumps(topology, indent=2, default=str))

# Validate configuration
issues = routing_manager.validate_routing_configuration(topology)
print("\nConfiguration Issues:")
for issue in issues:
    print(f"  {issue['severity'].upper()}: {issue['description']}")

# Get optimization recommendations
optimizations = routing_manager.optimize_routing_paths(topology)
print("\nOptimization Recommendations:")
for rec in optimizations['recommendations']:
    print(f"  {rec['type']}: {rec['description']}")

# Create routing diagram
diagram = routing_manager.create_routing_diagram(topology)
print("\nRouting Diagram:")
print(diagram)
```

### Route Table Management

```bash
#!/bin/bash
# Advanced route table management for inter-region peering

# Function to update route tables with error handling
update_route_table() {
    local region=$1
    local route_table_id=$2
    local destination_cidr=$3
    local peering_connection_id=$4
    
    echo "Updating route table $route_table_id in $region"
    
    # Check if route already exists
    existing_route=$(aws ec2 describe-route-tables \
        --route-table-ids $route_table_id \
        --region $region \
        --query "RouteTables[0].Routes[?DestinationCidrBlock=='$destination_cidr'].VpcPeeringConnectionId" \
        --output text)
    
    if [ "$existing_route" != "" ]; then
        echo "Route to $destination_cidr already exists in $route_table_id"
        
        # Check if it points to the correct peering connection
        if [ "$existing_route" != "$peering_connection_id" ]; then
            echo "Updating existing route to use correct peering connection"
            aws ec2 replace-route \
                --route-table-id $route_table_id \
                --destination-cidr-block $destination_cidr \
                --vpc-peering-connection-id $peering_connection_id \
                --region $region
        fi
    else
        echo "Creating new route to $destination_cidr"
        aws ec2 create-route \
            --route-table-id $route_table_id \
            --destination-cidr-block $destination_cidr \
            --vpc-peering-connection-id $peering_connection_id \
            --region $region
    fi
}

# Function to manage route propagation
manage_route_propagation() {
    local source_region=$1
    local target_region=$2
    local peering_connection_id=$3
    
    echo "Managing route propagation between $source_region and $target_region"
    
    # Get VPC IDs and CIDR blocks
    source_vpc_info=$(aws ec2 describe-vpc-peering-connections \
        --vpc-peering-connection-ids $peering_connection_id \
        --region $source_region \
        --query 'VpcPeeringConnections[0].RequesterVpcInfo')
    
    target_vpc_info=$(aws ec2 describe-vpc-peering-connections \
        --vpc-peering-connection-ids $peering_connection_id \
        --region $source_region \
        --query 'VpcPeeringConnections[0].AccepterVpcInfo')
    
    # Extract information
    source_vpc_id=$(echo $source_vpc_info | jq -r '.VpcId')
    source_cidr=$(echo $source_vpc_info | jq -r '.CidrBlock')
    target_vpc_id=$(echo $target_vpc_info | jq -r '.VpcId')
    target_cidr=$(echo $target_vpc_info | jq -r '.CidrBlock')
    
    echo "Source VPC: $source_vpc_id ($source_cidr)"
    echo "Target VPC: $target_vpc_id ($target_cidr)"
    
    # Update route tables in source region
    source_route_tables=$(aws ec2 describe-route-tables \
        --filters "Name=vpc-id,Values=$source_vpc_id" \
        --region $source_region \
        --query 'RouteTables[].RouteTableId' \
        --output text)
    
    for rt_id in $source_route_tables; do
        update_route_table $source_region $rt_id $target_cidr $peering_connection_id
    done
    
    # Update route tables in target region
    target_route_tables=$(aws ec2 describe-route-tables \
        --filters "Name=vpc-id,Values=$target_vpc_id" \
        --region $target_region \
        --query 'RouteTables[].RouteTableId' \
        --output text)
    
    for rt_id in $target_route_tables; do
        update_route_table $target_region $rt_id $source_cidr $peering_connection_id
    done
}

# Function to validate routing configuration
validate_routing() {
    local region1=$1
    local region2=$2
    local peering_connection_id=$3
    
    echo "Validating routing configuration..."
    
    # Test connectivity (requires instances in both regions)
    echo "Manual connectivity testing required:"
    echo "1. Launch test instances in both regions"
    echo "2. Ensure security groups allow ICMP/SSH"
    echo "3. Test ping between private IP addresses"
    echo "4. Verify DNS resolution if enabled"
    
    # Check route table entries
    echo "Verifying route table entries..."
    
    aws ec2 describe-route-tables \
        --region $region1 \
        --query "RouteTables[?Routes[?VpcPeeringConnectionId=='$peering_connection_id']].[RouteTableId,Routes[?VpcPeeringConnectionId=='$peering_connection_id'].DestinationCidrBlock]" \
        --output table
    
    aws ec2 describe-route-tables \
        --region $region2 \
        --query "RouteTables[?Routes[?VpcPeeringConnectionId=='$peering_connection_id']].[RouteTableId,Routes[?VpcPeeringConnectionId=='$peering_connection_id'].DestinationCidrBlock]" \
        --output table
}

# Main execution
PEERING_CONNECTION_ID="pcx-1234567890abcdef0"
SOURCE_REGION="us-east-1"
TARGET_REGION="eu-west-1"

manage_route_propagation $SOURCE_REGION $TARGET_REGION $PEERING_CONNECTION_ID
validate_routing $SOURCE_REGION $TARGET_REGION $PEERING_CONNECTION_ID

echo "Route table management completed"
```

## Security Considerations {#security}

### Security Group Configuration

```yaml
# Security group configuration for inter-region peering
SecurityGroupStrategy:
  
  Principle: "Least Privilege Access"
  
  Web_Tier_Security_Groups:
    Source_Region_Web_SG:
      GroupName: "web-tier-us-east-1"
      Description: "Web tier security group for us-east-1"
      InboundRules:
        - Protocol: TCP
          Port: 80
          Source: "0.0.0.0/0"
          Description: "HTTP from anywhere"
        - Protocol: TCP
          Port: 443
          Source: "0.0.0.0/0"
          Description: "HTTPS from anywhere"
        - Protocol: TCP
          Port: 22
          Source: "10.0.0.0/8"
          Description: "SSH from internal networks"
      OutboundRules:
        - Protocol: TCP
          Port: 3306
          Destination: "app-tier-eu-west-1-sg"
          Description: "MySQL to app tier in eu-west-1"
        - Protocol: TCP
          Port: 443
          Destination: "0.0.0.0/0"
          Description: "HTTPS outbound"
    
    Target_Region_App_SG:
      GroupName: "app-tier-eu-west-1"
      Description: "App tier security group for eu-west-1"
      InboundRules:
        - Protocol: TCP
          Port: 3306
          Source: "web-tier-us-east-1-sg"
          Description: "MySQL from web tier in us-east-1"
        - Protocol: TCP
          Port: 8080
          Source: "10.2.0.0/16"
          Description: "App traffic from local VPC"
      OutboundRules:
        - Protocol: TCP
          Port: 443
          Destination: "0.0.0.0/0"
          Description: "HTTPS outbound for external APIs"
  
  Cross_Region_Communication:
    Strategy: "Named Security Group References"
    Implementation:
      - Reference security groups by ID across regions
      - Use descriptive naming conventions
      - Document cross-region dependencies
      - Implement regular access reviews
      
    Example_Configuration:
      WebTierToAppTier:
        Source: "sg-12345678 (web-tier-us-east-1)"
        Target: "sg-87654321 (app-tier-eu-west-1)"
        Protocol: "TCP"
        Port: "3306"
        Purpose: "Database connectivity"
        
      AppTierToCache:
        Source: "sg-87654321 (app-tier-eu-west-1)"
        Target: "sg-abcdef12 (cache-tier-eu-west-1)"
        Protocol: "TCP"
        Port: "6379"
        Purpose: "Redis cache access"
```

### Network ACL Configuration

```bash
#!/bin/bash
# Network ACL configuration for inter-region security

# Create restrictive NACLs for cross-region traffic
create_cross_region_nacl() {
    local region=$1
    local vpc_id=$2
    local remote_cidr=$3
    local nacl_name=$4
    
    echo "Creating NACL $nacl_name in $region for VPC $vpc_id"
    
    # Create Network ACL
    NACL_ID=$(aws ec2 create-network-acl \
        --vpc-id $vpc_id \
        --region $region \
        --tag-specifications "ResourceType=network-acl,Tags=[{Key=Name,Value=$nacl_name}]" \
        --query 'NetworkAcl.NetworkAclId' \
        --output text)
    
    echo "Created NACL: $NACL_ID"
    
    # Inbound rules
    
    # Allow HTTP from remote region
    aws ec2 create-network-acl-entry \
        --network-acl-id $NACL_ID \
        --rule-number 100 \
        --protocol tcp \
        --port-range From=80,To=80 \
        --cidr-block $remote_cidr \
        --rule-action allow \
        --region $region
    
    # Allow HTTPS from remote region
    aws ec2 create-network-acl-entry \
        --network-acl-id $NACL_ID \
        --rule-number 110 \
        --protocol tcp \
        --port-range From=443,To=443 \
        --cidr-block $remote_cidr \
        --rule-action allow \
        --region $region
    
    # Allow MySQL from remote region
    aws ec2 create-network-acl-entry \
        --network-acl-id $NACL_ID \
        --rule-number 120 \
        --protocol tcp \
        --port-range From=3306,To=3306 \
        --cidr-block $remote_cidr \
        --rule-action allow \
        --region $region
    
    # Allow ephemeral ports for return traffic
    aws ec2 create-network-acl-entry \
        --network-acl-id $NACL_ID \
        --rule-number 200 \
        --protocol tcp \
        --port-range From=1024,To=65535 \
        --cidr-block $remote_cidr \
        --rule-action allow \
        --region $region
    
    # Outbound rules
    
    # Allow HTTP to remote region
    aws ec2 create-network-acl-entry \
        --network-acl-id $NACL_ID \
        --rule-number 100 \
        --protocol tcp \
        --port-range From=80,To=80 \
        --cidr-block $remote_cidr \
        --rule-action allow \
        --egress \
        --region $region
    
    # Allow HTTPS to remote region
    aws ec2 create-network-acl-entry \
        --network-acl-id $NACL_ID \
        --rule-number 110 \
        --protocol tcp \
        --port-range From=443,To=443 \
        --cidr-block $remote_cidr \
        --rule-action allow \
        --egress \
        --region $region
    
    # Allow MySQL to remote region
    aws ec2 create-network-acl-entry \
        --network-acl-id $NACL_ID \
        --rule-number 120 \
        --protocol tcp \
        --port-range From=3306,To=3306 \
        --cidr-block $remote_cidr \
        --rule-action allow \
        --egress \
        --region $region
    
    # Allow ephemeral ports for outbound responses
    aws ec2 create-network-acl-entry \
        --network-acl-id $NACL_ID \
        --rule-number 200 \
        --protocol tcp \
        --port-range From=1024,To=65535 \
        --cidr-block $remote_cidr \
        --rule-action allow \
        --egress \
        --region $region
    
    echo "NACL $NACL_ID configured with cross-region rules"
    return $NACL_ID
}

# Associate NACL with subnets
associate_nacl_with_subnets() {
    local region=$1
    local vpc_id=$2
    local nacl_id=$3
    local subnet_type=$4  # "private" or "public"
    
    echo "Associating NACL $nacl_id with $subnet_type subnets in $region"
    
    # Get subnets of specified type
    SUBNET_IDS=$(aws ec2 describe-subnets \
        --filters "Name=vpc-id,Values=$vpc_id" "Name=tag:Type,Values=$subnet_type" \
        --region $region \
        --query 'Subnets[].SubnetId' \
        --output text)
    
    for subnet_id in $SUBNET_IDS; do
        echo "Associating NACL with subnet $subnet_id"
        
        # Get current association
        ASSOCIATION_ID=$(aws ec2 describe-network-acls \
            --filters "Name=association.subnet-id,Values=$subnet_id" \
            --region $region \
            --query 'NetworkAcls[0].Associations[?SubnetId==`'$subnet_id'`].NetworkAclAssociationId' \
            --output text)
        
        if [ "$ASSOCIATION_ID" != "" ]; then
            # Replace existing association
            aws ec2 replace-network-acl-association \
                --association-id $ASSOCIATION_ID \
                --network-acl-id $nacl_id \
                --region $region
        fi
    done
}

# Example usage
US_EAST_VPC="vpc-12345678"
EU_WEST_VPC="vpc-87654321"
US_EAST_CIDR="10.0.0.0/16"
EU_WEST_CIDR="10.1.0.0/16"

# Create NACLs for both regions
create_cross_region_nacl "us-east-1" $US_EAST_VPC $EU_WEST_CIDR "cross-region-nacl-us-east-1"
create_cross_region_nacl "eu-west-1" $EU_WEST_VPC $US_EAST_CIDR "cross-region-nacl-eu-west-1"

echo "Cross-region NACL configuration completed"
```

## Data Transfer Costs {#costs}

### Cost Analysis and Optimization

```python
import boto3
import json
from datetime import datetime, timedelta
from typing import Dict, List

class InterRegionCostAnalyzer:
    def __init__(self):
        self.ce_client = boto3.client('ce')  # Cost Explorer
        self.ec2_clients = {}
        
    def get_data_transfer_costs(self, start_date: str, end_date: str, regions: List[str]) -> Dict:
        """Analyze data transfer costs for inter-region communication"""
        
        try:
            response = self.ce_client.get_cost_and_usage(
                TimePeriod={'Start': start_date, 'End': end_date},
                Granularity='DAILY',
                Metrics=['UnblendedCost'],
                GroupBy=[
                    {'Type': 'DIMENSION', 'Key': 'REGION'},
                    {'Type': 'DIMENSION', 'Key': 'USAGE_TYPE'}
                ],
                Filter={
                    'Dimensions': {
                        'Key': 'SERVICE',
                        'Values': ['Amazon Elastic Compute Cloud - Compute']
                    }
                }
            )
            
            transfer_costs = {}
            
            for result in response['ResultsByTime']:
                date = result['TimePeriod']['Start']
                transfer_costs[date] = {}
                
                for group in result['Groups']:
                    if len(group['Keys']) >= 2:
                        region = group['Keys'][0]
                        usage_type = group['Keys'][1]
                        cost = float(group['Metrics']['UnblendedCost']['Amount'])
                        
                        # Filter for data transfer usage types
                        if 'DataTransfer' in usage_type and region in regions:
                            if region not in transfer_costs[date]:
                                transfer_costs[date][region] = {}
                            transfer_costs[date][region][usage_type] = cost
            
            return transfer_costs
            
        except Exception as e:
            print(f"Error retrieving cost data: {e}")
            return {}
    
    def calculate_inter_region_costs(self, regions: List[str], monthly_gb: float) -> Dict:
        """Calculate estimated inter-region data transfer costs"""
        
        # AWS inter-region data transfer pricing (USD per GB)
        # Prices as of 2024 - verify current pricing
        pricing_matrix = {
            ('us-east-1', 'us-west-1'): 0.02,
            ('us-east-1', 'us-west-2'): 0.02,
            ('us-east-1', 'eu-west-1'): 0.02,
            ('us-east-1', 'ap-south-1'): 0.09,
            ('us-west-2', 'eu-west-1'): 0.02,
            ('us-west-2', 'ap-south-1'): 0.09,
            ('eu-west-1', 'ap-south-1'): 0.09,
        }
        
        cost_analysis = {
            'monthly_costs': {},
            'annual_projection': {},
            'optimization_opportunities': []
        }
        
        for (source, destination), price_per_gb in pricing_matrix.items():
            if source in regions and destination in regions:
                monthly_cost = monthly_gb * price_per_gb
                annual_cost = monthly_cost * 12
                
                route = f"{source} -> {destination}"
                cost_analysis['monthly_costs'][route] = {
                    'data_gb': monthly_gb,
                    'price_per_gb': price_per_gb,
                    'monthly_cost': monthly_cost
                }
                cost_analysis['annual_projection'][route] = annual_cost
                
                # Identify optimization opportunities
                if price_per_gb > 0.05:  # High-cost routes
                    cost_analysis['optimization_opportunities'].append({
                        'route': route,
                        'current_cost': monthly_cost,
                        'recommendation': 'Consider CloudFront or regional optimization',
                        'potential_savings': monthly_cost * 0.3  # Estimated 30% savings
                    })
        
        return cost_analysis
    
    def recommend_cost_optimizations(self, transfer_patterns: Dict) -> List[Dict]:
        """Recommend cost optimization strategies"""
        
        recommendations = []
        
        # Analyze transfer patterns for optimization opportunities
        total_monthly_transfer = sum(
            pattern.get('monthly_gb', 0) 
            for pattern in transfer_patterns.values()
        )
        
        if total_monthly_transfer > 100:  # > 100 GB/month
            recommendations.append({
                'strategy': 'CloudFront Implementation',
                'description': 'Implement CloudFront for static content delivery',
                'estimated_savings': total_monthly_transfer * 0.015 * 0.6,  # 60% reduction in transfer costs
                'implementation_effort': 'Medium',
                'prerequisites': ['Static content identification', 'Cache strategy design']
            })
        
        if total_monthly_transfer > 1000:  # > 1 TB/month
            recommendations.append({
                'strategy': 'Regional Caching',
                'description': 'Implement regional caching layers (ElastiCache)',
                'estimated_savings': total_monthly_transfer * 0.02 * 0.4,  # 40% reduction in cross-region requests
                'implementation_effort': 'High',
                'prerequisites': ['Cache architecture design', 'Application modifications']
            })
        
        # Check for inefficient routing patterns
        high_cost_routes = [
            route for route, data in transfer_patterns.items()
            if data.get('price_per_gb', 0) > 0.05
        ]
        
        if high_cost_routes:
            recommendations.append({
                'strategy': 'Route Optimization',
                'description': f'Optimize high-cost routes: {", ".join(high_cost_routes)}',
                'estimated_savings': sum(
                    transfer_patterns[route].get('monthly_cost', 0) * 0.2
                    for route in high_cost_routes
                ),
                'implementation_effort': 'Low',
                'prerequisites': ['Architecture review', 'Alternative routing analysis']
            })
        
        return recommendations
    
    def create_cost_dashboard_data(self, cost_analysis: Dict, recommendations: List[Dict]) -> Dict:
        """Create dashboard data for cost visualization"""
        
        dashboard = {
            'summary': {
                'total_monthly_cost': sum(cost_analysis['monthly_costs'].values()),
                'total_annual_projection': sum(cost_analysis['annual_projection'].values()),
                'high_cost_routes': len([
                    route for route, data in cost_analysis['monthly_costs'].items()
                    if data['price_per_gb'] > 0.05
                ]),
                'optimization_potential': sum(
                    rec['estimated_savings'] for rec in recommendations
                )
            },
            'cost_breakdown': cost_analysis['monthly_costs'],
            'optimization_recommendations': recommendations,
            'charts': {
                'monthly_costs_by_route': [
                    {'route': route, 'cost': data['monthly_cost']}
                    for route, data in cost_analysis['monthly_costs'].items()
                ],
                'price_per_gb_comparison': [
                    {'route': route, 'price': data['price_per_gb']}
                    for route, data in cost_analysis['monthly_costs'].items()
                ]
            }
        }
        
        return dashboard

# Usage example
cost_analyzer = InterRegionCostAnalyzer()

# Define transfer patterns
transfer_patterns = {
    'us-east-1 -> eu-west-1': {'monthly_gb': 500, 'price_per_gb': 0.02},
    'us-east-1 -> ap-south-1': {'monthly_gb': 200, 'price_per_gb': 0.09},
    'eu-west-1 -> ap-south-1': {'monthly_gb': 150, 'price_per_gb': 0.09}
}

# Calculate costs
regions = ['us-east-1', 'eu-west-1', 'ap-south-1']
cost_analysis = cost_analyzer.calculate_inter_region_costs(regions, 500)

# Get recommendations
recommendations = cost_analyzer.recommend_cost_optimizations(transfer_patterns)

# Create dashboard
dashboard = cost_analyzer.create_cost_dashboard_data(cost_analysis, recommendations)

print("Cost Analysis Dashboard:")
print(json.dumps(dashboard, indent=2))

# Get historical data (last 30 days)
end_date = datetime.now().date()
start_date = end_date - timedelta(days=30)

historical_costs = cost_analyzer.get_data_transfer_costs(
    start_date.isoformat(),
    end_date.isoformat(),
    regions
)

print("\nHistorical Transfer Costs:")
print(json.dumps(historical_costs, indent=2, default=str))
```

### Cost Monitoring Setup

```bash
#!/bin/bash
# Set up CloudWatch alarms for data transfer cost monitoring

# Function to create cost alarm
create_cost_alarm() {
    local alarm_name=$1
    local threshold=$2
    local service=$3
    local region_filter=$4
    
    echo "Creating cost alarm: $alarm_name"
    
    # Create CloudWatch alarm for costs
    aws cloudwatch put-metric-alarm \
        --alarm-name "$alarm_name" \
        --alarm-description "Alert when $service costs exceed $threshold USD" \
        --metric-name EstimatedCharges \
        --namespace AWS/Billing \
        --statistic Maximum \
        --period 86400 \
        --threshold $threshold \
        --comparison-operator GreaterThanThreshold \
        --evaluation-periods 1 \
        --alarm-actions "arn:aws:sns:us-east-1:$AWS_ACCOUNT_ID:cost-alerts" \
        --dimensions Name=ServiceName,Value="$service" Name=Currency,Value=USD
}

# Function to create budget for data transfer
create_data_transfer_budget() {
    local budget_name=$1
    local budget_limit=$2
    
    echo "Creating budget: $budget_name"
    
    cat > budget-config.json << EOF
{
  "BudgetName": "$budget_name",
  "BudgetLimit": {
    "Amount": "$budget_limit",
    "Unit": "USD"
  },
  "TimeUnit": "MONTHLY",
  "TimePeriod": {
    "Start": "$(date +%Y-%m-01)",
    "End": "2087-06-15"
  },
  "BudgetType": "COST",
  "CostFilters": {
    "Service": ["Amazon Elastic Compute Cloud - Compute"],
    "UsageType": ["DataTransfer-Out-Bytes", "DataTransfer-In-Bytes"]
  }
}
EOF

    cat > budget-notification.json << EOF
{
  "Notification": {
    "NotificationType": "ACTUAL",
    "ComparisonOperator": "GREATER_THAN",
    "Threshold": 80,
    "ThresholdType": "PERCENTAGE",
    "NotificationState": "ALARM"
  },
  "Subscribers": [
    {
      "SubscriptionType": "EMAIL",
      "Address": "admin@example.com"
    }
  ]
}
EOF

    aws budgets create-budget \
        --account-id $AWS_ACCOUNT_ID \
        --budget file://budget-config.json \
        --notifications-with-subscribers file://budget-notification.json
    
    rm budget-config.json budget-notification.json
}

# Create cost alarms for different thresholds
create_cost_alarm "DataTransfer-Monthly-Warning" 100 "Amazon Elastic Compute Cloud - Compute"
create_cost_alarm "DataTransfer-Monthly-Critical" 500 "Amazon Elastic Compute Cloud - Compute"

# Create budget for data transfer costs
create_data_transfer_budget "InterRegion-DataTransfer-Budget" 1000

echo "Cost monitoring setup completed"
```

This comprehensive guide provides the foundation for implementing inter-region VPC peering with proper setup, routing configuration, security considerations, and cost management strategies. The examples and tools provided can be adapted to specific organizational requirements while maintaining best practices for performance and security.