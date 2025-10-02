# Lab: Public API Gateway with Authentication

## Overview

This hands-on lab demonstrates how to build a secure public-facing API using Amazon API Gateway that is accessible from the internet with proper authentication mechanisms. You'll learn to implement multiple authentication strategies including API keys, IAM authorization, Lambda authorizers, and Amazon Cognito user pools.

## Learning Objectives

By completing this lab, you will:
- Create a public REST API Gateway accessible from the internet
- Implement API key-based authentication with usage plans
- Configure IAM authorization for programmatic access
- Build a custom Lambda authorizer with JWT tokens
- Integrate Amazon Cognito User Pools for user authentication
- Apply security best practices for public APIs
- Monitor and troubleshoot authenticated API requests

## Prerequisites

- AWS account with appropriate permissions
- Basic understanding of API Gateway concepts
- Familiarity with Lambda functions
- AWS CLI configured with credentials
- curl or Postman for testing

## Lab Architecture

```
┌─────────────────────────────────────────────────────┐
│                  Public Internet                     │
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────────────┐  │
│  │  Mobile  │  │  Web App │  │  External Client │  │
│  │   App    │  │          │  │   (API Key)      │  │
│  └─────┬────┘  └────┬─────┘  └────────┬─────────┘  │
└────────┼────────────┼─────────────────┼─────────────┘
         │            │                 │
         │ Cognito    │ IAM SigV4      │ API Key
         │ Token      │ Auth           │
         │            │                 │
    ┌────▼────────────▼─────────────────▼──────┐
    │       Amazon API Gateway (Regional)       │
    │                                           │
    │  ┌─────────────────────────────────────┐ │
    │  │  Authentication Methods:            │ │
    │  │  • API Keys + Usage Plans           │ │
    │  │  • IAM Authorization                │ │
    │  │  • Lambda Authorizer (Custom JWT)   │ │
    │  │  • Cognito User Pool Authorizer     │ │
    │  └─────────────────────────────────────┘ │
    │                                           │
    │  ┌─────────────────────────────────────┐ │
    │  │  Rate Limiting & Throttling         │ │
    │  │  • Per-method throttle              │ │
    │  │  • Usage plan quotas                │ │
    │  └─────────────────────────────────────┘ │
    └───────────────────┬───────────────────────┘
                        │
           ┌────────────▼────────────┐
           │   Backend Integration   │
           │  • Lambda Functions     │
           │  • HTTP Endpoints       │
           └─────────────────────────┘
```

## Lab 1: API Key Authentication with Usage Plans

### Step 1: Create the API Gateway

**Console Steps:**
1. Open **API Gateway** console
2. Click **Create API** → **REST API** → **Build**
3. Choose **New API**
4. API name: `SecurePublicAPI`
5. Endpoint Type: **Regional**
6. Click **Create API**

**CLI Commands:**
```bash
# Create the API
aws apigateway create-rest-api \
  --name SecurePublicAPI \
  --description "Public API with multiple authentication methods" \
  --endpoint-configuration types=REGIONAL \
  --region us-east-1

# Save the API ID
export API_ID=$(aws apigateway get-rest-apis \
  --query "items[?name=='SecurePublicAPI'].id" \
  --output text)

echo "API ID: $API_ID"
```

### Step 2: Create API Resources and Methods

**CLI Commands:**
```bash
# Get root resource ID
export ROOT_ID=$(aws apigateway get-resources \
  --rest-api-id $API_ID \
  --query 'items[0].id' \
  --output text)

# Create /products resource
aws apigateway create-resource \
  --rest-api-id $API_ID \
  --parent-id $ROOT_ID \
  --path-part products

export PRODUCTS_RESOURCE_ID=$(aws apigateway get-resources \
  --rest-api-id $API_ID \
  --query 'items[?path==`/products`].id' \
  --output text)

# Create GET method with API Key requirement
aws apigateway put-method \
  --rest-api-id $API_ID \
  --resource-id $PRODUCTS_RESOURCE_ID \
  --http-method GET \
  --authorization-type NONE \
  --api-key-required

# Create POST method with API Key requirement
aws apigateway put-method \
  --rest-api-id $API_ID \
  --resource-id $PRODUCTS_RESOURCE_ID \
  --http-method POST \
  --authorization-type NONE \
  --api-key-required
```

### Step 3: Create Lambda Backend Function

**Create Lambda execution role:**
```bash
# Create IAM role for Lambda
aws iam create-role \
  --role-name ApiGatewayLambdaRole \
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
  --role-name ApiGatewayLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Wait for role to propagate
sleep 10
```

