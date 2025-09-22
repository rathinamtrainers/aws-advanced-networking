# Topic 49: DDoS Protection Strategies on AWS

## Introduction

Distributed Denial of Service (DDoS) attacks remain one of the most common and disruptive threats to online services. This comprehensive guide covers DDoS protection strategies, AWS-native solutions, and best practices for building resilient architectures.

## Understanding DDoS Attacks

### Types of DDoS Attacks

#### Volumetric Attacks
- **UDP Floods**: Overwhelm bandwidth with UDP packets
- **ICMP Floods**: Exhaust network resources with ping requests
- **DNS Amplification**: Exploit DNS servers to amplify attack traffic
- **NTP Amplification**: Use Network Time Protocol for traffic amplification

#### Protocol Attacks
- **SYN Floods**: Exhaust server connection tables
- **Ping of Death**: Malformed packets to crash systems
- **Smurf Attacks**: ICMP echo requests to broadcast addresses
- **Fragmented Packet Attacks**: Malformed packet fragments

#### Application Layer Attacks
- **HTTP Floods**: Overwhelm web servers with HTTP requests
- **Slowloris**: Keep connections open with slow HTTP requests
- **Zero-Day Exploits**: Target application vulnerabilities
- **Resource Exhaustion**: Consume CPU, memory, or database resources

### Attack Vectors and Methods

#### Botnet-Based Attacks
```
Attacker → Command & Control → Botnet (10,000+ devices) → Target
```

#### Reflection/Amplification Attacks
```
Attacker → Spoofed Request → Amplification Servers → Target (Amplified Response)
```

#### Multi-Vector Attacks
```
Layer 3: UDP Flood (100 Gbps)
Layer 4: SYN Flood (10M packets/sec)
Layer 7: HTTP Flood (100K requests/sec)
```

## AWS DDoS Protection Architecture

### Defense in Depth Strategy

#### Edge Protection
```
Internet → AWS Edge Locations → Regional Infrastructure → VPC → Application
         └── Shield Standard ──┘
```

#### Multi-Layer Defense
```
1. DNS Level: Route 53 with health checks
2. CDN Level: CloudFront with Shield
3. Load Balancer: ALB/NLB with WAF
4. Application: EC2 with Security Groups
5. Network: VPC with NACLs
```

### AWS Native Protection Services

#### AWS Shield Standard (Included)
- **Always-on protection** for all AWS customers
- **Layer 3/4 protection** against common attacks
- **Automatic scaling** with AWS infrastructure
- **No additional cost**

#### AWS Shield Advanced (Premium)
- **Enhanced DDoS protection** for critical applications
- **24/7 DDoS Response Team** access
- **Application layer protection**
- **Cost protection** during attacks

#### AWS WAF (Web Application Firewall)
- **Layer 7 protection** against application attacks
- **Rate limiting** for traffic control
- **IP reputation filtering**
- **Custom rule creation**

## Comprehensive Protection Strategies

### CloudFront-Based Protection

#### CDN Edge Protection
```json
{
    "DistributionConfig": {
        "Origins": [
            {
                "Id": "origin1",
                "DomainName": "origin.example.com",
                "CustomOriginConfig": {
                    "HTTPPort": 80,
                    "HTTPSPort": 443,
                    "OriginProtocolPolicy": "https-only",
                    "OriginReadTimeout": 30,
                    "OriginKeepaliveTimeout": 5
                }
            }
        ],
        "DefaultCacheBehavior": {
            "TargetOriginId": "origin1",
            "ViewerProtocolPolicy": "redirect-to-https",
            "AllowedMethods": {
                "Quantity": 7,
                "Items": ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"],
                "CachedMethods": {
                    "Quantity": 2,
                    "Items": ["GET", "HEAD"]
                }
            },
            "Compress": true,
            "DefaultTTL": 86400
        },
        "Comment": "DDoS Protection Distribution",
        "Enabled": true,
        "WebACLId": "arn:aws:wafv2:us-east-1:123456789012:global/webacl/DDoSProtectionWAF/12345678"
    }
}
```

