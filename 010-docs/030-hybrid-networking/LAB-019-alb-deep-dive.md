# Lab 19: Application Load Balancer Deep Dive

## Lab Overview
This comprehensive lab explores advanced Application Load Balancer (ALB) features and configurations. Students will implement sophisticated routing rules, authentication mechanisms, and integration patterns for modern web applications in hybrid architectures.

## Prerequisites
- AWS Account with EC2 and ELB permissions
- Understanding of HTTP/HTTPS protocols
- Basic knowledge of web application architectures
- Completed previous load balancing concepts

## Lab Duration
105 minutes

## Learning Objectives
By the end of this lab, you will:
- Configure advanced ALB routing rules and conditions
- Implement authentication integration (OIDC, SAML)
- Set up advanced target group configurations
- Configure SSL/TLS termination and security policies
- Implement request routing based on headers, paths, and host
- Set up monitoring, logging, and troubleshooting
- Integrate ALB with AWS services and hybrid architectures

## Lab Architecture
```
                        Internet
                           │
                    ┌─────────────┐
                    │     ALB     │
                    │  (Layer 7)  │
                    └─────────────┘
                           │
                    ┌─────────────┐
                    │   Listener  │
                    │    Rules    │
                    └─────────────┘
                           │
      ┌────────────────────┼────────────────────┐
      │                    │                    │
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Web App   │    │   API App   │    │  Auth App   │
│Target Group │    │Target Group │    │Target Group │
└─────────────┘    └─────────────┘    └─────────────┘
```

## Lab Steps

### Step 1: Create Multi-Tier Application Environment
1. **Create Application VPC**
   - Navigate to **VPC Console**
   - Create VPC:
     - Name: `ALB-DeepDive-VPC`
     - IPv4 CIDR: `10.50.0.0/16`
     - Enable DNS hostnames and resolution

2. **Create Multi-AZ Subnets**
   - Public Subnets (for ALB):
     - `ALB-Public-1A`: 10.50.1.0/24 (us-east-1a)
     - `ALB-Public-1B`: 10.50.2.0/24 (us-east-1b)
   - Private Subnets (for applications):
     - `ALB-Private-1A`: 10.50.11.0/24 (us-east-1a)
     - `ALB-Private-1B`: 10.50.12.0/24 (us-east-1b)
   - Database Subnets:
     - `ALB-DB-1A`: 10.50.21.0/24 (us-east-1a)
     - `ALB-DB-1B`: 10.50.22.0/24 (us-east-1b)

3. **Configure Internet Gateway and NAT**
   - Attach Internet Gateway
   - Create NAT Gateways in public subnets
   - Configure route tables for each tier

