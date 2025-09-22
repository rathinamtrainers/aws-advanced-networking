# Topic 50: Protecting Edge Applications with CloudFront + WAF

## Introduction

Edge protection using Amazon CloudFront and AWS WAF provides a powerful defense mechanism against web-based attacks. This combination offers global distribution, caching capabilities, and comprehensive security filtering at AWS edge locations worldwide.

## CloudFront Security Architecture

### Edge-Based Protection Model

#### Global Edge Network
```
User Request → CloudFront Edge Location → Origin Shield → Origin Server
              └── WAF Processing ──┘
```

#### Security Processing Flow
```
1. DNS Resolution → Route 53 (with health checks)
2. Edge Location → CloudFront security checks
3. WAF Processing → Rule evaluation at edge
4. Cache Decision → Serve from cache or fetch from origin
5. Response Delivery → Secure content delivery
```

### CloudFront Security Features

#### Built-in DDoS Protection
- **AWS Shield Standard** included automatically
- **Layer 3/4 protection** at edge locations
- **Automatic scaling** to handle attack traffic
- **Geographic distribution** of attack mitigation

#### SSL/TLS Security
- **End-to-end encryption** with custom certificates
- **Perfect Forward Secrecy** support
- **TLS 1.2/1.3** protocol support
- **HSTS header** enforcement

## CloudFront Configuration for Security

### Distribution Security Settings

#### Basic Security Configuration
```json
{
    "DistributionConfig": {
        "CallerReference": "secure-distribution-2023",
        "Comment": "Secure CloudFront distribution with WAF",
        "DefaultRootObject": "index.html",
        "Origins": [
            {
                "Id": "secure-origin",
                "DomainName": "origin.example.com",
                "CustomOriginConfig": {
                    "HTTPPort": 80,
                    "HTTPSPort": 443,
                    "OriginProtocolPolicy": "https-only",
                    "OriginSslProtocols": {
                        "Quantity": 2,
                        "Items": ["TLSv1.2", "TLSv1.3"]
                    },
                    "OriginReadTimeout": 30,
                    "OriginKeepaliveTimeout": 5
                }
            }
        ],
        "DefaultCacheBehavior": {
            "TargetOriginId": "secure-origin",
            "ViewerProtocolPolicy": "redirect-to-https",
            "TrustedSigners": {
                "Enabled": false,
                "Quantity": 0
            },
            "ForwardedValues": {
                "QueryString": false,
                "Cookies": {"Forward": "none"},
                "Headers": {
                    "Quantity": 1,
                    "Items": ["CloudFront-Viewer-Country"]
                }
            },
            "MinTTL": 0,
            "DefaultTTL": 86400,
            "MaxTTL": 31536000,
            "Compress": true
        },
        "Enabled": true,
        "PriceClass": "PriceClass_All"
    }
}
```

#### Origin Access Control (OAC)
```bash
# Create Origin Access Control
aws cloudfront create-origin-access-control \
    --origin-access-control-config '{
        "Name": "secure-oac",
        "Description": "Origin Access Control for S3 bucket",
        "OriginAccessControlOriginType": "s3",
        "SigningBehavior": "always",
        "SigningProtocol": "sigv4"
    }'

# Update S3 bucket policy for OAC
cat > bucket-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowCloudFrontServicePrincipal",
            "Effect": "Allow",
            "Principal": {
                "Service": "cloudfront.amazonaws.com"
            },
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::secure-bucket/*",
            "Condition": {
                "StringEquals": {
                    "AWS:SourceArn": "arn:aws:cloudfront::123456789012:distribution/EDFDVBD6EXAMPLE"
                }
            }
        }
    ]
}
EOF

aws s3api put-bucket-policy \
    --bucket secure-bucket \
    --policy file://bucket-policy.json
```

### Custom Headers and Security

