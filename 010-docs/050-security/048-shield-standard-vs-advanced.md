# Topic 48: AWS Shield Standard vs Shield Advanced

## Introduction

AWS Shield provides DDoS (Distributed Denial of Service) protection for AWS resources. Understanding the differences between Shield Standard and Shield Advanced is crucial for implementing appropriate DDoS protection strategies.

## AWS Shield Overview

### What is AWS Shield?

AWS Shield is a **managed DDoS protection service** that safeguards applications running on AWS. It provides always-on detection and automatic inline mitigations that minimize application downtime and latency.

### Types of DDoS Attacks

#### Layer 3/4 Attacks (Network/Transport Layer)
- **SYN Floods**: Exhaust connection state tables
- **UDP Floods**: Overwhelm network bandwidth
- **ICMP Floods**: Consume network resources
- **Reflection Attacks**: Amplify attack traffic

#### Layer 7 Attacks (Application Layer)
- **HTTP Floods**: Overwhelm web servers with requests
- **Slowloris**: Exhaust server connection pools
- **Application-Specific**: Target application vulnerabilities
- **Resource Exhaustion**: Consume computational resources

## AWS Shield Standard

### What's Included

#### Automatic Protection
- **Always-on protection** for all AWS customers
- **No additional cost** - included with AWS services
- **Layer 3/4 protection** against common attacks
- **Automatic scaling** with AWS Global infrastructure

#### Protected Services
```
✓ Amazon CloudFront
✓ Amazon Route 53
✓ AWS Global Accelerator
✓ Elastic Load Balancing (ALB, NLB, CLB)
✓ Amazon EC2 Elastic IP addresses
```

### Protection Capabilities

#### Network Layer Protection
```bash
# Automatic mitigation of:
- SYN/ACK Floods
- UDP Reflection attacks
- DNS Amplification
- ICMP Floods
- IP Protocol attacks
```

#### Bandwidth Protection
- **Infinite scaling capacity** via AWS edge locations
- **Automatic traffic filtering** at AWS edge
- **Geographic distribution** of attack mitigation
- **No bandwidth limits** for protected resources

### Standard Protection Features

#### Detection and Mitigation
```
Detection Time: < 1 second
Mitigation Time: < 1 second
Coverage: Layer 3/4 attacks
Scope: Common DDoS patterns
```

#### Monitoring
- **Basic CloudWatch metrics** for protected resources
- **DDoS attack notifications** via AWS Health Dashboard
- **Limited visibility** into attack details
- **No cost protection metrics**

#### Limitations
- **No application layer protection** (Layer 7)
- **No access to DDoS Response Team (DRT)**
- **Limited attack visibility** and reporting
- **No cost protection** for scaling during attacks

## AWS Shield Advanced

### Enhanced Protection Features

#### Comprehensive Coverage
```
✓ All Shield Standard protections
✓ Enhanced DDoS protection for EC2, ELB, CloudFront, Route 53
✓ Application layer (Layer 7) protection
✓ Protection against larger and more sophisticated attacks
✓ Advanced attack diagnostics
```

#### Additional Protected Services
```
✓ Amazon CloudFront distributions
✓ Amazon Route 53 hosted zones
✓ AWS Global Accelerator accelerators
✓ Elastic Load Balancers (ALB, NLB, CLB)
✓ Amazon EC2 Elastic IP addresses
✓ Amazon EC2 instances
```

### Advanced Capabilities

#### Enhanced Detection
```json
{
    "DetectionCapabilities": {
        "LayerCoverage": "Layer 3, 4, and 7",
        "AttackVectors": "All known DDoS attack patterns",
        "DetectionLatency": "Near real-time",
        "FalsePositiveRate": "Minimized through ML algorithms"
    }
}
```

#### 24/7 DDoS Response Team (DRT)
```bash
# DRT Services Include:
- 24/7 access to DDoS experts
- Attack analysis and mitigation guidance  
- Emergency contact during attacks
- Post-incident analysis and reporting
- Proactive monitoring and alerting
```

#### Advanced Attack Diagnostics
```json
{
    "AttackDiagnostics": {
        "RealTimeMetrics": "Detailed attack vectors and volumes",
        "HistoricalData": "Attack trends and patterns",
        "AttackClassification": "Automatic categorization",
        "ImpactAssessment": "Business impact analysis"
    }
}
```

