# Lab 18: Network Load Balancer in Hybrid Architecture

## Lab Overview
This lab demonstrates how to implement AWS Network Load Balancer (NLB) as part of a hybrid architecture, enabling high-performance load balancing across AWS and on-premises resources. Students will configure NLB with various target types and implement advanced routing patterns.

## Prerequisites
- AWS Account with EC2 and ELB permissions
- Understanding of load balancing concepts
- Completed hybrid connectivity labs (VPN or Direct Connect)
- Basic knowledge of TCP/UDP protocols

## Lab Duration
90 minutes

## Learning Objectives
By the end of this lab, you will:
- Configure Network Load Balancer for hybrid workloads
- Implement cross-zone load balancing and target groups
- Set up health checks for hybrid targets
- Configure NLB with on-premises targets via IP
- Implement advanced routing and traffic distribution
- Monitor and optimize NLB performance

## Lab Architecture
```
                    Internet
                       │
                 ┌─────────────┐
                 │     NLB     │
                 │   (Public)  │
                 └─────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   ┌─────────┐    ┌─────────┐    ┌─────────┐
   │AWS      │    │AWS      │    │On-Prem  │
   │Target 1 │    │Target 2 │    │Target 3 │
   │(EC2)    │    │(EC2)    │    │(IP)     │
   └─────────┘    └─────────┘    └─────────┘
```

## Lab Steps

### Step 1: Create VPC and Hybrid Connectivity
1. **Create Lab VPC**
   - Navigate to **VPC Console**
   - Create VPC:
     - Name: `NLB-Hybrid-VPC`
     - IPv4 CIDR: `10.100.0.0/16`
     - Enable DNS hostnames and resolution

2. **Create Subnets**
   - Public Subnet AZ-A:
     - Name: `NLB-Public-Subnet-1A`
     - CIDR: `10.100.1.0/24`
     - AZ: us-east-1a
   - Public Subnet AZ-B:
     - Name: `NLB-Public-Subnet-1B`
     - CIDR: `10.100.2.0/24`
     - AZ: us-east-1b
   - Private Subnet AZ-A:
     - Name: `NLB-Private-Subnet-1A`
     - CIDR: `10.100.11.0/24`
     - AZ: us-east-1a
   - Private Subnet AZ-B:
     - Name: `NLB-Private-Subnet-1B`
     - CIDR: `10.100.12.0/24`
     - AZ: us-east-1b

3. **Configure Internet Gateway and NAT**
   - Attach Internet Gateway
   - Create NAT Gateways in public subnets
   - Configure route tables appropriately

### Step 2: Set Up Hybrid Connectivity (Simulation)
1. **Create On-Premises Simulation VPC**
   - Name: `OnPrem-Simulation-VPC`
   - CIDR: `192.168.0.0/16`
   - Create public subnet for internet access

2. **Establish VPC Peering (for lab simulation)**
   - Create VPC Peering between NLB VPC and OnPrem VPC
   - Accept peering connection
   - Update route tables for connectivity
   - In production, this would be VPN or Direct Connect

3. **Test Connectivity**
   - Launch test instances in both VPCs
   - Verify connectivity between VPCs
   - Document IP ranges for target configuration

### Step 3: Deploy Application Targets
1. **Launch AWS Targets**
   - Launch EC2 instances in private subnets:
   ```bash
   # User data for web server
   #!/bin/bash
   yum update -y
   yum install -y httpd
   systemctl start httpd
   systemctl enable httpd
   
   # Create unique content per instance
   INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
   AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
   PRIVATE_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
   
   cat > /var/www/html/index.html << EOF
   <html>
   <head><title>NLB Hybrid Lab</title></head>
   <body>
   <h1>AWS Target Server</h1>
   <p><strong>Instance ID:</strong> $INSTANCE_ID</p>
   <p><strong>Availability Zone:</strong> $AZ</p>
   <p><strong>Private IP:</strong> $PRIVATE_IP</p>
   <p><strong>Timestamp:</strong> $(date)</p>
   <p><strong>Target Type:</strong> AWS EC2 Instance</p>
   </body>
   </html>
   EOF
   
   # Configure health check endpoint
   cat > /var/www/html/health << EOF
   OK
   EOF
   ```

