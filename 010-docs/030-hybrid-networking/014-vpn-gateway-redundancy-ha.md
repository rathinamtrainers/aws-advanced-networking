# Topic 14: VPN Gateway Redundancy and High Availability

## Prerequisites
- Completed Topics 11-13 (Direct Connect, Client VPN, Site-to-Site VPN)
- Understanding of high availability design principles
- Knowledge of BGP routing and failover mechanisms
- Familiarity with AWS networking architecture patterns

## Learning Objectives
By the end of this topic, you will be able to:
- Design highly available VPN gateway architectures
- Implement redundant VPN connections and routing
- Configure automatic failover mechanisms
- Monitor and maintain high availability VPN solutions

## Theory

### High Availability VPN Design Principles

#### Definition
VPN Gateway High Availability involves designing resilient network connectivity that maintains service during component failures, ensuring continuous access to cloud and on-premises resources.

#### Key Availability Factors
- **Component Redundancy**: Multiple gateways, connections, and paths
- **Geographic Distribution**: Multi-AZ and multi-region deployment
- **Automatic Failover**: BGP-based routing convergence
- **Health Monitoring**: Proactive failure detection and alerting
- **Disaster Recovery**: Comprehensive backup and recovery procedures

### VPN High Availability Architecture Patterns

#### 1. Dual Tunnel Pattern (Basic HA)
```
Customer Gateway → Tunnel 1 (Primary) → AWS VPN Endpoint 1
                → Tunnel 2 (Backup)  → AWS VPN Endpoint 2
```

#### 2. Dual Customer Gateway Pattern
```
Primary CGW → VPN Connection 1 → Virtual Private Gateway
Backup CGW  → VPN Connection 2 → Virtual Private Gateway
```

#### 3. Multi-Region HA Pattern
```
Region A: Primary CGW → VPN → VGW → VPC-A
Region B: Backup CGW  → VPN → VGW → VPC-B
          ↓
Cross-Region Connectivity (TGW/Peering)
```

#### 4. Hybrid HA Pattern (VPN + Direct Connect)
```
Primary:   Direct Connect → DX Gateway → VGW
Backup:    Site-to-Site VPN → VGW
Emergency: Client VPN → VPC
```

### BGP Configuration for High Availability

#### BGP Path Selection Criteria
1. **Weight** (Cisco-specific): Highest weight preferred
2. **Local Preference**: Higher local preference preferred  
3. **AS Path Length**: Shorter AS path preferred
4. **Origin**: IGP preferred over EGP
5. **MED (Multi-Exit Discriminator)**: Lower MED preferred
6. **Path Selection**: External over internal BGP

#### HA BGP Configuration Strategies
- **Primary/Backup Routes**: Use local preference manipulation
- **Load Balancing**: Equal-cost multipath (ECMP) configuration
- **Fast Convergence**: Aggressive BGP timers
- **Route Filtering**: Prevent routing loops and control path selection

## Lab Exercise: Comprehensive VPN High Availability Implementation

### Lab Duration: 360 minutes

### Step 1: Design HA VPN Architecture
**Objective**: Plan comprehensive high availability solution

**Architecture Planning Assessment**:
```bash
# Create HA VPN planning script
cat << 'EOF' > ha-vpn-planning.sh
#!/bin/bash

echo "=== High Availability VPN Architecture Planning ==="
echo ""

echo "1. AVAILABILITY REQUIREMENTS"
echo "   Target uptime SLA: _______% (99.9%, 99.95%, 99.99%)"
echo "   Maximum acceptable downtime: _______ minutes/year"
echo "   RTO (Recovery Time Objective): _______ minutes"
echo "   RPO (Recovery Point Objective): _______ minutes"
echo "   Business impact of outages: _______"
echo ""

echo "2. REDUNDANCY STRATEGY"
echo "   Primary connectivity: Direct Connect/VPN"
echo "   Backup connectivity: VPN/Direct Connect"
echo "   Emergency connectivity: Client VPN/Internet"
echo "   Geographic redundancy: Single/Multi-region"
echo "   Customer gateway redundancy: Single/Dual/Multiple"
echo ""

echo "3. FAILURE SCENARIOS TO PROTECT AGAINST"
echo "   □ Single tunnel failure"
echo "   □ Customer gateway device failure"
echo "   □ ISP/Internet connectivity failure"
echo "   □ AWS Availability Zone failure"
echo "   □ AWS Region failure"
echo "   □ Data center power/cooling failure"
echo "   □ Human error in configuration"
echo ""

echo "4. ROUTING DESIGN"
echo "   Primary path preference: _______"
echo "   Backup path preference: _______"
echo "   Load balancing strategy: Active/Standby or Active/Active"
echo "   BGP convergence time target: _______ seconds"
echo "   Route filtering requirements: _______"
echo ""

echo "5. MONITORING AND ALERTING"
echo "   Health check frequency: _______ seconds"
echo "   Alert notification methods: Email/SMS/Slack"
echo "   Escalation procedures: _______"
echo "   Automated remediation: Yes/No"
echo "   Performance baseline monitoring: Yes/No"

EOF

chmod +x ha-vpn-planning.sh
./ha-vpn-planning.sh
```

### Step 2: Implement Dual Customer Gateway HA
**Objective**: Set up redundant customer gateways