## Feature Comparison

### Protection Scope
| Feature | Shield Standard | Shield Advanced |
|---------|----------------|-----------------|
| **Layer 3/4 Protection** | ✓ Basic | ✓ Enhanced |
| **Layer 7 Protection** | ✗ | ✓ |
| **Attack Size Mitigation** | Common attacks | All sizes |
| **Sophisticated Attacks** | Limited | ✓ |
| **Custom Protections** | ✗ | ✓ |

### Support and Response
| Feature | Shield Standard | Shield Advanced |
|---------|----------------|-----------------|
| **DDoS Response Team** | ✗ | ✓ 24/7 |
| **Emergency Support** | Standard AWS Support | Immediate DRT access |
| **Attack Analysis** | Basic | Detailed forensics |
| **Mitigation Guidance** | Self-service | Expert assistance |
| **Proactive Monitoring** | ✗ | ✓ |

### Cost Protection
| Feature | Shield Standard | Shield Advanced |
|---------|----------------|-----------------|
| **Scaling Cost Protection** | ✗ | ✓ |
| **Data Transfer Protection** | ✗ | ✓ |
| **EC2 Instance Protection** | ✗ | ✓ |
| **Cost Coverage** | None | DDoS-related costs |

### Monitoring and Reporting
| Feature | Shield Standard | Shield Advanced |
|---------|----------------|-----------------|
| **CloudWatch Metrics** | Basic | Enhanced |
| **Attack Visibility** | Limited | Comprehensive |
| **Real-time Reporting** | ✗ | ✓ |
| **Historical Analysis** | ✗ | ✓ |
| **Custom Dashboards** | ✗ | ✓ |

## Shield Advanced Configuration

### Subscription Setup

#### Enable Shield Advanced
```bash
# Subscribe to Shield Advanced
aws shield subscribe-to-proactive-engagement \
    --proactive-engagement-status ENABLED \
    --emergency-contact-list ContactNotes="Primary security contact",Email="security@example.com",PhoneNumber="+1-555-123-4567"
```

#### Configure Protected Resources
```bash
# Add CloudFront distribution
aws shield create-protection \
    --name "CloudFront-Protection" \
    --resource-arn "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"

# Add Application Load Balancer
aws shield create-protection \
    --name "ALB-Protection" \
    --resource-arn "arn:aws:elasticloadbalancing:us-west-2:123456789012:loadbalancer/app/my-alb/50dc6c495c0c9188"

# Add Elastic IP
aws shield create-protection \
    --name "EIP-Protection" \
    --resource-arn "arn:aws:ec2:us-west-2:123456789012:eip-allocation/eipalloc-12345678"
```

### Emergency Contact Configuration

#### Set Up Emergency Contacts
```bash
# Configure emergency contacts for DRT
aws shield put-subscription \
    --subscription '{
        "AutoRenew": "ENABLED",
        "ProactiveEngagementStatus": "ENABLED"
    }'

aws shield update-emergency-contact-settings \
    --emergency-contact-list '[
        {
            "ContactNotes": "Primary Security Team",
            "Email": "security-team@example.com",
            "PhoneNumber": "+1-555-123-4567"
        },
        {
            "ContactNotes": "Secondary On-Call",
            "Email": "oncall@example.com", 
            "PhoneNumber": "+1-555-987-6543"
        }
    ]'
```

### Health-Based Detection

#### Configure Health Checks
```bash
# Create Route 53 health check
aws route53 create-health-check \
    --caller-reference "shield-health-check-$(date +%s)" \
    --health-check-config '{
        "Type": "HTTPS",
        "ResourcePath": "/health",
        "FullyQualifiedDomainName": "api.example.com",
        "Port": 443,
        "RequestInterval": 30,
        "FailureThreshold": 3
    }'

# Associate health check with Shield Advanced
aws shield associate-health-check \
    --protection-id "12345678-1234-1234-1234-123456789012" \
    --health-check-arn "arn:aws:route53:::healthcheck/Z123456789012"
```

### Proactive Engagement