#### Origin Shield Configuration
```bash
# Enable Origin Shield for additional protection
aws cloudfront create-distribution \
    --distribution-config '{
        "Origins": [{
            "Id": "protected-origin",
            "DomainName": "origin.example.com",
            "CustomOriginConfig": {
                "OriginShieldEnabled": true,
                "OriginShieldRegion": "us-east-1"
            }
        }]
    }'
```

### Load Balancer Protection

#### Application Load Balancer Setup
```bash
# Create ALB with DDoS protection
aws elbv2 create-load-balancer \
    --name "ddos-protected-alb" \
    --subnets subnet-12345678 subnet-87654321 \
    --security-groups sg-12345678 \
    --scheme internet-facing \
    --type application \
    --ip-address-type ipv4

# Configure target group with health checks
aws elbv2 create-target-group \
    --name "ddos-protected-targets" \
    --protocol HTTP \
    --port 80 \
    --vpc-id vpc-12345678 \
    --health-check-interval-seconds 10 \
    --health-check-timeout-seconds 5 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 3
```

#### Network Load Balancer for TCP Protection
```json
{
    "LoadBalancer": {
        "LoadBalancerName": "ddos-tcp-nlb",
        "Scheme": "internet-facing",
        "Type": "network",
        "IpAddressType": "ipv4",
        "Subnets": ["subnet-12345678", "subnet-87654321"],
        "LoadBalancerAttributes": [
            {
                "Key": "load_balancing.cross_zone.enabled",
                "Value": "true"
            },
            {
                "Key": "deletion_protection.enabled", 
                "Value": "true"
            }
        ]
    }
}
```

### WAF Protection Rules

#### Rate Limiting Rules
```json
{
    "Name": "DDoSRateLimit",
    "Priority": 1,
    "Statement": {
        "RateBasedStatement": {
            "Limit": 2000,
            "AggregateKeyType": "IP",
            "EvaluationWindowSec": 300
        }
    },
    "Action": {
        "Block": {
            "CustomResponse": {
                "ResponseCode": 429,
                "ResponseHeaders": [
                    {
                        "Name": "Retry-After",
                        "Value": "300"
                    }
                ]
            }
        }
    }
}
```

#### IP Reputation Rules
```json
{
    "Name": "MaliciousIPBlocking",
    "Priority": 2,
    "Statement": {
        "ManagedRuleGroupStatement": {
            "VendorName": "AWS",
            "Name": "AWSManagedRulesAmazonIpReputationList"
        }
    },
    "OverrideAction": {
        "None": {}
    }
}
```

#### Geographic Filtering
```json
{
    "Name": "GeographicBlocking",
    "Priority": 3,
    "Statement": {
        "NotStatement": {
            "Statement": {
                "GeoMatchStatement": {
                    "CountryCodes": ["US", "CA", "GB", "DE", "FR", "AU", "JP"]
                }
            }
        }
    },
    "Action": {
        "Block": {}
    }
}
```

## Auto Scaling for DDoS Resilience

### Elastic Scaling Configuration

#### Auto Scaling Groups
```json
{
    "AutoScalingGroup": {
        "AutoScalingGroupName": "ddos-resilient-asg",
        "LaunchTemplate": {
            "LaunchTemplateName": "ddos-protected-template",
            "Version": "$Latest"
        },
        "MinSize": 2,
        "MaxSize": 100,
        "DesiredCapacity": 4,
        "VPCZoneIdentifier": ["subnet-12345678", "subnet-87654321"],
        "HealthCheckType": "ELB",
        "HealthCheckGracePeriod": 300,
        "DefaultCooldown": 300
    }
}
```

#### Dynamic Scaling Policies
```bash
# Create target tracking scaling policy
aws autoscaling put-scaling-policy \
    --policy-name "ddos-cpu-scaling" \
    --auto-scaling-group-name "ddos-resilient-asg" \
    --policy-type "TargetTrackingScaling" \
    --target-tracking-configuration '{
        "TargetValue": 70.0,
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "ASGAverageCPUUtilization"
        }
    }'

# Create step scaling for rapid response
aws autoscaling put-scaling-policy \
    --policy-name "ddos-rapid-scale-out" \
    --auto-scaling-group-name "ddos-resilient-asg" \
    --policy-type "StepScaling" \
    --adjustment-type "ChangeInCapacity" \
    --step-adjustments '[
        {
            "MetricIntervalLowerBound": 0,
            "MetricIntervalUpperBound": 50,
            "ScalingAdjustment": 2
        },
        {
            "MetricIntervalLowerBound": 50,
            "ScalingAdjustment": 4
        }
    ]'
```

