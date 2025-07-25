# Topic 47: Configuring AWS WAF Rules and Web ACLs

## Introduction

Configuring AWS WAF rules and Web ACLs requires understanding rule types, evaluation order, and best practices for effective web application protection. This guide provides comprehensive coverage of WAF configuration techniques.

## Web ACL Fundamentals

### Web ACL Structure

#### Basic Components
```json
{
    "Name": "ProductionWebACL",
    "Scope": "CLOUDFRONT",
    "DefaultAction": {
        "Allow": {}
    },
    "Description": "Production web application firewall",
    "Rules": [],
    "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "ProductionWebACL"
    },
    "CaptchaConfig": {
        "ImmunityTimeProperty": {
            "ImmunityTime": 300
        }
    }
}
```

#### Scope Considerations
- **CLOUDFRONT**: Global scope for CloudFront distributions
- **REGIONAL**: Regional scope for ALB, API Gateway, App Runner

### Rule Evaluation Order

#### Priority-Based Processing
```
Rule Priority 1: IP Whitelist (Allow)
Rule Priority 2: Rate Limiting (Block)
Rule Priority 3: Geo Restrictions (Block)
Rule Priority 4: SQL Injection (Block)
Rule Priority 5: XSS Protection (Block)
Rule Priority 6: Managed Rules (Block)
Default Action: Allow/Block
```

#### Rule Actions and Flow
```
Request → Rule 1 (Allow) → Terminates (Allow)
Request → Rule 1 (Block) → Terminates (Block)
Request → Rule 1 (Count) → Continue to Rule 2
Request → Rule 1 (No Match) → Continue to Rule 2
```

## Rule Types and Configuration

### IP Set Rules

#### Creating IP Sets
```bash
# Create blocked IP set
aws wafv2 create-ip-set \
    --name "BlockedIPs" \
    --scope REGIONAL \
    --ip-address-version IPV4 \
    --addresses "192.168.1.0/24" "10.0.0.0/8" "203.0.113.0/24" \
    --description "Known malicious IP addresses"

# Create allowed IP set
aws wafv2 create-ip-set \
    --name "TrustedIPs" \
    --scope REGIONAL \
    --ip-address-version IPV4 \
    --addresses "203.0.113.100/32" "198.51.100.0/24" \
    --description "Trusted IP addresses for admin access"
```

#### IP Set Rule Configuration
```json
{
    "Name": "BlockMaliciousIPs",
    "Priority": 1,
    "Statement": {
        "IPSetReferenceStatement": {
            "ARN": "arn:aws:wafv2:us-west-2:123456789012:regional/ipset/BlockedIPs/12345678",
            "IPSetForwardedIPConfig": {
                "HeaderName": "X-Forwarded-For",
                "FallbackBehavior": "MATCH",
                "Position": "FIRST"
            }
        }
    },
    "Action": {
        "Block": {
            "CustomResponse": {
                "ResponseCode": 403,
                "CustomResponseBodyKey": "blocked-ip-response"
            }
        }
    },
    "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "BlockedIPRule"
    }
}
```

### Geographic Rules

#### Country-Based Blocking
```json
{
    "Name": "GeoBlocking",
    "Priority": 2,
    "Statement": {
        "GeoMatchStatement": {
            "CountryCodes": ["CN", "RU", "KP", "IR"],
            "ForwardedIPConfig": {
                "HeaderName": "CloudFront-Viewer-Country",
                "FallbackBehavior": "MATCH"
            }
        }
    },
    "Action": {
        "Block": {}
    },
    "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "GeoBlockingRule"
    }
}
```

#### Geographic Allow List
```json
{
    "Name": "AllowedCountries",
    "Priority": 1,
    "Statement": {
        "NotStatement": {
            "Statement": {
                "GeoMatchStatement": {
                    "CountryCodes": ["US", "CA", "GB", "DE", "FR"]
                }
            }
        }
    },
    "Action": {
        "Block": {}
    }
}
```

### Rate-Based Rules