#### Security Headers Configuration
```json
{
    "ResponseHeadersPolicy": {
        "ResponseHeadersPolicyConfig": {
            "Name": "security-headers-policy",
            "Comment": "Security headers for edge protection",
            "SecurityHeadersConfig": {
                "StrictTransportSecurity": {
                    "AccessControlMaxAgeSec": 31536000,
                    "IncludeSubdomains": true,
                    "Preload": true
                },
                "ContentTypeOptions": {
                    "Override": true
                },
                "FrameOptions": {
                    "FrameOption": "DENY",
                    "Override": true
                },
                "ReferrerPolicy": {
                    "ReferrerPolicy": "strict-origin-when-cross-origin",
                    "Override": true
                },
                "ContentSecurityPolicy": {
                    "ContentSecurityPolicy": "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'",
                    "Override": true
                }
            },
            "CustomHeadersConfig": {
                "Quantity": 1,
                "Items": [
                    {
                        "Header": "X-Security-Level",
                        "Value": "high",
                        "Override": false
                    }
                ]
            }
        }
    }
}
```

#### Request Headers Forwarding
```json
{
    "ForwardedValues": {
        "QueryString": true,
        "Cookies": {"Forward": "whitelist", "WhitelistedNames": {"Quantity": 2, "Items": ["session", "auth"]}},
        "Headers": {
            "Quantity": 5,
            "Items": [
                "Authorization",
                "CloudFront-Viewer-Country", 
                "CloudFront-Is-Mobile-Viewer",
                "CloudFront-Is-Desktop-Viewer",
                "User-Agent"
            ]
        }
    }
}
```

## WAF Integration with CloudFront

### Web ACL Configuration for CloudFront

#### Global Scope WAF Setup
```bash
# Create WAF Web ACL for CloudFront (Global scope)
aws wafv2 create-web-acl \
    --name "CloudFront-Security-WAF" \
    --scope CLOUDFRONT \
    --default-action Allow={} \
    --description "WAF for CloudFront edge protection" \
    --rules file://cloudfront-waf-rules.json
```

#### Edge-Optimized Rules
```json
{
    "Rules": [
        {
            "Name": "RateLimitPerIP",
            "Priority": 1,
            "Statement": {
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
            },
            "VisibilityConfig": {
                "SampledRequestsEnabled": true,
                "CloudWatchMetricsEnabled": true,
                "MetricName": "EdgeRateLimit"
            }
        },
        {
            "Name": "GeoBlocking",
            "Priority": 2,
            "Statement": {
                "GeoMatchStatement": {
                    "CountryCodes": ["CN", "RU", "KP"],
                    "ForwardedIPConfig": {
                        "HeaderName": "CloudFront-Viewer-Country",
                        "FallbackBehavior": "MATCH"
                    }
                }
            },
            "Action": {
                "Block": {}
            }
        },
        {
            "Name": "MaliciousUserAgents",
            "Priority": 3,
            "Statement": {
                "ByteMatchStatement": {
                    "SearchString": "sqlmap|nikto|nmap|masscan|nessus",
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
    ]
}
```

### Bot Management at Edge

#### CloudFront Bot Control
```json
{
    "Name": "BotControlAtEdge",
    "Priority": 10,
    "Statement": {
        "ManagedRuleGroupStatement": {
            "VendorName": "AWS",
            "Name": "AWSManagedRulesBotControlRuleSet",
            "ManagedRuleGroupConfigs": [
                {
                    "LoginPath": "/login",
                    "PayloadType": "JSON",
                    "UsernameField": "email",
                    "PasswordField": "password"
                }
            ],
            "RuleActionOverrides": [
                {
                    "Name": "CategoryVerifiedSearchEngine",
                    "ActionToUse": {
                        "Allow": {}
                    }
                },
                {
                    "Name": "CategorySocialMedia",
                    "ActionToUse": {
                        "Allow": {}
                    }
                }
            ]
        }
    },
    "OverrideAction": {
        "None": {}
    }
}
```

#### Custom Bot Detection
```json
{
    "Name": "CustomBotDetection", 
    "Priority": 11,
    "Statement": {
        "OrStatement": {
            "Statements": [
                {
                    "ByteMatchStatement": {
                        "SearchString": "bot",
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
                {
                    "SizeConstraintStatement": {
                        "FieldToMatch": {
                            "SingleHeader": {
                                "Name": "user-agent"
                            }
                        },
                        "ComparisonOperator": "LT",
                        "Size": 10,
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
    },
    "Action": {
        "Challenge": {}
    }
}
```

