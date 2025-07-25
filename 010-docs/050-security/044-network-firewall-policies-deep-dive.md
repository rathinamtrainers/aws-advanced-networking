# Topic 44: Deep Dive into AWS Network Firewall Policies

## Introduction

AWS Network Firewall policies define how traffic is processed and filtered through your firewall. Understanding policy structure, rule evaluation, and advanced configurations is crucial for implementing effective network security.

## Policy Architecture Overview

### Policy Components Hierarchy
```
Firewall Policy
├── Stateless Rule Groups (Fast Path)
│   ├── Custom Stateless Rules
│   └── Default Actions
├── Stateful Rule Groups (Deep Inspection)
│   ├── Custom Stateful Rules
│   ├── Managed Rule Groups
│   └── Domain Lists
└── Policy Settings
    ├── Rule Order
    ├── Fragment Handling
    └── Engine Options
```

### Traffic Processing Flow
```
Incoming Traffic
    ↓
Stateless Rule Groups (Layer 3/4)
    ↓
Stateful Rule Groups (Layer 3-7)
    ↓
Default Actions
    ↓
Outgoing Traffic/Drop/Alert
```

## Stateless Rule Groups Deep Dive

### Rule Structure and Components

#### Match Criteria
```json
{
    "MatchAttributes": {
        "Sources": [
            {"AddressDefinition": "192.168.1.0/24"},
            {"AddressDefinition": "10.0.0.0/8"}
        ],
        "Destinations": [
            {"AddressDefinition": "172.16.0.0/12"}
        ],
        "SourcePorts": [
            {"FromPort": 1024, "ToPort": 65535}
        ],
        "DestinationPorts": [
            {"FromPort": 80, "ToPort": 80},
            {"FromPort": 443, "ToPort": 443}
        ],
        "Protocols": [6, 17],
        "TCPFlags": [
            {
                "Flags": ["SYN"],
                "Masks": ["SYN", "ACK"]
            }
        ]
    }
}
```

#### Actions and Priorities
```json
{
    "StatelessRules": [
        {
            "RuleDefinition": {
                "MatchAttributes": { /* match criteria */ },
                "Actions": ["aws:pass"]
            },
            "Priority": 1
        },
        {
            "RuleDefinition": {
                "MatchAttributes": { /* match criteria */ },
                "Actions": ["aws:drop"]
            },
            "Priority": 10
        },
        {
            "RuleDefinition": {
                "MatchAttributes": { /* match criteria */ },
                "Actions": ["CustomAction"]
            },
            "Priority": 5
        }
    ]
}
```

### Custom Actions

#### Metric Publishing
```json
{
    "CustomActions": [
        {
            "ActionName": "PublishSecurityMetrics",
            "ActionDefinition": {
                "PublishMetricAction": {
                    "Dimensions": [
                        {
                            "Key": "SourceIP",
                            "Value": "${source_ip}"
                        },
                        {
                            "Key": "Protocol", 
                            "Value": "${protocol}"
                        },
                        {
                            "Key": "ThreatLevel",
                            "Value": "High"
                        }
                    ]
                }
            }
        }
    ]
}
```

#### Complex Action Chains
```json
{
    "ActionName": "LogAndForward",
    "ActionDefinition": {
        "PublishMetricAction": {
            "Dimensions": [
                {
                    "Key": "Action",
                    "Value": "InspectAndLog"
                }
            ]
        }
    }
}
```

## Stateful Rule Groups Deep Dive

### 5-Tuple Stateful Rules

#### Basic Stateful Rule
```json
{
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
                    "Settings": ["1001"]
                },
                {
                    "Keyword": "msg",
                    "Settings": ["HTTPS traffic from internal network"]
                }
            ]
        }
    ]
}
```

#### Advanced Rule Options
```json
{
    "RuleOptions": [
        {
            "Keyword": "sid",
            "Settings": ["2001"]
        },
        {
            "Keyword": "msg", 
            "Settings": ["Database connection monitoring"]
        },
        {
            "Keyword": "flow",
            "Settings": ["to_server,established"]
        },
        {
            "Keyword": "content",
            "Settings": ["SELECT * FROM users"]
        },
        {
            "Keyword": "nocase",
            "Settings": []
        }
    ]
}
```

### Suricata Rule Language

#### HTTP Traffic Inspection
```bash
# Detect SQL injection attempts
alert http any any -> any any (
    msg:"SQL Injection Attempt";
    content:"union select";
    nocase;
    http_uri;
    sid:3001;
    rev:1;
)

# Monitor large file uploads
alert http any any -> any any (
    msg:"Large File Upload";
    content:"Content-Length:";
    http_header;
    content:"|20|10000000";
    distance:0;
    within:20;
    sid:3002;
)

# Detect suspicious user agents
alert http any any -> any any (
    msg:"Suspicious Bot User Agent";
    content:"bot";
    nocase;
    http_user_agent;
    sid:3003;
)
```

