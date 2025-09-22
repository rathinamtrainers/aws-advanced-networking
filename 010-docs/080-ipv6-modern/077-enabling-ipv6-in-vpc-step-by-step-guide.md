# Topic 77: Enabling IPv6 in VPC: Step-by-Step Guide

## Overview

This comprehensive guide walks through the process of enabling IPv6 in an existing VPC and launching IPv6-enabled resources. We'll cover both AWS Console and CLI approaches, along with common troubleshooting scenarios and best practices.

## Prerequisites

### Before You Begin
- Existing VPC with IPv4 CIDR block
- Appropriate IAM permissions for VPC and EC2 operations
- Understanding of basic VPC concepts (subnets, route tables, security groups)
- AWS CLI configured (for CLI examples)

### Required IAM Permissions
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ec2:AssociateVpcCidrBlock",
                "ec2:DisassociateVpcCidrBlock",
                "ec2:DescribeVpcs",
                "ec2:DescribeSubnets",
                "ec2:AssociateSubnetCidrBlock",
                "ec2:DisassociateSubnetCidrBlock",
                "ec2:ModifySubnetAttribute",
                "ec2:CreateRoute",
                "ec2:DeleteRoute",
                "ec2:DescribeRouteTables",
                "ec2:ModifyVpcAttribute"
            ],
            "Resource": "*"
        }
    ]
}
```

## Step 1: Associate IPv6 CIDR Block with VPC

### Using AWS Console

1. **Navigate to VPC Console**
   - Open AWS Console → VPC → Your VPCs
   - Select the target VPC

2. **Associate IPv6 CIDR Block**
   ```
   Actions → Edit CIDRs → Add IPv6 CIDR
   ├── IPv6 CIDR block: Amazon-provided IPv6 CIDR block
   ├── IPv6 pool: Amazon pool
   └── Click "Select CIDR"
   ```

3. **Verify Association**
   - IPv6 CIDR will appear in VPC details
   - Typical format: `2001:db8:1234::/56`

### Using AWS CLI

```bash
# Get VPC ID
VPC_ID="vpc-12345678"

# Associate IPv6 CIDR block
aws ec2 associate-vpc-cidr-block \
    --vpc-id $VPC_ID \
    --amazon-provided-ipv6-cidr-block

# Verify the association
aws ec2 describe-vpcs \
    --vpc-ids $VPC_ID \
    --query 'Vpcs[0].Ipv6CidrBlockAssociationSet'
```

**Expected Output:**
```json
[
    {
        "AssociationId": "vpc-cidr-assoc-12345678",
        "Ipv6CidrBlock": "2001:db8:1234::/56",
        "Ipv6CidrBlockState": {
            "State": "associated"
        }
    }
]
```

## Step 2: Configure Subnets for IPv6

### Assign IPv6 CIDR to Subnets

#### Console Method
```
1. VPC Console → Subnets
2. Select target subnet
3. Actions → Edit IPv6 CIDRs
4. Add IPv6 CIDR: /64 from VPC range
   Example: 2001:db8:1234:1::/64
5. Save changes
```

#### CLI Method
```bash
# Get subnet details
SUBNET_ID="subnet-12345678"
VPC_IPV6_CIDR="2001:db8:1234::/56"

# Associate IPv6 CIDR with subnet
aws ec2 associate-subnet-cidr-block \
    --subnet-id $SUBNET_ID \
    --ipv6-cidr-block "2001:db8:1234:1::/64"

# Verify subnet IPv6 configuration
aws ec2 describe-subnets \
    --subnet-ids $SUBNET_ID \
    --query 'Subnets[0].Ipv6CidrBlockAssociationSet'
```

### Enable Auto-Assign IPv6 Addresses

#### Console Method
```
1. Select subnet
2. Actions → Modify auto-assign IP settings
3. Check "Enable auto-assign IPv6 address"
4. Save
```

#### CLI Method
```bash
# Enable auto-assign IPv6 for new instances
aws ec2 modify-subnet-attribute \
    --subnet-id $SUBNET_ID \
    --assign-ipv6-address-on-creation
```

## Step 3: Update Route Tables

### Add IPv6 Routes

#### For Public Subnets
```bash
# Get route table ID
ROUTE_TABLE_ID=$(aws ec2 describe-route-tables \
    --filters "Name=association.subnet-id,Values=$SUBNET_ID" \
    --query 'RouteTables[0].RouteTableId' \
    --output text)

# Add IPv6 default route via Internet Gateway
IGW_ID=$(aws ec2 describe-internet-gateways \
    --filters "Name=attachment.vpc-id,Values=$VPC_ID" \
    --query 'InternetGateways[0].InternetGatewayId' \
    --output text)

aws ec2 create-route \
    --route-table-id $ROUTE_TABLE_ID \
    --destination-ipv6-cidr-block ::/0 \
    --gateway-id $IGW_ID