**Create Lambda function:**
```bash
# Create function code
cat > products-handler.js <<'EOF'
exports.handler = async (event) => {
    console.log('Received event:', JSON.stringify(event, null, 2));

    const method = event.httpMethod;
    const identity = event.requestContext.identity;

    // Sample product data
    const products = [
        { id: 1, name: "Product A", price: 29.99 },
        { id: 2, name: "Product B", price: 49.99 },
        { id: 3, name: "Product C", price: 19.99 }
    ];

    if (method === 'GET') {
        return {
            statusCode: 200,
            headers: {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            body: JSON.stringify({
                message: 'Successfully retrieved products',
                products: products,
                authenticatedBy: identity.apiKey ? 'API Key' : identity.cognitoIdentityId ? 'Cognito' : 'IAM',
                timestamp: new Date().toISOString()
            })
        };
    } else if (method === 'POST') {
        const body = JSON.parse(event.body || '{}');
        return {
            statusCode: 201,
            headers: {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            body: JSON.stringify({
                message: 'Product created successfully',
                product: body,
                timestamp: new Date().toISOString()
            })
        };
    }

    return {
        statusCode: 405,
        body: JSON.stringify({ message: 'Method not allowed' })
    };
};
EOF

# Package function
zip products-handler.zip products-handler.js

# Get account ID
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create Lambda function
aws lambda create-function \
  --function-name ProductsHandler \
  --runtime nodejs20.x \
  --role arn:aws:iam::${ACCOUNT_ID}:role/ApiGatewayLambdaRole \
  --handler products-handler.handler \
  --zip-file fileb://products-handler.zip \
  --timeout 10 \
  --memory-size 256

export LAMBDA_ARN=$(aws lambda get-function \
  --function-name ProductsHandler \
  --query 'Configuration.FunctionArn' \
  --output text)

echo "Lambda ARN: $LAMBDA_ARN"
```

### Step 4: Configure Lambda Integration

**CLI Commands:**
```bash
# Create GET method integration
aws apigateway put-integration \
  --rest-api-id $API_ID \
  --resource-id $PRODUCTS_RESOURCE_ID \
  --http-method GET \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/${LAMBDA_ARN}/invocations

# Create POST method integration
aws apigateway put-integration \
  --rest-api-id $API_ID \
  --resource-id $PRODUCTS_RESOURCE_ID \
  --http-method POST \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/${LAMBDA_ARN}/invocations

# Grant API Gateway permission to invoke Lambda
aws lambda add-permission \
  --function-name ProductsHandler \
  --statement-id apigateway-products-get \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:${ACCOUNT_ID}:${API_ID}/*/*/products"
```

### Step 5: Create API Keys and Usage Plans

**Create API Key:**
```bash
# Create API key for external client
aws apigateway create-api-key \
  --name ExternalClientKey \
  --description "API key for external client access" \
  --enabled

export API_KEY_ID=$(aws apigateway get-api-keys \
  --query "items[?name=='ExternalClientKey'].id" \
  --output text)

# Get the actual API key value
export API_KEY_VALUE=$(aws apigateway get-api-key \
  --api-key $API_KEY_ID \
  --include-value \
  --query 'value' \
  --output text)

echo "API Key: $API_KEY_VALUE"
# SAVE this value - you'll need it for testing!
```

**Create Usage Plan:**
```bash
# Create usage plan with throttle and quota limits
aws apigateway create-usage-plan \
  --name StandardPlan \
  --description "Standard usage plan - 1000 req/day" \
  --throttle rateLimit=100,burstLimit=200 \
  --quota limit=1000,period=DAY

export USAGE_PLAN_ID=$(aws apigateway get-usage-plans \
  --query "items[?name=='StandardPlan'].id" \
  --output text)

echo "Usage Plan ID: $USAGE_PLAN_ID"
```

**Deploy API:**
```bash
# Create deployment
aws apigateway create-deployment \
  --rest-api-id $API_ID \
  --stage-name prod \
  --description "Production deployment with API key auth"

# Associate usage plan with API stage
aws apigateway update-usage-plan \
  --usage-plan-id $USAGE_PLAN_ID \
  --patch-operations \
    op=add,path=/apiStages,value="${API_ID}:prod"

# Associate API key with usage plan
aws apigateway create-usage-plan-key \
  --usage-plan-id $USAGE_PLAN_ID \
  --key-id $API_KEY_ID \
  --key-type API_KEY

# Get invoke URL
export INVOKE_URL="https://${API_ID}.execute-api.us-east-1.amazonaws.com/prod"
echo "Invoke URL: $INVOKE_URL"
```

### Step 6: Test API Key Authentication

**Test without API key (should fail):**
```bash
# This should return 403 Forbidden
curl -X GET ${INVOKE_URL}/products

# Expected response:
# {"message":"Forbidden"}
```

**Test with API key (should succeed):**
```bash
# This should return 200 OK with products
curl -X GET ${INVOKE_URL}/products \
  -H "x-api-key: ${API_KEY_VALUE}"

# Expected response:
# {
#   "message": "Successfully retrieved products",
#   "products": [...],
#   "authenticatedBy": "API Key",
#   "timestamp": "2025-01-15T10:30:00Z"
# }

# Test POST method
curl -X POST ${INVOKE_URL}/products \
  -H "x-api-key: ${API_KEY_VALUE}" \
  -H "Content-Type: application/json" \
  -d '{"name":"Product D","price":39.99}'
```

