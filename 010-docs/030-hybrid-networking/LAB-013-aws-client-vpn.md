# Lab 13: AWS Client VPN - Secure Remote Access

## Lab Overview
This lab demonstrates how to set up and configure AWS Client VPN for secure remote access to VPC resources. Students will create a Client VPN endpoint, configure authentication, and establish secure connections from remote clients.

## Prerequisites
- AWS Account with VPN permissions
- Certificate management knowledge (basic)
- Understanding of VPC concepts
- OpenVPN client software

## Lab Duration
75 minutes

## Learning Objectives
By the end of this lab, you will:
- Create and configure AWS Client VPN endpoint
- Set up certificate-based authentication
- Configure authorization rules and routing
- Connect remote clients to VPC resources
- Monitor and troubleshoot VPN connections

## Lab Architecture
```
Remote Clients → Internet → AWS Client VPN Endpoint → VPC Resources
                                                   → On-premises (via Transit Gateway)
```

## Lab Steps

### Step 1: Prepare the Environment
1. **Create Lab VPC**
   - Navigate to **VPC Console**
   - Click **Create VPC**
   - Configuration:
     - Name: `ClientVPN-Lab-VPC`
     - IPv4 CIDR: `10.200.0.0/16`
     - IPv6: No IPv6 CIDR block
   - Click **Create VPC**

2. **Create Subnets**
   - Create Private Subnet:
     - Name: `ClientVPN-Private-Subnet`
     - IPv4 CIDR: `10.200.1.0/24`
     - AZ: us-east-1a
   - Create Public Subnet:
     - Name: `ClientVPN-Public-Subnet`
     - IPv4 CIDR: `10.200.2.0/24`
     - AZ: us-east-1a

3. **Create Internet Gateway**
   - Click **Internet Gateways** → **Create internet gateway**
   - Name: `ClientVPN-IGW`
   - Attach to your lab VPC

4. **Configure Route Tables**
   - Update public subnet route table:
     - Add route: `0.0.0.0/0` → Internet Gateway
   - Create route for private subnet access

### Step 2: Generate Certificates
1. **Install Certificate Tools**
   - For Windows: Download OpenVPN Easy-RSA
   - For Linux/Mac: Install easy-rsa package
   ```bash
   # Linux/Mac example
   git clone https://github.com/OpenVPN/easy-rsa.git
   cd easy-rsa/easyrsa3
   ```

2. **Create Certificate Authority (CA)**
   ```bash
   ./easyrsa init-pki
   ./easyrsa build-ca nopass
   # Enter Common Name: ClientVPN-CA
   ```

3. **Generate Server Certificate**
   ```bash
   ./easyrsa build-server-full server nopass
   ```

4. **Generate Client Certificate**
   ```bash
   ./easyrsa build-client-full client1.domain.tld nopass
   ```

5. **Upload Certificates to ACM**
   - Navigate to **AWS Certificate Manager**
   - Click **Import a certificate**
   - For Server Certificate:
     - Certificate body: Copy contents of `pki/issued/server.crt`
     - Certificate private key: Copy contents of `pki/private/server.key`
     - Certificate chain: Copy contents of `pki/ca.crt`
   - Import and note the ARN
   - Repeat for client certificate

### Step 3: Create Client VPN Endpoint
1. **Basic Configuration**
   - Navigate to **VPC Console** → **Client VPN Endpoints**
   - Click **Create Client VPN endpoint**
   - Configuration:
     - Name: `Lab-ClientVPN-Endpoint`
     - Description: `Lab Client VPN for remote access`
     - Client IPv4 CIDR: `172.16.0.0/16` (non-overlapping with VPC)
     - Server certificate ARN: Select your imported server certificate

2. **Authentication Configuration**
   - Authentication Options:
     - Use mutual authentication: Yes
     - Client certificate ARN: Select your imported client certificate
   - Connection Logging:
     - Enable: Yes
     - CloudWatch log group: Create new group `ClientVPN-Logs`

3. **Advanced Options**
   - Transport Protocol: UDP (default)
   - Port: 443 (default)
   - DNS Servers: Leave blank for VPC default
   - VPC Security Groups: Create new or select existing
   - VPN Session Timeout: 24 hours

4. **Create Endpoint**
   - Review configuration
   - Click **Create Client VPN endpoint**
   - Wait for status to change to "Available"

### Step 4: Configure Network Associations
1. **Associate Target Networks**
   - Select your Client VPN endpoint
   - Click **Associations** tab
   - Click **Associate**
   - Configuration:
     - VPC: Select your lab VPC
     - Subnet: Select private subnet
   - Click **Associate**

2. **Review Association**
   - Wait for association status: "Associated"
   - Note the security group created automatically
   - Review the network interface created in the subnet

### Step 5: Configure Authorization Rules
1. **Create Authorization Rule for VPC Access**
   - Click **Authorization rules** tab
   - Click **Add authorization rule**
   - Configuration:
     - Destination network: `10.200.0.0/16` (VPC CIDR)
     - Description: `Allow access to VPC resources`
     - Grant access to: All users
   - Click **Add authorization rule**

2. **Create Authorization Rule for Internet Access**
   - Add another authorization rule:
     - Destination network: `0.0.0.0/0`
     - Description: `Allow internet access through VPN`
     - Grant access to: All users

3. **Verify Authorization Rules**
   - Confirm both rules show status "Active"
   - Review the scope of access granted