### Step 2: Deploy Multi-Tier Applications
1. **Deploy Web Application Tier**
   - Launch EC2 instances in private subnets:
   ```bash
   #!/bin/bash
   yum update -y
   yum install -y httpd php
   systemctl start httpd
   systemctl enable httpd
   
   # Get instance metadata
   INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
   AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
   PRIVATE_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
   
   # Create web application
   cat > /var/www/html/index.php << 'EOF'
   <!DOCTYPE html>
   <html>
   <head>
       <title>ALB Deep Dive - Web App</title>
       <style>
           body { font-family: Arial; margin: 40px; background: #f0f8ff; }
           .container { background: white; padding: 20px; border-radius: 8px; box-shadow: 0 2px 4px rgba(0,0,0,0.1); }
           .header { color: #2e8b57; border-bottom: 2px solid #2e8b57; padding-bottom: 10px; }
           .info { margin: 10px 0; padding: 10px; background: #f9f9f9; border-left: 4px solid #2e8b57; }
       </style>
   </head>
   <body>
       <div class="container">
           <h1 class="header">Web Application Tier</h1>
           <div class="info">
               <strong>Instance ID:</strong> <?php echo gethostname(); ?><br>
               <strong>Availability Zone:</strong> <?php echo file_get_contents('http://169.254.169.254/latest/meta-data/placement/availability-zone'); ?><br>
               <strong>Private IP:</strong> <?php echo file_get_contents('http://169.254.169.254/latest/meta-data/local-ipv4'); ?><br>
               <strong>Timestamp:</strong> <?php echo date('Y-m-d H:i:s'); ?><br>
               <strong>Application:</strong> Web Frontend<br>
               <strong>User Agent:</strong> <?php echo $_SERVER['HTTP_USER_AGENT'] ?? 'Not Available'; ?>
           </div>
           
           <h3>Request Headers:</h3>
           <div class="info">
               <?php
               foreach (getallheaders() as $name => $value) {
                   echo "<strong>$name:</strong> $value<br>";
               }
               ?>
           </div>
           
           <h3>Health Check</h3>
           <p><a href="/health">Health Check Endpoint</a></p>
       </div>
   </body>
   </html>
   EOF
   
   # Create health check endpoint
   cat > /var/www/html/health << 'EOF'
   OK
   EOF
   
   # Create admin section
   mkdir -p /var/www/html/admin
   cat > /var/www/html/admin/index.php << 'EOF'
   <!DOCTYPE html>
   <html>
   <head><title>Admin Panel</title></head>
   <body style="font-family: Arial; margin: 40px; background: #ffe4e1;">
       <h1 style="color: #dc143c;">Admin Panel</h1>
       <p><strong>Instance:</strong> <?php echo gethostname(); ?></p>
       <p><strong>Access Time:</strong> <?php echo date('Y-m-d H:i:s'); ?></p>
       <p>This is the admin section - requires authentication</p>
   </body>
   </html>
   EOF
   ```

2. **Deploy API Application Tier**
   - Launch instances for API services:
   ```bash
   #!/bin/bash
   yum update -y
   yum install -y httpd php
   systemctl start httpd
   systemctl enable httpd
   
   # Create API application
   cat > /var/www/html/index.php << 'EOF'
   <?php
   header('Content-Type: application/json');
   
   $data = [
       'service' => 'API Application',
       'instance_id' => gethostname(),
       'az' => file_get_contents('http://169.254.169.254/latest/meta-data/placement/availability-zone'),
       'private_ip' => file_get_contents('http://169.254.169.254/latest/meta-data/local-ipv4'),
       'timestamp' => date('c'),
       'version' => 'v1.0',
       'status' => 'healthy'
   ];
   
   echo json_encode($data, JSON_PRETTY_PRINT);
   ?>
   EOF
   
   # Create API endpoints
   mkdir -p /var/www/html/api/v1
   cat > /var/www/html/api/v1/users.php << 'EOF'
   <?php
   header('Content-Type: application/json');
   echo json_encode([
       'endpoint' => 'users',
       'method' => $_SERVER['REQUEST_METHOD'],
       'instance' => gethostname(),
       'data' => ['user1', 'user2', 'user3']
   ]);
   ?>
   EOF
   
   cat > /var/www/html/api/v1/products.php << 'EOF'
   <?php
   header('Content-Type: application/json');
   echo json_encode([
       'endpoint' => 'products',
       'method' => $_SERVER['REQUEST_METHOD'],
       'instance' => gethostname(),
       'data' => ['product1', 'product2', 'product3']
   ]);
   ?>
   EOF
   
   # Health check
   cat > /var/www/html/health << 'EOF'
   OK
   EOF
   ```

3. **Deploy Authentication Service**
   - Launch instance for authentication simulation:
   ```bash
   #!/bin/bash
   yum update -y
   yum install -y httpd php
   systemctl start httpd
   systemctl enable httpd
   
   cat > /var/www/html/index.php << 'EOF'
   <!DOCTYPE html>
   <html>
   <head><title>Authentication Service</title></head>
   <body style="font-family: Arial; margin: 40px; background: #f0f8ff;">
       <h1 style="color: #4169e1;">Authentication Service</h1>
       <p><strong>Instance:</strong> <?php echo gethostname(); ?></p>
       <p><strong>Service:</strong> Identity Provider Simulation</p>
       <p><strong>Status:</strong> Active</p>
       
       <h3>Login Form</h3>
       <form action="/login" method="post">
           <p>Username: <input type="text" name="username" value="testuser"></p>
           <p>Password: <input type="password" name="password" value="password"></p>
           <p><input type="submit" value="Login"></p>
       </form>
   </body>
   </html>
   EOF
   
   cat > /var/www/html/health << 'EOF'
   OK
   EOF
   ```