2. **Configure Security Groups for AWS Targets**
   - Create security group: `NLB-Targets-SG`
   - Inbound rules:
     - HTTP (80) from NLB subnets
     - HTTPS (443) from NLB subnets
     - Health check port from NLB
     - SSH (22) for management

3. **Launch On-Premises Simulation Target**
   - Launch instance in OnPrem simulation VPC:
   ```bash
   #!/bin/bash
   yum update -y
   yum install -y httpd
   systemctl start httpd
   systemctl enable httpd
   
   PRIVATE_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)
   
   cat > /var/www/html/index.html << EOF
   <html>
   <head><title>NLB Hybrid Lab</title></head>
   <body>
   <h1>On-Premises Target Server</h1>
   <p><strong>Private IP:</strong> $PRIVATE_IP</p>
   <p><strong>Location:</strong> Simulated On-Premises</p>
   <p><strong>Timestamp:</strong> $(date)</p>
   <p><strong>Target Type:</strong> On-Premises IP Target</p>
   </body>
   </html>
   EOF
   
   cat > /var/www/html/health << EOF
   OK
   EOF
   ```

### Step 4: Create Network Load Balancer
1. **Create NLB**
   - Navigate to **EC2** → **Load Balancers**
   - Click **Create Load Balancer**
   - Choose **Network Load Balancer**
   - Configuration:
     - Name: `Hybrid-Network-LoadBalancer`
     - Scheme: Internet-facing
     - IP address type: IPv4

2. **Configure Network Mapping**
   - VPC: NLB-Hybrid-VPC
   - Mappings:
     - us-east-1a: NLB-Public-Subnet-1A
     - us-east-1b: NLB-Public-Subnet-1B
   - For each AZ, choose "Use an Elastic IP address" if you want static IPs