**Primary Customer Gateway Setup**:
```bash
# Create primary customer gateway and VPN
cat << 'EOF' > setup-primary-vpn.sh
#!/bin/bash

echo "=== Setting Up Primary VPN Connection ==="
echo ""

# Primary site configuration
PRIMARY_CGW_IP="203.0.113.10"
PRIMARY_CGW_NAME="Primary-Customer-Gateway"
PRIMARY_VPN_NAME="Primary-Site-VPN"
CUSTOMER_ASN="65001"

echo "1. CREATING PRIMARY CUSTOMER GATEWAY"
echo "Public IP: $PRIMARY_CGW_IP"
echo "ASN: $CUSTOMER_ASN"
echo ""

# Create primary customer gateway
PRIMARY_CGW_ID=$(aws ec2 create-customer-gateway \
    --type ipsec.1 \
    --public-ip $PRIMARY_CGW_IP \
    --bgp-asn $CUSTOMER_ASN \
    --device-name "Primary-Cisco-ASR1001" \
    --tag-specifications "ResourceType=customer-gateway,Tags=[{Key=Name,Value=$PRIMARY_CGW_NAME},{Key=Role,Value=Primary}]" \
    --query 'CustomerGateway.CustomerGatewayId' \
    --output text)

echo "✅ Primary Customer Gateway created: $PRIMARY_CGW_ID"

# Create or get VGW
if [ -z "$VGW_ID" ]; then
    VGW_ID=$(aws ec2 create-vpn-gateway \
        --type ipsec.1 \
        --amazon-side-asn 64512 \
        --tag-specifications "ResourceType=vpn-gateway,Tags=[{Key=Name,Value=HA-VPN-Gateway}]" \
        --query 'VpnGateway.VpnGatewayId' \
        --output text)
    
    echo "✅ Virtual Private Gateway created: $VGW_ID"
fi

echo ""
echo "2. CREATING PRIMARY VPN CONNECTION"

# Create primary VPN connection
PRIMARY_VPN_ID=$(aws ec2 create-vpn-connection \
    --type ipsec.1 \
    --customer-gateway-id $PRIMARY_CGW_ID \
    --vpn-gateway-id $VGW_ID \
    --options StaticRoutesOnly=false \
    --tag-specifications "ResourceType=vpn-connection,Tags=[{Key=Name,Value=$PRIMARY_VPN_NAME},{Key=Role,Value=Primary}]" \
    --query 'VpnConnection.VpnConnectionId' \
    --output text)

echo "✅ Primary VPN Connection created: $PRIMARY_VPN_ID"

# Wait for VPN to become available
echo "Waiting for primary VPN to become available..."
aws ec2 wait vpn-connection-available --vpn-connection-ids $PRIMARY_VPN_ID

echo "✅ Primary VPN is now available"

# Save configuration
cat << CONFIG > ha-vpn-config.env
export VGW_ID="$VGW_ID"
export PRIMARY_CGW_ID="$PRIMARY_CGW_ID"
export PRIMARY_VPN_ID="$PRIMARY_VPN_ID"
CONFIG

echo ""
echo "Primary VPN configuration saved to ha-vpn-config.env"

EOF

chmod +x setup-primary-vpn.sh
./setup-primary-vpn.sh
```

**Backup Customer Gateway Setup**:
```bash
# Create backup customer gateway and VPN
cat << 'EOF' > setup-backup-vpn.sh
#!/bin/bash

source ha-vpn-config.env

echo "=== Setting Up Backup VPN Connection ==="
echo ""

# Backup site configuration
BACKUP_CGW_IP="203.0.113.11"
BACKUP_CGW_NAME="Backup-Customer-Gateway"
BACKUP_VPN_NAME="Backup-Site-VPN"
CUSTOMER_ASN="65001"

echo "1. CREATING BACKUP CUSTOMER GATEWAY"
echo "Public IP: $BACKUP_CGW_IP"
echo "ASN: $CUSTOMER_ASN"
echo ""

# Create backup customer gateway
BACKUP_CGW_ID=$(aws ec2 create-customer-gateway \
    --type ipsec.1 \
    --public-ip $BACKUP_CGW_IP \
    --bgp-asn $CUSTOMER_ASN \
    --device-name "Backup-Cisco-ASR1002" \
    --tag-specifications "ResourceType=customer-gateway,Tags=[{Key=Name,Value=$BACKUP_CGW_NAME},{Key=Role,Value=Backup}]" \
    --query 'CustomerGateway.CustomerGatewayId' \
    --output text)

echo "✅ Backup Customer Gateway created: $BACKUP_CGW_ID"

echo ""
echo "2. CREATING BACKUP VPN CONNECTION"

# Create backup VPN connection
BACKUP_VPN_ID=$(aws ec2 create-vpn-connection \
    --type ipsec.1 \
    --customer-gateway-id $BACKUP_CGW_ID \
    --vpn-gateway-id $VGW_ID \
    --options StaticRoutesOnly=false \
    --tag-specifications "ResourceType=vpn-connection,Tags=[{Key=Name,Value=$BACKUP_VPN_NAME},{Key=Role,Value=Backup}]" \
    --query 'VpnConnection.VpnConnectionId' \
    --output text)

echo "✅ Backup VPN Connection created: $BACKUP_VPN_ID"

# Wait for VPN to become available
echo "Waiting for backup VPN to become available..."
aws ec2 wait vpn-connection-available --vpn-connection-ids $BACKUP_VPN_ID

echo "✅ Backup VPN is now available"

# Update configuration
cat << CONFIG >> ha-vpn-config.env
export BACKUP_CGW_ID="$BACKUP_CGW_ID"
export BACKUP_VPN_ID="$BACKUP_VPN_ID"
CONFIG

echo ""
echo "Backup VPN configuration added to ha-vpn-config.env"

echo ""
echo "HA VPN SETUP SUMMARY:"
echo "Primary VPN: $PRIMARY_VPN_ID"
echo "Backup VPN:  $BACKUP_VPN_ID"
echo "Shared VGW:  $VGW_ID"

EOF

chmod +x setup-backup-vpn.sh
./setup-backup-vpn.sh
```

### Step 3: Configure HA BGP Routing
**Objective**: Set up BGP for automatic failover

