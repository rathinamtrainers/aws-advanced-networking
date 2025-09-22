# Topic 51: API Security Networking Tools

## Introduction

API security is critical in modern cloud architectures where APIs serve as the primary interface for application communication. AWS provides comprehensive networking tools and security features to protect APIs at various layers. This topic covers API Gateway security, VPC endpoints for APIs, private APIs, authentication mechanisms, rate limiting, monitoring, and DDoS protection strategies.

## API Gateway Security Overview

### AWS API Gateway Architecture

API Gateway serves as a fully managed service that makes it easy to create, publish, maintain, monitor, and secure APIs at any scale. From a networking perspective, API Gateway provides multiple security layers:

```
Internet → CloudFront → API Gateway → Backend Services
    ↓           ↓           ↓            ↓
   WAF      Edge Cache   Auth/Rate    VPC/Lambda
                        Limiting
```

### API Gateway Deployment Types

#### Edge-Optimized APIs
- Default deployment type
- Uses CloudFront edge locations
- Best for geographically distributed clients
- Automatic DDoS protection via CloudFront

```json
{
  "endpointConfiguration": {
    "types": ["EDGE"]
  }
}
```

#### Regional APIs
- Deployed in specific AWS region
- Better for clients in same region
- Direct integration with regional services
- Lower latency for regional traffic

```json
{
  "endpointConfiguration": {
    "types": ["REGIONAL"]
  }
}
```

#### Private APIs
- Accessible only from VPC
- Uses VPC endpoints
- Enhanced security and compliance
- No internet gateway required

```json
{
  "endpointConfiguration": {
    "types": ["PRIVATE"],
    "vpcEndpointIds": ["vpce-1234567890abcdef0"]
  }
}
```

## VPC Endpoints for APIs

### Interface VPC Endpoints for API Gateway

VPC endpoints enable private connectivity between your VPC and API Gateway without traversing the internet:

```yaml
# CloudFormation template for API Gateway VPC Endpoint
VPCEndpointAPIGateway:
  Type: AWS::EC2::VPCEndpoint
  Properties:
    VpcId: !Ref VPC
    ServiceName: !Sub 'com.amazonaws.${AWS::Region}.execute-api'
    VpcEndpointType: Interface
    SubnetIds:
      - !Ref PrivateSubnet1
      - !Ref PrivateSubnet2
    SecurityGroupIds:
      - !Ref APIGatewayVPCEndpointSG
    PolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal: '*'
          Action:
            - execute-api:Invoke
          Resource: '*'
    PrivateDnsEnabled: true
```

### Security Group Configuration for VPC Endpoints

```yaml
APIGatewayVPCEndpointSG:
  Type: AWS::EC2::SecurityGroup
  Properties:
    GroupDescription: Security group for API Gateway VPC endpoint
    VpcId: !Ref VPC
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        CidrIp: 10.0.0.0/16
        Description: HTTPS traffic from VPC
    SecurityGroupEgress:
      - IpProtocol: tcp
        FromPort: 443
        ToPort: 443
        CidrIp: 0.0.0.0/0
        Description: HTTPS traffic to API Gateway
```

### Private API Configuration

```python
import boto3
import json

def create_private_api():
    client = boto3.client('apigateway')
    
    # Create private REST API
    response = client.create_rest_api(
        name='PrivateAPI',
        description='Private API accessible only from VPC',
        endpointConfiguration={
            'types': ['PRIVATE'],
            'vpcEndpointIds': ['vpce-1234567890abcdef0']
        },
        policy=json.dumps({
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": "*",
                    "Action": "execute-api:Invoke",
                    "Resource": "*",
                    "Condition": {
                        "StringEquals": {
                            "aws:sourceVpce": "vpce-1234567890abcdef0"
                        }
                    }
                }
            ]
        })
    )
    
    return response['id']
```

## Authentication and Authorization

### IAM Authentication

API Gateway integrates with AWS IAM for service-to-service authentication:

```python
import boto3
from botocore.auth import SigV4Auth
from botocore.awsrequest import AWSRequest
import requests

def invoke_api_with_iam(api_url, region, method='GET', data=None):
    """Invoke API Gateway with IAM authentication"""
    
    # Create AWS request
    request = AWSRequest(
        method=method,
        url=api_url,
        data=data
    )
    
    # Sign request with IAM credentials
    session = boto3.Session()
    credentials = session.get_credentials()
    SigV4Auth(credentials, 'execute-api', region).add_auth(request)
    
    # Make the request
    response = requests.request(
        method=request.method,
        url=request.url,
        headers=dict(request.headers),
        data=request.body
    )
    
    return response
```

### Cognito User Pool Authorization

```yaml
# Cognito Authorizer
CognitoAuthorizer:
  Type: AWS::ApiGateway::Authorizer
  Properties:
    Name: CognitoAuthorizer
    Type: COGNITO_USER_POOLS
    IdentitySource: method.request.header.Authorization
    RestApiId: !Ref RestApi
    ProviderARNs:
      - !GetAtt UserPool.Arn

# Method with Cognito authorization
ApiMethod:
  Type: AWS::ApiGateway::Method
  Properties:
    RestApiId: !Ref RestApi
    ResourceId: !Ref ApiResource
    HttpMethod: GET
    AuthorizationType: COGNITO_USER_POOLS
    AuthorizerId: !Ref CognitoAuthorizer
```

### Lambda Custom Authorizers

```python
import json
import jwt
from datetime import datetime

def lambda_handler(event, context):
    """Custom authorizer for API Gateway"""
    
    try:
        # Extract token from Authorization header
        token = event['authorizationToken'].replace('Bearer ', '')
        
        # Validate JWT token
        decoded_token = jwt.decode(
            token, 
            'your-secret-key', 
            algorithms=['HS256']
        )
        
        # Check token expiration
        if decoded_token['exp'] < datetime.utcnow().timestamp():
            raise Exception('Token expired')
        
        # Generate policy
        policy = generate_policy(
            decoded_token['sub'], 
            'Allow', 
            event['methodArn']
        )
        
        # Add user context
        policy['context'] = {
            'userId': decoded_token['sub'],
            'username': decoded_token.get('username', ''),
            'roles': json.dumps(decoded_token.get('roles', []))
        }
        
        return policy
        
    except Exception as e:
        print(f"Authorization failed: {str(e)}")
        raise Exception('Unauthorized')

def generate_policy(principal_id, effect, resource):
    """Generate IAM policy for API Gateway"""
    return {
        'principalId': principal_id,
        'policyDocument': {
            'Version': '2012-10-17',
            'Statement': [
                {
                    'Action': 'execute-api:Invoke',
                    'Effect': effect,
                    'Resource': resource
                }
            ]
        }
    }
```

## Rate Limiting and Throttling

### API Gateway Throttling

```yaml
# Usage Plan with throttling
UsagePlan:
  Type: AWS::ApiGateway::UsagePlan
  Properties:
    UsagePlanName: StandardPlan
    Description: Standard usage plan with rate limiting
    Throttle:
      RateLimit: 1000     # Requests per second
      BurstLimit: 2000    # Burst capacity
    Quota:
      Limit: 10000        # Total requests per period
      Period: DAY         # DAY, WEEK, MONTH
    ApiStages:
      - ApiId: !Ref RestApi
        Stage: prod
        Throttle:
          '/users/GET':
            RateLimit: 100
            BurstLimit: 200
          '/orders/POST':
            RateLimit: 50
            BurstLimit: 100
```

### Method-Level Throttling

```python
import boto3

def configure_method_throttling():
    client = boto3.client('apigateway')
    
    # Update method settings for throttling
    response = client.update_stage(
        restApiId='your-api-id',
        stageName='prod',
        patchOps=[
            {
                'op': 'replace',
                'path': '/throttle/rate',
                'value': '100'
            },
            {
                'op': 'replace',
                'path': '/throttle/burst',
                'value': '200'
            },
            {
                'op': 'replace',
                'path': '/*/throttle/rate',
                'value': '50'
            }
        ]
    )
    
    return response
```

### Custom Rate Limiting with Lambda