## Advanced Security Configurations

### Multi-Behavior Security

#### Path-Based Security Rules
```json
{
    "CacheBehaviors": {
        "Quantity": 3,
        "Items": [
            {
                "PathPattern": "/api/*",
                "TargetOriginId": "api-origin",
                "ViewerProtocolPolicy": "https-only",
                "AllowedMethods": {
                    "Quantity": 7,
                    "Items": ["GET", "HEAD", "OPTIONS", "PUT", "POST", "PATCH", "DELETE"]
                },
                "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
                "OriginRequestPolicyId": "88a5eaf4-2fd4-4709-b370-b4c650ea3fcf",
                "ResponseHeadersPolicyId": "67f7725c-6f97-4210-82d7-5512b31e9d03"
            },
            {
                "PathPattern": "/admin/*",
                "TargetOriginId": "admin-origin",
                "ViewerProtocolPolicy": "https-only",
                "TrustedKeyGroups": {
                    "Enabled": true,
                    "Quantity": 1,
                    "Items": ["admin-key-group"]
                },
                "AllowedMethods": {
                    "Quantity": 3,
                    "Items": ["GET", "HEAD", "OPTIONS"]
                }
            },
            {
                "PathPattern": "/static/*",
                "TargetOriginId": "static-origin",
                "ViewerProtocolPolicy": "redirect-to-https",
                "Compress": true,
                "CachePolicyId": "static-content-policy"
            }
        ]
    }
}
```

### Signed URLs and Cookies

#### Signed URL Configuration
```python
import boto3
from datetime import datetime, timedelta
import base64
import json
from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import rsa, padding

def create_signed_url(url, private_key_path, key_pair_id, expiration_time):
    """
    Create CloudFront signed URL for secure content access
    """
    # Load private key
    with open(private_key_path, 'rb') as key_file:
        private_key = serialization.load_pem_private_key(
            key_file.read(),
            password=None
        )
    
    # Create policy
    policy = {
        "Statement": [
            {
                "Resource": url,
                "Condition": {
                    "DateLessThan": {
                        "AWS:EpochTime": int(expiration_time.timestamp())
                    }
                }
            }
        ]
    }
    
    # Encode policy
    policy_json = json.dumps(policy, separators=(',', ':'))
    policy_base64 = base64.b64encode(policy_json.encode()).decode()
    policy_base64 = policy_base64.replace('+', '-').replace('=', '_').replace('/', '~')
    
    # Sign policy
    signature = private_key.sign(
        policy_json.encode(),
        padding.PKCS1v15(),
        hashes.SHA1()
    )
    signature_base64 = base64.b64encode(signature).decode()
    signature_base64 = signature_base64.replace('+', '-').replace('=', '_').replace('/', '~')
    
    # Create signed URL
    signed_url = f"{url}?Policy={policy_base64}&Signature={signature_base64}&Key-Pair-Id={key_pair_id}"
    
    return signed_url

# Usage example
private_key_path = '/path/to/private_key.pem'
key_pair_id = 'APKAEIBAERJR2EXAMPLE'
url = 'https://d1234567890.cloudfront.net/secure-content.pdf'
expiration = datetime.utcnow() + timedelta(hours=1)

signed_url = create_signed_url(url, private_key_path, key_pair_id, expiration)
```