#### Enable Proactive Engagement
```json
{
    "ProactiveEngagementConfig": {
        "ProactiveEngagementStatus": "ENABLED",
        "EmergencyContactList": [
            {
                "ContactNotes": "Primary security contact",
                "Email": "security@example.com",
                "PhoneNumber": "+1-555-123-4567"
            }
        ]
    }
}
```

## Cost Protection Features

### DDoS Cost Protection

#### How It Works
```
Normal Traffic → Standard AWS charges
DDoS Attack Traffic → No additional charges for:
├── Data Transfer costs
├── EC2 instance hours  
├── Auto Scaling charges
└── CloudFront bandwidth
```

#### Coverage Details
```bash
# Protected Cost Categories:
- EC2 instance hours during scaling events
- Data transfer costs from DDoS traffic
- CloudFront bandwidth overages
- Route 53 query costs
- Global Accelerator data transfer
```

#### Cost Protection Request Process
```bash
# Request cost protection credit
aws support create-case \
    --subject "Shield Advanced Cost Protection Request" \
    --service-code "shield-advanced" \
    --severity-code "high" \
    --category-code "ddos-cost-protection" \
    --communication-body "Requesting cost protection for DDoS attack on [date]"
```

## Monitoring and Alerting

### Enhanced CloudWatch Metrics

#### Shield Advanced Metrics
```bash
# Available metrics in AWS/DDoSProtection namespace:
- DDoSDetected (binary indicator)
- DDoSAttackBitsPerSecond
- DDoSAttackPacketsPerSecond  
- DDoSAttackRequestsPerSecond
```

#### Custom Alarms
```bash
# Create DDoS detection alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "DDoS-Attack-Detected" \
    --alarm-description "Alert when DDoS attack is detected" \
    --metric-name DDoSDetected \
    --namespace AWS/DDoSProtection \
    --statistic Maximum \
    --period 60 \
    --threshold 1 \
    --comparison-operator GreaterThanOrEqualToThreshold \
    --evaluation-periods 1 \
    --alarm-actions "arn:aws:sns:us-west-2:123456789012:ddos-alerts"
```

### Global Threat Environment Dashboard

#### Real-time Visibility
```json
{
    "DashboardFeatures": {
        "AttackVectors": "Real-time attack classification",
        "AttackVolume": "Traffic volume and patterns",
        "MitigationStatus": "Active protection measures",
        "GeographicDistribution": "Attack source locations",
        "TimelineAnalysis": "Attack duration and evolution"
    }
}
```

## Integration with Other Services

### WAF Integration

#### Enhanced Layer 7 Protection
```bash
# Combine Shield Advanced with WAF
aws wafv2 associate-web-acl \
    --web-acl-arn "arn:aws:wafv2:us-east-1:123456789012:global/webacl/DDoSProtectionWAF/12345678" \
    --resource-arn "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
```

#### Rate-Based Rules for DDoS
```json
{
    "RateBasedStatement": {
        "Limit": 2000,
        "AggregateKeyType": "IP",
        "ScopeDownStatement": {
            "NotStatement": {
                "Statement": {
                    "IPSetReferenceStatement": {
                        "ARN": "arn:aws:wafv2:us-east-1:123456789012:global/ipset/trusted-ips/12345678"
                    }
                }
            }
        }
    }
}
```

### CloudFront Integration

#### Advanced Edge Protection
```json
{
    "DistributionConfig": {
        "Origins": [
            {
                "Id": "origin1",
                "DomainName": "origin.example.com",
                "CustomOriginConfig": {
                    "OriginShieldEnabled": true,
                    "OriginShieldRegion": "us-east-1"
                }
            }
        ],
        "DefaultCacheBehavior": {
            "TrustedSigners": {
                "Enabled": false,
                "Quantity": 0
            },
            "ViewerProtocolPolicy": "redirect-to-https"
        }
    }
}
```

## Incident Response Procedures

### During an Attack

