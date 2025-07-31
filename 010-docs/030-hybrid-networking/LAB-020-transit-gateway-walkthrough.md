# Lab 20: AWS Transit Gateway - Complete Walkthrough

## Lab Overview
This comprehensive lab demonstrates how to implement AWS Transit Gateway as a central hub for connecting multiple VPCs, on-premises networks, and remote users. Students will build a hub-and-spoke network architecture with advanced routing and security policies.

## Prerequisites
- AWS Account with Transit Gateway permissions
- Understanding of VPC concepts and routing
- Basic knowledge of BGP (for advanced scenarios)
- Completed previous VPN labs (recommended)

## Lab Duration
120 minutes

## Learning Objectives
By the end of this lab, you will:
- Create and configure AWS Transit Gateway
- Attach multiple VPCs and VPN connections
- Configure route tables and routing policies
- Implement network segmentation and security
- Test inter-VPC and hybrid connectivity
- Monitor and troubleshoot Transit Gateway

## Lab Architecture
```
                    ┌─────────────────┐
                    │  Transit Gateway │
                    │   (Central Hub)  │
                    └─────────────────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
   ┌─────────┐        ┌─────────┐        ┌─────────┐
   │ VPC-A   │        │ VPC-B   │        │ VPC-C   │
   │ Prod    │        │ Dev     │        │ Shared  │
   └─────────┘        └─────────┘        └─────────┘
        │                                      │
   ┌─────────┐                           ┌─────────┐
   │On-Prem  │                           │Client   │
   │Network  │                           │VPN      │
   └─────────┘                           └─────────┘
```

## Lab Steps

### Step 1: Create Transit Gateway
1. **Access Transit Gateway Console**
   - Navigate to **VPC Console** → **Transit Gateways**
   - Click **Create Transit Gateway**

2. **Basic Configuration**
   - Name: `Lab-Transit-Gateway`
   - Description: `Lab Transit Gateway for multi-VPC connectivity`
   - Amazon side ASN: `64512` (default)
   - Auto accept shared attachments: Enable
   - Default route table association: Enable
   - Default route table propagation: Enable

3. **Advanced Settings**
   - DNS support: Enable
   - Multicast support: Disable (for this lab)
   - CIDR blocks: Leave empty (will be populated automatically)

4. **Create Transit Gateway**
   - Review configuration
   - Click **Create Transit Gateway**
   - Wait for state to change to "Available" (takes 5-10 minutes)

### Step 2: Create Multiple VPCs
1. **Production VPC (VPC-A)**
   - Navigate to **VPC Console**
   - Create VPC:
     - Name: `Production-VPC`
     - IPv4 CIDR: `10.10.0.0/16`
   - Create subnets:
     - Private: `10.10.1.0/24` (us-east-1a)
     - Private: `10.10.2.0/24` (us-east-1b)

2. **Development VPC (VPC-B)**
   - Create VPC:
     - Name: `Development-VPC`
     - IPv4 CIDR: `10.20.0.0/16`
   - Create subnets:
     - Private: `10.20.1.0/24` (us-east-1a)
     - Public: `10.20.2.0/24` (us-east-1b)
   - Attach Internet Gateway to public subnet

3. **Shared Services VPC (VPC-C)**
   - Create VPC:
     - Name: `SharedServices-VPC`
     - IPv4 CIDR: `10.30.0.0/16`
   - Create subnets:
     - Private: `10.30.1.0/24` (us-east-1a)
     - Private: `10.30.2.0/24` (us-east-1b)

### Step 3: Create VPC Attachments
1. **Attach Production VPC**
   - In Transit Gateway console, click **Attachments**
   - Click **Create Transit Gateway Attachment**
   - Configuration:
     - Name: `Prod-VPC-Attachment`
     - Transit Gateway: Select your lab TGW
     - Attachment Type: VPC
     - VPC: Production-VPC
     - Subnets: Select both private subnets
   - Click **Create attachment**

2. **Attach Development VPC**
   - Create attachment:
     - Name: `Dev-VPC-Attachment`
     - VPC: Development-VPC
     - Subnets: Select both subnets

3. **Attach Shared Services VPC**
   - Create attachment:
     - Name: `Shared-VPC-Attachment`
     - VPC: SharedServices-VPC
     - Subnets: Select both private subnets

4. **Wait for Attachments**
   - Monitor attachment status
   - Wait for all to show "Available"

