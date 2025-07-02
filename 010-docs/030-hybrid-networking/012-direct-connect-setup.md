# Topic 12: Setting Up AWS Direct Connect - Step-by-Step

## Prerequisites
- Completed Topic 11 (Direct Connect Introduction)
- Understanding of BGP routing protocol
- Access to AWS console with Direct Connect permissions
- Basic knowledge of network equipment configuration

## Learning Objectives
By the end of this topic, you will be able to:
- Create and configure Direct Connect connections
- Set up Virtual Interfaces (VIFs) for different use cases
- Establish BGP sessions and configure routing
- Test and validate Direct Connect connectivity

## Theory

### Direct Connect Setup Process Overview

#### High-Level Steps
1. **Connection Creation**: Order physical Direct Connect connection
2. **Cross Connect**: Install physical fiber connection
3. **Virtual Interface**: Create logical interfaces on the connection
4. **BGP Configuration**: Establish routing protocol sessions
5. **Testing and Validation**: Verify connectivity and performance

#### Timeline Considerations
- **Connection Provisioning**: 1-4 weeks depending on location
- **Cross Connect Installation**: 2-5 business days
- **Configuration and Testing**: 1-2 days
- **Total Implementation Time**: 2-6 weeks typical

### Connection Types and Specifications

#### Dedicated Connections
- **Bandwidth Options**: 1 Gbps, 10 Gbps, 100 Gbps
- **Port Types**: Single-mode fiber (1000BASE-LX, 10GBASE-LR, 100GBASE-LR4)
- **Customer Ownership**: Customer owns the entire bandwidth
- **Billing**: Fixed monthly port charges

#### Hosted Connections
- **Bandwidth Options**: 50 Mbps to 10 Gbps
- **Provider Managed**: APN partner manages the physical connection
- **Shared Infrastructure**: Multiple customers share physical connection
- **Billing**: Typically hourly charges

### Virtual Interface (VIF) Configuration

#### Private VIF Requirements
- **VLAN ID**: 1-4094 (must be unique on the connection)
- **BGP ASN**: Customer ASN (private or public)
- **IP Addresses**: /30 subnet from 169.254.0.0/16 or customer public IPs
- **BGP Authentication Key**: Optional MD5 authentication

#### Public VIF Requirements
- **VLAN ID**: 1-4094 (must be unique on the connection)
- **BGP ASN**: Customer public ASN required
- **IP Addresses**: Customer public IP addresses
- **Route Advertisements**: Customer must own advertised prefixes

## Lab Exercise: Complete Direct Connect Setup

### Lab Duration: 240 minutes (simulated - actual setup takes weeks)

### Step 1: Plan Direct Connect Implementation
**Objective**: Create detailed implementation plan

**Pre-Implementation Checklist**:
```bash
# Create implementation checklist
cat << 'EOF' > dx-implementation-checklist.sh
#!/bin/bash

echo "=== Direct Connect Implementation Checklist ==="
echo ""

echo "1. PREREQUISITES VERIFICATION"
echo "   □ AWS account with Direct Connect permissions"
echo "   □ Direct Connect location selected"
echo "   □ Customer router available at DX location"
echo "   □ BGP ASN obtained (private: 64512-65534, public: registered)"
echo "   □ IP addressing plan defined"
echo "   □ Network design approved"
echo ""

echo "2. TECHNICAL REQUIREMENTS"
echo "   □ Customer router supports BGP"
echo "   □ Router supports 802.1Q VLAN tagging"
echo "   □ Appropriate SFP/SFP+ modules available"
echo "   □ Single-mode fiber patch cables"
echo "   □ Router configuration templates prepared"
echo ""

echo "3. DOCUMENTATION PREPARATION"
echo "   □ Network diagrams created"
echo "   □ IP address assignments documented"
echo "   □ BGP configuration planned"
echo "   □ Change management procedures"
echo "   □ Rollback procedures defined"
echo ""

echo "4. COORDINATION REQUIREMENTS"
echo "   □ Colocation facility contacts identified"
echo "   □ Cross connect installation scheduled"
echo "   □ AWS support engagement (if needed)"
echo "   □ Change window scheduled"
echo "   □ Testing procedures defined"

EOF

chmod +x dx-implementation-checklist.sh
./dx-implementation-checklist.sh
```

