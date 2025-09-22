# Topic 63: Edge Computing - Local Zones and Wavelength Architecture

## Table of Contents
1. [Edge Computing Overview](#overview)
2. [AWS Local Zones Architecture](#local-zones)
3. [AWS Wavelength for 5G Applications](#wavelength)
4. [Edge Computing Patterns](#patterns)
5. [Network Architecture Design](#network-design)
6. [Performance Optimization](#performance)
7. [Application Design for Edge](#application-design)
8. [Security in Edge Computing](#security)
9. [Monitoring and Observability](#monitoring)
10. [Cost Management](#cost-management)
11. [Implementation Examples](#implementation)
12. [Best Practices](#best-practices)

## Edge Computing Overview {#overview}

### What is Edge Computing?

Edge computing brings computation and data storage closer to end users and devices, reducing latency and improving application performance. AWS provides several edge computing solutions to address different use cases and proximity requirements.

**AWS Edge Computing Spectrum**:

```
Edge Computing Proximity Spectrum:

Ultra-Low Latency (< 5ms)     Low Latency (5-20ms)     Regional (20-100ms)
┌─────────────────────┐     ┌─────────────────────┐     ┌─────────────────────┐
│   AWS Wavelength   │     │   AWS Local Zones   │     │  AWS Availability   │
│                     │     │                     │     │      Zones          │
│ • 5G network edge  │     │ • Metro area edge   │     │                     │
│ • Sub-5ms latency  │     │ • 10-20ms latency   │     │ • Regional services │
│ • Mobile/IoT apps  │     │ • Media processing   │     │ • Full AWS services │
│ • Real-time gaming │     │ • Real-time analytics│     │ • Traditional apps  │
│ • AR/VR apps       │     │ • Content caching   │     │ • Batch processing  │
└─────────────────────┘     └─────────────────────┘     └─────────────────────┘
         │                           │                           │
         ▼                           ▼                           ▼
   Carrier Network              Metro Fiber               Regional Data Center
   Infrastructure              Infrastructure                Infrastructure
```

### Edge Computing Use Cases

```yaml
Edge_Computing_Use_Cases:
  
  Ultra_Low_Latency_Applications:
    Gaming:
      - Real-time multiplayer gaming
      - Cloud gaming streaming
      - AR/VR experiences
      - Interactive simulations
      
    Industrial_IoT:
      - Autonomous vehicles
      - Robotic control systems
      - Industrial automation
      - Safety-critical systems
      
    Media_Streaming:
      - Live video streaming
      - Interactive broadcasting
      - Real-time video processing
      - Low-latency CDN
      
  Proximity_Sensitive_Applications:
    Content_Delivery:
      - Video on demand
      - Software distribution
      - Website acceleration
      - API response caching
      
    Data_Processing:
      - Local data aggregation
      - Real-time analytics
      - Edge AI/ML inference
      - Stream processing
      
    Compliance_Requirements:
      - Data residency
      - Local data processing
      - Regional compliance
      - Privacy regulations
```

### AWS Edge Services Comparison

```
Service Comparison Matrix:

┌──────────────────┬────────────────┬────────────────┬────────────────┐
│    Aspect        │  Wavelength    │  Local Zones   │   Outposts     │
├──────────────────┼────────────────┼────────────────┼────────────────┤
│ Location         │ 5G Carrier    │ Metro Areas    │ On-premises    │
│                  │ Networks       │                │                │
├──────────────────┼────────────────┼────────────────┼────────────────┤
│ Latency          │ < 5ms          │ 5-20ms         │ On-site        │
├──────────────────┼────────────────┼────────────────┼────────────────┤
│ Connectivity     │ 5G Wireless    │ Fiber/Internet │ Local Network  │
├──────────────────┼────────────────┼────────────────┼────────────────┤
│ Use Cases        │ Mobile Apps    │ Media/Gaming   │ Hybrid Cloud   │
│                  │ IoT Devices    │ Content Delivery│ Data Residency │
├──────────────────┼────────────────┼────────────────┼────────────────┤
│ AWS Services     │ Subset         │ Extended Subset│ Full Stack     │
├──────────────────┼────────────────┼────────────────┼────────────────┤
│ Management       │ AWS Managed    │ AWS Managed    │ Customer       │
│                  │                │                │ Managed        │
└──────────────────┴────────────────┴────────────────┴────────────────┘
```

## AWS Local Zones Architecture {#local-zones}

### Local Zones Overview

AWS Local Zones are a type of AWS infrastructure deployment that places compute, storage, database, and other select services closer to large population centers, enabling single-digit millisecond latency to end users.

```
Local Zones Architecture:

                          ┌─────────────────────────────────┐
                          │         Parent Region           │
                          │         (us-east-1)             │
                          │                                 │
                          │  ┌─────────────────────────┐   │
                          │  │    Availability Zone    │   │
                          │  │         AZ-1a           │   │
                          │  └─────────────────────────┘   │
                          │  ┌─────────────────────────┐   │
                          │  │    Availability Zone    │   │
                          │  │         AZ-1b           │   │
                          │  └─────────────────────────┘   │
                          │  ┌─────────────────────────┐   │
                          │  │    Availability Zone    │   │
                          │  │         AZ-1c           │   │
                          │  └─────────────────────────┘   │
                          └─────────────────┬───────────────┘
                                            │
                              High-bandwidth, low-latency
                                      connection
                                            │
                          ┌─────────────────▼───────────────┐
                          │        Local Zone               │
                          │     (us-east-1-bos-1a)         │
                          │      Boston Metro Area          │
                          │                                 │
                          │  ┌─────────────────────────┐   │
                          │  │      Edge Services      │   │
                          │  │  • EC2 Instances        │   │
                          │  │  • EBS Volumes          │   │
                          │  │  • Application LB       │   │
                          │  │  • VPC Subnets          │   │
                          │  │  • Security Groups      │   │
                          │  └─────────────────────────┘   │
                          └─────────────────┬───────────────┘
                                            │
                                     < 10ms latency
                                            │
                          ┌─────────────────▼───────────────┐
                          │        End Users                │
                          │      Boston Area Users          │
                          └─────────────────────────────────┘
```

### Local Zones Implementation

```python
import boto3
import json
from typing import Dict, List, Optional

class LocalZonesManager:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
        self.elbv2 = boto3.client('elbv2')
        
    def discover_local_zones(self, region: str = None) -> Dict:
        """Discover available Local Zones and their characteristics"""
        
        try:
            # Get all available zones
            response = self.ec2.describe_availability_zones(
                Filters=[
                    {'Name': 'zone-type', 'Values': ['local-zone']}
                ]
            )
            
            local_zones = {}
            
            for zone in response['AvailabilityZones']:
                zone_id = zone['ZoneId']
                zone_name = zone['ZoneName']
                
                # Extract location information
                location_parts = zone_name.split('-')
                if len(location_parts) >= 4:
                    metro_code = location_parts[3]
                else:
                    metro_code = 'unknown'
                
                local_zones[zone_name] = {
                    'zone_id': zone_id,
                    'zone_name': zone_name,
                    'state': zone['State'],
                    'parent_region': zone['RegionName'],
                    'metro_code': metro_code,
                    'network_border_group': zone.get('NetworkBorderGroup', ''),
                    'available_services': self._get_available_services(zone_name)
                }
            
            return {
                'total_local_zones': len(local_zones),
                'local_zones': local_zones,
                'metro_areas': list(set(zone['metro_code'] for zone in local_zones.values()))
            }
            
        except Exception as e:
            return {'error': str(e)}
    
    def _get_available_services(self, zone_name: str) -> List[str]:
        """Get list of available services in a Local Zone"""
        
        # This is a representative list - actual availability may vary
        common_services = [
            'EC2',
            'EBS',
            'VPC',
            'Subnet',
            'Security Groups',
            'Application Load Balancer',
            'Auto Scaling',
            'FSx for Lustre'
        ]
        
        # Some services may have limited availability in certain zones
        try:
            # Check if we can describe instances in this zone
            self.ec2.describe_instances(
                Filters=[{'Name': 'availability-zone', 'Values': [zone_name]}]
            )
            return common_services
        except:
            return ['Limited service availability']
    
    def create_local_zone_infrastructure(self, 
                                       config: Dict) -> Dict:
        """Create infrastructure in Local Zone"""
        
        results = {
            'vpc_id': None,
            'subnet_id': None,
            'security_group_id': None,
            'instance_ids': [],
            'load_balancer_arn': None,
            'created_resources': []
        }
        
        try:
            # Create VPC (if not provided)
            if 'vpc_id' not in config:
                vpc_response = self.ec2.create_vpc(
                    CidrBlock=config.get('vpc_cidr', '10.0.0.0/16'),
                    TagSpecifications=[{
                        'ResourceType': 'vpc',
                        'Tags': [
                            {'Key': 'Name', 'Value': f"LocalZone-VPC-{config['local_zone']}"},
                            {'Key': 'LocalZone', 'Value': config['local_zone']}
                        ]
                    }]
                )
                results['vpc_id'] = vpc_response['Vpc']['VpcId']
                results['created_resources'].append(f"VPC: {results['vpc_id']}")
            else:
                results['vpc_id'] = config['vpc_id']
            
            # Create subnet in Local Zone
            subnet_response = self.ec2.create_subnet(
                VpcId=results['vpc_id'],
                CidrBlock=config.get('subnet_cidr', '10.0.1.0/24'),
                AvailabilityZone=config['local_zone'],
                TagSpecifications=[{
                    'ResourceType': 'subnet',
                    'Tags': [
                        {'Key': 'Name', 'Value': f"LocalZone-Subnet-{config['local_zone']}"},
                        {'Key': 'LocalZone', 'Value': config['local_zone']},
                        {'Key': 'Type', 'Value': 'Edge'}
                    ]
                }]
            )
            results['subnet_id'] = subnet_response['Subnet']['SubnetId']
            results['created_resources'].append(f"Subnet: {results['subnet_id']}")
            
            # Create security group
            sg_response = self.ec2.create_security_group(
                GroupName=f"LocalZone-SG-{config['local_zone']}",
                Description=f"Security group for Local Zone {config['local_zone']}",
                VpcId=results['vpc_id'],
                TagSpecifications=[{
                    'ResourceType': 'security-group',
                    'Tags': [
                        {'Key': 'Name', 'Value': f"LocalZone-SG-{config['local_zone']}"},
                        {'Key': 'LocalZone', 'Value': config['local_zone']}
                    ]
                }]
            )
            results['security_group_id'] = sg_response['GroupId']
            results['created_resources'].append(f"Security Group: {results['security_group_id']}")
            
            # Configure security group rules
            self._configure_security_group_rules(
                results['security_group_id'],
                config.get('security_rules', [])
            )
            
            # Launch EC2 instances
            if config.get('instance_config'):
                instance_results = self._launch_instances(
                    config['instance_config'],
                    results['subnet_id'],
                    results['security_group_id'],
                    config['local_zone']
                )
                results['instance_ids'] = instance_results
                results['created_resources'].extend([f"Instance: {iid}" for iid in instance_results])
            
            # Create Application Load Balancer
            if config.get('create_alb', False):
                alb_arn = self._create_load_balancer(
                    results['subnet_id'],
                    results['security_group_id'],
                    config['local_zone']
                )
                results['load_balancer_arn'] = alb_arn
                results['created_resources'].append(f"ALB: {alb_arn}")
            
            return results
            
        except Exception as e:
            return {'error': str(e), 'partial_results': results}
    
    def _configure_security_group_rules(self, 
                                      security_group_id: str,
                                      rules: List[Dict]):
        """Configure security group rules"""
        
        default_rules = [
            {
                'IpProtocol': 'tcp',
                'FromPort': 80,
                'ToPort': 80,
                'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
            },
            {
                'IpProtocol': 'tcp',
                'FromPort': 443,
                'ToPort': 443,
                'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
            }
        ]
        
        rules_to_apply = rules if rules else default_rules
        
        try:
            self.ec2.authorize_security_group_ingress(
                GroupId=security_group_id,
                IpPermissions=rules_to_apply
            )
        except Exception as e:
            print(f"Warning: Failed to configure security group rules: {e}")
    
    def _launch_instances(self,
                        instance_config: Dict,
                        subnet_id: str,
                        security_group_id: str,
                        local_zone: str) -> List[str]:
        """Launch EC2 instances in Local Zone"""
        
        try:
            response = self.ec2.run_instances(
                ImageId=instance_config.get('ami_id', 'ami-0c55b159cbfafe1d0'),
                MinCount=instance_config.get('min_count', 1),
                MaxCount=instance_config.get('max_count', 2),
                InstanceType=instance_config.get('instance_type', 't3.medium'),
                SubnetId=subnet_id,
                SecurityGroupIds=[security_group_id],
                Placement={'AvailabilityZone': local_zone},
                TagSpecifications=[{
                    'ResourceType': 'instance',
                    'Tags': [
                        {'Key': 'Name', 'Value': f"LocalZone-Instance-{local_zone}"},
                        {'Key': 'LocalZone', 'Value': local_zone},
                        {'Key': 'Purpose', 'Value': instance_config.get('purpose', 'Edge Computing')}
                    ]
                }],
                UserData=instance_config.get('user_data', '')
            )
            
            return [instance['InstanceId'] for instance in response['Instances']]
            
        except Exception as e:
            print(f"Failed to launch instances: {e}")
            return []
    
    def _create_load_balancer(self,
                            subnet_id: str,
                            security_group_id: str,
                            local_zone: str) -> Optional[str]:
        """Create Application Load Balancer in Local Zone"""
        
        try:
            response = self.elbv2.create_load_balancer(
                Name=f"LocalZone-ALB-{local_zone.replace('.', '-')}",
                Subnets=[subnet_id],
                SecurityGroups=[security_group_id],
                Scheme='internet-facing',
                Type='application',
                IpAddressType='ipv4',
                Tags=[
                    {'Key': 'Name', 'Value': f"LocalZone-ALB-{local_zone}"},
                    {'Key': 'LocalZone', 'Value': local_zone}
                ]
            )
            
            return response['LoadBalancers'][0]['LoadBalancerArn']
            
        except Exception as e:
            print(f"Failed to create load balancer: {e}")
            return None
    
    def analyze_latency_improvement(self,
                                  local_zone: str,
                                  target_region: str) -> Dict:
        """Analyze potential latency improvement using Local Zone"""
        
        # This would typically involve actual network measurements
        # For demo purposes, we'll use estimated values
        
        metro_latency_estimates = {
            'bos': {'region_latency': 45, 'local_zone_latency': 8},   # Boston
            'chi': {'region_latency': 35, 'local_zone_latency': 6},   # Chicago
            'dfw': {'region_latency': 40, 'local_zone_latency': 7},   # Dallas
            'den': {'region_latency': 50, 'local_zone_latency': 9},   # Denver
            'iah': {'region_latency': 42, 'local_zone_latency': 8},   # Houston
            'lax': {'region_latency': 15, 'local_zone_latency': 5},   # Los Angeles
            'mia': {'region_latency': 55, 'local_zone_latency': 10},  # Miami
            'msp': {'region_latency': 38, 'local_zone_latency': 7},   # Minneapolis
            'nyc': {'region_latency': 25, 'local_zone_latency': 5},   # New York
            'phl': {'region_latency': 35, 'local_zone_latency': 7},   # Philadelphia
            'phx': {'region_latency': 60, 'local_zone_latency': 12},  # Phoenix
            'pdx': {'region_latency': 70, 'local_zone_latency': 15},  # Portland
            'sea': {'region_latency': 65, 'local_zone_latency': 12}   # Seattle
        }
        
        # Extract metro code from local zone name
        metro_code = local_zone.split('-')[3] if len(local_zone.split('-')) > 3 else 'unknown'
        
        if metro_code in metro_latency_estimates:
            estimates = metro_latency_estimates[metro_code]
            
            latency_improvement = estimates['region_latency'] - estimates['local_zone_latency']
            improvement_percentage = (latency_improvement / estimates['region_latency']) * 100
            
            return {
                'local_zone': local_zone,
                'metro_area': metro_code,
                'target_region': target_region,
                'estimated_region_latency_ms': estimates['region_latency'],
                'estimated_local_zone_latency_ms': estimates['local_zone_latency'],
                'latency_improvement_ms': latency_improvement,
                'improvement_percentage': round(improvement_percentage, 1),
                'use_case_benefits': self._generate_use_case_benefits(latency_improvement)
            }
        else:
            return {
                'error': f'No latency estimates available for metro area: {metro_code}',
                'local_zone': local_zone
            }
    
    def _generate_use_case_benefits(self, latency_improvement: float) -> List[str]:
        """Generate use case benefits based on latency improvement"""
        
        benefits = []
        
        if latency_improvement > 40:
            benefits.extend([
                'Excellent for real-time gaming applications',
                'Ideal for live video streaming',
                'Suitable for IoT control systems',
                'Perfect for AR/VR applications'
            ])
        elif latency_improvement > 20:
            benefits.extend([
                'Good for interactive web applications',
                'Suitable for real-time analytics',
                'Beneficial for content delivery',
                'Improves user experience significantly'
            ])
        elif latency_improvement > 10:
            benefits.extend([
                'Moderate improvement for web applications',
                'Better API response times',
                'Enhanced user experience'
            ])
        else:
            benefits.append('Minimal latency improvement')
        
        return benefits

# Usage example
local_zones_manager = LocalZonesManager()

# Discover available Local Zones
local_zones_info = local_zones_manager.discover_local_zones()
print("Available Local Zones:")
print(json.dumps(local_zones_info, indent=2))

# Create infrastructure in Local Zone
infrastructure_config = {
    'local_zone': 'us-east-1-bos-1a',  # Boston Local Zone
    'vpc_cidr': '10.0.0.0/16',
    'subnet_cidr': '10.0.1.0/24',
    'create_alb': True,
    'instance_config': {
        'ami_id': 'ami-0c55b159cbfafe1d0',  # Replace with actual AMI
        'instance_type': 't3.medium',
        'min_count': 2,
        'max_count': 2,
        'purpose': 'Edge Web Servers',
        'user_data': '''#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Edge Server in Local Zone</h1>" > /var/www/html/index.html
echo "<p>Serving from: $(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)</p>" >> /var/www/html/index.html
'''
    },
    'security_rules': [
        {
            'IpProtocol': 'tcp',
            'FromPort': 80,
            'ToPort': 80,
            'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
        },
        {
            'IpProtocol': 'tcp',
            'FromPort': 443,
            'ToPort': 443,
            'IpRanges': [{'CidrIp': '0.0.0.0/0'}]
        }
    ]
}

# Create the infrastructure
print("\nCreating Local Zone infrastructure...")
infrastructure_results = local_zones_manager.create_local_zone_infrastructure(infrastructure_config)
print("Infrastructure creation results:")
print(json.dumps(infrastructure_results, indent=2))

# Analyze latency improvement
latency_analysis = local_zones_manager.analyze_latency_improvement(
    'us-east-1-bos-1a',
    'us-east-1'
)
print("\nLatency improvement analysis:")
print(json.dumps(latency_analysis, indent=2))
```

### Local Zones Networking

```bash
#!/bin/bash
# Local Zones networking setup script

LOCAL_ZONE="us-east-1-bos-1a"
VPC_ID="vpc-12345678"
PARENT_REGION="us-east-1"

echo "Setting up Local Zone networking for $LOCAL_ZONE"

# Create subnet in Local Zone
echo "Creating subnet in Local Zone..."
SUBNET_ID=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.100.0/24 \
    --availability-zone $LOCAL_ZONE \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=LocalZone-Subnet},{Key=LocalZone,Value='$LOCAL_ZONE'}]' \
    --query 'Subnet.SubnetId' \
    --output text)

echo "Created subnet: $SUBNET_ID"

# Get the main route table for the VPC
ROUTE_TABLE_ID=$(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$VPC_ID" "Name=association.main,Values=true" \
    --query 'RouteTables[0].RouteTableId' \
    --output text)

# Associate subnet with route table
echo "Associating subnet with route table..."
aws ec2 associate-route-table \
    --subnet-id $SUBNET_ID \
    --route-table-id $ROUTE_TABLE_ID

# Create security group for Local Zone resources
echo "Creating security group..."
SG_ID=$(aws ec2 create-security-group \
    --group-name LocalZone-SecurityGroup \
    --description "Security group for Local Zone resources" \
    --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=security-group,Tags=[{Key=Name,Value=LocalZone-SG},{Key=LocalZone,Value='$LOCAL_ZONE'}]' \
    --query 'GroupId' \
    --output text)

echo "Created security group: $SG_ID"

# Configure security group rules
echo "Configuring security group rules..."
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 443 \
    --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 22 \
    --cidr 10.0.0.0/16

# Launch instances in Local Zone
echo "Launching instances in Local Zone..."
INSTANCE_IDS=$(aws ec2 run-instances \
    --image-id ami-0c55b159cbfafe1d0 \
    --count 2 \
    --instance-type t3.medium \
    --subnet-id $SUBNET_ID \
    --security-group-ids $SG_ID \
    --placement AvailabilityZone=$LOCAL_ZONE \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=LocalZone-Instance},{Key=LocalZone,Value='$LOCAL_ZONE'}]' \
    --user-data file://user-data.sh \
    --query 'Instances[].InstanceId' \
    --output text)

echo "Launched instances: $INSTANCE_IDS"

# Create Application Load Balancer
echo "Creating Application Load Balancer..."
ALB_ARN=$(aws elbv2 create-load-balancer \
    --name LocalZone-ALB \
    --subnets $SUBNET_ID \
    --security-groups $SG_ID \
    --scheme internet-facing \
    --type application \
    --ip-address-type ipv4 \
    --tags Key=Name,Value=LocalZone-ALB Key=LocalZone,Value=$LOCAL_ZONE \
    --query 'LoadBalancers[0].LoadBalancerArn' \
    --output text)

echo "Created ALB: $ALB_ARN"

# Create target group
echo "Creating target group..."
TG_ARN=$(aws elbv2 create-target-group \
    --name LocalZone-TG \
    --protocol HTTP \
    --port 80 \
    --vpc-id $VPC_ID \
    --target-type instance \
    --health-check-path /health \
    --health-check-interval-seconds 30 \
    --health-check-timeout-seconds 5 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 2 \
    --tags Key=Name,Value=LocalZone-TG Key=LocalZone,Value=$LOCAL_ZONE \
    --query 'TargetGroups[0].TargetGroupArn' \
    --output text)

echo "Created target group: $TG_ARN"

# Register instances with target group
echo "Registering instances with target group..."
for INSTANCE_ID in $INSTANCE_IDS; do
    aws elbv2 register-targets \
        --target-group-arn $TG_ARN \
        --targets Id=$INSTANCE_ID,Port=80
done

# Create listener
echo "Creating ALB listener..."
aws elbv2 create-listener \
    --load-balancer-arn $ALB_ARN \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=$TG_ARN

echo "Local Zone networking setup completed successfully!"
echo "Resources created:"
echo "  Subnet ID: $SUBNET_ID"
echo "  Security Group ID: $SG_ID"
echo "  Instance IDs: $INSTANCE_IDS"
echo "  ALB ARN: $ALB_ARN"
echo "  Target Group ARN: $TG_ARN"

# Create user-data script for instances
cat > user-data.sh << 'EOF'
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

# Create a simple health check endpoint
echo "OK" > /var/www/html/health

# Create main page with zone information
cat > /var/www/html/index.html << 'HTML'
<!DOCTYPE html>
<html>
<head>
    <title>Local Zone Edge Server</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        .zone-info { background: #f0f0f0; padding: 20px; border-radius: 5px; }
        .metrics { background: #e8f4f8; padding: 15px; margin-top: 20px; border-radius: 5px; }
    </style>
</head>
<body>
    <h1>Edge Computing with AWS Local Zones</h1>
    <div class="zone-info">
        <h2>Server Information</h2>
        <p><strong>Availability Zone:</strong> <span id="az"></span></p>
        <p><strong>Instance ID:</strong> <span id="instance-id"></span></p>
        <p><strong>Local IP:</strong> <span id="local-ip"></span></p>
        <p><strong>Server Time:</strong> <span id="server-time"></span></p>
    </div>
    <div class="metrics">
        <h2>Edge Performance</h2>
        <p>This server is running in an AWS Local Zone, providing low-latency access to users in the local metro area.</p>
        <p><strong>Expected Latency:</strong> < 10ms to local users</p>
        <p><strong>Use Cases:</strong> Real-time gaming, live streaming, IoT applications</p>
    </div>
    
    <script>
        // Fetch instance metadata
        fetch('http://169.254.169.254/latest/meta-data/placement/availability-zone')
            .then(response => response.text())
            .then(data => document.getElementById('az').textContent = data);
            
        fetch('http://169.254.169.254/latest/meta-data/instance-id')
            .then(response => response.text())
            .then(data => document.getElementById('instance-id').textContent = data);
            
        fetch('http://169.254.169.254/latest/meta-data/local-ipv4')
            .then(response => response.text())
            .then(data => document.getElementById('local-ip').textContent = data);
            
        document.getElementById('server-time').textContent = new Date().toLocaleString();
    </script>
</body>
</html>
HTML

# Start a simple performance monitoring service
cat > /opt/monitor.sh << 'SCRIPT'
#!/bin/bash
while true; do
    echo "$(date): Server running in $(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)" >> /var/log/edge-monitor.log
    sleep 60
done
SCRIPT

chmod +x /opt/monitor.sh
nohup /opt/monitor.sh &
EOF

chmod +x user-data.sh
```

## AWS Wavelength for 5G Applications {#wavelength}

### Wavelength Overview

AWS Wavelength embeds AWS compute and storage services within 5G networks, providing mobile edge computing infrastructure for ultra-low latency applications.

```
AWS Wavelength Architecture:

                    ┌─────────────────────────────────┐
                    │         AWS Region              │
                    │        (us-east-1)              │
                    │                                 │
                    │  ┌─────────────────────────┐   │
                    │  │    VPC in Region        │   │
                    │  │                         │   │
                    │  │  ┌─────────────────┐   │   │
                    │  │  │   Subnets       │   │   │
                    │  │  │   Services      │   │   │
                    │  │  │   Databases     │   │   │
                    │  │  └─────────────────┘   │   │
                    │  └─────────────────────────┘   │
                    └─────────────────┬───────────────┘
                                      │
                               Carrier Gateway
                                      │
                    ┌─────────────────▼───────────────┐
                    │      Wavelength Zone            │
                    │   (us-east-1-wl1-bos-wlz-1)    │
                    │                                 │
                    │  ┌─────────────────────────┐   │
                    │  │   AWS Services at Edge  │   │
                    │  │                         │   │
                    │  │  • EC2 Instances        │   │
                    │  │  • EBS Volumes          │   │
                    │  │  • VPC Subnets          │   │
                    │  │  • ECS/EKS              │   │
                    │  └─────────────────────────┘   │
                    │                                 │
                    │     5G Carrier Network          │
                    └─────────────────┬───────────────┘
                                      │
                                 < 5ms latency
                                      │
                    ┌─────────────────▼───────────────┐
                    │      5G Devices                 │
                    │                                 │
                    │  • Mobile Applications          │
                    │  • IoT Devices                  │
                    │  • Autonomous Vehicles          │
                    │  • AR/VR Applications           │
                    └─────────────────────────────────┘
```

### Wavelength Implementation

```python
import boto3
import json
from typing import Dict, List, Optional

class WavelengthManager:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
        self.ecs = boto3.client('ecs')
        
    def discover_wavelength_zones(self) -> Dict:
        """Discover available Wavelength Zones"""
        
        try:
            response = self.ec2.describe_availability_zones(
                Filters=[
                    {'Name': 'zone-type', 'Values': ['wavelength-zone']}
                ]
            )
            
            wavelength_zones = {}
            
            for zone in response['AvailabilityZones']:
                zone_name = zone['ZoneName']
                
                # Parse Wavelength zone naming convention
                # Format: us-east-1-wl1-bos-wlz-1
                parts = zone_name.split('-')
                if len(parts) >= 6 and 'wl' in parts[3]:
                    carrier_info = parts[4]  # Carrier/location code
                    zone_suffix = parts[6] if len(parts) > 6 else '1'
                else:
                    carrier_info = 'unknown'
                    zone_suffix = '1'
                
                wavelength_zones[zone_name] = {
                    'zone_id': zone['ZoneId'],
                    'zone_name': zone_name,
                    'state': zone['State'],
                    'parent_region': zone['RegionName'],
                    'carrier_location': carrier_info,
                    'zone_suffix': zone_suffix,
                    'network_border_group': zone.get('NetworkBorderGroup', ''),
                    'supported_services': self._get_wavelength_services()
                }
            
            return {
                'total_wavelength_zones': len(wavelength_zones),
                'wavelength_zones': wavelength_zones,
                'carrier_locations': list(set(zone['carrier_location'] for zone in wavelength_zones.values()))
            }
            
        except Exception as e:
            return {'error': str(e)}
    
    def _get_wavelength_services(self) -> List[str]:
        """Get supported services in Wavelength zones"""
        
        return [
            'EC2 Instances',
            'EBS Volumes',
            'VPC and Subnets',
            'Application Load Balancer',
            'ECS (Fargate)',
            'EKS',
            'Systems Manager',
            'CloudWatch',
            'Auto Scaling'
        ]
    
    def create_wavelength_application(self, 
                                    config: Dict) -> Dict:
        """Create application infrastructure in Wavelength zone"""
        
        results = {
            'wavelength_zone': config['wavelength_zone'],
            'vpc_id': None,
            'subnet_id': None,
            'security_group_id': None,
            'carrier_gateway_id': None,
            'instance_ids': [],
            'application_endpoints': [],
            'created_resources': []
        }
        
        try:
            # Create VPC if not provided
            if 'vpc_id' not in config:
                vpc_response = self.ec2.create_vpc(
                    CidrBlock=config.get('vpc_cidr', '10.0.0.0/16'),
                    TagSpecifications=[{
                        'ResourceType': 'vpc',
                        'Tags': [
                            {'Key': 'Name', 'Value': f"Wavelength-VPC-{config['wavelength_zone']}"},
                            {'Key': 'WavelengthZone', 'Value': config['wavelength_zone']},
                            {'Key': 'Purpose', 'Value': '5G Edge Computing'}
                        ]
                    }]
                )
                results['vpc_id'] = vpc_response['Vpc']['VpcId']
                results['created_resources'].append(f"VPC: {results['vpc_id']}")
            else:
                results['vpc_id'] = config['vpc_id']
            
            # Create Carrier Gateway for 5G connectivity
            cagw_response = self.ec2.create_carrier_gateway(
                VpcId=results['vpc_id'],
                TagSpecifications=[{
                    'ResourceType': 'carrier-gateway',
                    'Tags': [
                        {'Key': 'Name', 'Value': f"Wavelength-CAGW-{config['wavelength_zone']}"},
                        {'Key': 'WavelengthZone', 'Value': config['wavelength_zone']}
                    ]
                }]
            )
            results['carrier_gateway_id'] = cagw_response['CarrierGateway']['CarrierGatewayId']
            results['created_resources'].append(f"Carrier Gateway: {results['carrier_gateway_id']}")
            
            # Create subnet in Wavelength zone
            subnet_response = self.ec2.create_subnet(
                VpcId=results['vpc_id'],
                CidrBlock=config.get('subnet_cidr', '10.0.1.0/24'),
                AvailabilityZone=config['wavelength_zone'],
                TagSpecifications=[{
                    'ResourceType': 'subnet',
                    'Tags': [
                        {'Key': 'Name', 'Value': f"Wavelength-Subnet-{config['wavelength_zone']}"},
                        {'Key': 'WavelengthZone', 'Value': config['wavelength_zone']},
                        {'Key': 'Type', 'Value': '5G Edge'}
                    ]
                }]
            )
            results['subnet_id'] = subnet_response['Subnet']['SubnetId']
            results['created_resources'].append(f"Subnet: {results['subnet_id']}")
            
            # Create route table for Wavelength subnet
            route_table = self._create_wavelength_route_table(
                results['vpc_id'],
                results['carrier_gateway_id'],
                results['subnet_id'],
                config['wavelength_zone']
            )
            results['created_resources'].append(f"Route Table: {route_table}")
            
            # Create security group
            sg_response = self.ec2.create_security_group(
                GroupName=f"Wavelength-SG-{config['wavelength_zone'].replace('.', '-')}",
                Description=f"Security group for Wavelength zone {config['wavelength_zone']}",
                VpcId=results['vpc_id'],
                TagSpecifications=[{
                    'ResourceType': 'security-group',
                    'Tags': [
                        {'Key': 'Name', 'Value': f"Wavelength-SG-{config['wavelength_zone']}"},
                        {'Key': 'WavelengthZone', 'Value': config['wavelength_zone']}
                    ]
                }]
            )
            results['security_group_id'] = sg_response['GroupId']
            results['created_resources'].append(f"Security Group: {results['security_group_id']}")
            
            # Configure security group for 5G applications
            self._configure_5g_security_rules(results['security_group_id'])
            
            # Deploy application based on type
            app_type = config.get('application_type', 'web')
            
            if app_type == 'gaming':
                app_results = self._deploy_gaming_application(
                    results['subnet_id'],
                    results['security_group_id'],
                    config
                )
            elif app_type == 'iot':
                app_results = self._deploy_iot_application(
                    results['subnet_id'],
                    results['security_group_id'],
                    config
                )
            elif app_type == 'ar_vr':
                app_results = self._deploy_ar_vr_application(
                    results['subnet_id'],
                    results['security_group_id'],
                    config
                )
            else:
                app_results = self._deploy_web_application(
                    results['subnet_id'],
                    results['security_group_id'],
                    config
                )
            
            results.update(app_results)
            
            return results
            
        except Exception as e:
            return {'error': str(e), 'partial_results': results}
    
    def _create_wavelength_route_table(self,
                                     vpc_id: str,
                                     carrier_gateway_id: str,
                                     subnet_id: str,
                                     wavelength_zone: str) -> str:
        """Create route table for Wavelength subnet with carrier gateway"""
        
        # Create route table
        rt_response = self.ec2.create_route_table(
            VpcId=vpc_id,
            TagSpecifications=[{
                'ResourceType': 'route-table',
                'Tags': [
                    {'Key': 'Name', 'Value': f"Wavelength-RT-{wavelength_zone}"},
                    {'Key': 'WavelengthZone', 'Value': wavelength_zone}
                ]
            }]
        )
        route_table_id = rt_response['RouteTable']['RouteTableId']
        
        # Add route to carrier gateway for 5G traffic
        self.ec2.create_route(
            RouteTableId=route_table_id,
            DestinationCidrBlock='0.0.0.0/0',
            CarrierGatewayId=carrier_gateway_id
        )
        
        # Associate route table with subnet
        self.ec2.associate_route_table(
            RouteTableId=route_table_id,
            SubnetId=subnet_id
        )
        
        return route_table_id
    
    def _configure_5g_security_rules(self, security_group_id: str):
        """Configure security group rules for 5G applications"""
        
        # Common 5G application ports
        rules = [
            # HTTP/HTTPS for web applications
            {'IpProtocol': 'tcp', 'FromPort': 80, 'ToPort': 80, 'IpRanges': [{'CidrIp': '0.0.0.0/0'}]},
            {'IpProtocol': 'tcp', 'FromPort': 443, 'ToPort': 443, 'IpRanges': [{'CidrIp': '0.0.0.0/0'}]},
            
            # Gaming protocols
            {'IpProtocol': 'udp', 'FromPort': 7777, 'ToPort': 7777, 'IpRanges': [{'CidrIp': '0.0.0.0/0'}]},
            {'IpProtocol': 'tcp', 'FromPort': 27015, 'ToPort': 27015, 'IpRanges': [{'CidrIp': '0.0.0.0/0'}]},
            
            # IoT/MQTT
            {'IpProtocol': 'tcp', 'FromPort': 1883, 'ToPort': 1883, 'IpRanges': [{'CidrIp': '0.0.0.0/0'}]},
            {'IpProtocol': 'tcp', 'FromPort': 8883, 'ToPort': 8883, 'IpRanges': [{'CidrIp': '0.0.0.0/0'}]},
            
            # WebRTC for AR/VR
            {'IpProtocol': 'udp', 'FromPort': 3478, 'ToPort': 3478, 'IpRanges': [{'CidrIp': '0.0.0.0/0'}]},
            {'IpProtocol': 'tcp', 'FromPort': 3478, 'ToPort': 3478, 'IpRanges': [{'CidrIp': '0.0.0.0/0'}]}
        ]
        
        try:
            self.ec2.authorize_security_group_ingress(
                GroupId=security_group_id,
                IpPermissions=rules
            )
        except Exception as e:
            print(f"Warning: Some security group rules may already exist: {e}")
    
    def _deploy_gaming_application(self,
                                 subnet_id: str,
                                 security_group_id: str,
                                 config: Dict) -> Dict:
        """Deploy gaming application optimized for 5G"""
        
        user_data = '''#!/bin/bash
yum update -y
yum install -y docker
service docker start
usermod -a -G docker ec2-user

# Install game server (example)
docker run -d --name game-server \\
  -p 7777:7777/udp \\
  -p 27015:27015/tcp \\
  -e SERVER_NAME="5G Edge Game Server" \\
  -e MAX_PLAYERS=64 \\
  game-server:latest

# Setup monitoring
echo "*/5 * * * * /opt/monitor-latency.sh" | crontab -
'''
        
        return self._launch_wavelength_instances(
            subnet_id, security_group_id, config, user_data, 'Gaming'
        )
    
    def _deploy_iot_application(self,
                              subnet_id: str,
                              security_group_id: str,
                              config: Dict) -> Dict:
        """Deploy IoT gateway application"""
        
        user_data = '''#!/bin/bash
yum update -y
yum install -y docker mosquitto mosquitto-clients
service docker start

# Start MQTT broker
systemctl start mosquitto
systemctl enable mosquitto

# Deploy IoT gateway service
docker run -d --name iot-gateway \\
  -p 1883:1883 \\
  -p 8883:8883 \\
  -p 8080:8080 \\
  -e MQTT_BROKER=localhost \\
  -e EDGE_LOCATION="5G" \\
  iot-gateway:latest
'''
        
        return self._launch_wavelength_instances(
            subnet_id, security_group_id, config, user_data, 'IoT Gateway'
        )
    
    def _deploy_ar_vr_application(self,
                                subnet_id: str,
                                security_group_id: str,
                                config: Dict) -> Dict:
        """Deploy AR/VR application with WebRTC support"""
        
        user_data = '''#!/bin/bash
yum update -y
yum install -y docker
service docker start

# Deploy WebRTC signaling server
docker run -d --name webrtc-server \\
  -p 3478:3478/udp \\
  -p 3478:3478/tcp \\
  -p 8080:8080 \\
  -e TURN_SECRET=your-secret-key \\
  -e REALM=5g-edge \\
  webrtc-server:latest

# Deploy AR/VR content server
docker run -d --name arvr-content \\
  -p 80:80 \\
  -p 443:443 \\
  -v /opt/content:/usr/share/nginx/html \\
  nginx:latest
'''
        
        return self._launch_wavelength_instances(
            subnet_id, security_group_id, config, user_data, 'AR/VR'
        )
    
    def _deploy_web_application(self,
                              subnet_id: str,
                              security_group_id: str,
                              config: Dict) -> Dict:
        """Deploy standard web application"""
        
        user_data = '''#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

cat > /var/www/html/index.html << 'HTML'
<!DOCTYPE html>
<html>
<head>
    <title>5G Edge Application</title>
    <style>
        body { font-family: Arial; margin: 40px; background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); color: white; }
        .container { background: rgba(255,255,255,0.1); padding: 30px; border-radius: 10px; }
        .metrics { background: rgba(255,255,255,0.2); padding: 20px; margin: 20px 0; border-radius: 5px; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🚀 5G Edge Computing with AWS Wavelength</h1>
        <div class="metrics">
            <h2>Performance Metrics</h2>
            <p><strong>Expected Latency:</strong> < 5ms</p>
            <p><strong>Network Type:</strong> 5G Ultra-Low Latency</p>
            <p><strong>Edge Location:</strong> Carrier Network</p>
            <p><strong>Zone:</strong> <span id="zone"></span></p>
        </div>
        <div class="metrics">
            <h2>5G Use Cases</h2>
            <ul>
                <li>Real-time Gaming</li>
                <li>Augmented Reality</li>
                <li>Autonomous Vehicles</li>
                <li>Industrial IoT Control</li>
                <li>Live Video Processing</li>
            </ul>
        </div>
    </div>
    
    <script>
        fetch('http://169.254.169.254/latest/meta-data/placement/availability-zone')
            .then(response => response.text())
            .then(data => document.getElementById('zone').textContent = data);
    </script>
</body>
</html>
HTML
'''
        
        return self._launch_wavelength_instances(
            subnet_id, security_group_id, config, user_data, 'Web Application'
        )
    
    def _launch_wavelength_instances(self,
                                   subnet_id: str,
                                   security_group_id: str,
                                   config: Dict,
                                   user_data: str,
                                   purpose: str) -> Dict:
        """Launch EC2 instances in Wavelength zone"""
        
        try:
            response = self.ec2.run_instances(
                ImageId=config.get('ami_id', 'ami-0c55b159cbfafe1d0'),
                MinCount=config.get('min_instances', 1),
                MaxCount=config.get('max_instances', 2),
                InstanceType=config.get('instance_type', 't3.medium'),
                SubnetId=subnet_id,
                SecurityGroupIds=[security_group_id],
                Placement={'AvailabilityZone': config['wavelength_zone']},
                UserData=user_data,
                TagSpecifications=[{
                    'ResourceType': 'instance',
                    'Tags': [
                        {'Key': 'Name', 'Value': f"Wavelength-{purpose}-{config['wavelength_zone']}"},
                        {'Key': 'WavelengthZone', 'Value': config['wavelength_zone']},
                        {'Key': 'Purpose', 'Value': purpose},
                        {'Key': 'Network', 'Value': '5G Edge'}
                    ]
                }]
            )
            
            instance_ids = [instance['InstanceId'] for instance in response['Instances']]
            
            return {
                'instance_ids': instance_ids,
                'application_type': config.get('application_type', 'web'),
                'purpose': purpose
            }
            
        except Exception as e:
            print(f"Failed to launch Wavelength instances: {e}")
            return {'instance_ids': [], 'error': str(e)}

# Usage example
wavelength_manager = WavelengthManager()

# Discover Wavelength zones
wavelength_info = wavelength_manager.discover_wavelength_zones()
print("Available Wavelength Zones:")
print(json.dumps(wavelength_info, indent=2))

# Deploy gaming application in Wavelength
gaming_config = {
    'wavelength_zone': 'us-east-1-wl1-bos-wlz-1',  # Boston Wavelength Zone
    'application_type': 'gaming',
    'vpc_cidr': '10.0.0.0/16',
    'subnet_cidr': '10.0.1.0/24',
    'ami_id': 'ami-0c55b159cbfafe1d0',
    'instance_type': 'r5.large',  # Gaming requires more memory
    'min_instances': 2,
    'max_instances': 4
}

print("\nDeploying gaming application in Wavelength zone...")
gaming_results = wavelength_manager.create_wavelength_application(gaming_config)
print("Gaming application deployment results:")
print(json.dumps(gaming_results, indent=2))

# Deploy IoT gateway application
iot_config = {
    'wavelength_zone': 'us-east-1-wl1-bos-wlz-1',
    'application_type': 'iot',
    'vpc_cidr': '10.1.0.0/16',
    'subnet_cidr': '10.1.1.0/24',
    'ami_id': 'ami-0c55b159cbfafe1d0',
    'instance_type': 'm5.large',
    'min_instances': 1,
    'max_instances': 2
}

print("\nDeploying IoT gateway in Wavelength zone...")
iot_results = wavelength_manager.create_wavelength_application(iot_config)
print("IoT gateway deployment results:")
print(json.dumps(iot_results, indent=2))
```

This comprehensive guide provides detailed implementations for AWS Local Zones and Wavelength edge computing services, including practical examples for different application types, networking setup, and deployment automation. The code can be adapted to specific organizational requirements while maintaining best practices for edge computing architectures.