```python
import json
import boto3
from datetime import datetime, timedelta

dynamodb = boto3.resource('dynamodb')
rate_limit_table = dynamodb.Table('RateLimitTable')

def rate_limiter(event, context):
    """Custom rate limiter using DynamoDB"""
    
    # Extract client identifier
    client_id = get_client_id(event)
    current_time = datetime.utcnow()
    window_start = current_time.replace(second=0, microsecond=0)
    
    try:
        # Get current usage
        response = rate_limit_table.get_item(
            Key={
                'client_id': client_id,
                'window': window_start.isoformat()
            }
        )
        
        current_count = response.get('Item', {}).get('count', 0)
        rate_limit = get_rate_limit(client_id)
        
        if current_count >= rate_limit:
            return {
                'statusCode': 429,
                'body': json.dumps({
                    'error': 'Rate limit exceeded',
                    'retry_after': 60
                })
            }
        
        # Increment counter
        rate_limit_table.update_item(
            Key={
                'client_id': client_id,
                'window': window_start.isoformat()
            },
            UpdateExpression='ADD #count :increment SET #ttl = :ttl',
            ExpressionAttributeNames={
                '#count': 'count',
                '#ttl': 'ttl'
            },
            ExpressionAttributeValues={
                ':increment': 1,
                ':ttl': int((window_start + timedelta(minutes=2)).timestamp())
            }
        )
        
        # Continue to backend
        return invoke_backend(event)
        
    except Exception as e:
        print(f"Rate limiting error: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Internal server error'})
        }

def get_client_id(event):
    """Extract client identifier from request"""
    # Try API key first
    if 'x-api-key' in event.get('headers', {}):
        return event['headers']['x-api-key']
    
    # Try IP address
    return event.get('requestContext', {}).get('identity', {}).get('sourceIp', 'unknown')

def get_rate_limit(client_id):
    """Get rate limit for client"""
    # Default rate limit
    default_limit = 100
    
    # Premium clients get higher limits
    premium_clients = {
        'premium-key-1': 1000,
        'premium-key-2': 500
    }
    
    return premium_clients.get(client_id, default_limit)
```

## API Monitoring and Logging

### CloudWatch Integration

```yaml
# API Gateway CloudWatch role
ApiGatewayCloudWatchRole:
  Type: AWS::IAM::Role
  Properties:
    AssumeRolePolicyDocument:
      Version: '2012-10-17'
      Statement:
        - Effect: Allow
          Principal:
            Service: apigateway.amazonaws.com
          Action: sts:AssumeRole
    ManagedPolicyArns:
      - arn:aws:iam::aws:policy/service-role/AmazonAPIGatewayPushToCloudWatchLogs

# API Gateway account settings
ApiGatewayAccount:
  Type: AWS::ApiGateway::Account
  Properties:
    CloudWatchRoleArn: !GetAtt ApiGatewayCloudWatchRole.Arn

# Stage with logging enabled
ApiStage:
  Type: AWS::ApiGateway::Stage
  Properties:
    RestApiId: !Ref RestApi
    DeploymentId: !Ref ApiDeployment
    StageName: prod
    MethodSettings:
      - ResourcePath: '/*'
        HttpMethod: '*'
        LoggingLevel: INFO
        DataTraceEnabled: true
        MetricsEnabled: true
        ThrottlingBurstLimit: 2000
        ThrottlingRateLimit: 1000
```

### Custom Metrics and Alarms