### Step 6: Configure Route Table
1. **Add Route for VPC Access**
   - Click **Route table** tab
   - Click **Create route**
   - Configuration:
     - Route destination: `10.200.0.0/16`
     - Target VPC Subnet ID: Select your associated subnet
     - Description: `Route to VPC resources`

2. **Add Route for Internet Access (Optional)**
   - Create route:
     - Route destination: `0.0.0.0/0`
     - Target VPC Subnet ID: Select public subnet with IGW access
     - Description: `Internet access via VPC`

### Step 7: Download Client Configuration
1. **Generate Client Configuration File**
   - Select your Client VPN endpoint
   - Click **Download Client Configuration**
   - Save the file as `clientvpn-config.ovpn`

2. **Modify Configuration File**
   - Open the downloaded file in text editor
   - Add client certificate and key at the end:
   ```
   <cert>
   [Contents of pki/issued/client1.domain.tld.crt]
   </cert>

   <key>
   [Contents of pki/private/client1.domain.tld.key]
   </key>
   ```

### Step 8: Test VPN Connection
1. **Install OpenVPN Client**
   - Download from official OpenVPN website
   - Install on your local machine

2. **Import Configuration**
   - Launch OpenVPN client
   - Import your modified configuration file
   - Connect to the VPN

3. **Verify Connection**
   - Check VPN client shows "Connected"
   - Verify IP address assignment from Client IPv4 CIDR
   - Test connectivity to VPC resources

### Step 9: Create Test Resources
1. **Launch EC2 Instance**
   - Navigate to **EC2 Console**
   - Launch instance in private subnet:
     - AMI: Amazon Linux 2
     - Instance type: t3.micro
     - Subnet: ClientVPN-Private-Subnet
     - Security group: Allow SSH (port 22) from Client VPN CIDR

2. **Test Connectivity**
   - With VPN connected, SSH to the private instance
   - Verify you can reach the instance without bastion host
   - Test internet connectivity from the instance

### Step 10: Monitor and Troubleshoot
1. **CloudWatch Logs Review**
   - Navigate to **CloudWatch** → **Log Groups**
   - Select `ClientVPN-Logs` group
   - Review connection and authentication logs
   - Look for successful connections and any errors

2. **Connection Monitoring**
   - In Client VPN endpoint, click **Connections** tab
   - Review active connections
   - Note client IP assignments and connection duration
   - Monitor connection status and any disconnections

3. **Metrics and Alarms**
   - Set up CloudWatch metrics for:
     - Active connections count
     - Authentication failures
     - Data transfer volumes
   - Create alarms for unusual activity

### Step 11: Security Hardening
1. **Security Group Review**
   - Review auto-created security group rules
   - Tighten rules to minimum required access
   - Document allowed ports and protocols

2. **Certificate Management**
   - Implement certificate rotation schedule
   - Store certificates securely
   - Document certificate lifecycle

3. **Access Controls**
   - Review IAM policies for VPN management
   - Implement principle of least privilege
   - Set up user-specific authorization rules if needed

## Lab Validation
Complete these verification steps:

1. **VPN Endpoint Status**
   - [ ] Client VPN endpoint shows "Available"
   - [ ] Network association is "Associated"
   - [ ] Authorization rules are "Active"
   - [ ] Routes are properly configured

2. **Client Connectivity**
   - [ ] VPN client connects successfully
   - [ ] Client receives IP from designated CIDR
   - [ ] Can access VPC resources through VPN
   - [ ] Internet access works (if configured)

3. **Monitoring Setup**
   - [ ] CloudWatch logs are capturing events
   - [ ] Connection monitoring shows active sessions
   - [ ] Security groups allow appropriate access

## Troubleshooting Common Issues

### Certificate Import Errors
- Verify certificate format (PEM)
- Check certificate chain completeness
- Ensure private key matches certificate
- Validate certificate hasn't expired

### Connection Failures
- Check client configuration syntax
- Verify certificate embedding in config file
- Review authorization rules
- Check security group rules in associated subnet

### No Internet Access Through VPN
- Verify route table has 0.0.0.0/0 route
- Check target subnet has internet gateway access
- Review NAT gateway configuration if using private subnet
- Confirm authorization rule allows 0.0.0.0/0

### Performance Issues
- Monitor bandwidth utilization
- Check client location vs VPN endpoint region
- Review concurrent connection limits
- Consider multiple endpoint regions

## Cost Optimization
1. **Endpoint Sizing**
   - Monitor actual concurrent connections
   - Right-size endpoint based on usage
   - Consider session timeout settings

2. **Data Transfer Costs**
   - Monitor outbound data transfer
   - Optimize routing for cost efficiency
   - Review regional data transfer rates

## Clean Up
To avoid ongoing charges:

1. **Disconnect VPN Clients**
   - Disconnect all active VPN sessions
   - Verify no active connections remain

2. **Delete Client VPN Endpoint**
   - Remove authorization rules
   - Delete route table entries
   - Disassociate target networks
   - Delete the endpoint

3. **Clean Up Supporting Resources**
   - Delete test EC2 instances
   - Remove CloudWatch log groups
   - Delete imported certificates from ACM
   - Delete VPC and associated resources

## Key Takeaways
- Client VPN provides secure remote access to VPC resources
- Certificate-based authentication ensures strong security
- Authorization rules control resource access granularly
- Proper routing configuration enables internet and VPC access
- Monitoring is crucial for security and troubleshooting
- Cost scales with active connections and data transfer

## Next Steps
- Proceed to Lab 14: VPN Gateway Redundancy and High Availability
- Explore Active Directory integration for user authentication
- Implement automated certificate management
- Consider split-tunnel configurations for optimization