### Database Protection Strategies

#### RDS Protection
```bash
# Multi-AZ deployment for resilience
aws rds create-db-instance \
    --db-instance-identifier "ddos-protected-db" \
    --db-instance-class db.r5.large \
    --engine mysql \
    --master-username admin \
    --master-user-password "SecurePassword123!" \
    --allocated-storage 100 \
    --vpc-security-group-ids sg-database \
    --multi-az \
    --storage-encrypted \
    --backup-retention-period 7
```

#### ElastiCache for Request Reduction
```json
{
    "CacheCluster": {
        "CacheClusterId": "ddos-cache-cluster",
        "CacheNodeType": "cache.r6g.large",
        "Engine": "redis",
        "NumCacheNodes": 3,
        "PreferredAvailabilityZones": [
            "us-west-2a", "us-west-2b", "us-west-2c"
        ],
        "VpcSecurityGroupIds": ["sg-cache-security"],
        "SubnetGroupName": "cache-subnet-group"
    }
}
```

## Advanced Protection Techniques

### API Gateway Protection

#### Throttling Configuration
```bash
# Configure API Gateway throttling
aws apigateway put-method \
    --rest-api-id abcdef123 \
    --resource-id 123456 \
    --http-method GET \
    --authorization-type NONE \
    --api-key-required \
    --request-parameters '{
        "method.request.header.X-API-Key": true
    }'

# Set usage plan with throttling
aws apigateway create-usage-plan \
    --name "DDoSProtectedPlan" \
    --description "API usage plan with DDoS protection" \
    --throttle '{
        "BurstLimit": 200,
        "RateLimit": 100.0
    }' \
    --quota '{
        "Limit": 10000,
        "Period": "DAY"
    }'
```

### Route 53 Health Checks

#### Failover Configuration
```json
{
    "HealthCheck": {
        "Type": "HTTPS",
        "ResourcePath": "/health",
        "FullyQualifiedDomainName": "api.example.com",
        "Port": 443,
        "RequestInterval": 30,
        "FailureThreshold": 3,
        "MeasureLatency": true,
        "Regions": ["us-east-1", "us-west-2", "eu-west-1"],
        "AlarmIdentifier": {
            "Region": "us-east-1",
            "Name": "api-health-alarm"
        }
    }
}
```

#### DNS Failover Records
```bash
# Primary record
aws route53 change-resource-record-sets \
    --hosted-zone-id Z123456789 \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "api.example.com",
                "Type": "A",
                "SetIdentifier": "primary",
                "Failover": "PRIMARY",
                "AliasTarget": {
                    "DNSName": "primary-alb.us-west-2.elb.amazonaws.com",
                    "EvaluateTargetHealth": true,
                    "HostedZoneId": "Z1D633PJN98FT9"
                },
                "HealthCheckId": "12345678-1234-1234-1234-123456789012"
            }
        }]
    }'

# Failover record
aws route53 change-resource-record-sets \
    --hosted-zone-id Z123456789 \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE", 
            "ResourceRecordSet": {
                "Name": "api.example.com",
                "Type": "A",
                "SetIdentifier": "secondary",
                "Failover": "SECONDARY",
                "AliasTarget": {
                    "DNSName": "failover-alb.us-east-1.elb.amazonaws.com",
                    "EvaluateTargetHealth": true,
                    "HostedZoneId": "Z35SXDOTRQ7X7K"
                }
            }
        }]
    }'
```

## Monitoring and Detection

### CloudWatch Metrics and Alarms

#### DDoS Detection Metrics
```bash
# Create DDoS detection alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "DDoS-Attack-Detected" \
    --alarm-description "Detect potential DDoS attack" \
    --metric-name RequestCount \
    --namespace AWS/ApplicationELB \
    --statistic Sum \
    --period 60 \
    --threshold 10000 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions "arn:aws:sns:us-west-2:123456789012:ddos-alerts"

# Monitor error rates
aws cloudwatch put-metric-alarm \
    --alarm-name "High-Error-Rate" \
    --alarm-description "High error rate indicating attack" \
    --metric-name HTTPCode_Target_5XX_Count \
    --namespace AWS/ApplicationELB \
    --statistic Sum \
    --period 300 \
    --threshold 100 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 1
```