### Step 4: Configure Route Tables
1. **Create Custom Route Tables**
   - Navigate to **Transit Gateway Route Tables**
   - Click **Create Transit Gateway Route Table**
   - Create three route tables:
     - Name: `Production-Route-Table`
     - Name: `Development-Route-Table`
     - Name: `Shared-Route-Table`

2. **Associate Attachments**
   - Select `Production-Route-Table`
   - Click **Associations** → **Create association**
   - Associate with `Prod-VPC-Attachment`
   - Repeat for other route tables and their respective attachments

3. **Configure Route Propagation**
   - For Production Route Table:
     - Enable propagation from Shared Services attachment
     - Disable propagation from Development attachment
   - For Development Route Table:
     - Enable propagation from Shared Services attachment
     - Disable propagation from Production attachment
   - For Shared Route Table:
     - Enable propagation from all attachments

### Step 5: Add Static Routes
1. **Production to Shared Services**
   - In Production Route Table, click **Routes** → **Create route**
   - Configuration:
     - CIDR: `10.30.0.0/16`
     - Attachment: Shared-VPC-Attachment
   - Click **Create route**

2. **Development to Shared Services**
   - In Development Route Table, create route:
     - CIDR: `10.30.0.0/16`
     - Attachment: Shared-VPC-Attachment

3. **Shared Services to All**
   - In Shared Route Table, routes should be auto-propagated
   - Verify routes to `10.10.0.0/16` and `10.20.0.0/16` exist

### Step 6: Update VPC Route Tables
1. **Production VPC Routes**
   - Navigate to **VPC** → **Route Tables**
   - Select Production VPC route tables
   - Add routes:
     - `10.20.0.0/16` → Transit Gateway
     - `10.30.0.0/16` → Transit Gateway
     - `192.168.0.0/16` → Transit Gateway (for on-premises)

2. **Development VPC Routes**
   - Update Development VPC route tables:
     - `10.10.0.0/16` → Transit Gateway (only if needed)
     - `10.30.0.0/16` → Transit Gateway
     - `192.168.0.0/16` → Transit Gateway

3. **Shared Services VPC Routes**
   - Update Shared Services route tables:
     - `10.10.0.0/16` → Transit Gateway
     - `10.20.0.0/16` → Transit Gateway
     - `192.168.0.0/16` → Transit Gateway

### Step 7: Create Test Instances
1. **Production Instance**
   - Launch EC2 instance in Production VPC:
     - Name: `Prod-Test-Instance`
     - AMI: Amazon Linux 2
     - Instance Type: t3.micro
     - Subnet: Production private subnet
     - Security Group: Allow ICMP and SSH from 10.0.0.0/8

2. **Development Instance**
   - Launch instance in Development VPC:
     - Name: `Dev-Test-Instance`
     - Subnet: Development private subnet
     - Same security group configuration

3. **Shared Services Instance**
   - Launch instance in Shared Services VPC:
     - Name: `Shared-Test-Instance`
     - Subnet: Shared Services private subnet
     - Security Group: Allow access from all VPCs

### Step 8: Test Inter-VPC Connectivity
1. **Basic Connectivity Tests**
   - Connect to Production instance (via Session Manager or bastion)
   - Test ping to Development instance private IP
   - Test ping to Shared Services instance private IP
   - Document which connections work based on routing rules

2. **Verify Route Isolation**
   - From Production, attempt to reach Development directly
   - Should fail due to route table configuration
   - From Shared Services, should reach both Production and Development

3. **Application Level Testing**
   - Set up simple web server on Shared Services instance
   - Test HTTP connectivity from Production and Development
   - Verify service discovery and name resolution

### Step 9: Add VPN Connection
1. **Create Customer Gateway**
   - Navigate to **VPC** → **Customer Gateways**
   - Create Customer Gateway:
     - Name: `OnPrem-Customer-Gateway`
     - BGP ASN: `65000`
     - IP Address: Your public IP or test IP

2. **Create VPN Connection**
   - Navigate to **VPC** → **Site-to-Site VPN Connections**
   - Click **Create VPN Connection**
   - Configuration:
     - Target Gateway Type: Transit Gateway
     - Transit Gateway: Select your lab TGW
     - Customer Gateway: Select created gateway
     - Routing: Dynamic (BGP)

3. **Configure VPN Routing**
   - Once VPN is available, check Transit Gateway route tables
   - Verify on-premises routes are propagated
   - Test connectivity from VPCs to simulated on-premises

