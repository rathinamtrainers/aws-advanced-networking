# Lab 22: Route 53 Resolver - Hybrid DNS Configuration

## Lab Overview
This lab demonstrates how to configure Route 53 Resolver for seamless DNS resolution between AWS VPC and on-premises networks. Students will set up inbound and outbound resolver endpoints to enable hybrid DNS functionality.

## Prerequisites
- AWS Account with Route 53 and VPC permissions
- Understanding of DNS concepts and zones
- Basic knowledge of hybrid networking
- Completed Transit Gateway lab (recommended)

## Lab Duration
90 minutes

## Learning Objectives
By the end of this lab, you will:
- Configure Route 53 Resolver endpoints (inbound/outbound)
- Set up DNS forwarding rules and conditional forwarding
- Integrate with on-premises DNS infrastructure
- Test DNS resolution across hybrid environments
- Implement DNS security and monitoring
- Troubleshoot common DNS resolution issues

## Lab Architecture
```
On-Premises DNS ←→ Outbound Endpoint ←→ Route 53 Resolver ←→ Inbound Endpoint ←→ AWS VPC
      |                                                                            |
   Internal Zones                                                            Private Hosted Zones
```

## Lab Steps

### Step 1: Prepare the Environment
1. **Create Lab VPC**
   - Navigate to **VPC Console**
   - Create VPC:
     - Name: `Route53Resolver-Lab-VPC`
     - IPv4 CIDR: `10.100.0.0/16`
     - Enable DNS hostnames: Yes
     - Enable DNS resolution: Yes

2. **Create Subnets**
   - Create Private Subnets for resolver endpoints:
     - Subnet 1: `R53-Resolver-Subnet-1A`
       - CIDR: `10.100.1.0/24`
       - AZ: us-east-1a
     - Subnet 2: `R53-Resolver-Subnet-1B`
       - CIDR: `10.100.2.0/24`
       - AZ: us-east-1b
   - Create Application Subnets:
     - App Private: `10.100.10.0/24` (us-east-1a)
     - App Public: `10.100.20.0/24` (us-east-1a)

3. **Create Security Groups**
   - Resolver Security Group:
     - Name: `Route53-Resolver-SG`
     - Allow inbound DNS (TCP/UDP 53) from VPC CIDR
     - Allow outbound DNS (TCP/UDP 53) to anywhere
   - Application Security Group:
     - Name: `App-Instance-SG`
     - Allow SSH and ICMP
     - Allow DNS to resolver security group

### Step 2: Create On-Premises Simulation
1. **Create Simulation VPC**
   - Name: `OnPrem-Simulation-VPC`
   - CIDR: `192.168.0.0/16`
   - Create public subnet: `192.168.1.0/24`
   - Attach Internet Gateway

2. **Launch DNS Server Instance**
   - Launch EC2 instance in simulation VPC:
     - Name: `OnPrem-DNS-Server`
     - AMI: Amazon Linux 2
     - Instance Type: t3.small
     - Security Group: Allow DNS (53), SSH (22)
     - User Data:
   ```bash
   #!/bin/bash
   yum update -y
   yum install -y bind bind-utils
   systemctl enable named
   systemctl start named
   ```

3. **Configure BIND DNS Server**
   - SSH to the DNS server instance
   - Configure BIND for local zone:
   ```bash
   # Edit /etc/named.conf
   sudo vi /etc/named.conf
   
   # Add zone configuration
   zone "onprem.local" IN {
       type master;
       file "onprem.local.zone";
       allow-update { none; };
   };
   ```

   - Create zone file:
   ```bash
   # Create /var/named/onprem.local.zone
   sudo vi /var/named/onprem.local.zone
   
   $TTL 86400
   @   IN  SOA dns.onprem.local. admin.onprem.local. (
           2023010101  ; Serial
           3600        ; Refresh
           1800        ; Retry
           604800      ; Expire
           86400       ; Minimum TTL
   )
   @   IN  NS  dns.onprem.local.
   dns IN  A   192.168.1.10
   app IN  A   192.168.1.20
   db  IN  A   192.168.1.30
   ```

   - Restart BIND:
   ```bash
   sudo systemctl restart named
   sudo systemctl status named
   ```