**Monitor usage:**
```bash
# Check current usage
aws apigateway get-usage \
  --usage-plan-id $USAGE_PLAN_ID \
  --start-date $(date -u -d '1 day ago' +%Y-%m-%d) \
  --end-date $(date -u +%Y-%m-%d)
```

## Lab 2: IAM Authorization for Programmatic Access

### Step 1: Create IAM-Authenticated Endpoint

**Create new resource:**
```bash
# Create /admin resource for IAM-authenticated access
aws apigateway create-resource \
  --rest-api-id $API_ID \
  --parent-id $ROOT_ID \
  --path-part admin

export ADMIN_RESOURCE_ID=$(aws apigateway get-resources \
  --rest-api-id $API_ID \
  --query 'items[?path==`/admin`].id' \
  --output text)

# Create GET method with IAM authorization
aws apigateway put-method \
  --rest-api-id $API_ID \
  --resource-id $ADMIN_RESOURCE_ID \
  --http-method GET \
  --authorization-type AWS_IAM \
  --api-key-required false

# Create Lambda integration
aws apigateway put-integration \
  --rest-api-id $API_ID \
  --resource-id $ADMIN_RESOURCE_ID \
  --http-method GET \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/${LAMBDA_ARN}/invocations

# Grant permission
aws lambda add-permission \
  --function-name ProductsHandler \
  --statement-id apigateway-admin-get \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:${ACCOUNT_ID}:${API_ID}/*/*/admin"

# Redeploy API
aws apigateway create-deployment \
  --rest-api-id $API_ID \
  --stage-name prod \
  --description "Added IAM auth endpoint"
```

### Step 2: Create IAM Role for API Access

**Create IAM role and policy:**
```bash
# Create IAM policy for API access
cat > api-invoke-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "execute-api:Invoke",
      "Resource": "arn:aws:execute-api:us-east-1:${ACCOUNT_ID}:${API_ID}/prod/GET/admin"
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name ApiGatewayInvokePolicy \
  --policy-document file://api-invoke-policy.json

export POLICY_ARN=$(aws iam list-policies \
  --query "Policies[?PolicyName=='ApiGatewayInvokePolicy'].Arn" \
  --output text)

# Create IAM user for testing
aws iam create-user --user-name api-test-user

# Attach policy to user
aws iam attach-user-policy \
  --user-name api-test-user \
  --policy-arn $POLICY_ARN

# Create access keys
aws iam create-access-key --user-name api-test-user > api-user-credentials.json

export TEST_ACCESS_KEY=$(cat api-user-credentials.json | jq -r '.AccessKey.AccessKeyId')
export TEST_SECRET_KEY=$(cat api-user-credentials.json | jq -r '.AccessKey.SecretAccessKey')

echo "Test Access Key: $TEST_ACCESS_KEY"
echo "Test Secret Key: $TEST_SECRET_KEY"
```

### Step 3: Test IAM Authentication

**Test with AWS CLI (SigV4 signing):**
```bash
# Configure temporary profile for test user
aws configure set aws_access_key_id $TEST_ACCESS_KEY --profile api-test
aws configure set aws_secret_access_key $TEST_SECRET_KEY --profile api-test
aws configure set region us-east-1 --profile api-test

# Test with IAM authentication
aws apigatewaymanagementapi get-connection \
  --endpoint-url ${INVOKE_URL}/admin \
  --profile api-test 2>/dev/null || \
  curl --request GET ${INVOKE_URL}/admin \
    --user "${TEST_ACCESS_KEY}:${TEST_SECRET_KEY}" \
    --aws-sigv4 "aws:amz:us-east-1:execute-api"
```

**Test with Python boto3:**
```python
# Create test script
cat > test_iam_auth.py <<'EOF'
import boto3
import requests
from requests_aws4auth import AWS4Auth

# Configuration
region = 'us-east-1'
service = 'execute-api'
access_key = 'YOUR_ACCESS_KEY'  # Replace
secret_key = 'YOUR_SECRET_KEY'  # Replace
url = 'YOUR_API_URL/admin'      # Replace

# Create AWS SigV4 auth
auth = AWS4Auth(access_key, secret_key, region, service)

# Make request
response = requests.get(url, auth=auth)
print(f"Status: {response.status_code}")
print(f"Response: {response.json()}")
EOF

# Install dependencies and run (if Python available)
# pip install requests requests-aws4auth
# python test_iam_auth.py
```

**Test without IAM credentials (should fail):**
```bash
# This should return 403 Forbidden
curl -X GET ${INVOKE_URL}/admin

# Expected response:
# {"message":"Missing Authentication Token"}
```

