# Topic 19: Application Load Balancer Deep Dive

## Prerequisites
- Completed Topic 18 (Network Load Balancer for Hybrid Architecture)
- Understanding of HTTP/HTTPS protocols and Layer 7 concepts
- Knowledge of SSL/TLS certificate management
- Familiarity with microservices and container architectures

## Learning Objectives
By the end of this topic, you will be able to:
- Design sophisticated ALB routing strategies
- Implement advanced SSL/TLS configurations
- Configure WebSocket and HTTP/2 support
- Optimize ALB performance for complex applications

## Theory

### Application Load Balancer Advanced Architecture

#### Layer 7 Load Balancing
Application Load Balancer operates at the application layer (Layer 7), enabling intelligent routing decisions based on HTTP/HTTPS content including:
- **Path-based Routing**: Route based on URL paths
- **Host-based Routing**: Route based on host headers
- **Header-based Routing**: Route based on HTTP headers
- **Query String Routing**: Route based on query parameters

#### Advanced Routing Capabilities
```
Client Request → ALB → Listener Rules → Target Groups → Targets
                 ↓
            Content-based
              Routing
```

### ALB Advanced Features

#### Sticky Sessions (Session Affinity)
- **Duration-based**: Fixed time period
- **Application-based**: Custom cookie management
- **Use Cases**: Stateful applications, shopping carts
- **Considerations**: Can impact load distribution

#### WebSocket Support
- **Persistent Connections**: Long-lived bidirectional communication
- **Automatic Detection**: Seamless WebSocket upgrade handling
- **Load Balancing**: Distributes WebSocket connections across targets
- **Use Cases**: Real-time applications, chat systems

#### HTTP/2 Support
- **Client-side HTTP/2**: Browser to ALB communication
- **Server-side HTTP/1.1**: ALB to targets (backward compatibility)
- **Multiplexing**: Multiple requests over single connection
- **Performance Benefits**: Reduced latency, improved efficiency

### SSL/TLS Advanced Configuration

#### Server Name Indication (SNI)
- **Multiple Certificates**: Support multiple SSL certificates per listener
- **Host-based Selection**: Certificate selection based on hostname
- **Wildcard Support**: Wildcard and multi-domain certificates
- **Certificate Sources**: ACM, IAM, or imported certificates

#### SSL Policies
- **Security Levels**: Various predefined security policies
- **Cipher Suites**: Control encryption algorithms
- **Protocol Versions**: TLS 1.0, 1.1, 1.2, 1.3 support
- **Perfect Forward Secrecy**: Enhanced security through ephemeral keys

## Lab Exercise: Advanced ALB Implementation

### Lab Duration: 360 minutes

### Step 1: Design Advanced ALB Architecture
**Objective**: Plan sophisticated ALB deployment for microservices

**ALB Architecture Planning**:
```bash
# Create advanced ALB planning assessment
cat << 'EOF' > alb-advanced-planning.sh
#!/bin/bash

echo "=== Advanced Application Load Balancer Planning ==="
echo ""

echo "1. APPLICATION ARCHITECTURE ANALYSIS"
echo "   Application type: Microservices/Monolith/Hybrid"
echo "   Number of services: _______"
echo "   Service communication: HTTP/HTTPS/WebSocket/gRPC"
echo "   Session requirements: Stateful/Stateless"
echo "   Real-time features: WebSocket/Server-Sent Events/Long Polling"
echo ""

echo "2. ROUTING STRATEGY DESIGN"
echo "   Path-based routing requirements: Yes/No"
echo "   Host-based routing requirements: Yes/No"
echo "   Header-based routing requirements: Yes/No"
echo "   Query parameter routing: Yes/No"
echo "   Blue/Green deployment support: Yes/No"
echo "   Canary deployment support: Yes/No"
echo ""

echo "3. SSL/TLS REQUIREMENTS"
echo "   Number of domains: _______"
echo "   Wildcard certificates needed: Yes/No"
echo "   SNI requirements: Yes/No"
echo "   SSL termination: ALB/End-to-end"
echo "   Client certificate authentication: Yes/No"
echo "   HSTS requirements: Yes/No"
echo ""

echo "4. PERFORMANCE REQUIREMENTS"
echo "   Expected requests per second: _______"
echo "   Geographic distribution: Single/Multi-region"
echo "   Caching requirements: CloudFront/ALB/Application"
echo "   Compression requirements: Yes/No"
echo "   Keep-alive optimization: Yes/No"
echo ""

echo "5. SECURITY AND COMPLIANCE"
echo "   WAF integration: Yes/No"
echo "   Authentication: Cognito/OIDC/SAML"
echo "   Authorization requirements: _______"
echo "   Compliance standards: SOC/PCI/HIPAA"
echo "   Rate limiting requirements: Yes/No"
echo ""

echo "6. MONITORING AND OBSERVABILITY"
echo "   Access logging: Standard/Enhanced"
echo "   Request tracing: X-Ray/Third-party"
echo "   Custom metrics: Yes/No"
echo "   Real-time monitoring: Yes/No"
echo "   Error tracking: Yes/No"

EOF

chmod +x alb-advanced-planning.sh
./alb-advanced-planning.sh
```