### Step 3: Create Application Load Balancer
1. **Create ALB**
   - Navigate to **EC2** → **Load Balancers**
   - Click **Create Load Balancer**
   - Choose **Application Load Balancer**
   - Configuration:
     - Name: `ALB-DeepDive-LoadBalancer`
     - Scheme: Internet-facing
     - IP address type: IPv4

2. **Configure Network Mapping**
   - VPC: ALB-DeepDive-VPC
   - Mappings:
     - us-east-1a: ALB-Public-1A
     - us-east-1b: ALB-Public-1B

3. **Configure Security Groups**
   - Create ALB Security Group:
     - Name: `ALB-DeepDive-SG`
     - Inbound rules:
       - HTTP (80) from 0.0.0.0/0
       - HTTPS (443) from 0.0.0.0/0

### Step 4: Create Target Groups
1. **Create Web Application Target Group**
   - Navigate to **EC2** → **Target Groups**
   - Create target group:
     - Name: `Web-App-TG`
     - Protocol: HTTP
     - Port: 80
     - VPC: ALB-DeepDive-VPC
     - Health check path: `/health`
     - Health check interval: 30 seconds

2. **Create API Application Target Group**
   - Create target group:
     - Name: `API-App-TG`
     - Protocol: HTTP
     - Port: 80
     - Health check path: `/health`

3. **Create Auth Service Target Group**
   - Create target group:
     - Name: `Auth-Service-TG`
     - Protocol: HTTP
     - Port: 80
     - Health check path: `/health`

4. **Register Targets**
   - Register appropriate EC2 instances to each target group
   - Verify all targets show healthy status

### Step 5: Configure Advanced Routing Rules
1. **Create Default Listener (HTTP)**
   - Return to your ALB
   - Configure HTTP:80 listener
   - Default action: Forward to Web-App-TG

