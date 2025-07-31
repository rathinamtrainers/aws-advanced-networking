# Lab 16: VPN Tunneling and IPSec Deep Dive

## Lab Overview
This advanced lab provides an in-depth exploration of IPSec tunneling, VPN protocols, and security configurations for AWS Site-to-Site VPN connections. Students will configure advanced tunnel parameters, implement custom security policies, and troubleshoot complex VPN issues.

## Prerequisites
- AWS Account with VPN permissions
- Completed Labs 14 and 15 (VPN basics and HA)
- Strong understanding of IPSec, IKE, and cryptographic concepts
- Familiarity with network troubleshooting tools

## Lab Duration
120 minutes

## Learning Objectives
By the end of this lab, you will:
- Configure advanced IPSec tunnel parameters
- Implement custom IKE and IPSec security policies
- Troubleshoot VPN tunnel establishment issues
- Optimize tunnel performance and security
- Implement NAT traversal and DPD (Dead Peer Detection)
- Analyze VPN traffic with packet capture techniques

## Lab Architecture
```
On-Premises Router ←→ Internet ←→ AWS VPN Gateway
       |                              |
   [IPSec Tunnel]              [Security Policies]
       |                              |
   IKE Phase 1                  IKE Phase 1
   (Main/Aggressive Mode)       (Policy Matching)
       |                              |
   IKE Phase 2                  IKE Phase 2
   (Quick Mode)                 (SA Establishment)
       |                              |
   [Encrypted Traffic]         [Encrypted Traffic]
```

## Lab Steps

### Step 1: Create Advanced VPN Configuration
1. **Create Lab VPC with Custom Settings**
   - Navigate to **VPC Console**
   - Create VPC:
     - Name: `IPSec-Deep-Dive-VPC`
     - IPv4 CIDR: `10.50.0.0/16`
     - IPv6 CIDR: Enable IPv6 (for advanced testing)
     - DNS hostnames/resolution: Enabled

2. **Create Subnets with Specific Requirements**
   - Create multiple subnets for testing:
     - Tunnel Test Subnet: `10.50.1.0/24`
     - Performance Test Subnet: `10.50.2.0/24`
     - Security Test Subnet: `10.50.3.0/24`

3. **Create Virtual Private Gateway with Custom ASN**
   - Create VGW:
     - Name: `IPSec-Deep-Dive-VGW`
     - ASN: `64999` (custom ASN for testing)
     - Attach to VPC

