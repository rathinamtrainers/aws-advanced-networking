# Topic 56: AWS Global Accelerator - Use Cases and Implementation

## Table of Contents
1. [Introduction to AWS Global Accelerator](#introduction)
2. [Core Concepts and Architecture](#core-concepts)
3. [Anycast IP Addresses](#anycast-ips)
4. [Health Checks and Monitoring](#health-checks)
5. [Traffic Management Features](#traffic-management)
6. [Use Cases and Scenarios](#use-cases)
7. [Implementation Guide](#implementation)
8. [Performance Optimization](#performance)
9. [Cost Analysis](#cost-analysis)
10. [Troubleshooting](#troubleshooting)
11. [Best Practices](#best-practices)
12. [Labs and Exercises](#labs)

## Introduction to AWS Global Accelerator {#introduction}

AWS Global Accelerator is a networking service that improves performance for your applications with local or global users by using the AWS global network infrastructure. It provides two static IP addresses that act as a fixed entry point to your application endpoints in one or more AWS Regions, directing traffic to optimal endpoints based on health, geography, and routing policies.

### Key Benefits

**Performance Improvement**
- Up to 60% performance improvement for applications
- Reduced latency through AWS backbone network
- Improved packet loss and jitter characteristics
- Faster time to first byte (TTFB)

**Reliability and Availability**
- Automatic failover between endpoints
- Health check monitoring
- Network-level redundancy
- DDoS protection at the edge

**Global Reach**
- 450+ points of presence worldwide
- Anycast IP addresses for global accessibility
- Regional traffic optimization
- Multi-region deployment support

### Service Architecture

```
Global Accelerator Architecture:

User Request → Edge Location → AWS Backbone → Regional Endpoint
     ↓              ↓              ↓              ↓
 Internet      Anycast IP    Optimized      Application
 Traffic        Address       Routing         Server

                    Global Accelerator Components:
                    ┌─────────────────────────────────┐
                    │        Edge Locations          │
                    │  ┌─────────┐  ┌─────────┐     │
                    │  │ Anycast │  │ Anycast │     │
                    │  │   IP1   │  │   IP2   │     │
                    │  └─────────┘  └─────────┘     │
                    └─────────────────────────────────┘
                                    │
                                    ▼
                    ┌─────────────────────────────────┐
                    │        AWS Backbone             │
                    │      Network Routing            │
                    └─────────────────────────────────┘
                                    │
                                    ▼
                    ┌─────────────────────────────────┐
                    │     Regional Endpoints          │
                    │  ┌─────────┐  ┌─────────┐     │
                    │  │   ALB   │  │   NLB   │     │
                    │  │ us-east │  │ eu-west │     │
                    │  └─────────┘  └─────────┘     │
                    └─────────────────────────────────┘
```

## Core Concepts and Architecture {#core-concepts}

### Global Accelerator Components

**Accelerator**
- Top-level resource that creates two static IP addresses
- Routes traffic to optimal endpoints
- Can be standard or custom routing accelerator

**Listener**
- Processes inbound connections from clients
- Supports TCP and UDP protocols
- Configured with port ranges and protocols

**Endpoint Groups**
- Collection of endpoints in a specific AWS Region
- Traffic distribution policies
- Health check configuration
- Failover capabilities

**Endpoints**
- Target resources for traffic routing
- Application Load Balancers (ALB)
- Network Load Balancers (NLB)
- Elastic IP addresses
- EC2 instances

### Traffic Flow Diagram

```
Traffic Flow Through Global Accelerator:

Client Request (1)
     │
     ▼
Edge Location (2) ──── Anycast IP Resolution
     │
     ▼
AWS Backbone (3) ──── Optimized Routing
     │
     ▼
Region Selection (4) ─ Health Check + Policy
     │
     ▼
Endpoint Group (5) ─── Traffic Distribution
     │
     ▼
Target Endpoint (6) ── Application Processing

Flow Details:
1. Client initiates connection to anycast IP
2. Nearest edge location receives traffic
3. Traffic routed over AWS backbone network
4. Optimal region selected based on policies
5. Endpoint group distributes traffic
6. Target endpoint processes request
```

## Anycast IP Addresses {#anycast-ips}

### Understanding Anycast

Anycast IP addresses are globally advertised from multiple locations, allowing clients to connect to the nearest point of presence. Global Accelerator provides two static anycast IP addresses per accelerator.

### Anycast Implementation

```bash
# View accelerator configuration
aws globalaccelerator describe-accelerator \
  --accelerator-arn arn:aws:globalaccelerator::account:accelerator/abc123

# Example response structure
{
  "Accelerator": {
    "AcceleratorArn": "arn:aws:globalaccelerator::account:accelerator/abc123",
    "Name": "MyGlobalAccelerator",
    "IpAddressType": "IPV4",
    "Enabled": true,
    "IpSets": [
      {
        "IpFamily": "IPv4",
        "IpAddresses": [
          "75.2.76.26",
          "99.83.89.81"
        ]
      }
    ],
    "DnsName": "a1234567890abcdef.awsglobalaccelerator.com",
    "Status": "IN_SERVICE"
  }
}
```

### DNS Configuration

```bash
# Create DNS records pointing to anycast IPs
# A Record Configuration
api.example.com.     300    IN    A    75.2.76.26
api.example.com.     300    IN    A    99.83.89.81

# Or use CNAME to accelerator DNS name
api.example.com.     300    IN    CNAME    a1234567890abcdef.awsglobalaccelerator.com
```

### Anycast Routing Behavior

```
Anycast IP Routing Patterns:

Global Advertisement:
┌──────────────────────────────────────────────────────────────┐
│                     Internet Routing                         │
│                                                              │
│  Client A (Asia)     Client B (US)     Client C (Europe)    │
│       │                   │                   │             │
│       ▼                   ▼                   ▼             │
│  Edge Location       Edge Location      Edge Location       │
│   (Singapore)         (Virginia)         (Dublin)          │
│       │                   │                   │             │
│       └───────────────────┼───────────────────┘             │
│                           │                                 │
│                    Same Anycast IPs                         │
│                   75.2.76.26, 99.83.89.81                  │
└──────────────────────────────────────────────────────────────┘

Automatic Edge Selection:
- BGP routing determines nearest edge
- No DNS resolution delays
- Consistent IP addresses globally
- Network-level load distribution
```

## Health Checks and Monitoring {#health-checks}

### Health Check Configuration

Global Accelerator performs continuous health checks on endpoints to ensure traffic is only directed to healthy resources.

```bash
# Configure endpoint health checks
aws globalaccelerator create-endpoint-group \
  --listener-arn arn:aws:globalaccelerator::account:listener/abc123 \
  --endpoint-group-region us-east-1 \
  --health-check-interval-seconds 10 \
  --health-check-path "/health" \
  --health-check-protocol TCP \
  --health-check-port 80 \
  --threshold-count 3 \
  --endpoint-configurations '[
    {
      "EndpointId": "arn:aws:elasticloadbalancing:us-east-1:account:loadbalancer/app/my-alb/123",
      "Weight": 100,
      "ClientIPPreservationEnabled": true
    }
  ]'
```

### Health Check Parameters

```yaml
# Health Check Configuration Template
HealthCheckConfiguration:
  Interval: 10-30 seconds
  Protocol: TCP | HTTP | HTTPS
  Port: 1-65535
  Path: "/health" (HTTP/HTTPS only)
  ThresholdCount: 1-10 failures
  
  # Health Check Behavior
  HealthyThreshold: 3 consecutive successes
  UnhealthyThreshold: 3 consecutive failures
  Timeout: 5 seconds per check
  
  # Endpoint Status Transitions
  States:
    - INITIAL: New endpoint, checking health
    - HEALTHY: Passing health checks
    - UNHEALTHY: Failing health checks
    - UNAVAILABLE: Cannot reach endpoint
```

### Monitoring and Metrics

```bash
# CloudWatch metrics for Global Accelerator
aws cloudwatch get-metric-statistics \
  --namespace AWS/GlobalAccelerator \
  --metric-name NewFlowCount_TCP \
  --dimensions Name=Accelerator,Value=abc123 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-01T01:00:00Z \
  --period 300 \
  --statistics Sum
```

### Key Metrics to Monitor

```
Global Accelerator CloudWatch Metrics:

Connection Metrics:
- NewFlowCount_TCP: New TCP connections
- NewFlowCount_UDP: New UDP connections
- ProcessedBytesIn: Inbound traffic volume
- ProcessedBytesOut: Outbound traffic volume

Endpoint Health:
- OriginLatency: Backend response time
- HealthCheckStatus: Endpoint health state
- UnhealthyRoutingFailovers: Failover events

Performance Metrics:
- PacketDropCount: Dropped packets
- ConnectTimeClientToEdge: Edge connection time
- ConnectTimeEdgeToOrigin: Backend connection time
```

## Traffic Management Features {#traffic-management}

### Traffic Dial

Traffic dial allows you to control the percentage of traffic directed to endpoint groups, enabling blue-green deployments and gradual rollouts.

```bash
# Configure traffic dial for gradual deployment
aws globalaccelerator update-endpoint-group \
  --endpoint-group-arn arn:aws:globalaccelerator::account:endpointgroup/def456 \
  --traffic-dial-percentage 25

# Gradually increase traffic to new version
# Week 1: 25% to new version
# Week 2: 50% to new version  
# Week 3: 75% to new version
# Week 4: 100% to new version
```

### Endpoint Weights

Distribute traffic proportionally across endpoints within an endpoint group.

```bash
# Configure weighted routing
aws globalaccelerator update-endpoint-group \
  --endpoint-group-arn arn:aws:globalaccelerator::account:endpointgroup/def456 \
  --endpoint-configurations '[
    {
      "EndpointId": "arn:aws:elasticloadbalancing:us-east-1:account:loadbalancer/app/alb-v1/123",
      "Weight": 80,
      "ClientIPPreservationEnabled": true
    },
    {
      "EndpointId": "arn:aws:elasticloadbalancing:us-east-1:account:loadbalancer/app/alb-v2/456", 
      "Weight": 20,
      "ClientIPPreservationEnabled": true
    }
  ]'
```

### Traffic Distribution Example

```
Traffic Distribution Scenarios:

Scenario 1: Blue-Green Deployment
┌─────────────────────────────────────────────────────────────┐
│                    Endpoint Group                           │
│                                                             │
│  Traffic Dial: 100%                                        │
│                                                             │
│  Blue Environment    │    Green Environment                │
│  Weight: 100         │    Weight: 0                        │
│  ┌─────────────┐    │    ┌─────────────┐                 │
│  │   ALB-v1    │    │    │   ALB-v2    │                 │
│  │  (Current)  │    │    │    (New)    │                 │
│  └─────────────┘    │    └─────────────┘                 │
│                                                             │
│  Stage 1: Test new version with 0% traffic                 │
│  Stage 2: Switch to 100% green, 0% blue                   │
└─────────────────────────────────────────────────────────────┘

Scenario 2: Canary Deployment
┌─────────────────────────────────────────────────────────────┐
│                    Endpoint Group                           │
│                                                             │
│  Traffic Dial: 100%                                        │
│                                                             │
│  Production          │    Canary                           │
│  Weight: 90          │    Weight: 10                       │
│  ┌─────────────┐    │    ┌─────────────┐                 │
│  │   ALB-v1    │    │    │   ALB-v2    │                 │
│  │  (Stable)   │    │    │  (Testing)  │                 │
│  └─────────────┘    │    └─────────────┘                 │
│                                                             │
│  Gradual increase: 10% → 25% → 50% → 100%                 │
└─────────────────────────────────────────────────────────────┘

Scenario 3: Multi-Region Failover
┌─────────────────────────────────────────────────────────────┐
│  Primary Region (us-east-1)    │  Secondary Region (us-west-2) │
│  Traffic Dial: 100%            │  Traffic Dial: 0%             │
│                                │                                │
│  ┌─────────────────┐          │  ┌─────────────────┐          │
│  │ Endpoint Group  │          │  │ Endpoint Group  │          │
│  │   (Active)      │          │  │   (Standby)     │          │
│  └─────────────────┘          │  └─────────────────┘          │
│                                │                                │
│  Failover: Set primary to 0%, secondary to 100%              │
└─────────────────────────────────────────────────────────────┘
```

## Use Cases and Scenarios {#use-cases}

### Gaming Applications

Global Accelerator is ideal for real-time gaming applications that require low latency and consistent performance.

```bash
# Gaming application configuration
aws globalaccelerator create-accelerator \
  --name "GameServerAccelerator" \
  --ip-address-type IPV4 \
  --enabled

# Create UDP listener for game traffic
aws globalaccelerator create-listener \
  --accelerator-arn arn:aws:globalaccelerator::account:accelerator/game123 \
  --protocol UDP \
  --port-ranges '[{"FromPort": 7777, "ToPort": 7777}]'
```

### Financial Trading Platforms

High-frequency trading requires minimal latency and maximum reliability.

```bash
# Trading platform configuration
aws globalaccelerator create-accelerator \
  --name "TradingPlatform" \
  --ip-address-type IPV4 \
  --enabled

# TCP listener for trading data
aws globalaccelerator create-listener \
  --accelerator-arn arn:aws:globalaccelerator::account:accelerator/trade123 \
  --protocol TCP \
  --port-ranges '[{"FromPort": 443, "ToPort": 443}]'
```

### Video Streaming Services

Improve streaming quality and reduce buffering for global audiences.

```
Video Streaming Architecture:

Global Users
     │
     ▼
Global Accelerator (Anycast IPs)
     │
     ▼
AWS Backbone Network
     │
     ▼
Regional Endpoint Groups
     │
     ┌─────┴─────┐
     ▼           ▼
Origin Servers  CDN Integration
(Primary)       (CloudFront)
     │               │
     ▼               ▼
Video Content   Cached Content
Processing      Distribution
```

### IoT and Edge Computing

Optimize connectivity for distributed IoT devices and edge computing workloads.

```bash
# IoT device connectivity
aws globalaccelerator create-endpoint-group \
  --listener-arn arn:aws:globalaccelerator::account:listener/iot123 \
  --endpoint-group-region us-east-1 \
  --endpoint-configurations '[
    {
      "EndpointId": "arn:aws:elasticloadbalancing:us-east-1:account:loadbalancer/network/iot-nlb/123",
      "Weight": 100,
      "ClientIPPreservationEnabled": false
    }
  ]'
```

### Enterprise Applications

Improve performance for global enterprise applications and remote workforce.

```yaml
# Enterprise Application Deployment
ApplicationStack:
  GlobalAccelerator:
    Type: Standard
    IpAddressType: IPV4
    
  Listeners:
    WebListener:
      Protocol: TCP
      Ports: [80, 443]
      
    APIListener:
      Protocol: TCP
      Ports: [8080, 8443]
      
  EndpointGroups:
    Primary:
      Region: us-east-1
      TrafficDial: 100%
      Endpoints:
        - Type: ALB
          Weight: 100
          
    Secondary:
      Region: eu-west-1  
      TrafficDial: 0%
      Endpoints:
        - Type: ALB
          Weight: 100
```

## Implementation Guide {#implementation}

### Basic Setup

```bash
# Step 1: Create accelerator
aws globalaccelerator create-accelerator \
  --name "MyAccelerator" \
  --ip-address-type IPV4 \
  --enabled

# Step 2: Create listener
aws globalaccelerator create-listener \
  --accelerator-arn arn:aws:globalaccelerator::account:accelerator/abc123 \
  --protocol TCP \
  --port-ranges '[{"FromPort": 80, "ToPort": 80}, {"FromPort": 443, "ToPort": 443}]'

# Step 3: Create endpoint group
aws globalaccelerator create-endpoint-group \
  --listener-arn arn:aws:globalaccelerator::account:listener/def456 \
  --endpoint-group-region us-east-1 \
  --traffic-dial-percentage 100

# Step 4: Add endpoints
aws globalaccelerator update-endpoint-group \
  --endpoint-group-arn arn:aws:globalaccelerator::account:endpointgroup/ghi789 \
  --endpoint-configurations '[
    {
      "EndpointId": "arn:aws:elasticloadbalancing:us-east-1:account:loadbalancer/app/my-alb/123",
      "Weight": 100,
      "ClientIPPreservationEnabled": true
    }
  ]'
```

### CloudFormation Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Global Accelerator with multi-region endpoints'

Parameters:
  AcceleratorName:
    Type: String
    Default: 'MyGlobalAccelerator'
    
  PrimaryALBArn:
    Type: String
    Description: 'Primary ALB ARN in us-east-1'
    
  SecondaryALBArn:
    Type: String  
    Description: 'Secondary ALB ARN in eu-west-1'

Resources:
  GlobalAccelerator:
    Type: AWS::GlobalAccelerator::Accelerator
    Properties:
      Name: !Ref AcceleratorName
      IpAddressType: IPV4
      Enabled: true
      
  WebListener:
    Type: AWS::GlobalAccelerator::Listener
    Properties:
      AcceleratorArn: !Ref GlobalAccelerator
      Protocol: TCP
      PortRanges:
        - FromPort: 80
          ToPort: 80
        - FromPort: 443
          ToPort: 443
          
  PrimaryEndpointGroup:
    Type: AWS::GlobalAccelerator::EndpointGroup
    Properties:
      ListenerArn: !Ref WebListener
      EndpointGroupRegion: us-east-1
      TrafficDialPercentage: 100
      HealthCheckIntervalSeconds: 10
      HealthCheckPath: '/health'
      HealthCheckProtocol: HTTP
      HealthCheckPort: 80
      ThresholdCount: 3
      EndpointConfigurations:
        - EndpointId: !Ref PrimaryALBArn
          Weight: 100
          ClientIPPreservationEnabled: true
          
  SecondaryEndpointGroup:
    Type: AWS::GlobalAccelerator::EndpointGroup
    Properties:
      ListenerArn: !Ref WebListener
      EndpointGroupRegion: eu-west-1
      TrafficDialPercentage: 0
      HealthCheckIntervalSeconds: 10
      HealthCheckPath: '/health'
      HealthCheckProtocol: HTTP
      HealthCheckPort: 80
      ThresholdCount: 3
      EndpointConfigurations:
        - EndpointId: !Ref SecondaryALBArn
          Weight: 100
          ClientIPPreservationEnabled: true

Outputs:
  AcceleratorDnsName:
    Description: 'Global Accelerator DNS name'
    Value: !GetAtt GlobalAccelerator.DnsName
    
  AcceleratorIpAddresses:
    Description: 'Global Accelerator IP addresses'
    Value: !Join [', ', !GetAtt GlobalAccelerator.Ipv4Addresses]
```

### Terraform Configuration

```hcl
# Global Accelerator configuration
resource "aws_globalaccelerator_accelerator" "main" {
  name            = "MyGlobalAccelerator"
  ip_address_type = "IPV4"
  enabled         = true

  attributes {
    flow_logs_enabled   = true
    flow_logs_s3_bucket = aws_s3_bucket.flow_logs.bucket
    flow_logs_s3_prefix = "global-accelerator/"
  }

  tags = {
    Name = "MyGlobalAccelerator"
    Environment = "production"
  }
}

resource "aws_globalaccelerator_listener" "web" {
  accelerator_arn = aws_globalaccelerator_accelerator.main.id
  client_affinity = "SOURCE_IP"
  protocol        = "TCP"

  port_range {
    from_port = 80
    to_port   = 80
  }

  port_range {
    from_port = 443
    to_port   = 443
  }
}

resource "aws_globalaccelerator_endpoint_group" "primary" {
  listener_arn = aws_globalaccelerator_listener.web.id

  endpoint_group_region        = "us-east-1"
  traffic_dial_percentage      = 100
  health_check_interval_seconds = 10
  health_check_path            = "/health"
  health_check_protocol        = "HTTP"
  health_check_port            = 80
  threshold_count              = 3

  endpoint_configuration {
    endpoint_id                    = aws_lb.primary.arn
    weight                        = 100
    client_ip_preservation_enabled = true
  }
}

resource "aws_globalaccelerator_endpoint_group" "secondary" {
  listener_arn = aws_globalaccelerator_listener.web.id

  endpoint_group_region        = "eu-west-1"
  traffic_dial_percentage      = 0
  health_check_interval_seconds = 10
  health_check_path            = "/health"
  health_check_protocol        = "HTTP"
  health_check_port            = 80
  threshold_count              = 3

  endpoint_configuration {
    endpoint_id                    = aws_lb.secondary.arn
    weight                        = 100
    client_ip_preservation_enabled = true
  }
}

# Flow logs S3 bucket
resource "aws_s3_bucket" "flow_logs" {
  bucket = "my-global-accelerator-flow-logs"
}

resource "aws_s3_bucket_versioning" "flow_logs" {
  bucket = aws_s3_bucket.flow_logs.id
  versioning_configuration {
    status = "Enabled"
  }
}
```

## Performance Optimization {#performance}

### Client IP Preservation

```bash
# Enable client IP preservation
aws globalaccelerator update-endpoint-group \
  --endpoint-group-arn arn:aws:globalaccelerator::account:endpointgroup/abc123 \
  --endpoint-configurations '[
    {
      "EndpointId": "arn:aws:elasticloadbalancing:us-east-1:account:loadbalancer/network/my-nlb/123",
      "Weight": 100,
      "ClientIPPreservationEnabled": true
    }
  ]'
```

### Performance Testing

```bash
#!/bin/bash
# Performance comparison script

# Test direct connection
echo "Testing direct connection..."
time curl -w "@curl-format.txt" -o /dev/null -s "http://direct-endpoint.example.com/api/test"

# Test Global Accelerator
echo "Testing Global Accelerator..."
time curl -w "@curl-format.txt" -o /dev/null -s "http://75.2.76.26/api/test"

# curl-format.txt content:
#     time_namelookup:  %{time_namelookup}\n
#      time_connect:  %{time_connect}\n
#   time_appconnect:  %{time_appconnect}\n
#  time_pretransfer:  %{time_pretransfer}\n
#     time_redirect:  %{time_redirect}\n
#time_starttransfer:  %{time_starttransfer}\n
#                   ----------\n
#        time_total:  %{time_total}\n
```

### Optimization Strategies

```
Performance Optimization Techniques:

1. Endpoint Selection:
   ┌─────────────────────────────────────────┐
   │ ALB (Layer 7)      │ NLB (Layer 4)     │
   │ ─────────────      │ ─────────────     │
   │ + SSL termination  │ + Ultra-low latency│
   │ + Path-based routing│ + Static IP support│
   │ + HTTP/2 support   │ + Source IP preserve│
   │ - Additional latency│ - Limited features  │
   └─────────────────────────────────────────┘

2. Health Check Optimization:
   - Reduce check interval for faster failover
   - Use lightweight health check endpoints
   - Implement custom health logic
   - Monitor false positive rates

3. Traffic Distribution:
   - Use traffic dial for gradual deployments
   - Implement weighted routing for A/B testing
   - Configure proper endpoint weights
   - Monitor traffic patterns

4. Regional Placement:
   - Place endpoints close to users
   - Use multiple regions for redundancy
   - Consider data sovereignty requirements
   - Optimize for disaster recovery
```

## Cost Analysis {#cost-analysis}

### Pricing Components

```
Global Accelerator Pricing Structure:

Fixed Costs:
- Accelerator Fee: $0.025/hour per accelerator
- Data Transfer Out: Varies by region
  * Americas: $0.015/GB
  * Europe: $0.015/GB  
  * Asia Pacific: $0.025/GB
  * Middle East/Africa: $0.025/GB

Additional Costs:
- Endpoint resources (ALB, NLB, EC2)
- CloudWatch metrics and logs
- Cross-region data transfer
- Health check overhead

Monthly Cost Example:
Accelerator (730 hours): $18.25
Data Transfer (1TB): $15-25
Total Base Cost: $33-43/month
```

### Cost Optimization

```bash
# Monitor data transfer costs
aws cloudwatch get-metric-statistics \
  --namespace AWS/GlobalAccelerator \
  --metric-name ProcessedBytesOut \
  --dimensions Name=Accelerator,Value=abc123 \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-31T23:59:59Z \
  --period 86400 \
  --statistics Sum

# Calculate monthly data transfer
aws logs start-query \
  --log-group-name /aws/globalaccelerator/flowlogs \
  --start-time 1704067200 \
  --end-time 1706659200 \
  --query-string 'fields @timestamp, bytes | filter bytes > 0 | stats sum(bytes) by bin(5m)'
```

## Troubleshooting {#troubleshooting}

### Common Issues

**Connectivity Problems**
```bash
# Check accelerator status
aws globalaccelerator describe-accelerator \
  --accelerator-arn arn:aws:globalaccelerator::account:accelerator/abc123

# Verify listener configuration
aws globalaccelerator describe-listener \
  --listener-arn arn:aws:globalaccelerator::account:listener/def456

# Check endpoint health
aws globalaccelerator describe-endpoint-group \
  --endpoint-group-arn arn:aws:globalaccelerator::account:endpointgroup/ghi789
```

**Health Check Failures**
```bash
# Test endpoint directly
curl -I http://endpoint-dns-name/health

# Check security group rules
aws ec2 describe-security-groups \
  --group-ids sg-12345678 \
  --query 'SecurityGroups[0].IpPermissions'

# Verify NLB target health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:us-east-1:account:targetgroup/my-tg/123
```

### Diagnostic Commands

```bash
# Flow logs analysis
aws logs describe-log-groups \
  --log-group-name-prefix /aws/globalaccelerator

# VPC Flow Logs for endpoints
aws ec2 describe-flow-logs \
  --filter Name=resource-type,Values=NetworkInterface

# CloudWatch insights query
aws logs start-query \
  --log-group-name /aws/globalaccelerator/flowlogs \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    fields @timestamp, srcaddr, dstaddr, action
    | filter action = "REJECT"
    | stats count() by srcaddr
    | sort count desc
    | limit 20'
```

## Best Practices {#best-practices}

### Security Best Practices

```yaml
SecurityConfiguration:
  NetworkACLs:
    - Allow Global Accelerator IP ranges
    - Restrict source ports appropriately
    - Log denied connections
    
  SecurityGroups:
    - Allow traffic from Global Accelerator
    - Use specific port ranges
    - Implement least privilege access
    
  WAF Integration:
    - Apply WAF rules at ALB level
    - Monitor for suspicious patterns
    - Implement rate limiting
    
  DDoS Protection:
    - Enable AWS Shield Standard (free)
    - Consider Shield Advanced for critical apps
    - Monitor CloudWatch metrics
```

### Operational Best Practices

```bash
# Automated failover script
#!/bin/bash
ENDPOINT_GROUP_ARN="arn:aws:globalaccelerator::account:endpointgroup/abc123"
BACKUP_ENDPOINT_GROUP_ARN="arn:aws:globalaccelerator::account:endpointgroup/def456"

# Check primary health
PRIMARY_HEALTH=$(aws globalaccelerator describe-endpoint-group \
  --endpoint-group-arn $ENDPOINT_GROUP_ARN \
  --query 'EndpointGroup.EndpointDescriptions[0].HealthState' \
  --output text)

if [ "$PRIMARY_HEALTH" != "HEALTHY" ]; then
  echo "Primary unhealthy, failing over..."
  
  # Disable primary
  aws globalaccelerator update-endpoint-group \
    --endpoint-group-arn $ENDPOINT_GROUP_ARN \
    --traffic-dial-percentage 0
    
  # Enable backup
  aws globalaccelerator update-endpoint-group \
    --endpoint-group-arn $BACKUP_ENDPOINT_GROUP_ARN \
    --traffic-dial-percentage 100
    
  echo "Failover completed"
fi
```

### Monitoring and Alerting

```yaml
CloudWatchAlarms:
  HealthCheckFailure:
    MetricName: AWS/GlobalAccelerator/HealthCheckStatus
    Threshold: 0
    ComparisonOperator: LessThanThreshold
    EvaluationPeriods: 2
    
  HighLatency:
    MetricName: AWS/GlobalAccelerator/OriginLatency
    Threshold: 1000
    ComparisonOperator: GreaterThanThreshold
    EvaluationPeriods: 3
    
  PacketLoss:
    MetricName: AWS/GlobalAccelerator/PacketDropCount
    Threshold: 100
    ComparisonOperator: GreaterThanThreshold
    EvaluationPeriods: 2
```

## Labs and Exercises {#labs}

### Lab 1: Basic Global Accelerator Setup

**Objective**: Create a Global Accelerator with multi-region endpoints

**Prerequisites**:
- Two ALBs in different regions
- Basic understanding of AWS CLI

**Steps**:
1. Create Global Accelerator
2. Configure TCP listener
3. Add endpoint groups
4. Test connectivity and performance

### Lab 2: Blue-Green Deployment

**Objective**: Implement blue-green deployment using traffic dial

**Steps**:
1. Set up two identical environments
2. Configure endpoint groups with traffic dial
3. Perform zero-downtime deployment
4. Validate traffic switching

### Lab 3: Performance Comparison

**Objective**: Measure performance improvement with Global Accelerator

**Tools Needed**:
- curl with timing
- Multi-region test clients
- Performance monitoring scripts

**Metrics to Measure**:
- Connection time
- Time to first byte
- Total request time
- Packet loss and jitter

### Lab 4: Disaster Recovery Simulation

**Objective**: Test automated failover capabilities

**Scenario**:
1. Simulate primary region failure
2. Verify automatic failover
3. Test manual failover procedures
4. Measure recovery time objectives

This comprehensive guide covers AWS Global Accelerator from basic concepts to advanced implementation strategies. The service provides significant performance improvements for global applications through AWS's optimized network infrastructure, anycast IP addresses, and intelligent traffic routing capabilities.