# Topic 15: VPN on AWS - Site-to-Site VPN Explained

## Prerequisites
- Completed Foundation Module (Topics 1-10)
- Understanding of IPSec VPN concepts
- Knowledge of routing protocols and tunneling
- Basic familiarity with network security concepts

## Learning Objectives
By the end of this topic, you will be able to:
- Understand AWS Site-to-Site VPN architecture and components
- Configure VPN connections with high availability
- Implement routing for VPN connectivity
- Compare VPN with other AWS connectivity options

## Theory

### AWS Site-to-Site VPN Overview

#### Definition
AWS Site-to-Site VPN creates a secure connection between your on-premises network and your Amazon VPC over the internet using encrypted IPSec tunnels.

#### Key Characteristics
- **Encrypted Tunnels**: IPSec encryption for secure data transmission
- **High Availability**: Two tunnels per VPN connection for redundancy
- **Dynamic Routing**: BGP support for automatic route exchange (or static routing for simpler setups)
- **Quick Setup**: Can be established in minutes vs weeks for Direct Connect
- **Cost Effective**: Lower setup costs, pay-per-use model
- **DH Group Requirements**: AWS VPN requires DH Group 2 (modp1024) support in customer equipment

### VPN Connection Components

#### 1. Virtual Private Gateway (VGW)
- **AWS Side**: VPN concentrator on the AWS side
- **Managed Service**: Fully managed by AWS
- **High Availability**: Redundant across multiple AZs
- **BGP Support**: ASN configuration for dynamic routing

#### 2. Customer Gateway
- **Customer Side**: Physical device or software appliance on customer premises
- **Public IP**: Must have static public IP address (Elastic IP recommended for EC2-based solutions)
- **IPSec Support**: Must support IPSec tunneling with specific AWS requirements:
  - **IKE Version**: IKEv1 (default for AWS VPN)
  - **DH Group**: Must support DH Group 2 (modp1024)
  - **Encryption**: AES-128 or AES-256
  - **Hash**: SHA-1 or SHA-256
- **BGP Capability**: Optional but recommended for dynamic routing
- **Software Recommendations**: 
  - **StrongSwan** on Ubuntu (reliable DH Group 2 support)
  - **Cisco ASA, ISR** (hardware solutions)
  - **Avoid**: Libreswan on modern distributions (DH Group 2 compatibility issues)

#### 3. VPN Connection
- **Tunnel Pair**: Two IPSec tunnels for high availability
- **Two-Phase Process**:
  - **Phase 1 (IKE)**: Establishes secure channel for negotiation
  - **Phase 2 (ESP)**: Creates Child SAs for actual data transmission
- **Encryption**: AES-128, AES-256 encryption options
- **Authentication**: Pre-shared keys (PSK) or certificates
- **Routing Options**:
  - **Static**: Manual route configuration, simpler for small environments
  - **Dynamic (BGP)**: Automatic route exchange, better for complex topologies
- **Common Issues**: Child SA establishment may require IPSec service restart

### VPN Architecture Components

#### Physical Architecture
```
Customer Premises → Internet → AWS VPC
    [Customer Gateway] → [VPN Tunnels] → [Virtual Private Gateway]
         ↓                    ↓                    ↓
    [On-Prem Network] → [Encrypted Traffic] → [VPC Resources]
```

#### Logical Architecture
```
On-Premises Network (10.1.0.0/16)
         ↓
[Customer Gateway Device]
         ↓
[Internet - IPSec Tunnels]
    Tunnel 1: 169.254.10.0/30
    Tunnel 2: 169.254.11.0/30
         ↓
[Virtual Private Gateway]
         ↓
AWS VPC (10.0.0.0/16)
```

### VPN Tunnel Configuration

#### Tunnel Endpoints
- **AWS Endpoints**: Two public IP addresses provided by AWS
- **Customer Endpoint**: Single public IP address from customer
- **Tunnel IPs**: 169.254.x.x/30 subnets for tunnel interfaces
- **Redundancy**: Active/standby or active/active configuration

#### IPSec Parameters
```
Encryption Algorithm: AES-128 or AES-256
Authentication Algorithm: SHA-1 or SHA-256
Perfect Forward Secrecy (PFS): Group 2, 14-24
DPD (Dead Peer Detection): Enabled
NAT Traversal: Enabled if behind NAT
```

#### BGP Configuration
- **AWS ASN**: Configurable (default 64512)
- **Customer ASN**: Customer specified (private or public)
- **Hold Time**: 30 seconds default
- **Keep Alive**: 10 seconds default