#### DNS Traffic Monitoring
```bash
# Detect DNS tunneling
alert dns any any -> any any (
    msg:"DNS Tunneling Suspected";
    dns.query;
    content:".";
    within:255;
    pcre:"/^[a-f0-9]{30,}/";
    sid:4001;
)

# Monitor DGA domains
alert dns any any -> any any (
    msg:"DGA Domain Pattern";
    dns.query;
    pcre:"/^[a-z0-9]{8,20}\.(com|net|org)$/";
    sid:4002;
)

# Detect C2 communications
alert dns any any -> any any (
    msg:"Known C2 Domain";
    dns.query;
    content:"malware.example.com";
    sid:4003;
)
```

#### TLS/SSL Inspection
```bash
# Expired certificate detection
alert tls any any -> any any (
    msg:"Expired TLS Certificate";
    tls.cert_expired;
    sid:5001;
)

# Self-signed certificate alert
alert tls any any -> any any (
    msg:"Self-Signed Certificate";
    tls.cert_subject;
    content:"Self";
    sid:5002;
)

# Monitor cipher suites
alert tls any any -> any any (
    msg:"Weak Cipher Suite";
    tls.version:1.0;
    sid:5003;
)
```

### Domain List Rules

#### Allow List Configuration
```json
{
    "RulesSource": {
        "RulesSourceList": {
            "Targets": [
                "amazonaws.com",
                "amazon.com",
                "*.s3.amazonaws.com",
                "github.com",
                "*.github.io"
            ],
            "TargetTypes": ["HTTP_HOST", "TLS_SNI"],
            "GeneratedRulesType": "ALLOWLIST"
        }
    }
}
```

#### Deny List Configuration
```json
{
    "RulesSource": {
        "RulesSourceList": {
            "Targets": [
                "malicious-domain.com",
                "*.phishing.net", 
                "suspicious.org",
                "*.torrent-site.com"
            ],
            "TargetTypes": ["HTTP_HOST", "TLS_SNI"],
            "GeneratedRulesType": "DENYLIST"
        }
    }
}
```

## Policy Configuration

### Basic Policy Structure
```json
{
    "StatelessDefaultActions": ["aws:forward_to_sfe"],
    "StatelessFragmentDefaultActions": ["aws:forward_to_sfe"],
    "StatefulDefaultActions": ["aws:drop_strict"],
    "StatelessRuleGroupReferences": [
        {
            "ResourceArn": "arn:aws:network-firewall:region:account:stateless-rulegroup/basic-filtering",
            "Priority": 1
        }
    ],
    "StatefulRuleGroupReferences": [
        {
            "ResourceArn": "arn:aws:network-firewall:region:account:stateful-rulegroup/threat-detection",
            "Priority": 1
        }
    ]
}
```

### Advanced Policy Options

#### Stateful Engine Configuration
```json
{
    "StatefulEngineOptions": {
        "RuleOrder": "STRICT_ORDER",
        "StreamExceptionPolicy": "DROP"
    },
    "TLSInspectionConfigurationArn": "arn:aws:network-firewall:region:account:tls-configuration/deep-inspection"
}
```

#### Policy Variables
```json
{
    "PolicyVariables": {
        "RuleVariables": {
            "HOME_NET": {
                "Definition": ["10.0.0.0/16", "192.168.0.0/16"]
            },
            "WEB_SERVERS": {
                "Definition": ["10.0.1.0/24"]
            },
            "DB_SERVERS": {
                "Definition": ["10.0.2.0/24"]
            }
        }
    }
}
```

## Advanced Rule Techniques

### Flow Control and State Tracking

#### Connection State Rules
```bash
# Track connection establishment
alert tcp any any -> $HOME_NET any (
    msg:"New Connection to Internal Network";
    flow:to_server,not_established;
    sid:6001;
)

# Monitor established connections
alert tcp any any -> $HOME_NET any (
    msg:"Data Transfer on Established Connection";
    flow:established;
    dsize:>1000;
    sid:6002;
)

# Detect connection teardown anomalies
alert tcp any any -> any any (
    msg:"Abnormal Connection Termination";
    flags:R+A;
    flow:established;
    sid:6003;
)
```

#### Application Protocol Detection
```bash
# HTTP on non-standard ports
alert tcp any any -> any !80 (
    msg:"HTTP on Non-Standard Port";
    content:"GET ";
    depth:4;
    sid:7001;
)

# SSH brute force detection
alert ssh any any -> $HOME_NET 22 (
    msg:"SSH Brute Force Attempt";
    ssh.software:"OpenSSH";
    threshold:type both,track by_src,count 5,seconds 60;
    sid:7002;
)
```

### Pattern Matching and Content Inspection

