# Topic 43: AWS Network Firewall: Setup and Rules

## Introduction

AWS Network Firewall is a managed firewall service that provides fine-grained control over network traffic at the VPC level. It offers stateful inspection, intrusion detection and prevention, and web filtering capabilities to protect your AWS workloads.

## AWS Network Firewall Overview

### What is AWS Network Firewall?

**AWS Network Firewall** is a stateful, managed firewall service that uses rules to filter traffic entering and leaving your VPC. It provides:

- **Stateful packet inspection**
- **Application-level gateway filtering** 
- **Intrusion detection and prevention (IDS/IPS)**
- **Web filtering and domain blocking**
- **Custom rule creation and management**

### Key Features
- **Managed Infrastructure**: Fully managed by AWS
- **High Availability**: Automatically scales across AZs
- **Deep Packet Inspection**: Layer 3-7 traffic analysis
- **Flexible Rules**: Custom and AWS managed rule groups
- **Integration**: Works with AWS services and third-party tools
- **Compliance**: Supports various compliance standards

## Architecture and Components

### Core Components

#### 1. Firewall
- **Central resource** that contains firewall policy
- **Deployed per VPC** with multi-AZ availability
- **Processing capacity** scales automatically
- **Logging and monitoring** integrated

#### 2. Firewall Policy
- **Collection of rule groups** and policy settings
- **Stateful and stateless** rule group associations
- **Traffic flow configuration** (strict or loose)
- **Logging configuration** for monitoring

#### 3. Rule Groups
- **Stateful Rule Groups**: Connection tracking and application awareness
- **Stateless Rule Groups**: Fast packet filtering without state
- **Managed Rule Groups**: AWS-provided security rules
- **Custom Rule Groups**: User-defined filtering rules

#### 4. Firewall Endpoints
- **VPC endpoints** for firewall processing
- **One per Availability Zone** for high availability
- **Traffic routing** through firewall subnets
- **Elastic network interfaces** for connectivity

### Traffic Flow Architecture

```
Internet → IGW → Firewall Subnet → Protected Subnet → Application
                      ↓
               Network Firewall
                 (Rule Processing)
```

#### Traffic Inspection Points
1. **North-South Traffic**: Internet to VPC and vice versa
2. **East-West Traffic**: VPC to VPC communication
3. **Outbound Traffic**: VPC to internet egress
4. **Inbound Traffic**: Internet to VPC ingress

## Setup and Deployment

### Prerequisites

#### IAM Permissions
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "network-firewall:*",
                "ec2:*",
                "logs:*",
                "s3:*"
            ],
            "Resource": "*"
        }
    ]
}
```

#### VPC Requirements
- **Dedicated subnets** for firewall endpoints
- **Route table modifications** for traffic routing
- **Sufficient IP addresses** in firewall subnets
- **Multi-AZ deployment** for high availability

### Step-by-Step Setup

#### Step 1: Create Firewall Subnets
```bash
# Create dedicated subnets for Network Firewall
aws ec2 create-subnet \
    --vpc-id vpc-12345678 \
    --cidr-block 10.0.1.0/28 \
    --availability-zone us-west-2a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Firewall-Subnet-AZ1}]'

aws ec2 create-subnet \
    --vpc-id vpc-12345678 \
    --cidr-block 10.0.2.0/28 \
    --availability-zone us-west-2b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Firewall-Subnet-AZ2}]'
```

#### Step 2: Create Rule Groups
```bash
# Create stateless rule group
aws network-firewall create-rule-group \
    --rule-group-name "StatelessRuleGroup" \
    --type STATELESS \
    --capacity 100 \
    --rule-group '{
        "RulesSource": {
            "StatelessRulesAndCustomActions": {
                "StatelessRules": [
                    {
                        "RuleDefinition": {
                            "MatchAttributes": {
                                "Sources": [{"AddressDefinition": "0.0.0.0/0"}],
                                "Destinations": [{"AddressDefinition": "10.0.0.0/16"}],
                                "DestinationPorts": [{"FromPort": 80, "ToPort": 80}],
                                "Protocols": [6]
                            },
                            "Actions": ["aws:pass"]
                        },
                        "Priority": 1
                    }
                ]
            }
        }
    }'
```

#### Step 3: Create Firewall Policy
```bash
# Create firewall policy
aws network-firewall create-firewall-policy \
    --firewall-policy-name "MyFirewallPolicy" \
    --firewall-policy '{
        "StatelessDefaultActions": ["aws:forward_to_sfe"],
        "StatelessFragmentDefaultActions": ["aws:forward_to_sfe"],
        "StatelessRuleGroupReferences": [
            {
                "ResourceArn": "arn:aws:network-firewall:us-west-2:123456789012:stateless-rulegroup/StatelessRuleGroup",
                "Priority": 1
            }
        ]
    }'
```

#### Step 4: Create Firewall
```bash
# Create the firewall
aws network-firewall create-firewall \
    --firewall-name "MyNetworkFirewall" \
    --firewall-policy-arn "arn:aws:network-firewall:us-west-2:123456789012:firewall-policy/MyFirewallPolicy" \
    --vpc-id vpc-12345678 \
    --subnet-mappings SubnetId=subnet-12345678,SubnetId=subnet-87654321 \
    --tags Key=Name,Value=Production-Firewall