### Routing Options

#### 1. Static Routing
- **Configuration**: Manual route configuration
- **Use Case**: Simple networks with few routes
- **Limitations**: No automatic failover, manual updates required
- **Best For**: Small, stable networks

#### 2. Dynamic Routing (BGP)
- **Configuration**: Automatic route exchange via BGP
- **Use Case**: Complex networks with many routes
- **Benefits**: Automatic failover, route summarization
- **Best For**: Enterprise networks, multiple connections

### High Availability Design

#### Dual Tunnel Architecture
```
Customer Gateway
    ├── Tunnel 1 (Primary) → AWS Endpoint 1
    └── Tunnel 2 (Backup) → AWS Endpoint 2
```

#### Dual Customer Gateway
```
Primary Customer Gateway → VPN Connection 1 → VGW
Backup Customer Gateway → VPN Connection 2 → VGW
```

#### Multi-AZ VGW
- VGW automatically spans multiple AZs
- Provides protection against AZ failures
- No additional configuration required

## Lab Exercise: Complete Site-to-Site VPN Setup

### Lab Duration: 180 minutes

### Step 1: Plan VPN Implementation
**Objective**: Design VPN architecture for hybrid connectivity

**VPN Planning Assessment**:
```bash
# Create VPN planning script
cat << 'EOF' > vpn-planning-assessment.sh
#!/bin/bash

echo "=== AWS Site-to-Site VPN Planning Assessment ==="
echo ""

echo "1. NETWORK REQUIREMENTS"
echo "   On-premises network CIDR: _______"
echo "   AWS VPC CIDR: _______"
echo "   Expected bandwidth: _______ Mbps"
echo "   Latency requirements: _______ ms"
echo "   High availability needed: Yes/No"
echo ""

echo "2. CUSTOMER GATEWAY REQUIREMENTS"
echo "   Device type: Hardware/Software/Cloud"
echo "   Public IP address: _______"
echo "   IPSec support: Yes/No"
echo "   BGP support: Yes/No"
echo "   Redundant devices: Yes/No"
echo ""

echo "3. ROUTING REQUIREMENTS"
echo "   Routing type: Static/Dynamic (BGP)"
echo "   Number of routes: _______"
echo "   Route summarization needed: Yes/No"
echo "   Failover requirements: _______"
echo ""

echo "4. SECURITY REQUIREMENTS"
echo "   Encryption level: AES-128/AES-256"
echo "   Authentication method: PSK/Certificates"
echo "   DPD required: Yes/No"
echo "   NAT traversal needed: Yes/No"
echo ""

echo "5. TRAFFIC PATTERNS"
echo "   Primary traffic types: _______"
echo "   Peak usage times: _______"
echo "   Data transfer volume: _______ GB/month"
echo "   Critical applications: _______"

EOF

chmod +x vpn-planning-assessment.sh
./vpn-planning-assessment.sh
```

### Step 2: Create Virtual Private Gateway
**Objective**: Set up AWS-side VPN endpoint

```bash
# Create Virtual Private Gateway
cat << 'EOF' > create-vpn-gateway.sh
#!/bin/bash

echo "=== Creating Virtual Private Gateway ==="
echo ""

VGW_NAME="Production-VGW"
AWS_ASN="64512"  # Private ASN for AWS side

echo "Creating Virtual Private Gateway..."
echo "Name: $VGW_NAME"
echo "AWS ASN: $AWS_ASN"
echo ""

# Create VGW
VGW_ID=$(aws ec2 create-vpn-gateway \
    --type ipsec.1 \
    --amazon-side-asn $AWS_ASN \
    --tag-specifications "ResourceType=vpn-gateway,Tags=[{Key=Name,Value=$VGW_NAME}]" \
    --query 'VpnGateway.VpnGatewayId' \
    --output text)

echo "✅ Virtual Private Gateway created: $VGW_ID"
echo ""

# Wait for VGW to be available
echo "Waiting for VGW to become available..."
aws ec2 wait vpn-gateway-available --vpn-gateway-ids $VGW_ID

echo "✅ VGW is now available"
echo ""

# Attach VGW to VPC
VPC_ID="vpc-0123456789abcdef0"  # Replace with your VPC ID
echo "Attaching VGW to VPC: $VPC_ID"

aws ec2 attach-vpn-gateway \
    --vpn-gateway-id $VGW_ID \
    --vpc-id $VPC_ID

echo "Waiting for VGW attachment..."
aws ec2 wait vpn-gateway-attached --vpn-gateway-ids $VGW_ID

echo "✅ VGW attached to VPC successfully"

# Store VGW ID for later use
echo "export VGW_ID=$VGW_ID" > vpn-config.env
echo "VGW ID saved to vpn-config.env"

EOF

chmod +x create-vpn-gateway.sh
./create-vpn-gateway.sh
```

