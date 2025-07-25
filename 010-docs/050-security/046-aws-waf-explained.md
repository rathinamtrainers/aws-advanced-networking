# Topic 46: AWS WAF (Web Application Firewall) Explained

## Introduction

AWS WAF is a web application firewall that helps protect web applications from common web exploits and attacks. It provides customizable security rules to filter malicious traffic before it reaches your applications.

## AWS WAF Overview

### What is AWS WAF?

AWS WAF is a **cloud-native web application firewall** that protects web applications from:
- **SQL injection attacks**
- **Cross-site scripting (XSS)**
- **DDoS attacks**
- **Bot traffic and scrapers**
- **IP-based attacks**
- **Geographic restrictions**

### WAF Architecture Components

#### Web ACLs (Access Control Lists)
- **Central configuration** for web application protection
- **Rule evaluation engine** for traffic filtering
- **Default action** for unmatched traffic
- **CloudWatch metrics** for monitoring

#### Rules and Rule Groups
- **Individual rules** for specific attack patterns
- **Rule groups** for organized rule collections
- **Managed rule groups** provided by AWS and partners
- **Custom rules** for application-specific needs

#### Conditions and Statements
- **Match conditions** for traffic analysis
- **Logical operators** for complex rule logic
- **Rate limiting** for traffic control
- **IP reputation** for threat intelligence

## Core Components Deep Dive

### Web ACLs Configuration

#### Basic Web ACL Structure
```json
{
    "Name": "WebApplicationFirewall",
    "Scope": "CLOUDFRONT",
    "DefaultAction": {
        "Allow": {}
    },
    "Rules": [
        {
            "Name": "SQLInjectionRule",
            "Priority": 1,
            "Statement": {
                "SqliMatchStatement": {
                    "FieldToMatch": {
                        "Body": {}
                    },
                    "TextTransformations": [
                        {
                            "Priority": 0,
                            "Type": "URL_DECODE"
                        }
                    ]
                }
            },
            "Action": {
                "Block": {}
            }
        }
    ]
}
```

#### Advanced Web ACL Features
```json
{
    "CaptchaConfig": {
        "ImmunityTimeProperty": {
            "ImmunityTime": 300
        }
    },
    "ChallengeConfig": {
        "ImmunityTimeProperty": {
            "ImmunityTime": 300
        }
    },
    "CustomResponseBodies": {
        "custom-block-response": {
            "ContentType": "TEXT_HTML",
            "Content": "<html><body>Access Denied</body></html>"
        }
    }
}
```

### Rule Types and Conditions

#### IP Set Rules
```json
{
    "IPSetReferenceStatement": {
        "ARN": "arn:aws:wafv2:us-east-1:123456789012:global/ipset/blocked-ips/12345678",
        "IPSetForwardedIPConfig": {
            "HeaderName": "X-Forwarded-For",
            "FallbackBehavior": "MATCH",
            "Position": "FIRST"
        }
    }
}
```

#### Geographic Rules
```json
{
    "GeoMatchStatement": {
        "CountryCodes": ["CN", "RU", "KP"],
        "ForwardedIPConfig": {
            "HeaderName": "X-Forwarded-For",
            "FallbackBehavior": "MATCH"
        }
    }
}
```

#### String Match Rules
```json
{
    "ByteMatchStatement": {
        "SearchString": "malicious-pattern",
        "FieldToMatch": {
            "UriPath": {}
        },
        "TextTransformations": [
            {
                "Priority": 0,
                "Type": "LOWERCASE"
            }
        ],
        "PositionalConstraint": "CONTAINS"
    }
}
```

#### Rate Limiting Rules
```json
{
    "RateBasedStatement": {
        "Limit": 1000,
        "AggregateKeyType": "IP",
        "ScopeDownStatement": {
            "ByteMatchStatement": {
                "SearchString": "/api/",
                "FieldToMatch": {
                    "UriPath": {}
                },
                "TextTransformations": [
                    {
                        "Priority": 0,
                        "Type": "NONE"
                    }
                ],
                "PositionalConstraint": "STARTS_WITH"
            }
        }
    }
}
```