```python
import boto3
import json

cloudwatch = boto3.client('cloudwatch')

def create_api_alarms():
    """Create CloudWatch alarms for API monitoring"""
    
    alarms = [
        {
            'AlarmName': 'API-HighErrorRate',
            'MetricName': '4XXError',
            'Threshold': 10,
            'ComparisonOperator': 'GreaterThanThreshold',
            'EvaluationPeriods': 2,
            'Period': 300
        },
        {
            'AlarmName': 'API-HighLatency',
            'MetricName': 'Latency',
            'Threshold': 5000,
            'ComparisonOperator': 'GreaterThanThreshold',
            'EvaluationPeriods': 3,
            'Period': 300
        },
        {
            'AlarmName': 'API-ThrottleExceeded',
            'MetricName': 'ThrottledCount',
            'Threshold': 0,
            'ComparisonOperator': 'GreaterThanThreshold',
            'EvaluationPeriods': 1,
            'Period': 300
        }
    ]
    
    for alarm in alarms:
        cloudwatch.put_metric_alarm(
            AlarmName=alarm['AlarmName'],
            ComparisonOperator=alarm['ComparisonOperator'],
            EvaluationPeriods=alarm['EvaluationPeriods'],
            MetricName=alarm['MetricName'],
            Namespace='AWS/ApiGateway',
            Period=alarm['Period'],
            Statistic='Sum',
            Threshold=alarm['Threshold'],
            ActionsEnabled=True,
            AlarmActions=[
                'arn:aws:sns:us-east-1:123456789012:api-alerts'
            ],
            AlarmDescription=f'Alarm for {alarm["AlarmName"]}',
            Dimensions=[
                {
                    'Name': 'ApiName',
                    'Value': 'MyAPI'
                },
                {
                    'Name': 'Stage',
                    'Value': 'prod'
                }
            ]
        )

def publish_custom_metric(metric_name, value, unit='Count'):
    """Publish custom metric to CloudWatch"""
    
    cloudwatch.put_metric_data(
        Namespace='Custom/API',
        MetricData=[
            {
                'MetricName': metric_name,
                'Value': value,
                'Unit': unit,
                'Dimensions': [
                    {
                        'Name': 'Environment',
                        'Value': 'production'
                    }
                ]
            }
        ]
    )
```

### X-Ray Tracing

```yaml
# Enable X-Ray tracing for API Gateway
ApiStageWithXRay:
  Type: AWS::ApiGateway::Stage
  Properties:
    RestApiId: !Ref RestApi
    DeploymentId: !Ref ApiDeployment
    StageName: prod
    TracingConfig:
      TracingEnabled: true
```

```python
from aws_xray_sdk.core import xray_recorder
from aws_xray_sdk.core import patch_all
import requests

# Patch AWS SDK calls
patch_all()

@xray_recorder.capture('api_handler')
def lambda_handler(event, context):
    """Lambda function with X-Ray tracing"""
    
    subsegment = xray_recorder.begin_subsegment('external_api_call')
    try:
        # External API call
        response = requests.get('https://api.example.com/data')
        subsegment.put_metadata('response_status', response.status_code)
        
        return {
            'statusCode': 200,
            'body': response.text
        }
    except Exception as e:
        subsegment.add_exception(e)
        raise
    finally:
        xray_recorder.end_subsegment()
```

## DDoS Protection for APIs

### CloudFront Integration

```yaml
CloudFrontDistribution:
  Type: AWS::CloudFront::Distribution
  Properties:
    DistributionConfig:
      Origins:
        - Id: APIGatewayOrigin
          DomainName: !Sub '${RestApi}.execute-api.${AWS::Region}.amazonaws.com'
          OriginPath: '/prod'
          CustomOriginConfig:
            HTTPPort: 443
            HTTPSPort: 443
            OriginProtocolPolicy: https-only
            OriginSSLProtocols:
              - TLSv1.2
      DefaultCacheBehavior:
        TargetOriginId: APIGatewayOrigin
        ViewerProtocolPolicy: redirect-to-https
        CachePolicyId: 4135ea2d-6df8-44a3-9df3-4b5a84be39ad  # CachingDisabled
        OriginRequestPolicyId: 88a5eaf4-2fd4-4709-b370-b4c650ea3fcf  # CORS-S3Origin
        AllowedMethods:
          - DELETE
          - GET
          - HEAD
          - OPTIONS
          - PATCH
          - POST
          - PUT
        Compress: true
      WebACLId: !GetAtt WebACL.Arn
      Enabled: true
      PriceClass: PriceClass_100
```

### AWS WAF Integration