### Step 3: Create Customer Gateway
**Objective**: Define customer-side VPN endpoint

```bash
# Create Customer Gateway
cat << 'EOF' > create-customer-gateway.sh
#!/bin/bash

source vpn-config.env

echo "=== Creating Customer Gateway ==="
echo ""

CGW_NAME="HQ-Customer-Gateway"
CUSTOMER_IP="203.0.113.10"  # Replace with your public IP
CUSTOMER_ASN="65001"        # Private ASN for customer side
DEVICE_NAME="Cisco-ASR1001"

echo "Creating Customer Gateway..."
echo "Name: $CGW_NAME"
echo "Public IP: $CUSTOMER_IP"
echo "Customer ASN: $CUSTOMER_ASN"
echo "Device: $DEVICE_NAME"
echo ""

# Create Customer Gateway
CGW_ID=$(aws ec2 create-customer-gateway \
    --type ipsec.1 \
    --public-ip $CUSTOMER_IP \
    --bgp-asn $CUSTOMER_ASN \
    --device-name "$DEVICE_NAME" \
    --tag-specifications "ResourceType=customer-gateway,Tags=[{Key=Name,Value=$CGW_NAME}]" \
    --query 'CustomerGateway.CustomerGatewayId' \
    --output text)

echo "✅ Customer Gateway created: $CGW_ID"
echo ""

# Display Customer Gateway details
aws ec2 describe-customer-gateways \
    --customer-gateway-ids $CGW_ID \
    --query 'CustomerGateways[0].[CustomerGatewayId,IpAddress,BgpAsn,State]' \
    --output table

# Store CGW ID for later use
echo "export CGW_ID=$CGW_ID" >> vpn-config.env
echo "CGW ID saved to vpn-config.env"

EOF

chmod +x create-customer-gateway.sh
./create-customer-gateway.sh
```

### Step 4: Create VPN Connection
**Objective**: Establish IPSec tunnels between gateways

```bash
# Create VPN Connection
cat << 'EOF' > create-vpn-connection.sh
#!/bin/bash

source vpn-config.env

echo "=== Creating VPN Connection ==="
echo ""

VPN_NAME="HQ-to-AWS-VPN"
ROUTING_TYPE="dynamic"  # or "static"

echo "Creating VPN Connection..."
echo "Name: $VPN_NAME"
echo "VGW ID: $VGW_ID"
echo "CGW ID: $CGW_ID"
echo "Routing: $ROUTING_TYPE"
echo ""

# Create VPN Connection
VPN_ID=$(aws ec2 create-vpn-connection \
    --type ipsec.1 \
    --customer-gateway-id $CGW_ID \
    --vpn-gateway-id $VGW_ID \
    --options StaticRoutesOnly=false \
    --tag-specifications "ResourceType=vpn-connection,Tags=[{Key=Name,Value=$VPN_NAME}]" \
    --query 'VpnConnection.VpnConnectionId' \
    --output text)

echo "✅ VPN Connection created: $VPN_ID"
echo ""

# Wait for VPN connection to be available
echo "Waiting for VPN connection to become available..."
aws ec2 wait vpn-connection-available --vpn-connection-ids $VPN_ID

echo "✅ VPN Connection is now available"
echo ""

# Get VPN connection details
echo "VPN Connection Details:"
aws ec2 describe-vpn-connections \
    --vpn-connection-ids $VPN_ID \
    --query 'VpnConnections[0].[VpnConnectionId,State,Type,Routes[0].DestinationCidrBlock]' \
    --output table

# Store VPN ID for later use
echo "export VPN_ID=$VPN_ID" >> vpn-config.env
echo "VPN ID saved to vpn-config.env"

EOF

chmod +x create-vpn-connection.sh
./create-vpn-connection.sh
```

### Step 5: Download VPN Configuration
**Objective**: Get configuration templates for customer gateway device