### Step 2: Create Direct Connect Connection
**Objective**: Order and configure the physical connection

**Connection Creation (Console Simulation)**:
```bash
# Simulate Direct Connect connection creation
cat << 'EOF' > create-dx-connection.sh
#!/bin/bash

echo "=== Creating Direct Connect Connection ==="
echo ""

# Connection parameters
DX_LOCATION="EqDC2"  # Equinix DC2 (Ashburn)
BANDWIDTH="1Gbps"
CONNECTION_NAME="Primary-DX-Connection"

echo "Creating Direct Connect connection..."
echo "Location: $DX_LOCATION"
echo "Bandwidth: $BANDWIDTH"
echo "Connection Name: $CONNECTION_NAME"
echo ""

# In real implementation, this would be:
# aws directconnect create-connection \
#     --location $DX_LOCATION \
#     --bandwidth $BANDWIDTH \
#     --connection-name $CONNECTION_NAME

# Simulated response
cat << 'RESPONSE'
{
    "ownerAccount": "123456789012",
    "connectionId": "dxcon-fhajolyq",
    "connectionName": "Primary-DX-Connection",
    "connectionState": "requested",
    "region": "us-east-1",
    "location": "EqDC2",
    "bandwidth": "1Gbps",
    "vlan": null,
    "partnerName": null,
    "loaIssueTime": "2024-01-15T10:30:00Z"
}
RESPONSE

echo ""
echo "✅ Connection created successfully!"
echo "Connection ID: dxcon-fhajolyq"
echo "State: requested"
echo ""
echo "Next steps:"
echo "1. Download Letter of Authorization (LOA)"
echo "2. Provide LOA to colocation facility"
echo "3. Schedule cross connect installation"
echo "4. Wait for connection state to become 'available'"

EOF

chmod +x create-dx-connection.sh
./create-dx-connection.sh
```

**Download Letter of Authorization**:
```bash
# Simulate LOA download
cat << 'EOF' > download-loa.sh
#!/bin/bash

CONNECTION_ID="dxcon-fhajolyq"

echo "=== Downloading Letter of Authorization (LOA) ==="
echo ""

# In real implementation:
# aws directconnect describe-loa \
#     --connection-id $CONNECTION_ID \
#     --output text \
#     --query 'loaContent' | base64 --decode > loa.pdf

echo "LOA downloaded for connection: $CONNECTION_ID"
echo "File: loa.pdf"
echo ""
echo "LOA contains:"
echo "- AWS equipment information"
echo "- Port assignment details"
echo "- Cross connect instructions"
echo "- Customer information"
echo ""
echo "Provide LOA to colocation facility to install cross connect"

EOF

chmod +x download-loa.sh
./download-loa.sh
```

### Step 3: Monitor Connection Status
**Objective**: Track connection provisioning progress

```bash
# Create connection monitoring script
cat << 'EOF' > monitor-dx-connection.sh
#!/bin/bash

CONNECTION_ID="dxcon-fhajolyq"

echo "=== Monitoring Direct Connect Connection Status ==="
echo ""

# Simulate connection status progression
statuses=("requested" "pending" "available")
states=("Connection requested" "Cross connect being installed" "Ready for configuration")

for i in "${!statuses[@]}"; do
    echo "Status: ${statuses[$i]} - ${states[$i]}"
    if [ "${statuses[$i]}" = "available" ]; then
        echo "✅ Connection is ready for Virtual Interface creation"
        break
    fi
    echo "⏳ Waiting for next status update..."
    echo ""
done

echo ""
echo "Connection Details:"
echo "- Connection ID: $CONNECTION_ID"
echo "- AWS Device: EqDC2-1a2b3c4d"
echo "- Your Device: Customer managed"
echo "- Port Speed: 1 Gbps"
echo "- Port Type: 1000BASE-LX"

# In real implementation:
# aws directconnect describe-connections \
#     --connection-id $CONNECTION_ID \
#     --query 'connections[0].[connectionState,awsDevice,bandwidth]' \
#     --output table

EOF

chmod +x monitor-dx-connection.sh
./monitor-dx-connection.sh
```

### Step 4: Create Private Virtual Interface
**Objective**: Configure Private VIF for VPC connectivity