### Step 2: Create Multi-Service Target Infrastructure
**Objective**: Deploy multiple services for advanced ALB routing

**Multi-Service Infrastructure Setup**:
```bash
# Create multi-service infrastructure for ALB
cat << 'EOF' > setup-multiservice-infrastructure.sh
#!/bin/bash

# Reuse VPC from previous NLB lab if available
if [ -f "nlb-vpc-config.env" ]; then
    source nlb-vpc-config.env
else
    echo "Please run NLB lab first or set VPC variables manually"
    exit 1
fi

echo "=== Setting Up Multi-Service Infrastructure for ALB ==="
echo ""

echo "1. CREATING SECURITY GROUPS FOR SERVICES"

# API service security group
API_SG_ID=$(aws ec2 create-security-group \
    --group-name ALB-API-Service-SG \
    --description "Security group for API service" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=ALB-API-Service-SG}]" \
    --query 'GroupId' \
    --output text)

# Web service security group
WEB_SG_ID=$(aws ec2 create-security-group \
    --group-name ALB-Web-Service-SG \
    --description "Security group for Web service" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=ALB-Web-Service-SG}]" \
    --query 'GroupId' \
    --output text)

# Admin service security group
ADMIN_SG_ID=$(aws ec2 create-security-group \
    --group-name ALB-Admin-Service-SG \
    --description "Security group for Admin service" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=ALB-Admin-Service-SG}]" \
    --query 'GroupId' \
    --output text)

echo "✅ Security groups created:"
echo "  API Service: $API_SG_ID"
echo "  Web Service: $WEB_SG_ID"
echo "  Admin Service: $ADMIN_SG_ID"

# Configure security group rules
for sg_id in $API_SG_ID $WEB_SG_ID $ADMIN_SG_ID; do
    # Allow HTTP traffic from ALB
    aws ec2 authorize-security-group-ingress \
        --group-id $sg_id \
        --protocol tcp \
        --port 80 \
        --cidr 10.0.0.0/16
    
    # Allow HTTPS traffic from ALB
    aws ec2 authorize-security-group-ingress \
        --group-id $sg_id \
        --protocol tcp \
        --port 443 \
        --cidr 10.0.0.0/16
    
    # Allow custom application ports
    aws ec2 authorize-security-group-ingress \
        --group-id $sg_id \
        --protocol tcp \
        --port 8080 \
        --cidr 10.0.0.0/16
        
    # Allow WebSocket port
    aws ec2 authorize-security-group-ingress \
        --group-id $sg_id \
        --protocol tcp \
        --port 3000 \
        --cidr 10.0.0.0/16
    
    # Allow SSH for management
    aws ec2 authorize-security-group-ingress \
        --group-id $sg_id \
        --protocol tcp \
        --port 22 \
        --cidr 10.0.0.0/16
done

echo "✅ Security group rules configured"

# Save configuration
cat << CONFIG > alb-multiservice-config.env
export API_SG_ID="$API_SG_ID"
export WEB_SG_ID="$WEB_SG_ID"
export ADMIN_SG_ID="$ADMIN_SG_ID"
CONFIG

echo ""
echo "Multi-service infrastructure setup complete"
echo "Configuration saved to alb-multiservice-config.env"

EOF

chmod +x setup-multiservice-infrastructure.sh
./setup-multiservice-infrastructure.sh
```