```bash
# Download VPN configuration
cat << 'EOF' > download-vpn-config.sh
#!/bin/bash

source vpn-config.env

echo "=== Downloading VPN Configuration ==="
echo ""

VENDOR="cisco"      # cisco, juniper, generic, etc.
PLATFORM="ios"      # ios, ios-xe, junos, etc.
SOFTWARE="12.4+"    # software version

echo "Downloading configuration for:"
echo "Vendor: $VENDOR"
echo "Platform: $PLATFORM"
echo "Software: $SOFTWARE"
echo ""

# Download configuration file
aws ec2 describe-vpn-connections \
    --vpn-connection-ids $VPN_ID \
    --query 'VpnConnections[0].CustomerGatewayConfiguration' \
    --output text > vpn-configuration.txt

echo "✅ VPN configuration downloaded to vpn-configuration.txt"
echo ""

# Parse key information from configuration
echo "Key Configuration Details:"
echo "=========================="

# Extract tunnel information
grep -E "IPSec Tunnel|outside_interface|tunnel_interface|bgp" vpn-configuration.txt | head -20

echo ""
echo "Full configuration saved in vpn-configuration.txt"
echo "Apply this configuration to your customer gateway device"

EOF

chmod +x download-vpn-config.sh
./download-vpn-config.sh
```

### Step 6: Configure Customer Gateway Device
**Objective**: Apply VPN configuration to customer equipment

```bash
# Generate customer gateway configuration templates
cat << 'EOF' > generate-cgw-config.sh
#!/bin/bash

echo "=== Customer Gateway Configuration Templates ==="
echo ""

echo "1. CISCO IOS/IOS-XE CONFIGURATION:"
cat << 'CISCO_CONFIG'
! Interface configuration
interface Tunnel1
 description VPN Tunnel 1 to AWS
 ip address 169.254.44.10 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 52.1.1.1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile AWS-IPSEC-PROFILE-1
 ip tcp adjust-mss 1436
 no shutdown

interface Tunnel2
 description VPN Tunnel 2 to AWS
 ip address 169.254.44.14 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 52.1.1.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile AWS-IPSEC-PROFILE-2
 ip tcp adjust-mss 1436
 no shutdown

! IPSec configuration
crypto isakmp policy 200
 encryption aes 128
 authentication pre-share
 group 2
 lifetime 28800
 hash sha

crypto isakmp key YOUR_PSK_HERE address 52.1.1.1
crypto isakmp key YOUR_PSK_HERE address 52.1.1.2

crypto ipsec transform-set AWS-TRANSFORM-SET esp-aes esp-sha-hmac
 mode tunnel

crypto ipsec profile AWS-IPSEC-PROFILE-1
 set transform-set AWS-TRANSFORM-SET
 set pfs group2

crypto ipsec profile AWS-IPSEC-PROFILE-2
 set transform-set AWS-TRANSFORM-SET
 set pfs group2

! BGP configuration
router bgp 65001
 bgp log-neighbor-changes
 neighbor 169.254.44.9 remote-as 64512
 neighbor 169.254.44.9 timers 10 30
 neighbor 169.254.44.9 activate
 neighbor 169.254.44.13 remote-as 64512
 neighbor 169.254.44.13 timers 10 30
 neighbor 169.254.44.13 activate
 network 10.1.0.0 mask 255.255.0.0

! Static routes (if not using BGP)
ip route 10.0.0.0 255.255.0.0 Tunnel1
ip route 10.0.0.0 255.255.0.0 Tunnel2 backup

CISCO_CONFIG

echo ""
echo "2. JUNIPER JUNOS CONFIGURATION:"
cat << 'JUNOS_CONFIG'
# Interface configuration
set interfaces st0 unit 1 description "VPN Tunnel 1 to AWS"
set interfaces st0 unit 1 family inet address 169.254.44.10/30
set interfaces st0 unit 2 description "VPN Tunnel 2 to AWS"  
set interfaces st0 unit 2 family inet address 169.254.44.14/30

# IPSec configuration
set security ike policy AWS-IKE-POLICY mode main
set security ike policy AWS-IKE-POLICY proposal-set standard
set security ike policy AWS-IKE-POLICY pre-shared-key ascii-text "YOUR_PSK_HERE"

set security ike gateway AWS-GATEWAY-1 ike-policy AWS-IKE-POLICY
set security ike gateway AWS-GATEWAY-1 address 52.1.1.1
set security ike gateway AWS-GATEWAY-1 external-interface ge-0/0/0

set security ike gateway AWS-GATEWAY-2 ike-policy AWS-IKE-POLICY
set security ike gateway AWS-GATEWAY-2 address 52.1.1.2
set security ike gateway AWS-GATEWAY-2 external-interface ge-0/0/0

set security ipsec policy AWS-IPSEC-POLICY perfect-forward-secrecy keys group2
set security ipsec policy AWS-IPSEC-POLICY proposal-set standard

set security ipsec vpn AWS-VPN-1 bind-interface st0.1
set security ipsec vpn AWS-VPN-1 ike gateway AWS-GATEWAY-1
set security ipsec vpn AWS-VPN-1 ike ipsec-policy AWS-IPSEC-POLICY

set security ipsec vpn AWS-VPN-2 bind-interface st0.2
set security ipsec vpn AWS-VPN-2 ike gateway AWS-GATEWAY-2
set security ipsec vpn AWS-VPN-2 ike ipsec-policy AWS-IPSEC-POLICY

# BGP configuration
set protocols bgp group aws type external
set protocols bgp group aws peer-as 64512
set protocols bgp group aws neighbor 169.254.44.9
set protocols bgp group aws neighbor 169.254.44.13
set protocols bgp group aws export ADVERTISE-ROUTES

set policy-options policy-statement ADVERTISE-ROUTES term 1 from route-filter 10.1.0.0/16 exact
set policy-options policy-statement ADVERTISE-ROUTES term 1 then accept

set routing-options autonomous-system 65001

JUNOS_CONFIG

echo ""
echo "3. CONFIGURATION VERIFICATION CHECKLIST:"
echo "   □ IPSec tunnels established"
echo "   □ BGP sessions up (if using dynamic routing)"
echo "   □ Routes advertised and received"
echo "   □ Traffic flowing through tunnels"
echo "   □ Redundancy tested"

EOF

chmod +x generate-cgw-config.sh
./generate-cgw-config.sh
```