**Private VIF Configuration**:
```bash
# Create Private VIF
cat << 'EOF' > create-private-vif.sh
#!/bin/bash

CONNECTION_ID="dxcon-fhajolyq"
VIF_NAME="Production-Private-VIF"
VLAN_ID="100"
CUSTOMER_ASN="65001"
CUSTOMER_IP="169.254.100.1/30"
AWS_IP="169.254.100.2/30"
BGP_AUTH_KEY="MySecretKey123"

echo "=== Creating Private Virtual Interface ==="
echo ""

echo "Configuration:"
echo "VIF Name: $VIF_NAME"
echo "VLAN ID: $VLAN_ID"
echo "Customer ASN: $CUSTOMER_ASN"
echo "Customer IP: $CUSTOMER_IP"
echo "AWS IP: $AWS_IP"
echo "BGP Auth Key: [CONFIGURED]"
echo ""

# Create VGW first (if not exists)
VGW_ID=$(aws ec2 create-vpn-gateway --type ipsec.1 \
    --tag-specifications 'ResourceType=vpn-gateway,Tags=[{Key=Name,Value=DX-VGW}]' \
    --query 'VpnGateway.VpnGatewayId' --output text)

echo "Created Virtual Private Gateway: $VGW_ID"

# In real implementation:
# aws directconnect create-private-virtual-interface \
#     --connection-id $CONNECTION_ID \
#     --new-private-virtual-interface \
#         "virtualInterfaceName=$VIF_NAME,\
#          vlan=$VLAN_ID,\
#          asn=$CUSTOMER_ASN,\
#          customerAddress=$CUSTOMER_IP,\
#          amazonAddress=$AWS_IP,\
#          authKey=$BGP_AUTH_KEY,\
#          vgwId=$VGW_ID"

# Simulated response
cat << 'RESPONSE'
{
    "ownerAccount": "123456789012",
    "virtualInterfaceId": "dxvif-fhqp2ig6",
    "virtualInterfaceName": "Production-Private-VIF",
    "virtualInterfaceType": "private",
    "vlan": 100,
    "asn": 65001,
    "customerAddress": "169.254.100.1/30",
    "amazonAddress": "169.254.100.2/30",
    "virtualInterfaceState": "pending",
    "connectionId": "dxcon-fhajolyq",
    "vgwId": "vgw-12345678"
}
RESPONSE

echo ""
echo "✅ Private VIF created successfully!"
echo "VIF ID: dxvif-fhqp2ig6"
echo "State: pending"
echo ""
echo "Next step: Configure customer router BGP session"

EOF

chmod +x create-private-vif.sh
./create-private-vif.sh
```

### Step 5: Configure Customer Router BGP
**Objective**: Set up BGP session on customer equipment