## Lab 3: Lambda Authorizer with Custom JWT Tokens

### Step 1: Create Lambda Authorizer Function

**Create authorizer Lambda:**
```bash
# Create authorizer function code
cat > jwt-authorizer.js <<'EOF'
const jwt = require('jsonwebtoken');

const SECRET_KEY = process.env.JWT_SECRET || 'your-secret-key-change-in-production';

exports.handler = async (event) => {
    console.log('Authorizer event:', JSON.stringify(event, null, 2));

    // Get token from Authorization header
    const token = event.authorizationToken?.replace('Bearer ', '');

    if (!token) {
        throw new Error('Unauthorized');
    }

    try {
        // Verify JWT token
        const decoded = jwt.verify(token, SECRET_KEY);
        console.log('Decoded token:', decoded);

        // Generate IAM policy
        const policy = generatePolicy(decoded.sub, 'Allow', event.methodArn, decoded);
        console.log('Generated policy:', JSON.stringify(policy, null, 2));

        return policy;
    } catch (err) {
        console.error('Token verification failed:', err.message);
        throw new Error('Unauthorized');
    }
};

function generatePolicy(principalId, effect, resource, context) {
    const authResponse = {
        principalId: principalId
    };

    if (effect && resource) {
        authResponse.policyDocument = {
            Version: '2012-10-17',
            Statement: [{
                Action: 'execute-api:Invoke',
                Effect: effect,
                Resource: resource
            }]
        };
    }

    // Add context data to pass to backend
    authResponse.context = {
        userId: context.sub,
        email: context.email || '',
        role: context.role || 'user'
    };

    return authResponse;
}
EOF

# Create package.json
cat > package.json <<'EOF'
{
  "name": "jwt-authorizer",
  "version": "1.0.0",
  "dependencies": {
    "jsonwebtoken": "^9.0.2"
  }
}
EOF

# Install dependencies and package
npm install
zip -r jwt-authorizer.zip jwt-authorizer.js package.json node_modules/

# Create Lambda function
aws lambda create-function \
  --function-name JwtAuthorizer \
  --runtime nodejs20.x \
  --role arn:aws:iam::${ACCOUNT_ID}:role/ApiGatewayLambdaRole \
  --handler jwt-authorizer.handler \
  --zip-file fileb://jwt-authorizer.zip \
  --timeout 10 \
  --environment Variables={JWT_SECRET=my-super-secret-key-2025}

export AUTHORIZER_LAMBDA_ARN=$(aws lambda get-function \
  --function-name JwtAuthorizer \
  --query 'Configuration.FunctionArn' \
  --output text)

echo "Authorizer Lambda ARN: $AUTHORIZER_LAMBDA_ARN"
```

### Step 2: Create API Gateway Authorizer

**Configure authorizer in API Gateway:**
```bash
# Create Lambda authorizer
aws apigateway create-authorizer \
  --rest-api-id $API_ID \
  --name JwtTokenAuthorizer \
  --type TOKEN \
  --authorizer-uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/${AUTHORIZER_LAMBDA_ARN}/invocations \
  --identity-source method.request.header.Authorization \
  --authorizer-result-ttl-in-seconds 300

export AUTHORIZER_ID=$(aws apigateway get-authorizers \
  --rest-api-id $API_ID \
  --query "items[?name=='JwtTokenAuthorizer'].id" \
  --output text)

# Grant API Gateway permission to invoke authorizer
aws lambda add-permission \
  --function-name JwtAuthorizer \
  --statement-id apigateway-authorizer-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:${ACCOUNT_ID}:${API_ID}/authorizers/${AUTHORIZER_ID}"
```

### Step 3: Create JWT-Protected Endpoint

**Create resource with Lambda authorizer:**
```bash
# Create /secure resource
aws apigateway create-resource \
  --rest-api-id $API_ID \
  --parent-id $ROOT_ID \
  --path-part secure

export SECURE_RESOURCE_ID=$(aws apigateway get-resources \
  --rest-api-id $API_ID \
  --query 'items[?path==`/secure`].id' \
  --output text)

# Create GET method with custom authorizer
aws apigateway put-method \
  --rest-api-id $API_ID \
  --resource-id $SECURE_RESOURCE_ID \
  --http-method GET \
  --authorization-type CUSTOM \
  --authorizer-id $AUTHORIZER_ID

# Create Lambda integration
aws apigateway put-integration \
  --rest-api-id $API_ID \
  --resource-id $SECURE_RESOURCE_ID \
  --http-method GET \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/${LAMBDA_ARN}/invocations

# Grant permission
aws lambda add-permission \
  --function-name ProductsHandler \
  --statement-id apigateway-secure-get \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:${ACCOUNT_ID}:${API_ID}/*/*/secure"

# Deploy
aws apigateway create-deployment \
  --rest-api-id $API_ID \
  --stage-name prod \
  --description "Added JWT authorizer endpoint"
```