### Step 10: Implement Network Segmentation
1. **Create Inspection VPC**
   - Create additional VPC for security inspection:
     - Name: `Inspection-VPC`
     - CIDR: `10.40.0.0/16`
   - Deploy firewall or inspection appliance

2. **Configure Traffic Inspection**
   - Attach Inspection VPC to Transit Gateway
   - Configure routes to force traffic through inspection VPC
   - Test that inter-VPC traffic passes through inspection

3. **Implement Security Policies**
   - Use security groups for micro-segmentation
   - Configure NACLs for additional security
   - Document traffic flows and security zones

### Step 11: Monitor and Troubleshoot
1. **CloudWatch Metrics**
   - Navigate to **CloudWatch** → **Metrics**
   - Browse Transit Gateway metrics:
     - BytesIn/BytesOut
     - PacketDropCount
     - AttachmentState
   - Create custom dashboard

2. **VPC Flow Logs**
   - Enable Flow Logs for all VPCs
   - Configure delivery to CloudWatch Logs
   - Use Flow Logs to troubleshoot connectivity issues

3. **Route Analysis**
   - Use Transit Gateway Network Manager (if available)
   - Analyze route tables and propagation
   - Identify routing loops or misconfigurations

### Step 12: Advanced Features
1. **Multicast Configuration** (Optional)
   - Enable multicast support on Transit Gateway
   - Create multicast domain
   - Test multicast traffic between VPCs

2. **Inter-Region Peering** (Optional)
   - Create Transit Gateway in another region
   - Establish inter-region peering
   - Test cross-region connectivity

3. **Resource Sharing**
   - Use AWS Resource Access Manager (RAM)
   - Share Transit Gateway with other AWS accounts
   - Configure cross-account attachment permissions

## Lab Validation
Complete these verification steps:

1. **Transit Gateway Status**
   - [ ] Transit Gateway shows "Available" state
   - [ ] All VPC attachments are "Available"
   - [ ] Route tables are properly associated
   - [ ] Routes are propagated correctly

2. **Connectivity Tests**
   - [ ] Production can reach Shared Services
   - [ ] Development can reach Shared Services
   - [ ] Production cannot directly reach Development
   - [ ] VPN connectivity works (if configured)

3. **Routing Verification**
   - [ ] Route tables contain expected routes
   - [ ] VPC route tables point to Transit Gateway
   - [ ] BGP sessions are established (for VPN)
   - [ ] No routing loops exist

## Troubleshooting Common Issues

### Attachment Failures
- Check subnet selection (must be in different AZs)
- Verify Transit Gateway is in Available state
- Review IAM permissions for attachment creation
- Check for CIDR conflicts

### Routing Issues
- Verify route table associations
- Check route propagation settings
- Review VPC route table entries
- Use VPC Reachability Analyzer for path testing

### Connectivity Problems
- Check security group rules
- Review Network ACL configurations
- Verify instance subnet associations
- Test with VPC Flow Logs analysis

### Performance Issues
- Monitor Transit Gateway metrics
- Check for bandwidth limitations
- Review attachment utilization
- Consider multiple attachments for high bandwidth

## Cost Optimization
1. **Attachment Optimization**
   - Monitor actual attachment utilization
   - Remove unused attachments
   - Consider consolidating low-traffic VPCs

2. **Data Transfer Costs**
   - Monitor inter-AZ data transfer
   - Optimize traffic patterns
   - Use same-AZ resources where possible

3. **Route Table Management**
   - Use default route tables where appropriate
   - Minimize custom route table count
   - Regularly audit routing configurations

## Clean Up
To avoid ongoing charges:

1. **Delete Attachments**
   - Remove all VPC attachments
   - Delete VPN connections
   - Wait for deletions to complete

2. **Delete Transit Gateway**
   - Remove custom route tables
   - Delete the Transit Gateway
   - Verify deletion completes

3. **Clean Up Test Resources**
   - Terminate EC2 instances
   - Delete test VPCs
   - Remove CloudWatch resources

## Key Takeaways
- Transit Gateway simplifies complex network topologies
- Route tables enable flexible traffic control and segmentation
- Proper planning prevents routing loops and security issues
- Monitoring is essential for troubleshooting and optimization
- Cost scales with attachments and data transfer
- Integration with other AWS services enhances functionality

## Next Steps
- Proceed to Lab 21: Global Accelerator Integration
- Explore AWS Network Manager for monitoring
- Implement automation with CloudFormation
- Design production-ready network architectures
- Consider AWS Cloud WAN for global networks