### Step 7: Configure Route Propagation
**Objective**: Enable automatic route learning in VPC

```bash
# Configure route propagation
cat << 'EOF' > configure-route-propagation.sh
#!/bin/bash

source vpn-config.env

echo "=== Configuring Route Propagation ==="
echo ""

VPC_ID="vpc-0123456789abcdef0"  # Replace with your VPC ID

echo "Enabling route propagation for VGW: $VGW_ID"
echo "VPC: $VPC_ID"
echo ""

# Get all route tables in the VPC
ROUTE_TABLES=$(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query 'RouteTables[*].RouteTableId' \
    --output text)

echo "Found route tables:"
for RT_ID in $ROUTE_TABLES; do
    echo "  - $RT_ID"
done
echo ""

# Enable route propagation for each route table
for RT_ID in $ROUTE_TABLES; do
    echo "Enabling route propagation on $RT_ID..."
    
    aws ec2 enable-vgw-route-propagation \
        --route-table-id $RT_ID \
        --gateway-id $VGW_ID
    
    if [ $? -eq 0 ]; then
        echo "✅ Route propagation enabled on $RT_ID"
    else
        echo "❌ Failed to enable route propagation on $RT_ID"
    fi
done

echo ""
echo "✅ Route propagation configuration complete"
echo ""

echo "Verifying route propagation status..."
for RT_ID in $ROUTE_TABLES; do
    echo "Route table $RT_ID:"
    aws ec2 describe-route-tables \
        --route-table-ids $RT_ID \
        --query 'RouteTables[0].PropagatingVgws[*].[GatewayId,State]' \
        --output table
done

EOF

chmod +x configure-route-propagation.sh
./configure-route-propagation.sh
```

### Step 8: Test VPN Connectivity
**Objective**: Validate VPN functionality and performance