#### Signed Cookies for Session Management
```python
def create_signed_cookies(resource, private_key_path, key_pair_id, expiration_time):
    """
    Create CloudFront signed cookies for session-based access
    """
    # Create policy (similar to signed URL)
    policy = {
        "Statement": [
            {
                "Resource": resource,
                "Condition": {
                    "DateLessThan": {
                        "AWS:EpochTime": int(expiration_time.timestamp())
                    }
                }
            }
        ]
    }
    
    # Load private key and sign policy
    with open(private_key_path, 'rb') as key_file:
        private_key = serialization.load_pem_private_key(key_file.read(), password=None)
    
    policy_json = json.dumps(policy, separators=(',', ':'))
    policy_base64 = base64.b64encode(policy_json.encode()).decode()
    policy_base64 = policy_base64.replace('+', '-').replace('=', '_').replace('/', '~')
    
    signature = private_key.sign(policy_json.encode(), padding.PKCS1v15(), hashes.SHA1())
    signature_base64 = base64.b64encode(signature).decode()
    signature_base64 = signature_base64.replace('+', '-').replace('=', '_').replace('/', '~')
    
    # Return cookies
    return {
        'CloudFront-Policy': policy_base64,
        'CloudFront-Signature': signature_base64,
        'CloudFront-Key-Pair-Id': key_pair_id
    }
```

## Edge Functions for Security

### CloudFront Functions for Request Processing

#### Security Headers Function
```javascript
function handler(event) {
    var request = event.request;
    var headers = request.headers;
    
    // Block requests with suspicious patterns
    var userAgent = headers['user-agent'] ? headers['user-agent'].value : '';
    
    if (userAgent.toLowerCase().includes('sqlmap') || 
        userAgent.toLowerCase().includes('nikto') ||
        userAgent.toLowerCase().includes('nmap')) {
        return {
            statusCode: 403,
            statusDescription: 'Forbidden',
            headers: {
                'content-type': {value: 'text/plain'}
            },
            body: 'Access denied due to suspicious user agent'
        };
    }
    
    // Add security headers to request
    headers['x-forwarded-for-security'] = {value: headers['cloudfront-viewer-address'].value};
    headers['x-request-id'] = {value: generateRequestId()};
    
    return request;
}

function generateRequestId() {
    return 'req-' + Date.now() + '-' + Math.random().toString(36).substr(2, 9);
}
```

#### Geographic Blocking Function
```javascript
function handler(event) {
    var request = event.request;
    var headers = request.headers;
    
    // Get viewer country
    var country = headers['cloudfront-viewer-country'] ? 
                  headers['cloudfront-viewer-country'].value : 'Unknown';
    
    // Blocked countries
    var blockedCountries = ['CN', 'RU', 'KP', 'IR'];
    
    if (blockedCountries.includes(country)) {
        return {
            statusCode: 403,
            statusDescription: 'Forbidden',
            headers: {
                'content-type': {value: 'text/html'},
                'cache-control': {value: 'max-age=300'}
            },
            body: '<html><body><h1>Access Denied</h1><p>Access from your location is not permitted.</p></body></html>'
        };
    }
    
    return request;
}
```

### Lambda@Edge for Advanced Processing

#### Request Authentication
```python
import json
import base64
import hmac
import hashlib
from datetime import datetime, timedelta

def lambda_handler(event, context):
    """
    Lambda@Edge function for request authentication
    """
    request = event['Records'][0]['cf']['request']
    headers = request['headers']
    
    # Check for authentication header
    auth_header = None
    if 'authorization' in headers:
        auth_header = headers['authorization'][0]['value']
    
    if not auth_header or not validate_token(auth_header):
        return {
            'status': '401',
            'statusDescription': 'Unauthorized',
            'headers': {
                'www-authenticate': [{
                    'key': 'WWW-Authenticate',
                    'value': 'Bearer realm="CloudFront"'
                }],
                'content-type': [{
                    'key': 'Content-Type',
                    'value': 'text/plain'
                }]
            },
            'body': 'Authentication required'
        }
    
    # Add validated user info to headers
    user_info = decode_token(auth_header)
    headers['x-authenticated-user'] = [{
        'key': 'X-Authenticated-User',
        'value': user_info['username']
    }]
    
    return request

def validate_token(token):
    """Validate JWT token"""
    try:
        # Extract token from Bearer scheme
        if token.startswith('Bearer '):
            token = token[7:]
        
        # Decode and validate JWT (simplified)
        parts = token.split('.')
        if len(parts) != 3:
            return False
        
        # Validate signature and expiration
        header = json.loads(base64.urlsafe_b64decode(parts[0] + '=='))
        payload = json.loads(base64.urlsafe_b64decode(parts[1] + '=='))
        
        # Check expiration
        if payload.get('exp', 0) < datetime.utcnow().timestamp():
            return False
        
        return True
    except Exception:
        return False

def decode_token(token):
    """Decode JWT token to extract user info"""
    try:
        if token.startswith('Bearer '):
            token = token[7:]
        
        parts = token.split('.')
        payload = json.loads(base64.urlsafe_b64decode(parts[1] + '=='))
        
        return {
            'username': payload.get('sub', 'unknown'),
            'role': payload.get('role', 'user')
        }
    except Exception:
        return {'username': 'unknown', 'role': 'user'}
```