**BGP Configuration Templates**:
```bash
# Create BGP configuration templates
cat << 'EOF' > bgp-config-templates.sh
#!/bin/bash

echo "=== BGP Configuration Templates ==="
echo ""

echo "1. CISCO IOS/IOS-XE CONFIGURATION:"
cat << 'CISCO_CONFIG'
! Interface configuration
interface GigabitEthernet0/1
 description Direct Connect to AWS
 no shutdown
 no ip address

! VLAN interface
interface GigabitEthernet0/1.100
 description AWS Private VIF VLAN 100
 encapsulation dot1Q 100
 ip address 169.254.100.1 255.255.255.252
 no shutdown

! BGP configuration
router bgp 65001
 bgp log-neighbor-changes
 neighbor 169.254.100.2 remote-as 7224
 neighbor 169.254.100.2 password MySecretKey123
 neighbor 169.254.100.2 timers 10 30
 neighbor 169.254.100.2 soft-reconfiguration inbound
 !
 address-family ipv4
  neighbor 169.254.100.2 activate
  neighbor 169.254.100.2 prefix-list CUSTOMER-OUT out
  neighbor 169.254.100.2 prefix-list AWS-IN in
  network 10.1.0.0 mask 255.255.0.0
 exit-address-family

! Prefix lists
ip prefix-list CUSTOMER-OUT seq 10 permit 10.1.0.0/16
ip prefix-list AWS-IN seq 10 permit 10.0.0.0/16
ip prefix-list AWS-IN seq 20 permit 172.16.0.0/12
ip prefix-list AWS-IN seq 30 permit 192.168.0.0/16

! Route map (optional)
route-map SET-LOCAL-PREF permit 10
 set local-preference 200

CISCO_CONFIG

echo ""
echo "2. JUNIPER JUNOS CONFIGURATION:"
cat << 'JUNOS_CONFIG'
# Interface configuration
set interfaces ge-0/0/1 description "Direct Connect to AWS"
set interfaces ge-0/0/1 vlan-tagging
set interfaces ge-0/0/1 unit 100 description "AWS Private VIF VLAN 100"
set interfaces ge-0/0/1 unit 100 vlan-id 100
set interfaces ge-0/0/1 unit 100 family inet address 169.254.100.1/30

# BGP configuration
set protocols bgp group aws type external
set protocols bgp group aws peer-as 7224
set protocols bgp group aws neighbor 169.254.100.2 authentication-key "MySecretKey123"
set protocols bgp group aws neighbor 169.254.100.2 export CUSTOMER-OUT
set protocols bgp group aws neighbor 169.254.100.2 import AWS-IN
set protocols bgp group aws neighbor 169.254.100.2 hold-time 30
set protocols bgp group aws neighbor 169.254.100.2 keepalive 10

# Policy configuration
set policy-options prefix-list CUSTOMER-NETWORKS 10.1.0.0/16
set policy-options prefix-list AWS-NETWORKS 10.0.0.0/16
set policy-options prefix-list AWS-NETWORKS 172.16.0.0/12
set policy-options prefix-list AWS-NETWORKS 192.168.0.0/16

set policy-options policy-statement CUSTOMER-OUT term 1 from prefix-list CUSTOMER-NETWORKS
set policy-options policy-statement CUSTOMER-OUT term 1 then accept
set policy-options policy-statement CUSTOMER-OUT term 2 then reject

set policy-options policy-statement AWS-IN term 1 from prefix-list AWS-NETWORKS
set policy-options policy-statement AWS-IN term 1 then accept
set policy-options policy-statement AWS-IN term 2 then reject

# Autonomous system
set routing-options autonomous-system 65001

JUNOS_CONFIG

echo ""
echo "3. CONFIGURATION CHECKLIST:"
echo "   □ Physical interface configured and up"
echo "   □ VLAN subinterface created with correct VLAN ID"
echo "   □ IP addresses assigned (customer and AWS)"
echo "   □ BGP session configured with AWS ASN 7224"
echo "   □ BGP authentication key configured"
echo "   □ Route filters/prefix lists applied"
echo "   □ Customer networks advertised to AWS"
echo "   □ AWS route acceptance policies configured"

EOF

chmod +x bgp-config-templates.sh
./bgp-config-templates.sh
```

### Step 6: Establish BGP Session and Verify Connectivity
**Objective**: Bring up BGP session and test connectivity

**BGP Session Verification**:
```bash
# Create BGP verification script
cat << 'EOF' > verify-bgp-session.sh
#!/bin/bash

echo "=== BGP Session Verification ==="
echo ""

echo "1. CHECK VIRTUAL INTERFACE STATUS"
VIF_ID="dxvif-fhqp2ig6"

# Check VIF status
echo "Checking VIF status..."
echo "VIF ID: $VIF_ID"
echo "Expected State: available"
echo "BGP State: established"
echo ""

# In real implementation:
# aws directconnect describe-virtual-interfaces \
#     --virtual-interface-id $VIF_ID \
#     --query 'virtualInterfaces[0].[virtualInterfaceState,bgpPeers[0].bgpStatus]' \
#     --output table

echo "2. VERIFY BGP SESSION (Customer Router)"
cat << 'ROUTER_COMMANDS'

For Cisco IOS:
Router# show ip bgp summary
Router# show ip bgp neighbors 169.254.100.2
Router# show ip route bgp

For Juniper:
user@router> show bgp summary
user@router> show bgp neighbor 169.254.100.2
user@router> show route protocol bgp

Expected BGP session state: Established
Expected received routes: AWS VPC CIDRs
Expected advertised routes: Customer networks

ROUTER_COMMANDS

echo ""
echo "3. CONNECTIVITY TESTING"
echo ""
echo "Test steps:"
echo "a) Ping AWS BGP peer: ping 169.254.100.2"
echo "b) Ping VPC resources from on-premises"
echo "c) Ping on-premises from VPC instances"
echo "d) Test application connectivity"
echo ""

echo "4. ROUTING TABLE VERIFICATION"
cat << 'ROUTING_VERIFICATION'

Verify routes are learned via BGP:
- On customer router: Check BGP routes received from AWS
- In AWS: Check VGW route propagation to VPC route tables
- Test traffic flow in both directions

Sample verification commands:
Router# show ip route 10.0.0.0 255.255.0.0
Router# traceroute 10.0.1.10 (VPC resource)

ROUTING_VERIFICATION

EOF

chmod +x verify-bgp-session.sh
./verify-bgp-session.sh
```