**Primary Site BGP Configuration**:
```bash
# Generate primary site BGP configuration
cat << 'EOF' > configure-primary-bgp.sh
#!/bin/bash

source ha-vpn-config.env

echo "=== Primary Site BGP Configuration ==="
echo ""

# Get VPN connection details
PRIMARY_VPN_DETAILS=$(aws ec2 describe-vpn-connections \
    --vpn-connection-ids $PRIMARY_VPN_ID \
    --query 'VpnConnections[0]')

echo "Generating BGP configuration for primary site..."
echo ""

echo "1. CISCO IOS/IOS-XE CONFIGURATION (Primary Site):"
cat << 'PRIMARY_BGP_CONFIG'

! Primary Site Router Configuration
! Higher local preference for primary paths

hostname Primary-Site-Router

! Physical interface
interface GigabitEthernet0/0
 description Internet connection
 ip address 203.0.113.10 255.255.255.0
 no shutdown

! VPN Tunnel 1 (Primary preferred)
interface Tunnel10
 description AWS VPN Tunnel 1 (Primary)
 ip address 169.254.10.1 255.255.255.252
 ip tcp adjust-mss 1436
 tunnel source GigabitEthernet0/0
 tunnel destination 52.1.1.1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile AWS-IPSEC-PROFILE-1
 no shutdown

! VPN Tunnel 2 (Backup)
interface Tunnel11
 description AWS VPN Tunnel 2 (Backup)
 ip address 169.254.11.1 255.255.255.252
 ip tcp adjust-mss 1436
 tunnel source GigabitEthernet0/0
 tunnel destination 52.1.1.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile AWS-IPSEC-PROFILE-2
 no shutdown

! BGP Configuration
router bgp 65001
 bgp router-id 203.0.113.10
 bgp log-neighbor-changes
 
 ! AWS BGP neighbors
 neighbor 169.254.10.2 remote-as 7224
 neighbor 169.254.10.2 password MyPrimaryKey123
 neighbor 169.254.10.2 timers 10 30
 neighbor 169.254.10.2 soft-reconfiguration inbound
 neighbor 169.254.10.2 route-map SET-PRIMARY-PREF in
 neighbor 169.254.10.2 route-map ADVERTISE-NETWORKS out
 
 neighbor 169.254.11.2 remote-as 7224
 neighbor 169.254.11.2 password MyPrimaryKey456
 neighbor 169.254.11.2 timers 10 30
 neighbor 169.254.11.2 soft-reconfiguration inbound
 neighbor 169.254.11.2 route-map SET-BACKUP-PREF in
 neighbor 169.254.11.2 route-map ADVERTISE-NETWORKS out
 
 ! Network advertisements
 network 10.1.0.0 mask 255.255.0.0
 network 192.168.1.0 mask 255.255.255.0
 
 ! Enable multipath for load balancing
 maximum-paths 2

! Route maps for path preference
route-map SET-PRIMARY-PREF permit 10
 description Prefer primary tunnel
 set local-preference 200
 set weight 100

route-map SET-BACKUP-PREF permit 10
 description Backup tunnel lower preference
 set local-preference 150
 set weight 50

route-map ADVERTISE-NETWORKS permit 10
 description Advertise customer networks
 match ip address prefix-list CUSTOMER-NETWORKS

! Prefix lists
ip prefix-list CUSTOMER-NETWORKS seq 10 permit 10.1.0.0/16
ip prefix-list CUSTOMER-NETWORKS seq 20 permit 192.168.1.0/24

! Track object for tunnel health
track 1 ip sla 1
 delay down 10 up 5

ip sla 1
 icmp-echo 169.254.10.2 source-interface Tunnel10
 frequency 10

ip sla schedule 1 life forever start-time now

! Conditional route advertisement based on tunnel health
route-map ADVERTISE-NETWORKS permit 20
 match track 1
 set local-preference 200

route-map ADVERTISE-NETWORKS permit 30
 set local-preference 100

PRIMARY_BGP_CONFIG

echo ""
echo "2. JUNIPER JUNOS CONFIGURATION (Primary Site):"
cat << 'PRIMARY_JUNOS_CONFIG'

# Primary Site Router Configuration (Juniper)

# Interface configuration
set interfaces ge-0/0/0 description "Internet connection"
set interfaces ge-0/0/0 unit 0 family inet address 203.0.113.10/24

set interfaces st0 unit 10 description "AWS VPN Tunnel 1 (Primary)"
set interfaces st0 unit 10 family inet address 169.254.10.1/30
set interfaces st0 unit 10 family inet mtu 1436

set interfaces st0 unit 11 description "AWS VPN Tunnel 2 (Backup)"
set interfaces st0 unit 11 family inet address 169.254.11.1/30
set interfaces st0 unit 11 family inet mtu 1436

# BGP configuration
set protocols bgp group aws type external
set protocols bgp group aws local-preference 200
set protocols bgp group aws peer-as 7224
set protocols bgp group aws neighbor 169.254.10.2 local-preference 200
set protocols bgp group aws neighbor 169.254.10.2 import AWS-ROUTES-PRIMARY
set protocols bgp group aws neighbor 169.254.10.2 export CUSTOMER-ROUTES

set protocols bgp group aws neighbor 169.254.11.2 local-preference 150
set protocols bgp group aws neighbor 169.254.11.2 import AWS-ROUTES-BACKUP
set protocols bgp group aws neighbor 169.254.11.2 export CUSTOMER-ROUTES

# Routing policies
set policy-options prefix-list CUSTOMER-NETWORKS 10.1.0.0/16
set policy-options prefix-list CUSTOMER-NETWORKS 192.168.1.0/24

set policy-options policy-statement CUSTOMER-ROUTES term 1 from prefix-list CUSTOMER-NETWORKS
set policy-options policy-statement CUSTOMER-ROUTES term 1 then accept
set policy-options policy-statement CUSTOMER-ROUTES term 2 then reject

set policy-options policy-statement AWS-ROUTES-PRIMARY term 1 then local-preference 200
set policy-options policy-statement AWS-ROUTES-PRIMARY term 1 then accept

set policy-options policy-statement AWS-ROUTES-BACKUP term 1 then local-preference 150
set policy-options policy-statement AWS-ROUTES-BACKUP term 1 then accept

# Autonomous system
set routing-options autonomous-system 65001
set routing-options router-id 203.0.113.10

PRIMARY_JUNOS_CONFIG

EOF

chmod +x configure-primary-bgp.sh
./configure-primary-bgp.sh
```

**Backup Site BGP Configuration**:
```bash
# Generate backup site BGP configuration
cat << 'EOF' > configure-backup-bgp.sh
#!/bin/bash

source ha-vpn-config.env

echo "=== Backup Site BGP Configuration ==="
echo ""

echo "1. CISCO IOS/IOS-XE CONFIGURATION (Backup Site):"
cat << 'BACKUP_BGP_CONFIG'

! Backup Site Router Configuration
! Lower local preference as backup paths

hostname Backup-Site-Router

! Physical interface
interface GigabitEthernet0/0
 description Internet connection
 ip address 203.0.113.11 255.255.255.0
 no shutdown

! VPN Tunnel 1 (Backup role)
interface Tunnel20
 description AWS VPN Tunnel 1 (Backup Site)
 ip address 169.254.20.1 255.255.255.252
 ip tcp adjust-mss 1436
 tunnel source GigabitEthernet0/0
 tunnel destination 52.2.2.1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile AWS-IPSEC-PROFILE-3
 no shutdown

! VPN Tunnel 2 (Backup role)
interface Tunnel21
 description AWS VPN Tunnel 2 (Backup Site)
 ip address 169.254.21.1 255.255.255.252
 ip tcp adjust-mss 1436
 tunnel source GigabitEthernet0/0
 tunnel destination 52.2.2.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile AWS-IPSEC-PROFILE-4
 no shutdown

! BGP Configuration
router bgp 65001
 bgp router-id 203.0.113.11
 bgp log-neighbor-changes
 
 ! AWS BGP neighbors with lower preference
 neighbor 169.254.20.2 remote-as 7224
 neighbor 169.254.20.2 password MyBackupKey123
 neighbor 169.254.20.2 timers 10 30
 neighbor 169.254.20.2 soft-reconfiguration inbound
 neighbor 169.254.20.2 route-map SET-BACKUP-PREF in
 neighbor 169.254.20.2 route-map ADVERTISE-NETWORKS out
 
 neighbor 169.254.21.2 remote-as 7224
 neighbor 169.254.21.2 password MyBackupKey456
 neighbor 169.254.21.2 timers 10 30
 neighbor 169.254.21.2 soft-reconfiguration inbound
 neighbor 169.254.21.2 route-map SET-BACKUP-PREF in
 neighbor 169.254.21.2 route-map ADVERTISE-NETWORKS out
 
 ! Network advertisements with AS path prepending
 network 10.1.0.0 mask 255.255.0.0
 network 192.168.1.0 mask 255.255.255.0
 
 ! Enable multipath
 maximum-paths 2

! Route maps for backup paths (lower preference)
route-map SET-BACKUP-PREF permit 10
 description Lower preference for backup site
 set local-preference 100
 set weight 25

route-map ADVERTISE-NETWORKS permit 10
 description Advertise customer networks with AS prepending
 match ip address prefix-list CUSTOMER-NETWORKS
 set as-path prepend 65001 65001  # Make path less preferred

! Prefix lists
ip prefix-list CUSTOMER-NETWORKS seq 10 permit 10.1.0.0/16
ip prefix-list CUSTOMER-NETWORKS seq 20 permit 192.168.1.0/24

! Health monitoring
track 2 ip sla 2
 delay down 10 up 5

ip sla 2
 icmp-echo 169.254.20.2 source-interface Tunnel20
 frequency 10

ip sla schedule 2 life forever start-time now

! Conditional advertisements
event manager applet BACKUP-FAILOVER
 event track 2 state down
 action 1.0 cli command "enable"
 action 2.0 cli command "conf t"
 action 3.0 cli command "router bgp 65001"
 action 4.0 cli command "neighbor 169.254.20.2 route-map FAILOVER-ROUTES out"
 action 5.0 cli command "end"

route-map FAILOVER-ROUTES permit 10
 description Emergency failover - remove AS prepending
 match ip address prefix-list CUSTOMER-NETWORKS

BACKUP_BGP_CONFIG

echo ""
echo "2. BGP CONVERGENCE OPTIMIZATION"
cat << 'BGP_OPTIMIZATION'

! Fast BGP Convergence Settings
router bgp 65001
 bgp fast-convergence
 bgp bestpath compare-routerid
 
 ! Aggressive timers for faster detection
 neighbor 169.254.20.2 timers 10 30
 neighbor 169.254.20.2 fall-over bfd
 
 ! BGP graceful restart
 bgp graceful-restart restart-time 120
 bgp graceful-restart stalepath-time 360

! BFD for sub-second failure detection
interface Tunnel20
 bfd interval 300 min_rx 300 multiplier 3

interface Tunnel21
 bfd interval 300 min_rx 300 multiplier 3

router bgp 65001
 neighbor 169.254.20.2 fall-over bfd
 neighbor 169.254.21.2 fall-over bfd

BGP_OPTIMIZATION

EOF

chmod +x configure-backup-bgp.sh
./configure-backup-bgp.sh
```