## Monitoring and Analytics

### CloudWatch Metrics

#### CloudFront Security Metrics
```bash
# Monitor cache hit ratio
aws cloudwatch get-metric-statistics \
    --namespace AWS/CloudFront \
    --metric-name CacheHitRate \
    --dimensions Name=DistributionId,Value=EDFDVBD6EXAMPLE \
    --start-time 2023-01-01T00:00:00Z \
    --end-time 2023-01-02T00:00:00Z \
    --period 3600 \
    --statistics Average

# Monitor 4xx error rate
aws cloudwatch get-metric-statistics \
    --namespace AWS/CloudFront \
    --metric-name 4xxErrorRate \
    --dimensions Name=DistributionId,Value=EDFDVBD6EXAMPLE \
    --start-time 2023-01-01T00:00:00Z \
    --end-time 2023-01-02T00:00:00Z \
    --period 3600 \
    --statistics Average
```

#### WAF Metrics at Edge
```bash
# Monitor blocked requests
aws cloudwatch get-metric-statistics \
    --namespace AWS/WAFV2 \
    --metric-name BlockedRequests \
    --dimensions Name=WebACL,Value=CloudFront-Security-WAF Name=Region,Value=CloudFront \
    --start-time 2023-01-01T00:00:00Z \
    --end-time 2023-01-02T00:00:00Z \
    --period 300 \
    --statistics Sum
```

### Real-Time Logs

#### CloudFront Real-Time Logs Configuration
```bash
# Create real-time log configuration
aws cloudfront create-realtime-log-config \
    --name "security-realtime-logs" \
    --end-points '[
        {
            "StreamType": "Kinesis",
            "KinesisStreamConfig": {
                "RoleArn": "arn:aws:iam::123456789012:role/CloudFrontRealtimeLogRole",
                "StreamArn": "arn:aws:kinesis:us-east-1:123456789012:stream/cloudfront-logs"
            }
        }
    ]' \
    --fields 'timestamp,c-ip,sc-status,cs-method,cs-uri-stem,cs-user-agent,cs-referer,x-edge-location,x-edge-response-result-type,x-edge-request-id'
```

#### Log Analysis for Security Events
```python
import json
import boto3
from datetime import datetime

def analyze_security_logs(event, context):
    """
    Analyze CloudFront real-time logs for security events
    """
    for record in event['Records']:
        # Decode Kinesis data
        payload = json.loads(
            base64.b64decode(record['kinesis']['data']).decode('utf-8')
        )
        
        # Analyze for security patterns
        analyze_request(payload)

def analyze_request(log_entry):
    """
    Analyze individual request for security threats
    """
    ip = log_entry.get('c-ip', '')
    status = log_entry.get('sc-status', '')
    method = log_entry.get('cs-method', '')
    uri = log_entry.get('cs-uri-stem', '')
    user_agent = log_entry.get('cs-user-agent', '')
    
    # Detect potential attacks
    threats = []
    
    # SQL injection patterns in URI
    if any(pattern in uri.lower() for pattern in ['union', 'select', 'drop', 'insert']):
        threats.append('SQLInjection')
    
    # XSS patterns
    if any(pattern in uri.lower() for pattern in ['<script', 'javascript:', 'onload=']):
        threats.append('XSS')
    
    # Suspicious user agents
    suspicious_agents = ['sqlmap', 'nikto', 'nmap', 'masscan']
    if any(agent in user_agent.lower() for agent in suspicious_agents):
        threats.append('SuspiciousUserAgent')
    
    # High error rates from single IP
    if status.startswith('4') or status.startswith('5'):
        check_error_rate(ip)
    
    if threats:
        send_security_alert(ip, threats, log_entry)

def send_security_alert(ip, threats, log_entry):
    """
    Send security alert for detected threats
    """
    sns = boto3.client('sns')
    
    message = {
        'timestamp': datetime.utcnow().isoformat(),
        'source_ip': ip,
        'threats': threats,
        'log_entry': log_entry
    }
    
    sns.publish(
        TopicArn='arn:aws:sns:us-east-1:123456789012:security-alerts',
        Subject=f'CloudFront Security Alert: {", ".join(threats)}',
        Message=json.dumps(message, indent=2)
    )
```