### Step 4: Generate and Test JWT Token

**Generate test JWT token:**
```bash
# Using Node.js to generate token
node <<'EOF'
const jwt = require('jsonwebtoken');

const SECRET_KEY = 'my-super-secret-key-2025';

const payload = {
  sub: 'user123',
  email: 'user@example.com',
  role: 'admin',
  iat: Math.floor(Date.now() / 1000),
  exp: Math.floor(Date.now() / 1000) + (60 * 60) // 1 hour
};

const token = jwt.sign(payload, SECRET_KEY);
console.log('JWT Token:');
console.log(token);
EOF

# Save the token
export JWT_TOKEN="<paste-token-from-above>"
```

**Test with JWT token:**
```bash
# Test with valid JWT (should succeed)
curl -X GET ${INVOKE_URL}/secure \
  -H "Authorization: Bearer ${JWT_TOKEN}"

# Expected response:
# {
#   "message": "Successfully retrieved products",
#   "products": [...],
#   "authenticatedBy": "Custom Authorizer",
#   "timestamp": "..."
# }

# Test without token (should fail)
curl -X GET ${INVOKE_URL}/secure

# Expected response:
# {"message":"Unauthorized"}

# Test with invalid token (should fail)
curl -X GET ${INVOKE_URL}/secure \
  -H "Authorization: Bearer invalid.token.here"
```

## Lab 4: Amazon Cognito User Pool Authentication

### Step 1: Create Cognito User Pool

**Create user pool:**
```bash
# Create Cognito User Pool
aws cognito-idp create-user-pool \
  --pool-name SecureAPIUserPool \
  --policies '{
    "PasswordPolicy": {
      "MinimumLength": 8,
      "RequireUppercase": true,
      "RequireLowercase": true,
      "RequireNumbers": true,
      "RequireSymbols": false
    }
  }' \
  --auto-verified-attributes email \
  --username-attributes email

export USER_POOL_ID=$(aws cognito-idp list-user-pools \
  --max-results 60 \
  --query "UserPools[?Name=='SecureAPIUserPool'].Id" \
  --output text)

echo "User Pool ID: $USER_POOL_ID"

# Create user pool client
aws cognito-idp create-user-pool-client \
  --user-pool-id $USER_POOL_ID \
  --client-name SecureAPIClient \
  --generate-secret \
  --explicit-auth-flows ALLOW_USER_PASSWORD_AUTH ALLOW_REFRESH_TOKEN_AUTH

export CLIENT_ID=$(aws cognito-idp list-user-pool-clients \
  --user-pool-id $USER_POOL_ID \
  --query "UserPoolClients[?ClientName=='SecureAPIClient'].ClientId" \
  --output text)

export CLIENT_SECRET=$(aws cognito-idp describe-user-pool-client \
  --user-pool-id $USER_POOL_ID \
  --client-id $CLIENT_ID \
  --query 'UserPoolClient.ClientSecret' \
  --output text)

echo "Client ID: $CLIENT_ID"
echo "Client Secret: $CLIENT_SECRET"
```

### Step 2: Create Test User

**Create and confirm user:**
```bash
# Create user
aws cognito-idp admin-create-user \
  --user-pool-id $USER_POOL_ID \
  --username testuser@example.com \
  --user-attributes Name=email,Value=testuser@example.com \
  --temporary-password "TempPass123!" \
  --message-action SUPPRESS

# Set permanent password
aws cognito-idp admin-set-user-password \
  --user-pool-id $USER_POOL_ID \
  --username testuser@example.com \
  --password "SecurePass123!" \
  --permanent
```

### Step 3: Configure Cognito Authorizer in API Gateway

**Create Cognito authorizer:**
```bash
# Create Cognito User Pool authorizer
aws apigateway create-authorizer \
  --rest-api-id $API_ID \
  --name CognitoUserPoolAuthorizer \
  --type COGNITO_USER_POOLS \
  --provider-arns arn:aws:cognito-idp:us-east-1:${ACCOUNT_ID}:userpool/${USER_POOL_ID} \
  --identity-source method.request.header.Authorization

export COGNITO_AUTHORIZER_ID=$(aws apigateway get-authorizers \
  --rest-api-id $API_ID \
  --query "items[?name=='CognitoUserPoolAuthorizer'].id" \
  --output text)

echo "Cognito Authorizer ID: $COGNITO_AUTHORIZER_ID"
```

### Step 4: Create Cognito-Protected Endpoint