### Step 4: Implement VPN + Direct Connect HA
**Objective**: Configure hybrid connectivity with DX primary, VPN backup

```bash
# Configure hybrid HA with Direct Connect primary
cat << 'EOF' > setup-hybrid-ha.sh
#!/bin/bash

echo "=== Setting Up Hybrid HA (Direct Connect + VPN) ==="
echo ""

echo "1. HYBRID HA ARCHITECTURE OVERVIEW"
cat << 'ARCHITECTURE'

Primary Path:   On-premises → Direct Connect → AWS
Backup Path:    On-premises → Site-to-Site VPN → AWS
Emergency Path: Remote Users → Client VPN → AWS

Benefits:
- High performance primary (Direct Connect)
- Cost-effective backup (VPN)
- Automatic failover via BGP
- Multiple connectivity options

ARCHITECTURE

echo ""
echo "2. BGP CONFIGURATION FOR HYBRID HA"
cat << 'HYBRID_BGP_CONFIG'

! Customer Router Configuration for Hybrid HA
! Direct Connect as primary, VPN as backup

! Direct Connect interface (higher preference)
interface GigabitEthernet0/1.100
 description AWS Direct Connect VLAN 100
 encapsulation dot1Q 100
 ip address 169.254.100.1 255.255.255.252
 no shutdown

! VPN tunnels (lower preference)
interface Tunnel1
 description AWS VPN Backup Tunnel 1
 ip address 169.254.10.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 52.1.1.1
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile AWS-IPSEC-PROFILE
 no shutdown

interface Tunnel2
 description AWS VPN Backup Tunnel 2
 ip address 169.254.11.1 255.255.255.252
 tunnel source GigabitEthernet0/0
 tunnel destination 52.1.1.2
 tunnel mode ipsec ipv4
 tunnel protection ipsec profile AWS-IPSEC-PROFILE
 no shutdown

! BGP Configuration
router bgp 65001
 bgp router-id 10.1.1.1
 bgp log-neighbor-changes
 
 ! Direct Connect BGP (Primary - Higher Local Preference)
 neighbor 169.254.100.2 remote-as 7224
 neighbor 169.254.100.2 password DXSecretKey123
 neighbor 169.254.100.2 timers 10 30
 neighbor 169.254.100.2 route-map DX-PRIMARY in
 neighbor 169.254.100.2 route-map ADVERTISE-CUSTOMER out
 neighbor 169.254.100.2 fall-over bfd
 
 ! VPN BGP (Backup - Lower Local Preference)
 neighbor 169.254.10.2 remote-as 7224
 neighbor 169.254.10.2 password VPNSecretKey123
 neighbor 169.254.10.2 timers 10 30
 neighbor 169.254.10.2 route-map VPN-BACKUP in
 neighbor 169.254.10.2 route-map ADVERTISE-CUSTOMER out
 
 neighbor 169.254.11.2 remote-as 7224
 neighbor 169.254.11.2 password VPNSecretKey456
 neighbor 169.254.11.2 timers 10 30
 neighbor 169.254.11.2 route-map VPN-BACKUP in
 neighbor 169.254.11.2 route-map ADVERTISE-CUSTOMER out
 
 ! Customer networks
 network 10.1.0.0 mask 255.255.0.0
 network 192.168.0.0 mask 255.255.0.0

! Route maps for path preference
route-map DX-PRIMARY permit 10
 description Direct Connect - Primary path
 set local-preference 300
 set weight 200

route-map VPN-BACKUP permit 10
 description VPN - Backup path
 set local-preference 100
 set weight 50

route-map ADVERTISE-CUSTOMER permit 10
 description Advertise customer networks
 match ip address prefix-list CUSTOMER-PREFIXES

! Health monitoring for Direct Connect
track 10 ip sla 10
 delay down 15 up 5

ip sla 10
 icmp-echo 169.254.100.2 source-interface GigabitEthernet0/1.100
 frequency 5
 timeout 2

ip sla schedule 10 life forever start-time now

! Automatic failover configuration
event manager applet DX-FAILOVER
 event track 10 state down
 action 1.0 syslog msg "Direct Connect failed - failing over to VPN"
 action 2.0 cli command "enable"
 action 3.0 cli command "conf t"
 action 4.0 cli command "router bgp 65001"
 action 5.0 cli command "neighbor 169.254.10.2 route-map VPN-ACTIVE in"
 action 6.0 cli command "neighbor 169.254.11.2 route-map VPN-ACTIVE in"
 action 7.0 cli command "end"

route-map VPN-ACTIVE permit 10
 description VPN becomes active during DX failure
 set local-preference 250

HYBRID_BGP_CONFIG

echo ""
echo "3. MONITORING AND ALERTING SETUP"
cat << 'MONITORING_CONFIG'

! SNMP monitoring for path status
snmp-server community public RO
snmp-server location "Customer Data Center"
snmp-server contact "network-team@company.com"

! Track interface states
track 20 interface GigabitEthernet0/1.100 line-protocol
track 21 interface Tunnel1 line-protocol
track 22 interface Tunnel2 line-protocol

! BGP neighbor tracking
track 30 ip route 169.254.100.2 255.255.255.255 reachability
track 31 ip route 169.254.10.2 255.255.255.255 reachability
track 32 ip route 169.254.11.2 255.255.255.255 reachability

! Syslog for path changes
event manager applet PATH-CHANGE
 event routing network 10.0.0.0/16 type add protocol bgp
 action 1.0 syslog msg "Route to AWS via new path: $_routing_tablename"

MONITORING_CONFIG

EOF

chmod +x setup-hybrid-ha.sh
./setup-hybrid-ha.sh
```