### Step 3: Create Route 53 Private Hosted Zone
1. **Create Private Hosted Zone**
   - Navigate to **Route 53 Console**
   - Click **Hosted zones** → **Create hosted zone**
   - Configuration:
     - Domain name: `aws.local`
     - Type: Private hosted zone
     - VPCs: Select your lab VPC and region

2. **Add DNS Records**
   - Create A records in the hosted zone:
   ```
   api.aws.local    → 10.100.10.10
   web.aws.local    → 10.100.10.20
   db.aws.local     → 10.100.10.30
   vault.aws.local  → 10.100.10.40
   ```

3. **Test Local Resolution**
   - Launch test instance in AWS VPC
   - Test DNS resolution:
   ```bash
   nslookup api.aws.local
   dig web.aws.local
   ```

### Step 4: Create Inbound Resolver Endpoint
1. **Create Inbound Endpoint**
   - Navigate to **Route 53 Console** → **Resolver** → **Inbound endpoints**
   - Click **Create inbound endpoint**
   - Configuration:
     - Name: `Lab-Inbound-Endpoint`
     - VPC: Route53Resolver-Lab-VPC
     - Security group: Route53-Resolver-SG
     - IP addresses:
       - us-east-1a: Select subnet 1A, choose IP or auto-assign
       - us-east-1b: Select subnet 1B, choose IP or auto-assign

2. **Record Endpoint IPs**
   - Note the IP addresses assigned to the endpoint
   - These will be used for on-premises DNS forwarding
   - Example: 10.100.1.100, 10.100.2.100

3. **Configure On-Premises Forwarding**
   - SSH to on-premises DNS server
   - Configure conditional forwarding in BIND:
   ```bash
   # Edit /etc/named.conf
   zone "aws.local" IN {
       type forward;
       forward only;
       forwarders { 10.100.1.100; 10.100.2.100; };
   };
   ```
   
   - Restart BIND and test:
   ```bash
   sudo systemctl restart named
   nslookup api.aws.local
   ```

### Step 5: Create Outbound Resolver Endpoint
1. **Create Outbound Endpoint**
   - Navigate to **Resolver** → **Outbound endpoints**
   - Click **Create outbound endpoint**
   - Configuration:
     - Name: `Lab-Outbound-Endpoint`
     - VPC: Route53Resolver-Lab-VPC
     - Security group: Route53-Resolver-SG
     - IP addresses:
       - us-east-1a: Select subnet 1A
       - us-east-1b: Select subnet 1B

2. **Create Resolver Rules**
   - Navigate to **Resolver** → **Rules**
   - Click **Create rule**
   - Configuration:
     - Name: `OnPrem-DNS-Rule`
     - Rule type: Forward
     - Domain name: `onprem.local`
     - VPCs: Select your lab VPC
     - Outbound endpoint: Select your outbound endpoint
     - Target IP addresses: IP of on-premises DNS server (192.168.1.x)

3. **Associate Rule with VPC**
   - The rule should automatically associate with your VPC
   - Verify association is active
   - Wait for rule to show "Complete" status

### Step 6: Test Hybrid DNS Resolution
1. **Test from AWS to On-Premises**
   - SSH to test instance in AWS VPC
   - Test resolution of on-premises domains:
   ```bash
   nslookup dns.onprem.local
   nslookup app.onprem.local
   dig +short db.onprem.local
   ```

2. **Test from On-Premises to AWS**
   - SSH to on-premises DNS server or client
   - Test resolution of AWS domains:
   ```bash
   nslookup api.aws.local
   nslookup web.aws.local
   ```

3. **Verify Bi-directional Resolution**
   - Confirm both directions work
   - Test with different record types (A, CNAME, TXT)
   - Document any resolution failures

### Step 7: Configure Advanced DNS Features
1. **DNS Logging**
   - Enable Route 53 Resolver query logging:
     - Navigate to **Resolver** → **Query logging**
     - Click **Configure query logging**
     - Destination: CloudWatch Logs
     - Log group: `/aws/route53resolver/querylog`

2. **DNS Firewall** (Optional)
   - Navigate to **Resolver** → **DNS Firewall**
   - Create rule group for malicious domain blocking
   - Associate with your VPC
   - Test blocked domain resolution

3. **DNSSEC Validation**
   - Enable DNSSEC validation:
     - Navigate to **Resolver** → **DNSSEC validation**
     - Enable for your VPC
     - Test with DNSSEC-enabled domains

