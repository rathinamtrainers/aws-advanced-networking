# Topic 152: Building a Secure API Gateway with Private VPC Access

## Overview

Amazon API Gateway enables you to create RESTful and WebSocket APIs that act as entry points to your backend services. When combined with private VPC access patterns, API Gateway provides secure, controlled access to internal resources without exposing them to the public internet. This topic covers advanced patterns for securing API Gateway integrations with private VPC resources using VPC Links, interface endpoints, and private integrations.

## Learning Objectives

By the end of this lesson, you will be able to:
- Design secure API Gateway architectures with private VPC access
- Configure VPC Links for private integrations with Network Load Balancers
- Implement API Gateway VPC endpoints for private API access
- Apply security controls including resource policies, IAM, and endpoint policies
- Troubleshoot common private API Gateway connectivity issues
- Optimize performance and cost for private API architectures

## Prerequisites

- Understanding of API Gateway fundamentals (REST and HTTP APIs)
- Knowledge of VPC endpoints and PrivateLink (Topic 28)
- Familiarity with Network Load Balancers
- IAM policies and resource-based policies
- Basic understanding of DNS resolution in VPCs

## Architecture Patterns

### Pattern 1: Private API with VPC Endpoint

**Use Case**: API only accessible from within VPCs (no internet access)

**Components**:
- API Gateway Private API
- VPC Interface Endpoint (execute-api)
- VPC endpoint policy
- Resource policy on API Gateway
- Private hosted zone for DNS resolution

**Benefits**:
- Complete isolation from public internet
- Reduced attack surface
- Simplified compliance (PCI, HIPAA)
- Lower data transfer costs (within same region)

### Pattern 2: Public API with Private Integration via VPC Link

**Use Case**: Public-facing API that accesses private VPC resources

**Components**:
- API Gateway Regional or Edge-optimized API
- VPC Link (HTTP API or REST API)
- Network Load Balancer in private subnets
- Backend services (ECS, EKS, EC2, Lambda)
- Security groups and NACLs

**Benefits**:
- Public accessibility with private backend security
- Support for existing private microservices
- PrivateLink-based secure connectivity
- No NAT Gateway required for backend traffic

### Pattern 3: Hybrid Private API with Cross-Account Access

**Use Case**: Private APIs shared across multiple AWS accounts

**Components**:
- API Gateway Private API
- VPC endpoints in multiple accounts/VPCs
- VPC endpoint services (PrivateLink)
- Cross-account IAM roles
- Resource policies with account/org conditions

**Benefits**:
- Centralized API management
- Secure multi-account architecture
- Fine-grained access control
- Simplified network topology

### Pattern 4: Private API with Direct Connect or VPN Access

**Use Case**: On-premises applications accessing AWS-hosted private APIs

**Components**:
- API Gateway Private API
- VPC Interface Endpoint
- AWS Direct Connect or Site-to-Site VPN
- Hybrid DNS resolution (Route 53 Resolver)
- Transit Gateway (optional for multi-VPC)

**Benefits**:
- Extends private API access to on-premises
- No internet exposure required
- Consistent security posture
- Low latency with Direct Connect

## Detailed Implementation Guide

### Lab 1: Creating a Private API with VPC Endpoint Access

#### Step 1: Create the API Gateway Private API

**Theory**: Private APIs can only be accessed from VPC endpoints. This provides complete network isolation and prevents any internet-based access.

**AWS Console Steps**:
1. Navigate to **API Gateway** console
2. Click **Create API** → Choose **REST API** → **Build**
3. Enter API name: `PrivateSecureAPI`
4. Endpoint Type: Select **Private**
5. Click **Create API**

**CLI Commands**:
```bash
# Create the private API
aws apigateway create-rest-api \
  --name PrivateSecureAPI \
  --endpoint-configuration types=PRIVATE \
  --region us-east-1

# Note the API ID from output
export API_ID=<your-api-id>
```

**Expected Output**:
```json
{
    "id": "abc123xyz",
    "name": "PrivateSecureAPI",
    "createdDate": "2025-01-15T10:30:00Z",
    "endpointConfiguration": {
        "types": ["PRIVATE"]
    }
}
```

#### Step 2: Create VPC Interface Endpoint for API Gateway

**Theory**: The execute-api VPC endpoint creates an ENI in your VPC that provides private connectivity to API Gateway. DNS resolution routes API calls through this endpoint instead of the public internet.