**Create user-specific endpoint:**
```bash
# Create /users resource
aws apigateway create-resource \
  --rest-api-id $API_ID \
  --parent-id $ROOT_ID \
  --path-part users

export USERS_RESOURCE_ID=$(aws apigateway get-resources \
  --rest-api-id $API_ID \
  --query 'items[?path==`/users`].id' \
  --output text)

# Create GET method with Cognito authorizer
aws apigateway put-method \
  --rest-api-id $API_ID \
  --resource-id $USERS_RESOURCE_ID \
  --http-method GET \
  --authorization-type COGNITO_USER_POOLS \
  --authorizer-id $COGNITO_AUTHORIZER_ID

# Create Lambda integration
aws apigateway put-integration \
  --rest-api-id $API_ID \
  --resource-id $USERS_RESOURCE_ID \
  --http-method GET \
  --type AWS_PROXY \
  --integration-http-method POST \
  --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/${LAMBDA_ARN}/invocations

# Grant permission
aws lambda add-permission \
  --function-name ProductsHandler \
  --statement-id apigateway-users-get \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:us-east-1:${ACCOUNT_ID}:${API_ID}/*/*/users"

# Deploy
aws apigateway create-deployment \
  --rest-api-id $API_ID \
  --stage-name prod \
  --description "Added Cognito auth endpoint"
```

### Step 5: Test Cognito Authentication

**Authenticate and get ID token:**
```bash
# Create authentication helper script
cat > cognito-auth.sh <<EOF
#!/bin/bash

USER_POOL_ID="$USER_POOL_ID"
CLIENT_ID="$CLIENT_ID"
USERNAME="testuser@example.com"
PASSWORD="SecurePass123!"

# Authenticate user
AUTH_RESPONSE=\$(aws cognito-idp admin-initiate-auth \
  --user-pool-id \$USER_POOL_ID \
  --client-id \$CLIENT_ID \
  --auth-flow ADMIN_NO_SRP_AUTH \
  --auth-parameters USERNAME=\$USERNAME,PASSWORD=\$PASSWORD)

# Extract ID token
ID_TOKEN=\$(echo \$AUTH_RESPONSE | jq -r '.AuthenticationResult.IdToken')
echo "ID Token: \$ID_TOKEN"
echo ""

# Test API call
curl -X GET ${INVOKE_URL}/users \
  -H "Authorization: \$ID_TOKEN"
EOF

chmod +x cognito-auth.sh
./cognito-auth.sh
```

**Test without token (should fail):**
```bash
curl -X GET ${INVOKE_URL}/users

# Expected response:
# {"message":"Unauthorized"}
```

## Lab 5: Monitoring and Security Best Practices

### Step 1: Enable API Gateway Logging

**Configure CloudWatch logging:**
```bash
# Create CloudWatch Logs role (if not exists)
ROLE_EXISTS=$(aws iam get-role --role-name APIGatewayCloudWatchRole 2>/dev/null)

if [ -z "$ROLE_EXISTS" ]; then
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
fi

# Set account-level logging role
aws apigateway update-account \
  --patch-operations op=replace,path=/cloudwatchRoleArn,value=arn:aws:iam::${ACCOUNT_ID}:role/APIGatewayCloudWatchRole

# Create log group
aws logs create-log-group --log-group-name /aws/apigateway/${API_ID}

# Enable execution logging
aws apigateway update-stage \
  --rest-api-id $API_ID \
  --stage-name prod \
  --patch-operations \
    op=replace,path=/*/*/logging/loglevel,value=INFO \
    op=replace,path=/*/*/logging/dataTrace,value=true \
    op=replace,path=/*/*/metrics/enabled,value=true

# Enable access logging
aws apigateway update-stage \
  --rest-api-id $API_ID \
  --stage-name prod \
  --patch-operations \
    op=replace,path=/accessLogSettings/destinationArn,value=arn:aws:logs:us-east-1:${ACCOUNT_ID}:log-group:/aws/apigateway/${API_ID} \
    op=replace,path=/accessLogSettings/format,value='$context.requestId $context.extendedRequestId $context.identity.sourceIp $context.authorizer.claims.sub $context.status $context.error.message'
```

### Step 2: Configure Rate Limiting and Throttling

**Set throttle limits:**
```bash
# Stage-level throttling
aws apigateway update-stage \
  --rest-api-id $API_ID \
  --stage-name prod \
  --patch-operations \
    op=replace,path=/throttle/rateLimit,value=500 \
    op=replace,path=/throttle/burstLimit,value=1000

# Method-level throttling for sensitive endpoints
aws apigateway update-method \
  --rest-api-id $API_ID \
  --resource-id $ADMIN_RESOURCE_ID \
  --http-method GET \
  --patch-operations \
    op=replace,path=/throttle/rateLimit,value=50 \
    op=replace,path=/throttle/burstLimit,value=100
```

### Step 3: Set Up CloudWatch Alarms