### Step 7: Attach Virtual Private Gateway to VPC
**Objective**: Connect VGW to VPC for resource access

```bash
# Attach VGW to VPC
cat << 'EOF' > attach-vgw-to-vpc.sh
#!/bin/bash

VGW_ID="vgw-12345678"
VPC_ID="vpc-0123456789abcdef0"  # Target VPC

echo "=== Attaching Virtual Private Gateway to VPC ==="
echo ""

echo "Attaching VGW $VGW_ID to VPC $VPC_ID..."

# Attach VGW to VPC
aws ec2 attach-vpn-gateway \
    --vpn-gateway-id $VGW_ID \
    --vpc-id $VPC_ID

echo "Waiting for attachment to complete..."
aws ec2 wait vpn-gateway-attached --vpn-gateway-ids $VGW_ID

echo "✅ VGW attached successfully!"
echo ""

echo "Enabling route propagation..."
# Get route table IDs
ROUTE_TABLES=$(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query 'RouteTables[*].RouteTableId' \
    --output text)

# Enable route propagation on all route tables
for RT_ID in $ROUTE_TABLES; do
    echo "Enabling route propagation on route table: $RT_ID"
    aws ec2 enable-vgw-route-propagation \
        --route-table-id $RT_ID \
        --gateway-id $VGW_ID
done

echo ""
echo "Route propagation enabled on all route tables"
echo "BGP routes from on-premises will automatically appear in VPC route tables"

EOF

chmod +x attach-vgw-to-vpc.sh
./attach-vgw-to-vpc.sh
```

### Step 8: Configure Security Groups and NACLs
**Objective**: Allow traffic through security controls

```bash
# Configure security for Direct Connect traffic
cat << 'EOF' > configure-dx-security.sh
#!/bin/bash

echo "=== Configuring Security for Direct Connect Traffic ==="
echo ""

# Create security group for Direct Connect access
DX_SG=$(aws ec2 create-security-group \
    --group-name DirectConnect-SG \
    --description "Security group for Direct Connect traffic" \
    --vpc-id $VPC_ID \
    --query 'GroupId' --output text)

echo "Created security group: $DX_SG"

# Allow inbound from on-premises networks
ONPREM_CIDR="10.1.0.0/16"

# Common ports for application connectivity
aws ec2 authorize-security-group-ingress \
    --group-id $DX_SG \
    --protocol tcp \
    --port 22 \
    --cidr $ONPREM_CIDR \
    --tag-specifications 'ResourceType=security-group-rule,Tags=[{Key=Name,Value=SSH-from-OnPrem}]'

aws ec2 authorize-security-group-ingress \
    --group-id $DX_SG \
    --protocol tcp \
    --port 80 \
    --cidr $ONPREM_CIDR

aws ec2 authorize-security-group-ingress \
    --group-id $DX_SG \
    --protocol tcp \
    --port 443 \
    --cidr $ONPREM_CIDR

aws ec2 authorize-security-group-ingress \
    --group-id $DX_SG \
    --protocol tcp \
    --port 3306 \
    --cidr $ONPREM_CIDR

# Allow all outbound (default)
echo "✅ Security group configured for Direct Connect traffic"
echo ""

echo "Security Group Rules Created:"
echo "- SSH (22) from $ONPREM_CIDR"
echo "- HTTP (80) from $ONPREM_CIDR"
echo "- HTTPS (443) from $ONPREM_CIDR"
echo "- MySQL (3306) from $ONPREM_CIDR"
echo ""

echo "Apply this security group to EC2 instances that need Direct Connect access"

EOF

chmod +x configure-dx-security.sh
./configure-dx-security.sh
```

### Step 9: Performance Testing and Optimization
**Objective**: Validate performance and optimize configuration

