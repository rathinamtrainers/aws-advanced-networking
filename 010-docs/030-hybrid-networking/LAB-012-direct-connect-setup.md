# Lab 12: Setting Up AWS Direct Connect - Step-by-Step

## Lab Overview
This comprehensive lab walks through the complete process of setting up AWS Direct Connect, including creating connections, configuring Virtual Interfaces (VIFs), and establishing connectivity to VPC resources.

## Prerequisites
- AWS Account with Direct Connect permissions
- Basic networking knowledge (BGP, VLANs)
- Completed Lab 11: Direct Connect Introduction
- Access to network equipment (simulated in this lab)

## Lab Duration
90 minutes

## Learning Objectives
By the end of this lab, you will:
- Create and configure a Direct Connect connection
- Set up Private and Public Virtual Interfaces
- Configure BGP routing
- Test connectivity and troubleshoot issues
- Implement redundancy patterns

## Lab Architecture
```
On-Premises Network ↔ Direct Connect Location ↔ AWS Direct Connect ↔ VPC
                                                                    ↔ Public AWS Services
```

## Lab Steps

### Step 1: Prepare the Environment
1. **Create a Test VPC**
   - Navigate to **VPC Console**
   - Click **Create VPC**
   - Configuration:
     - Name: `DirectConnect-Lab-VPC`
     - IPv4 CIDR: `10.100.0.0/16`
     - IPv6 CIDR: No IPv6 CIDR block
     - Tenancy: Default
   - Click **Create VPC**

2. **Create Subnets**
   - Create Private Subnet:
     - Name: `DirectConnect-Private-Subnet`
     - VPC: Select your lab VPC
     - IPv4 CIDR: `10.100.1.0/24`
     - Availability Zone: us-east-1a
   - Create Public Subnet:
     - Name: `DirectConnect-Public-Subnet`
     - IPv4 CIDR: `10.100.2.0/24`
     - Availability Zone: us-east-1a

3. **Create Virtual Private Gateway**
   - Navigate to **VPC** → **Virtual Private Gateways**
   - Click **Create virtual private gateway**
   - Configuration:
     - Name: `DirectConnect-VGW`
     - ASN: Amazon default ASN
   - Click **Create virtual private gateway**
   - Select the VGW and click **Actions** → **Attach to VPC**
   - Select your lab VPC and click **Attach**

### Step 2: Request Direct Connect Connection
1. **Create Connection Request**
   - Navigate to **Direct Connect Console**
   - Click **Connections** → **Create connection**
   - Configuration:
     - Name: `Lab-DX-Connection`
     - Location: Select nearest location
     - Port Speed: 1Gbps (lowest cost option)
     - On-premises equipment: No (partner equipment)
   - Click **Create connection**

2. **Connection Status Monitoring**
   - Note the connection state: "Requested"
   - Record the Connection ID
   - In a real scenario, this would require physical setup at the location
   - For this lab, we'll simulate the "Available" state

### Step 3: Create Private Virtual Interface (VIF)
1. **Private VIF Configuration**
   - Select your connection
   - Click **Create virtual interface**
   - Choose **Private**
   - Configuration:
     - Name: `Lab-Private-VIF`
     - Connection: Your connection ID
     - VLAN: 100
     - BGP ASN: 65000 (your on-premises ASN)
     - Router IP: 192.168.1.1/30 (your side)
     - Amazon IP: 192.168.1.2/30 (AWS side)
     - BGP Auth Key: (leave blank for auto-generation)
     - Virtual Private Gateway: Select your VGW

2. **BGP Configuration Review**
   - Note the BGP session details
   - Record the BGP authentication key
   - Understand the /30 subnet for BGP peering

### Step 4: Create Public Virtual Interface (VIF)
1. **Public VIF Setup**
   - Click **Create virtual interface**
   - Choose **Public**
   - Configuration:
     - Name: `Lab-Public-VIF`
     - Connection: Your connection ID
     - VLAN: 200
     - BGP ASN: 65000
     - Router IP: 192.168.2.1/30
     - Amazon IP: 192.168.2.2/30
     - Prefixes to advertise: 203.0.113.0/24 (example public IP)

2. **Route Filter Configuration**
   - Add route filters for AWS services:
     - S3 prefixes for your region
     - CloudFront prefixes
     - Other AWS service prefixes as needed

