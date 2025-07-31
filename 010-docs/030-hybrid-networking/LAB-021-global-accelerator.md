# Lab 21: AWS Global Accelerator for Hybrid Applications

## Lab Overview
This lab demonstrates how to implement AWS Global Accelerator to improve performance and availability for hybrid applications that span on-premises and AWS infrastructure. Students will configure Global Accelerator with various endpoint types and test performance improvements.

## Prerequisites
- AWS Account with Global Accelerator permissions
- Understanding of hybrid networking concepts
- Completed previous hybrid networking labs
- Basic knowledge of anycast networking

## Lab Duration
75 minutes

## Learning Objectives
By the end of this lab, you will:
- Configure AWS Global Accelerator for hybrid applications
- Set up multiple endpoint types (EC2, ALB, on-premises)
- Implement traffic routing and health checks
- Measure performance improvements with Global Accelerator
- Configure endpoint weights and failover
- Monitor and optimize Global Accelerator performance

## Lab Architecture
```
Global Users → AWS Edge Locations → Global Accelerator → Multiple Endpoints
     │              │                      │                 │
   [US East]    [London Edge]         [Anycast IPs]    ┌─AWS VPC─┐
   [Asia]       [Tokyo Edge]           [Static IPs]     │ ALB/EC2 │
   [Europe]     [Sydney Edge]         [Health Checks]   └─────────┘
                                                              │
                                                      ┌─On-Premises─┐
                                                      │   Servers    │
                                                      └──────────────┘
```

## Lab Steps

### Step 1: Prepare Multi-Region Infrastructure
1. **Create Primary VPC (US East)**
   - Navigate to **VPC Console**
   - Select **US East (N. Virginia)** region
   - Create VPC:
     - Name: `GlobalAccel-Primary-VPC`
     - IPv4 CIDR: `10.100.0.0/16`

2. **Create Secondary VPC (US West)**
   - Switch to **US West (Oregon)** region
   - Create VPC:
     - Name: `GlobalAccel-Secondary-VPC`
     - IPv4 CIDR: `10.200.0.0/16`

3. **Create Application Subnets**
   - In each VPC, create:
     - Public subnet for ALB: `/24` subnet
     - Private subnet for instances: `/24` subnet
     - Configure Internet Gateways and route tables

### Step 2: Deploy Test Applications
1. **Launch Application in US East**
   - Create Application Load Balancer:
     - Name: `GlobalAccel-Primary-ALB`
     - Scheme: Internet-facing
     - Subnets: Select public subnets
   - Launch EC2 instances:
     - Count: 2 instances
     - Instance type: t3.micro
     - User data:
   ```bash
   #!/bin/bash
   yum update -y
   yum install -y httpd
   systemctl start httpd
   systemctl enable httpd
   
   # Create unique content per region
   cat > /var/www/html/index.html << EOF
   <html>
   <head><title>Global Accelerator Lab</title></head>
   <body>
   <h1>US East Region</h1>
   <p>Server: $(hostname)</p>
   <p>Region: us-east-1</p>
   <p>Time: $(date)</p>
   <p>Client IP: \$remote_addr</p>
   </body>
   </html>
   EOF
   ```

2. **Launch Application in US West**
   - Repeat similar setup in US West:
     - ALB: `GlobalAccel-Secondary-ALB`
     - Update HTML content to show "US West Region"
     - Use same instance configuration

3. **Configure Target Groups**
   - Create target groups for each ALB
   - Register EC2 instances
   - Configure health checks:
     - Path: `/`
     - Healthy threshold: 2
     - Unhealthy threshold: 3
     - Interval: 30 seconds

### Step 3: Create Global Accelerator
1. **Create Accelerator**
   - Navigate to **Global Accelerator Console**
   - Click **Create accelerator**
   - Configuration:
     - Name: `Lab-Global-Accelerator`
     - IP address type: IPv4
     - Enabled: Yes
   - Click **Create accelerator**

2. **Note Static IP Addresses**
   - Record the two static IP addresses assigned
   - These will be your anycast IPs for global access
   - Example: `75.2.60.5` and `99.83.89.7`

### Step 4: Configure Listeners
1. **Create HTTP Listener**
   - In your accelerator, click **Add listener**
   - Protocol: TCP
   - Port: 80
   - Client affinity: None (for testing)
   - Click **Add listener**

2. **Create HTTPS Listener (Optional)**
   - Add another listener:
   - Protocol: TCP
   - Port: 443
   - Client affinity: Source IP (if session persistence needed)