```bash
# Create VPN testing script
cat << 'EOF' > test-vpn-connectivity.sh
#!/bin/bash

source vpn-config.env

echo "=== Testing VPN Connectivity ==="
echo ""

echo "1. VPN CONNECTION STATUS"
echo "Checking VPN connection state..."

aws ec2 describe-vpn-connections \
    --vpn-connection-ids $VPN_ID \
    --query 'VpnConnections[0].[State,VgwTelemetry[*].[Status,StatusMessage]]' \
    --output table

echo ""

echo "2. TUNNEL STATUS"
echo "Checking individual tunnel status..."

aws ec2 describe-vpn-connections \
    --vpn-connection-ids $VPN_ID \
    --query 'VpnConnections[0].VgwTelemetry[*].[OutsideIpAddress,Status,LastStatusChange,StatusMessage]' \
    --output table

echo ""

echo "3. BGP SESSION STATUS (if using dynamic routing)"
echo "Check BGP session status on customer gateway:"
echo ""
echo "Cisco IOS commands:"
echo "  show ip bgp summary"
echo "  show ip bgp neighbors"
echo "  show ip route bgp"
echo ""
echo "Juniper commands:"
echo "  show bgp summary"
echo "  show bgp neighbor"
echo "  show route protocol bgp"
echo ""

echo "4. CONNECTIVITY TESTS"
echo "Test connectivity from on-premises to AWS:"
echo ""

# Example test targets in AWS VPC
AWS_TARGETS=("10.0.1.10" "10.0.2.20")

for target in "${AWS_TARGETS[@]}"; do
    echo "Testing connectivity to $target:"
    echo "  ping $target"
    echo "  telnet $target 22"
    echo "  traceroute $target"
    echo ""
done

echo "5. PERFORMANCE TESTING"
cat << 'PERF_TESTS'
Bandwidth testing with iperf3:

On AWS instance:
sudo yum install iperf3 -y
iperf3 -s

From on-premises:
iperf3 -c <aws-instance-ip> -t 60

Expected throughput:
- Single tunnel: ~1.25 Gbps maximum
- Load balanced: Higher with multiple connections
- Latency: Variable based on internet path

PERF_TESTS

echo ""
echo "6. ROUTE VERIFICATION"
echo "Verify routes are learned via VPN:"
echo ""
echo "AWS VPC route tables:"
aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=vpc-0123456789abcdef0" \
    --query 'RouteTables[*].[RouteTableId,Routes[?GatewayId==`'$VGW_ID'`].[DestinationCidrBlock,State]]' \
    --output table

EOF

chmod +x test-vpn-connectivity.sh
./test-vpn-connectivity.sh
```

### Step 9: Configure High Availability
**Objective**: Implement redundant VPN connectivity

```bash
# Configure HA VPN setup
cat << 'EOF' > configure-vpn-ha.sh
#!/bin/bash

echo "=== Configuring High Availability VPN ==="
echo ""

echo "HA VPN Options:"
echo "1. Dual Tunnel (Single CGW)"
echo "2. Dual Customer Gateway"
echo "3. VPN + Direct Connect"
echo ""

echo "OPTION 1: DUAL TUNNEL CONFIGURATION"
echo "Already implemented - each VPN connection has 2 tunnels"
echo "Configure equal-cost load balancing on customer gateway:"
echo ""

cat << 'DUAL_TUNNEL_CONFIG'
Cisco IOS configuration for tunnel load balancing:
router bgp 65001
 maximum-paths 2
 neighbor 169.254.44.9 weight 100
 neighbor 169.254.44.13 weight 100

Juniper configuration:
set routing-options forwarding-table export LOAD-BALANCE
set policy-options policy-statement LOAD-BALANCE then load-balance per-packet

DUAL_TUNNEL_CONFIG

echo ""
echo "OPTION 2: DUAL CUSTOMER GATEWAY"
echo "Create second customer gateway for full redundancy:"
echo ""

# Create second customer gateway
BACKUP_CGW_IP="203.0.113.11"  # Backup public IP
BACKUP_CGW_NAME="Backup-Customer-Gateway"

echo "Creating backup customer gateway..."
echo "Primary CGW IP: 203.0.113.10"
echo "Backup CGW IP: $BACKUP_CGW_IP"

# This would create a second CGW and VPN connection
echo "Commands to create backup setup:"
cat << 'BACKUP_COMMANDS'
# Create backup customer gateway
aws ec2 create-customer-gateway \
    --type ipsec.1 \
    --public-ip 203.0.113.11 \
    --bgp-asn 65001 \
    --tag-specifications 'ResourceType=customer-gateway,Tags=[{Key=Name,Value=Backup-Customer-Gateway}]'

# Create backup VPN connection
aws ec2 create-vpn-connection \
    --type ipsec.1 \
    --customer-gateway-id <backup-cgw-id> \
    --vpn-gateway-id <vgw-id> \
    --options StaticRoutesOnly=false

BACKUP_COMMANDS

echo ""
echo "OPTION 3: VPN + DIRECT CONNECT BACKUP"
echo "Use VPN as backup for Direct Connect:"
echo ""

cat << 'HYBRID_CONFIG'
BGP configuration for DX + VPN backup:
- Direct Connect: Higher local preference (200)
- VPN: Lower local preference (100)
- Automatic failover when DX fails

router bgp 65001
 neighbor <dx-peer> route-map SET-DX-PREF in
 neighbor <vpn-peer> route-map SET-VPN-PREF in

route-map SET-DX-PREF permit 10
 set local-preference 200

route-map SET-VPN-PREF permit 10
 set local-preference 100

HYBRID_CONFIG

echo ""
echo "HA MONITORING AND ALERTING"
echo "Set up monitoring for:"
echo "- VPN tunnel status"
echo "- BGP session status"
echo "- Traffic patterns"
echo "- Failover events"

EOF

chmod +x configure-vpn-ha.sh
./configure-vpn-ha.sh
```