### Step 3: Create Advanced Application Load Balancer
**Objective**: Deploy ALB with sophisticated routing rules

**Advanced ALB Creation**:
```bash
# Create advanced Application Load Balancer
cat << 'EOF' > create-advanced-alb.sh
#!/bin/bash

source nlb-vpc-config.env
source alb-multiservice-config.env

echo "=== Creating Advanced Application Load Balancer ==="
echo ""

ALB_NAME="Advanced-Application-LB"

echo "1. CREATING APPLICATION LOAD BALANCER"
echo "Name: $ALB_NAME"
echo "Scheme: internet-facing"
echo "Type: application"
echo ""

# Create ALB
ALB_ARN=$(aws elbv2 create-load-balancer \
    --name $ALB_NAME \
    --scheme internet-facing \
    --type application \
    --ip-address-type ipv4 \
    --subnets ${PUBLIC_SUBNET_IDS[@]} \
    --security-groups $TARGET_SG_ID \
    --tags Key=Name,Value=$ALB_NAME Key=Purpose,Value=Advanced-Routing \
    --query 'LoadBalancers[0].LoadBalancerArn' \
    --output text)

echo "✅ Application Load Balancer created: $ALB_ARN"

# Wait for ALB to be active
echo "Waiting for ALB to become active..."
aws elbv2 wait load-balancer-available --load-balancer-arns $ALB_ARN
echo "✅ ALB is now active"

# Get ALB DNS name
ALB_DNS=$(aws elbv2 describe-load-balancers \
    --load-balancer-arns $ALB_ARN \
    --query 'LoadBalancers[0].DNSName' \
    --output text)

echo "ALB DNS Name: $ALB_DNS"

echo ""
echo "2. CREATING TARGET GROUPS FOR DIFFERENT SERVICES"

# Create target groups for API, Web, and Admin services
TG_API_ARN=$(aws elbv2 create-target-group \
    --name ALB-API-Service-TG \
    --protocol HTTP \
    --port 8080 \
    --vpc-id $VPC_ID \
    --target-type instance \
    --health-check-protocol HTTP \
    --health-check-path /api/health \
    --health-check-port 8080 \
    --health-check-interval-seconds 30 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 3 \
    --matcher HttpCode=200 \
    --tags Key=Name,Value=ALB-API-Service-TG \
    --query 'TargetGroups[0].TargetGroupArn' \
    --output text)

echo "✅ API service target group created: $TG_API_ARN"

# Create HTTP listener with advanced routing
LISTENER_HTTP_ARN=$(aws elbv2 create-listener \
    --load-balancer-arn $ALB_ARN \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=fixed-response,FixedResponseConfig='{StatusCode=200,ContentType=text/plain,MessageBody=ALB-Health-Check}' \
    --tags Key=Name,Value=ALB-HTTP-Listener \
    --query 'Listeners[0].ListenerArn' \
    --output text)

echo "✅ HTTP listener created: $LISTENER_HTTP_ARN"

# Save configuration
cat << CONFIG > alb-advanced-config.env
export ALB_ARN="$ALB_ARN"
export ALB_DNS="$ALB_DNS"
export TG_API_ARN="$TG_API_ARN"
export LISTENER_HTTP_ARN="$LISTENER_HTTP_ARN"
CONFIG

echo ""
echo "ADVANCED ALB SUMMARY:"
echo "ALB ARN: $ALB_ARN"
echo "ALB DNS: $ALB_DNS"
echo "API Target Group: $TG_API_ARN"
echo ""
echo "Configuration saved to alb-advanced-config.env"

EOF

chmod +x create-advanced-alb.sh
./create-advanced-alb.sh
```

### Step 4: Implement SSL/TLS with SNI
**Objective**: Configure advanced SSL/TLS features

