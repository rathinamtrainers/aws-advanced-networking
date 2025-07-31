# Lab 23: PrivateLink and VPC Endpoints for Hybrid Connectivity

## Lab Overview
This lab demonstrates how to use AWS PrivateLink and VPC Endpoints to create secure, private connectivity between VPCs, on-premises networks, and AWS services without using the public internet. Students will implement various endpoint types and test hybrid service connectivity patterns.

## Prerequisites
- AWS Account with VPC and PrivateLink permissions
- Understanding of VPC concepts and hybrid networking
- Completed Transit Gateway lab (recommended)
- Basic knowledge of service-oriented architectures

## Lab Duration
90 minutes

## Learning Objectives
By the end of this lab, you will:
- Create and configure VPC Interface and Gateway Endpoints
- Implement PrivateLink for cross-VPC service access
- Set up endpoint services for custom applications
- Configure DNS resolution for private endpoints
- Test hybrid connectivity through PrivateLink
- Implement security policies for private endpoints

## Lab Architecture
```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Consumer      │    │   AWS Services   │    │   Provider      │
│     VPC         │    │                  │    │     VPC         │
│                 │    │  ┌─────────────┐ │    │                 │
│  ┌───────────┐  │────┼──│  S3 Gateway │ │    │ ┌─────────────┐ │
│  │Interface  │  │    │  │  Endpoint   │ │    │ │   Custom    │ │
│  │Endpoints  │  │    │  └─────────────┘ │    │ │  Service    │ │
│  └───────────┘  │    │                  │    │ └─────────────┘ │
│                 │    │  ┌─────────────┐ │    │       │         │
│                 │────┼──│EC2, ECS, etc│ │    │ ┌─────────────┐ │
└─────────────────┘    │  │Interface    │ │    │ │  Endpoint   │ │
          │            │  │Endpoints    │ │    │ │  Service    │ │
          │            │  └─────────────┘ │    │ └─────────────┘ │
┌─────────────────┐    └──────────────────┘    └─────────────────┘
│  On-Premises    │
│   Network       │
└─────────────────┘
```

## Lab Steps

### Step 1: Create Multi-VPC Environment
1. **Create Consumer VPC**
   - Navigate to **VPC Console**
   - Create VPC:
     - Name: `PrivateLink-Consumer-VPC`
     - IPv4 CIDR: `10.100.0.0/16`
     - Enable DNS hostnames and resolution

2. **Create Provider VPC**
   - Create second VPC:
     - Name: `PrivateLink-Provider-VPC`
     - IPv4 CIDR: `10.200.0.0/16`
     - Enable DNS hostnames and resolution

3. **Create Subnets**
   - In Consumer VPC:
     - Private subnet: `PrivateLink-Consumer-Private` (10.100.1.0/24)
     - Public subnet: `PrivateLink-Consumer-Public` (10.100.10.0/24)
   - In Provider VPC:
     - Private subnet: `PrivateLink-Provider-Private` (10.200.1.0/24)
     - Public subnet: `PrivateLink-Provider-Public` (10.200.10.0/24)

4. **Configure Internet Gateways and Route Tables**
   - Attach IGWs to both VPCs
   - Configure routing for public subnets
   - Set up NAT Gateways for private subnet internet access

### Step 2: Create Gateway Endpoints
1. **Create S3 Gateway Endpoint**
   - In Consumer VPC, navigate to **Endpoints**
   - Click **Create Endpoint**
   - Configuration:
     - Name: `S3-Gateway-Endpoint`
     - Service category: AWS services
     - Service name: `com.amazonaws.us-east-1.s3`
     - VPC: PrivateLink-Consumer-VPC
     - Route tables: Select private subnet route table
     - Policy: Full access (for lab)

2. **Test S3 Gateway Endpoint**
   - Launch EC2 instance in private subnet
   - SSH via Session Manager
   - Test S3 access:
   ```bash
   # Should use VPC endpoint, not internet
   aws s3 ls
   
   # Check routes
   curl -s http://169.254.169.254/latest/meta-data/public-ipv4 || echo "No public IP"
   
   # Test S3 access without internet
   aws s3 mb s3://privatelink-test-bucket-$(date +%s)
   ```

3. **Create DynamoDB Gateway Endpoint**
   - Create another gateway endpoint:
     - Service name: `com.amazonaws.us-east-1.dynamodb`
     - Same configuration as S3 endpoint

### Step 3: Create Interface Endpoints for AWS Services
1. **Create EC2 Interface Endpoint**
   - Create endpoint:
     - Name: `EC2-Interface-Endpoint`
     - Service name: `com.amazonaws.us-east-1.ec2`
     - VPC: PrivateLink-Consumer-VPC
     - Subnets: Select private subnet
     - Security Groups: Create/select appropriate SG
     - DNS names: Enable private DNS names

2. **Configure Security Group for Interface Endpoints**
   - Create security group: `PrivateLink-Endpoints-SG`
   - Inbound rules:
     - HTTPS (443) from VPC CIDR
     - HTTP (80) from VPC CIDR (if needed)
   - Apply to all interface endpoints