**Create monitoring alarms:**
```bash
# Alarm for 4XX errors
aws cloudwatch put-metric-alarm \
  --alarm-name "${API_ID}-high-4xx-errors" \
  --alarm-description "Alert when 4XX errors exceed threshold" \
  --metric-name 4XXError \
  --namespace AWS/ApiGateway \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 100 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=ApiName,Value=$API_ID Name=Stage,Value=prod

# Alarm for 5XX errors
aws cloudwatch put-metric-alarm \
  --alarm-name "${API_ID}-high-5xx-errors" \
  --alarm-description "Alert when 5XX errors exceed threshold" \
  --metric-name 5XXError \
  --namespace AWS/ApiGateway \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=ApiName,Value=$API_ID Name=Stage,Value=prod

# Alarm for high latency
aws cloudwatch put-metric-alarm \
  --alarm-name "${API_ID}-high-latency" \
  --alarm-description "Alert when latency is high" \
  --metric-name Latency \
  --namespace AWS/ApiGateway \
  --statistic Average \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 1000 \
  --comparison-operator GreaterThanThreshold \
  --dimensions Name=ApiName,Value=$API_ID Name=Stage,Value=prod
```

### Step 4: View Logs and Metrics

**Query CloudWatch Logs:**
```bash
# View recent logs
aws logs tail /aws/apigateway/${API_ID} --follow

# Filter for errors
aws logs filter-log-events \
  --log-group-name /aws/apigateway/${API_ID} \
  --filter-pattern "ERROR"

# Filter for unauthorized access attempts
aws logs filter-log-events \
  --log-group-name /aws/apigateway/${API_ID} \
  --filter-pattern "Unauthorized"
```

**View metrics:**
```bash
# Get API request count
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApiGateway \
  --metric-name Count \
  --dimensions Name=ApiName,Value=$API_ID \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum

# Get error rates
aws cloudwatch get-metric-statistics \
  --namespace AWS/ApiGateway \
  --metric-name 4XXError \
  --dimensions Name=ApiName,Value=$API_ID Name=Stage,Value=prod \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Sum
```

## Security Best Practices Summary

### 1. Authentication Strategy Selection

| Method | Use Case | Security Level | Complexity |
|--------|----------|----------------|------------|
| API Keys | External partners, simple auth | Low | Low |
| IAM | AWS services, internal apps | High | Medium |
| Lambda Authorizer | Custom logic, existing auth | High | High |
| Cognito | User authentication, mobile/web apps | High | Medium |

### 2. Rate Limiting Strategy

**Implement multiple layers:**
```bash
# 1. API-level limits (default)
- Rate: 10,000 req/sec
- Burst: 5,000 req

# 2. Stage-level limits
- Rate: 500-1000 req/sec
- Burst: 1000-2000 req

# 3. Method-level limits
- Critical endpoints: 50-100 req/sec
- Public endpoints: 100-200 req/sec

# 4. Usage plan limits
- Daily quota: 1000-100000 req/day
- Per-key rate limits
```

### 3. Security Checklist

- ✅ **Enable HTTPS only** - Reject HTTP requests
- ✅ **Implement authentication** - Never allow anonymous access to sensitive data
- ✅ **Use authorization** - Check permissions at method level
- ✅ **Enable logging** - Track all access and errors
- ✅ **Set throttle limits** - Prevent DDoS and abuse
- ✅ **Use WAF** - Protect against common web exploits (see Lab 6)
- ✅ **Validate input** - Use request validators
- ✅ **Secure secrets** - Use AWS Secrets Manager or Parameter Store
- ✅ **Rotate credentials** - Regular API key and token rotation
- ✅ **Monitor metrics** - Set up CloudWatch alarms
- ✅ **Enable CORS properly** - Restrict origins in production
- ✅ **Use resource policies** - Additional layer of access control

### 4. Request Validation

**Enable request validation:**
```bash
# Create request validator
aws apigateway create-request-validator \
  --rest-api-id $API_ID \
  --name product-validator \
  --validate-request-body \
  --validate-request-parameters

# Create request model
aws apigateway create-model \
  --rest-api-id $API_ID \
  --name ProductModel \
  --content-type application/json \
  --schema '{
    "$schema": "http://json-schema.org/draft-04/schema#",
    "type": "object",
    "properties": {
      "name": {"type": "string", "minLength": 1},
      "price": {"type": "number", "minimum": 0}
    },
    "required": ["name", "price"]
  }'
```

## Lab 6: Adding WAF Protection (Optional)

### Step 1: Create WAF Web ACL