**SSL/TLS Configuration**:
```bash
# Configure SSL/TLS with SNI for ALB
cat << 'EOF' > configure-alb-ssl-sni.sh
#!/bin/bash

source alb-advanced-config.env

echo "=== Configuring SSL/TLS with SNI for ALB ==="
echo ""

echo "1. CREATING SSL CERTIFICATES"

# Note: In production, use real domains and DNS validation
DOMAINS=("app.example.com" "api.example.com" "admin.example.com")

# Create self-signed certificates for demo (use ACM in production)
for domain in "${DOMAINS[@]}"; do
    echo "Creating certificate for $domain..."
    
    # Create private key
    openssl genrsa -out ${domain}.key 2048
    
    # Create certificate signing request
    openssl req -new -key ${domain}.key -out ${domain}.csr -subj "/C=US/ST=CA/L=San Francisco/O=Example Corp/CN=$domain"
    
    # Create self-signed certificate
    openssl x509 -req -days 365 -in ${domain}.csr -signkey ${domain}.key -out ${domain}.crt
    
    echo "✅ Certificate created for $domain"
done

echo ""
echo "2. IMPORTING CERTIFICATES TO ACM"

# Import certificates to AWS Certificate Manager
declare -A CERT_ARNS

for domain in "${DOMAINS[@]}"; do
    echo "Importing certificate for $domain to ACM..."
    
    CERT_ARN=$(aws acm import-certificate \
        --certificate fileb://${domain}.crt \
        --private-key fileb://${domain}.key \
        --tags Key=Name,Value=${domain}-certificate \
        --query 'CertificateArn' \
        --output text)
    
    CERT_ARNS[$domain]=$CERT_ARN
    echo "✅ Certificate imported for $domain: $CERT_ARN"
done

echo ""
echo "3. CREATING HTTPS LISTENER WITH SNI"

# Create HTTPS listener with default certificate
DEFAULT_CERT_ARN=${CERT_ARNS["app.example.com"]}

LISTENER_HTTPS_ARN=$(aws elbv2 create-listener \
    --load-balancer-arn $ALB_ARN \
    --protocol HTTPS \
    --port 443 \
    --certificates CertificateArn=$DEFAULT_CERT_ARN \
    --ssl-policy ELBSecurityPolicy-TLS-1-2-2017-01 \
    --default-actions Type=fixed-response,FixedResponseConfig='{StatusCode=200,ContentType=text/plain,MessageBody=HTTPS-Default-Response}' \
    --tags Key=Name,Value=ALB-HTTPS-Listener \
    --query 'Listeners[0].ListenerArn' \
    --output text)

echo "✅ HTTPS listener created: $LISTENER_HTTPS_ARN"

# Save SSL configuration
cat << CONFIG > alb-ssl-config.env
export LISTENER_HTTPS_ARN="$LISTENER_HTTPS_ARN"
export DEFAULT_CERT_ARN="$DEFAULT_CERT_ARN"
CONFIG

echo ""
echo "SSL/TLS CONFIGURATION SUMMARY:"
echo "HTTPS Listener: $LISTENER_HTTPS_ARN"
echo "Default Certificate: $DEFAULT_CERT_ARN"
echo "SSL Policy: ELBSecurityPolicy-TLS-1-2-2017-01"
echo ""
echo "Configuration saved to alb-ssl-config.env"

EOF

chmod +x configure-alb-ssl-sni.sh
./configure-alb-ssl-sni.sh
```

### Step 5: Implement WebSocket and HTTP/2 Support
**Objective**: Configure advanced protocol support