3. **Create Additional Interface Endpoints**
   - ECS endpoint: `com.amazonaws.us-east-1.ecs`
   - CloudWatch endpoint: `com.amazonaws.us-east-1.monitoring`
   - CloudWatch Logs: `com.amazonaws.us-east-1.logs`
   - Systems Manager: `com.amazonaws.us-east-1.ssm`

### Step 4: Test AWS Service Connectivity
1. **Launch Test Instance**
   - Launch EC2 instance in Consumer VPC private subnet
   - Use Systems Manager Session Manager for access
   - Install AWS CLI if not present

2. **Test Interface Endpoint Connectivity**
   ```bash
   # Test EC2 API through interface endpoint
   aws ec2 describe-instances --region us-east-1
   
   # Test CloudWatch
   aws cloudwatch list-metrics --region us-east-1
   
   # Test Systems Manager
   aws ssm get-parameters-by-path --path "/" --region us-east-1
   ```

3. **Verify DNS Resolution**
   ```bash
   # Should resolve to private IP addresses
   nslookup ec2.us-east-1.amazonaws.com
   nslookup monitoring.us-east-1.amazonaws.com
   
   # Check specific endpoint DNS
   nslookup vpce-xxxxxx-yyyyyyy.ec2.us-east-1.vpce.amazonaws.com
   ```

### Step 5: Create Custom Service with Endpoint Service
1. **Deploy Application in Provider VPC**
   - Launch EC2 instance in Provider VPC
   - Install simple web application:
   ```bash
   #!/bin/bash
   yum update -y
   yum install -y httpd
   systemctl start httpd
   systemctl enable httpd
   
   cat > /var/www/html/index.html << EOF
   <html>
   <body>
   <h1>Private Service via PrivateLink</h1>
   <p>Server: $(hostname)</p>
   <p>Private IP: $(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)</p>
   <p>Time: $(date)</p>
   </body>
   </html>
   EOF
   ```

2. **Create Network Load Balancer**
   - Navigate to **EC2** → **Load Balancers**
   - Create Network Load Balancer:
     - Name: `PrivateLink-Provider-NLB`
     - Scheme: Internal
     - VPC: PrivateLink-Provider-VPC
     - Subnets: Select private subnet
     - Target group: HTTP:80, register your instance

3. **Create VPC Endpoint Service**
   - Navigate to **VPC** → **Endpoint Services**
   - Click **Create Endpoint Service**
   - Configuration:
     - Name: `Custom-Service-Endpoint`
     - Load balancer type: Network
     - Load balancer: Select your NLB
     - Acceptance required: No (for lab)
     - Supported principals: Add your AWS account ID

### Step 6: Create Interface Endpoint for Custom Service
1. **Create Interface Endpoint**
   - In Consumer VPC, create endpoint:
     - Service category: Find service by name
     - Service name: Copy from your endpoint service
     - VPC: PrivateLink-Consumer-VPC
     - Subnets: Select private subnet
     - Security Groups: Allow HTTP/HTTPS

2. **Configure DNS for Custom Service**
   - Note the endpoint DNS names
   - Configure private hosted zone if needed
   - Test DNS resolution

### Step 7: Test Cross-VPC Connectivity
1. **Test Custom Service Access**
   - From Consumer VPC instance:
   ```bash
   # Get endpoint DNS name from console
   ENDPOINT_DNS="vpce-xxxxxx-yyyyyyy.vpce-svc-zzzzzzzzz.us-east-1.vpce.amazonaws.com"
   
   # Test connectivity
   curl http://$ENDPOINT_DNS
   
   # Test multiple times to verify load balancing
   for i in {1..10}; do
     curl -s http://$ENDPOINT_DNS | grep "Server:"
   done
   ```

2. **Verify Network Path**
   ```bash
   # Traffic should stay within AWS network
   traceroute $ENDPOINT_DNS
   
   # Should see private IP addresses only
   ```

### Step 8: Implement Hybrid Connectivity
1. **Create On-Premises Simulation**
   - Create separate VPC: `OnPrem-Simulation-VPC` (192.168.0.0/16)
   - Launch instance to simulate on-premises server
   - Connect to Consumer VPC via Transit Gateway or VPC Peering

2. **Configure Transit Gateway (if using)**
   - Create Transit Gateway
   - Attach Consumer VPC and OnPrem simulation VPC
   - Configure route tables for connectivity
   - Test connectivity between VPCs

3. **Test Hybrid PrivateLink Access**
   - From simulated on-premises instance:
   ```bash
   # Should access AWS services via Consumer VPC endpoints
   # Configure routing to use Consumer VPC as gateway
   
   # Test S3 access through Consumer VPC
   aws s3 ls --region us-east-1
   
   # Test custom service access
   curl http://$ENDPOINT_DNS
   ```

### Step 9: Configure DNS Resolution
1. **Set Up Route 53 Resolver**
   - Create inbound resolver endpoint in Consumer VPC
   - Configure on-premises DNS to forward AWS queries
   - Test DNS resolution for endpoint services

2. **Configure Private Hosted Zone**
   - Create private hosted zone for custom domain
   - Add CNAME records pointing to endpoint DNS names
   - Test custom domain resolution