### Step 5: Set Up Endpoint Groups
1. **Create US East Endpoint Group**
   - Select your HTTP listener
   - Click **Add endpoint group**
   - Configuration:
     - Region: US East (N. Virginia)
     - Traffic dial: 100%
     - Health check grace period: 30 seconds
   - Click **Add endpoint group**

2. **Create US West Endpoint Group**
   - Add another endpoint group:
   - Region: US West (Oregon)
   - Traffic dial: 100%
   - Health check grace period: 30 seconds

### Step 6: Add Endpoints
1. **Add ALB Endpoints**
   - In US East endpoint group:
     - Click **Add endpoint**
     - Type: Application Load Balancer
     - Endpoint: Select `GlobalAccel-Primary-ALB`
     - Weight: 128 (default)
   
   - In US West endpoint group:
     - Add `GlobalAccel-Secondary-ALB`
     - Weight: 128

2. **Add EC2 Instance Endpoints**
   - Also add direct EC2 instance endpoints for testing:
   - Select specific instances from each region
   - Configure weights for load distribution

### Step 7: Configure On-Premises Endpoint (Simulation)
1. **Create On-Premises Simulation**
   - Launch EC2 instance with Elastic IP
   - Configure as on-premises server simulation:
   ```bash
   #!/bin/bash
   yum update -y
   yum install -y httpd
   systemctl start httpd
   systemctl enable httpd
   
   cat > /var/www/html/index.html << EOF
   <html>
   <body>
   <h1>On-Premises Data Center</h1>
   <p>Server: $(hostname)</p>
   <p>Location: Simulated On-Premises</p>
   <p>Time: $(date)</p>
   </body>
   </html>
   EOF
   ```

2. **Add as Global Accelerator Endpoint**
   - In one of your endpoint groups:
   - Add endpoint type: IP address
   - IP address: Elastic IP of simulation instance
   - Weight: 64 (lower weight for testing)

### Step 8: Performance Testing Setup
1. **Create Test Scripts**
   - Create performance testing script:
   ```bash
   #!/bin/bash
   # global_accelerator_test.sh
   
   echo "Testing without Global Accelerator..."
   # Direct ALB access
   curl -w "@curl-format.txt" -s -o /dev/null http://primary-alb-dns-name/
   curl -w "@curl-format.txt" -s -o /dev/null http://secondary-alb-dns-name/
   
   echo "Testing with Global Accelerator..."
   # Global Accelerator access  
   curl -w "@curl-format.txt" -s -o /dev/null http://75.2.60.5/
   curl -w "@curl-format.txt" -s -o /dev/null http://99.83.89.7/
   ```

2. **Create curl format file**:
   ```
   # curl-format.txt
   time_namelookup:  %{time_namelookup}\n
   time_connect:     %{time_connect}\n
   time_appconnect:  %{time_appconnect}\n
   time_pretransfer: %{time_pretransfer}\n
   time_redirect:    %{time_redirect}\n
   time_starttransfer: %{time_starttransfer}\n
   time_total:       %{time_total}\n
   ```

### Step 9: Test from Multiple Locations
1. **Test from Different AWS Regions**
   - Launch small instances in different regions:
     - EU (Ireland)
     - Asia Pacific (Tokyo)
     - South America (São Paulo)
   
2. **Run Performance Tests**
   ```bash
   # From each test instance
   for i in {1..10}; do
     echo "Test $i:"
     curl -w "Total time: %{time_total}s\n" -s -o /dev/null http://your-global-accelerator-ip/
     sleep 2
   done
   ```

3. **Compare Performance**
   - Test direct ALB access vs Global Accelerator
   - Measure latency improvements
   - Document performance gains by region

### Step 10: Configure Traffic Management
1. **Set Up Weighted Routing**
   - Adjust endpoint weights:
     - US East ALB: Weight 100 (primary)
     - US West ALB: Weight 50 (secondary)
     - On-premises: Weight 25 (backup)

2. **Test Traffic Distribution**
   ```bash
   # Test traffic distribution
   for i in {1..20}; do
     curl -s http://your-global-accelerator-ip/ | grep Region
   done
   ```

3. **Configure Failover**
   - Set traffic dial to 0% for one region
   - Test automatic failover behavior
   - Monitor health check status

### Step 11: Implement Health Checks and Monitoring
1. **Configure Custom Health Checks**
   - Set up health check paths: `/health`
   - Configure health check intervals and thresholds
   - Test health check behavior