## Managed Rule Groups

### AWS Managed Rules

#### Core Rule Set
```json
{
    "ManagedRuleGroupStatement": {
        "VendorName": "AWS",
        "Name": "AWSManagedRulesCommonRuleSet",
        "ExcludedRules": [
            {
                "Name": "SizeRestrictions_BODY"
            }
        ]
    }
}
```

#### Application-Specific Rules
```json
{
    "ManagedRuleGroupStatement": {
        "VendorName": "AWS",
        "Name": "AWSManagedRulesWordPressRuleSet"
    }
}
```

#### IP Reputation Rules
```json
{
    "ManagedRuleGroupStatement": {
        "VendorName": "AWS", 
        "Name": "AWSManagedRulesAmazonIpReputationList"
    }
}
```

#### Known Bad Inputs
```json
{
    "ManagedRuleGroupStatement": {
        "VendorName": "AWS",
        "Name": "AWSManagedRulesKnownBadInputsRuleSet"
    }
}
```

### Third-Party Managed Rules

#### Fortinet Rules
```bash
# Subscribe to Fortinet managed rules
aws wafv2 get-managed-rule-set \
    --vendor-name "Fortinet" \
    --name "FortinetManagedRulesWebApplicationsRuleSet" \
    --scope CLOUDFRONT
```

#### Imperva Rules
```bash
# Subscribe to Imperva managed rules  
aws wafv2 get-managed-rule-set \
    --vendor-name "Imperva" \
    --name "ImpervaApplicationProtectionRuleSet" \
    --scope REGIONAL
```

## Integration with AWS Services

### CloudFront Integration

#### CloudFront Distribution with WAF
```json
{
    "DistributionConfig": {
        "WebACLId": "arn:aws:wafv2:us-east-1:123456789012:global/webacl/WebApplicationFirewall/12345678",
        "Origins": [
            {
                "Id": "origin1",
                "DomainName": "example.com",
                "CustomOriginConfig": {
                    "HTTPPort": 80,
                    "HTTPSPort": 443,
                    "OriginProtocolPolicy": "https-only"
                }
            }
        ],
        "DefaultCacheBehavior": {
            "TargetOriginId": "origin1",
            "ViewerProtocolPolicy": "redirect-to-https"
        }
    }
}
```

### Application Load Balancer Integration

#### ALB with WAF Configuration
```bash
# Associate WAF with ALB
aws wafv2 associate-web-acl \
    --web-acl-arn "arn:aws:wafv2:us-west-2:123456789012:regional/webacl/WebApplicationFirewall/12345678" \
    --resource-arn "arn:aws:elasticloadbalancing:us-west-2:123456789012:loadbalancer/app/my-alb/1234567890123456"
```

### API Gateway Integration

#### API Gateway with WAF
```bash
# Associate WAF with API Gateway
aws wafv2 associate-web-acl \
    --web-acl-arn "arn:aws:wafv2:us-east-1:123456789012:regional/webacl/ApiGatewayWAF/12345678" \
    --resource-arn "arn:aws:apigateway:us-east-1::/restapis/abc123/stages/prod"
```

## Advanced WAF Features

### CAPTCHA and Challenge Actions

#### CAPTCHA Configuration
```json
{
    "Action": {
        "Captcha": {
            "CustomRequestHandling": {
                "InsertHeaders": [
                    {
                        "Name": "X-Captcha-Challenge",
                        "Value": "required"
                    }
                ]
            }
        }
    }
}
```

#### Challenge Action
```json
{
    "Action": {
        "Challenge": {
            "CustomRequestHandling": {
                "InsertHeaders": [
                    {
                        "Name": "X-Challenge-Required",
                        "Value": "javascript"
                    }
                ]
            }
        }
    }
}
```

### Custom Request/Response Handling

#### Custom Headers
```json
{
    "CustomRequestHandling": {
        "InsertHeaders": [
            {
                "Name": "X-WAF-Token",
                "Value": "verified-user"
            },
            {
                "Name": "X-Security-Level",
                "Value": "high"
            }
        ]
    }
}
```