### Step 10: Monitor and Troubleshoot VPN
**Objective**: Set up monitoring and resolve common issues

```bash
# VPN monitoring and troubleshooting
cat << 'EOF' > vpn-monitoring-troubleshooting.sh
#!/bin/bash

echo "=== VPN Monitoring and Troubleshooting ==="
echo ""

echo "1. CLOUDWATCH MONITORING"
echo "Set up CloudWatch alarms for VPN metrics:"
echo ""

# Create CloudWatch alarm for tunnel state
aws cloudwatch put-metric-alarm \
    --alarm-name "VPN-Tunnel-State" \
    --alarm-description "Alert when VPN tunnel goes down" \
    --metric-name TunnelState \
    --namespace AWS/VPN \
    --statistic Maximum \
    --period 60 \
    --threshold 0 \
    --comparison-operator LessThanThreshold \
    --dimensions Name=VpnId,Value=$VPN_ID Name=TunnelIpAddress,Value=52.1.1.1 \
    --evaluation-periods 2

echo "✅ CloudWatch alarm created for tunnel monitoring"
echo ""

echo "2. COMMON TROUBLESHOOTING SCENARIOS"
echo ""

echo "ISSUE: VPN tunnels not establishing"
echo "Symptoms: Tunnel state shows 'DOWN'"
echo "Troubleshooting steps:"
echo "a) Check customer gateway public IP accessibility"
echo "b) Verify IPSec parameters match AWS configuration"
echo "c) Check firewall rules for IPSec traffic (UDP 500, 4500)"
echo "d) Validate pre-shared keys (case-sensitive)"
echo "e) Review customer gateway logs"
echo "f) **CRITICAL**: Verify DH Group 2 (modp1024) support in IPSec software"
echo "g) Use StrongSwan on Ubuntu instead of Libreswan for reliable compatibility"
echo "h) Ensure IKEv1 is configured (AWS VPN default)"
echo ""

echo "ISSUE: BGP sessions not establishing"
echo "Symptoms: Tunnels UP but no route exchange"
echo "Troubleshooting steps:"
echo "a) Verify BGP ASN configuration"
echo "b) Check tunnel IP addressing"
echo "c) Validate BGP timers"
echo "d) Review BGP authentication"
echo "e) Check route filters and prefix lists"
echo "f) **ALTERNATIVE**: Consider static routing for simpler configurations"
echo "g) Static routing eliminates BGP complexity and is suitable for most lab/small environments"
echo ""
echo ""
echo "ISSUE: Child SAs (Phase 2) not establishing"
echo "Symptoms: IKE (Phase 1) works but ESP tunnels fail, ping doesn't work"
echo "Troubleshooting steps:"
echo "a) Check Child SA status: sudo ipsec status"
echo "b) Restart IPSec service: sudo ipsec restart"
echo "c) Verify XFRM policies: ip xfrm policy show"
echo "d) Check for routing conflicts bypassing IPSec"
echo "e) Remove conflicting routes: ip route del <conflicting_route>"
echo "f) Monitor traffic: sudo tcpdump -i any icmp"
echo ""

echo "ISSUE: Asymmetric routing"
echo "Symptoms: Traffic works in one direction only"
echo "Troubleshooting steps:"
echo "a) Check route propagation in VPC route tables"
echo "b) Verify on-premises routing to AWS"
echo "c) Review BGP route preferences"
echo "d) Check security group and NACL rules"
echo ""

echo "3. DIAGNOSTIC COMMANDS"
echo ""
echo "AWS CLI diagnostics:"
echo "aws ec2 describe-vpn-connections --vpn-connection-ids $VPN_ID"
echo "aws logs describe-log-streams --log-group-name /aws/vpn/logs"
echo ""

echo "Customer gateway diagnostics:"
cat << 'CGW_DIAGNOSTICS'

Cisco IOS:
show crypto isakmp sa
show crypto ipsec sa
show ip bgp summary
show ip bgp neighbors
debug crypto isakmp
debug crypto ipsec

StrongSwan (Ubuntu):
sudo ipsec statusall
sudo ipsec status
ip xfrm state
ip xfrm policy
sudo ipsec restart
sudo systemctl status strongswan-starter
journalctl -u strongswan-starter -f

Juniper:
show security ike security-associations
show security ipsec security-associations
show bgp summary
show bgp neighbor
set traceoptions file ike-trace
set traceoptions flag all

CGW_DIAGNOSTICS

echo ""
echo "4. PERFORMANCE OPTIMIZATION"
echo ""
echo "Optimization techniques:"
echo "- Enable TCP MSS clamping (1436 bytes)"
echo "- Use equal-cost load balancing"
echo "- Optimize BGP timers for faster convergence"
echo "- Consider multiple VPN connections for higher bandwidth"
echo "- Implement proper QoS policies"

EOF

chmod +x vpn-monitoring-troubleshooting.sh
./vpn-monitoring-troubleshooting.sh
```