## Performance Optimization with Security

### Caching Strategy for Security

#### Security-Aware Cache Policies
```json
{
    "CachePolicyConfig": {
        "Name": "security-optimized-caching",
        "Comment": "Cache policy optimized for security and performance",
        "DefaultTTL": 86400,
        "MaxTTL": 31536000,
        "MinTTL": 1,
        "ParametersInCacheKeyAndForwardedToOrigin": {
            "EnableAcceptEncodingGzip": true,
            "EnableAcceptEncodingBrotli": true,
            "QueryStringsConfig": {
                "QueryStringBehavior": "whitelist",
                "QueryStrings": {
                    "Quantity": 2,
                    "Items": ["version", "lang"]
                }
            },
            "HeadersConfig": {
                "HeaderBehavior": "whitelist",
                "Headers": {
                    "Quantity": 3,
                    "Items": [
                        "Authorization",
                        "CloudFront-Viewer-Country",
                        "User-Agent"
                    ]
                }
            },
            "CookiesConfig": {
                "CookieBehavior": "whitelist",
                "Cookies": {
                    "Quantity": 2,
                    "Items": ["session", "auth"]
                }
            }
        }
    }
}
```

### Origin Shield for Security

#### Shield Configuration
```bash
# Enable Origin Shield
aws cloudfront create-distribution \
    --distribution-config '{
        "Origins": [{
            "Id": "protected-origin",
            "DomainName": "origin.example.com",
            "CustomOriginConfig": {
                "OriginShieldEnabled": true,
                "OriginShieldRegion": "us-east-1",
                "HTTPPort": 80,
                "HTTPSPort": 443,
                "OriginProtocolPolicy": "https-only"
            }
        }]
    }'
```

## Cost Optimization

### Efficient Security Configuration

#### Price Class Optimization
```json
{
    "PriceClass": "PriceClass_100",
    "Comment": "Use only North America and Europe edge locations for cost optimization"
}
```

#### Request/Response Size Optimization
```json
{
    "Compress": true,
    "DefaultTTL": 86400,
    "MaxTTL": 31536000,
    "ViewerProtocolPolicy": "redirect-to-https"
}
```

## Best Practices

### Security Best Practices
1. **Always Use HTTPS**: Redirect HTTP to HTTPS
2. **Implement WAF Rules**: Use comprehensive rule sets
3. **Monitor Continuously**: Set up real-time alerting
4. **Use Origin Shield**: Reduce origin load and improve security
5. **Regular Updates**: Keep security configurations current

### Performance Best Practices
1. **Optimize Caching**: Use appropriate cache policies
2. **Compress Content**: Enable compression for better performance
3. **Use Edge Functions**: Process at edge for better user experience
4. **Monitor Metrics**: Track performance and security metrics
5. **Test Regularly**: Validate configurations and performance

### Cost Optimization
1. **Right-size Price Class**: Choose appropriate edge locations
2. **Optimize Cache Ratios**: Improve cache hit rates
3. **Use Compression**: Reduce data transfer costs
4. **Monitor Usage**: Track costs and optimize accordingly

## Conclusion

CloudFront and WAF integration provides comprehensive edge protection combining global distribution, advanced security filtering, and performance optimization. Proper configuration ensures robust security while maintaining excellent user experience and cost efficiency.