```

### Route Table Configuration

#### Internet Gateway Route Table
```bash
# Route internet traffic through firewall
aws ec2 create-route \
    --route-table-id rtb-12345678 \
    --destination-cidr-block 10.0.0.0/16 \
    --vpc-endpoint-id vpce-firewall-endpoint
```

#### Protected Subnet Route Table
```bash
# Route outbound traffic through firewall
aws ec2 create-route \
    --route-table-id rtb-87654321 \
    --destination-cidr-block 0.0.0.0/0 \
    --vpc-endpoint-id vpce-firewall-endpoint
```

#### Firewall Subnet Route Table
```bash
# Route to Internet Gateway for outbound traffic
aws ec2 create-route \
    --route-table-id rtb-firewall \
    --destination-cidr-block 0.0.0.0/0 \
    --gateway-id igw-12345678
```

## Rule Groups Configuration

### Stateless Rule Groups

#### Basic Packet Filtering
```json
{
    "RulesSource": {
        "StatelessRulesAndCustomActions": {
            "StatelessRules": [
                {
                    "RuleDefinition": {
                        "MatchAttributes": {
                            "Sources": [{"AddressDefinition": "192.168.1.0/24"}],
                            "Destinations": [{"AddressDefinition": "10.0.0.0/16"}],
                            "DestinationPorts": [
                                {"FromPort": 443, "ToPort": 443}
                            ],
                            "Protocols": [6],
                            "TCPFlags": [
                                {
                                    "Flags": ["SYN"],
                                    "Masks": ["SYN", "ACK"]
                                }
                            ]
                        },
                        "Actions": ["aws:pass"]
                    },
                    "Priority": 1
                }
            ]
        }
    }
}
```

#### Custom Actions
```json
{
    "CustomActions": [
        {
            "ActionName": "LogAndAlert",
            "ActionDefinition": {
                "PublishMetricAction": {
                    "Dimensions": [
                        {
                            "Key": "CustomDimension",
                            "Value": "SuspiciousTraffic"
                        }
                    ]
                }
            }
        }
    ]
}
```

### Stateful Rule Groups

#### 5-Tuple Rules
```json
{
    "RulesSource": {
        "StatefulRules": [
            {
                "Action": "PASS",
                "Header": {
                    "Protocol": "TCP",
                    "Source": "10.0.0.0/16",
                    "SourcePort": "ANY",
                    "Direction": "ANY",
                    "Destination": "ANY",
                    "DestinationPort": "443"
                },
                "RuleOptions": [
                    {
                        "Keyword": "sid",
                        "Settings": ["1"]
                    }
                ]
            }
        ]
    }
}
```

#### Suricata Rules
```bash
# HTTP inspection rule
alert http any any -> any any (msg:"Suspicious User Agent"; content:"malware"; http_user_agent; sid:1001;)

# DNS inspection rule
alert dns any any -> any any (msg:"Suspicious DNS Query"; content:"malicious.com"; sid:1002;)

# TLS inspection rule
alert tls any any -> any any (msg:"Expired Certificate"; tls.cert_expired; sid:1003;)
```

#### Domain Filtering
```json
{
    "RulesSource": {
        "RulesSourceList": {
            "Targets": [
                "malicious-domain.com",
                "phishing-site.net",
                "*.suspicious.org"
            ],
            "TargetTypes": ["HTTP_HOST", "TLS_SNI"],
            "GeneratedRulesType": "DENYLIST"
        }
    }
}
```

### AWS Managed Rule Groups

#### Available Managed Rules
```bash
# List available managed rule groups
aws network-firewall describe-managed-rule-groups \
    --type STATEFUL \
    --region us-west-2