#### Custom Metrics
```python
import boto3
from datetime import datetime

def publish_custom_ddos_metrics():
    cloudwatch = boto3.client('cloudwatch')
    
    # Simulate application-level metrics
    metrics = {
        'ActiveConnections': 15000,
        'RequestLatency': 2500,
        'DatabaseConnections': 980,
        'CacheHitRate': 0.65
    }
    
    for metric_name, value in metrics.items():
        cloudwatch.put_metric_data(
            Namespace='DDoSProtection/Application',
            MetricData=[
                {
                    'MetricName': metric_name,
                    'Value': value,
                    'Unit': 'Count' if 'Connections' in metric_name else 'Milliseconds',
                    'Timestamp': datetime.utcnow()
                }
            ]
        )
```

### AWS Config Rules

#### Security Configuration Monitoring
```json
{
    "ConfigRuleName": "shield-advanced-enabled",
    "Description": "Checks if Shield Advanced is enabled for critical resources",
    "Source": {
        "Owner": "AWS",
        "SourceIdentifier": "SHIELD_ADVANCED_ENABLED_AUTO_REMEDIATION"
    },
    "Scope": {
        "ComplianceResourceTypes": [
            "AWS::ElasticLoadBalancingV2::LoadBalancer",
            "AWS::CloudFront::Distribution",
            "AWS::EC2::EIP"
        ]
    }
}
```

## Incident Response Procedures

### Automated Response Systems

#### Lambda-Based Auto-Response
```python
import boto3
import json

def ddos_auto_response(event, context):
    """
    Automated DDoS response function
    """
    # Parse CloudWatch alarm
    message = json.loads(event['Records'][0]['Sns']['Message'])
    alarm_name = message['AlarmName']
    
    if 'DDoS' in alarm_name:
        # Implement auto-scaling response
        scale_out_resources()
        
        # Update WAF rules for emergency blocking
        update_emergency_waf_rules()
        
        # Notify security team
        send_security_notification(message)
        
        # Enable enhanced monitoring
        enable_enhanced_monitoring()

def scale_out_resources():
    """Scale out resources quickly"""
    autoscaling = boto3.client('autoscaling')
    
    # Increase desired capacity
    autoscaling.update_auto_scaling_group(
        AutoScalingGroupName='ddos-resilient-asg',
        DesiredCapacity=20,
        MaxSize=50
    )

def update_emergency_waf_rules():
    """Activate emergency WAF rules"""
    waf = boto3.client('wafv2')
    
    # Lower rate limit threshold
    emergency_rule = {
        "Name": "EmergencyRateLimit",
        "Priority": 0,
        "Statement": {
            "RateBasedStatement": {
                "Limit": 100,
                "AggregateKeyType": "IP"
            }
        },
        "Action": {"Block": {}}
    }
    
    # Update Web ACL with emergency rule
    # Implementation details...

def send_security_notification(alarm_data):
    """Send notification to security team"""
    sns = boto3.client('sns')
    
    message = f"""
    DDoS Attack Detected!
    
    Alarm: {alarm_data['AlarmName']}
    Time: {alarm_data['StateChangeTime']}
    Reason: {alarm_data['NewStateReason']}
    
    Automated Response Initiated:
    - Auto Scaling activated
    - Emergency WAF rules deployed
    - Enhanced monitoring enabled
    
    Please review and take additional action if needed.
    """
    
    sns.publish(
        TopicArn='arn:aws:sns:us-west-2:123456789012:security-alerts',
        Subject='DDoS Attack Detected - Automated Response Active',
        Message=message
    )
```

### Manual Response Procedures