#### Custom Response
```json
{
    "Action": {
        "Block": {
            "CustomResponse": {
                "ResponseCode": 403,
                "CustomResponseBodyKey": "custom-block-response",
                "ResponseHeaders": [
                    {
                        "Name": "X-Block-Reason",
                        "Value": "Security-Policy-Violation"
                    }
                ]
            }
        }
    }
}
```

### Label-Based Rule Logic

#### Rule Labels
```json
{
    "Action": {
        "Allow": {}
    },
    "RuleLabels": [
        {
            "Name": "trusted-user"
        },
        {
            "Name": "verified-source"
        }
    ]
}
```

#### Label Matching
```json
{
    "LabelMatchStatement": {
        "Scope": "LABEL",
        "Key": "trusted-user"
    }
}
```

## Bot Control and Management

### AWS Managed Bot Control

#### Bot Control Rule Group
```json
{
    "ManagedRuleGroupStatement": {
        "VendorName": "AWS",
        "Name": "AWSManagedRulesBotControlRuleSet",
        "ManagedRuleGroupConfigs": [
            {
                "LoginPath": "/login",
                "PayloadType": "JSON",
                "UsernameField": "username",
                "PasswordField": "password"
            }
        ]
    }
}
```

#### Bot Control Categories
```bash
# Verified bots (Google, Bing, etc.)
Category: SignalVerifiedBot
Action: Allow

# Non-verified bots
Category: SignalNonVerifiedBot  
Action: Block

# Automated browsers
Category: CategoryHttpLibrary
Action: Challenge
```

### Custom Bot Detection

#### User Agent Analysis
```json
{
    "ByteMatchStatement": {
        "SearchString": "bot|crawler|spider",
        "FieldToMatch": {
            "SingleHeader": {
                "Name": "user-agent"
            }
        },
        "TextTransformations": [
            {
                "Priority": 0,
                "Type": "LOWERCASE"
            }
        ],
        "PositionalConstraint": "CONTAINS"
    }
}
```

#### JavaScript Challenge
```json
{
    "Action": {
        "Challenge": {
            "ImmunityTimeProperty": {
                "ImmunityTime": 300
            }
        }
    }
}
```

## Monitoring and Analytics

### CloudWatch Metrics

#### Key WAF Metrics
```bash
# Blocked requests
AWS/WAFV2/BlockedRequests

# Allowed requests  
AWS/WAFV2/AllowedRequests

# Rule match counts
AWS/WAFV2/CountedRequests

# CAPTCHA attempts
AWS/WAFV2/CaptchaRequests
```

#### Custom Metric Filters
```bash
# Create metric alarm for high block rate
aws cloudwatch put-metric-alarm \
    --alarm-name "WAF-High-Block-Rate" \
    --alarm-description "High percentage of blocked requests" \
    --metric-name BlockedRequests \
    --namespace AWS/WAFV2 \
    --statistic Sum \
    --period 300 \
    --threshold 1000 \
    --comparison-operator GreaterThanThreshold
```

### Request Sampling and Logging

#### Web ACL Logging
```bash
# Enable WAF logging to Kinesis Data Firehose
aws wafv2 put-logging-configuration \
    --logging-configuration '{
        "ResourceArn": "arn:aws:wafv2:us-east-1:123456789012:global/webacl/WebApplicationFirewall/12345678",
        "LogDestinationConfigs": [
            "arn:aws:firehose:us-east-1:123456789012:deliverystream/aws-waf-logs"
        ],
        "RedactedFields": [
            {
                "SingleHeader": {
                    "Name": "authorization"
                }
            }
        ]
    }'
```

#### Request Sampling
```json
{
    "SampledRequests": [
        {
            "Request": {
                "ClientIP": "192.168.1.100",
                "Country": "US",
                "URI": "/api/login",
                "Method": "POST",
                "HTTPVersion": "HTTP/1.1",
                "Headers": [
                    {
                        "Name": "User-Agent",
                        "Value": "Mozilla/5.0..."
                    }
                ]
            },
            "Weight": 1,
            "Timestamp": "2023-01-01T12:00:00Z",
            "Action": "BLOCK",
            "RuleNameWithinRuleGroup": "SQLInjectionRule"
        }
    ]
}
```