```

#### For Private Subnets (Egress-Only Internet Gateway)
```bash
# Create Egress-Only Internet Gateway
EIGW_ID=$(aws ec2 create-egress-only-internet-gateway \
    --vpc-id $VPC_ID \
    --query 'EgressOnlyInternetGateway.EgressOnlyInternetGatewayId' \
    --output text)

# Add IPv6 egress route
aws ec2 create-route \
    --route-table-id $PRIVATE_ROUTE_TABLE_ID \
    --destination-ipv6-cidr-block ::/0 \
    --egress-only-internet-gateway-id $EIGW_ID
```

### Verify Route Configuration
```bash
# Check route table entries
aws ec2 describe-route-tables \
    --route-table-ids $ROUTE_TABLE_ID \
    --query 'RouteTables[0].Routes[?DestinationIpv6CidrBlock]'
```

## Step 4: Update Security Groups

### Create IPv6-Aware Security Group Rules

#### Basic Web Server Example
```bash
# Get security group ID
SG_ID="sg-12345678"

# Allow HTTP from anywhere (IPv6)
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 80 \
    --source-ipv6 ::/0

# Allow HTTPS from anywhere (IPv6)
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 443 \
    --source-ipv6 ::/0

# Allow ICMPv6 (essential for IPv6 operation)
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol icmpv6 \
    --source-ipv6 ::/0
```

#### SSH Access with IPv6
```bash
# Allow SSH from specific IPv6 range
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 22 \
    --source-ipv6 2001:db8:admin::/48
```

### Security Group JSON Configuration
```json
{
    "GroupId": "sg-12345678",
    "IpPermissions": [
        {
            "IpProtocol": "tcp",
            "FromPort": 443,
            "ToPort": 443,
            "Ipv6Ranges": [
                {
                    "CidrIpv6": "::/0",
                    "Description": "HTTPS from anywhere"
                }
            ]
        },
        {
            "IpProtocol": "icmpv6",
            "FromPort": -1,
            "ToPort": -1,
            "Ipv6Ranges": [
                {
                    "CidrIpv6": "::/0",
                    "Description": "ICMPv6 for neighbor discovery"
                }
            ]
        }
    ]
}
```

## Step 5: Launch IPv6-Enabled EC2 Instance

### Using AWS Console

1. **Launch Instance Wizard**
   ```
   EC2 Console → Launch Instance
   ├── Choose AMI (ensure IPv6 support)
   ├── Select instance type
   ├── Configure Instance Details:
   │   ├── Network: Select IPv6-enabled VPC
   │   ├── Subnet: Select IPv6-enabled subnet
   │   └── Auto-assign IPv6 IP: Enable
   └── Configure Security Group: Use IPv6-aware SG
   ```

### Using AWS CLI

```bash
# Launch instance with IPv6
aws ec2 run-instances \
    --image-id ami-12345678 \
    --count 1 \
    --instance-type t3.micro \
    --subnet-id $SUBNET_ID \
    --security-group-ids $SG_ID \
    --ipv6-address-count 1 \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=IPv6-Test}]'
```

### Verify Instance IPv6 Configuration

```bash
# Get instance details
INSTANCE_ID="i-12345678"

aws ec2 describe-instances \
    --instance-ids $INSTANCE_ID \
    --query 'Reservations[0].Instances[0].NetworkInterfaces[0].Ipv6Addresses'
```

## Step 6: Test IPv6 Connectivity

### From Instance (SSH/Session Manager)

```bash
# Check IPv6 configuration
ip -6 addr show

# Test IPv6 connectivity
ping6 ipv6.google.com

# Test DNS resolution
nslookup -type=AAAA google.com

# Check routing table
ip -6 route show
```

### From External Source

```bash
# Test connectivity to instance
ping6 2001:db8:1234:1::a3f

# Test web service (if running)
curl -6 http://[2001:db8:1234:1::a3f]/
```

## Step 7: Configure DNS (Route 53)

### Create AAAA Records

```bash
# Create hosted zone record
aws route53 change-resource-record-sets \
    --hosted-zone-id Z1234567890ABC \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "web.example.com",
                "Type": "AAAA",
                "TTL": 300,
                "ResourceRecords": [{
                    "Value": "2001:db8:1234:1::a3f"
                }]
            }
        }]
    }'
```

## Troubleshooting Common Issues

### Issue 1: IPv6 CIDR Association Fails

**Symptom:** "The specified VPC already has the maximum number of CIDR blocks"
```bash
# Check current CIDR associations
aws ec2 describe-vpcs \
    --vpc-ids $VPC_ID \
    --query 'Vpcs[0].[CidrBlockAssociationSet,Ipv6CidrBlockAssociationSet]'
```

**Solution:** Remove unused CIDR blocks or use a different VPC

### Issue 2: Instance Not Getting IPv6 Address

**Diagnostic Steps:**
```bash
# 1. Verify subnet has IPv6 CIDR
aws ec2 describe-subnets --subnet-ids $SUBNET_ID