2. **Set Up CloudWatch Monitoring**
   - Monitor Global Accelerator metrics:
     - NewFlowCount
     - ProcessedBytesIn/Out
     - OriginLatency
   - Create CloudWatch dashboard

3. **Configure Alarms**
   - Set up alarms for:
     - Endpoint health changes
     - High latency
     - Traffic drops
   - Configure SNS notifications

### Step 12: Advanced Configuration
1. **Client Affinity Testing**
   - Configure source IP affinity
   - Test session persistence
   - Monitor client routing behavior

2. **Cross-Zone Load Balancing**
   - Enable cross-zone load balancing
   - Test traffic distribution
   - Monitor performance impact

3. **Custom Routing Accelerator** (Optional)
   - Create custom routing accelerator
   - Configure subnet endpoints
   - Test direct VPC access

## Lab Validation
Complete these verification steps:

1. **Global Accelerator Setup**
   - [ ] Accelerator created with static IPs
   - [ ] Listeners configured correctly
   - [ ] Endpoint groups active
   - [ ] All endpoints healthy

2. **Performance Improvements**
   - [ ] Latency measurements completed
   - [ ] Performance improvements documented
   - [ ] Traffic routing working correctly
   - [ ] Failover mechanisms tested

3. **Monitoring and Management**
   - [ ] CloudWatch metrics available
   - [ ] Health checks functioning
   - [ ] Alarms configured
   - [ ] Traffic management working

## Performance Testing Results Template

### Latency Comparison (Sample)
```
Test Location      | Direct ALB | Global Accelerator | Improvement
US East           | 45ms       | 12ms              | 73% better
US West           | 150ms      | 28ms              | 81% better  
EU Ireland        | 180ms      | 45ms              | 75% better
Asia Tokyo        | 250ms      | 68ms              | 73% better
```

### Availability Testing
```
Scenario                    | Direct ALB | Global Accelerator
Single region failure      | 100% down  | <1s failover
Network path issues        | Variable   | Automatic routing
DDoS attack mitigation     | Limited    | AWS Shield Standard
```

## Troubleshooting Common Issues

### Endpoint Health Issues
- Check security groups allow Global Accelerator traffic
- Verify health check path returns 200 OK
- Review target group health in ALB
- Confirm DNS resolution for endpoints

### Performance Not Improving
- Verify traffic is going through Global Accelerator IPs
- Check endpoint weights and traffic dial settings
- Review client location vs edge location mapping
- Monitor for asymmetric routing issues

### Traffic Distribution Problems
- Verify endpoint weights configuration
- Check client affinity settings
- Review health check status
- Monitor traffic dial percentages

## Cost Optimization
1. **Right-size Endpoints**
   - Monitor actual traffic patterns
   - Adjust endpoint weights based on capacity
   - Use appropriate instance types

2. **Traffic Management**
   - Use traffic dial to control data transfer costs
   - Route traffic to most cost-effective regions
   - Implement smart failover strategies

## Use Cases and Best Practices

### Ideal Use Cases
- **Global Applications**: Applications with worldwide user base
- **Gaming**: Low-latency gaming applications
- **Live Streaming**: Real-time content delivery
- **IoT**: Distributed IoT data collection
- **Financial Services**: Low-latency trading applications

### Best Practices
1. **Endpoint Diversity**: Use multiple regions and endpoint types
2. **Health Monitoring**: Implement comprehensive health checks
3. **Traffic Management**: Use weights and traffic dial effectively
4. **Security**: Combine with AWS WAF and Shield
5. **Testing**: Regular performance testing from various locations

## Clean Up
To avoid ongoing charges:

1. **Delete Global Accelerator**
   - Remove all endpoints
   - Delete endpoint groups
   - Delete listeners
   - Delete accelerator

2. **Clean Up Infrastructure**
   - Delete ALBs and target groups
   - Terminate EC2 instances
   - Delete VPCs in all regions
   - Remove CloudWatch dashboards

## Key Takeaways
- Global Accelerator improves performance through AWS global network
- Static anycast IPs simplify DNS management
- Health checks and failover provide high availability
- Performance improvements vary by client location
- Cost scales with data transfer and number of endpoints
- Ideal for applications requiring global reach and low latency

## Next Steps
- Proceed to Lab 23: PrivateLink and VPC Endpoints
- Implement Global Accelerator in production applications
- Explore integration with AWS WAF and Shield
- Consider custom routing for specialized use cases
- Monitor and optimize based on usage patterns