# Topic 81: AWS Transit Gateway with IPv6: Deployment Demo

## Overview

AWS Transit Gateway simplifies network connectivity by acting as a hub that connects VPCs, on-premises networks, and other AWS services. With IPv6 support, Transit Gateway enables scalable, future-proof network architectures that can handle massive address spaces efficiently. This topic provides a comprehensive deployment demonstration of Transit Gateway with IPv6 configuration.

## Transit Gateway IPv6 Architecture

### Core Concepts

```
Hub-and-Spoke Architecture with IPv6:

                    ┌─────────────────────────────────┐
                    │      Transit Gateway            │
                    │   (IPv6 + IPv4 Support)        │
                    │                                 │
                    │  Default Route Table            │
                    │  Propagation & Association      │
                    └─────────────┬───────────────────┘
                                  │
             ┌────────────────────┼────────────────────┐
             │                    │                    │
    ┌────────▼──────┐   ┌────────▼──────┐   ┌────────▼──────┐
    │   VPC-A       │   │   VPC-B       │   │  On-Premises  │
    │IPv4+IPv6      │   │IPv6-Only      │   │  IPv6 Network │
    │10.0.0.0/16    │   │               │   │               │
    │2001:db8:a::/48│   │2001:db8:b::/48│   │2001:db8::/32  │
    └───────────────┘   └───────────────┘   └───────────────┘
```

### Key Benefits for IPv6

1. **Simplified Routing**: Centralized routing for complex IPv6 networks
2. **Scalability**: Support for thousands of route entries
3. **Multi-VPC Connectivity**: Connect IPv6-only and dual-stack VPCs
4. **Hybrid Connectivity**: Bridge on-premises IPv6 networks
5. **Route Management**: Advanced routing policies and segmentation

## Prerequisites and Planning

### Requirements Assessment

```bash
# Check current Transit Gateway limits
aws ec2 describe-transit-gateways \
    --query 'TransitGateways[*].{TgwId:TransitGatewayId,State:State,AmazonSideAsn:AmazonSideAsn}'

# Check available IPv6 CIDRs in region
aws ec2 describe-vpcs \
    --query 'Vpcs[*].{VpcId:VpcId,Ipv6CidrBlocks:Ipv6CidrBlockAssociationSet[*].Ipv6CidrBlock}'
```

### Network Design Plan

```yaml
IPv6_Network_Design:
  Transit_Gateway:
    ASN: 64512
    Default_Route_Table: "auto-propagate"
    DNS_Support: enabled
    Multicast_Support: disabled

  Connected_VPCs:
    VPC_Production:
      CIDR_IPv4: "10.1.0.0/16"
      CIDR_IPv6: "2001:db8:prod::/48"
      Purpose: "Production workloads"

    VPC_Development:
      CIDR_IPv4: "10.2.0.0/16"
      CIDR_IPv6: "2001:db8:dev::/48"
      Purpose: "Development environment"

    VPC_SharedServices:
      CIDR_IPv4: "10.0.0.0/16"
      CIDR_IPv6: "2001:db8:shared::/48"
      Purpose: "DNS, monitoring, security services"

  Routing_Strategy:
    Production_to_Shared: "allow"
    Development_to_Shared: "allow"
    Production_to_Development: "deny"
    OnPremises_to_All: "allow"
```

## Hands-On Transit Gateway IPv6 Deployment

This section documents the complete hands-on deployment we performed, showing the actual steps and results from implementing Transit Gateway with IPv6 connectivity.

### Real-World Implementation: Dual Environment Setup

We implemented **two complete IPv6 Transit Gateway environments**:
1. **Production Environment**: VPC-A (10.0.0.0/16) ↔ VPC-B (10.1.0.0/16)
2. **Demo Environment**: Demo-VPC-A (10.10.0.0/16) ↔ Demo-VPC-B (10.11.0.0/16)

### Step 1: Create VPCs with IPv6 CIDR Blocks

**AWS Console Steps:**

1. Go to **VPC Console** → **Your VPCs** → **Create VPC**
2. Configure first VPC:
   - **Name**: `IPv6-TGW-VPC-A`
   - **IPv4 CIDR**: `10.0.0.0/16`
   - **IPv6 CIDR block**: Select **Amazon-provided IPv6 CIDR block**
   - **Tenancy**: Default

3. Create second VPC:
   - **Name**: `IPv6-TGW-VPC-B`
   - **IPv4 CIDR**: `10.1.0.0/16`
   - **IPv6 CIDR block**: Select **Amazon-provided IPv6 CIDR block**

**Results from our implementation:**
```bash
# VPC verification
aws ec2 describe-vpcs --filters "Name=cidr,Values=10.0.0.0/16,10.1.0.0/16" --output table

# Output showed:
# VPC-A: vpc-0eb79eb804c95ac20 | 10.0.0.0/16 | 2600:1f18:3352:e00::/56
# VPC-B: vpc-0f0ed78002894f2fe | 10.1.0.0/16 | 2600:1f18:587b:1a00::/56
```

**Key Learning:** Each VPC receives a unique `/56` IPv6 block, providing 256 possible `/64` subnets per VPC.

### Step 2: Create Dual-Stack Subnets

**AWS Console Steps:**