**Protocol Support Configuration**:
```bash
# Configure WebSocket and HTTP/2 support
cat << 'EOF' > configure-websocket-http2.sh
#!/bin/bash

source alb-advanced-config.env

echo "=== Configuring WebSocket and HTTP/2 Support ==="
echo ""

echo "1. WEBSOCKET SUPPORT VERIFICATION"
echo "WebSocket support is automatically enabled on ALB"
echo "ALB automatically detects WebSocket upgrade requests"
echo "Connection upgrade headers are preserved and forwarded"
echo ""

echo "2. HTTP/2 SUPPORT VERIFICATION"
echo "HTTP/2 support is automatically enabled for HTTPS listeners"
echo "ALB supports HTTP/2 from clients to ALB"
echo "ALB uses HTTP/1.1 from ALB to targets for compatibility"
echo ""

echo "3. CREATING WEBSOCKET TEST CLIENT"
cat << 'WEBSOCKET_TEST' > websocket-test.html
<!DOCTYPE html>
<html>
<head>
    <title>WebSocket Test Client</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        .test-container { background: #f5f5f5; padding: 20px; border-radius: 5px; margin: 20px 0; }
        .status { padding: 10px; border-radius: 3px; margin: 10px 0; }
        .connected { background: #d4edda; color: #155724; }
        .disconnected { background: #f8d7da; color: #721c24; }
        #messages { height: 200px; overflow-y: scroll; border: 1px solid #ccc; padding: 10px; background: white; }
    </style>
</head>
<body>
    <h1>WebSocket Test Client for ALB</h1>
    
    <div class="test-container">
        <h2>Connection Test</h2>
        <button onclick="connectWebSocket()">Connect WebSocket</button>
        <button onclick="disconnectWebSocket()">Disconnect</button>
        <button onclick="sendMessage()">Send Test Message</button>
        
        <div id="status" class="status disconnected">Disconnected</div>
        
        <h3>Messages</h3>
        <div id="messages"></div>
    </div>
    
    <script>
        let ws = null;
        const statusDiv = document.getElementById('status');
        const messagesDiv = document.getElementById('messages');
        
        function updateStatus(message, type) {
            statusDiv.textContent = message;
            statusDiv.className = 'status ' + type;
        }
        
        function addMessage(message) {
            const timestamp = new Date().toLocaleTimeString();
            messagesDiv.innerHTML += `<div>[${timestamp}] ${message}</div>`;
            messagesDiv.scrollTop = messagesDiv.scrollHeight;
        }
        
        function connectWebSocket() {
            try {
                ws = new WebSocket('ws://' + window.location.hostname + '/ws');
                
                ws.onopen = function() {
                    updateStatus('Connected to WebSocket', 'connected');
                    addMessage('WebSocket connection established');
                };
                
                ws.onmessage = function(event) {
                    addMessage('Received: ' + event.data);
                };
                
                ws.onclose = function() {
                    updateStatus('WebSocket disconnected', 'disconnected');
                    addMessage('WebSocket connection closed');
                };
                
                ws.onerror = function(error) {
                    updateStatus('WebSocket error', 'error');
                    addMessage('Error: Connection failed');
                };
                
            } catch (error) {
                updateStatus('Connection failed', 'error');
                addMessage('Failed to connect: ' + error.message);
            }
        }
        
        function disconnectWebSocket() {
            if (ws) {
                ws.close();
                ws = null;
            }
        }
        
        function sendMessage() {
            if (ws && ws.readyState === WebSocket.OPEN) {
                const message = 'Test message: ' + new Date().toISOString();
                ws.send(message);
                addMessage('Sent: ' + message);
            } else {
                alert('WebSocket not connected');
            }
        }
    </script>
</body>
</html>
WEBSOCKET_TEST

echo "✅ WebSocket test client created: websocket-test.html"

echo ""
echo "4. CREATING HTTP/2 TEST SCRIPT"
cat << 'HTTP2_TEST' > test-http2-support.sh
#!/bin/bash

ALB_DNS="$ALB_DNS"

echo "Testing HTTP/2 support on ALB..."
echo "ALB DNS: $ALB_DNS"

# Test with curl (if HTTP/2 support is available)
if curl --version | grep -q "HTTP2"; then
    echo "Testing HTTP/2 connection:"
    curl -I --http2 https://$ALB_DNS/ 2>/dev/null | head -5
    
    echo ""
    echo "Testing HTTP/2 with verbose output:"
    curl -I --http2 -v https://$ALB_DNS/ 2>&1 | grep -E "(HTTP/2|h2|ALPN)"
else
    echo "curl does not support HTTP/2"
    echo "Use browser developer tools to verify HTTP/2 support"
fi

echo ""
echo "Alternative HTTP/2 testing methods:"
echo "1. Browser Developer Tools - Network tab shows 'h2' protocol"
echo "2. Online tools: https://tools.keycdn.com/http2-test"
echo "3. Command line: nghttp -nv https://$ALB_DNS/"

HTTP2_TEST

chmod +x test-http2-support.sh

echo "✅ HTTP/2 test script created"
echo ""
echo "PROTOCOL SUPPORT CONFIGURATION COMPLETED"
echo ""
echo "Key Points:"
echo "- WebSocket upgrade is automatically handled by ALB"
echo "- HTTP/2 is enabled by default for HTTPS listeners"
echo "- Use websocket-test.html to test WebSocket functionality"
echo "- Run ./test-http2-support.sh to verify HTTP/2 support"

EOF

chmod +x configure-websocket-http2.sh
./configure-websocket-http2.sh
```