```bash
# Create performance testing script
cat << 'EOF' > dx-performance-testing.sh
#!/bin/bash

echo "=== Direct Connect Performance Testing ==="
echo ""

echo "1. LATENCY TESTING"
echo "Test from on-premises to VPC instances:"

# Example latency test targets
VPC_TARGETS=("10.0.1.10" "10.0.2.20" "10.0.3.30")

for target in "${VPC_TARGETS[@]}"; do
    echo "Testing latency to $target"
    echo "Command: ping -c 10 $target"
    echo "Expected: Consistent low latency (< 10ms for local region)"
    echo ""
done

echo "2. BANDWIDTH TESTING"
echo "Test throughput using iperf3:"
echo ""
echo "On VPC instance (server):"
echo "sudo yum install iperf3 -y"
echo "iperf3 -s"
echo ""
echo "From on-premises (client):"
echo "iperf3 -c <vpc-instance-ip> -t 30 -P 4"
echo ""
echo "Expected throughput:"
echo "- 1 Gbps connection: ~900-950 Mbps"
echo "- 10 Gbps connection: ~9-9.5 Gbps"
echo ""

echo "3. ROUTING VERIFICATION"
cat << 'ROUTING_TESTS'

Verify optimal routing:
a) Traceroute from on-premises to VPC
   - Should show direct path via Direct Connect
   - No internet hops visible

b) Check BGP route preferences
   - Verify Direct Connect routes preferred over VPN
   - Confirm load balancing if multiple connections

c) Test failover scenarios
   - Simulate Direct Connect failure
   - Verify traffic fails over to backup (VPN)
   - Measure failover time

ROUTING_TESTS

echo ""
echo "4. MONITORING SETUP"
echo "Set up CloudWatch monitoring for:"
echo "- Direct Connect connection state"
echo "- BGP session status"
echo "- Data transfer metrics"
echo "- Connection utilization"
echo ""

echo "5. PERFORMANCE OPTIMIZATION"
cat << 'OPTIMIZATION_TIPS'

Performance optimization techniques:
- Enable jumbo frames if supported (9000 MTU)
- Optimize BGP timers for faster convergence
- Use multiple BGP sessions for load balancing
- Consider Link Aggregation Groups (LAG) for higher bandwidth
- Implement proper QoS policies

OPTIMIZATION_TIPS

EOF

chmod +x dx-performance-testing.sh
./dx-performance-testing.sh
```

### Step 10: Create Public Virtual Interface (Optional)
**Objective**: Configure Public VIF for AWS public services

```bash
# Create Public VIF for AWS services access
cat << 'EOF' > create-public-vif.sh
#!/bin/bash

CONNECTION_ID="dxcon-fhajolyq"
PUBLIC_VIF_NAME="Public-Services-VIF"
PUBLIC_VLAN_ID="200"
CUSTOMER_ASN="65001"  # Must be public ASN for production
CUSTOMER_PUBLIC_IP="203.0.113.1/30"  # Customer public IP
AWS_PUBLIC_IP="203.0.113.2/30"      # AWS assigned public IP

echo "=== Creating Public Virtual Interface ==="
echo ""

echo "⚠️  NOTE: Public VIF requires public ASN and IP addresses"
echo "This example uses documentation IPs for demonstration"
echo ""

echo "Configuration:"
echo "VIF Name: $PUBLIC_VIF_NAME"
echo "VLAN ID: $PUBLIC_VLAN_ID"
echo "Customer ASN: $CUSTOMER_ASN (must be public for production)"
echo "Customer IP: $CUSTOMER_PUBLIC_IP"
echo "AWS IP: $AWS_PUBLIC_IP"
echo ""

# In real implementation:
# aws directconnect create-public-virtual-interface \
#     --connection-id $CONNECTION_ID \
#     --new-public-virtual-interface \
#         "virtualInterfaceName=$PUBLIC_VIF_NAME,\
#          vlan=$PUBLIC_VLAN_ID,\
#          asn=$CUSTOMER_ASN,\
#          customerAddress=$CUSTOMER_PUBLIC_IP,\
#          amazonAddress=$AWS_PUBLIC_IP,\
#          routeFilterPrefixes=[{cidr=203.0.113.0/24}]"

echo "Public VIF use cases:"
echo "- S3 bucket access without internet gateway"
echo "- CloudFront distribution management"
echo "- AWS API calls via Direct Connect"
echo "- Hybrid applications using public AWS services"
echo ""

echo "BGP Configuration for Public VIF:"
cat << 'PUBLIC_BGP_CONFIG'

router bgp 65001
 neighbor 203.0.113.2 remote-as 7224
 neighbor 203.0.113.2 password PublicVIFKey123
 !
 address-family ipv4
  neighbor 203.0.113.2 activate
  neighbor 203.0.113.2 prefix-list PUBLIC-OUT out
  neighbor 203.0.113.2 prefix-list AWS-PUBLIC-IN in
  network 203.0.113.0 mask 255.255.255.0
 exit-address-family

! Prefix lists for public VIF
ip prefix-list PUBLIC-OUT seq 10 permit 203.0.113.0/24
ip prefix-list AWS-PUBLIC-IN seq 10 permit 0.0.0.0/0 le 24

PUBLIC_BGP_CONFIG

EOF

chmod +x create-public-vif.sh
./create-public-vif.sh
```