1. **VPC-A Subnet:**
   - **VPC**: `IPv6-TGW-VPC-A`
   - **Subnet name**: `VPC-A-Subnet`
   - **IPv4 CIDR**: `10.0.1.0/24`
   - **IPv6 CIDR**: `2600:1f18:3352:e01::/64` (first `/64` from VPC's `/56`)

2. **VPC-B Subnet:**
   - **VPC**: `IPv6-TGW-VPC-B`
   - **Subnet name**: `VPC-B-Subnet`
   - **IPv4 CIDR**: `10.1.1.0/24`
   - **IPv6 CIDR**: `2600:1f18:587b:1a01::/64`

3. **Enable Auto-Assign for Both:**
   - Select subnet → **Actions** → **Edit subnet settings**
   - ✅ **Auto-assign public IPv4 address**
   - ✅ **Auto-assign IPv6 address**

**IPv6 Subnet Structure Understanding:**
```
VPC CIDR:     2600:1f18:3352:e00::/56
Subnet CIDR:  2600:1f18:3352:e01::/64
                              ^^
                        Subnet ID (01)
```
The last 2 hex digits before `::` represent the subnet ID within the VPC's `/56` allocation.

### Step 3: Create Transit Gateway

**AWS Console Steps:**

1. Go to **VPC Console** → **Transit Gateways** → **Create transit gateway**
2. Configure:
   - **Name**: `IPv6-Demo-TGW`
   - **Amazon side ASN**: `64512`
   - **Default route table association**: `Enable`
   - **Default route table propagation**: `Enable`
   - **DNS support**: `Enable`

**Verification:**
```bash
# Transit Gateway status
aws ec2 describe-transit-gateways --transit-gateway-ids tgw-004afc54492872e90 --output table

# Result: State = "available", ASN = 64512
```

### Step 4: Attach VPCs to Transit Gateway

**AWS Console Steps:**

1. **VPC-A Attachment:**
   - Go to **Transit Gateway Attachments** → **Create attachment**
   - **Name**: `VPC-A-Attachment`
   - **Transit Gateway**: `IPv6-Demo-TGW`
   - **VPC**: `IPv6-TGW-VPC-A`
   - **Subnet**: `VPC-A-Subnet`

2. **VPC-B Attachment:**
   - **Name**: `VPC-B-Attachment`
   - **Transit Gateway**: `IPv6-Demo-TGW`
   - **VPC**: `IPv6-TGW-VPC-B`
   - **Subnet**: `VPC-B-Subnet`

**Verification Results:**
```bash
# Both attachments successful
# VPC-A: tgw-attach-04550fae47c7a7a4b → "available"
# VPC-B: tgw-attach-03bc8b7578427ab14 → "available"
```

### Step 5: Configure IPv6 Routing (Critical Step!)

**Problem Discovered:** IPv4 routes auto-propagated, but IPv6 routes required manual configuration.

#### 5A: Create IPv6 Static Routes in Transit Gateway

**AWS Console Steps:**

1. Go to **Transit Gateways** → **Route Tables** → Select default route table
2. **Routes** tab → **Create static route**

**Route 1:**
- **CIDR**: `2600:1f18:3352:e00::/56` (VPC-A IPv6 range)
- **Choose attachment**: `VPC-A-Attachment`

**Route 2:**
- **CIDR**: `2600:1f18:587b:1a00::/56` (VPC-B IPv6 range)
- **Choose attachment**: `VPC-B-Attachment`

#### 5B: Update VPC Route Tables (Critical for Inter-VPC Communication)

**Key Discovery:** VPC route tables needed **subnet-level routes** (`/64`), not VPC-level routes (`/56`).

**VPC-A Route Table Updates:**
- `10.1.0.0/16` → `tgw-004afc54492872e90` (IPv4 to VPC-B)
- `2600:1f18:587b:1a01::/64` → `tgw-004afc54492872e90` (IPv6 to VPC-B **subnet**)

**VPC-B Route Table Updates:**
- `10.0.0.0/16` → `tgw-004afc54492872e90` (IPv4 to VPC-A)
- `2600:1f18:3352:e01::/64` → `tgw-004afc54492872e90` (IPv6 to VPC-A **subnet**)

**Why /64 instead of /56?** Instances need specific subnet routes to know how to reach other subnets through Transit Gateway.

### Step 6: Create Security Groups for Inter-VPC Communication

**VPC-A Security Group:**
```
Inbound Rules:
- SSH: Port 22, Source: 10.1.0.0/16 (VPC-B IPv4)
- SSH: Port 22, Source: 2600:1f18:587b:1a00::/56 (VPC-B IPv6)
- ICMP: All ICMP, Source: 10.1.0.0/16
- ICMPv6: All ICMPv6, Source: 2600:1f18:587b:1a00::/56
```

**VPC-B Security Group:**
```
Inbound Rules:
- SSH: Port 22, Source: 10.0.0.0/16 (VPC-A IPv4)
- SSH: Port 22, Source: 2600:1f18:3352:e00::/56 (VPC-A IPv6)
- ICMP: All ICMP, Source: 10.0.0.0/16
- ICMPv6: All ICMPv6, Source: 2600:1f18:3352:e00::/56
```

### Step 7: Launch Test Instances

**Instance Results:**
```bash
# VPC-A Instance: i-09bb9219a331347ce
# - IPv4: 10.0.1.230
# - IPv6: 2600:1f18:3352:e01:20db:7571:612e:318e

# VPC-B Instance: i-0af58f82a37d96d40
# - IPv4: 10.1.1.31
# - IPv6: 2600:1f18:587b:1a01:920:6023:68a0:7db0
```

### Step 8: Test Inter-VPC IPv6 Connectivity

**Successful Results:**

**IPv4 Connectivity Test:**
```bash
[ec2-user@ip-10-0-1-230 ~]$ ping -c 4 10.1.1.31
PING 10.1.1.31 (10.1.1.31) 56(84) bytes of data.
64 bytes from 10.1.1.31: icmp_seq=1 ttl=126 time=1.75 ms
64 bytes from 10.1.1.31: icmp_seq=2 ttl=126 time=0.572 ms
# 100% success, TTL=126 indicates Transit Gateway routing
```

**IPv6 Connectivity Test:**
```bash
[ec2-user@ip-10-1-1-31 ~]$ ping6 -c 4 2600:1f18:3352:e01:20db:7571:612e:318e
64 bytes from 2600:1f18:3352:e01:20db:7571:612e:318e: icmp_seq=1 ttl=63 time=1.04 ms
64 bytes from 2600:1f18:3352:e01:20db:7571:612e:318e: icmp_seq=2 ttl=63 time=0.475 ms
# 100% success, TTL=63 confirms Transit Gateway routing
```

### Demo Environment Implementation

**Second Complete Environment Created:**

**Demo VPCs:**
- **Demo VPC-A**: `10.10.0.0/16` + `2600:1f18:239e:a200::/56`
- **Demo VPC-B**: `10.11.0.0/16` + `2600:1f18:60a8:5a00::/56`

**Demo Subnets:**
- **Demo VPC-A Subnet**: `10.10.1.0/24` + `2600:1f18:239e:a201::/64`
- **Demo VPC-B Subnet**: `10.11.1.0/24` + `2600:1f18:60a8:5a01::/64`

**Demo Transit Gateway:** `Demo-IPv6-TGW` with ASN `64513`

### Key Implementation Insights

1. **IPv6 Route Propagation**: Unlike IPv4, IPv6 routes require **manual static configuration** in Transit Gateway route tables.

2. **Subnet-Level Routing**: VPC route tables need `/64` subnet routes, not `/56` VPC-level routes for inter-VPC communication.

3. **Security Groups**: Require separate rules for IPv4 and IPv6 - there's no automatic translation.

4. **Address Assignment**: AWS automatically assigns IPv6 addresses from the `/64` subnet range with proper auto-configuration enabled.

5. **Troubleshooting**: The most common issue was missing subnet-level routes in VPC route tables.

### Verification Commands Used

```bash
# VPC and subnet verification
aws ec2 describe-vpcs --filters "Name=cidr,Values=10.0.0.0/16,10.1.0.0/16" --output table
aws ec2 describe-subnets --filters "Name=cidr-block,Values=10.0.1.0/24,10.1.1.0/24" --output table

# Transit Gateway verification
aws ec2 describe-transit-gateways --transit-gateway-ids tgw-004afc54492872e90 --output table
aws ec2 describe-transit-gateway-attachments --filters "Name=transit-gateway-id,Values=tgw-004afc54492872e90" --output table

# Route table verification
aws ec2 search-transit-gateway-routes --transit-gateway-route-table-id tgw-rtb-0e3b7cddb8c4ebafb --filters Name=type,Values=static,propagated --output table

# Instance verification
aws ec2 describe-instances --filters "Name=vpc-id,Values=vpc-0eb79eb804c95ac20,vpc-0f0ed78002894f2fe" --output table
```

### Architecture Achievement

✅ **Complete dual-stack inter-VPC connectivity**
✅ **Two isolated Transit Gateway environments**
✅ **IPv6 routing through centralized hub**
✅ **No NAT required for IPv6 traffic**
✅ **Enterprise-ready scalable architecture**

This hands-on implementation demonstrates production-ready IPv6 Transit Gateway connectivity with all the complexities and solutions encountered in real-world deployments.

### Step 2: Create IPv6-Enabled VPCs

```bash
#!/bin/bash
# Create multiple VPCs with IPv6 support

create_ipv6_vpc() {
    local vpc_name=$1
    local ipv4_cidr=$2
    local environment=$3

    echo "Creating VPC: $vpc_name"

    # Create VPC with IPv4 CIDR
    VPC_ID=$(aws ec2 create-vpc \
        --cidr-block $ipv4_cidr \
        --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=$vpc_name},{Key=Environment,Value=$environment}]" \
        --query 'Vpc.VpcId' \
        --output text)

    # Associate IPv6 CIDR
    aws ec2 associate-vpc-cidr-block \
        --vpc-id $VPC_ID \
        --amazon-provided-ipv6-cidr-block

    # Wait for IPv6 CIDR association
    sleep 30

    # Get IPv6 CIDR
    IPV6_CIDR=$(aws ec2 describe-vpcs \
        --vpc-ids $VPC_ID \
        --query 'Vpcs[0].Ipv6CidrBlockAssociationSet[0].Ipv6CidrBlock' \
        --output text)\n    \n    echo \"VPC $vpc_name created: $VPC_ID with IPv6: $IPV6_CIDR\"\n    \n    # Enable DNS hostnames and resolution\n    aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames\n    aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support\n    \n    echo $VPC_ID\n}\n\n# Create VPCs\nPROD_VPC_ID=$(create_ipv6_vpc \"Production-VPC\" \"10.1.0.0/16\" \"Production\")\nDEV_VPC_ID=$(create_ipv6_vpc \"Development-VPC\" \"10.2.0.0/16\" \"Development\")\nSHARED_VPC_ID=$(create_ipv6_vpc \"Shared-Services-VPC\" \"10.0.0.0/16\" \"Shared\")\n\necho \"VPCs created:\"\necho \"Production: $PROD_VPC_ID\"\necho \"Development: $DEV_VPC_ID\"\necho \"Shared Services: $SHARED_VPC_ID\"\n```\n\n### Step 3: Create Subnets with IPv6\n\n```bash\n#!/bin/bash\n# Create subnets in each VPC with IPv6 support\n\ncreate_ipv6_subnets() {\n    local vpc_id=$1\n    local vpc_name=$2\n    local ipv4_base=$3\n    local ipv6_base=$4\n    \n    echo \"Creating subnets for $vpc_name ($vpc_id)\"\n    \n    # Public subnet\n    PUBLIC_SUBNET_ID=$(aws ec2 create-subnet \\\n        --vpc-id $vpc_id \\\n        --cidr-block \"${ipv4_base}.1.0/24\" \\\n        --ipv6-cidr-block \"${ipv6_base}:1::/64\" \\\n        --availability-zone us-east-1a \\\n        --tag-specifications \"ResourceType=subnet,Tags=[{Key=Name,Value=$vpc_name-Public-1a}]\" \\\n        --query 'Subnet.SubnetId' \\\n        --output text)\n    \n    # Private subnet\n    PRIVATE_SUBNET_ID=$(aws ec2 create-subnet \\\n        --vpc-id $vpc_id \\\n        --cidr-block \"${ipv4_base}.2.0/24\" \\\n        --ipv6-cidr-block \"${ipv6_base}:2::/64\" \\\n        --availability-zone us-east-1a \\\n        --tag-specifications \"ResourceType=subnet,Tags=[{Key=Name,Value=$vpc_name-Private-1a}]\" \\\n        --query 'Subnet.SubnetId' \\\n        --output text)\n    \n    # Enable auto-assign IPv6\n    aws ec2 modify-subnet-attribute --subnet-id $PUBLIC_SUBNET_ID --assign-ipv6-address-on-creation\n    aws ec2 modify-subnet-attribute --subnet-id $PRIVATE_SUBNET_ID --assign-ipv6-address-on-creation\n    \n    echo \"$vpc_name subnets created: Public=$PUBLIC_SUBNET_ID, Private=$PRIVATE_SUBNET_ID\"\n}\n\n# Get IPv6 CIDRs for each VPC\nPROD_IPV6=$(aws ec2 describe-vpcs --vpc-ids $PROD_VPC_ID --query 'Vpcs[0].Ipv6CidrBlockAssociationSet[0].Ipv6CidrBlock' --output text | cut -d':' -f1-3)\nDEV_IPV6=$(aws ec2 describe-vpcs --vpc-ids $DEV_VPC_ID --query 'Vpcs[0].Ipv6CidrBlockAssociationSet[0].Ipv6CidrBlock' --output text | cut -d':' -f1-3)\nSHARED_IPV6=$(aws ec2 describe-vpcs --vpc-ids $SHARED_VPC_ID --query 'Vpcs[0].Ipv6CidrBlockAssociationSet[0].Ipv6CidrBlock' --output text | cut -d':' -f1-3)\n\n# Create subnets\ncreate_ipv6_subnets $PROD_VPC_ID \"Production\" \"10.1\" \"$PROD_IPV6\"\ncreate_ipv6_subnets $DEV_VPC_ID \"Development\" \"10.2\" \"$DEV_IPV6\"\ncreate_ipv6_subnets $SHARED_VPC_ID \"Shared\" \"10.0\" \"$SHARED_IPV6\"\n```\n\n### Step 4: Attach VPCs to Transit Gateway\n\n```bash\n#!/bin/bash\n# Attach VPCs to Transit Gateway\n\nattach_vpc_to_tgw() {\n    local vpc_id=$1\n    local vpc_name=$2\n    local subnet_id=$3\n    \n    echo \"Attaching $vpc_name VPC to Transit Gateway...\"\n    \n    ATTACHMENT_ID=$(aws ec2 create-transit-gateway-vpc-attachment \\\n        --transit-gateway-id $TGW_ID \\\n        --vpc-id $vpc_id \\\n        --subnet-ids $subnet_id \\\n        --tag-specifications \"ResourceType=transit-gateway-attachment,Tags=[{Key=Name,Value=$vpc_name-TGW-Attachment}]\" \\\n        --query 'TransitGatewayVpcAttachment.TransitGatewayAttachmentId' \\\n        --output text)\n    \n    echo \"$vpc_name attached with ID: $ATTACHMENT_ID\"\n    \n    # Wait for attachment to be available\n    aws ec2 wait transit-gateway-attachment-available --transit-gateway-attachment-ids $ATTACHMENT_ID\n    \n    echo $ATTACHMENT_ID\n}\n\n# Get private subnet IDs for attachments\nPROD_PRIVATE_SUBNET=$(aws ec2 describe-subnets --filters \"Name=vpc-id,Values=$PROD_VPC_ID\" \"Name=tag:Name,Values=Production-Private-1a\" --query 'Subnets[0].SubnetId' --output text)\nDEV_PRIVATE_SUBNET=$(aws ec2 describe-subnets --filters \"Name=vpc-id,Values=$DEV_VPC_ID\" \"Name=tag:Name,Values=Development-Private-1a\" --query 'Subnets[0].SubnetId' --output text)\nSHARED_PRIVATE_SUBNET=$(aws ec2 describe-subnets --filters \"Name=vpc-id,Values=$SHARED_VPC_ID\" \"Name=tag:Name,Values=Shared-Private-1a\" --query 'Subnets[0].SubnetId' --output text)\n\n# Attach VPCs\nPROD_ATTACHMENT=$(attach_vpc_to_tgw $PROD_VPC_ID \"Production\" $PROD_PRIVATE_SUBNET)\nDEV_ATTACHMENT=$(attach_vpc_to_tgw $DEV_VPC_ID \"Development\" $DEV_PRIVATE_SUBNET)\nSHARED_ATTACHMENT=$(attach_vpc_to_tgw $SHARED_VPC_ID \"Shared\" $SHARED_PRIVATE_SUBNET)\n\necho \"All VPCs attached to Transit Gateway\"\n```\n\n### Step 5: Configure IPv6 Routing\n\n```bash\n#!/bin/bash\n# Configure IPv6 routes in Transit Gateway\n\necho \"Configuring IPv6 routes in Transit Gateway...\"\n\n# Add IPv6 routes for each VPC to the default route table\nadd_ipv6_route() {\n    local destination_cidr=$1\n    local attachment_id=$2\n    local description=$3\n    \n    echo \"Adding route: $destination_cidr -> $attachment_id ($description)\"\n    \n    aws ec2 create-route \\\n        --route-table-id $DEFAULT_RT_ID \\\n        --destination-ipv6-cidr-block $destination_cidr \\\n        --transit-gateway-attachment-id $attachment_id\n}\n\n# Get IPv6 CIDRs for routing\nPROD_IPV6_CIDR=$(aws ec2 describe-vpcs --vpc-ids $PROD_VPC_ID --query 'Vpcs[0].Ipv6CidrBlockAssociationSet[0].Ipv6CidrBlock' --output text)\nDEV_IPV6_CIDR=$(aws ec2 describe-vpcs --vpc-ids $DEV_VPC_ID --query 'Vpcs[0].Ipv6CidrBlockAssociationSet[0].Ipv6CidrBlock' --output text)\nSHARED_IPV6_CIDR=$(aws ec2 describe-vpcs --vpc-ids $SHARED_VPC_ID --query 'Vpcs[0].Ipv6CidrBlockAssociationSet[0].Ipv6CidrBlock' --output text)\n\n# Add IPv6 routes (these are typically auto-propagated, but showing explicit configuration)\necho \"IPv6 routes will be auto-propagated by default route table association\"\necho \"Production IPv6: $PROD_IPV6_CIDR\"\necho \"Development IPv6: $DEV_IPV6_CIDR\"\necho \"Shared Services IPv6: $SHARED_IPV6_CIDR\"\n```\n\n### Step 6: Update VPC Route Tables\n\n```bash\n#!/bin/bash\n# Update VPC route tables to use Transit Gateway for inter-VPC traffic\n\nupdate_vpc_routes() {\n    local vpc_id=$1\n    local vpc_name=$2\n    \n    echo \"Updating route tables for $vpc_name VPC...\"\n    \n    # Get route table IDs\n    ROUTE_TABLES=$(aws ec2 describe-route-tables \\\n        --filters \"Name=vpc-id,Values=$vpc_id\" \\\n        --query 'RouteTables[*].RouteTableId' \\\n        --output text)\n    \n    for RT_ID in $ROUTE_TABLES; do\n        echo \"Updating route table: $RT_ID\"\n        \n        # Add IPv6 routes to other VPCs via Transit Gateway\n        if [[ \"$vpc_id\" != \"$PROD_VPC_ID\" ]]; then\n            aws ec2 create-route \\\n                --route-table-id $RT_ID \\\n                --destination-ipv6-cidr-block $PROD_IPV6_CIDR \\\n                --transit-gateway-id $TGW_ID 2>/dev/null || echo \"Route may already exist\"\n        fi\n        \n        if [[ \"$vpc_id\" != \"$DEV_VPC_ID\" ]]; then\n            aws ec2 create-route \\\n                --route-table-id $RT_ID \\\n                --destination-ipv6-cidr-block $DEV_IPV6_CIDR \\\n                --transit-gateway-id $TGW_ID 2>/dev/null || echo \"Route may already exist\"\n        fi\n        \n        if [[ \"$vpc_id\" != \"$SHARED_VPC_ID\" ]]; then\n            aws ec2 create-route \\\n                --route-table-id $RT_ID \\\n                --destination-ipv6-cidr-block $SHARED_IPV6_CIDR \\\n                --transit-gateway-id $TGW_ID 2>/dev/null || echo \"Route may already exist\"\n        fi\n    done\n}\n\n# Update all VPC route tables\nupdate_vpc_routes $PROD_VPC_ID \"Production\"\nupdate_vpc_routes $DEV_VPC_ID \"Development\" \nupdate_vpc_routes $SHARED_VPC_ID \"Shared\"\n\necho \"VPC route tables updated\"\n```\n\n## Advanced Routing Scenarios\n\n### Custom Route Tables for Segmentation\n\n```bash\n#!/bin/bash\n# Create custom route tables for network segmentation\n\necho \"Creating custom route tables for advanced segmentation...\"\n\n# Create Production route table\nPROD_RT_ID=$(aws ec2 create-transit-gateway-route-table \\\n    --transit-gateway-id $TGW_ID \\\n    --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=Production-RouteTable}]' \\\n    --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' \\\n    --output text)\n\n# Create Development route table\nDEV_RT_ID=$(aws ec2 create-transit-gateway-route-table \\\n    --transit-gateway-id $TGW_ID \\\n    --tag-specifications 'ResourceType=transit-gateway-route-table,Tags=[{Key=Name,Value=Development-RouteTable}]' \\\n    --query 'TransitGatewayRouteTable.TransitGatewayRouteTableId' \\\n    --output text)\n\necho \"Custom route tables created:\"\necho \"Production: $PROD_RT_ID\"\necho \"Development: $DEV_RT_ID\"\n\n# Associate attachments with custom route tables\naws ec2 associate-transit-gateway-route-table \\\n    --transit-gateway-attachment-id $PROD_ATTACHMENT \\\n    --transit-gateway-route-table-id $PROD_RT_ID\n\naws ec2 associate-transit-gateway-route-table \\\n    --transit-gateway-attachment-id $DEV_ATTACHMENT \\\n    --transit-gateway-route-table-id $DEV_RT_ID\n\n# Configure selective routing\necho \"Configuring selective IPv6 routing...\"\n\n# Production can access Shared Services only\naws ec2 create-route \\\n    --route-table-id $PROD_RT_ID \\\n    --destination-ipv6-cidr-block $SHARED_IPV6_CIDR \\\n    --transit-gateway-attachment-id $SHARED_ATTACHMENT\n\n# Development can access Shared Services only  \naws ec2 create-route \\\n    --route-table-id $DEV_RT_ID \\\n    --destination-ipv6-cidr-block $SHARED_IPV6_CIDR \\\n    --transit-gateway-attachment-id $SHARED_ATTACHMENT\n\n# Shared Services can access all environments\naws ec2 propagate-transit-gateway-route-table \\\n    --transit-gateway-route-table-id $DEFAULT_RT_ID \\\n    --transit-gateway-attachment-id $PROD_ATTACHMENT\n\naws ec2 propagate-transit-gateway-route-table \\\n    --transit-gateway-route-table-id $DEFAULT_RT_ID \\\n    --transit-gateway-attachment-id $DEV_ATTACHMENT\n\necho \"Advanced routing configuration complete\"\n```\n\n### Route Analysis and Monitoring\n\n```bash\n#!/bin/bash\n# Analyze Transit Gateway IPv6 routes\n\nanalyze_tgw_routes() {\n    local rt_id=$1\n    local rt_name=$2\n    \n    echo \"Analyzing routes for $rt_name ($rt_id):\"\n    echo \"======================================\"\n    \n    # Get all routes\n    aws ec2 search-transit-gateway-routes \\\n        --transit-gateway-route-table-id $rt_id \\\n        --filters \"Name=state,Values=active\" \\\n        --query 'Routes[*].{Destination:DestinationCidrBlock,IPv6Destination:DestinationIpv6CidrBlock,Type:Type,State:State,AttachmentId:TransitGatewayAttachments[0].TransitGatewayAttachmentId}' \\\n        --output table\n    \n    echo \"\"\n}\n\n# Analyze all route tables\nanalyze_tgw_routes $DEFAULT_RT_ID \"Default Route Table\"\nanalyze_tgw_routes $PROD_RT_ID \"Production Route Table\"\nanalyze_tgw_routes $DEV_RT_ID \"Development Route Table\"\n\n# Monitor Transit Gateway metrics\necho \"Setting up CloudWatch monitoring for Transit Gateway IPv6 traffic...\"\n\n# Create CloudWatch dashboard\naws cloudwatch put-dashboard \\\n    --dashboard-name \"TGW-IPv6-Monitoring\" \\\n    --dashboard-body '{\n        \"widgets\": [\n            {\n                \"type\": \"metric\",\n                \"properties\": {\n                    \"metrics\": [\n                        [\"AWS/TransitGateway\", \"BytesIn\", \"TransitGateway\", \"'$TGW_ID'\"],\n                        [\".\", \"BytesOut\", \".\", \".\"],\n                        [\".\", \"PacketDropCount\", \".\", \".\"]\n                    ],\n                    \"period\": 300,\n                    \"stat\": \"Sum\",\n                    \"region\": \"us-east-1\",\n                    \"title\": \"Transit Gateway Traffic\"\n                }\n            }\n        ]\n    }'\n```\n\n## Testing and Validation\n\n### Connectivity Testing\n\n```bash\n#!/bin/bash\n# Test IPv6 connectivity through Transit Gateway\n\ntest_ipv6_connectivity() {\n    echo \"Testing IPv6 connectivity through Transit Gateway...\"\n    echo \"====================================================\"\n    \n    # Deploy test instances in each VPC\n    echo \"Deploying test instances...\"\n    \n    # Get public subnets for test instances\n    PROD_PUBLIC_SUBNET=$(aws ec2 describe-subnets --filters \"Name=vpc-id,Values=$PROD_VPC_ID\" \"Name=tag:Name,Values=Production-Public-1a\" --query 'Subnets[0].SubnetId' --output text)\n    DEV_PUBLIC_SUBNET=$(aws ec2 describe-subnets --filters \"Name=vpc-id,Values=$DEV_VPC_ID\" \"Name=tag:Name,Values=Development-Public-1a\" --query 'Subnets[0].SubnetId' --output text)\n    \n    # Launch test instances with IPv6\n    PROD_INSTANCE=$(aws ec2 run-instances \\\n        --image-id ami-0c55b159cbfafe1d0 \\\n        --count 1 \\\n        --instance-type t3.micro \\\n        --subnet-id $PROD_PUBLIC_SUBNET \\\n        --ipv6-address-count 1 \\\n        --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Production-Test}]' \\\n        --query 'Instances[0].InstanceId' \\\n        --output text)\n    \n    DEV_INSTANCE=$(aws ec2 run-instances \\\n        --image-id ami-0c55b159cbfafe1d0 \\\n        --count 1 \\\n        --instance-type t3.micro \\\n        --subnet-id $DEV_PUBLIC_SUBNET \\\n        --ipv6-address-count 1 \\\n        --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Development-Test}]' \\\n        --query 'Instances[0].InstanceId' \\\n        --output text)\n    \n    echo \"Test instances launched:\"\n    echo \"Production: $PROD_INSTANCE\"\n    echo \"Development: $DEV_INSTANCE\"\n    \n    # Wait for instances to be running\n    aws ec2 wait instance-running --instance-ids $PROD_INSTANCE $DEV_INSTANCE\n    \n    # Get IPv6 addresses\n    PROD_IPV6=$(aws ec2 describe-instances --instance-ids $PROD_INSTANCE --query 'Reservations[0].Instances[0].NetworkInterfaces[0].Ipv6Addresses[0].Ipv6Address' --output text)\n    DEV_IPV6=$(aws ec2 describe-instances --instance-ids $DEV_INSTANCE --query 'Reservations[0].Instances[0].NetworkInterfaces[0].Ipv6Addresses[0].Ipv6Address' --output text)\n    \n    echo \"IPv6 addresses:\"\n    echo \"Production: $PROD_IPV6\"\n    echo \"Development: $DEV_IPV6\"\n    \n    # Test connectivity would require SSH access and security group configuration\n    echo \"Manual testing required:\"\n    echo \"1. Configure security groups to allow SSH and ICMP\"\n    echo \"2. SSH to instances and test ping6 between IPv6 addresses\"\n    echo \"3. Verify traffic flows through Transit Gateway\"\n}\n\ntest_ipv6_connectivity\n```\n\n### Performance Testing\n\n```python\n#!/usr/bin/env python3\n# Performance testing for IPv6 transit gateway connectivity\n\nimport boto3\nimport time\nimport json\nfrom datetime import datetime, timedelta\n\nclass TGWPerformanceMonitor:\n    def __init__(self, tgw_id):\n        self.tgw_id = tgw_id\n        self.cloudwatch = boto3.client('cloudwatch')\n        self.ec2 = boto3.client('ec2')\n    \n    def get_tgw_metrics(self, hours=1):\n        \"\"\"Get Transit Gateway performance metrics\"\"\"\n        end_time = datetime.utcnow()\n        start_time = end_time - timedelta(hours=hours)\n        \n        metrics = [\n            'BytesIn', 'BytesOut', 'PacketsIn', 'PacketsOut', \n            'PacketDropCount', 'ActiveFlowCount_IPv6'\n        ]\n        \n        results = {}\n        \n        for metric in metrics:\n            try:\n                response = self.cloudwatch.get_metric_statistics(\n                    Namespace='AWS/TransitGateway',\n                    MetricName=metric,\n                    Dimensions=[\n                        {\n                            'Name': 'TransitGateway',\n                            'Value': self.tgw_id\n                        }\n                    ],\n                    StartTime=start_time,\n                    EndTime=end_time,\n                    Period=300,\n                    Statistics=['Sum', 'Average', 'Maximum']\n                )\n                results[metric] = response['Datapoints']\n            except Exception as e:\n                print(f\"Error getting {metric}: {e}\")\n                results[metric] = []\n        \n        return results\n    \n    def analyze_performance(self):\n        \"\"\"Analyze Transit Gateway IPv6 performance\"\"\"\n        print(f\"Analyzing performance for Transit Gateway: {self.tgw_id}\")\n        print(\"=\" * 60)\n        \n        metrics = self.get_tgw_metrics()\n        \n        for metric_name, datapoints in metrics.items():\n            if datapoints:\n                latest = sorted(datapoints, key=lambda x: x['Timestamp'])[-1]\n                print(f\"{metric_name}:\")\n                print(f\"  Latest Value: {latest.get('Sum', latest.get('Average', 0))}\")\n                print(f\"  Timestamp: {latest['Timestamp']}\")\n                print()\n    \n    def check_route_propagation(self):\n        \"\"\"Check IPv6 route propagation status\"\"\"\n        print(\"Checking IPv6 route propagation...\")\n        print(\"=\" * 40)\n        \n        # Get route tables\n        route_tables = self.ec2.describe_transit_gateway_route_tables(\n            Filters=[\n                {\n                    'Name': 'transit-gateway-id',\n                    'Values': [self.tgw_id]\n                }\n            ]\n        )\n        \n        for rt in route_tables['TransitGatewayRouteTables']:\n            rt_id = rt['TransitGatewayRouteTableId']\n            print(f\"Route Table: {rt_id}\")\n            \n            # Search for IPv6 routes\n            routes = self.ec2.search_transit_gateway_routes(\n                TransitGatewayRouteTableId=rt_id,\n                Filters=[\n                    {\n                        'Name': 'type',\n                        'Values': ['propagated', 'static']\n                    }\n                ]\n            )\n            \n            ipv6_routes = [r for r in routes['Routes'] if 'DestinationIpv6CidrBlock' in r]\n            print(f\"  IPv6 Routes: {len(ipv6_routes)}\")\n            \n            for route in ipv6_routes:\n                print(f\"    {route.get('DestinationIpv6CidrBlock', 'N/A')} -> {route['Type']}\")\n            print()\n\n# Usage\nif __name__ == \"__main__\":\n    import sys\n    if len(sys.argv) != 2:\n        print(\"Usage: python3 tgw_monitor.py <transit-gateway-id>\")\n        sys.exit(1)\n    \n    tgw_id = sys.argv[1]\n    monitor = TGWPerformanceMonitor(tgw_id)\n    monitor.analyze_performance()\n    monitor.check_route_propagation()\n```\n\n## Troubleshooting Common Issues\n\n### Route Propagation Issues\n\n```bash\n#!/bin/bash\n# Troubleshoot IPv6 route propagation issues\n\ntroubleshoot_routes() {\n    local tgw_id=$1\n    \n    echo \"Troubleshooting IPv6 routes for TGW: $tgw_id\"\n    echo \"=============================================\"\n    \n    # Check attachment status\n    echo \"1. Checking attachment status...\"\n    aws ec2 describe-transit-gateway-attachments \\\n        --filters \"Name=transit-gateway-id,Values=$tgw_id\" \\\n        --query 'TransitGatewayAttachments[*].{AttachmentId:TransitGatewayAttachmentId,State:State,ResourceType:ResourceType,ResourceId:ResourceId}' \\\n        --output table\n    \n    echo \"\"\n    \n    # Check route table associations\n    echo \"2. Checking route table associations...\"\n    ROUTE_TABLES=$(aws ec2 describe-transit-gateway-route-tables \\\n        --filters \"Name=transit-gateway-id,Values=$tgw_id\" \\\n        --query 'TransitGatewayRouteTables[*].TransitGatewayRouteTableId' \\\n        --output text)\n    \n    for RT_ID in $ROUTE_TABLES; do\n        echo \"Route Table: $RT_ID\"\n        \n        # Check associations\n        aws ec2 get-transit-gateway-route-table-associations \\\n            --transit-gateway-route-table-id $RT_ID \\\n            --query 'Associations[*].{AttachmentId:TransitGatewayAttachmentId,ResourceType:ResourceType,State:State}' \\\n            --output table\n        \n        echo \"\"\n        \n        # Check propagations\n        echo \"Propagations for $RT_ID:\"\n        aws ec2 get-transit-gateway-route-table-propagations \\\n            --transit-gateway-route-table-id $RT_ID \\\n            --query 'TransitGatewayRouteTablePropagations[*].{AttachmentId:TransitGatewayAttachmentId,ResourceType:ResourceType,State:State}' \\\n            --output table\n        \n        echo \"\"\n    done\n    \n    # Check for conflicting routes\n    echo \"3. Checking for route conflicts...\"\n    for RT_ID in $ROUTE_TABLES; do\n        echo \"Checking routes in $RT_ID:\"\n        aws ec2 search-transit-gateway-routes \\\n            --transit-gateway-route-table-id $RT_ID \\\n            --filters \"Name=state,Values=active,blackhole\" \\\n            --query 'Routes[*].{IPv6Dest:DestinationIpv6CidrBlock,IPv4Dest:DestinationCidrBlock,Type:Type,State:State}' \\\n            --output table\n        echo \"\"\n    done\n}\n\n# Check VPC route tables\ncheck_vpc_routes() {\n    local vpc_id=$1\n    local vpc_name=$2\n    \n    echo \"Checking VPC routes for $vpc_name ($vpc_id):\"\n    echo \"==========================================\"\n    \n    ROUTE_TABLES=$(aws ec2 describe-route-tables \\\n        --filters \"Name=vpc-id,Values=$vpc_id\" \\\n        --query 'RouteTables[*].RouteTableId' \\\n        --output text)\n    \n    for RT_ID in $ROUTE_TABLES; do\n        echo \"VPC Route Table: $RT_ID\"\n        aws ec2 describe-route-tables \\\n            --route-table-ids $RT_ID \\\n            --query 'RouteTables[0].Routes[*].{Destination:DestinationCidrBlock,IPv6Destination:DestinationIpv6CidrBlock,Target:GatewayId,TGW:TransitGatewayId,State:State}' \\\n            --output table\n        echo \"\"\n    done\n}\n\n# Usage\ntroubleshoot_routes $TGW_ID\ncheck_vpc_routes $PROD_VPC_ID \"Production\"\ncheck_vpc_routes $DEV_VPC_ID \"Development\"\ncheck_vpc_routes $SHARED_VPC_ID \"Shared\"\n```\n\n### Performance Diagnostics\n\n```bash\n#!/bin/bash\n# Diagnose Transit Gateway IPv6 performance issues\n\nperformance_diagnostics() {\n    local tgw_id=$1\n    \n    echo \"Performance Diagnostics for TGW: $tgw_id\"\n    echo \"=======================================\"\n    \n    # Check current limits\n    echo \"1. Checking service limits...\"\n    aws service-quotas get-service-quota \\\n        --service-code ec2 \\\n        --quota-code L-A2B9B5E8 \\\n        --query 'Quota.{Name:QuotaName,Value:Value,Unit:Unit}' \\\n        --output table\n    \n    echo \"\"\n    \n    # Check attachment count\n    ATTACHMENT_COUNT=$(aws ec2 describe-transit-gateway-attachments \\\n        --filters \"Name=transit-gateway-id,Values=$tgw_id\" \\\n        --query 'length(TransitGatewayAttachments[?State==`available`])')\n    \n    echo \"Active attachments: $ATTACHMENT_COUNT\"\n    \n    # Check route table count and routes\n    ROUTE_TABLE_COUNT=$(aws ec2 describe-transit-gateway-route-tables \\\n        --filters \"Name=transit-gateway-id,Values=$tgw_id\" \\\n        --query 'length(TransitGatewayRouteTables[?State==`available`])')\n    \n    echo \"Route tables: $ROUTE_TABLE_COUNT\"\n    \n    # Check total route count\n    ROUTE_TABLES=$(aws ec2 describe-transit-gateway-route-tables \\\n        --filters \"Name=transit-gateway-id,Values=$tgw_id\" \\\n        --query 'TransitGatewayRouteTables[?State==`available`].TransitGatewayRouteTableId' \\\n        --output text)\n    \n    TOTAL_ROUTES=0\n    for RT_ID in $ROUTE_TABLES; do\n        ROUTE_COUNT=$(aws ec2 search-transit-gateway-routes \\\n            --transit-gateway-route-table-id $RT_ID \\\n            --filters \"Name=state,Values=active\" \\\n            --query 'length(Routes)')\n        TOTAL_ROUTES=$((TOTAL_ROUTES + ROUTE_COUNT))\n    done\n    \n    echo \"Total routes: $TOTAL_ROUTES\"\n    \n    # Performance recommendations\n    echo \"\"\n    echo \"Performance Recommendations:\"\n    echo \"=============================\"\n    \n    if [ $ATTACHMENT_COUNT -gt 100 ]; then\n        echo \"⚠️  High attachment count may impact performance\"\n    fi\n    \n    if [ $TOTAL_ROUTES -gt 1000 ]; then\n        echo \"⚠️  High route count may impact performance\"\n    fi\n    \n    if [ $ROUTE_TABLE_COUNT -gt 20 ]; then\n        echo \"⚠️  Consider consolidating route tables\"\n    fi\n    \n    echo \"✅ Monitor CloudWatch metrics for packet drops\"\n    echo \"✅ Use VPC Flow Logs to analyze traffic patterns\"\n    echo \"✅ Consider AWS Transit Gateway Network Manager for insights\"\n}\n\nperformance_diagnostics $TGW_ID\n```\n\n## Security and Compliance\n\n### Security Best Practices\n\n```bash\n#!/bin/bash\n# Implement security best practices for Transit Gateway IPv6\n\nsecure_tgw_ipv6() {\n    local tgw_id=$1\n    \n    echo \"Implementing IPv6 security best practices for TGW: $tgw_id\"\n    echo \"==========================================================\"\n    \n    # Enable VPC Flow Logs for all attached VPCs\n    echo \"1. Enabling VPC Flow Logs...\"\n    \n    ATTACHED_VPCS=$(aws ec2 describe-transit-gateway-attachments \\\n        --filters \"Name=transit-gateway-id,Values=$tgw_id\" \"Name=resource-type,Values=vpc\" \\\n        --query 'TransitGatewayAttachments[?State==`available`].ResourceId' \\\n        --output text)\n    \n    for VPC_ID in $ATTACHED_VPCS; do\n        echo \"Enabling flow logs for VPC: $VPC_ID\"\n        \n        aws ec2 create-flow-logs \\\n            --resource-type VPC \\\n            --resource-ids $VPC_ID \\\n            --traffic-type ALL \\\n            --log-destination-type cloud-watch-logs \\\n            --log-group-name \"/aws/tgw/flowlogs/$VPC_ID\" \\\n            --deliver-logs-permission-arn \"arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/flowlogsRole\" 2>/dev/null || echo \"Flow logs may already exist\"\n    done\n    \n    # Create security monitoring CloudWatch alarms\n    echo \"2. Creating security monitoring alarms...\"\n    \n    # Alarm for unusual traffic patterns\n    aws cloudwatch put-metric-alarm \\\n        --alarm-name \"TGW-IPv6-High-Traffic\" \\\n        --alarm-description \"High IPv6 traffic through Transit Gateway\" \\\n        --metric-name BytesIn \\\n        --namespace AWS/TransitGateway \\\n        --statistic Sum \\\n        --period 300 \\\n        --threshold 1000000000 \\\n        --comparison-operator GreaterThanThreshold \\\n        --dimensions Name=TransitGateway,Value=$tgw_id \\\n        --evaluation-periods 2\n    \n    # Alarm for packet drops\n    aws cloudwatch put-metric-alarm \\\n        --alarm-name \"TGW-IPv6-Packet-Drops\" \\\n        --alarm-description \"Packet drops in Transit Gateway\" \\\n        --metric-name PacketDropCount \\\n        --namespace AWS/TransitGateway \\\n        --statistic Sum \\\n        --period 300 \\\n        --threshold 1000 \\\n        --comparison-operator GreaterThanThreshold \\\n        --dimensions Name=TransitGateway,Value=$tgw_id \\\n        --evaluation-periods 1\n    \n    echo \"3. Security recommendations:\"\n    echo \"   ✅ Use custom route tables for network segmentation\"\n    echo \"   ✅ Implement least privilege routing policies\"\n    echo \"   ✅ Regular audit of route propagations\"\n    echo \"   ✅ Monitor for unauthorized route advertisements\"\n    echo \"   ✅ Use AWS Config rules for compliance monitoring\"\n}\n\nsecure_tgw_ipv6 $TGW_ID\n```\n\n## Cost Optimization\n\n### Cost Analysis\n\n```python\n#!/usr/bin/env python3\n# Analyze Transit Gateway IPv6 costs\n\nimport boto3\nfrom datetime import datetime, timedelta\nimport json\n\nclass TGWCostAnalyzer:\n    def __init__(self, tgw_id):\n        self.tgw_id = tgw_id\n        self.ce = boto3.client('ce')  # Cost Explorer\n        self.ec2 = boto3.client('ec2')\n        \n    def get_tgw_costs(self, days=30):\n        \"\"\"Get Transit Gateway costs for the past N days\"\"\"\n        end_date = datetime.now().date()\n        start_date = end_date - timedelta(days=days)\n        \n        response = self.ce.get_cost_and_usage(\n            TimePeriod={\n                'Start': start_date.strftime('%Y-%m-%d'),\n                'End': end_date.strftime('%Y-%m-%d')\n            },\n            Granularity='DAILY',\n            Metrics=['BlendedCost'],\n            GroupBy=[\n                {\n                    'Type': 'DIMENSION',\n                    'Key': 'SERVICE'\n                }\n            ],\n            Filter={\n                'Dimensions': {\n                    'Key': 'SERVICE',\n                    'Values': ['Amazon Elastic Compute Cloud - Compute']\n                }\n            }\n        )\n        \n        return response\n    \n    def analyze_attachment_costs(self):\n        \"\"\"Analyze costs by attachment type and count\"\"\"\n        attachments = self.ec2.describe_transit_gateway_attachments(\n            Filters=[\n                {\n                    'Name': 'transit-gateway-id',\n                    'Values': [self.tgw_id]\n                }\n            ]\n        )\n        \n        cost_breakdown = {\n            'vpc_attachments': 0,\n            'vpn_attachments': 0,\n            'direct_connect_attachments': 0,\n            'peering_attachments': 0\n        }\n        \n        attachment_costs = {\n            'vpc': 0.05,  # $0.05 per hour per VPC attachment\n            'vpn': 0.05,  # $0.05 per hour per VPN attachment  \n            'direct-connect-gateway': 0.05,  # $0.05 per hour per DX attachment\n            'peering': 0.05  # $0.05 per hour per peering attachment\n        }\n        \n        for attachment in attachments['TransitGatewayAttachments']:\n            if attachment['State'] == 'available':\n                resource_type = attachment['ResourceType']\n                if resource_type in attachment_costs:\n                    cost_breakdown[f\"{resource_type.replace('-', '_')}_attachments\"] += 1\n        \n        # Calculate monthly costs\n        monthly_costs = {}\n        hours_per_month = 24 * 30  # Approximate\n        \n        for attachment_type, count in cost_breakdown.items():\n            resource_type = attachment_type.replace('_attachments', '').replace('_', '-')\n            if resource_type in attachment_costs and count > 0:\n                monthly_cost = count * attachment_costs[resource_type] * hours_per_month\n                monthly_costs[attachment_type] = {\n                    'count': count,\n                    'monthly_cost': monthly_cost\n                }\n        \n        return monthly_costs\n    \n    def data_processing_costs(self):\n        \"\"\"Estimate data processing costs\"\"\"\n        print(\"Data Processing Cost Estimates:\")\n        print(\"===============================\")\n        print(\"Transit Gateway Data Processing: $0.02 per GB\")\n        print(\"\")\n        print(\"Cost optimization recommendations:\")\n        print(\"- Monitor data transfer patterns\")\n        print(\"- Use VPC endpoints to reduce TGW traffic\")\n        print(\"- Optimize application architectures\")\n        print(\"- Consider Regional data locality\")\n    \n    def generate_cost_report(self):\n        \"\"\"Generate comprehensive cost report\"\"\"\n        print(f\"Transit Gateway Cost Analysis: {self.tgw_id}\")\n        print(\"=\" * 50)\n        \n        # Attachment costs\n        attachment_costs = self.analyze_attachment_costs()\n        total_monthly = 0\n        \n        print(\"Monthly Attachment Costs:\")\n        for attachment_type, details in attachment_costs.items():\n            print(f\"  {attachment_type}: {details['count']} x ${details['monthly_cost']:.2f}\")\n            total_monthly += details['monthly_cost']\n        \n        print(f\"\\nTotal Monthly Attachment Cost: ${total_monthly:.2f}\")\n        print(f\"Total Annual Attachment Cost: ${total_monthly * 12:.2f}\")\n        \n        print(\"\\n\")\n        self.data_processing_costs()\n        \n        # Cost optimization recommendations\n        print(\"\\nCost Optimization Recommendations:\")\n        print(\"=\" * 35)\n        print(\"1. Use custom route tables to reduce unnecessary traffic\")\n        print(\"2. Implement VPC endpoints for AWS services\")\n        print(\"3. Monitor and optimize inter-VPC communication\")\n        print(\"4. Consider VPC peering for simple point-to-point connections\")\n        print(\"5. Use CloudWatch metrics to identify unused attachments\")\n\n# Usage\nif __name__ == \"__main__\":\n    import sys\n    if len(sys.argv) != 2:\n        print(\"Usage: python3 tgw_cost_analyzer.py <transit-gateway-id>\")\n        sys.exit(1)\n    \n    tgw_id = sys.argv[1]\n    analyzer = TGWCostAnalyzer(tgw_id)\n    analyzer.generate_cost_report()\n```\n\n## Cleanup Script\n\n```bash\n#!/bin/bash\n# Clean up Transit Gateway IPv6 demo resources\n\ncleanup_tgw_demo() {\n    echo \"Cleaning up Transit Gateway IPv6 demo resources...\"\n    echo \"=================================================\"\n    \n    # Terminate test instances\n    echo \"1. Terminating test instances...\"\n    TEST_INSTANCES=$(aws ec2 describe-instances \\\n        --filters \"Name=tag:Name,Values=Production-Test,Development-Test\" \"Name=instance-state-name,Values=running\" \\\n        --query 'Reservations[*].Instances[*].InstanceId' \\\n        --output text)\n    \n    if [ ! -z \"$TEST_INSTANCES\" ]; then\n        aws ec2 terminate-instances --instance-ids $TEST_INSTANCES\n        aws ec2 wait instance-terminated --instance-ids $TEST_INSTANCES\n        echo \"Test instances terminated\"\n    fi\n    \n    # Detach VPCs from Transit Gateway\n    echo \"2. Detaching VPCs from Transit Gateway...\"\n    ATTACHMENTS=$(aws ec2 describe-transit-gateway-attachments \\\n        --filters \"Name=transit-gateway-id,Values=$TGW_ID\" \"Name=resource-type,Values=vpc\" \\\n        --query 'TransitGatewayAttachments[?State==`available`].TransitGatewayAttachmentId' \\\n        --output text)\n    \n    for ATTACHMENT_ID in $ATTACHMENTS; do\n        echo \"Deleting attachment: $ATTACHMENT_ID\"\n        aws ec2 delete-transit-gateway-vpc-attachment --transit-gateway-attachment-id $ATTACHMENT_ID\n    done\n    \n    # Wait for attachments to be deleted\n    if [ ! -z \"$ATTACHMENTS\" ]; then\n        for ATTACHMENT_ID in $ATTACHMENTS; do\n            aws ec2 wait transit-gateway-attachment-deleted --transit-gateway-attachment-ids $ATTACHMENT_ID\n        done\n    fi\n    \n    # Delete custom route tables\n    echo \"3. Deleting custom route tables...\"\n    CUSTOM_RTS=$(aws ec2 describe-transit-gateway-route-tables \\\n        --filters \"Name=transit-gateway-id,Values=$TGW_ID\" \"Name=default-route-table,Values=false\" \\\n        --query 'TransitGatewayRouteTables[?State==`available`].TransitGatewayRouteTableId' \\\n        --output text)\n    \n    for RT_ID in $CUSTOM_RTS; do\n        echo \"Deleting route table: $RT_ID\"\n        aws ec2 delete-transit-gateway-route-table --transit-gateway-route-table-id $RT_ID\n    done\n    \n    # Delete Transit Gateway\n    echo \"4. Deleting Transit Gateway...\"\n    aws ec2 delete-transit-gateway --transit-gateway-id $TGW_ID\n    aws ec2 wait transit-gateway-deleted --transit-gateway-ids $TGW_ID\n    echo \"Transit Gateway deleted\"\n    \n    # Delete VPCs\n    echo \"5. Deleting VPCs...\"\n    for VPC_ID in $PROD_VPC_ID $DEV_VPC_ID $SHARED_VPC_ID; do\n        if [ ! -z \"$VPC_ID\" ] && [ \"$VPC_ID\" != \"None\" ]; then\n            echo \"Deleting VPC: $VPC_ID\"\n            \n            # Delete subnets\n            SUBNETS=$(aws ec2 describe-subnets --filters \"Name=vpc-id,Values=$VPC_ID\" --query 'Subnets[*].SubnetId' --output text)\n            for SUBNET_ID in $SUBNETS; do\n                aws ec2 delete-subnet --subnet-id $SUBNET_ID\n            done\n            \n            # Delete Internet Gateway\n            IGW_ID=$(aws ec2 describe-internet-gateways --filters \"Name=attachment.vpc-id,Values=$VPC_ID\" --query 'InternetGateways[0].InternetGatewayId' --output text)\n            if [ \"$IGW_ID\" != \"None\" ] && [ ! -z \"$IGW_ID\" ]; then\n                aws ec2 detach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID\n                aws ec2 delete-internet-gateway --internet-gateway-id $IGW_ID\n            fi\n            \n            # Delete VPC\n            aws ec2 delete-vpc --vpc-id $VPC_ID\n        fi\n    done\n    \n    echo \"Cleanup complete!\"\n}\n\n# Prompt for confirmation\nread -p \"Are you sure you want to delete all demo resources? (yes/no): \" confirm\nif [ \"$confirm\" = \"yes\" ]; then\n    cleanup_tgw_demo\nelse\n    echo \"Cleanup cancelled\"\nfi\n```\n\n## Conclusion\n\nThis comprehensive demonstration shows how to deploy and manage AWS Transit Gateway with IPv6 support. Key takeaways:\n\n1. **Simplified Connectivity**: Transit Gateway provides centralized routing for complex IPv6 networks\n2. **Scalable Architecture**: Support for thousands of routes and multiple VPC attachments\n3. **Advanced Routing**: Custom route tables enable network segmentation and security policies\n4. **Monitoring**: CloudWatch metrics and VPC Flow Logs provide visibility into IPv6 traffic\n5. **Cost Management**: Understanding attachment and data processing costs for optimization\n6. **Security**: Proper configuration of route tables and monitoring for security compliance\n\nTransit Gateway with IPv6 enables organizations to build large-scale, future-proof network architectures that can grow with their needs while maintaining security and performance standards. The patterns demonstrated here provide a foundation for production deployments in complex enterprise environments.