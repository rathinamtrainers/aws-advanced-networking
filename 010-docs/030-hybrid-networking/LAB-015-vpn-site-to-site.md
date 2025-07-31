# Lab 15: VPN on AWS - Site-to-Site VPN Configuration

## Lab Overview
This comprehensive lab demonstrates how to establish secure Site-to-Site VPN connectivity between an on-premises network and AWS VPC using AWS VPN Gateway and Customer Gateway components.

## Prerequisites
- AWS Account with VPN permissions
- Basic understanding of BGP and IPSec
- Access to simulate on-premises environment
- Understanding of routing concepts

## Lab Duration
90 minutes

## Learning Objectives
By the end of this lab, you will:
- Configure AWS Site-to-Site VPN components
- Set up Customer Gateway and VPN Gateway
- Establish IPSec tunnels with redundancy
- Configure dynamic routing with BGP
- Test and troubleshoot VPN connectivity
- Implement monitoring and alerting

## Lab Architecture
```
On-Premises Network ←→ Customer Gateway ←→ Internet ←→ AWS VPN Gateway ←→ VPC
                      (Firewall/Router)              (Virtual Private Gateway)
```

## Lab Steps

### Step 1: Prepare the AWS Environment
1. **Create Lab VPC**
   - Navigate to **VPC Console**
   - Click **Create VPC**
   - Configuration:
     - Name: `SiteToSite-VPN-VPC`
     - IPv4 CIDR: `10.100.0.0/16`
     - IPv6: No IPv6 CIDR block
     - Tenancy: Default
   - Click **Create VPC**

2. **Create Subnets**
   - Private Subnet:
     - Name: `VPN-Private-Subnet-1A`
     - IPv4 CIDR: `10.100.1.0/24`
     - AZ: us-east-1a
   - Private Subnet:
     - Name: `VPN-Private-Subnet-1B`
     - IPv4 CIDR: `10.100.2.0/24`
     - AZ: us-east-1b

3. **Create and Attach Virtual Private Gateway**
   - Navigate to **VPC** → **Virtual Private Gateways**
   - Click **Create virtual private gateway**
   - Configuration:
     - Name: `SiteToSite-VGW`
     - ASN: Amazon default ASN (64512)
   - Click **Create virtual private gateway**
   - Select the VGW → **Actions** → **Attach to VPC**
   - Select your lab VPC and attach

### Step 2: Configure Customer Gateway
1. **Determine Public IP Address**
   - For this lab, we'll simulate an on-premises public IP
   - Use a placeholder IP: `203.0.113.10` (example IP)
   - In production, use your actual public IP address

2. **Create Customer Gateway**
   - Navigate to **VPC** → **Customer Gateways**
   - Click **Create customer gateway**
   - Configuration:
     - Name: `OnPrem-Customer-Gateway`
     - BGP ASN: `65000` (your on-premises ASN)
     - IP Address: `203.0.113.10`
     - Certificate ARN: (leave blank for this lab)
     - Device: (leave blank)
   - Click **Create customer gateway**

### Step 3: Create Site-to-Site VPN Connection
1. **VPN Connection Setup**
   - Navigate to **VPC** → **Site-to-Site VPN Connections**
   - Click **Create VPN connection**
   - Configuration:
     - Name: `Lab-SiteToSite-VPN`
     - Target Gateway Type: Virtual Private Gateway
     - Virtual Private Gateway: Select your VGW
     - Customer Gateway: Existing → Select your customer gateway
     - Routing Options: Dynamic (requires BGP)
     - Tunnel Options: Use default settings for both tunnels

2. **Advanced Tunnel Configuration**
   - Expand **Tunnel 1 Options**:
     - Inside IPv4 CIDR: `169.254.21.0/30` (AWS will auto-assign if left blank)
     - Pre-Shared Key: (auto-generate or specify)
   - Expand **Tunnel 2 Options**:
     - Inside IPv4 CIDR: `169.254.22.0/30`
     - Pre-Shared Key: (auto-generate or specify)
   
3. **Create Connection**
   - Review all settings
   - Click **Create VPN connection**
   - Wait for state to change to "Available" (may take a few minutes)

### Step 4: Download Configuration
1. **Download VPN Configuration**
   - Select your VPN connection
   - Click **Download configuration**
   - Configuration:
     - Vendor: Generic
     - Platform: Generic
     - Software: Vendor Agnostic
   - Click **Download**

2. **Review Configuration File**
   - Open the downloaded configuration file
   - Note the important parameters:
     - Outside IP addresses (AWS tunnel endpoints)
     - Inside IP addresses (BGP peering IPs)
     - Pre-shared keys
     - BGP settings

### Step 5: Simulate On-Premises Configuration
For this lab, we'll use an EC2 instance to simulate the on-premises router.

1. **Create "On-Premises" Simulation VPC**
   - Create a separate VPC: `OnPrem-Simulation-VPC`
   - CIDR: `192.168.0.0/16`
   - Create public subnet: `192.168.1.0/24`
   - Attach Internet Gateway

2. **Launch Router Simulation Instance**
   - Launch EC2 instance in the simulation VPC:
     - AMI: Amazon Linux 2
     - Instance Type: t3.small
     - Subnet: OnPrem public subnet
     - Security Group: Allow UDP 500, 4500 and ICMP
     - User Data Script:
   ```bash
   #!/bin/bash
   yum update -y
   yum install -y openswan
   echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
   sysctl -p
   ```

3. **Configure OpenSwan VPN**
   - SSH to the simulation instance
   - Create IPSec configuration based on downloaded config
   - Configure both tunnels for redundancy