**AWS Console Steps**:
1. Navigate to **VPC** → **Endpoints**
2. Click **Create Endpoint**
3. Service category: **AWS services**
4. Service name: `com.amazonaws.us-east-1.execute-api`
5. VPC: Select your VPC (e.g., `vpc-private-api`)
6. Subnets: Select subnets in multiple AZs for HA
7. Security group: Create/select security group allowing HTTPS (443) inbound
8. Policy: Select **Full access** (we'll restrict via resource policy)
9. Enable **Private DNS names**: ✓ (checked)
10. Click **Create endpoint**

**CLI Commands**:
```bash
# Get your VPC ID
export VPC_ID=vpc-0123456789abcdef0

# Get subnet IDs
export SUBNET_IDS="subnet-abc123,subnet-def456"

# Create security group for endpoint
aws ec2 create-security-group \
  --group-name api-gateway-endpoint-sg \
  --description "Security group for API Gateway VPC endpoint" \
  --vpc-id $VPC_ID

export SG_ID=<security-group-id>

# Allow HTTPS inbound from VPC CIDR
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 443 \
  --cidr 10.0.0.0/16

# Create VPC endpoint
aws ec2 create-vpc-endpoint \
  --vpc-id $VPC_ID \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.execute-api \
  --subnet-ids $SUBNET_IDS \
  --security-group-ids $SG_ID \
  --private-dns-enabled

export VPCE_ID=<vpc-endpoint-id>
```

**Verification**:
```bash
# Check endpoint status (should show "available")
aws ec2 describe-vpc-endpoints \
  --vpc-endpoint-ids $VPCE_ID \
  --query 'VpcEndpoints[0].State'

# Verify DNS entries created
aws ec2 describe-vpc-endpoints \
  --vpc-endpoint-ids $VPCE_ID \
  --query 'VpcEndpoints[0].DnsEntries'
```

#### Step 3: Configure API Gateway Resource Policy

**Theory**: Resource policies control which VPCs, VPC endpoints, and accounts can access the private API. This provides defense-in-depth beyond network-level controls.

**AWS Console Steps**:
1. In API Gateway console, select your private API
2. Click **Resource Policy** in left navigation
3. Enter the following policy (replace values):

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:us-east-1:123456789012:abc123xyz/*",
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": "vpce-0123456789abcdef0"
        }
      }
    },
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:us-east-1:123456789012:abc123xyz/*"
    }
  ]
}
```

4. Click **Save**

**CLI Commands**:
```bash
# Create resource policy JSON file
cat > resource-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:us-east-1:123456789012:${API_ID}/*",
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": "${VPCE_ID}"
        }
      }
    },
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:us-east-1:123456789012:${API_ID}/*"
    }
  ]
}
EOF

# Update API resource policy
aws apigateway update-rest-api \
  --rest-api-id $API_ID \
  --patch-operations op=replace,path=/policy,value="$(cat resource-policy.json | jq -c .)"
```

**Policy Explanation**:
- **First statement (Deny)**: Explicitly denies access if request doesn't come from specified VPC endpoint
- **Second statement (Allow)**: Allows access for requests that pass the Deny condition
- **Evaluation order**: Explicit deny is always evaluated first

**Advanced Resource Policy Options**:

Restrict by source VPC:
```json
"Condition": {
  "StringEquals": {
    "aws:sourceVpc": "vpc-0123456789abcdef0"
  }
}
```

Restrict by AWS account:
```json
"Condition": {
  "StringEquals": {
    "aws:PrincipalAccount": "123456789012"
  }
}
```

Restrict by AWS Organization:
```json
"Condition": {
  "StringEquals": {
    "aws:PrincipalOrgID": "o-xxxxxxxxxx"
  }
}
```

#### Step 4: Create API Resources and Methods

**Theory**: For testing, we'll create a simple Lambda-backed API endpoint.

**Create Lambda Function**:
```bash
# Create Lambda execution role
aws iam create-role \
  --role-name lambda-api-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "lambda.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Attach basic execution policy
aws iam attach-role-policy \
  --role-name lambda-api-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Create function code
cat > index.js <<EOF
exports.handler = async (event) => {
    return {
        statusCode: 200,
        body: JSON.stringify({
            message: 'Hello from private API!',
            sourceIp: event.requestContext.identity.sourceIp,
            vpce: event.requestContext.identity.vpce
        })
    };
};
EOF

zip function.zip index.js

# Create Lambda function
aws lambda create-function \
  --function-name PrivateAPIFunction \
  --runtime nodejs20.x \
  --role arn:aws:iam::123456789012:role/lambda-api-role \
  --handler index.handler \
  --zip-file fileb://function.zip

export LAMBDA_ARN=$(aws lambda get-function --function-name PrivateAPIFunction --query 'Configuration.FunctionArn' --output text)
```

**Create API Gateway Resource and Method**:
```bash
# Get root resource ID
export ROOT_RESOURCE_ID=$(aws apigateway get-resources \
  --rest-api-id $API_ID \
  --query 'items[0].id' \
  --output text)

# Create /hello resource
aws apigateway create-resource \
  --rest-api-id $API_ID \
  --parent-id $ROOT_RESOURCE_ID \
  --path-part hello

export RESOURCE_ID=$(aws apigateway get-resources \
  --rest-api-id $API_ID \
  --query 'items[?path==`/hello`].id' \
  --output text)

# Create GET method
aws apigateway put-method \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method GET \
  --authorization-type NONE

# Create Lambda integration
aws apigateway put-integration \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method GET \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/$LAMBDA_ARN/invocations

# Grant API Gateway permission to invoke Lambda
aws lambda add-permission \
  --function-name PrivateAPIFunction \
  --statement-id apigateway-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:123456789012:${API_ID}/*"
```

#### Step 5: Deploy the API

**CLI Commands**:
```bash
# Create deployment
aws apigateway create-deployment \
  --rest-api-id $API_ID \
  --stage-name prod \
  --description "Production deployment"

# Get invoke URL
echo "API Invoke URL: https://${API_ID}.execute-api.us-east-1.amazonaws.com/prod/hello"
```

#### Step 6: Test Private API Access

**From EC2 Instance in VPC**:
```bash
# SSH into EC2 instance in same VPC
# Test API access
curl -v https://${API_ID}.execute-api.us-east-1.amazonaws.com/prod/hello

# Expected response
{
  "message": "Hello from private API!",
  "sourceIp": "10.0.1.50",
  "vpce": "vpce-0123456789abcdef0"
}

# Verify DNS resolution uses VPC endpoint
dig ${API_ID}.execute-api.us-east-1.amazonaws.com

# Should resolve to private IP addresses (10.x.x.x)
```

**From Outside VPC (Should Fail)**:
```bash
# From local machine or different VPC
curl https://${API_ID}.execute-api.us-east-1.amazonaws.com/prod/hello

# Expected error
{"message":"Forbidden"}
```

**Verification Checklist**:
- ✓ API accessible from VPC instances
- ✓ API returns VPC endpoint ID in response
- ✓ DNS resolves to private IPs
- ✓ Access denied from outside VPC
- ✓ CloudWatch logs show successful invocations

### Lab 2: Public API with Private Integration using VPC Link

#### Step 1: Create Network Load Balancer in Private Subnets

**Theory**: VPC Link uses PrivateLink to connect API Gateway to a Network Load Balancer, which distributes traffic to backend services in private subnets.

**AWS Console Steps**:
1. Navigate to **EC2** → **Load Balancers**
2. Click **Create Load Balancer** → **Network Load Balancer**
3. Name: `private-api-backend-nlb`
4. Scheme: **Internal**
5. IP address type: IPv4
6. VPC: Select your VPC
7. Subnets: Select private subnets in multiple AZs
8. Security groups: (NLB doesn't use security groups, but targets do)
9. Listeners:
   - Protocol: TCP
   - Port: 80
   - Default action: Forward to target group (create new)
10. Target group configuration:
    - Target type: IP addresses (or Instances/Lambda)
    - Name: `api-backend-targets`
    - Protocol: TCP
    - Port: 80
    - Health check protocol: HTTP
    - Health check path: `/health`
11. Register targets (your backend services)
12. Click **Create**

**CLI Commands**:
```bash
# Create target group
aws elbv2 create-target-group \
  --name api-backend-targets \
  --protocol TCP \
  --port 80 \
  --vpc-id $VPC_ID \
  --target-type ip \
  --health-check-protocol HTTP \
  --health-check-path /health \
  --health-check-interval-seconds 30

export TG_ARN=$(aws elbv2 describe-target-groups \
  --names api-backend-targets \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

# Create NLB
aws elbv2 create-load-balancer \
  --name private-api-backend-nlb \
  --type network \
  --scheme internal \
  --subnets subnet-private1 subnet-private2

export NLB_ARN=$(aws elbv2 describe-load-balancers \
  --names private-api-backend-nlb \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

# Create listener
aws elbv2 create-listener \
  --load-balancer-arn $NLB_ARN \
  --protocol TCP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN

# Register targets (example with IP addresses)
aws elbv2 register-targets \
  --target-group-arn $TG_ARN \
  --targets Id=10.0.10.100 Id=10.0.20.100
```

**Deploy Sample Backend Service (Optional)**:
```bash
# Simple backend on EC2 or ECS
# Example: Python Flask app
cat > app.py <<EOF
from flask import Flask, jsonify
app = Flask(__name__)

@app.route('/health')
def health():
    return 'OK', 200

@app.route('/api/data')
def data():
    return jsonify({'message': 'Data from private backend'})

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=80)
EOF
```

#### Step 2: Create VPC Link

**Theory**: VPC Link creates a PrivateLink connection that API Gateway uses to route requests to your NLB. This connection is private and doesn't traverse the internet.

**AWS Console Steps** (REST API VPC Link):
1. Navigate to **API Gateway** → **VPC Links** (under **Settings**)
2. Click **Create**
3. Name: `private-backend-vpclink`
4. Target NLB: Select `private-api-backend-nlb`
5. Click **Create**
6. Wait for status: **Available** (takes 2-5 minutes)

**CLI Commands** (REST API):
```bash
# Create VPC Link for REST API
aws apigateway create-vpc-link \
  --name private-backend-vpclink \
  --target-arns $NLB_ARN

export VPCLINK_ID=$(aws apigateway get-vpc-links \
  --query 'items[?name==`private-backend-vpclink`].id' \
  --output text)

# Check status
aws apigateway get-vpc-link --vpc-link-id $VPCLINK_ID \
  --query 'status'
# Wait until "AVAILABLE"
```

**For HTTP API (different syntax)**:
```bash
# Create VPC Link for HTTP API
aws apigatewayv2 create-vpc-link \
  --name private-backend-vpclink-http \
  --subnet-ids subnet-private1 subnet-private2 \
  --security-group-ids $SG_ID

export VPCLINK_V2_ID=$(aws apigatewayv2 get-vpc-links \
  --query 'Items[?Name==`private-backend-vpclink-http`].VpcLinkId' \
  --output text)
```

#### Step 3: Create Public API with Private Integration

**Theory**: The API is publicly accessible, but all backend traffic routes through the VPC Link to private resources.

**AWS Console Steps**:
1. Navigate to **API Gateway** → **Create API**
2. Choose **REST API** → **Build**
3. Name: `PublicAPIPrivateBackend`
4. Endpoint type: **Regional**
5. Click **Create API**
6. Create resource: `/data`
7. Create method: **GET**
8. Integration type: **VPC Link**
9. VPC Link: Select `private-backend-vpclink`
10. Endpoint URL: `http://private-api-backend-nlb.elb.us-east-1.amazonaws.com/api/data`
11. Click **Save**
12. Deploy to stage `prod`

**CLI Commands**:
```bash
# Create public API
aws apigateway create-rest-api \
  --name PublicAPIPrivateBackend \
  --endpoint-configuration types=REGIONAL

export PUBLIC_API_ID=<api-id>

# Get root resource
export ROOT_ID=$(aws apigateway get-resources \
  --rest-api-id $PUBLIC_API_ID \
  --query 'items[0].id' \
  --output text)

# Create /data resource
aws apigateway create-resource \
  --rest-api-id $PUBLIC_API_ID \
  --parent-id $ROOT_ID \
  --path-part data

export DATA_RESOURCE_ID=$(aws apigateway get-resources \
  --rest-api-id $PUBLIC_API_ID \
  --query 'items[?path==`/data`].id' \
  --output text)

# Create GET method
aws apigateway put-method \
  --rest-api-id $PUBLIC_API_ID \
  --resource-id $DATA_RESOURCE_ID \
  --http-method GET \
  --authorization-type NONE

# Get NLB DNS name
export NLB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $NLB_ARN \
  --query 'LoadBalancers[0].DNSName' \
  --output text)

# Create VPC Link integration
aws apigateway put-integration \
  --rest-api-id $PUBLIC_API_ID \
  --resource-id $DATA_RESOURCE_ID \
  --http-method GET \
  --type HTTP_PROXY \
  --integration-http-method GET \
  --connection-type VPC_LINK \
  --connection-id $VPCLINK_ID \
  --uri "http://${NLB_DNS}/api/data"

# Deploy
aws apigateway create-deployment \
  --rest-api-id $PUBLIC_API_ID \
  --stage-name prod
```

#### Step 4: Test Public API with Private Integration

**From Internet**:
```bash
# Test from your local machine
curl https://${PUBLIC_API_ID}.execute-api.us-east-1.amazonaws.com/prod/data

# Expected response
{
  "message": "Data from private backend"
}

# Check CloudWatch logs
aws logs tail /aws/apigateway/${PUBLIC_API_ID} --follow
```

**Verify Private Network Path**:
```bash
# From EC2 instance in VPC, check NLB targets are healthy
aws elbv2 describe-target-health \
  --target-group-arn $TG_ARN

# VPC Flow Logs should show traffic from VPC Link ENIs to NLB
```

## Security Best Practices

### 1. Defense in Depth Strategy

**Layer 1: Network Controls**
- VPC endpoints with security groups restricting source IPs
- NACLs on subnets containing endpoint ENIs
- Private subnets for backend resources

**Layer 2: API Gateway Resource Policy**
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Deny",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": "execute-api:/*",
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": ["vpce-123", "vpce-456"]
        }
      }
    },
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": "execute-api:/*",
      "Condition": {
        "IpAddress": {
          "aws:sourceIp": ["10.0.0.0/8", "172.16.0.0/12"]
        }
      }
    }
  ]
}
```

**Layer 3: IAM Authorization**
- Use IAM authorization for API methods
- Enforce SigV4 signing for API requests
- Leverage IAM policies on caller principals

**Layer 4: Method-Level Authorization**
- Lambda authorizers for custom logic
- Cognito user pools for user authentication
- OAuth 2.0 / OIDC integration

### 2. VPC Endpoint Security

**Endpoint Policy Example** (restrict to specific APIs):
```json
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "execute-api:Invoke",
      "Resource": [
        "arn:aws:execute-api:us-east-1:123456789012:abc123xyz/*",
        "arn:aws:execute-api:us-east-1:123456789012:def456uvw/*"
      ]
    }
  ]
}
```

**Security Group Rules**:
```bash
# Inbound: Allow HTTPS only from application subnet CIDR
aws ec2 authorize-security-group-ingress \
  --group-id $ENDPOINT_SG_ID \
  --protocol tcp \
  --port 443 \
  --cidr 10.0.1.0/24 \
  --description "HTTPS from application subnet"

# Outbound: Allow to API Gateway service (implicitly allowed)
```

### 3. Cross-Account Access Controls

**Scenario**: Account A hosts private API, Account B needs access

**Account A** (API owner):
1. Resource policy allows Account B:
```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::ACCOUNT-B:root"
  },
  "Action": "execute-api:Invoke",
  "Resource": "execute-api:/*",
  "Condition": {
    "StringEquals": {
      "aws:sourceVpce": "vpce-account-b-endpoint"
    }
  }
}
```

**Account B** (consumer):
1. Create VPC endpoint to API Gateway
2. Create IAM role with permission:
```json
{
  "Effect": "Allow",
  "Action": "execute-api:Invoke",
  "Resource": "arn:aws:execute-api:us-east-1:ACCOUNT-A:abc123xyz/*"
}
```
3. Applications assume role and call API

### 4. Monitoring and Logging

**Enable Execution Logging**:
```bash
# Create CloudWatch Logs role for API Gateway
aws iam create-role \
  --role-name APIGatewayCloudWatchRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "apigateway.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

aws iam attach-role-policy \
  --role-name APIGatewayCloudWatchRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonAPIGatewayPushToCloudWatchLogs

# Set account-level logging role
aws apigateway update-account \
  --patch-operations op=replace,path=/cloudwatchRoleArn,value=arn:aws:iam::123456789012:role/APIGatewayCloudWatchRole

# Enable stage-level logging
aws apigateway update-stage \
  --rest-api-id $API_ID \
  --stage-name prod \
  --patch-operations \
    op=replace,path=/accessLogSettings/destinationArn,value=arn:aws:logs:us-east-1:123456789012:log-group:api-gateway-logs \
    op=replace,path=/accessLogSettings/format,value='$context.requestId $context.extendedRequestId $context.identity.sourceIp $context.identity.vpce' \
    op=replace,path=/*/*/logging/dataTrace,value=true \
    op=replace,path=/*/*/logging/loglevel,value=INFO
```

**Key Metrics to Monitor**:
- `Count` - Total API requests
- `4XXError` - Client errors (check authentication/authorization issues)
- `5XXError` - Server errors (backend/integration problems)
- `Latency` - End-to-end latency
- `IntegrationLatency` - Backend response time
- `CacheHitCount` / `CacheMissCount` - Cache effectiveness

**VPC Flow Logs**:
```bash
# Enable flow logs for endpoint subnet
aws ec2 create-flow-logs \
  --resource-type Subnet \
  --resource-ids subnet-endpoint1 subnet-endpoint2 \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs \
  --deliver-logs-permission-arn arn:aws:iam::123456789012:role/VPCFlowLogsRole
```

### 5. Data Protection

**TLS Configuration**:
- Minimum TLS 1.2 for API Gateway endpoints
- Enable TLS 1.3 where supported
- Use AWS Certificate Manager for custom domains
- Enforce HTTPS-only (reject HTTP)

**Request/Response Validation**:
```json
// Define request model
{
  "$schema": "http://json-schema.org/draft-04/schema#",
  "type": "object",
  "properties": {
    "userId": {"type": "string", "pattern": "^[a-zA-Z0-9-]+$"},
    "action": {"type": "string", "enum": ["read", "write"]}
  },
  "required": ["userId", "action"]
}
```

**WAF Integration** (for public-facing APIs):
```bash
# Create WAF WebACL
aws wafv2 create-web-acl \
  --name api-gateway-waf \
  --scope REGIONAL \
  --region us-east-1 \
  --default-action Allow={} \
  --rules file://waf-rules.json

# Associate with API Gateway stage
aws wafv2 associate-web-acl \
  --web-acl-arn arn:aws:wafv2:us-east-1:123456789012:regional/webacl/api-gateway-waf/abc123 \
  --resource-arn arn:aws:apigateway:us-east-1::/restapis/${API_ID}/stages/prod
```

## Performance Optimization

### 1. Caching Strategy

**Enable API Gateway Caching**:
```bash
# Enable cache (0.5GB to 237GB)
aws apigateway update-stage \
  --rest-api-id $API_ID \
  --stage-name prod \
  --patch-operations \
    op=replace,path=/cacheClusterEnabled,value=true \
    op=replace,path=/cacheClusterSize,value=0.5

# Set cache TTL per method (seconds)
aws apigateway update-method \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method GET \
  --patch-operations \
    op=replace,path=/caching/ttlInSeconds,value=300 \
    op=replace,path=/caching/enabled,value=true
```

**Cache Key Parameters**:
```json
{
  "requestParameters": {
    "method.request.querystring.userId": true,
    "method.request.header.Authorization": false
  },
  "cacheKeyParameters": [
    "method.request.querystring.userId"
  ]
}
```

### 2. Connection Pooling and Keep-Alive

**NLB Target Group Configuration**:
```bash
# Enable connection reuse
aws elbv2 modify-target-group-attributes \
  --target-group-arn $TG_ARN \
  --attributes \
    Key=deregistration_delay.timeout_seconds,Value=30 \
    Key=preserve_client_ip.enabled,Value=false
```

**Backend Service Configuration**:
- Enable HTTP keep-alive on backend servers
- Configure appropriate connection timeout (e.g., 60 seconds)
- Use connection pooling in application code

### 3. Request Throttling

**Stage-Level Throttling**:
```bash
aws apigateway update-stage \
  --rest-api-id $API_ID \
  --stage-name prod \
  --patch-operations \
    op=replace,path=/throttle/rateLimit,value=1000 \
    op=replace,path=/throttle/burstLimit,value=2000
```

**Method-Level Throttling**:
```bash
aws apigateway update-method \
  --rest-api-id $API_ID \
  --resource-id $RESOURCE_ID \
  --http-method GET \
  --patch-operations \
    op=replace,path=/throttle/rateLimit,value=100 \
    op=replace,path=/throttle/burstLimit,value=200
```

**Usage Plans for API Keys**:
```bash
# Create usage plan
aws apigateway create-usage-plan \
  --name premium-plan \
  --throttle rateLimit=5000,burstLimit=10000 \
  --quota limit=1000000,period=MONTH \
  --api-stages apiId=${API_ID},stage=prod
```

## Cost Optimization

### 1. HTTP API vs REST API

**Cost Comparison**:
- **HTTP API**: $1.00 per million requests (71% cheaper)
- **REST API**: $3.50 per million requests
- **Data Transfer**: Same for both

**When to use HTTP API**:
- Simple proxy integrations
- JWT authorizers sufficient
- No need for API keys, usage plans, or request/response transformation
- WebSocket not required

**When to use REST API**:
- Need API keys and usage plans
- Complex request/response mapping
- Caching required
- Legacy integrations (SOAP, XML)

### 2. Data Transfer Optimization

**Strategies**:
- Use compression (gzip) for responses
- Implement API Gateway caching to reduce backend calls
- Place API Gateway in same region as backend services
- Use VPC endpoints to avoid NAT Gateway data transfer charges

**Cost Example** (us-east-1):
```
Scenario: 100M requests/month, 5KB avg response
- API Gateway: 100M * $3.50/M = $350
- Data Transfer (without VPC endpoint): 100M * 5KB * $0.09/GB = $45
- Data Transfer (with VPC endpoint): $0 (same AZ)
- NAT Gateway (avoided with VPC Link): $0
Total savings: $45/month
```

### 3. CloudWatch Logs Optimization

**Selective Logging**:
```bash
# Only log errors (4XX/5XX)
aws logs put-metric-filter \
  --log-group-name /aws/apigateway/${API_ID} \
  --filter-name api-errors \
  --filter-pattern '[request_id, ..., status_code = 4* || status_code = 5*]' \
  --metric-transformations \
    metricName=APIErrors,metricNamespace=CustomAPI,metricValue=1

# Set log retention
aws logs put-retention-policy \
  --log-group-name /aws/apigateway/${API_ID} \
  --retention-in-days 7
```

## Troubleshooting Guide

### Issue 1: Private API Returns 403 Forbidden

**Symptoms**:
- API accessible from some VPC instances but not others
- Error: `{"message":"Forbidden"}`

**Diagnostic Steps**:
```bash
# 1. Check VPC endpoint status
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids $VPCE_ID

# 2. Verify DNS resolution
nslookup ${API_ID}.execute-api.us-east-1.amazonaws.com

# Should return private IPs (10.x.x.x, 172.x.x.x)
# If returns public IPs, Private DNS not working

# 3. Check security group
aws ec2 describe-security-groups --group-ids $SG_ID

# 4. Test connectivity to endpoint
telnet <endpoint-eni-ip> 443

# 5. Check resource policy
aws apigateway get-rest-api --rest-api-id $API_ID --query 'policy'

# 6. Check VPC endpoint policy
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids $VPCE_ID \
  --query 'VpcEndpoints[0].PolicyDocument'
```

**Common Causes**:

| Cause | Symptoms | Solution |
|-------|----------|----------|
| Private DNS not enabled | DNS resolves to public IPs | Enable Private DNS on VPC endpoint |
| Wrong VPC endpoint in resource policy | All requests denied | Update resource policy with correct `aws:sourceVpce` |
| Security group blocks port 443 | Connection timeout | Allow inbound 443 from source CIDR |
| Route 53 Resolver not working | DNS resolution fails | Check VPC DNS settings (enableDnsHostnames, enableDnsSupport) |
| Wrong VPC/subnet for endpoint | Can't reach endpoint | Create endpoint in same VPC as application |

**Resolution**:
```bash
# Enable Private DNS if not enabled
aws ec2 modify-vpc-endpoint \
  --vpc-endpoint-id $VPCE_ID \
  --private-dns-enabled

# Fix security group
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 443 \
  --cidr 10.0.0.0/16

# Update resource policy
aws apigateway update-rest-api \
  --rest-api-id $API_ID \
  --patch-operations op=replace,path=/policy,value='<updated-policy>'
```

### Issue 2: VPC Link Integration Returns 500/503

**Symptoms**:
- Public API works, but returns 500 or 503 errors
- CloudWatch logs show `INTEGRATION_FAILURE` or `INTEGRATION_TIMEOUT`

**Diagnostic Steps**:
```bash
# 1. Check VPC Link status
aws apigateway get-vpc-link --vpc-link-id $VPCLINK_ID

# Status must be "AVAILABLE"

# 2. Check NLB health
aws elbv2 describe-target-health --target-group-arn $TG_ARN

# All targets should show "healthy"

# 3. Test NLB directly from VPC
curl http://<nlb-dns-name>/api/data

# 4. Check NLB listener rules
aws elbv2 describe-listeners --load-balancer-arn $NLB_ARN

# 5. Check API Gateway integration configuration
aws apigateway get-integration \
  --rest-api-id $PUBLIC_API_ID \
  --resource-id $RESOURCE_ID \
  --http-method GET

# 6. Review CloudWatch logs
aws logs tail /aws/apigateway/${PUBLIC_API_ID}/prod --follow
```

**Common Causes**:

| Cause | Error Code | Solution |
|-------|------------|----------|
| NLB targets unhealthy | 503 | Fix backend health check endpoint |
| VPC Link not available | 500 | Wait for VPC Link creation to complete |
| Wrong NLB DNS in integration URI | 500 | Update integration with correct URI |
| Backend timeout (>29s) | 504 | Optimize backend or use async pattern |
| Security group blocks NLB → backend | 503 | Allow NLB traffic to backend targets |
| NLB in wrong subnets | 503 | Ensure NLB subnets route to backend |

**Resolution**:
```bash
# Fix unhealthy targets
# Check backend security group
aws ec2 authorize-security-group-ingress \
  --group-id $BACKEND_SG_ID \
  --protocol tcp \
  --port 80 \
  --source-group $NLB_SG_ID

# Adjust health check settings
aws elbv2 modify-target-group \
  --target-group-arn $TG_ARN \
  --health-check-interval-seconds 10 \
  --health-check-timeout-seconds 5 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2

# Update integration timeout (max 29 seconds)
aws apigateway update-integration \
  --rest-api-id $PUBLIC_API_ID \
  --resource-id $RESOURCE_ID \
  --http-method GET \
  --patch-operations op=replace,path=/timeoutInMillis,value=29000
```

### Issue 3: Cross-Account Private API Access Denied

**Symptoms**:
- Consumer account cannot access producer account's private API
- 403 Forbidden despite correct VPC endpoint setup

**Diagnostic Steps**:
```bash
# In producer account
# 1. Check resource policy allows consumer account
aws apigateway get-rest-api \
  --rest-api-id $API_ID \
  --query 'policy' \
  | jq .

# 2. Verify VPC endpoint ID in resource policy
aws ec2 describe-vpc-endpoints \
  --filters Name=tag:Account,Values=Consumer \
  --query 'VpcEndpoints[*].[VpcEndpointId,VpcId]'

# In consumer account
# 3. Check IAM permissions
aws iam get-role-policy \
  --role-name ApplicationRole \
  --policy-name APIAccessPolicy

# 4. Test with explicit credentials
aws execute-api invoke \
  --endpoint-url https://${API_ID}.execute-api.us-east-1.amazonaws.com \
  --rest-api-id $API_ID \
  --stage-name prod \
  --path /hello \
  output.json
```

**Common Causes**:
- Resource policy doesn't include consumer's VPC endpoint ID
- Resource policy allows account but has conflicting Deny statement
- Consumer's IAM policy doesn't grant `execute-api:Invoke`
- Consumer's VPC endpoint has restrictive endpoint policy

**Resolution**:
```json
// Producer account: Update resource policy
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::CONSUMER-ACCOUNT:root"
      },
      "Action": "execute-api:Invoke",
      "Resource": "execute-api:/*",
      "Condition": {
        "StringEquals": {
          "aws:sourceVpce": ["vpce-consumer-endpoint"]
        }
      }
    }
  ]
}