## Testing and Validation

### Rule Testing Methodology

#### Test Environment Setup
```bash
# Create test Web ACL
aws wafv2 create-web-acl \
    --name "Test-WebACL" \
    --scope REGIONAL \
    --default-action Allow={} \
    --rules file://test-rules.json
```

#### Penetration Testing
```bash
# SQL Injection test
curl -X POST "https://test-app.example.com/login" \
    -d "username=admin' OR '1'='1&password=test"

# XSS test
curl "https://test-app.example.com/search?q=<script>alert('xss')</script>"

# Rate limiting test
for i in {1..1001}; do
    curl "https://test-app.example.com/api/data" &
done
```

### A/B Testing with WAF

#### Gradual Rule Deployment
```json
{
    "Name": "TestRule",
    "Priority": 1,
    "Action": {
        "Count": {}
    },
    "Statement": {
        "SqliMatchStatement": {
            "FieldToMatch": {
                "Body": {}
            },
            "TextTransformations": [
                {
                    "Priority": 0,
                    "Type": "URL_DECODE"
                }
            ]
        }
    }
}
```

## Cost Optimization

### WAF Pricing Components

#### Request Processing
- **Web ACL evaluation**: $1.00 per million requests
- **Rule evaluation**: $1.00 per million rule evaluations
- **Bot Control**: $1.00 per million requests (additional)

#### Rule Storage
- **Managed rule groups**: $1.00 per rule group per month
- **IP Sets**: $1.00 per IP set per month
- **Regex pattern sets**: $1.00 per regex set per month

### Cost Optimization Strategies

#### Efficient Rule Design
```bash
# Order rules by likelihood of match
Priority 1: Common attack patterns (high match rate)
Priority 2: IP reputation (medium match rate)  
Priority 3: Geographic restrictions (low match rate)
```

#### Request Sampling
```json
{
    "SampledRequestsEnabled": true,
    "CloudWatchMetricsEnabled": true,
    "MetricName": "SampledRequests"
}
```

## Security Best Practices

### Defense in Depth
1. **Multiple Layers**: WAF + Security Groups + NACLs
2. **Rate Limiting**: Protect against DDoS
3. **IP Reputation**: Block known malicious sources
4. **Geographic Filtering**: Restrict by location
5. **Bot Management**: Control automated traffic

### Rule Management
1. **Regular Updates**: Keep managed rules current
2. **Custom Tuning**: Reduce false positives
3. **Testing**: Validate rules before production
4. **Monitoring**: Track rule effectiveness
5. **Documentation**: Maintain rule inventory

### Incident Response
1. **Automated Blocking**: Quick threat response
2. **Manual Override**: Emergency rule deployment
3. **Log Analysis**: Threat investigation
4. **Rule Refinement**: Improve detection accuracy

## Common Use Cases

### E-commerce Protection
```json
{
    "Rules": [
        {
            "Name": "PaymentPageProtection",
            "Statement": {
                "AndStatement": {
                    "Statements": [
                        {
                            "ByteMatchStatement": {
                                "SearchString": "/checkout",
                                "FieldToMatch": {"UriPath": {}},
                                "PositionalConstraint": "STARTS_WITH"
                            }
                        },
                        {
                            "SqliMatchStatement": {
                                "FieldToMatch": {"Body": {}}
                            }
                        }
                    ]
                }
            },
            "Action": {"Block": {}}
        }
    ]
}
```

### API Protection
```json
{
    "RateBasedStatement": {
        "Limit": 100,
        "AggregateKeyType": "IP",
        "ScopeDownStatement": {
            "ByteMatchStatement": {
                "SearchString": "/api/",
                "FieldToMatch": {"UriPath": {}},
                "PositionalConstraint": "STARTS_WITH"
            }
        }
    }
}
```

## Conclusion

AWS WAF provides comprehensive web application protection with flexible rule configuration, managed rule groups, and seamless AWS service integration. Proper implementation and ongoing management ensure effective security while maintaining application performance and user experience.