## Advanced Configuration Topics

### Link Aggregation Groups (LAG)
```bash
# Create LAG configuration example
cat << 'EOF' > configure-dx-lag.sh
#!/bin/bash

echo "=== Direct Connect Link Aggregation Group (LAG) ==="
echo ""

echo "LAG Benefits:"
echo "- Increased bandwidth (combine multiple connections)"
echo "- Enhanced redundancy"
echo "- Load balancing across links"
echo "- Simplified management"
echo ""

# Create LAG
LAG_NAME="Production-LAG"
MINIMUM_LINKS=1

echo "Creating LAG with 2 x 10 Gbps connections..."

# In real implementation:
# LAG_ID=$(aws directconnect create-lag \
#     --number-of-connections 2 \
#     --location "EqDC2" \
#     --connections-bandwidth "10Gbps" \
#     --lag-name $LAG_NAME \
#     --tags Key=Environment,Value=Production \
#     --query 'lagId' --output text)

echo "LAG ID: dxlag-fh6lq2g8"
echo "Total Bandwidth: 20 Gbps"
echo "Minimum Links: $MINIMUM_LINKS"
echo ""

echo "LAG Configuration Requirements:"
echo "- All connections must be same bandwidth"
echo "- All connections must be in same location"
echo "- Maximum 4 connections per LAG"
echo "- LACP (Link Aggregation Control Protocol) used"

EOF

chmod +x configure-dx-lag.sh
./configure-dx-lag.sh
```

### Direct Connect Gateway
```bash
# Configure Direct Connect Gateway for multi-region
cat << 'EOF' > configure-dx-gateway.sh
#!/bin/bash

echo "=== Direct Connect Gateway Configuration ==="
echo ""

DXGW_NAME="Global-DX-Gateway"

echo "Creating Direct Connect Gateway..."
echo "Name: $DXGW_NAME"
echo "Purpose: Multi-region VPC connectivity"
echo ""

# Create DX Gateway
# DXGW_ID=$(aws directconnect create-direct-connect-gateway \
#     --name $DXGW_NAME \
#     --query 'directConnectGateway.directConnectGatewayId' --output text)

echo "DX Gateway ID: dx-gateway-12345678"
echo ""

echo "DX Gateway Benefits:"
echo "- Connect to VPCs in multiple regions"
echo "- Single Direct Connect location serves multiple regions"
echo "- Simplified network architecture"
echo "- Cost optimization for multi-region deployments"
echo ""

echo "Association Steps:"
echo "1. Create Virtual Interface to DX Gateway"
echo "2. Associate DX Gateway with VGWs in target regions"
echo "3. Configure routing between regions"
echo "4. Test inter-region connectivity"

EOF

chmod +x configure-dx-gateway.sh
./configure-dx-gateway.sh
```

## Troubleshooting Common Issues