```yaml
# WAF Web ACL
WebACL:
  Type: AWS::WAFv2::WebACL
  Properties:
    Name: APIProtectionWebACL
    Scope: CLOUDFRONT
    DefaultAction:
      Allow: {}
    Rules:
      - Name: AWSManagedRulesCommonRuleSet
        Priority: 1
        OverrideAction:
          None: {}
        Statement:
          ManagedRuleGroupStatement:
            VendorName: AWS
            Name: AWSManagedRulesCommonRuleSet
        VisibilityConfig:
          SampledRequestsEnabled: true
          CloudWatchMetricsEnabled: true
          MetricName: CommonRuleSetMetric
      
      - Name: RateLimitRule
        Priority: 2
        Action:
          Block: {}
        Statement:
          RateBasedStatement:
            Limit: 2000
            AggregateKeyType: IP
        VisibilityConfig:
          SampledRequestsEnabled: true
          CloudWatchMetricsEnabled: true
          MetricName: RateLimitMetric
      
      - Name: IPReputationList
        Priority: 3
        OverrideAction:
          None: {}
        Statement:
          ManagedRuleGroupStatement:
            VendorName: AWS
            Name: AWSManagedRulesAmazonIpReputationList
        VisibilityConfig:
          SampledRequestsEnabled: true
          CloudWatchMetricsEnabled: true
          MetricName: IPReputationMetric

    VisibilityConfig:
      SampledRequestsEnabled: true
      CloudWatchMetricsEnabled: true
      MetricName: APIProtectionWebACL
```

### Application-Layer Protection

```python
import hashlib
import hmac
import time
from functools import wraps

def api_protection(f):
    """Decorator for API endpoint protection"""
    
    @wraps(f)
    def decorated_function(event, context):
        # Validate request signature
        if not validate_signature(event):
            return {
                'statusCode': 401,
                'body': json.dumps({'error': 'Invalid signature'})
            }
        
        # Check timestamp to prevent replay attacks
        if not validate_timestamp(event):
            return {
                'statusCode': 401,
                'body': json.dumps({'error': 'Request too old'})
            }
        
        # Rate limiting
        if is_rate_limited(event):
            return {
                'statusCode': 429,
                'body': json.dumps({'error': 'Rate limit exceeded'})
            }
        
        return f(event, context)
    
    return decorated_function

def validate_signature(event):
    """Validate HMAC signature"""
    try:
        signature = event['headers'].get('x-signature', '')
        timestamp = event['headers'].get('x-timestamp', '')
        body = event.get('body', '')
        
        # Create expected signature
        message = f"{timestamp}.{body}"
        expected_signature = hmac.new(
            SECRET_KEY.encode(),
            message.encode(),
            hashlib.sha256
        ).hexdigest()
        
        return hmac.compare_digest(signature, expected_signature)
    except Exception:
        return False

def validate_timestamp(event):
    """Validate request timestamp"""
    try:
        timestamp = int(event['headers'].get('x-timestamp', '0'))
        current_time = int(time.time())
        
        # Allow 5-minute window
        return abs(current_time - timestamp) <= 300
    except Exception:
        return False
```

## Security Best Practices

### API Design Security

1. **Use HTTPS Everywhere**
```yaml
# Force HTTPS in API Gateway
SecurityHeaders:
  Type: AWS::ApiGateway::GatewayResponse
  Properties:
    RestApiId: !Ref RestApi
    ResponseType: UNAUTHORIZED
    ResponseParameters:
      gatewayresponse.header.Strict-Transport-Security: "'max-age=31536000; includeSubDomains'"
      gatewayresponse.header.X-Content-Type-Options: "'nosniff'"
      gatewayresponse.header.X-Frame-Options: "'DENY'"
      gatewayresponse.header.X-XSS-Protection: "'1; mode=block'"
```

2. **Input Validation**
```yaml
# Request validation model
RequestValidationModel:
  Type: AWS::ApiGateway::Model
  Properties:
    RestApiId: !Ref RestApi
    ContentType: application/json
    Name: UserModel
    Schema:
      type: object
      properties:
        username:
          type: string
          pattern: '^[a-zA-Z0-9_]{3,20}$'
        email:
          type: string
          format: email
        age:
          type: integer
          minimum: 18
          maximum: 120
      required:
        - username
        - email
      additionalProperties: false

# Method with request validation
ValidatedMethod:
  Type: AWS::ApiGateway::Method
  Properties:
    RestApiId: !Ref RestApi
    ResourceId: !Ref UserResource
    HttpMethod: POST
    RequestValidatorId: !Ref RequestValidator
    RequestModels:
      application/json: !Ref RequestValidationModel
```

