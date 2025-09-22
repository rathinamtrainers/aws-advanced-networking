# Topic 61: Route 53 Latency-Based Routing - Configuration and Optimization

## Table of Contents
1. [Latency-Based Routing Overview](#overview)
2. [How Latency-Based Routing Works](#how-it-works)
3. [Configuration and Setup](#configuration)
4. [Health Check Integration](#health-checks)
5. [Performance Monitoring](#monitoring)
6. [Traffic Optimization Strategies](#optimization)
7. [Multi-Region Implementation](#multi-region)
8. [Advanced Routing Policies](#advanced-policies)
9. [Cost Analysis and Optimization](#cost-analysis)
10. [Troubleshooting and Diagnostics](#troubleshooting)
11. [Best Practices](#best-practices)
12. [Real-World Use Cases](#use-cases)

## Latency-Based Routing Overview {#overview}

### What is Latency-Based Routing?

Route 53 latency-based routing automatically directs users to the AWS region that provides the lowest latency for their requests. This routing policy measures the latency between users and AWS regions, then routes traffic to the region that can serve users most quickly.

**Key Benefits**:
- Improved user experience through reduced latency
- Automatic optimization based on real-time network conditions
- Global load distribution based on performance
- Seamless integration with health checks
- Dynamic routing adjustments

**Core Components**:

```
Latency-Based Routing Architecture:

                    ┌─────────────────────────────────┐
                    │         User Requests           │
                    │    (Global Distribution)        │
                    └─────────────────┬───────────────┘
                                      │
                    ┌─────────────────▼───────────────┐
                    │        Route 53 Resolver        │
                    │    Latency Measurement Engine   │
                    │                                 │
                    │  ┌─────────────────────────────┐ │
                    │  │    Latency Database         │ │
                    │  │  - User → Region latency    │ │
                    │  │  - Historical performance   │ │
                    │  │  - Real-time measurements   │ │
                    │  └─────────────────────────────┘ │
                    └─────────────────┬───────────────┘
                                      │
                    ┌─────────────────▼───────────────┐
                    │      Routing Decision           │
                    │   (Lowest Latency Region)       │
                    └─────────────────┬───────────────┘
                                      │
         ┌────────────────────────────┼────────────────────────────┐
         │                            │                            │
         ▼                            ▼                            ▼
┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐
│   Region A      │        │   Region B      │        │   Region C      │
│  (us-east-1)    │        │  (eu-west-1)    │        │ (ap-south-1)    │
│                 │        │                 │        │                 │
│ User A: 50ms    │        │ User A: 150ms   │        │ User A: 300ms   │
│ User B: 120ms   │        │ User B: 80ms    │        │ User B: 200ms   │
│ User C: 280ms   │        │ User C: 200ms   │        │ User C: 50ms    │
└─────────────────┘        └─────────────────┘        └─────────────────┘
         ↑                           ↑                           ↑
    Routes User A             Routes User B             Routes User C
   (Lowest Latency)         (Lowest Latency)         (Lowest Latency)
```

### Latency Measurement Process

```
Latency Measurement Framework:

1. Data Collection:
   ┌─────────────────────────────────────────────────────────────┐
   │ Source: Amazon CloudFront Edge Locations                   │
   │ Method: Periodic latency probes to all regions             │
   │ Frequency: Continuous monitoring                           │
   │ Granularity: Per region, per resolver location            │
   └─────────────────────────────────────────────────────────────┘

2. Latency Database:
   ┌─────────────────────────────────────────────────────────────┐
   │ Storage: Route 53 internal latency database                │
   │ Updates: Real-time based on measurements                   │
   │ Scope: Global coverage across all AWS regions              │
   │ Accuracy: Sub-millisecond precision                        │
   └─────────────────────────────────────────────────────────────┘

3. Routing Decision:
   ┌─────────────────────────────────────────────────────────────┐
   │ Input: User's DNS resolver location                         │
   │ Lookup: Latency to all configured regions                  │
   │ Selection: Region with lowest average latency               │
   │ Response: IP address of selected region endpoint            │
   └─────────────────────────────────────────────────────────────┘
```

## How Latency-Based Routing Works {#how-it-works}

### Latency Measurement Methodology

Route 53 uses a sophisticated system to measure and track latency between user locations and AWS regions:

```python
import boto3
import json
import time
from typing import Dict, List, Tuple
from collections import defaultdict
import statistics

class LatencyAnalyzer:
    def __init__(self):
        self.route53 = boto3.client('route53')
        self.cloudwatch = boto3.client('cloudwatch')
        
    def simulate_latency_measurements(self, 
                                    user_locations: List[str],
                                    aws_regions: List[str]) -> Dict:
        """Simulate how Route 53 measures latency from users to regions"""
        
        # Simulated latency data (in real world, this comes from CloudFront edge locations)
        latency_matrix = {
            'New York': {
                'us-east-1': 15,
                'us-west-1': 70,
                'us-west-2': 80,
                'eu-west-1': 90,
                'eu-central-1': 110,
                'ap-south-1': 200,
                'ap-southeast-1': 220,
                'ap-northeast-1': 180
            },
            'London': {
                'us-east-1': 80,
                'us-west-1': 150,
                'us-west-2': 160,
                'eu-west-1': 10,
                'eu-central-1': 25,
                'ap-south-1': 120,
                'ap-southeast-1': 180,
                'ap-northeast-1': 240
            },
            'Mumbai': {
                'us-east-1': 200,
                'us-west-1': 280,
                'us-west-2': 250,
                'eu-west-1': 120,
                'eu-central-1': 140,
                'ap-south-1': 15,
                'ap-southeast-1': 80,
                'ap-northeast-1': 100
            },
            'Singapore': {
                'us-east-1': 220,
                'us-west-1': 180,
                'us-west-2': 160,
                'eu-west-1': 180,
                'eu-central-1': 200,
                'ap-south-1': 80,
                'ap-southeast-1': 5,
                'ap-northeast-1': 60
            },
            'Tokyo': {
                'us-east-1': 180,
                'us-west-1': 120,
                'us-west-2': 100,
                'eu-west-1': 240,
                'eu-central-1': 260,
                'ap-south-1': 100,
                'ap-southeast-1': 60,
                'ap-northeast-1': 8
            }
        }
        
        routing_decisions = {}
        
        for location in user_locations:
            if location in latency_matrix:
                region_latencies = latency_matrix[location]
                
                # Filter to only configured regions
                filtered_latencies = {
                    region: latency 
                    for region, latency in region_latencies.items() 
                    if region in aws_regions
                }
                
                # Find region with lowest latency
                best_region = min(filtered_latencies, key=filtered_latencies.get)
                best_latency = filtered_latencies[best_region]
                
                routing_decisions[location] = {
                    'selected_region': best_region,
                    'latency_ms': best_latency,
                    'all_latencies': filtered_latencies,
                    'alternative_options': sorted(
                        [(region, latency) for region, latency in filtered_latencies.items()],
                        key=lambda x: x[1]
                    )[1:3]  # Next 2 best options
                }
        
        return routing_decisions
    
    def analyze_latency_patterns(self, 
                               routing_decisions: Dict) -> Dict:
        """Analyze latency patterns and routing efficiency"""
        
        analysis = {
            'regional_utilization': defaultdict(int),
            'latency_distribution': defaultdict(list),
            'efficiency_metrics': {},
            'optimization_opportunities': []
        }
        
        # Count regional utilization
        for location, decision in routing_decisions.items():
            region = decision['selected_region']
            latency = decision['latency_ms']
            
            analysis['regional_utilization'][region] += 1
            analysis['latency_distribution'][region].append(latency)
        
        # Calculate efficiency metrics
        total_locations = len(routing_decisions)
        all_latencies = [decision['latency_ms'] for decision in routing_decisions.values()]
        
        analysis['efficiency_metrics'] = {
            'average_latency': statistics.mean(all_latencies),
            'median_latency': statistics.median(all_latencies),
            'max_latency': max(all_latencies),
            'min_latency': min(all_latencies),
            'latency_variance': statistics.variance(all_latencies),
            'regional_distribution': {
                region: (count / total_locations) * 100
                for region, count in analysis['regional_utilization'].items()
            }
        }
        
        # Identify optimization opportunities
        for region, latencies in analysis['latency_distribution'].items():
            if latencies:
                avg_latency = statistics.mean(latencies)
                if avg_latency > 100:  # High latency threshold
                    analysis['optimization_opportunities'].append({
                        'region': region,
                        'issue': 'High average latency',
                        'current_avg': avg_latency,
                        'recommendation': 'Consider adding closer region or CDN'
                    })
        
        return analysis
    
    def simulate_failover_impact(self, 
                               routing_decisions: Dict,
                               failed_region: str) -> Dict:
        """Simulate impact of region failure on latency-based routing"""
        
        failover_analysis = {
            'affected_users': [],
            'new_routing': {},
            'latency_impact': {},
            'total_impact_summary': {}
        }
        
        for location, decision in routing_decisions.items():
            if decision['selected_region'] == failed_region:
                # User was routed to failed region, need alternative
                failover_analysis['affected_users'].append(location)
                
                # Find next best region
                all_latencies = decision['all_latencies']
                available_regions = {
                    region: latency 
                    for region, latency in all_latencies.items() 
                    if region != failed_region
                }
                
                if available_regions:
                    new_best_region = min(available_regions, key=available_regions.get)
                    new_latency = available_regions[new_best_region]
                    original_latency = decision['latency_ms']
                    
                    failover_analysis['new_routing'][location] = {
                        'new_region': new_best_region,
                        'new_latency': new_latency,
                        'original_latency': original_latency,
                        'latency_increase': new_latency - original_latency,
                        'performance_degradation': (
                            (new_latency - original_latency) / original_latency
                        ) * 100
                    }
        
        # Calculate overall impact
        if failover_analysis['new_routing']:
            latency_increases = [
                routing['latency_increase'] 
                for routing in failover_analysis['new_routing'].values()
            ]
            
            performance_degradations = [
                routing['performance_degradation']
                for routing in failover_analysis['new_routing'].values()
            ]
            
            failover_analysis['total_impact_summary'] = {
                'affected_user_count': len(failover_analysis['affected_users']),
                'avg_latency_increase': statistics.mean(latency_increases),
                'max_latency_increase': max(latency_increases),
                'avg_performance_degradation': statistics.mean(performance_degradations),
                'users_with_significant_impact': len([
                    d for d in performance_degradations if d > 50
                ])  # >50% degradation
            }
        
        return failover_analysis

# Usage example
analyzer = LatencyAnalyzer()

# Define user locations and available regions
user_locations = ['New York', 'London', 'Mumbai', 'Singapore', 'Tokyo']
aws_regions = ['us-east-1', 'us-west-2', 'eu-west-1', 'ap-south-1', 'ap-northeast-1']

# Simulate routing decisions
routing_decisions = analyzer.simulate_latency_measurements(user_locations, aws_regions)
print("Latency-Based Routing Decisions:")
print(json.dumps(routing_decisions, indent=2))

# Analyze patterns
analysis = analyzer.analyze_latency_patterns(routing_decisions)
print("\nLatency Pattern Analysis:")
print(json.dumps(analysis, indent=2))

# Simulate failover impact
failover_impact = analyzer.simulate_failover_impact(routing_decisions, 'us-east-1')
print("\nFailover Impact Analysis (us-east-1 failure):")
print(json.dumps(failover_impact, indent=2))
```

### DNS Resolution Flow

```
Detailed DNS Resolution Flow for Latency-Based Routing:

1. User Request:
   ┌─────────────────────────────────────────────────────────────┐
   │ User in London requests: api.example.com                   │
   │ DNS Resolver: London ISP (Approximate location)            │
   └─────────────────┬───────────────────────────────────────────┘
                     │
2. Route 53 Lookup:  ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ Route 53 identifies resolver location: London area         │
   │ Queries latency database for London → AWS regions          │
   │                                                             │
   │ Latency Results:                                           │
   │ - London → us-east-1: 80ms                                │
   │ - London → eu-west-1: 10ms  ← LOWEST                      │
   │ - London → ap-south-1: 120ms                              │
   └─────────────────┬───────────────────────────────────────────┘
                     │
3. Health Check:     ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ Verifies eu-west-1 endpoint health status                  │
   │ - Health check: PASSING                                     │
   │ - Endpoint available: YES                                   │
   └─────────────────┬───────────────────────────────────────────┘
                     │
4. Response:         ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ Returns IP address of eu-west-1 endpoint                   │
   │ Response: 52.48.x.x (eu-west-1 IP)                        │
   │ TTL: 60 seconds                                            │
   └─────────────────┬───────────────────────────────────────────┘
                     │
5. User Connection:  ▼
   ┌─────────────────────────────────────────────────────────────┐
   │ User connects directly to eu-west-1 endpoint               │
   │ Actual latency: ~10ms (optimal for London user)            │
   └─────────────────────────────────────────────────────────────┘
```

## Configuration and Setup {#configuration}

### Basic Latency-Based Routing Setup

```bash
#!/bin/bash
# Script to set up latency-based routing with Route 53

# Configuration variables
HOSTED_ZONE_ID="Z123456789ABCDEF"
DOMAIN_NAME="api.example.com"
TTL=60

# Health check IDs (created separately)
US_EAST_1_HEALTH_CHECK="hc-us-east-1-12345"
EU_WEST_1_HEALTH_CHECK="hc-eu-west-1-67890"
AP_SOUTH_1_HEALTH_CHECK="hc-ap-south-1-abcde"

echo "Setting up latency-based routing for $DOMAIN_NAME"

# Create latency-based record for us-east-1
echo "Creating latency record for us-east-1..."
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Comment": "Add latency-based routing for us-east-1",
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "'$DOMAIN_NAME'",
          "Type": "A",
          "SetIdentifier": "us-east-1-latency",
          "Region": "us-east-1",
          "TTL": '$TTL',
          "ResourceRecords": [
            {
              "Value": "54.239.28.85"
            }
          ],
          "HealthCheckId": "'$US_EAST_1_HEALTH_CHECK'"
        }
      }
    ]
  }'

# Create latency-based record for eu-west-1
echo "Creating latency record for eu-west-1..."
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Comment": "Add latency-based routing for eu-west-1",
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "'$DOMAIN_NAME'",
          "Type": "A",
          "SetIdentifier": "eu-west-1-latency",
          "Region": "eu-west-1",
          "TTL": '$TTL',
          "ResourceRecords": [
            {
              "Value": "52.48.120.2"
            }
          ],
          "HealthCheckId": "'$EU_WEST_1_HEALTH_CHECK'"
        }
      }
    ]
  }'

# Create latency-based record for ap-south-1
echo "Creating latency record for ap-south-1..."
aws route53 change-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --change-batch '{
    "Comment": "Add latency-based routing for ap-south-1",
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "'$DOMAIN_NAME'",
          "Type": "A",
          "SetIdentifier": "ap-south-1-latency",
          "Region": "ap-south-1",
          "TTL": '$TTL',
          "ResourceRecords": [
            {
              "Value": "13.232.67.1"
            }
          ],
          "HealthCheckId": "'$AP_SOUTH_1_HEALTH_CHECK'"
        }
      }
    ]
  }'

echo "Latency-based routing setup completed"

# Verify the configuration
echo "Verifying configuration..."
aws route53 list-resource-record-sets \
  --hosted-zone-id $HOSTED_ZONE_ID \
  --query "ResourceRecordSets[?Name=='$DOMAIN_NAME.']" \
  --output table
```

### Advanced CloudFormation Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Advanced latency-based routing with health checks and monitoring'

Parameters:
  DomainName:
    Type: String
    Description: 'Domain name for latency-based routing'
    Default: 'api.example.com'
    
  HostedZoneId:
    Type: String
    Description: 'Route 53 hosted zone ID'
    
  TTLValue:
    Type: Number
    Description: 'TTL for DNS records'
    Default: 60
    MinValue: 30
    MaxValue: 3600

Resources:
  # Health checks for each region
  USEast1HealthCheck:
    Type: AWS::Route53::HealthCheck
    Properties:
      Type: HTTPS
      ResourcePath: '/health'
      FullyQualifiedDomainName: !Sub 'us-east-1-${DomainName}'
      Port: 443
      RequestInterval: 30
      FailureThreshold: 3
      MeasureLatency: true
      Regions:
        - us-east-1
        - us-west-1
        - eu-west-1
      Tags:
        - Key: Name
          Value: !Sub '${DomainName}-us-east-1-health'
        - Key: Region
          Value: 'us-east-1'

  EUWest1HealthCheck:
    Type: AWS::Route53::HealthCheck
    Properties:
      Type: HTTPS
      ResourcePath: '/health'
      FullyQualifiedDomainName: !Sub 'eu-west-1-${DomainName}'
      Port: 443
      RequestInterval: 30
      FailureThreshold: 3
      MeasureLatency: true
      Regions:
        - us-east-1
        - eu-west-1
        - ap-south-1
      Tags:
        - Key: Name
          Value: !Sub '${DomainName}-eu-west-1-health'
        - Key: Region
          Value: 'eu-west-1'

  APSouth1HealthCheck:
    Type: AWS::Route53::HealthCheck
    Properties:
      Type: HTTPS
      ResourcePath: '/health'
      FullyQualifiedDomainName: !Sub 'ap-south-1-${DomainName}'
      Port: 443
      RequestInterval: 30
      FailureThreshold: 3
      MeasureLatency: true
      Regions:
        - ap-south-1
        - ap-southeast-1
        - eu-west-1
      Tags:
        - Key: Name
          Value: !Sub '${DomainName}-ap-south-1-health'
        - Key: Region
          Value: 'ap-south-1'

  # Latency-based records
  USEast1LatencyRecord:
    Type: AWS::Route53::RecordSet
    Properties:
      HostedZoneId: !Ref HostedZoneId
      Name: !Ref DomainName
      Type: A
      SetIdentifier: 'us-east-1-latency'
      Region: 'us-east-1'
      TTL: !Ref TTLValue
      ResourceRecords:
        - '54.239.28.85'  # Replace with actual endpoint IP
      HealthCheckId: !Ref USEast1HealthCheck

  EUWest1LatencyRecord:
    Type: AWS::Route53::RecordSet
    Properties:
      HostedZoneId: !Ref HostedZoneId
      Name: !Ref DomainName
      Type: A
      SetIdentifier: 'eu-west-1-latency'
      Region: 'eu-west-1'
      TTL: !Ref TTLValue
      ResourceRecords:
        - '52.48.120.2'  # Replace with actual endpoint IP
      HealthCheckId: !Ref EUWest1HealthCheck

  APSouth1LatencyRecord:
    Type: AWS::Route53::RecordSet
    Properties:
      HostedZoneId: !Ref HostedZoneId
      Name: !Ref DomainName
      Type: A
      SetIdentifier: 'ap-south-1-latency'
      Region: 'ap-south-1'
      TTL: !Ref TTLValue
      ResourceRecords:
        - '13.232.67.1'  # Replace with actual endpoint IP
      HealthCheckId: !Ref APSouth1HealthCheck

  # CloudWatch Alarms for health check monitoring
  USEast1HealthAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${DomainName}-us-east-1-health-alarm'
      AlarmDescription: 'Health check alarm for us-east-1 endpoint'
      MetricName: HealthCheckStatus
      Namespace: AWS/Route53
      Statistic: Minimum
      Period: 60
      EvaluationPeriods: 2
      Threshold: 1
      ComparisonOperator: LessThanThreshold
      Dimensions:
        - Name: HealthCheckId
          Value: !Ref USEast1HealthCheck
      AlarmActions:
        - !Ref HealthCheckTopic

  EUWest1HealthAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${DomainName}-eu-west-1-health-alarm'
      AlarmDescription: 'Health check alarm for eu-west-1 endpoint'
      MetricName: HealthCheckStatus
      Namespace: AWS/Route53
      Statistic: Minimum
      Period: 60
      EvaluationPeriods: 2
      Threshold: 1
      ComparisonOperator: LessThanThreshold
      Dimensions:
        - Name: HealthCheckId
          Value: !Ref EUWest1HealthCheck
      AlarmActions:
        - !Ref HealthCheckTopic

  APSouth1HealthAlarm:
    Type: AWS::CloudWatch::Alarm
    Properties:
      AlarmName: !Sub '${DomainName}-ap-south-1-health-alarm'
      AlarmDescription: 'Health check alarm for ap-south-1 endpoint'
      MetricName: HealthCheckStatus
      Namespace: AWS/Route53
      Statistic: Minimum
      Period: 60
      EvaluationPeriods: 2
      Threshold: 1
      ComparisonOperator: LessThanThreshold
      Dimensions:
        - Name: HealthCheckId
          Value: !Ref APSouth1HealthCheck
      AlarmActions:
        - !Ref HealthCheckTopic

  # SNS Topic for health check notifications
  HealthCheckTopic:
    Type: AWS::SNS::Topic
    Properties:
      TopicName: !Sub '${DomainName}-health-check-alerts'
      DisplayName: 'Route 53 Health Check Alerts'

  # Lambda function for latency monitoring
  LatencyMonitoringFunction:
    Type: AWS::Lambda::Function
    Properties:
      FunctionName: !Sub '${DomainName}-latency-monitor'
      Runtime: python3.9
      Handler: index.handler
      Timeout: 60
      Role: !GetAtt LatencyMonitoringRole.Arn
      Code:
        ZipFile: |
          import boto3
          import json
          import logging
          
          logger = logging.getLogger()
          logger.setLevel(logging.INFO)
          
          route53 = boto3.client('route53')
          cloudwatch = boto3.client('cloudwatch')
          
          def handler(event, context):
              try:
                  # Get health check metrics and publish custom metrics
                  health_checks = route53.list_health_checks()
                  
                  for hc in health_checks['HealthChecks']:
                      hc_id = hc['Id']
                      
                      # Get health check status
                      status = route53.get_health_check_status(Id=hc_id)
                      
                      # Calculate health percentage
                      checkers = status['StatusCheckers']
                      healthy_count = sum(1 for c in checkers if c['Status'] == 'Success')
                      total_count = len(checkers)
                      health_percentage = (healthy_count / total_count * 100) if total_count > 0 else 0
                      
                      # Publish custom metric
                      cloudwatch.put_metric_data(
                          Namespace='Route53/LatencyRouting',
                          MetricData=[
                              {
                                  'MetricName': 'HealthPercentage',
                                  'Dimensions': [
                                      {'Name': 'HealthCheckId', 'Value': hc_id}
                                  ],
                                  'Value': health_percentage,
                                  'Unit': 'Percent'
                              }
                          ]
                      )
                  
                  return {'statusCode': 200, 'body': 'Metrics published successfully'}
                  
              except Exception as e:
                  logger.error(f'Error: {e}')
                  return {'statusCode': 500, 'body': str(e)}

  # IAM Role for Lambda function
  LatencyMonitoringRole:
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
        - PolicyName: Route53HealthCheckAccess
          PolicyDocument:
            Version: '2012-10-17'
            Statement:
              - Effect: Allow
                Action:
                  - route53:ListHealthChecks
                  - route53:GetHealthCheckStatus
                  - cloudwatch:PutMetricData
                Resource: '*'

  # EventBridge rule to trigger Lambda function
  LatencyMonitoringSchedule:
    Type: AWS::Events::Rule
    Properties:
      Description: 'Trigger latency monitoring function every 5 minutes'
      ScheduleExpression: 'rate(5 minutes)'
      State: ENABLED
      Targets:
        - Arn: !GetAtt LatencyMonitoringFunction.Arn
          Id: 'LatencyMonitoringTarget'

  # Permission for EventBridge to invoke Lambda
  LatencyMonitoringPermission:
    Type: AWS::Lambda::Permission
    Properties:
      FunctionName: !Ref LatencyMonitoringFunction
      Action: lambda:InvokeFunction
      Principal: events.amazonaws.com
      SourceArn: !GetAtt LatencyMonitoringSchedule.Arn

Outputs:
  DomainName:
    Description: 'Configured domain name'
    Value: !Ref DomainName
    
  HealthCheckIds:
    Description: 'Health check IDs'
    Value: !Sub '${USEast1HealthCheck}, ${EUWest1HealthCheck}, ${APSouth1HealthCheck}'
    
  HealthCheckTopic:
    Description: 'SNS topic for health check alerts'
    Value: !Ref HealthCheckTopic
    Export:
      Name: !Sub '${AWS::StackName}-health-check-topic'
```

### Terraform Configuration

```hcl
# Terraform configuration for latency-based routing
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Variables
variable "domain_name" {
  description = "Domain name for latency-based routing"
  type        = string
  default     = "api.example.com"
}

variable "hosted_zone_id" {
  description = "Route 53 hosted zone ID"
  type        = string
}

variable "ttl_value" {
  description = "TTL for DNS records"
  type        = number
  default     = 60
}

variable "regions" {
  description = "List of regions with their endpoints"
  type = map(object({
    endpoint_ip = string
    health_path = string
  }))
  default = {
    "us-east-1" = {
      endpoint_ip = "54.239.28.85"
      health_path = "/health"
    }
    "eu-west-1" = {
      endpoint_ip = "52.48.120.2"
      health_path = "/health"
    }
    "ap-south-1" = {
      endpoint_ip = "13.232.67.1"
      health_path = "/health"
    }
  }
}

# Health checks for each region
resource "aws_route53_health_check" "region_health_checks" {
  for_each = var.regions

  type                            = "HTTPS"
  resource_path                  = each.value.health_path
  fqdn                           = "${each.key}-${var.domain_name}"
  port                           = 443
  request_interval               = 30
  failure_threshold              = 3
  measure_latency                = true
  cloudwatch_alarm_region        = each.key
  cloudwatch_alarm_name          = "route53-health-check-${each.key}"
  insufficient_data_health_status = "Failure"

  tags = {
    Name   = "${var.domain_name}-${each.key}-health"
    Region = each.key
    Type   = "LatencyBasedRouting"
  }
}

# Latency-based DNS records
resource "aws_route53_record" "latency_records" {
  for_each = var.regions

  zone_id         = var.hosted_zone_id
  name            = var.domain_name
  type            = "A"
  ttl             = var.ttl_value
  set_identifier  = "${each.key}-latency"
  
  latency_routing_policy {
    region = each.key
  }

  records = [each.value.endpoint_ip]
  
  health_check_id = aws_route53_health_check.region_health_checks[each.key].id
}

# CloudWatch alarms for health checks
resource "aws_cloudwatch_metric_alarm" "health_check_alarms" {
  for_each = var.regions

  alarm_name          = "${var.domain_name}-${each.key}-health-alarm"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = "2"
  metric_name         = "HealthCheckStatus"
  namespace           = "AWS/Route53"
  period              = "60"
  statistic           = "Minimum"
  threshold           = "1"
  alarm_description   = "This metric monitors health check status for ${each.key}"
  alarm_actions       = [aws_sns_topic.health_check_alerts.arn]

  dimensions = {
    HealthCheckId = aws_route53_health_check.region_health_checks[each.key].id
  }

  tags = {
    Name   = "${var.domain_name}-${each.key}-alarm"
    Region = each.key
  }
}

# SNS topic for health check alerts
resource "aws_sns_topic" "health_check_alerts" {
  name = "${replace(var.domain_name, ".", "-")}-health-alerts"

  tags = {
    Name = "${var.domain_name} Health Check Alerts"
  }
}

# Lambda function for enhanced monitoring
resource "aws_lambda_function" "latency_monitor" {
  filename         = "latency_monitor.zip"
  function_name    = "${replace(var.domain_name, ".", "-")}-latency-monitor"
  role            = aws_iam_role.lambda_role.arn
  handler         = "index.handler"
  runtime         = "python3.9"
  timeout         = 60

  source_code_hash = data.archive_file.lambda_zip.output_base64sha256

  environment {
    variables = {
      DOMAIN_NAME = var.domain_name
      REGIONS     = jsonencode(keys(var.regions))
    }
  }

  tags = {
    Name = "${var.domain_name} Latency Monitor"
  }
}

# Lambda deployment package
data "archive_file" "lambda_zip" {
  type        = "zip"
  output_path = "latency_monitor.zip"
  source {
    content = templatefile("${path.module}/lambda/latency_monitor.py", {
      domain_name = var.domain_name
    })
    filename = "index.py"
  }
}

# IAM role for Lambda
resource "aws_iam_role" "lambda_role" {
  name = "${replace(var.domain_name, ".", "-")}-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
}

# IAM policy for Lambda
resource "aws_iam_role_policy" "lambda_policy" {
  name = "${replace(var.domain_name, ".", "-")}-lambda-policy"
  role = aws_iam_role.lambda_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:*:*:*"
      },
      {
        Effect = "Allow"
        Action = [
          "route53:ListHealthChecks",
          "route53:GetHealthCheckStatus",
          "cloudwatch:PutMetricData"
        ]
        Resource = "*"
      }
    ]
  })
}

# EventBridge rule for Lambda scheduling
resource "aws_cloudwatch_event_rule" "latency_monitor_schedule" {
  name                = "${replace(var.domain_name, ".", "-")}-latency-schedule"
  description         = "Trigger latency monitoring every 5 minutes"
  schedule_expression = "rate(5 minutes)"

  tags = {
    Name = "${var.domain_name} Latency Monitor Schedule"
  }
}

# EventBridge target
resource "aws_cloudwatch_event_target" "lambda_target" {
  rule      = aws_cloudwatch_event_rule.latency_monitor_schedule.name
  target_id = "LatencyMonitorTarget"
  arn       = aws_lambda_function.latency_monitor.arn
}

# Permission for EventBridge to invoke Lambda
resource "aws_lambda_permission" "allow_eventbridge" {
  statement_id  = "AllowExecutionFromEventBridge"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.latency_monitor.function_name
  principal     = "events.amazonaws.com"
  source_arn    = aws_cloudwatch_event_rule.latency_monitor_schedule.arn
}

# Outputs
output "domain_name" {
  description = "Configured domain name"
  value       = var.domain_name
}

output "health_check_ids" {
  description = "Map of region to health check ID"
  value = {
    for region, health_check in aws_route53_health_check.region_health_checks : 
    region => health_check.id
  }
}

output "sns_topic_arn" {
  description = "SNS topic ARN for health check alerts"
  value       = aws_sns_topic.health_check_alerts.arn
}

output "lambda_function_name" {
  description = "Lambda function name for latency monitoring"
  value       = aws_lambda_function.latency_monitor.function_name
}
```

This comprehensive implementation provides robust latency-based routing with health checks, monitoring, and automation capabilities. The configuration can be adapted to specific organizational requirements while maintaining best practices for performance and reliability.