### Step 5: Multi-Region HA Implementation
**Objective**: Deploy cross-region redundancy

```bash
# Configure multi-region HA VPN
cat << 'EOF' > setup-multiregion-ha.sh
#!/bin/bash

echo "=== Multi-Region HA VPN Implementation ==="
echo ""

echo "1. MULTI-REGION ARCHITECTURE"
cat << 'MULTIREGION_ARCH'

Primary Region (us-east-1):
Customer Gateway → VPN → VGW → VPC-Primary → TGW

Backup Region (us-west-2):
Customer Gateway → VPN → VGW → VPC-Backup → TGW

Cross-Region Connectivity:
TGW-us-east-1 ←→ TGW-us-west-2 (Peering)

Failover Strategy:
- Primary: All traffic to us-east-1
- Backup: Traffic fails over to us-west-2
- Database replication between regions
- Application-level failover logic

MULTIREGION_ARCH

echo ""
echo "2. CREATE BACKUP REGION VPN"

# Backup region VPN setup
BACKUP_REGION="us-west-2"
PRIMARY_REGION="us-east-1"

echo "Setting up VPN in backup region: $BACKUP_REGION"

# Create VGW in backup region
BACKUP_VGW_ID=$(aws ec2 create-vpn-gateway \
    --type ipsec.1 \
    --amazon-side-asn 64512 \
    --region $BACKUP_REGION \
    --tag-specifications "ResourceType=vpn-gateway,Tags=[{Key=Name,Value=Backup-Region-VGW}]" \
    --query 'VpnGateway.VpnGatewayId' \
    --output text)

echo "✅ Backup VGW created: $BACKUP_VGW_ID (Region: $BACKUP_REGION)"

# Create customer gateway in backup region
BACKUP_REGION_CGW_ID=$(aws ec2 create-customer-gateway \
    --type ipsec.1 \
    --public-ip 203.0.113.10 \
    --bgp-asn 65001 \
    --region $BACKUP_REGION \
    --tag-specifications "ResourceType=customer-gateway,Tags=[{Key=Name,Value=Backup-Region-CGW}]" \
    --query 'CustomerGateway.CustomerGatewayId' \
    --output text)

echo "✅ Backup region CGW created: $BACKUP_REGION_CGW_ID"

# Create VPN connection in backup region
BACKUP_REGION_VPN_ID=$(aws ec2 create-vpn-connection \
    --type ipsec.1 \
    --customer-gateway-id $BACKUP_REGION_CGW_ID \
    --vpn-gateway-id $BACKUP_VGW_ID \
    --region $BACKUP_REGION \
    --options StaticRoutesOnly=false \
    --tag-specifications "ResourceType=vpn-connection,Tags=[{Key=Name,Value=Backup-Region-VPN}]" \
    --query 'VpnConnection.VpnConnectionId' \
    --output text)

echo "✅ Backup region VPN created: $BACKUP_REGION_VPN_ID"

echo ""
echo "3. BGP CONFIGURATION FOR MULTI-REGION"
cat << 'MULTIREGION_BGP'

! Multi-Region BGP Configuration
! Primary region preferred, backup region as failover

router bgp 65001
 bgp router-id 10.1.1.1
 
 ! Primary Region neighbors (us-east-1) - Higher preference
 neighbor 169.254.10.2 remote-as 7224
 neighbor 169.254.10.2 description "Primary Region Tunnel 1"
 neighbor 169.254.10.2 route-map PRIMARY-REGION in
 neighbor 169.254.10.2 route-map ADVERTISE-ALL out
 
 neighbor 169.254.11.2 remote-as 7224
 neighbor 169.254.11.2 description "Primary Region Tunnel 2"
 neighbor 169.254.11.2 route-map PRIMARY-REGION in
 neighbor 169.254.11.2 route-map ADVERTISE-ALL out
 
 ! Backup Region neighbors (us-west-2) - Lower preference
 neighbor 169.254.20.2 remote-as 7224
 neighbor 169.254.20.2 description "Backup Region Tunnel 1"
 neighbor 169.254.20.2 route-map BACKUP-REGION in
 neighbor 169.254.20.2 route-map ADVERTISE-BACKUP out
 
 neighbor 169.254.21.2 remote-as 7224
 neighbor 169.254.21.2 description "Backup Region Tunnel 2"
 neighbor 169.254.21.2 route-map BACKUP-REGION in
 neighbor 169.254.21.2 route-map ADVERTISE-BACKUP out

! Route maps for regional preference
route-map PRIMARY-REGION permit 10
 description Primary region - highest preference
 set local-preference 300
 set community 65001:100

route-map BACKUP-REGION permit 10
 description Backup region - lower preference
 set local-preference 200
 set community 65001:200

! Advertisement policies
route-map ADVERTISE-ALL permit 10
 description Advertise all customer routes to primary
 match ip address prefix-list CUSTOMER-NETWORKS

route-map ADVERTISE-BACKUP permit 10
 description Advertise customer routes to backup with AS prepend
 match ip address prefix-list CUSTOMER-NETWORKS
 set as-path prepend 65001 65001

! Health-based failover
track 100 ip sla 100
 delay down 30 up 10

ip sla 100
 icmp-echo 10.0.1.1 source-interface loopback1
 frequency 10

ip sla schedule 100 life forever start-time now

! Conditional route advertisements
route-map ADVERTISE-ALL permit 20
 match track 100
 continue

route-map ADVERTISE-BACKUP permit 20
 match track 100 state down
 set as-path prepend 65001  # Only single prepend during primary failure

MULTIREGION_BGP

echo ""
echo "4. CROSS-REGION TRANSIT GATEWAY PEERING"
cat << 'TGW_PEERING'

# Create Transit Gateway peering between regions
PRIMARY_TGW_ID="tgw-primary123"  # Replace with actual TGW ID
BACKUP_TGW_ID="tgw-backup456"    # Replace with actual TGW ID

# Create peering attachment from primary to backup region
aws ec2 create-transit-gateway-peering-attachment \
    --transit-gateway-id $PRIMARY_TGW_ID \
    --peer-transit-gateway-id $BACKUP_TGW_ID \
    --peer-region $BACKUP_REGION \
    --tag-specifications "ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=Primary-to-Backup-Peering}]"

# Accept peering in backup region
aws ec2 accept-transit-gateway-peering-attachment \
    --transit-gateway-attachment-id <peering-attachment-id> \
    --region $BACKUP_REGION

# Configure routing between regions
aws ec2 create-route \
    --route-table-id <primary-tgw-route-table> \
    --destination-cidr-block 10.1.0.0/16 \
    --transit-gateway-attachment-id <peering-attachment-id>

TGW_PEERING

echo ""
echo "Multi-region HA VPN setup complete"
echo "Primary Region: $PRIMARY_REGION"
echo "Backup Region: $BACKUP_REGION"
echo "Cross-region peering enables failover between regions"

EOF

chmod +x setup-multiregion-ha.sh
./setup-multiregion-ha.sh
```