## VPN Performance Characteristics

### Bandwidth and Throughput
- **Maximum Throughput**: ~1.25 Gbps per tunnel
- **Aggregate Bandwidth**: Up to 2.5 Gbps with dual tunnels
- **Factors Affecting Performance**:
  - Internet path quality
  - Customer gateway hardware
  - Encryption overhead
  - TCP window scaling

### Latency Considerations
- **Base Latency**: Internet path latency
- **Encryption Overhead**: ~1-5ms additional
- **Jitter**: Variable based on internet conditions
- **Optimization**: Choose geographically close AWS regions

## VPN vs Other Connectivity Options

| Factor | Site-to-Site VPN | Direct Connect | Internet Gateway |
|--------|------------------|----------------|------------------|
| **Setup Time** | Minutes | Weeks/Months | Immediate |
| **Bandwidth** | Up to 1.25 Gbps | Up to 100 Gbps | Variable |
| **Latency** | Variable (internet) | Consistent, low | Variable |
| **Cost** | Low setup, transfer charges | High setup, lower transfer | Transfer only |
| **Security** | Encrypted tunnels | Private connection | Public internet |
| **Reliability** | Internet dependent | 99.9% SLA | Best effort |

## Best Practices Summary

### Design Best Practices
- **High Availability**: Use multiple tunnels and customer gateways
- **Routing**: Prefer BGP over static routing for complex networks
- **Security**: Use strong encryption and proper authentication
- **Monitoring**: Set up comprehensive monitoring and alerting

### Implementation Best Practices
- **Testing**: Thoroughly test before production deployment
- **Documentation**: Document configuration and procedures
- **Backup**: Maintain backup connectivity options
- **Training**: Ensure operations team understands VPN management

### Operational Best Practices
- **Monitoring**: Continuous monitoring of tunnel and BGP status
- **Maintenance**: Regular review and updates of configurations
- **Troubleshooting**: Maintain troubleshooting procedures and contacts
- **Performance**: Regular performance testing and optimization

## Key Takeaways
- Site-to-Site VPN provides quick, secure connectivity to AWS
- High availability requires multiple tunnels and proper BGP configuration
- BGP routing provides automatic failover and optimal path selection
- Performance depends on internet path quality and customer gateway capabilities
- Comprehensive monitoring is essential for reliable VPN operations
- VPN works well as backup for Direct Connect or primary connectivity for smaller workloads

## Additional Resources
- [AWS VPN User Guide](https://docs.aws.amazon.com/vpn/latest/s2svpn/)
- [VPN Configuration Examples](https://docs.aws.amazon.com/vpn/latest/s2svpn/Examples.html)
- [VPN Troubleshooting Guide](https://docs.aws.amazon.com/vpn/latest/s2svpn/Troubleshooting.html)

## Exam Tips
- VPN connections automatically create two tunnels for high availability
- BGP ASN 7224 is used by AWS for VPN connections
- Maximum bandwidth per tunnel is approximately 1.25 Gbps
- Route propagation must be enabled in VPC route tables
- Pre-shared keys are auto-generated by AWS
- Customer gateway must have static public IP address
- IPSec Dead Peer Detection (DPD) is enabled by default
- VPN supports both static and dynamic (BGP) routing