#### Basic Rate Limiting
```json
{
    "Name": "RateLimitRule",
    "Priority": 3,
    "Statement": {
        "RateBasedStatement": {
            "Limit": 1000,
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

#### Advanced Rate Limiting with Scope
```json
{
    "Name": "APIRateLimit",
    "Priority": 4,
    "Statement": {
        "RateBasedStatement": {
            "Limit": 100,
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
    },
    "Action": {
        "Captcha": {}
    }
}
```

#### Custom Key Rate Limiting
```json
{
    "RateBasedStatement": {
        "Limit": 50,
        "AggregateKeyType": "CUSTOM_KEYS",
        "CustomKeys": [
            {
                "Header": {
                    "Name": "x-api-key",
                    "TextTransformations": [
                        {
                            "Priority": 0,
                            "Type": "NONE"
                        }
                    ]
                }
            }
        ]
    }
}
```

### String Match Rules

#### SQL Injection Protection
```json
{
    "Name": "SQLInjectionProtection",
    "Priority": 5,
    "Statement": {
        "SqliMatchStatement": {
            "FieldToMatch": {
                "AllQueryArguments": {}
            },
            "TextTransformations": [
                {
                    "Priority": 0,
                    "Type": "URL_DECODE"
                },
                {
                    "Priority": 1,
                    "Type": "HTML_ENTITY_DECODE"
                }
            ]
        }
    },
    "Action": {
        "Block": {}
    }
}
```

#### XSS Protection
```json
{
    "Name": "XSSProtection",
    "Priority": 6,
    "Statement": {
        "XssMatchStatement": {
            "FieldToMatch": {
                "Body": {
                    "OversizeHandling": "CONTINUE"
                }
            },
            "TextTransformations": [
                {
                    "Priority": 0,
                    "Type": "URL_DECODE"
                },
                {
                    "Priority": 1,
                    "Type": "HTML_ENTITY_DECODE"
                }
            ]
        }
    },
    "Action": {
        "Block": {}
    }
}
```

#### Custom String Matching
```json
{
    "Name": "AdminPathProtection",
    "Priority": 7,
    "Statement": {
        "ByteMatchStatement": {
            "SearchString": "/admin",
            "FieldToMatch": {
                "UriPath": {}
            },
            "TextTransformations": [
                {
                    "Priority": 0,
                    "Type": "LOWERCASE"
                }
            ],
            "PositionalConstraint": "STARTS_WITH"
        }
    },
    "Action": {
        "Allow": {}
    },
    "OverrideAction": {
        "None": {}
    }
}
```

### Regular Expression Rules

#### Creating Regex Pattern Sets
```bash
# Create regex pattern set
aws wafv2 create-regex-pattern-set \
    --name "MaliciousPatterns" \
    --scope REGIONAL \
    --regular-expression-list "Pattern=.*union.*select.*" "Pattern=.*drop.*table.*" "Pattern=.*exec.*xp_.*" \
    --description "Common SQL injection patterns"
```

#### Regex Rule Configuration
```json
{
    "Name": "RegexMatchRule",
    "Priority": 8,
    "Statement": {
        "RegexMatchStatement": {
            "RegexString": "(?i)(union|select|insert|update|delete|drop|create|alter|exec)",
            "FieldToMatch": {
                "AllQueryArguments": {}
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
```

#### Regex Pattern Set Reference
```json
{
    "Name": "RegexPatternSetRule",
    "Priority": 9,
    "Statement": {
        "RegexPatternSetReferenceStatement": {
            "ARN": "arn:aws:wafv2:us-west-2:123456789012:regional/regexpatternset/MaliciousPatterns/12345678",
            "FieldToMatch": {
                "Body": {}
            },
            "TextTransformations": [
                {
                    "Priority": 0,
                    "Type": "NONE"
                }
            ]
        }
    },
    "Action": {
        "Block": {}
    }
}
```

## Complex Rule Logic

### Logical Operators

#### AND Statement
```json
{
    "Name": "ComplexAndRule",
    "Priority": 10,
    "Statement": {
        "AndStatement": {
            "Statements": [
                {
                    "ByteMatchStatement": {
                        "SearchString": "/login",
                        "FieldToMatch": {
                            "UriPath": {}
                        },
                        "PositionalConstraint": "EXACTLY"
                    }
                },
                {
                    "ByteMatchStatement": {
                        "SearchString": "POST",
                        "FieldToMatch": {
                            "Method": {}
                        },
                        "PositionalConstraint": "EXACTLY"
                    }
                },
                {
                    "NotStatement": {
                        "Statement": {
                            "IPSetReferenceStatement": {
                                "ARN": "arn:aws:wafv2:us-west-2:123456789012:regional/ipset/TrustedIPs/12345678"
                            }
                        }
                    }
                }
            ]
        }
    },
    "Action": {
        "Captcha": {}
    }
}
```

#### OR Statement
```json
{
    "Name": "ComplexOrRule",
    "Priority": 11,
    "Statement": {
        "OrStatement": {
            "Statements": [
                {
                    "SqliMatchStatement": {
                        "FieldToMatch": {
                            "AllQueryArguments": {}
                        }
                    }
                },
                {
                    "XssMatchStatement": {
                        "FieldToMatch": {
                            "Body": {}
                        }
                    }
                },
                {
                    "RegexMatchStatement": {
                        "RegexString": "(?i)script.*alert",
                        "FieldToMatch": {
                            "UriPath": {}
                        }
                    }
                }
            ]
        }
    },
    "Action": {
        "Block": {}
    }
}
```

### Label-Based Rules

#### Rule with Labels
```json
{
    "Name": "LabelingRule",
    "Priority": 12,
    "Statement": {
        "ByteMatchStatement": {
            "SearchString": "/api/",
            "FieldToMatch": {
                "UriPath": {}
            },
            "PositionalConstraint": "STARTS_WITH"
        }
    },
    "Action": {
        "Allow": {}
    },
    "RuleLabels": [
        {
            "Name": "api-request"
        },
        {
            "Name": "authenticated-endpoint"
        }
    ]
}
```

#### Label-Based Rate Limiting
```json
{
    "Name": "LabelBasedRateLimit",
    "Priority": 13,
    "Statement": {
        "AndStatement": {
            "Statements": [
                {
                    "LabelMatchStatement": {
                        "Scope": "LABEL",
                        "Key": "api-request"
                    }
                },
                {
                    "RateBasedStatement": {
                        "Limit": 50,
                        "AggregateKeyType": "IP"
                    }
                }
            ]
        }
    },
    "Action": {
        "Block": {}
    }
}
```

## Managed Rule Groups

### AWS Managed Rules Configuration

#### Core Rule Set with Exclusions
```json
{
    "Name": "AWSManagedRulesCommonRuleSet",
    "Priority": 20,
    "Statement": {
        "ManagedRuleGroupStatement": {
            "VendorName": "AWS",
            "Name": "AWSManagedRulesCommonRuleSet",
            "ExcludedRules": [
                {
                    "Name": "SizeRestrictions_BODY"
                },
                {
                    "Name": "GenericRFI_BODY"
                }
            ]
        }
    },
    "OverrideAction": {
        "None": {}
    },
    "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "AWSManagedRulesCommonRuleSetMetric"
    }
}
```

#### Rule Group with Action Override
```json
{
    "Name": "AWSManagedRulesKnownBadInputsRuleSet",
    "Priority": 21,
    "Statement": {
        "ManagedRuleGroupStatement": {
            "VendorName": "AWS",
            "Name": "AWSManagedRulesKnownBadInputsRuleSet"
        }
    },
    "OverrideAction": {
        "Count": {}
    }
}
```

#### Application-Specific Managed Rules
```json
{
    "Name": "WordPressRuleSet",
    "Priority": 22,
    "Statement": {
        "ManagedRuleGroupStatement": {
            "VendorName": "AWS",
            "Name": "AWSManagedRulesWordPressRuleSet",
            "RuleActionOverrides": [
                {
                    "Name": "WordPressExploitableTheme",
                    "ActionToUse": {
                        "Count": {}
                    }
                }
            ]
        }
    }
}
```

### Third-Party Managed Rules

#### Fortinet Managed Rules
```json
{
    "Name": "FortinetWebApplicationFirewall",
    "Priority": 30,
    "Statement": {
        "ManagedRuleGroupStatement": {
            "VendorName": "Fortinet",
            "Name": "FortinetManagedRulesWebApplicationsRuleSet",
            "ManagedRuleGroupConfigs": [
                {
                    "LoginPath": "/login",
                    "PayloadType": "JSON",
                    "UsernameField": "username",
                    "PasswordField": "password"
                }
            ]
        }
    },
    "OverrideAction": {
        "None": {}
    }
}
```

## Custom Rule Groups

### Creating Custom Rule Groups

#### Basic Rule Group Structure
```bash
# Create custom rule group
aws wafv2 create-rule-group \
    --name "CustomProtectionRules" \
    --scope REGIONAL \
    --capacity 100 \
    --description "Custom application-specific protection rules" \
    --rules file://custom-rules.json
```

#### Custom Rule Group Definition
```json
{
    "Rules": [
        {
            "Name": "BlockSuspiciousUserAgents",
            "Priority": 1,
            "Statement": {
                "ByteMatchStatement": {
                    "SearchString": "bot|crawler|scanner",
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
            },
            "Action": {
                "Block": {}
            }
        }
    ],
    "VisibilityConfig": {
        "SampledRequestsEnabled": true,
        "CloudWatchMetricsEnabled": true,
        "MetricName": "CustomProtectionRules"
    }
}
```

### Rule Group Reference
```json
{
    "Name": "CustomRuleGroupReference",
    "Priority": 40,
    "Statement": {
        "RuleGroupReferenceStatement": {
            "ARN": "arn:aws:wafv2:us-west-2:123456789012:regional/rulegroup/CustomProtectionRules/12345678",
            "ExcludedRules": []
        }
    },
    "OverrideAction": {
        "None": {}
    }
}
```

## Advanced Actions and Responses

### CAPTCHA Configuration

#### CAPTCHA Action
```json
{
    "Action": {
        "Captcha": {
            "CustomRequestHandling": {
                "InsertHeaders": [
                    {
                        "Name": "X-Captcha-Timestamp",
                        "Value": "${aws:request-time}"
                    }
                ]
            }
        }
    }
}
```

#### CAPTCHA Immunity Configuration
```json
{
    "CaptchaConfig": {
        "ImmunityTimeProperty": {
            "ImmunityTime": 3600
        }
    }
}
```

### Challenge Actions

#### JavaScript Challenge
```json
{
    "Action": {
        "Challenge": {
            "CustomRequestHandling": {
                "InsertHeaders": [
                    {
                        "Name": "X-Challenge-Type", 
                        "Value": "javascript"
                    }
                ]
            }
        }
    }
}
```

### Custom Responses

#### Custom Block Response
```json
{
    "CustomResponseBodies": {
        "security-block": {
            "ContentType": "TEXT_HTML",
            "Content": "<html><head><title>Access Denied</title></head><body><h1>Your request has been blocked by our security system</h1><p>If you believe this is an error, please contact support.</p></body></html>"
        }
    }
}
```

#### Response with Headers
```json
{
    "Action": {
        "Block": {
            "CustomResponse": {
                "ResponseCode": 403,
                "CustomResponseBodyKey": "security-block",
                "ResponseHeaders": [
                    {
                        "Name": "X-Block-Reason",
                        "Value": "Security-Policy-Violation"
                    },
                    {
                        "Name": "X-Block-Timestamp",
                        "Value": "${aws:request-time}"
                    }
                ]
            }
        }
    }
}
```

## Testing and Validation

### Rule Testing Methodology

#### Testing Environment Setup
```bash
# Create test Web ACL
aws wafv2 create-web-acl \
    --name "TestWebACL" \
    --scope REGIONAL \
    --default-action Allow={} \
    --description "Testing environment for rule validation"
```

#### Count Mode Testing
```json
{
    "Action": {
        "Count": {
            "CustomRequestHandling": {
                "InsertHeaders": [
                    {
                        "Name": "X-WAF-Test-Mode",
                        "Value": "count"
                    }
                ]
            }
        }
    }
}
```

### Automated Testing Scripts

#### Rule Validation Script
```bash
#!/bin/bash

# Test SQL injection detection
echo "Testing SQL injection detection..."
curl -H "Content-Type: application/x-www-form-urlencoded" \
     -d "username=admin' OR '1'='1&password=test" \
     -X POST \
     https://test-app.example.com/login

# Test XSS detection
echo "Testing XSS detection..."
curl "https://test-app.example.com/search?q=<script>alert('xss')</script>"

# Test rate limiting
echo "Testing rate limiting..."
for i in {1..101}; do
    curl "https://test-app.example.com/api/test" &
done
wait

# Test geographic blocking
echo "Testing geographic restrictions..."
curl -H "CloudFront-Viewer-Country: CN" \
     https://test-app.example.com/
```

## Best Practices

### Rule Configuration Best Practices

1. **Start with Count Mode**: Test rules before blocking
2. **Use Appropriate Priorities**: Critical rules first
3. **Implement Gradual Deployment**: Roll out rules progressively
4. **Monitor False Positives**: Adjust rules based on legitimate traffic
5. **Document Rule Purpose**: Maintain clear documentation

### Performance Optimization

1. **Efficient Rule Ordering**: Most likely matches first
2. **Minimize Complex Logic**: Avoid overly complex AND/OR statements
3. **Use Appropriate Text Transformations**: Only necessary transformations
4. **Regular Rule Review**: Remove unused or ineffective rules

### Security Considerations

1. **Defense in Depth**: Combine multiple rule types
2. **Regular Updates**: Keep managed rules current
3. **Custom Rule Tuning**: Adjust for application-specific threats
4. **Incident Response**: Plan for rule updates during attacks

## Conclusion

Effective WAF rule and Web ACL configuration requires understanding rule types, logical operators, and testing methodologies. Proper implementation provides robust web application protection while maintaining optimal performance and user experience.