3. **Create Initial Listener**
   - Protocol: TCP
   - Port: 80
   - Default action: Forward to target group (we'll create this next)

### Step 5: Create Target Groups
1. **Create AWS Instance Target Group**
   - Navigate to **EC2** → **Target Groups**
   - Click **Create target group**
   - Configuration:
     - Target type: Instances
     - Name: `AWS-Instances-TG`
     - Protocol: TCP
     - Port: 80
     - VPC: NLB-Hybrid-VPC
     - Health check protocol: HTTP
     - Health check path: `/health`

2. **Create IP Target Group for On-Premises**
   - Create another target group:
     - Target type: IP addresses
     - Name: `OnPrem-IP-TG`
     - Protocol: TCP
     - Port: 80
     - VPC: NLB-Hybrid-VPC
     - Health check protocol: HTTP
     - Health check path: `/health`

3. **Configure Health Check Settings**
   - For both target groups:
     - Health check interval: 30 seconds
     - Healthy threshold: 2
     - Unhealthy threshold: 2
     - Timeout: 10 seconds

### Step 6: Register Targets
1. **Register AWS EC2 Instances**
   - Select `AWS-Instances-TG`
   - Click **Actions** → **Register targets**
   - Select your EC2 instances in private subnets
   - Port: 80
   - Click **Register targets**

2. **Register On-Premises IP Target**
   - Select `OnPrem-IP-TG`
   - Click **Actions** → **Register targets**
   - Enter IP address of on-premises simulation instance
   - Port: 80
   - Click **Register targets**

3. **Verify Target Health**
   - Monitor target health status
   - Troubleshoot any unhealthy targets
   - Ensure all targets show "healthy" status

### Step 7: Configure Advanced Load Balancer Features
1. **Enable Cross-Zone Load Balancing**
   - Select your NLB
   - Click **Actions** → **Edit attributes**
   - Enable cross-zone load balancing
   - Note: This incurs additional charges but improves distribution

2. **Configure Connection Draining**
   - Set deregistration delay: 300 seconds
   - This allows in-flight requests to complete

3. **Enable Access Logs** (Optional)
   - Create S3 bucket for access logs
   - Enable access logging in NLB attributes
   - Configure log file prefix

### Step 8: Create Weighted Target Groups
1. **Create Combined Target Group**
   - Navigate to **Load Balancers**
   - Select your NLB
   - Click **Listeners** tab
   - Edit the listener

2. **Configure Weighted Routing**
   - Edit default action
   - Choose "Forward to multiple target groups"
   - Add target groups with weights:
     - AWS-Instances-TG: Weight 70
     - OnPrem-IP-TG: Weight 30
   - This sends 70% traffic to AWS, 30% to on-premises

### Step 9: Test Load Balancer Functionality
1. **Get NLB DNS Name**
   - Copy the DNS name from NLB details
   - Example: `hybrid-network-loadbalancer-1234567890.elb.us-east-1.amazonaws.com`

2. **Test Basic Connectivity**
   ```bash
   # Test HTTP connectivity
   curl http://your-nlb-dns-name
   
   # Test multiple times to see load distribution
   for i in {1..20}; do
     curl -s http://your-nlb-dns-name | grep "Target Type"
     sleep 1
   done
   ```

3. **Test Weighted Distribution**
   - Run multiple requests
   - Verify traffic distribution matches weights (70/30)
   - Document which targets are receiving traffic

### Step 10: Implement Health Check Monitoring
1. **Monitor Target Health**
   ```bash
   # CLI command to check target health
   aws elbv2 describe-target-health \
     --target-group-arn arn:aws:elasticloadbalancing:region:account:targetgroup/AWS-Instances-TG/1234567890123456
   
   aws elbv2 describe-target-health \
     --target-group-arn arn:aws:elasticloadbalancing:region:account:targetgroup/OnPrem-IP-TG/1234567890123456
   ```

2. **Set Up CloudWatch Monitoring**
   - Navigate to **CloudWatch** → **Metrics**
   - Browse NetworkELB metrics:
     - ActiveFlowCount
     - NewFlowCount
     - ProcessedBytes
     - TargetResponseTime
     - HealthyHostCount
     - UnHealthyHostCount

3. **Create CloudWatch Dashboard**
   ```json
   {
     "widgets": [
       {
         "type": "metric",
         "properties": {
           "metrics": [
             ["AWS/NetworkELB", "ActiveFlowCount", "LoadBalancer", "net/your-nlb-name"],
             [".", "NewFlowCount", ".", "."],
             [".", "ProcessedBytes", ".", "."]
           ],
           "period": 300,
           "stat": "Sum",
           "region": "us-east-1",
           "title": "NLB Traffic Metrics"
         }
       }
     ]
   }
   ```

### Step 11: Advanced Configuration
1. **Configure SSL/TLS Termination**
   - Add HTTPS listener (port 443)
   - Upload or request SSL certificate
   - Configure security policy
   - Test HTTPS connectivity

2. **Implement Sticky Sessions** (if needed)
   - Configure target group stickiness
   - Set stickiness duration
   - Test session persistence

3. **Configure Connection Idle Timeout**
   - Adjust idle timeout settings
   - Consider application requirements
   - Test with long-running connections

### Step 12: Performance Testing
1. **Load Testing Setup**
   ```bash
   # Install Apache Bench for load testing
   sudo yum install -y httpd-tools
   
   # Run performance test
   ab -n 10000 -c 100 http://your-nlb-dns-name/
   
   # Test sustained load
   ab -n 50000 -c 200 -t 300 http://your-nlb-dns-name/
   ```

2. **Monitor Performance Metrics**
   - Watch CloudWatch metrics during testing
   - Monitor target response times
   - Check for any error rates

3. **Optimize Based on Results**
   - Adjust target group health check settings
   - Modify instance types if needed
   - Consider adding more targets

## Lab Validation
Complete these verification steps:

1. **NLB Configuration**
   - [ ] NLB created and active
   - [ ] Multiple target groups configured
   - [ ] Listeners properly configured
   - [ ] Health checks passing

2. **Target Registration**
   - [ ] AWS EC2 targets registered and healthy
   - [ ] On-premises IP targets registered and healthy
   - [ ] Cross-zone load balancing enabled
   - [ ] Weighted routing configured

3. **Functionality Testing**
   - [ ] HTTP traffic distributed correctly
   - [ ] Health checks working properly
   - [ ] Failover working when targets fail
   - [ ] Performance metrics available

## Advanced Use Cases

### Blue/Green Deployments
```bash
# Create blue and green target groups
# Route traffic gradually from blue to green
# Monitor application metrics during transition
```

### Multi-Protocol Load Balancing
```bash
# Configure multiple listeners
# HTTP on port 80
# HTTPS on port 443  
# Custom TCP protocols on other ports
```

### Geographic Routing
```bash
# Use Route 53 with health checks
# Route traffic based on client location
# Failover between regions
```

## Troubleshooting Common Issues

### Targets Showing Unhealthy
```bash
# Check security groups allow health check traffic
# Verify health check path returns 200 OK
# Check network connectivity to targets
# Review health check logs

aws elbv2 describe-target-health --target-group-arn <arn>
```

### Uneven Traffic Distribution
```bash
# Verify cross-zone load balancing is enabled
# Check target weights are configured correctly
# Monitor connection draining settings
# Review client connection patterns
```

### Performance Issues
```bash
# Monitor target response times
# Check for network bottlenecks
# Review instance capacity
# Consider connection limits
```

### On-Premises Connectivity Issues
```bash
# Verify hybrid connectivity (VPN/DX)
# Check routing tables
# Test network path with traceroute
# Verify security groups and NACLs
```

## Cost Optimization
1. **NLB Pricing Considerations**
   - Hourly charges per AZ
   - Data processing charges
   - Cross-zone load balancing charges

2. **Optimization Strategies**
   - Right-size target instances
   - Monitor actual traffic patterns
   - Consider reserved capacity
   - Optimize health check frequency

## Security Best Practices
1. **Network Security**
   - Use security groups to restrict access
   - Implement WAF for HTTP/HTTPS traffic
   - Use VPC Flow Logs for monitoring
   - Enable access logging

2. **SSL/TLS Configuration**
   - Use strong cipher suites
   - Implement perfect forward secrecy
   - Regular certificate rotation
   - Monitor SSL/TLS metrics

## Clean Up
To avoid ongoing charges:

1. **Delete Load Balancer Resources**
   - Delete Network Load Balancer
   - Delete target groups
   - Release Elastic IP addresses (if used)

2. **Clean Up Infrastructure**
   - Terminate EC2 instances
   - Delete VPC peering connections
   - Delete VPCs and subnets
   - Remove security groups

3. **Clean Up Monitoring**
   - Delete CloudWatch dashboards
   - Remove custom metrics
   - Delete S3 bucket for access logs

## Key Takeaways
- NLB provides high-performance Layer 4 load balancing
- Supports both AWS and on-premises targets via IP addressing
- Cross-zone load balancing improves traffic distribution
- Health checks ensure traffic only goes to healthy targets
- Weighted routing enables gradual traffic migration
- Integration with hybrid connectivity enables true hybrid architectures
- Performance characteristics differ significantly from ALB

## Next Steps
- Proceed to Lab 19: Application Load Balancer Deep Dive
- Explore Global Load Balancer with Route 53
- Implement production hybrid load balancing
- Consider Gateway Load Balancer for network appliances
- Integrate with AWS Global Accelerator for global performance