### Step 6: Implement Advanced Monitoring
**Objective**: Set up comprehensive ALB monitoring

**Advanced Monitoring Setup**:
```bash
# Configure advanced monitoring for ALB
cat << 'EOF' > setup-alb-advanced-monitoring.sh
#!/bin/bash

source alb-advanced-config.env

echo "=== Setting Up Advanced ALB Monitoring ==="
echo ""

echo "1. ENABLING ALB ACCESS LOGGING"

# Create S3 bucket for ALB access logs
BUCKET_NAME="alb-access-logs-$(date +%s)"
ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
REGION=$(aws configure get region)

aws s3api create-bucket \
    --bucket $BUCKET_NAME \
    --region $REGION

echo "✅ S3 bucket created: $BUCKET_NAME"

# Configure bucket policy for ALB access
ELB_ACCOUNT_ID="027434742980"  # us-east-1 ELB service account

cat << BUCKET_POLICY > alb-bucket-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::$ELB_ACCOUNT_ID:root"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::$BUCKET_NAME/alb-access-logs/AWSLogs/$ACCOUNT_ID/*"
        }
    ]
}
BUCKET_POLICY

aws s3api put-bucket-policy \
    --bucket $BUCKET_NAME \
    --policy file://alb-bucket-policy.json

# Enable ALB access logging
aws elbv2 modify-load-balancer-attributes \
    --load-balancer-arn $ALB_ARN \
    --attributes Key=access_logs.s3.enabled,Value=true \
               Key=access_logs.s3.bucket,Value=$BUCKET_NAME \
               Key=access_logs.s3.prefix,Value=alb-access-logs

echo "✅ ALB access logging enabled"

echo ""
echo "2. CREATING CLOUDWATCH DASHBOARD"

ALB_NAME=$(echo $ALB_ARN | cut -d'/' -f2-)

DASHBOARD_BODY='{
    "widgets": [
        {
            "type": "metric",
            "x": 0,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "metrics": [
                    [ "AWS/ApplicationELB", "RequestCount", "LoadBalancer", "'$ALB_NAME'" ],
                    [ ".", "NewConnectionCount", ".", "." ]
                ],
                "period": 300,
                "stat": "Sum",
                "region": "'$REGION'",
                "title": "ALB Request and Connection Metrics"
            }
        },
        {
            "type": "metric",
            "x": 12,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "metrics": [
                    [ "AWS/ApplicationELB", "TargetResponseTime", "LoadBalancer", "'$ALB_NAME'" ]
                ],
                "period": 300,
                "stat": "Average",
                "region": "'$REGION'",
                "title": "Response Time"
            }
        },
        {
            "type": "metric",
            "x": 0,
            "y": 6,
            "width": 24,
            "height": 6,
            "properties": {
                "metrics": [
                    [ "AWS/ApplicationELB", "HTTPCode_Target_2XX_Count", "LoadBalancer", "'$ALB_NAME'" ],
                    [ ".", "HTTPCode_Target_4XX_Count", ".", "." ],
                    [ ".", "HTTPCode_Target_5XX_Count", ".", "." ]
                ],
                "period": 300,
                "stat": "Sum",
                "region": "'$REGION'",
                "title": "HTTP Response Codes"
            }
        }
    ]
}'

aws cloudwatch put-dashboard \
    --dashboard-name "ALB-Advanced-Monitoring" \
    --dashboard-body "$DASHBOARD_BODY"

echo "✅ CloudWatch dashboard created: ALB-Advanced-Monitoring"

echo ""
echo "3. CREATING CLOUDWATCH ALARMS"

# High error rate alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "ALB-HighErrorRate" \
    --alarm-description "Alert when ALB 5XX error rate is high" \
    --metric-name HTTPCode_Target_5XX_Count \
    --namespace AWS/ApplicationELB \
    --statistic Sum \
    --period 300 \
    --threshold 10 \
    --comparison-operator GreaterThanThreshold \
    --dimensions Name=LoadBalancer,Value=$ALB_NAME \
    --evaluation-periods 2

echo "✅ CloudWatch alarms created"

# Save monitoring configuration
cat << CONFIG > alb-monitoring-config.env
export BUCKET_NAME="$BUCKET_NAME"
export DASHBOARD_NAME="ALB-Advanced-Monitoring"
CONFIG

echo ""
echo "ADVANCED MONITORING CONFIGURATION COMPLETED"
echo ""
echo "Monitoring components:"
echo "- S3 bucket for access logs: $BUCKET_NAME"
echo "- CloudWatch dashboard: ALB-Advanced-Monitoring"
echo "- CloudWatch alarms for error monitoring"
echo ""
echo "View dashboard at: AWS Console → CloudWatch → Dashboards → ALB-Advanced-Monitoring"

EOF

chmod +x setup-alb-advanced-monitoring.sh
./setup-alb-advanced-monitoring.sh
```