#### Immediate Actions
```bash
# 1. Verify attack detection
aws shield describe-attack \
    --resource-arn "arn:aws:elasticloadbalancing:us-west-2:123456789012:loadbalancer/app/my-alb/50dc6c495c0c9188"

# 2. Contact DRT if needed
# Call Shield Advanced emergency number
# Reference: Your AWS account ID and resource ARN

# 3. Monitor attack metrics
aws cloudwatch get-metric-statistics \
    --namespace AWS/DDoSProtection \
    --metric-name DDoSAttackBitsPerSecond \
    --start-time 2023-01-01T12:00:00Z \
    --end-time 2023-01-01T13:00:00Z \
    --period 300 \
    --statistics Maximum
```

#### Post-Attack Analysis
```bash
# Generate attack report
aws shield describe-attack \
    --resource-arn "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE" \
    --attack-id "AttackId-12345678-1234-1234-1234-123456789012"

# Review cost impact
aws ce get-cost-and-usage \
    --time-period Start=2023-01-01,End=2023-01-02 \
    --granularity DAILY \
    --metrics BlendedCost \
    --group-by Type=DIMENSION,Key=SERVICE
```

## Cost Analysis

### Shield Advanced Pricing

#### Subscription Costs
```
Monthly Fee: $3,000 USD per organization
Data Transfer: $0.01 per GB for protected resources
Usage Fees: Based on protected resource usage
```

#### Cost Justification Scenarios
```bash
# High-value applications
Critical business applications
E-commerce platforms during peak seasons
Financial services applications
Healthcare systems with uptime requirements

# Cost-benefit analysis
Monthly Protection Cost: $3,000
Potential Downtime Cost: $100,000+ per hour
Break-even: 1.8 minutes of downtime prevention per month
```

### ROI Calculation

#### Business Impact Assessment
```python
def calculate_shield_advanced_roi():
    # Inputs
    monthly_subscription = 3000  # USD
    annual_subscription = monthly_subscription * 12
    
    # Potential losses
    hourly_revenue_loss = 50000  # USD per hour
    reputation_damage = 100000  # USD per incident
    compliance_penalties = 250000  # USD per incident
    
    # Risk reduction
    attack_probability_without = 0.3  # 30% chance per year
    attack_probability_with = 0.05   # 5% chance per year
    average_downtime_hours = 4
    
    # Calculate ROI
    potential_annual_loss_without = (
        attack_probability_without * 
        (hourly_revenue_loss * average_downtime_hours + 
         reputation_damage + compliance_penalties)
    )
    
    potential_annual_loss_with = (
        attack_probability_with * 
        (hourly_revenue_loss * average_downtime_hours + 
         reputation_damage + compliance_penalties)
    )
    
    annual_savings = potential_annual_loss_without - potential_annual_loss_with
    roi = ((annual_savings - annual_subscription) / annual_subscription) * 100
    
    return {
        'annual_savings': annual_savings,
        'annual_cost': annual_subscription,
        'roi_percentage': roi
    }
```

## Best Practices

### When to Use Shield Advanced

#### Strong Indicators
- **Business-critical applications** with high availability requirements
- **Public-facing services** with known DDoS risk
- **Compliance requirements** for DDoS protection
- **High traffic applications** with scaling costs
- **Previous DDoS attack** experience

#### Consider Shield Standard When
- **Development/test environments** with lower criticality
- **Internal applications** with limited internet exposure
- **Cost-sensitive deployments** with basic protection needs
- **Applications behind multiple layers** of protection

### Implementation Strategies

#### Phased Deployment
```bash
# Phase 1: Critical resources
Enable Shield Advanced for:
- Production CloudFront distributions
- Primary load balancers
- Customer-facing applications

# Phase 2: Secondary resources  
Extend protection to:
- Staging environments
- API endpoints
- Administrative interfaces

# Phase 3: Comprehensive coverage
Include all eligible resources:
- Development environments
- Internal applications
- Backup systems
```

#### Integration Planning
1. **Baseline Current Protection**: Assess existing DDoS mitigation
2. **Identify Critical Resources**: Prioritize high-value assets
3. **Configure Monitoring**: Set up comprehensive alerting
4. **Test Response Procedures**: Validate incident response
5. **Train Staff**: Ensure team readiness

## Conclusion

Choosing between AWS Shield Standard and Advanced depends on your application criticality, risk tolerance, and budget. Shield Standard provides solid baseline protection for all AWS customers, while Shield Advanced offers enterprise-grade DDoS protection with expert support and cost protection for mission-critical applications.