#### Emergency Response Checklist
```bash
# 1. Verify attack - Check metrics and logs
aws cloudwatch get-metric-statistics \
    --namespace AWS/ApplicationELB \
    --metric-name RequestCount \
    --start-time 2023-01-01T12:00:00Z \
    --end-time 2023-01-01T13:00:00Z \
    --period 300 \
    --statistics Sum

# 2. Contact AWS Support (if Shield Advanced)
# Call DDoS Response Team: +1-206-266-4064

# 3. Implement emergency scaling
aws autoscaling update-auto-scaling-group \
    --auto-scaling-group-name ddos-resilient-asg \
    --desired-capacity 25

# 4. Update WAF rules for immediate blocking
aws wafv2 update-web-acl \
    --id 12345678-1234-1234-1234-123456789012 \
    --scope REGIONAL \
    --default-action Block={} \
    --rules file://emergency-rules.json

# 5. Enable additional logging
aws wafv2 put-logging-configuration \
    --logging-configuration '{
        "ResourceArn": "arn:aws:wafv2:us-west-2:123456789012:regional/webacl/DDoSProtectionWAF/12345678",
        "LogDestinationConfigs": ["arn:aws:firehose:us-west-2:123456789012:deliverystream/waf-logs"]
    }'
```

## Cost Optimization During Attacks

### Cost Management Strategies

#### Shield Advanced Cost Protection
```bash
# Request cost protection for DDoS-related scaling
aws support create-case \
    --subject "Shield Advanced Cost Protection Request" \
    --service-code "shield-advanced" \
    --severity-code "high" \
    --category-code "ddos-cost-protection" \
    --communication-body "Requesting cost protection for DDoS attack scaling costs from [attack date/time]"
```

#### Intelligent Scaling Policies
```json
{
    "TargetTrackingConfiguration": {
        "TargetValue": 70.0,
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "ASGAverageCPUUtilization"
        },
        "ScaleOutCooldown": 60,
        "ScaleInCooldown": 300,
        "DisableScaleIn": false
    }
}
```

## Testing and Validation

### DDoS Simulation Testing

#### Load Testing Framework
```python
import asyncio
import aiohttp
import time

async def ddos_simulation_test():
    """
    Simulate traffic load for testing DDoS protection
    WARNING: Only use on your own infrastructure with permission
    """
    target_url = "https://test-app.example.com"
    concurrent_requests = 1000
    test_duration = 300  # 5 minutes
    
    async with aiohttp.ClientSession() as session:
        start_time = time.time()
        
        while time.time() - start_time < test_duration:
            tasks = []
            for _ in range(concurrent_requests):
                task = asyncio.create_task(
                    make_request(session, target_url)
                )
                tasks.append(task)
            
            await asyncio.gather(*tasks, return_exceptions=True)
            await asyncio.sleep(1)

async def make_request(session, url):
    try:
        async with session.get(url, timeout=10) as response:
            return response.status
    except Exception as e:
        return str(e)
```

#### Penetration Testing
```bash
# Use tools like hping3 for network layer testing
# WARNING: Only on your own infrastructure

# SYN flood simulation
hping3 -S -p 80 --flood test-target.example.com

# UDP flood simulation  
hping3 -2 -p 80 --flood test-target.example.com

# HTTP flood simulation using Apache Bench
ab -n 10000 -c 100 https://test-app.example.com/
```

## Best Practices Summary

### Architecture Best Practices
1. **Multi-Layer Defense**: Implement protection at multiple levels
2. **Geographic Distribution**: Use multiple AWS regions
3. **Auto Scaling**: Configure aggressive scaling policies
4. **Health Checks**: Implement comprehensive monitoring
5. **Failover Planning**: Design for graceful degradation

### Operational Best Practices
1. **Regular Testing**: Conduct DDoS simulation exercises
2. **Incident Response**: Maintain updated response procedures
3. **Monitoring**: Implement comprehensive alerting
4. **Documentation**: Keep architecture and procedures current
5. **Training**: Ensure team preparedness

### Cost Optimization
1. **Right-sizing**: Balance protection with cost
2. **Reserved Capacity**: Use reserved instances for baseline
3. **Spot Instances**: Consider for non-critical scaling
4. **Shield Advanced**: Evaluate cost protection benefits
5. **Regular Review**: Optimize based on attack patterns

## Conclusion

Effective DDoS protection on AWS requires a comprehensive strategy combining AWS native services, proper architecture design, and operational procedures. By implementing these protection strategies and maintaining vigilant monitoring, organizations can build resilient systems capable of withstanding sophisticated DDoS attacks while maintaining cost efficiency and performance.