## ALB Advanced Features Summary

### Layer 7 Intelligent Routing
- **Path-based Routing**: Direct traffic based on URL paths
- **Host-based Routing**: Route using host headers for multi-tenant applications
- **Header-based Routing**: Advanced routing using HTTP headers
- **Query String Routing**: Route based on URL parameters

### SSL/TLS Advanced Features
- **SNI Support**: Multiple SSL certificates per listener
- **SSL Policies**: Configurable cipher suites and protocol versions
- **Certificate Management**: Integration with ACM for automated renewals
- **Security Headers**: HSTS, CSP, and other security enhancements

### Modern Protocol Support
- **WebSocket**: Automatic upgrade handling and session persistence
- **HTTP/2**: Multiplexing and performance improvements
- **gRPC**: Support for modern microservices communication
- **Keep-Alive**: Connection reuse optimization

### Performance Optimization
- **Connection Draining**: Graceful target deregistration
- **Sticky Sessions**: Session affinity for stateful applications
- **Health Checks**: Configurable health monitoring
- **Load Balancing Algorithms**: Round robin and least outstanding requests

## Best Practices for Advanced ALB

### Design Principles
- **Service-oriented Routing**: Use path and host-based routing for microservices
- **Security First**: Implement SSL/TLS with proper cipher policies
- **Performance Optimization**: Enable HTTP/2 and optimize connection handling
- **Observability**: Comprehensive monitoring and logging

### Security Considerations
- **Certificate Management**: Use ACM for automated certificate lifecycle
- **Security Headers**: Implement HSTS, CSP, and other security headers
- **Access Control**: Use security groups and NACLs appropriately
- **WAF Integration**: Implement AWS WAF for application-layer protection

### Monitoring and Troubleshooting
- **Access Logging**: Enable detailed access logs for analysis
- **CloudWatch Metrics**: Monitor performance and error rates
- **X-Ray Tracing**: Implement distributed tracing for complex applications
- **Custom Metrics**: Track business-specific metrics

## Key Takeaways
- ALB provides sophisticated Layer 7 load balancing with content-based routing
- SNI enables multiple SSL certificates for complex applications
- WebSocket and HTTP/2 support modern application requirements
- Advanced monitoring and observability are essential for production deployments
- Proper SSL/TLS configuration enhances security and performance
- Sticky sessions enable support for stateful applications

## Additional Resources
- [AWS Application Load Balancer User Guide](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/)
- [ALB Advanced Request Routing](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/load-balancer-listeners.html)
- [SSL/TLS Certificates on ALB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-https-listener.html)

## Exam Tips
- ALB operates at Layer 7 (application layer)
- Supports path, host, header, and query string-based routing
- SNI allows multiple SSL certificates per listener
- WebSocket connections require sticky sessions
- HTTP/2 is automatically enabled for HTTPS listeners
- Health checks can be HTTP, HTTPS, or custom
- Access logs provide detailed request information
- Integration with WAF provides application-layer security
- Supports weighted routing for blue/green deployments