3. **Output Sanitization**
```python
import re
import html

def sanitize_output(data):
    """Sanitize output data"""
    if isinstance(data, str):
        # HTML escape
        data = html.escape(data)
        # Remove potentially dangerous characters
        data = re.sub(r'[<>"\']', '', data)
    elif isinstance(data, dict):
        return {k: sanitize_output(v) for k, v in data.items()}
    elif isinstance(data, list):
        return [sanitize_output(item) for item in data]
    
    return data
```

### Network Security Layers

```yaml
# Complete security stack
SecurityStack:
  Type: AWS::CloudFormation::Stack
  Properties:
    TemplateURL: security-template.yaml
    Parameters:
      VPCId: !Ref VPC
      PrivateSubnets: !Join [',', [!Ref PrivateSubnet1, !Ref PrivateSubnet2]]
      
# Template includes:
# 1. VPC endpoints for private API access
# 2. Security groups with least privilege
# 3. NACLs for subnet-level filtering
# 4. WAF for application-layer protection
# 5. CloudFront for edge protection
# 6. Shield Advanced for DDoS protection
```

### Monitoring and Alerting

```python
def setup_comprehensive_monitoring():
    """Setup comprehensive API monitoring"""
    
    metrics_to_monitor = [
        {
            'name': 'API_Errors',
            'threshold': 50,
            'period': 300,
            'evaluation_periods': 2
        },
        {
            'name': 'API_Latency',
            'threshold': 10000,
            'period': 300,
            'evaluation_periods': 3
        },
        {
            'name': 'Throttle_Count',
            'threshold': 10,
            'period': 60,
            'evaluation_periods': 1
        },
        {
            'name': 'WAF_Blocked_Requests',
            'threshold': 100,
            'period': 300,
            'evaluation_periods': 2
        }
    ]
    
    for metric in metrics_to_monitor:
        create_alarm(metric)
        create_dashboard_widget(metric)
    
    # Setup automated response
    setup_automated_incident_response()

def setup_automated_incident_response():
    """Setup automated response to security events"""
    
    # Lambda function to handle security alerts
    response_actions = {
        'high_error_rate': 'enable_additional_throttling',
        'ddos_detected': 'activate_shield_response_team',
        'unauthorized_access': 'temporary_api_suspension',
        'suspicious_patterns': 'enhanced_logging_mode'
    }
    
    return response_actions
```

## Real-World Implementation Scenarios

### Enterprise API Security Architecture

```yaml
# Multi-layer enterprise API security
EnterpriseAPIStack:
  Description: Enterprise-grade API security implementation
  
  # Network Layer
  VPCEndpoints:
    - APIGateway Interface Endpoint
    - CloudWatch Interface Endpoint
    - Systems Manager Interface Endpoint
  
  # Application Layer
  SecurityComponents:
    - Custom Domain with TLS 1.3
    - WAF with custom rules
    - API Gateway with private endpoints
    - Lambda authorizers
    - Cognito User Pools
  
  # Monitoring Layer
  ObservabilityStack:
    - CloudWatch dashboards
    - X-Ray tracing
    - Custom metrics
    - Security event correlation
  
  # Response Layer
  IncidentResponse:
    - Automated threat detection
    - Dynamic rate limiting
    - Failover mechanisms
    - Security team alerts
```

### Microservices API Security

For microservices architectures, implement defense in depth:

```python
# Service mesh security configuration
service_mesh_config = {
    "mtls": {
        "enabled": True,
        "mode": "STRICT"
    },
    "authorization": {
        "enabled": True,
        "policies": [
            {
                "service": "user-service",
                "allowed_callers": ["frontend", "admin-service"]
            },
            {
                "service": "payment-service", 
                "allowed_callers": ["order-service"],
                "additional_auth": "jwt_validation"
            }
        ]
    },
    "network_policies": {
        "default_deny": True,
        "ingress_rules": [
            {
                "from": "api-gateway",
                "to": "frontend-service",
                "ports": [80, 443]
            }
        ]
    }
}
```

This comprehensive approach to API security ensures protection at multiple layers, from network-level controls to application-specific security measures, providing robust defense against various threat vectors in cloud environments.