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
     - **Routing Options**: Choose based on complexity needs:
       - **Static**: Simpler, manually configure routes (recommended for labs)
       - **Dynamic**: Uses BGP, automatic route exchange (production environments)
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

**IMPORTANT**: AWS VPN requires DH Group 2 (modp1024) support. Due to compatibility issues, use Ubuntu with StrongSwan instead of Amazon Linux with Libreswan.

1. **Create "On-Premises" Simulation VPC**
   - Create a separate VPC: `OnPrem-Simulation-VPC`
   - CIDR: `192.168.0.0/16`
   - Create public subnet: `192.168.1.0/24`
   - Attach Internet Gateway

2. **Launch Router Simulation Instance**
   - Launch EC2 instance in the simulation VPC:
     - **AMI: Ubuntu Server 22.04 LTS** (recommended for StrongSwan compatibility)
     - Instance Type: t3.small
     - Subnet: OnPrem public subnet
     - Security Group: Allow UDP 500, 4500, SSH (22), and ICMP
     - Assign Elastic IP for consistent public IP

3. **Install and Configure StrongSwan**
   ```bash
   # Update system
   sudo apt update && sudo apt upgrade -y
   
   # Install StrongSwan
   sudo apt install -y strongswan
   
   # Enable IP forwarding
   echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.conf
   sudo sysctl -p
   ```

4. **Configure StrongSwan for AWS VPN**
   Create `/etc/ipsec.conf`:
   ```bash
   config setup
       charondebug="ike 1, knl 1, cfg 0"
       uniqueids=no

   conn aws-tunnel-1
       authby=secret
       auto=start
       left=%defaultroute
       leftid=YOUR_ELASTIC_IP
       right=AWS_TUNNEL_1_ENDPOINT
       type=tunnel
       ikelifetime=28800s
       lifetime=3600s
       ike=aes128-sha1-modp1024!
       esp=aes128-sha1!
       leftsubnet=192.168.0.0/16
       rightsubnet=10.100.0.0/16
       dpdaction=restart
       dpddelay=10s
       keyingtries=%forever
   ```
   
   Create `/etc/ipsec.secrets`:
   ```bash
   YOUR_ELASTIC_IP AWS_TUNNEL_1_ENDPOINT : PSK "YOUR_PRE_SHARED_KEY"
   ```
   
   **Note**: Replace placeholder values with actual IPs and keys from your downloaded VPN configuration.

5. **Alternative: Static Routing Approach**
   If BGP complexity is not needed, configure VPN connection with static routing:
   - During VPN creation, select "Static" routing option
   - Add static routes for on-premises CIDR (192.168.0.0/16)
   - This simplifies configuration and avoids BGP complexity

### Step 6: Configure Routing

#### Option A: Dynamic Routing (BGP)
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

#### Option B: Static Routing (Recommended for Labs)
1. **Add Static Routes**
   - Navigate to **VPC** → **Route Tables**
   - Select route tables associated with your private subnets
   - Click **Routes** tab → **Edit routes**
   - Add route:
     - Destination: `192.168.0.0/16` (on-premises CIDR)
     - Target: Virtual Private Gateway (your VGW)
   - Save changes

2. **Configure Static Routes in VPN Connection**
   - In VPN connection, add static routes for on-premises networks
   - This eliminates need for BGP configuration complexity

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
   - [ ] BGP sessions are established (if using dynamic routing)
   - [ ] Routes are being exchanged (dynamic) or manually configured (static)

2. **IPSec Tunnel Verification**
   ```bash
   # On StrongSwan instance, verify tunnel status
   sudo ipsec statusall
   
   # Check for active Child SAs (ESP tunnels)
   sudo ipsec status
   
   # Verify XFRM policies are installed
   ip xfrm policy show
   ip xfrm state show
   ```