```bash
# Create WAF Web ACL
aws wafv2 create-web-acl \
  --name SecureAPIWAF \
  --scope REGIONAL \
  --region us-east-1 \
  --default-action Allow={} \
  --rules file://waf-rules.json \
  --visibility-config SampledRequestsEnabled=true,CloudWatchMetricsEnabled=true,MetricName=SecureAPIWAF

# waf-rules.json
cat > waf-rules.json <<'EOF'
[
  {
    "Name": "RateLimitRule",
    "Priority": 1,
    "Statement": {
      "RateBasedStatement": {
        "Limit": 2000,
        "AggregateKeyType": "IP"
      }
    },
    "Action": {
      "Block": {}
    },
    "VisibilityConfig": {
      "SampledRequestsEnabled": true,
      "CloudWatchMetricsEnabled": true,
      "MetricName": "RateLimitRule"
    }
  },
  {
    "Name": "AWSManagedRulesCommonRuleSet",
    "Priority": 2,
    "Statement": {
      "ManagedRuleGroupStatement": {
        "VendorName": "AWS",
        "Name": "AWSManagedRulesCommonRuleSet"
      }
    },
    "OverrideAction": {
      "None": {}
    },
    "VisibilityConfig": {
      "SampledRequestsEnabled": true,
      "CloudWatchMetricsEnabled": true,
      "MetricName": "CommonRuleSet"
    }
  }
]
EOF

# Associate WAF with API Gateway
export WAF_ARN=$(aws wafv2 list-web-acls \
  --scope REGIONAL \
  --region us-east-1 \
  --query "WebACLs[?Name=='SecureAPIWAF'].ARN" \
  --output text)

aws wafv2 associate-web-acl \
  --web-acl-arn $WAF_ARN \
  --resource-arn arn:aws:apigateway:us-east-1::/restapis/${API_ID}/stages/prod
```

## Testing Summary

**Quick test script for all authentication methods:**
```bash
cat > test-all-auth.sh <<EOF
#!/bin/bash

API_URL="${INVOKE_URL}"

echo "=== Testing API Authentication Methods ==="
echo ""

echo "1. API Key Authentication:"
curl -s -X GET \${API_URL}/products -H "x-api-key: ${API_KEY_VALUE}" | jq .
echo ""

echo "2. IAM Authentication:"
aws apigatewaymanagementapi get-connection \
  --endpoint-url \${API_URL}/admin \
  --profile api-test 2>/dev/null || echo "IAM test (use Python script)"
echo ""

echo "3. JWT Token Authentication:"
curl -s -X GET \${API_URL}/secure -H "Authorization: Bearer ${JWT_TOKEN}" | jq .
echo ""

echo "4. Cognito Authentication:"
./cognito-auth.sh
echo ""

echo "=== Test Complete ==="
EOF

chmod +x test-all-auth.sh
```

## Cleanup

**Remove all resources:**
```bash
# Delete API Gateway
aws apigateway delete-rest-api --rest-api-id $API_ID

# Delete Lambda functions
aws lambda delete-function --function-name ProductsHandler
aws lambda delete-function --function-name JwtAuthorizer

# Delete Cognito User Pool
aws cognito-idp delete-user-pool --user-pool-id $USER_POOL_ID

# Delete IAM resources
aws iam detach-user-policy --user-name api-test-user --policy-arn $POLICY_ARN
aws iam delete-access-key --user-name api-test-user --access-key-id $TEST_ACCESS_KEY
aws iam delete-user --user-name api-test-user
aws iam delete-policy --policy-arn $POLICY_ARN

aws iam detach-role-policy \
  --role-name ApiGatewayLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name ApiGatewayLambdaRole

# Delete CloudWatch resources
aws logs delete-log-group --log-group-name /aws/apigateway/${API_ID}
aws cloudwatch delete-alarms \
  --alarm-names "${API_ID}-high-4xx-errors" "${API_ID}-high-5xx-errors" "${API_ID}-high-latency"

# Delete WAF (if created)
aws wafv2 disassociate-web-acl \
  --resource-arn arn:aws:apigateway:us-east-1::/restapis/${API_ID}/stages/prod
aws wafv2 delete-web-acl --name SecureAPIWAF --scope REGIONAL --region us-east-1 --id <web-acl-id>

echo "Cleanup complete!"
```

## Key Takeaways

1. **Multiple Authentication Options**: API Gateway supports API Keys, IAM, Lambda Authorizers, and Cognito - choose based on your use case
2. **Layered Security**: Combine authentication with throttling, WAF, and monitoring for defense in depth
3. **API Keys**: Best for simple partner access, always use with usage plans
4. **IAM Auth**: Ideal for service-to-service communication within AWS
5. **Lambda Authorizers**: Flexible for custom authentication logic and existing identity systems
6. **Cognito**: Perfect for user-facing applications with sign-up/sign-in flows
7. **Monitoring**: Enable CloudWatch logging and metrics for visibility and troubleshooting
8. **Rate Limiting**: Implement at multiple levels (API, stage, method, usage plan) to prevent abuse

## Additional Resources

- [API Gateway Authentication Documentation](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-control-access-to-api.html)
- [Lambda Authorizers Guide](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-use-lambda-authorizer.html)
- [Cognito User Pools with API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-integrate-with-cognito.html)
- [API Gateway Security Best Practices](https://docs.aws.amazon.com/apigateway/latest/developerguide/security-best-practices.html)