#### Advanced Content Matching
```bash
# Multi-pattern content matching
alert http any any -> any any (
    msg:"Multi-Stage Malware Download";
    content:"GET ";
    depth:4;
    content:".exe";
    distance:0;
    within:100;
    content:"User-Agent:";
    distance:0;
    content:"Mozilla";
    within:50;
    sid:8001;
)

# Binary data detection
alert tcp any any -> any any (
    msg:"Suspicious Binary Transfer";
    content:"|4D 5A|";  # MZ header
    offset:0;
    depth:2;
    sid:8002;
)
```

#### PCRE (Perl Compatible Regular Expressions)
```bash
# Credit card number detection
alert tcp any any -> any any (
    msg:"Credit Card Number Detected";
    pcre:"/\b(?:4[0-9]{12}(?:[0-9]{3})?|5[1-5][0-9]{14})\b/";
    sid:9001;
)

# Email address extraction
alert http any any -> any any (
    msg:"Email Address in HTTP Traffic";
    pcre:"/[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}/";
    http_uri;
    sid:9002;
)
```

## Managed Rule Groups Integration

### AWS Managed Rules

#### Threat Intelligence Rules
```json
{
    "StatefulManagedRuleGroupReferences": [
        {
            "ResourceArn": "arn:aws:network-firewall::aws-managed-rule-group:AbuseCH-MalwareDomainList",
            "Override": {
                "Action": "DROP_TO_ALERT"
            }
        },
        {
            "ResourceArn": "arn:aws:network-firewall::aws-managed-rule-group:BotNetCommandAndControlDomains"
        }
    ]
}
```

#### Rule Override Configuration
```json
{
    "Override": {
        "Action": "ALERT",
        "RuleVariables": {
            "HOME_NET": {
                "Definition": ["10.0.0.0/8"]
            }
        }
    }
}
```

### Third-Party Managed Rules

#### Partner Rule Integration
```bash
# Subscribe to partner rule feeds
aws network-firewall put-managed-rule-group \
    --managed-rule-group-name "ProofpointETRules" \
    --vendor-name "Proofpoint" \
    --rule-group-type "STATEFUL"
```

## Performance Optimization

### Rule Ordering Strategies

#### Strict Order Processing
```json
{
    "StatefulEngineOptions": {
        "RuleOrder": "STRICT_ORDER"
    }
}
```

#### Default Action Order
```json
{
    "StatefulEngineOptions": {
        "RuleOrder": "DEFAULT_ACTION_ORDER"
    }
}
```

### Capacity Planning

#### Rule Group Capacity
```bash
# Monitor rule group usage
aws network-firewall describe-rule-group \
    --rule-group-name "MyRuleGroup" \
    --query 'RuleGroup.Capacity'

# Check available capacity
aws cloudwatch get-metric-statistics \
    --namespace AWS/NetworkFirewall \
    --metric-name Capacity \
    --dimensions Name=RuleGroup,Value=MyRuleGroup
```

## Troubleshooting Policies

### Rule Evaluation Issues

#### Debug Rule Matching
```bash
# Enable detailed logging
aws network-firewall modify-logging-configuration \
    --firewall-name "MyFirewall" \
    --logging-configuration '{
        "LogDestinationConfigs": [
            {
                "LogType": "ALERT",
                "LogDestinationType": "CloudWatchLogs",
                "LogDestination": {
                    "logGroup": "/aws/networkfirewall/debug"
                }
            }
        ]
    }'
```

#### Common Rule Problems
1. **Syntax Errors**: Invalid Suricata syntax
2. **Rule Conflicts**: Overlapping or contradictory rules
3. **Performance Issues**: Complex regex patterns
4. **State Tracking**: Incorrect flow keywords

### Policy Validation

#### Validate Policy Syntax
```bash
# Test policy before deployment
aws network-firewall create-firewall-policy \
    --firewall-policy-name "TestPolicy" \
    --firewall-policy file://policy.json \
    --dry-run
```

#### Monitor Rule Performance
```bash
# Check rule hit statistics
aws cloudwatch get-metric-statistics \
    --namespace AWS/NetworkFirewall \
    --metric-name RuleHits \
    --dimensions Name=RuleGroupName,Value=MyRuleGroup
```

## Best Practices

### Policy Design
1. **Start Simple**: Begin with basic rules and add complexity
2. **Test Thoroughly**: Validate rules in development environment
3. **Document Rules**: Maintain clear rule documentation
4. **Version Control**: Track policy changes

### Performance Optimization
1. **Order by Frequency**: Place common rules first
2. **Optimize Patterns**: Use efficient regex patterns
3. **Monitor Metrics**: Track performance metrics
4. **Regular Review**: Periodically review and optimize

### Security Effectiveness
1. **Defense in Depth**: Layer with other security controls
2. **Threat Intelligence**: Integrate current threat feeds
3. **Regular Updates**: Keep rules current with threats
4. **False Positive Management**: Tune rules to reduce noise

## Conclusion

AWS Network Firewall policies provide powerful traffic filtering capabilities through flexible rule configurations. Understanding rule types, evaluation order, and optimization techniques enables effective network security implementation while maintaining optimal performance.