### Step 6: Implement Comprehensive Monitoring
**Objective**: Set up monitoring and alerting for HA VPN

```bash
# Set up comprehensive HA monitoring
cat << 'EOF' > setup-ha-monitoring.sh
#!/bin/bash

source ha-vpn-config.env

echo "=== Setting Up HA VPN Monitoring ==="
echo ""

echo "1. CLOUDWATCH ALARMS FOR VPN STATUS"

# Create alarms for primary VPN
aws cloudwatch put-metric-alarm \
    --alarm-name "Primary-VPN-Tunnel-State" \
    --alarm-description "Primary VPN tunnel state monitoring" \
    --metric-name TunnelState \
    --namespace AWS/VPN \
    --statistic Maximum \
    --period 60 \
    --threshold 0 \
    --comparison-operator LessThanThreshold \
    --dimensions Name=VpnId,Value=$PRIMARY_VPN_ID \
    --evaluation-periods 2 \
    --alarm-actions arn:aws:sns:us-east-1:123456789012:vpn-alerts

# Create alarms for backup VPN
aws cloudwatch put-metric-alarm \
    --alarm-name "Backup-VPN-Tunnel-State" \
    --alarm-description "Backup VPN tunnel state monitoring" \
    --metric-name TunnelState \
    --namespace AWS/VPN \
    --statistic Maximum \
    --period 60 \
    --threshold 0 \
    --comparison-operator LessThanThreshold \
    --dimensions Name=VpnId,Value=$BACKUP_VPN_ID \
    --evaluation-periods 2 \
    --alarm-actions arn:aws:sns:us-east-1:123456789012:vpn-alerts

echo "✅ VPN tunnel state alarms created"

echo ""
echo "2. BGP SESSION MONITORING"

# Create composite alarm for overall connectivity
aws cloudwatch put-composite-alarm \
    --alarm-name "HA-VPN-Connectivity-Status" \
    --alarm-description "Overall HA VPN connectivity status" \
    --alarm-rule "(ALARM(\"Primary-VPN-Tunnel-State\") AND ALARM(\"Backup-VPN-Tunnel-State\"))" \
    --actions-enabled \
    --alarm-actions arn:aws:sns:us-east-1:123456789012:critical-alerts

echo "✅ BGP session monitoring alarms created"

echo ""
echo "3. PERFORMANCE MONITORING DASHBOARD"

# Create comprehensive dashboard
DASHBOARD_BODY='{
    "widgets": [
        {
            "type": "metric",
            "x": 0,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "metrics": [
                    [ "AWS/VPN", "TunnelState", "VpnId", "'$PRIMARY_VPN_ID'", { "label": "Primary VPN" } ],
                    [ ".", ".", ".", "'$BACKUP_VPN_ID'", { "label": "Backup VPN" } ]
                ],
                "period": 300,
                "stat": "Maximum",
                "region": "us-east-1",
                "title": "VPN Tunnel States",
                "yAxis": {
                    "left": {
                        "min": 0,
                        "max": 1
                    }
                }
            }
        },
        {
            "type": "metric",
            "x": 0,
            "y": 6,
            "width": 12,
            "height": 6,
            "properties": {
                "metrics": [
                    [ "AWS/VPN", "TunnelIngressBytes", "VpnId", "'$PRIMARY_VPN_ID'" ],
                    [ ".", "TunnelEgressBytes", ".", "." ],
                    [ ".", "TunnelIngressBytes", ".", "'$BACKUP_VPN_ID'" ],
                    [ ".", "TunnelEgressBytes", ".", "." ]
                ],
                "period": 300,
                "stat": "Sum",
                "region": "us-east-1",
                "title": "VPN Data Transfer"
            }
        },
        {
            "type": "log",
            "x": 0,
            "y": 12,
            "width": 24,
            "height": 6,
            "properties": {
                "query": "SOURCE '\"'VPC-Flow-Logs'\"' | fields @timestamp, srcaddr, dstaddr, protocol, srcport, dstport\n| filter srcaddr like /10.1/\n| stats count() by dstaddr\n| sort count desc\n| limit 20",
                "region": "us-east-1",
                "title": "Top Destinations from On-Premises",
                "view": "table"
            }
        }
    ]
}'

aws cloudwatch put-dashboard \
    --dashboard-name "HA-VPN-Monitoring" \
    --dashboard-body "$DASHBOARD_BODY"

echo "✅ Comprehensive monitoring dashboard created"

echo ""
echo "4. CUSTOM HEALTH CHECK LAMBDA"

# Create Lambda function for advanced health checks
cat << 'LAMBDA_CODE' > ha-vpn-health-check.py
import json
import boto3
import logging
from datetime import datetime, timedelta

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """
    Advanced health check for HA VPN connections
    """
    
    ec2 = boto3.client('ec2')
    cloudwatch = boto3.client('cloudwatch')
    
    # VPN connections to monitor
    vpn_connections = [
        {'id': 'vpn-primary123', 'role': 'primary'},
        {'id': 'vpn-backup456', 'role': 'backup'}
    ]
    
    health_status = {}
    
    for vpn in vpn_connections:
        vpn_id = vpn['id']
        role = vpn['role']
        
        try:
            # Get VPN connection details
            response = ec2.describe_vpn_connections(VpnConnectionIds=[vpn_id])
            vpn_details = response['VpnConnections'][0]
            
            # Check tunnel states
            tunnels = vpn_details['VgwTelemetry']
            tunnel_states = [tunnel['Status'] for tunnel in tunnels]
            
            # Calculate health score
            up_tunnels = tunnel_states.count('UP')
            total_tunnels = len(tunnel_states)
            health_score = (up_tunnels / total_tunnels) * 100
            
            health_status[role] = {
                'vpn_id': vpn_id,
                'state': vpn_details['State'],
                'tunnel_states': tunnel_states,
                'health_score': health_score,
                'up_tunnels': up_tunnels,
                'total_tunnels': total_tunnels
            }
            
            # Send custom metric
            cloudwatch.put_metric_data(
                Namespace='Custom/VPN',
                MetricData=[
                    {
                        'MetricName': 'HealthScore',
                        'Dimensions': [
                            {'Name': 'VpnId', 'Value': vpn_id},
                            {'Name': 'Role', 'Value': role}
                        ],
                        'Value': health_score,
                        'Unit': 'Percent',
                        'Timestamp': datetime.utcnow()
                    }
                ]
            )
            
            logger.info(f"VPN {vpn_id} ({role}): Health Score {health_score}%")
            
        except Exception as e:
            logger.error(f"Error checking VPN {vpn_id}: {str(e)}")
            health_status[role] = {'error': str(e)}
    
    # Overall system health
    primary_health = health_status.get('primary', {}).get('health_score', 0)
    backup_health = health_status.get('backup', {}).get('health_score', 0)
    
    overall_health = 'HEALTHY'
    if primary_health == 0 and backup_health == 0:
        overall_health = 'CRITICAL'
    elif primary_health == 0:
        overall_health = 'DEGRADED'
    
    # Send overall health metric
    cloudwatch.put_metric_data(
        Namespace='Custom/VPN',
        MetricData=[
            {
                'MetricName': 'OverallHealth',
                'Value': 1 if overall_health == 'HEALTHY' else 0,
                'Unit': 'Count',
                'Timestamp': datetime.utcnow()
            }
        ]
    )
    
    return {
        'statusCode': 200,
        'body': json.dumps({
            'overall_health': overall_health,
            'vpn_status': health_status,
            'timestamp': datetime.utcnow().isoformat()
        })
    }
LAMBDA_CODE

echo "✅ Lambda health check function code generated"

echo ""
echo "5. AUTOMATED REMEDIATION"
cat << 'REMEDIATION_SCRIPT'

#!/bin/bash
# Automated VPN remediation script

remediate_vpn_failure() {
    local vpn_id=$1
    local tunnel_ip=$2
    
    echo "Attempting remediation for VPN: $vpn_id"
    
    # Reset VPN connection
    aws ec2 reset-vpn-connection --vpn-connection-id $vpn_id
    
    # Wait and check status
    sleep 60
    
    # Verify tunnel state
    tunnel_state=$(aws ec2 describe-vpn-connections \
        --vpn-connection-ids $vpn_id \
        --query 'VpnConnections[0].VgwTelemetry[0].Status' \
        --output text)
    
    if [ "$tunnel_state" = "UP" ]; then
        echo "✅ VPN $vpn_id remediation successful"
        return 0
    else
        echo "❌ VPN $vpn_id remediation failed"
        return 1
    fi
}

# Monitor and auto-remediate
monitor_and_remediate() {
    while true; do
        # Check primary VPN
        primary_state=$(aws ec2 describe-vpn-connections \
            --vpn-connection-ids $PRIMARY_VPN_ID \
            --query 'VpnConnections[0].VgwTelemetry[0].Status' \
            --output text)
        
        if [ "$primary_state" != "UP" ]; then
            echo "Primary VPN tunnel down - attempting remediation"
            remediate_vpn_failure $PRIMARY_VPN_ID
        fi
        
        # Check backup VPN
        backup_state=$(aws ec2 describe-vpn-connections \
            --vpn-connection-ids $BACKUP_VPN_ID \
            --query 'VpnConnections[0].VgwTelemetry[0].Status' \
            --output text)
        
        if [ "$backup_state" != "UP" ]; then
            echo "Backup VPN tunnel down - attempting remediation"
            remediate_vpn_failure $BACKUP_VPN_ID
        fi
        
        # Wait before next check
        sleep 300
    done
}

REMEDIATION_SCRIPT

echo ""
echo "MONITORING SETUP COMPLETE:"
echo "- CloudWatch alarms for tunnel states"
echo "- Composite alarm for overall connectivity"
echo "- Comprehensive monitoring dashboard"
echo "- Lambda-based health checks"
echo "- Automated remediation scripts"

EOF

chmod +x setup-ha-monitoring.sh
./setup-ha-monitoring.sh
```