# Common managed rule groups:
# - AbuseCH-MalwareDomainList
# - BotNetCommandAndControlDomains
# - MalwareDomainsAndIps
# - ThreatIntelligenceBlocklist
```

#### Using Managed Rules in Policy
```json
{
    "StatefulManagedRuleGroupReferences": [
        {
            "ResourceArn": "arn:aws:network-firewall::aws-managed-rule-group:AbuseCH-MalwareDomainList",
            "Override": {
                "Action": "DROP_TO_ALERT"
            }
        }
    ]
}
```

## Advanced Configuration

### High Availability Setup

#### Multi-AZ Deployment
```bash
# Deploy firewall across multiple AZs
SUBNET_MAPPINGS="[
    {\"SubnetId\":\"subnet-1a\"},
    {\"SubnetId\":\"subnet-1b\"},
    {\"SubnetId\":\"subnet-1c\"}
]"

aws network-firewall create-firewall \
    --firewall-name "HA-NetworkFirewall" \
    --firewall-policy-arn $POLICY_ARN \
    --vpc-id vpc-12345678 \
    --subnet-mappings $SUBNET_MAPPINGS
```

#### Capacity Planning
```bash
# Configure processing capacity
aws network-firewall modify-firewall \
    --firewall-name "MyNetworkFirewall" \
    --firewall-policy-arn $POLICY_ARN \
    --subnet-mappings SubnetId=subnet-12345678,IPAddressType=IPV4
```

### Performance Optimization

#### Rule Optimization
1. **Order Rules by Priority**: Most specific rules first
2. **Use Stateless for Simple Filtering**: Better performance
3. **Optimize Suricata Rules**: Efficient pattern matching
4. **Limit Rule Complexity**: Balance security and performance

#### Traffic Flow Optimization
```bash
# Strict order evaluation (better security)
"StatefulEngineOptions": {
    "RuleOrder": "STRICT_ORDER"
}

# Default action evaluation (better performance)
"StatefulEngineOptions": {
    "RuleOrder": "DEFAULT_ACTION_ORDER"
}
```

## Monitoring and Logging

### CloudWatch Metrics

#### Key Metrics
```bash
# Firewall metrics
AWS/NetworkFirewall/DroppedPackets
AWS/NetworkFirewall/PassedPackets
AWS/NetworkFirewall/RejectedPackets
AWS/NetworkFirewall/ReceivedPackets

# Performance metrics
AWS/NetworkFirewall/Utilization
AWS/NetworkFirewall/Capacity
```

#### Custom Metrics from Rules
```json
{
    "RuleOptions": [
        {
            "Keyword": "sid",
            "Settings": ["1001"]
        },
        {
            "Keyword": "aws:publish_metric",
            "Settings": ["SecurityAlerts"]
        }
    ]
}
```

### Logging Configuration

#### Flow Logs
```bash
# Enable flow logging
aws network-firewall modify-logging-configuration \
    --firewall-name "MyNetworkFirewall" \
    --logging-configuration '{
        "LogDestinationConfigs": [
            {
                "LogType": "FLOW",
                "LogDestinationType": "CloudWatchLogs",
                "LogDestination": {
                    "logGroup": "/aws/networkfirewall/flow"
                }
            }
        ]
    }'
```

#### Alert Logs
```bash
# Enable alert logging
aws network-firewall modify-logging-configuration \
    --firewall-name "MyNetworkFirewall" \
    --logging-configuration '{
        "LogDestinationConfigs": [
            {
                "LogType": "ALERT",
                "LogDestinationType": "S3",
                "LogDestination": {
                    "bucketName": "network-firewall-alerts",
                    "prefix": "alerts/"
                }
            }
        ]
    }'
```

## Troubleshooting

### Common Issues

#### Traffic Not Being Inspected
1. **Route Table Configuration**: Verify traffic routes through firewall
2. **Subnet Selection**: Ensure correct firewall subnet placement
3. **Security Groups**: Check that traffic is allowed to reach firewall

#### Rules Not Working
1. **Rule Syntax**: Validate Suricata rule syntax
2. **Rule Order**: Check stateful rule processing order
3. **Rule Priority**: Verify stateless rule priorities

#### Performance Issues
1. **Capacity Limits**: Monitor utilization metrics
2. **Rule Complexity**: Optimize heavy inspection rules
3. **Traffic Patterns**: Analyze flow logs for bottlenecks

### Debugging Commands

#### Check Firewall Status
```bash
# Get firewall details
aws network-firewall describe-firewall \
    --firewall-name "MyNetworkFirewall"

# Check rule group status
aws network-firewall describe-rule-group \
    --rule-group-name "MyRuleGroup"
```

#### Analyze Logs
```bash
# Query flow logs
aws logs filter-log-events \
    --log-group-name "/aws/networkfirewall/flow" \
    --start-time 1609459200000 \
    --filter-pattern "REJECT"

# Query alert logs
aws s3 cp s3://network-firewall-alerts/alerts/ . --recursive
```

## Cost Optimization

### Pricing Components
- **Firewall Endpoints**: $0.395 per hour per endpoint
- **Data Processing**: $0.065 per GB processed
- **Rule Groups**: Included in endpoint cost

### Cost Optimization Strategies
1. **Right-size Deployment**: Deploy only needed endpoints
2. **Optimize Rules**: Reduce unnecessary deep inspection
3. **Use Stateless Rules**: Lower cost for simple filtering
4. **Monitor Usage**: Track data processing costs

## Best Practices

### Security Best Practices
1. **Defense in Depth**: Combine with Security Groups and NACLs
2. **Regular Updates**: Keep managed rules updated
3. **Custom Rules**: Tailor rules to specific threats
4. **Monitoring**: Continuous security monitoring

### Operational Best Practices
1. **Version Control**: Track rule changes
2. **Testing**: Test rules in development environment
3. **Documentation**: Document rule purposes
4. **Automation**: Automate rule deployment

### Performance Best Practices
1. **Rule Optimization**: Order rules by likelihood
2. **Capacity Planning**: Monitor utilization
3. **Traffic Analysis**: Understand traffic patterns
4. **Regular Review**: Periodically review rules

## Conclusion

AWS Network Firewall provides comprehensive network security capabilities for protecting VPC resources. Proper setup, rule configuration, and ongoing management ensure effective security while maintaining optimal performance and cost efficiency.