### BGP Session Troubleshooting
```bash
# Create BGP troubleshooting guide
cat << 'EOF' > troubleshoot-dx-bgp.sh
#!/bin/bash

echo "=== Direct Connect BGP Troubleshooting ==="
echo ""

echo "1. BGP SESSION NOT ESTABLISHING"
echo "Symptoms: BGP state stuck in 'Idle' or 'Active'"
echo "Possible causes and solutions:"
echo ""
echo "a) Physical connectivity issues:"
echo "   - Check fiber connection and cross connect"
echo "   - Verify interface is up: show interface gi0/1.100"
echo "   - Check for light levels and errors"
echo ""
echo "b) Configuration mismatches:"
echo "   - Verify BGP ASN matches VIF configuration"
echo "   - Check IP addresses (customer and AWS)"
echo "   - Validate BGP authentication key"
echo "   - Confirm VLAN ID configuration"
echo ""
echo "c) Routing issues:"
echo "   - Check default route to BGP peer"
echo "   - Verify no firewall blocking BGP (TCP 179)"
echo "   - Confirm BGP timers configuration"
echo ""

echo "2. BGP SESSION ESTABLISHED BUT NO ROUTES"
echo "Symptoms: BGP status 'Established' but no route exchange"
echo ""
echo "a) Route filtering issues:"
echo "   - Check prefix lists and route maps"
echo "   - Verify route advertisements"
echo "   - Review BGP soft reconfiguration"
echo ""
echo "b) Route limits:"
echo "   - Check maximum prefix limits"
echo "   - Verify route dampening policies"
echo ""

echo "3. ROUTING ASYMMETRY"
echo "Symptoms: Traffic works in one direction only"
echo ""
echo "a) BGP route preferences:"
echo "   - Check local preference settings"
echo "   - Verify AS path prepending"
echo "   - Review MED (metric) values"
echo ""
echo "b) Multiple path scenarios:"
echo "   - Direct Connect vs VPN preferences"
echo "   - Load balancing configuration"
echo "   - Route propagation settings"

echo ""
echo "DIAGNOSTIC COMMANDS:"
cat << 'DIAGNOSTICS'

Cisco IOS diagnostics:
show ip bgp summary
show ip bgp neighbors [peer-ip] advertised-routes
show ip bgp neighbors [peer-ip] received-routes
show ip route bgp
debug ip bgp events
debug ip bgp updates

Juniper diagnostics:
show bgp summary
show route advertising-protocol bgp [peer-ip]
show route receive-protocol bgp [peer-ip]
show route protocol bgp
set traceoptions file bgp-trace
set traceoptions flag all

AWS CLI diagnostics:
aws directconnect describe-virtual-interfaces
aws directconnect describe-connections
aws logs describe-log-streams --log-group-name /aws/directconnect/flows

DIAGNOSTICS

EOF

chmod +x troubleshoot-dx-bgp.sh
./troubleshoot-dx-bgp.sh
```

## Best Practices Summary

### Implementation Best Practices
- **Planning**: Thorough requirements analysis and design review
- **Redundancy**: Multiple connections and locations for high availability
- **Testing**: Comprehensive testing before production deployment
- **Documentation**: Detailed configuration and procedure documentation

### Operational Best Practices
- **Monitoring**: Continuous monitoring of connection and BGP status
- **Alerting**: Automated alerts for connection failures and performance issues
- **Maintenance**: Regular maintenance windows and configuration reviews
- **Training**: Operations team training on Direct Connect management

### Security Best Practices
- **Authentication**: BGP MD5 authentication for session security
- **Filtering**: Strict route filtering and prefix list management
- **Encryption**: Consider MACsec for additional data protection
- **Access Control**: Proper security group and NACL configuration

## Key Takeaways
- Direct Connect setup requires careful planning and coordination
- Physical connectivity takes time - plan for 2-6 week implementation
- BGP configuration is critical for proper routing
- Multiple VIF types serve different connectivity needs
- Comprehensive testing validates implementation success
- Monitoring and alerting ensure ongoing operational health

## Additional Resources
- [Direct Connect User Guide](https://docs.aws.amazon.com/directconnect/latest/UserGuide/)
- [BGP Configuration Examples](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Routing.html)
- [Direct Connect Troubleshooting](https://docs.aws.amazon.com/directconnect/latest/UserGuide/Troubleshooting.html)

## Exam Tips
- Know the difference between dedicated and hosted connections
- Understand VIF types and their use cases
- Remember BGP is required for all Direct Connect connections
- Cross connects are managed by colocation facilities
- LOA (Letter of Authorization) is required for cross connect installation
- BGP ASN 7224 is used by AWS for all Direct Connect sessions
- Private VIF uses 169.254.x.x addressing or customer public IPs
- Public VIF requires customer-owned public ASN and IP addresses