### Step 7: Test HA Failover Scenarios
**Objective**: Validate automatic failover functionality

```bash
# Create HA failover testing script
cat << 'EOF' > test-ha-failover.sh
#!/bin/bash

source ha-vpn-config.env

echo "=== HA VPN Failover Testing ==="
echo ""

echo "1. PRE-TEST VALIDATION"
echo "Validating current connectivity status..."

# Check primary VPN status
PRIMARY_STATUS=$(aws ec2 describe-vpn-connections \
    --vpn-connection-ids $PRIMARY_VPN_ID \
    --query 'VpnConnections[0].State' \
    --output text)

# Check backup VPN status  
BACKUP_STATUS=$(aws ec2 describe-vpn-connections \
    --vpn-connection-ids $BACKUP_VPN_ID \
    --query 'VpnConnections[0].State' \
    --output text)

echo "Primary VPN Status: $PRIMARY_STATUS"
echo "Backup VPN Status: $BACKUP_STATUS"

echo ""
echo "2. FAILOVER TEST SCENARIOS"
cat << 'TEST_SCENARIOS'

SCENARIO 1: Primary Tunnel Failure
- Simulate primary tunnel failure
- Verify traffic switches to backup tunnel
- Measure convergence time
- Validate application connectivity

SCENARIO 2: Primary Customer Gateway Failure
- Simulate primary site failure
- Verify traffic switches to backup site
- Test BGP convergence
- Validate end-to-end connectivity

SCENARIO 3: Primary Region Failure
- Simulate entire region failure
- Verify cross-region failover
- Test application recovery
- Validate data consistency

SCENARIO 4: ISP Connectivity Failure
- Simulate internet connectivity loss
- Verify backup connectivity paths
- Test Direct Connect failover to VPN
- Validate business continuity

TEST_SCENARIOS

echo ""
echo "3. AUTOMATED FAILOVER TESTING"

# Test 1: Primary tunnel failover simulation
echo "TEST 1: Primary Tunnel Failover"
echo "Simulating primary tunnel failure..."

# This would be done on customer router in real scenario
cat << 'TUNNEL_FAILOVER_TEST'

# On customer router - simulate tunnel failure
interface Tunnel10
 shutdown

# Monitor BGP convergence
router bgp 65001
 show ip bgp summary
 show ip route bgp

# Expected behavior:
# - BGP session to primary tunnel goes down
# - Traffic switches to backup tunnel
# - Convergence time: < 30 seconds

# Restore primary tunnel
interface Tunnel10
 no shutdown

TUNNEL_FAILOVER_TEST

echo ""
echo "4. PERFORMANCE VALIDATION"

# Create connectivity test script
cat << 'CONNECTIVITY_TEST' > connectivity-test.sh
#!/bin/bash

TEST_TARGET="10.0.1.10"  # VPC instance IP
TEST_DURATION=300        # 5 minutes
PING_INTERVAL=1         # 1 second

echo "Starting connectivity test to $TEST_TARGET"
echo "Duration: $TEST_DURATION seconds"
echo "Ping interval: $PING_INTERVAL second"

# Start timestamp
START_TIME=$(date +%s)
PACKET_COUNT=0
PACKET_LOSS=0
TOTAL_RTT=0

while [ $(($(date +%s) - START_TIME)) -lt $TEST_DURATION ]; do
    # Ping test
    if ping -c 1 -W 2 $TEST_TARGET > /dev/null 2>&1; then
        RTT=$(ping -c 1 $TEST_TARGET | grep "time=" | sed 's/.*time=\([0-9.]*\).*/\1/')
        TOTAL_RTT=$(echo "$TOTAL_RTT + $RTT" | bc)
        echo "$(date '+%H:%M:%S') - Ping successful: ${RTT}ms"
    else
        PACKET_LOSS=$((PACKET_LOSS + 1))
        echo "$(date '+%H:%M:%S') - Ping failed"
    fi
    
    PACKET_COUNT=$((PACKET_COUNT + 1))
    sleep $PING_INTERVAL
done

# Calculate statistics
LOSS_PERCENTAGE=$(echo "scale=2; ($PACKET_LOSS * 100) / $PACKET_COUNT" | bc)
AVG_RTT=$(echo "scale=2; $TOTAL_RTT / ($PACKET_COUNT - $PACKET_LOSS)" | bc)

echo ""
echo "CONNECTIVITY TEST RESULTS:"
echo "Total packets: $PACKET_COUNT"
echo "Packet loss: $PACKET_LOSS ($LOSS_PERCENTAGE%)"
echo "Average RTT: ${AVG_RTT}ms"

CONNECTIVITY_TEST

chmod +x connectivity-test.sh

echo ""
echo "5. FAILOVER METRICS COLLECTION"

# Create metrics collection script
cat << 'METRICS_SCRIPT' > collect-failover-metrics.sh
#!/bin/bash

echo "Collecting failover metrics..."

# BGP convergence time
CONVERGENCE_START=$(date +%s)

# Monitor route changes
while true; do
    ROUTE_COUNT=$(ip route | grep "10.0.0.0/16" | wc -l)
    
    if [ $ROUTE_COUNT -gt 0 ]; then
        CONVERGENCE_END=$(date +%s)
        CONVERGENCE_TIME=$((CONVERGENCE_END - CONVERGENCE_START))
        echo "BGP convergence completed in $CONVERGENCE_TIME seconds"
        break
    fi
    
    sleep 1
done

# Application recovery time test
APP_START=$(date +%s)
while ! curl -s http://10.0.1.10:80 > /dev/null; do
    sleep 1
done
APP_END=$(date +%s)
APP_RECOVERY_TIME=$((APP_END - APP_START))

echo "Application recovery time: $APP_RECOVERY_TIME seconds"

# Log metrics to CloudWatch
aws cloudwatch put-metric-data \
    --namespace "Custom/VPN/Failover" \
    --metric-data MetricName=ConvergenceTime,Value=$CONVERGENCE_TIME,Unit=Seconds \
                  MetricName=AppRecoveryTime,Value=$APP_RECOVERY_TIME,Unit=Seconds

METRICS_SCRIPT

chmod +x collect-failover-metrics.sh

echo ""
echo "6. RECOVERY VALIDATION"
cat << 'RECOVERY_VALIDATION'

Post-Failover Validation Checklist:
□ Primary path restored to service
□ BGP routing tables correct
□ Application connectivity verified
□ Performance metrics within SLA
□ Monitoring alerts cleared
□ Failover event documented

Recovery Time Objectives:
- BGP Convergence: < 30 seconds
- Application Recovery: < 60 seconds
- Full Service Restoration: < 5 minutes

Key Performance Indicators:
- Packet Loss during failover: < 0.1%
- Latency increase during failover: < 50ms
- Service availability: > 99.95%

RECOVERY_VALIDATION

echo ""
echo "HA FAILOVER TESTING SETUP COMPLETE"
echo "Run connectivity-test.sh in background during failover tests"
echo "Use collect-failover-metrics.sh to measure performance"
echo "Follow recovery validation checklist after each test"

EOF

chmod +x test-ha-failover.sh
./test-ha-failover.sh
```