### Step 8: Implement DNS Security
1. **Security Group Hardening**
   - Review resolver security group rules
   - Restrict DNS access to known sources only
   - Implement source-based access control

2. **VPC Flow Logs for DNS**
   - Enable VPC Flow Logs
   - Filter for DNS traffic (port 53)
   - Monitor unusual DNS queries

3. **CloudTrail for DNS Changes**
   - Ensure CloudTrail captures Route 53 API calls
   - Monitor resolver rule changes
   - Set up alerts for unauthorized modifications

### Step 9: Performance Optimization
1. **Caching Configuration**
   - Configure DNS client caching on instances
   - Optimize TTL values for different record types
   - Monitor cache hit ratios

2. **Endpoint Placement**
   - Verify resolver endpoints are in optimal subnets
   - Consider multiple endpoints for high availability
   - Monitor latency metrics

3. **DNS Response Time Monitoring**
   - Set up CloudWatch metrics for DNS resolution time
   - Create alarms for slow DNS responses
   - Implement health checks for critical domains

### Step 10: Troubleshooting and Monitoring
1. **Query Log Analysis**
   - Review CloudWatch logs for DNS queries
   - Identify most frequently queried domains
   - Spot unusual query patterns

2. **Resolution Path Tracing**
   - Use dig with trace option:
   ```bash
   dig +trace api.aws.local
   dig +trace app.onprem.local
   ```
   - Document resolution paths
   - Identify potential bottlenecks

3. **Health Check Implementation**
   - Create Route 53 health checks for critical services
   - Configure DNS failover based on health
   - Test failover scenarios

## Lab Validation
Complete these verification steps:

1. **Resolver Endpoint Status**
   - [ ] Inbound endpoint shows "Operational"
   - [ ] Outbound endpoint shows "Operational"
   - [ ] Resolver rules show "Complete"
   - [ ] VPC associations are active

2. **DNS Resolution Tests**
   - [ ] AWS instances can resolve on-premises domains
   - [ ] On-premises can resolve AWS private domains
   - [ ] Public DNS resolution still works
   - [ ] DNS logging captures queries

3. **Security and Performance**
   - [ ] Security groups properly restrict access
   - [ ] DNS response times are acceptable
   - [ ] DNSSEC validation works (if enabled)
   - [ ] Query logs show expected traffic

## Troubleshooting Common Issues

### Resolution Failures
- Check security group rules (TCP/UDP 53)
- Verify resolver rule domain names
- Confirm target IP addresses are correct
- Test with dig +short for detailed output

### Slow DNS Resolution
- Monitor resolver endpoint metrics
- Check network latency to target DNS servers
- Review DNS server performance on-premises
- Consider caching optimizations

### Intermittent Issues
- Check on-premises DNS server health
- Monitor resolver endpoint availability
- Review VPC network connectivity
- Verify resolver rule associations

### Configuration Errors
- Validate resolver rule syntax
- Check VPC associations
- Verify security group rules
- Review CloudTrail for recent changes

## Cost Optimization
1. **Endpoint Sizing**
   - Monitor actual query volume
   - Right-size number of endpoints
   - Consider regional placement

2. **Query Logging Costs**
   - Monitor log volume and retention
   - Filter logs for specific domains if needed
   - Optimize log retention policies

## Clean Up
To avoid ongoing charges:

1. **Delete Resolver Components**
   - Disassociate and delete resolver rules
   - Delete outbound endpoint
   - Delete inbound endpoint

2. **Clean Up DNS Resources**
   - Delete private hosted zone
   - Disable query logging
   - Remove DNS firewall rules (if created)

3. **Remove Test Infrastructure**
   - Terminate EC2 instances
   - Delete VPCs and subnets
   - Remove CloudWatch logs and dashboards

## Key Takeaways
- Route 53 Resolver enables seamless hybrid DNS
- Inbound endpoints allow on-premises to query AWS DNS
- Outbound endpoints enable AWS to query on-premises DNS
- Proper security groups are critical for DNS traffic
- Monitoring and logging help troubleshoot DNS issues
- Performance optimization requires careful planning

## Next Steps
- Proceed to Lab 23: PrivateLink and VPC Endpoints
- Explore Route 53 Application Recovery Controller
- Implement DNS-based disaster recovery
- Consider Route 53 Profiles for multi-account DNS management