### Step 2: Configure Advanced Customer Gateway
1. **Create Customer Gateway with Advanced Options**
   - Navigate to **Customer Gateways**
   - Create with specific parameters:
     - Name: `Advanced-Customer-Gateway`
     - BGP ASN: `65500`
     - IP Address: Your test public IP
     - Certificate ARN: (we'll create custom certificates)

2. **Generate Custom Certificates (Optional)**
   - Create custom PKI for certificate-based authentication
   - Upload certificates to AWS Certificate Manager
   - Configure customer gateway to use certificates

### Step 3: Create VPN with Custom Tunnel Options
1. **Create Advanced VPN Connection**
   - Navigate to **Site-to-Site VPN Connections**
   - Click **Create VPN Connection**
   - Basic Configuration:
     - Name: `IPSec-Deep-Dive-VPN`
     - Virtual Private Gateway: Select your VGW
     - Customer Gateway: Advanced-Customer-Gateway
     - Routing: Dynamic (BGP)

2. **Configure Advanced Tunnel 1 Options**
   - Expand Tunnel 1 Options:
     - Inside IPv4 CIDR: `169.254.100.0/30`
     - Pre-Shared Key: `MySecurePreSharedKey123!`
     - Advanced Options:
       - IKE Version: IKEv2 (more secure)
       - Phase 1 Encryption: AES256
       - Phase 1 Integrity: SHA256
       - Phase 1 DH Group: 14 (2048-bit MODP)
       - Phase 1 Lifetime: 28800 seconds
       - Phase 2 Encryption: AES256
       - Phase 2 Integrity: SHA256
       - Phase 2 DH Group: 14
       - Phase 2 Lifetime: 3600 seconds
       - Rekey Margin: 540 seconds
       - Rekey Fuzz: 100%
       - Dead Peer Detection: 30 seconds
       - DPD Timeout Action: restart

3. **Configure Advanced Tunnel 2 Options**
   - Similar configuration with different PSK:
     - Inside IPv4 CIDR: `169.254.101.0/30`
     - Pre-Shared Key: `MySecurePreSharedKey456!`
     - Same advanced cryptographic parameters

### Step 4: Set Up Advanced On-Premises Simulation
1. **Launch Strongswan Instance**
   - Create EC2 instance for advanced VPN testing:
     - AMI: Ubuntu 20.04 LTS
     - Instance Type: t3.medium
     - Security Group: Allow UDP 500, 4500, ESP protocol
     - User Data:
   ```bash
   #!/bin/bash
   apt update -y
   apt install -y strongswan strongswan-pki libcharon-extra-plugins
   apt install -y tcpdump wireshark-common tshark
   
   # Enable IP forwarding
   echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
   echo 'net.ipv4.conf.all.accept_redirects = 0' >> /etc/sysctl.conf
   echo 'net.ipv4.conf.all.send_redirects = 0' >> /etc/sysctl.conf
   sysctl -p
   ```

2. **Configure Strongswan with Advanced Options**
   - SSH to the instance and configure `/etc/ipsec.conf`:
   ```
   config setup
       charondebug="ike 2, knl 2, cfg 2, net 2, esp 2, dmn 2, mgr 2"
       strictcrlpolicy=no
       uniqueids=no

   conn aws-tunnel-1
       auto=start
       left=%defaultroute
       leftid=203.0.113.10
       leftsubnet=192.168.0.0/16
       right=<AWS_TUNNEL_1_IP>
       rightsubnet=10.50.0.0/16
       ike=aes256-sha256-modp2048!
       esp=aes256-sha256-modp2048!
       keyexchange=ikev2
       ikelifetime=28800s
       lifetime=3600s
       dpddelay=30s
       dpdtimeout=120s
       dpdaction=restart
       authby=secret
       type=tunnel
       compress=no
       mobike=no
   ```

3. **Configure IPSec Secrets**
   - Edit `/etc/ipsec.secrets`:
   ```
   203.0.113.10 <AWS_TUNNEL_1_IP> : PSK "MySecurePreSharedKey123!"
   203.0.113.10 <AWS_TUNNEL_2_IP> : PSK "MySecurePreSharedKey456!"
   ```

### Step 5: Advanced Troubleshooting Setup
1. **Enable Detailed Logging**
   - Configure strongswan logging:
   ```bash
   # Edit /etc/strongswan.d/charon-logging.conf
   charon {
       filelog {
           /var/log/charon.log {
               time_format = %b %e %T
               ike_name = yes
               append = no
               default = 2
               flush_line = yes
           }
       }
   }
   ```

2. **Set Up Packet Capture**
   - Configure tcpdump for VPN traffic analysis:
   ```bash
   # Capture IKE negotiations
   sudo tcpdump -i any -n -s0 -w /tmp/ike.pcap "udp port 500 or udp port 4500"
   
   # Capture ESP traffic
   sudo tcpdump -i any -n -s0 -w /tmp/esp.pcap "ip proto 50"
   ```

3. **Create Monitoring Scripts**
   - Create tunnel monitoring script:
   ```bash
   #!/bin/bash
   # tunnel_monitor.sh
   while true; do
       echo "=== $(date) ==="
       sudo ipsec status
       sudo ipsec statusall
       echo "Tunnel 1 ping test:"
       ping -c 1 10.50.1.1
       echo "Tunnel 2 ping test:"
       ping -c 1 10.50.2.1
       sleep 30
   done
   ```

### Step 6: Tunnel Establishment Analysis
1. **Start VPN Connection**
   - Start strongswan:
   ```bash
   sudo systemctl start strongswan
   sudo ipsec up aws-tunnel-1
   sudo ipsec up aws-tunnel-2
   ```

2. **Analyze IKE Phase 1 Negotiation**
   - Monitor IKE negotiation:
   ```bash
   sudo tail -f /var/log/charon.log
   ```
   - Look for:
     - Proposal selection
     - Authentication success
     - Key exchange completion

3. **Analyze IKE Phase 2 (Quick Mode)**
   - Monitor IPSec SA establishment:
   - Verify encryption/integrity algorithms
   - Check lifetime parameters
   - Confirm traffic selectors

### Step 7: Performance Testing and Optimization
1. **Bandwidth Testing**
   - Install iperf3 on both sides:
   ```bash
   # On AWS instance
   sudo yum install -y iperf3
   iperf3 -s
   
   # On on-premises simulation
   sudo apt install -y iperf3
   iperf3 -c 10.50.1.10 -t 60
   ```

2. **Latency and Jitter Testing**
   - Use ping with different packet sizes:
   ```bash
   # Test different MTU sizes
   ping -c 100 -s 1450 10.50.1.10
   ping -c 100 -s 1400 10.50.1.10
   ping -c 100 -s 1350 10.50.1.10
   ```

3. **MTU Discovery**
   - Find optimal MTU:
   ```bash
   # Test fragmentation
   ping -M do -s 1472 10.50.1.10  # Should work
   ping -M do -s 1473 10.50.1.10  # May fail
   ```

### Step 8: Security Analysis
1. **Cipher Suite Analysis**
   - Verify negotiated algorithms:
   ```bash
   sudo ipsec statusall | grep -A 5 -B 5 "ESP\|AES\|SHA"
   ```

2. **Certificate Validation** (if using certificates)
   - Check certificate chain:
   ```bash
   sudo ipsec listcerts
   sudo ipsec listcacerts
   ```

3. **Key Rotation Testing**
   - Monitor automatic key rotation:
   - Force manual rekey:
   ```bash
   sudo ipsec rekey aws-tunnel-1
   ```

### Step 9: Advanced Troubleshooting Scenarios
1. **Simulate Authentication Failures**
   - Intentionally misconfigure PSK
   - Analyze error messages
   - Document troubleshooting steps

2. **Test NAT Traversal**
   - Configure NAT on simulated router
   - Test NAT-T functionality
   - Analyze UDP encapsulation

3. **DPD Testing**
   - Simulate network interruption
   - Monitor DPD probe behavior
   - Test tunnel recovery

### Step 10: Packet Analysis with Wireshark
1. **Capture and Analyze IKE Traffic**
   - Transfer packet captures to analysis machine
   - Open in Wireshark
   - Analyze IKE message exchanges:
     - IKE_SA_INIT messages
     - IKE_AUTH messages
     - CREATE_CHILD_SA messages

2. **ESP Traffic Analysis**
   - Examine ESP packet structure
   - Verify encryption (payload should be encrypted)
   - Check SPI (Security Parameter Index) values

3. **Performance Analysis**
   - Measure encryption overhead
   - Analyze packet loss patterns
   - Identify bottlenecks

## Lab Validation
Complete these verification steps:

1. **Tunnel Establishment**
   - [ ] Both tunnels establish successfully
   - [ ] Correct algorithms negotiated
   - [ ] BGP sessions established
   - [ ] Traffic flows through tunnels

2. **Security Verification**
   - [ ] Strong encryption algorithms in use
   - [ ] DPD functioning correctly
   - [ ] Key rotation working
   - [ ] Authentication successful

3. **Performance Metrics**
   - [ ] Acceptable bandwidth achieved
   - [ ] Latency within expected range
   - [ ] MTU optimization completed
   - [ ] No significant packet loss

## Troubleshooting Common Issues

### IKE Phase 1 Failures
```bash
# Check IKE proposals
sudo ipsec listall | grep -A 10 "proposals:"

# Common issues:
# - Mismatched encryption algorithms
# - Incorrect PSK
# - Clock skew between peers
# - Firewall blocking UDP 500/4500
```

### IPSec SA Establishment Issues
```bash
# Check SA status
sudo ipsec statusall

# Common issues:
# - Traffic selector mismatch
# - Lifetime parameter conflicts
# - PFS (Perfect Forward Secrecy) mismatch
```

### Performance Problems
```bash
# Check for fragmentation
sudo tcpdump -i any -n "icmp and icmp[icmptype] == 3 and icmp[icmpcode] == 4"

# Common issues:
# - MTU size too large
# - TCP MSS clamping needed
# - Encryption overhead
```

### BGP Issues Over VPN
```bash
# Check BGP session
vtysh -c "show ip bgp summary"

# Common issues:
# - BGP over IPSec latency
# - MTU discovery problems
# - Keepalive timer mismatches
```

## Advanced Optimization Techniques

### 1. Hardware Acceleration
- Use instances with hardware crypto acceleration
- Configure AES-NI support
- Monitor CPU utilization during encryption

### 2. Tunnel Bonding
- Configure multiple tunnels for load balancing
- Implement ECMP (Equal Cost Multi-Path)
- Use different tunnel endpoints for redundancy

### 3. Application-Specific Optimization
```bash
# TCP optimization over VPN
echo 'net.core.rmem_max = 134217728' >> /etc/sysctl.conf
echo 'net.core.wmem_max = 134217728' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_rmem = 4096 65536 134217728' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_wmem = 4096 65536 134217728' >> /etc/sysctl.conf
sysctl -p
```

## Security Best Practices

### 1. Strong Authentication
- Use certificates instead of PSK when possible
- Implement certificate lifecycle management
- Use strong entropy for key generation

### 2. Algorithm Selection
- Always use AES-256 for encryption
- Use SHA-256 or higher for integrity
- Prefer DH group 14 or higher
- Enable Perfect Forward Secrecy

### 3. Monitoring and Alerting
- Monitor for authentication failures
- Alert on tunnel down events
- Track unusual traffic patterns
- Log all VPN-related events

## Clean Up
To avoid ongoing charges:

1. **Stop VPN Services**
   ```bash
   sudo ipsec down aws-tunnel-1
   sudo ipsec down aws-tunnel-2
   sudo systemctl stop strongswan
   ```

2. **Delete AWS Resources**
   - Delete VPN connections
   - Remove customer gateways
   - Detach and delete VGW
   - Terminate EC2 instances

3. **Clean Up Packet Captures**
   - Remove large packet capture files
   - Clear log files

## Key Takeaways
- IPSec configuration requires precise parameter matching
- Strong cryptographic algorithms are essential for security
- Performance optimization requires careful MTU and TCP tuning
- Monitoring and logging are crucial for troubleshooting
- Packet analysis provides deep insights into VPN behavior
- Security best practices must be followed consistently

## Next Steps
- Proceed to Lab 17: VPN vs Direct Connect Comparison
- Explore certificate-based authentication in production
- Implement automated VPN monitoring and alerting
- Consider hardware VPN appliances for high-throughput requirements