# 2. Check auto-assign setting
aws ec2 describe-subnets \
    --subnet-ids $SUBNET_ID \
    --query 'Subnets[0].AssignIpv6AddressOnCreation'

# 3. Verify security groups allow ICMPv6
aws ec2 describe-security-groups --group-ids $SG_ID
```

### Issue 3: No IPv6 Internet Connectivity

**Check Route Table:**
```bash
# Verify IPv6 default route exists
aws ec2 describe-route-tables \
    --route-table-ids $ROUTE_TABLE_ID \
    --query 'RouteTables[0].Routes[?DestinationIpv6CidrBlock==`::/0`]'
```

**Check Internet Gateway:**
```bash
# Verify IGW is attached
aws ec2 describe-internet-gateways \
    --filters "Name=attachment.vpc-id,Values=$VPC_ID"
```

### Issue 4: Security Group Rules Not Working

**Common ICMPv6 Requirements:**
```json
{
    "IpProtocol": "icmpv6",
    "FromPort": -1,
    "ToPort": -1,
    "Ipv6Ranges": [{"CidrIpv6": "::/0"}]
}
```

## Automation Script

### Complete VPC IPv6 Enablement Script

```bash
#!/bin/bash

VPC_ID="vpc-12345678"
SUBNET_ID="subnet-12345678"
SG_ID="sg-12345678"

echo "Enabling IPv6 for VPC: $VPC_ID"

# 1. Associate IPv6 CIDR with VPC
echo "Associating IPv6 CIDR with VPC..."
aws ec2 associate-vpc-cidr-block \
    --vpc-id $VPC_ID \
    --amazon-provided-ipv6-cidr-block

# Wait for association
sleep 30

# 2. Get VPC IPv6 CIDR
VPC_IPV6_CIDR=$(aws ec2 describe-vpcs \
    --vpc-ids $VPC_ID \
    --query 'Vpcs[0].Ipv6CidrBlockAssociationSet[0].Ipv6CidrBlock' \
    --output text)

echo "VPC IPv6 CIDR: $VPC_IPV6_CIDR"

# 3. Associate subnet IPv6 CIDR
SUBNET_IPV6_CIDR="${VPC_IPV6_CIDR%::/56}:1::/64"
echo "Associating subnet IPv6 CIDR: $SUBNET_IPV6_CIDR"

aws ec2 associate-subnet-cidr-block \
    --subnet-id $SUBNET_ID \
    --ipv6-cidr-block $SUBNET_IPV6_CIDR

# 4. Enable auto-assign IPv6
aws ec2 modify-subnet-attribute \
    --subnet-id $SUBNET_ID \
    --assign-ipv6-address-on-creation

# 5. Add IPv6 route
ROUTE_TABLE_ID=$(aws ec2 describe-route-tables \
    --filters "Name=association.subnet-id,Values=$SUBNET_ID" \
    --query 'RouteTables[0].RouteTableId' \
    --output text)

IGW_ID=$(aws ec2 describe-internet-gateways \
    --filters "Name=attachment.vpc-id,Values=$VPC_ID" \
    --query 'InternetGateways[0].InternetGatewayId' \
    --output text)

aws ec2 create-route \
    --route-table-id $ROUTE_TABLE_ID \
    --destination-ipv6-cidr-block ::/0 \
    --gateway-id $IGW_ID

# 6. Update security group
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol icmpv6 \
    --source-ipv6 ::/0

echo "IPv6 enablement complete!"
```

## Validation Checklist

- [ ] VPC has associated IPv6 CIDR block
- [ ] Subnets have IPv6 CIDR assignments
- [ ] Auto-assign IPv6 is enabled on target subnets
- [ ] Route tables have appropriate IPv6 routes
- [ ] Security groups allow required IPv6 traffic
- [ ] ICMPv6 is permitted for proper IPv6 operation
- [ ] Instances receive IPv6 addresses automatically
- [ ] IPv6 connectivity works from instances
- [ ] DNS AAAA records are configured (if needed)

## Best Practices

1. **Plan Your IPv6 Addressing**
   - Use consistent /64 subnet allocation
   - Document your IPv6 addressing scheme
   - Reserve ranges for future expansion

2. **Security Configuration**
   - Always allow ICMPv6 for proper operation
   - Use specific IPv6 ranges instead of ::/0 when possible
   - Regularly audit security group rules

3. **Monitoring and Logging**
   - Update CloudWatch dashboards for IPv6 metrics
   - Configure VPC Flow Logs to capture IPv6 traffic
   - Set up alerts for IPv6 connectivity issues

4. **Testing Strategy**
   - Test dual-stack applications thoroughly
   - Verify fallback behavior for IPv4-only clients
   - Monitor application performance with IPv6

## Next Steps

With IPv6 successfully enabled in your VPC, you can now:
- Explore IPv6-only architectures (Topic 78)
- Implement IPv6 security best practices (Topic 79)
- Deploy dual-stack applications (Topic 80)
- Configure advanced IPv6 routing scenarios (Topics 81-85)