3. **Test DNS-based Service Discovery**
   ```bash
   # Create custom DNS names for services
   # Example: api.internal.company.com → endpoint DNS
   
   nslookup api.internal.company.com
   curl http://api.internal.company.com
   ```

### Step 10: Implement Security and Monitoring
1. **Configure Endpoint Policies**
   - Create restrictive endpoint policy:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": "*",
         "Action": [
           "s3:GetObject",
           "s3:PutObject"
         ],
         "Resource": [
           "arn:aws:s3:::my-secure-bucket/*"
         ]
       }
     ]
   }
   ```

2. **Configure Security Groups**
   - Restrict endpoint access to specific sources
   - Implement least-privilege access
   - Document security group rules

3. **Set Up CloudWatch Monitoring**
   - Monitor VPC endpoint metrics
   - Set up alarms for endpoint availability
   - Track data transfer through endpoints

4. **Enable VPC Flow Logs**
   - Enable flow logs for endpoint subnets
   - Monitor traffic patterns
   - Identify security issues

## Lab Validation
Complete these verification steps:

1. **Gateway Endpoints**
   - [ ] S3 gateway endpoint working
   - [ ] DynamoDB gateway endpoint working
   - [ ] Traffic routing through endpoints
   - [ ] No internet charges for S3/DynamoDB

2. **Interface Endpoints**
   - [ ] AWS service endpoints functional
   - [ ] Custom service endpoint working
   - [ ] DNS resolution correct
   - [ ] Security groups properly configured

3. **Hybrid Connectivity**
   - [ ] On-premises can access endpoints
   - [ ] DNS resolution working
   - [ ] Security policies effective
   - [ ] Monitoring configured

## Troubleshooting Common Issues

### DNS Resolution Problems
```bash
# Check DNS settings
cat /etc/resolv.conf

# Test specific endpoint DNS
dig vpce-xxxxxx.s3.us-east-1.vpce.amazonaws.com

# Verify private DNS names enabled
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids vpce-xxxxxx
```

### Connectivity Issues
```bash
# Check security group rules
aws ec2 describe-security-groups --group-ids sg-xxxxxx

# Test port connectivity
telnet endpoint-dns-name 443

# Check route tables
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-xxxxxx"
```

### Service Access Problems
```bash
# Check endpoint policy
aws ec2 describe-vpc-endpoints --vpc-endpoint-ids vpce-xxxxxx

# Test with different AWS CLI endpoints
aws s3 ls --endpoint-url https://vpce-xxxxxx.s3.us-east-1.vpce.amazonaws.com

# Verify IAM permissions
aws sts get-caller-identity
```

## Cost Optimization
1. **Endpoint Usage Analysis**
   - Monitor actual endpoint usage
   - Remove unused endpoints
   - Consolidate endpoints where possible

2. **Data Transfer Optimization**
   - Use gateway endpoints for S3/DynamoDB (no charge)
   - Monitor interface endpoint data transfer charges
   - Optimize application data access patterns

## Security Best Practices
1. **Least Privilege Access**
   - Use restrictive endpoint policies
   - Implement precise security group rules
   - Regular access reviews

2. **Network Segmentation**
   - Isolate endpoint subnets
   - Use NACLs for additional security
   - Monitor cross-VPC traffic

3. **Monitoring and Logging**
   - Enable comprehensive logging
   - Monitor for unusual access patterns
   - Set up security alerts

## Use Cases and Patterns

### Enterprise Hybrid Architecture
- **Use Case**: Enterprise with strict security requirements
- **Pattern**: All AWS service access through PrivateLink
- **Benefits**: No internet exposure, consistent security

### Multi-Account Service Sharing
- **Use Case**: Shared services across AWS accounts
- **Pattern**: Central service account with endpoint services
- **Benefits**: Secure cross-account access, cost efficiency

### SaaS Integration
- **Use Case**: SaaS provider offering AWS-native integration
- **Pattern**: PrivateLink-enabled service endpoints
- **Benefits**: Customer data stays private, reduced latency

## Clean Up
To avoid ongoing charges:

1. **Delete Endpoints**
   - Remove all VPC endpoints
   - Delete endpoint services
   - Remove associated security groups

2. **Clean Up Infrastructure**
   - Delete load balancers and target groups
   - Terminate EC2 instances
   - Delete VPCs and subnets
   - Remove Route 53 resources

3. **Clean Up Monitoring**
   - Delete CloudWatch dashboards
   - Remove custom metrics
   - Delete log groups

## Key Takeaways
- PrivateLink enables secure, private connectivity without internet exposure
- Gateway endpoints are free for S3 and DynamoDB
- Interface endpoints charge hourly plus data transfer
- Custom services can be exposed via endpoint services
- DNS configuration is crucial for seamless operation
- Security policies provide fine-grained access control
- Ideal for hybrid architectures requiring private connectivity

## Next Steps
- Proceed to Lab 24: Cloud WAN vs Transit Gateway
- Implement PrivateLink in production architectures
- Explore cross-account PrivateLink patterns
- Consider PrivateLink for SaaS integration
- Optimize costs based on usage patterns