## High Availability Best Practices

### Design Principles
- **Redundancy at Every Layer**: Multiple connections, gateways, and paths
- **Geographic Distribution**: Multi-AZ and multi-region deployment
- **Graceful Degradation**: Maintain partial service during failures
- **Automated Recovery**: Minimize manual intervention requirements

### BGP Optimization
- **Fast Convergence**: Aggressive timers and BFD implementation
- **Path Diversity**: Multiple independent network paths
- **Load Balancing**: ECMP for optimal bandwidth utilization
- **Route Filtering**: Prevent loops and control path selection

### Monitoring and Alerting
- **Proactive Monitoring**: Continuous health checking
- **Rapid Detection**: Sub-minute failure detection
- **Automated Response**: Self-healing where possible
- **Comprehensive Logging**: Full audit trail of events

### Operational Excellence
- **Regular Testing**: Scheduled failover testing
- **Documentation**: Detailed runbooks and procedures
- **Training**: Team preparation for failure scenarios
- **Continuous Improvement**: Learning from each incident

## Key Takeaways
- HA VPN requires redundancy at multiple layers
- BGP configuration is critical for automatic failover
- Monitoring and testing validate HA effectiveness
- Recovery time depends on BGP convergence speed
- Hybrid approaches provide optimal resilience
- Regular testing ensures HA readiness

## Additional Resources
- [AWS VPN High Availability Guide](https://docs.aws.amazon.com/vpn/latest/s2svpn/ha.html)
- [BGP Best Practices](https://docs.aws.amazon.com/directconnect/latest/UserGuide/routing-and-bgp.html)
- [Network Resilience on AWS](https://aws.amazon.com/architecture/well-architected/)

## Exam Tips
- Dual customer gateways provide site-level redundancy
- BGP local preference controls path selection
- BFD enables sub-second failure detection
- VPN + DX provides cost-effective hybrid HA
- Multi-region deployment protects against regional failures
- Route propagation must be enabled for automatic failover
- AS path prepending makes routes less preferred
- Health checks should monitor end-to-end connectivity