2. **Add Path-Based Routing Rules**
   - Edit the HTTP listener
   - Add rules:
     
     **Rule 1: API Routing**
     - Condition: Path is `/api/*`
     - Action: Forward to API-App-TG
     - Priority: 10
     
     **Rule 2: Admin Section**
     - Condition: Path is `/admin/*`
     - Action: Forward to Web-App-TG
     - Add authentication action (we'll configure this)
     
     **Rule 3: Auth Service**
     - Condition: Path is `/auth/*`
     - Action: Forward to Auth-Service-TG
     - Priority: 30

3. **Add Header-Based Routing**
   - Add rule:
     - Condition: HTTP header `X-API-Version` is `v2`
     - Action: Return fixed response
     - Response: 200 OK, "API v2 not available"

4. **Add Host-Based Routing**
   - Add rule:
     - Condition: Host header is `api.example.com`
     - Action: Forward to API-App-TG

### Step 6: Configure SSL/TLS Termination
1. **Request SSL Certificate**
   - Navigate to **AWS Certificate Manager**
   - Request public certificate for your domain
   - Or create self-signed certificate for testing

2. **Add HTTPS Listener**
   - In ALB, add listener:
     - Protocol: HTTPS
     - Port: 443
     - SSL certificate: Select your certificate
     - Security policy: ELBSecurityPolicy-TLS-1-2-2017-01

3. **Configure SSL Redirect**
   - Edit HTTP listener
   - Add rule to redirect HTTP to HTTPS:
     - Condition: All requests
     - Action: Redirect to HTTPS
     - Status code: 301

### Step 7: Implement Advanced Features
1. **Configure Sticky Sessions**
   - Edit Web-App-TG
   - Enable stickiness:
     - Type: Load balancer generated cookie
     - Duration: 1 hour

2. **Configure Cross-Zone Load Balancing**
   - Edit ALB attributes
   - Enable cross-zone load balancing
   - This is enabled by default for ALB

3. **Configure Connection Draining**
   - Set deregistration delay: 300 seconds
   - This allows graceful shutdown

### Step 8: Implement Authentication Integration
1. **Configure OIDC Authentication** (Simulation)
   - For admin paths, add authentication action:
   - Edit the admin rule
   - Add action: Authenticate (OIDC)
   - Configuration:
     - Issuer: `https://your-oidc-provider.com`
     - Authorization endpoint: `https://provider/auth`
     - Token endpoint: `https://provider/token`
     - Client ID: `your-client-id`
     - Client secret: Store in Secrets Manager

2. **Test Authentication Flow**
   - Access `/admin/` path
   - Should redirect to authentication
   - After authentication, forward to target

### Step 9: Configure Advanced Health Checks
1. **Custom Health Check Paths**
   - Create detailed health check endpoints:
   ```php
   <?php
   // /var/www/html/detailed-health
   $status = [
       'status' => 'healthy',
       'checks' => [
           'database' => 'connected',
           'cache' => 'available',
           'disk_space' => '85%',
           'memory' => '60%'
       ],
       'instance' => gethostname(),
       'timestamp' => date('c')
   ];
   
   header('Content-Type: application/json');
   echo json_encode($status);
   ?>
   ```

2. **Configure Health Check Parameters**
   - Health check interval: 15 seconds
   - Healthy threshold: 2
   - Unhealthy threshold: 5
   - Timeout: 10 seconds
   - Success codes: 200

### Step 10: Implement Monitoring and Logging
1. **Enable Access Logs**
   - Create S3 bucket for ALB access logs
   - Configure ALB to send access logs:
     - Navigate to ALB attributes
     - Enable access logs
     - Specify S3 bucket and prefix

2. **Configure CloudWatch Metrics**
   - Monitor key ALB metrics:
     - RequestCount
     - TargetResponseTime
     - HTTPCode_Target_2XX_Count
     - HTTPCode_Target_5XX_Count
     - ActiveConnectionCount
     - NewConnectionCount

3. **Create CloudWatch Dashboard**
   ```json
   {
     "widgets": [
       {
         "type": "metric",
         "properties": {
           "metrics": [
             ["AWS/ApplicationELB", "RequestCount", "LoadBalancer", "app/your-alb-name"],
             [".", "TargetResponseTime", ".", "."],
             [".", "HTTPCode_Target_5XX_Count", ".", "."]
           ],
           "period": 300,
           "stat": "Sum",
           "region": "us-east-1",
           "title": "ALB Performance Metrics"
         }
       }
     ]
   }
   ```

4. **Set Up CloudWatch Alarms**
   ```bash
   # High error rate alarm
   aws cloudwatch put-metric-alarm \
     --alarm-name "ALB-High-Error-Rate" \
     --alarm-description "High 5XX error rate on ALB" \
     --metric-name HTTPCode_Target_5XX_Count \
     --namespace AWS/ApplicationELB \
     --statistic Sum \
     --period 300 \
     --threshold 10 \
     --comparison-operator GreaterThanThreshold \
     --evaluation-periods 2
   ```

### Step 11: Advanced Testing and Validation
1. **Test Routing Rules**
   ```bash
   ALB_DNS="your-alb-dns-name.elb.amazonaws.com"
   
   # Test default web app
   curl http://$ALB_DNS/
   
   # Test API routing
   curl http://$ALB_DNS/api/v1/users.php
   curl http://$ALB_DNS/api/v1/products.php
   
   # Test admin routing (should require auth)
   curl http://$ALB_DNS/admin/
   
   # Test header-based routing
   curl -H "X-API-Version: v2" http://$ALB_DNS/
   
   # Test host-based routing
   curl -H "Host: api.example.com" http://$ALB_DNS/
   ```

2. **Test Load Distribution**
   ```bash
   # Test multiple requests to see load balancing
   for i in {1..20}; do
     curl -s http://$ALB_DNS/ | grep "Instance ID"
     sleep 1
   done
   ```

3. **Test Health Check Behavior**
   ```bash
   # Stop httpd on one instance to test failover
   sudo systemctl stop httpd
   
   # Monitor target health
   aws elbv2 describe-target-health \
     --target-group-arn arn:aws:elasticloadbalancing:region:account:targetgroup/Web-App-TG/1234567890123456
   ```

### Step 12: Performance Optimization
1. **Connection Multiplexing**
   - ALB automatically handles HTTP/2
   - Test HTTP/2 support:
   ```bash
   curl --http2 -I https://$ALB_DNS/
   ```

2. **Request Tracing**
   - Enable request tracing
   - Add X-Amzn-Trace-Id header analysis
   - Use for debugging and performance analysis

3. **WebSocket Support**
   - ALB supports WebSocket connections
   - Test WebSocket upgrade requests
   - Monitor connection duration

## Lab Validation
Complete these verification steps:

1. **ALB Configuration**
   - [ ] ALB created with multiple listeners
   - [ ] SSL/TLS termination working
   - [ ] Multiple target groups configured
   - [ ] Advanced routing rules active

2. **Routing Functionality**
   - [ ] Path-based routing working
   - [ ] Header-based routing working
   - [ ] Host-based routing working
   - [ ] Authentication integration functional

3. **Monitoring and Logging**
   - [ ] Access logs being generated
   - [ ] CloudWatch metrics available
   - [ ] Health checks working properly
   - [ ] Alarms configured and tested

## Advanced Scenarios

### Blue/Green Deployments
```bash
# Create blue and green target groups
# Use weighted routing to gradually shift traffic
# Monitor application metrics during deployment
```

### Canary Deployments
```bash
# Route small percentage of traffic to new version
# Monitor error rates and performance
# Gradually increase traffic percentage
```

### A/B Testing
```bash
# Route traffic based on user attributes
# Use custom headers or cookies
# Analyze conversion metrics
```

## Troubleshooting Common Issues

### 502 Bad Gateway Errors
```bash
# Check target group health
# Verify application is listening on correct port
# Check security groups allow ALB traffic
# Review application logs for errors
```

### SSL Certificate Issues
```bash
# Verify certificate is valid and not expired
# Check certificate chain is complete
# Verify domain names match
# Check security policy compatibility
```

### Routing Rule Conflicts
```bash
# Review rule priorities
# Check condition logic
# Test rule evaluation order
# Use ALB access logs for troubleshooting
```

### Performance Issues
```bash
# Monitor target response times
# Check for connection limits
# Review instance capacity
# Analyze access log patterns
```

## Cost Optimization
1. **ALB Pricing Components**
   - Hourly ALB charges
   - Load Balancer Capacity Units (LCU)
   - Data processing charges

2. **Optimization Strategies**
   - Right-size target instances
   - Optimize health check frequency
   - Use connection draining effectively
   - Monitor LCU consumption

## Security Best Practices
1. **Transport Security**
   - Use strong SSL/TLS policies
   - Implement HSTS headers
   - Regular certificate rotation

2. **Access Control**
   - Implement WAF integration
   - Use security groups effectively
   - Enable access logging
   - Monitor for suspicious patterns

## Clean Up
To avoid ongoing charges:

1. **Delete ALB Resources**
   - Delete Application Load Balancer
   - Delete target groups
   - Delete security groups

2. **Clean Up Infrastructure**
   - Terminate EC2 instances
   - Delete VPC and subnets
   - Remove certificates (if self-signed)

3. **Clean Up Monitoring**
   - Delete CloudWatch dashboards
   - Remove alarms
   - Empty and delete S3 bucket for logs

## Key Takeaways
- ALB provides sophisticated Layer 7 routing capabilities
- Advanced routing rules enable complex application architectures
- SSL/TLS termination offloads encryption from backend servers
- Authentication integration enables secure application access
- Comprehensive monitoring and logging aid in troubleshooting
- Performance features like HTTP/2 and WebSocket support modern applications
- Cost optimization requires understanding LCU consumption patterns

## Next Steps
- Proceed to Lab 24: Cloud WAN vs Transit Gateway
- Explore ALB integration with AWS WAF
- Implement production-grade authentication flows
- Consider Global Load Balancer patterns with Route 53
- Explore container integration with ECS and EKS