### Step 6: Configure Routing
1. **Enable Route Propagation**
   - Navigate to **VPC** → **Route Tables**
   - Select route tables associated with your private subnets
   - Click **Route Propagation** tab
   - Click **Edit route propagation**
   - Enable propagation for your Virtual Private Gateway
   - Save changes

2. **Verify Route Propagation**
   - In the route table, check **Routes** tab
   - You should see routes propagated from the VPN connection
   - Routes will appear once BGP sessions are established

### Step 7: Test Connectivity
1. **Launch Test Instances**
   - Create EC2 instance in AWS VPC private subnet:
     - Name: `AWS-Test-Instance`
     - Subnet: VPN-Private-Subnet-1A
     - Security Group: Allow ICMP and SSH from on-premises CIDR
   
   - Create EC2 instance in "on-premises" simulation:
     - Name: `OnPrem-Test-Instance`
     - Subnet: OnPrem simulation private subnet
     - Security Group: Allow ICMP and SSH

2. **Connectivity Testing**
   - From on-premises instance, ping AWS instance private IP
   - From AWS instance, ping on-premises instance private IP
   - Test application connectivity (SSH, HTTP, etc.)

### Step 8: Monitor VPN Status
1. **Check Tunnel Status**
   - In VPN connection details, click **Tunnel Details**
   - Verify both tunnels show "UP" status
   - Check BGP session status
   - Review tunnel statistics

2. **CloudWatch Metrics**
   - Navigate to **CloudWatch** → **Metrics**
   - Browse VPN metrics:
     - TunnelState (should be 1 for UP)
     - TunnelIpAddress
   - Create custom dashboard for VPN monitoring

### Step 9: Implement High Availability
1. **Verify Redundant Tunnels**
   - Confirm both IPSec tunnels are established
   - Test failover by disabling primary tunnel
   - Monitor automatic failover to secondary tunnel
   - Verify traffic continues flowing

2. **BGP Route Preferences**
   - Configure BGP attributes for preferred paths
   - Use AS-PATH prepending or local preference
   - Test load balancing across tunnels

### Step 10: Security Hardening
1. **Security Group Configuration**
   - Review and tighten security group rules
   - Allow only necessary traffic between networks
   - Implement principle of least privilege

2. **Network ACLs**
   - Configure subnet-level network ACLs
   - Add additional security layer
   - Test connectivity after ACL implementation

3. **VPN Tunnel Security**
   - Review IPSec parameters
   - Ensure strong encryption algorithms
   - Configure DPD (Dead Peer Detection) settings

## Lab Validation
Complete these verification steps:

1. **VPN Connection Status**
   - [ ] VPN connection shows "Available" state
   - [ ] Both tunnels show "UP" status
   - [ ] BGP sessions are established
   - [ ] Routes are being exchanged

2. **Connectivity Tests**
   - [ ] Ping works from AWS to on-premises
   - [ ] Ping works from on-premises to AWS
   - [ ] Application traffic flows correctly
   - [ ] Failover works when primary tunnel fails

3. **Routing Verification**
   - [ ] Route propagation is enabled
   - [ ] On-premises routes appear in VPC route tables
   - [ ] AWS routes are advertised to on-premises
   - [ ] Traffic follows expected paths

## Troubleshooting Common Issues

### Tunnels Won't Establish
- Check security groups allow UDP 500/4500
- Verify public IP addresses are correct
- Confirm pre-shared keys match
- Review on-premises firewall rules

### BGP Sessions Down
- Verify BGP ASN configuration
- Check inside IP address configuration
- Confirm routing instance configuration
- Review BGP authentication settings

### Connectivity Issues
- Verify route propagation is enabled
- Check security group rules
- Confirm network ACL settings
- Test with traceroute for path verification

### Performance Problems
- Monitor tunnel utilization
- Check for packet loss or latency
- Verify MTU settings (1436 bytes typical)
- Consider multiple VPN connections for bandwidth

## Monitoring and Alerting
1. **Key Metrics to Monitor**
   - Tunnel state changes
   - BGP session status
   - Data transfer volumes
   - Packet loss rates

2. **CloudWatch Alarms**
   - Create alarms for tunnel down events
   - Monitor unusual traffic patterns
   - Alert on BGP session failures
   - Set up SNS notifications

## Cost Optimization
1. **Connection Optimization**
   - Monitor actual bandwidth usage
   - Consider data transfer costs
   - Compare with Direct Connect for high volume

2. **Regional Considerations**
   - Place VPN gateways close to workloads
   - Consider multiple regions for DR
   - Optimize for lowest latency paths

## Clean Up
To avoid ongoing charges:

1. **Delete VPN Connection**
   - Terminate VPN connection
   - Wait for deletion to complete

2. **Clean Up Gateways**
   - Delete Customer Gateway
   - Detach and delete Virtual Private Gateway

3. **Remove Test Resources**
   - Terminate EC2 instances
   - Delete simulation VPC
   - Remove CloudWatch alarms and dashboards

## Key Takeaways
- Site-to-Site VPN provides secure connectivity over internet
- Redundant tunnels ensure high availability
- BGP enables dynamic routing and automatic failover
- Proper monitoring is essential for production deployments
- Security groups and NACLs provide additional security layers
- Cost scales with data transfer, not connection time

## Next Steps
- Proceed to Lab 16: VPN Tunneling and IPSec Deep Dive
- Explore Transit Gateway VPN attachments
- Implement automated VPN deployment with CloudFormation
- Consider AWS VPN CloudHub for multiple site connectivity