// Consumer account: IAM policy
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:us-east-1:PRODUCER-ACCOUNT:${API_ID}/*"
    }
  ]
}
```

### Issue 4: High Latency for Private API Calls

**Symptoms**:
- API Gateway latency > 100ms
- IntegrationLatency significantly higher than backend response time

**Diagnostic Steps**:
```bash
# 1. Check CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApiGateway \
  --metric-name Latency \
  --dimensions Name=ApiName,Value=$API_ID \
  --start-time 2025-01-15T00:00:00Z \
  --end-time 2025-01-15T23:59:59Z \
  --period 300 \
  --statistics Average,Maximum

# Also check IntegrationLatency

# 2. Review VPC endpoint placement
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids $VPCE_ID \
  --query 'VpcEndpoints[0].SubnetIds'

# 3. Check VPC Flow Logs for dropped packets
aws logs filter-log-events \
  --log-group-name /aws/vpc/flowlogs \
  --filter-pattern '[version, account, eni, source, destination, srcport, destport, protocol, packets, bytes, start, end, action=REJECT, status]'

# 4. Test direct connectivity
time curl -w "\nTotal: %{time_total}s\n" \
  https://${API_ID}.execute-api.us-east-1.amazonaws.com/prod/hello
```

**Common Causes**:
- VPC endpoint ENIs in different AZ than callers (cross-AZ latency)
- DNS caching issues causing stale endpoint IPs
- Backend service slow (check IntegrationLatency)
- Cold start for Lambda integrations
- API Gateway throttling causing retries

**Optimization Steps**:
```bash
# 1. Place endpoint ENIs in same AZs as application
aws ec2 modify-vpc-endpoint \
  --vpc-endpoint-id $VPCE_ID \
  --add-subnet-ids subnet-in-same-az-as-app

# 2. Enable API Gateway caching
aws apigateway update-stage \
  --rest-api-id $API_ID \
  --stage-name prod \
  --patch-operations \
    op=replace,path=/cacheClusterEnabled,value=true \
    op=replace,path=/cacheClusterSize,value=0.5

# 3. For Lambda: increase provisioned concurrency
aws lambda put-provisioned-concurrency-config \
  --function-name PrivateAPIFunction \
  --provisioned-concurrent-executions 5 \
  --qualifier prod

# 4. Optimize backend service
# - Enable connection pooling
# - Reduce database query time
# - Add caching layer (ElastiCache)

# 5. Implement client-side DNS caching
# In application code, cache DNS lookups for 60 seconds
```

## Advanced Architecture Patterns

### Pattern A: Multi-Region Private API with Route 53

**Use Case**: High availability private API across multiple regions

**Architecture**:
```
┌─────────────────────────────────────────────────────────┐
│                    Route 53 Private Hosted Zone          │
│              api.internal.example.com                    │
│         (Health Check Enabled, Failover Routing)         │
└──────────────┬──────────────────────────────┬────────────┘
               │                              │
       ┌───────▼────────┐           ┌────────▼────────┐
       │   us-east-1    │           │   us-west-2     │
       │  API Gateway   │           │  API Gateway    │
       │  (Private API) │           │  (Private API)  │
       └────────┬───────┘           └────────┬────────┘
                │                            │
       ┌────────▼───────┐           ┌────────▼────────┐
       │  VPC Endpoint  │           │  VPC Endpoint   │
       │  us-east-1-vpc │           │  us-west-2-vpc  │
       └────────────────┘           └─────────────────┘
```

**Implementation**:
```bash
# Create private hosted zone
aws route53 create-hosted-zone \
  --name api.internal.example.com \
  --vpc VPCRegion=us-east-1,VPCId=$VPC_ID_EAST \
  --caller-reference $(date +%s)

export ZONE_ID=<hosted-zone-id>

# Associate zone with additional VPCs
aws route53 associate-vpc-with-hosted-zone \
  --hosted-zone-id $ZONE_ID \
  --vpc VPCRegion=us-west-2,VPCId=$VPC_ID_WEST

# Create health checks for both APIs
aws route53 create-health-check \
  --health-check-config \
    Type=HTTPS,ResourcePath=/health,FullyQualifiedDomainName=${API_ID_EAST}.execute-api.us-east-1.amazonaws.com,Port=443

# Create failover records
aws route53 change-resource-record-sets \
  --hosted-zone-id $ZONE_ID \
  --change-batch file://failover-records.json

# failover-records.json
{
  "Changes": [
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.internal.example.com",
        "Type": "A",
        "SetIdentifier": "Primary-US-East",
        "Failover": "PRIMARY",
        "AliasTarget": {
          "HostedZoneId": "Z1UJRXOUMOOFQ8",
          "DNSName": "${API_ID_EAST}.execute-api.us-east-1.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    },
    {
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.internal.example.com",
        "Type": "A",
        "SetIdentifier": "Secondary-US-West",
        "Failover": "SECONDARY",
        "AliasTarget": {
          "HostedZoneId": "Z2OJLYMUO9EFXC",
          "DNSName": "${API_ID_WEST}.execute-api.us-west-2.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }
  ]
}
```

### Pattern B: Private API with On-Premises Access via Transit Gateway

**Use Case**: On-premises applications access AWS private APIs via Direct Connect

**Architecture**:
```
┌──────────────────┐
│  On-Premises DC  │
│                  │
└────────┬─────────┘
         │ Direct Connect
         │ (Private VIF)
┌────────▼────────────────────────────────────┐
│            Transit Gateway                   │
│  ┌──────────────┬──────────────────────┐   │
│  │  Attachment  │    Attachment        │   │
│  └──────┬───────┴─────────┬────────────┘   │
└─────────┼─────────────────┼─────────────────┘
          │                 │
    ┌─────▼──────┐    ┌────▼──────────┐
    │  Shared    │    │  Application  │
    │ Services   │    │  VPC          │
    │  VPC       │    └───────────────┘
    │            │
    │ ┌────────┐ │
    │ │API GW  │ │
    │ │VPC     │ │
    │ │Endpoint│ │
    │ └───┬────┘ │
    └─────┼──────┘
          │
    ┌─────▼──────┐
    │ API Gateway│
    │ Private API│
    └────────────┘
```

**Implementation**:
```bash
# 1. Create Transit Gateway
aws ec2 create-transit-gateway \
  --description "Hybrid connectivity TGW" \
  --options AmazonSideAsn=64512,DefaultRouteTableAssociation=enable,DefaultRouteTablePropagation=enable

export TGW_ID=<tgw-id>

# 2. Attach VPCs
aws ec2 create-transit-gateway-vpc-attachment \
  --transit-gateway-id $TGW_ID \
  --vpc-id $SHARED_SERVICES_VPC_ID \
  --subnet-ids subnet-1 subnet-2

# 3. Attach Direct Connect Gateway
aws ec2 create-transit-gateway-attachment \
  --transit-gateway-id $TGW_ID \
  --resource-id $DX_GATEWAY_ID \
  --resource-type direct-connect-gateway

# 4. Create VPC endpoint in shared services VPC
aws ec2 create-vpc-endpoint \
  --vpc-id $SHARED_SERVICES_VPC_ID \
  --vpc-endpoint-type Interface \
  --service-name com.amazonaws.us-east-1.execute-api \
  --subnet-ids subnet-shared-1 subnet-shared-2 \
  --security-group-ids $SG_ID \
  --private-dns-enabled

# 5. Configure Route 53 Resolver for hybrid DNS
# Create inbound endpoint for on-premises DNS queries
aws route53resolver create-resolver-endpoint \
  --name on-prem-inbound \
  --direction INBOUND \
  --creator-request-id $(date +%s) \
  --security-group-ids $RESOLVER_SG_ID \
  --ip-addresses SubnetId=subnet-shared-1 SubnetId=subnet-shared-2

# 6. Update on-premises DNS to forward *.execute-api.amazonaws.com queries to Resolver IPs

# 7. Test from on-premises
curl https://${API_ID}.execute-api.us-east-1.amazonaws.com/prod/hello
```

### Pattern C: Private API with AWS PrivateLink for SaaS

**Use Case**: SaaS provider offers private connectivity to customers

**Architecture**:
```
┌────────────────────────────────────┐
│    SaaS Provider Account           │
│                                    │
│  ┌──────────────┐                 │
│  │ API Gateway  │                 │
│  │ Private API  │                 │
│  └──────┬───────┘                 │
│         │                          │
│  ┌──────▼───────┐                 │
│  │ VPC Endpoint │                 │
│  │   Service    │◄────────────┐   │
│  │ (PrivateLink)│             │   │
│  └──────────────┘             │   │
└───────────────────────────────┼───┘
                                │
                   PrivateLink Connection
                                │
┌───────────────────────────────┼───┐
│    Customer Account           │   │
│                               │   │
│  ┌────────────────────────────▼─┐ │
│  │ VPC Endpoint to SaaS Service│ │
│  └────────────┬─────────────────┘ │
│               │                    │
│  ┌────────────▼─────────────────┐ │
│  │  Customer Applications       │ │
│  │  (Access SaaS API privately) │ │
│  └──────────────────────────────┘ │
└────────────────────────────────────┘
```

**Implementation** (SaaS Provider):
```bash
# 1. Create VPC Endpoint Service
aws ec2 create-vpc-endpoint-service-configuration \
  --network-load-balancer-arns $NLB_ARN \
  --acceptance-required

export SERVICE_NAME=<service-name>
# Example: com.amazonaws.vpce.us-east-1.vpce-svc-123456

# 2. Whitelist customer accounts
aws ec2 modify-vpc-endpoint-service-permissions \
  --service-id $SERVICE_ID \
  --add-allowed-principals arn:aws:iam::CUSTOMER-ACCOUNT:root

# 3. Share service name with customer
echo $SERVICE_NAME
```

**Implementation** (Customer):
```bash
# 1. Create VPC endpoint to SaaS service
aws ec2 create-vpc-endpoint \
  --vpc-id $CUSTOMER_VPC_ID \
  --vpc-endpoint-type Interface \
  --service-name $SERVICE_NAME \
  --subnet-ids subnet-1 subnet-2 \
  --security-group-ids $SG_ID

# 2. Wait for SaaS provider to accept connection

# 3. Create private hosted zone for custom domain
aws route53 create-hosted-zone \
  --name api.saas-provider.com \
  --vpc VPCRegion=us-east-1,VPCId=$CUSTOMER_VPC_ID \
  --caller-reference $(date +%s)

# 4. Create A record pointing to VPC endpoint
aws route53 change-resource-record-sets \
  --hosted-zone-id $ZONE_ID \
  --change-batch '{
    "Changes": [{
      "Action": "CREATE",
      "ResourceRecordSet": {
        "Name": "api.saas-provider.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z1HUB23UULQXV",
          "DNSName": "vpce-abc123.execute-api.us-east-1.vpce.amazonaws.com",
          "EvaluateTargetHealth": false
        }
      }
    }]
  }'

# 5. Access SaaS API privately
curl https://api.saas-provider.com/v1/data
```

## Exam Tips & Key Takeaways

### Common Exam Scenarios

1. **Scenario**: Application in VPC needs to call API Gateway without internet access
   - **Answer**: Create private API with VPC endpoint, enable Private DNS, configure resource policy

2. **Scenario**: Public API needs to access private backend services
   - **Answer**: Use VPC Link with Network Load Balancer in private subnets

3. **Scenario**: Multiple accounts need access to private API
   - **Answer**: Resource policy allowing specific accounts + VPC endpoints in each account + IAM permissions

4. **Scenario**: On-premises application needs to call private API
   - **Answer**: Direct Connect/VPN → Transit Gateway → VPC with API Gateway endpoint → Private API + Route 53 Resolver for DNS

5. **Scenario**: Minimize cost for high-volume API with private integration
   - **Answer**: HTTP API (cheaper) with VPC Link, enable caching, use VPC endpoint to avoid NAT Gateway charges

### Key Concepts to Remember

- **Private APIs**: Only accessible via VPC endpoints, never from internet
- **VPC Link**: Uses PrivateLink for private integration between API Gateway and NLB
- **Resource Policy**: Required for private APIs to specify which VPCs/endpoints can access
- **Private DNS**: Must be enabled on VPC endpoint for easy API access (otherwise need to use endpoint-specific URL)
- **Defense in Depth**: Combine network controls (security groups, NACLs) + resource policies + IAM + Lambda authorizers
- **REST API VPC Link**: Uses single NLB, created via API Gateway console
- **HTTP API VPC Link**: Uses ENIs in subnets, created separately, supports multiple integrations
- **Cross-Account Access**: Requires resource policy in API account + VPC endpoint in consumer account + IAM permissions
- **Monitoring**: Enable execution logging, access logging, CloudWatch metrics, VPC Flow Logs

### Gotchas and Common Mistakes

- **Forgetting to enable Private DNS**: API calls fail with DNS resolution errors
- **Wrong VPC endpoint ID in resource policy**: All API calls return 403 Forbidden
- **Security group doesn't allow port 443**: Connection timeouts to VPC endpoint
- **NLB in public subnets instead of private**: Defeats purpose of private integration
- **Not testing from correct source**: Must test from within VPC for private APIs
- **Confusing REST API VPC Link vs HTTP API VPC Link**: Different creation processes and capabilities
- **Assuming private API accessible from internet**: It never is, by design
- **Not considering cross-AZ data transfer costs**: Place VPC endpoint ENIs in same AZs as application
- **Forgetting Lambda warm-up for private integrations**: Cold starts add latency
- **Not monitoring VPC Link status**: If VPC Link unavailable, all integrations fail

## Additional Resources

### AWS Documentation
- [API Gateway Private APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-private-apis.html)
- [VPC Links for REST APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/set-up-private-integration.html)
- [VPC Links for HTTP APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/http-api-vpc-links.html)
- [API Gateway Resource Policies](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-resource-policies.html)
- [VPC Endpoints for API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-vpc-endpoint.html)

### Related Topics
- Topic 28: Configuring VPC Interface Endpoints - PrivateLink Explained
- Topic 29: Exposing Services via AWS PrivateLink
- Topic 51: Securing APIs with AWS Networking Tools
- Topic 139: Designing Secure VPC Endpoints with PrivateLink
- Topic 154: Preventing Data Exfiltration Using VPC Endpoint Policies

### Hands-On Practice
- [AWS Workshop: API Gateway with VPC Integration](https://catalog.workshops.aws/api-gateway)
- [AWS Samples: Private API Gateway Examples](https://github.com/aws-samples/api-gateway-vpc-link)

## Cleanup Steps

To avoid unnecessary charges, clean up resources created in labs:

```bash
# Delete API Gateway APIs
aws apigateway delete-rest-api --rest-api-id $API_ID
aws apigateway delete-rest-api --rest-api-id $PUBLIC_API_ID

# Delete VPC Link (REST API)
aws apigateway delete-vpc-link --vpc-link-id $VPCLINK_ID

# Delete VPC Link (HTTP API)
aws apigatewayv2 delete-vpc-link --vpc-link-id $VPCLINK_V2_ID

# Delete VPC Endpoint
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids $VPCE_ID

# Delete NLB
aws elbv2 delete-load-balancer --load-balancer-arn $NLB_ARN

# Delete Target Group (wait 5 minutes after deleting NLB)
sleep 300
aws elbv2 delete-target-group --target-group-arn $TG_ARN

# Delete Lambda Function
aws lambda delete-function --function-name PrivateAPIFunction

# Delete Security Groups
aws ec2 delete-security-group --group-id $SG_ID
aws ec2 delete-security-group --group-id $ENDPOINT_SG_ID

# Delete CloudWatch Log Groups
aws logs delete-log-group --log-group-name /aws/apigateway/${API_ID}
aws logs delete-log-group --log-group-name /aws/apigateway/${PUBLIC_API_ID}

# Verify cleanup
echo "Cleanup complete. Verify no remaining charges in AWS Billing Console."
```

## Summary

Building secure API Gateway architectures with private VPC access provides strong network isolation while maintaining flexibility for various use cases. Key patterns include:

1. **Private APIs with VPC Endpoints**: Complete isolation for internal-only APIs
2. **Public APIs with VPC Link**: Public access with private backend security
3. **Cross-Account Private Access**: Centralized API management across organization
4. **Hybrid Connectivity**: On-premises access to private AWS APIs

Success requires careful configuration of:
- VPC endpoints with proper security groups and endpoint policies
- API Gateway resource policies with least-privilege access
- VPC Links with healthy Network Load Balancers
- Monitoring and logging for visibility and troubleshooting

By combining network-level controls (VPC endpoints, security groups) with application-level security (resource policies, IAM, authorizers), you create defense-in-depth architectures that meet stringent security and compliance requirements.