### Step 5: Configure Route Tables
1. **VPC Route Table Updates**
   - Navigate to **VPC** → **Route Tables**
   - Select the route table associated with your private subnet
   - Click **Routes** → **Edit routes**
   - Add route:
     - Destination: `192.168.0.0/16` (on-premises network)
     - Target: Virtual Private Gateway
   - Save changes

2. **Propagate Routes**
   - In the same route table, click **Route propagation**
   - Click **Edit route propagation**
   - Enable propagation for your Virtual Private Gateway
   - Save changes

### Step 6: Test Connectivity (Simulated)
1. **BGP Session Verification**
   - In Direct Connect console, select your Private VIF
   - Check **BGP session status** (would show "Up" when configured)
   - Review advertised and received routes

2. **Connection Testing**
   - Create an EC2 instance in your private subnet
   - Configure security groups to allow ICMP and SSH
   - Simulate ping tests to on-premises resources
   - Test application connectivity

### Step 7: Implement Redundancy
1. **Create Second Connection**
   - Repeat connection creation process
   - Choose different Direct Connect location
   - Same configuration as primary connection

2. **Configure Second VIF**
   - Create another Private VIF on second connection
   - Use different VLAN (101) and BGP IPs (192.168.3.0/30)
   - Same ASN and VGW association

3. **Route Table Redundancy**
   - Verify both VIFs propagate routes
   - Test failover scenarios (simulated)
   - Implement route preferences using BGP attributes

### Step 8: Monitoring and Troubleshooting
1. **CloudWatch Metrics Setup**
   - Navigate to **CloudWatch Console**
   - Search for Direct Connect metrics
   - Create dashboard with key metrics:
     - ConnectionState
     - ConnectionBpsEgress/Ingress
     - ConnectionLightLevelTx/Rx
     - ConnectionCRCErrorCount

2. **Set Up Alarms**
   - Create CloudWatch alarm for connection state changes
   - Set up notification for BGP session down events
   - Configure bandwidth utilization alerts

### Step 9: Security Configuration
1. **Access Control**
   - Review IAM policies for Direct Connect management
   - Implement least-privilege access
   - Document who can modify connections and VIFs

2. **Network Security**
   - Configure on-premises firewall rules
   - Implement network ACLs in VPC
   - Document security group requirements

## Lab Validation
Complete these verification steps:

1. **Connection Status**
   - [ ] Connection shows "Available" state
   - [ ] Private VIF shows "Available" state
   - [ ] Public VIF shows "Available" state
   - [ ] BGP sessions are established

2. **Routing Verification**
   - [ ] On-premises routes appear in VPC route table
   - [ ] AWS routes are advertised to on-premises
   - [ ] Route propagation is working correctly

3. **Connectivity Tests**
   - [ ] Ping from VPC to on-premises succeeds
   - [ ] Application traffic flows correctly
   - [ ] Public VIF can reach AWS services

## Troubleshooting Common Issues

### BGP Session Won't Establish
- Verify IP addressing on both sides
- Check BGP ASN configuration
- Confirm VLAN tagging
- Validate BGP authentication key

### No Route Propagation
- Ensure route propagation is enabled
- Check Virtual Private Gateway attachment
- Verify BGP advertisements from on-premises
- Review route table associations

### Intermittent Connectivity
- Check physical layer statistics
- Monitor CRC errors
- Verify redundancy configuration
- Review BGP timers and keepalives

## Cost Optimization
1. **Right-size Connections**
   - Monitor bandwidth utilization
   - Consider hosted connections for lower bandwidth needs
   - Review data transfer patterns

2. **Regional Considerations**
   - Use Direct Connect Gateway for multi-region access
   - Optimize data transfer routing
   - Consider Local Zones for edge connectivity

## Clean Up
To avoid ongoing charges:

1. **Delete Virtual Interfaces**
   - Select each VIF and click **Delete**
   - Confirm deletion

2. **Delete Connections**
   - Select connection and click **Delete**
   - Note: In real scenarios, coordinate with facility/partner

3. **Clean Up VPC Resources**
   - Detach and delete Virtual Private Gateway
   - Delete test EC2 instances
   - Delete VPC and associated resources

## Key Takeaways
- Direct Connect setup requires coordination with AWS partners
- BGP configuration is critical for proper routing
- Redundancy requires multiple connections and proper route management
- Monitoring is essential for maintaining connectivity
- Proper planning reduces implementation time and issues

## Next Steps
- Proceed to Lab 13: Direct Connect Public vs Private VIF Deep Dive
- Implement Direct Connect in your production environment
- Explore Direct Connect Gateway for multi-region connectivity