3. **Connectivity Tests**
   - [ ] Phase 1 (IKE) tunnel establishes successfully
   - [ ] Phase 2 (ESP) Child SAs are created
   - [ ] Ping works from AWS to on-premises (expect ~2-5ms latency)
   - [ ] Ping works from on-premises to AWS
   - [ ] Application traffic flows correctly
   - [ ] Failover works when primary tunnel fails

4. **Routing Verification**
   - [ ] Route propagation is enabled (dynamic routing)
   - [ ] Static routes are configured (static routing)
   - [ ] On-premises routes appear in VPC route tables
   - [ ] AWS routes are advertised to on-premises (dynamic) or reachable (static)
   - [ ] Traffic follows expected paths (use `traceroute` to verify)
   - [ ] No routing conflicts bypass IPSec tunnels

5. **Performance Baseline**
   - [ ] Establish baseline latency (typically 2-5ms for same region)
   - [ ] Verify MTU discovery works correctly (try large ping packets)
   - [ ] Monitor tunnel utilization in CloudWatch

## Troubleshooting Common Issues

### DH Group Compatibility Issues
**Error**: `NO_PROPOSAL_CHOSEN` during IKE negotiation
**Cause**: IPSec software doesn't support DH Group 2 (modp1024) required by AWS
**Solution**: 
- Use Ubuntu 22.04 with StrongSwan instead of Amazon Linux with Libreswan
- Verify `ike=aes128-sha1-modp1024!` in configuration
- Check StrongSwan version supports legacy DH groups

### Tunnels Won't Establish
- Check security groups allow UDP 500/4500
- Verify public IP addresses are correct
- Confirm pre-shared keys match exactly (case-sensitive)
- Review on-premises firewall rules
- Ensure Elastic IP is assigned to customer gateway instance

### Child SAs Not Established
**Symptom**: IKE (Phase 1) connects but ESP tunnels (Phase 2) fail
**Debug Commands**:
```bash
sudo ipsec statusall
sudo ipsec restart
ip xfrm state
ip xfrm policy
```
**Solution**: Often resolved by restarting IPSec service after initial connection

### BGP Sessions Down
- Verify BGP ASN configuration
- Check inside IP address configuration
- Confirm routing instance configuration
- Review BGP authentication settings
- **Alternative**: Use static routing to avoid BGP complexity

### Connectivity Issues After Tunnel Establishment
**Symptom**: Tunnels show UP but ping fails
**Debug Steps**:
1. Check routing conflicts:
   ```bash
   ip route show
   # Look for routes bypassing IPSec
   ```
2. Monitor traffic with tcpdump:
   ```bash
   sudo tcpdump -i any icmp
   ```
3. Remove conflicting routes:
   ```bash
   sudo ip route del 10.100.0.0/16 dev ens5
   ```
4. Verify XFRM policies are active:
   ```bash
   ip xfrm policy show
   ```

### Route Propagation vs Static Routes
- **BGP/Dynamic**: Requires route propagation enabled in VPC route tables
- **Static**: Routes must be manually added to VPC route tables
- **Recommendation**: Use static routing for simpler lab environments

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
- **DH Group 2 (modp1024) compatibility is critical** - use StrongSwan on Ubuntu for reliable AWS VPN connectivity
- **Static routing simplifies lab environments** while BGP provides production-grade dynamic routing
- Redundant tunnels ensure high availability
- **Child SA establishment is separate from IKE negotiation** - both phases must complete successfully
- **Routing conflicts can bypass IPSec** - verify no direct routes conflict with tunnel traffic
- Proper monitoring is essential for production deployments
- Security groups and NACLs provide additional security layers
- Cost scales with data transfer, not connection time
- **IPSec restart often resolves Phase 2 negotiation issues**

## Next Steps
- Proceed to Lab 16: VPN Tunneling and IPSec Deep Dive
- Explore Transit Gateway VPN attachments
- Implement automated VPN deployment with CloudFormation
- Consider AWS VPN CloudHub for multiple site connectivity