# Topic 7: Elastic IP and Elastic Network Interface (ENI) Explained

## Prerequisites
- Completed Topics 1-6 (AWS Networking Foundations)
- Understanding of EC2 instances and networking
- Knowledge of IP addressing concepts

## Learning Objectives
By the end of this topic, you will be able to:
- Configure and manage Elastic IP addresses effectively
- Create and attach Elastic Network Interfaces (ENIs)
- Implement multi-homed instances and failover scenarios
- Design high availability architectures using EIPs and ENIs

## Theory

### Elastic IP Address (EIP)

#### Definition and Purpose
Elastic IP address is a static IPv4 address designed for dynamic cloud computing. It's associated with your AWS account and can be attached to any instance or network interface.

#### Key Characteristics
- **Static Public IP**: Doesn't change when instance stops/starts
- **Remappable**: Can be moved between instances instantly
- **Regional Resource**: Tied to specific AWS region
- **Billing**: Free when attached to running instance, charged when unattached
- **DNS**: Supports both forward and reverse DNS

#### EIP vs Instance Public IP
| Feature | Instance Public IP | Elastic IP |
|---------|-------------------|------------|
| **Persistence** | Changes on stop/start | Static until released |
| **Cost** | Free | Free when attached, charged when idle |
| **Control** | Automatic assignment | Manual management |
| **Portability** | Tied to instance | Can move between resources |
| **DNS** | Dynamic | Configurable |

### Elastic Network Interface (ENI)

#### Definition and Purpose
ENI is a virtual network interface that can be attached to EC2 instances. It represents a virtual network card with its own MAC address, IP addresses, and security groups.

#### Key Characteristics
- **Independent Resource**: Exists separately from EC2 instances
- **Portable**: Can be moved between instances in same AZ
- **Multiple IPs**: Can have multiple private and public IP addresses
- **Security Groups**: Has its own security group associations
- **Persistent**: Maintains configuration when moved between instances

#### ENI Components
- **Primary Private IP**: Assigned when ENI created
- **Secondary Private IPs**: Additional private IPs (up to 15 per ENI)
- **Elastic IPs**: Can be associated with private IPs
- **MAC Address**: Unique identifier for the interface
- **Security Groups**: Network access control
- **Source/Destination Check**: Traffic validation setting

### Use Cases and Scenarios

#### Elastic IP Use Cases
1. **Web Services**: Static IP for DNS A records
2. **Email Servers**: Maintain IP reputation
3. **API Endpoints**: Consistent external access point
4. **Disaster Recovery**: Quick failover to backup instance
5. **Development**: Consistent access during testing

#### ENI Use Cases
1. **Management Networks**: Separate administrative access
2. **Dual-Homed Instances**: Multi-subnet connectivity
3. **Network Appliances**: Firewalls, load balancers
4. **High Availability**: Quick network interface failover
5. **License Management**: MAC address-based licensing

## Lab Exercise: Advanced EIP and ENI Configurations

### Lab Duration: 150 minutes

### Step 1: Set Up Lab Environment
**Objective**: Create VPC infrastructure for EIP and ENI testing

```bash
# Create VPC for lab
VPC_ID=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=EIP-ENI-Lab-VPC}]' \
    --query 'Vpc.VpcId' --output text)

aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames

# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
    --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=EIP-ENI-IGW}]' \
    --query 'InternetGateway.InternetGatewayId' --output text)

aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID

# Create subnets in different AZs
PUBLIC_SUBNET_1A=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.1.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Public-1a}]' \
    --query 'Subnet.SubnetId' --output text)

PUBLIC_SUBNET_1B=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.2.0/24 --availability-zone us-east-1b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Public-1b}]' \
    --query 'Subnet.SubnetId' --output text)

PRIVATE_SUBNET_1A=$(aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.11.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Private-1a}]' \
    --query 'Subnet.SubnetId' --output text)

# Create route table for public subnets
PUBLIC_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Public-RT}]' \
    --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $PUBLIC_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $PUBLIC_RT --subnet-id $PUBLIC_SUBNET_1A
aws ec2 associate-route-table --route-table-id $PUBLIC_RT --subnet-id $PUBLIC_SUBNET_1B
```

### Step 2: Elastic IP Address Management
**Objective**: Demonstrate EIP allocation, association, and management

**Allocate Elastic IPs**:
```bash
# Allocate multiple Elastic IPs
EIP_1=$(aws ec2 allocate-address --domain vpc \
    --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=WebServer-EIP}]' \
    --query 'AllocationId' --output text)

EIP_2=$(aws ec2 allocate-address --domain vpc \
    --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=API-Server-EIP}]' \
    --query 'AllocationId' --output text)

EIP_3=$(aws ec2 allocate-address --domain vpc \
    --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=Standby-EIP}]' \
    --query 'AllocationId' --output text)

# View allocated EIPs
aws ec2 describe-addresses --allocation-ids $EIP_1 $EIP_2 $EIP_3 \
    --query 'Addresses[*].[AllocationId,PublicIp,Tags[?Key==`Name`].Value|[0]]' \
    --output table

echo "EIP_1 (WebServer): $EIP_1"
echo "EIP_2 (API-Server): $EIP_2" 
echo "EIP_3 (Standby): $EIP_3"
```

**Launch Test Instances**:
```bash
# Create security group
SECURITY_GROUP=$(aws ec2 create-security-group \
    --group-name EIP-ENI-SG \
    --description "Security group for EIP and ENI testing" \
    --vpc-id $VPC_ID \
    --query 'GroupId' --output text)

# Allow SSH and web traffic
aws ec2 authorize-security-group-ingress --group-id $SECURITY_GROUP --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SECURITY_GROUP --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $SECURITY_GROUP --protocol tcp --port 443 --cidr 0.0.0.0/0

# Launch primary web server instance
INSTANCE_1=$(aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --count 1 \
    --instance-type t3.micro \
    --key-name your-key-pair \
    --security-group-ids $SECURITY_GROUP \
    --subnet-id $PUBLIC_SUBNET_1A \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=WebServer-Primary}]' \
    --query 'Instances[0].InstanceId' --output text)

# Launch backup web server instance
INSTANCE_2=$(aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --count 1 \
    --instance-type t3.micro \
    --key-name your-key-pair \
    --security-group-ids $SECURITY_GROUP \
    --subnet-id $PUBLIC_SUBNET_1B \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=WebServer-Backup}]' \
    --query 'Instances[0].InstanceId' --output text)

# Wait for instances to be running
aws ec2 wait instance-running --instance-ids $INSTANCE_1 $INSTANCE_2
```

**Associate EIP with Instance**:
```bash
# Associate EIP with primary instance
aws ec2 associate-address --instance-id $INSTANCE_1 --allocation-id $EIP_1

# Verify association
aws ec2 describe-addresses --allocation-ids $EIP_1 \
    --query 'Addresses[0].[PublicIp,InstanceId,AssociationId]' \
    --output table

# Test connectivity
EIP_1_IP=$(aws ec2 describe-addresses --allocation-ids $EIP_1 --query 'Addresses[0].PublicIp' --output text)
echo "Test SSH to EIP: ssh -i your-key.pem ec2-user@$EIP_1_IP"
```

### Step 3: Elastic Network Interface (ENI) Configuration
**Objective**: Create and configure ENIs for advanced networking scenarios

**Create Multiple ENIs**:
```bash
# Create management ENI (for administrative access)
MGMT_ENI=$(aws ec2 create-network-interface \
    --subnet-id $PUBLIC_SUBNET_1A \
    --description "Management interface for dual-homed instance" \
    --groups $SECURITY_GROUP \
    --tag-specifications 'ResourceType=network-interface,Tags=[{Key=Name,Value=Management-ENI}]' \
    --query 'NetworkInterface.NetworkInterfaceId' --output text)

# Create data ENI (for application traffic)
DATA_ENI=$(aws ec2 create-network-interface \
    --subnet-id $PRIVATE_SUBNET_1A \
    --description "Data interface for application traffic" \
    --groups $SECURITY_GROUP \
    --tag-specifications 'ResourceType=network-interface,Tags=[{Key=Name,Value=Data-ENI}]' \
    --query 'NetworkInterface.NetworkInterfaceId' --output text)

# Create high-availability ENI (for failover scenarios)
HA_ENI=$(aws ec2 create-network-interface \
    --subnet-id $PUBLIC_SUBNET_1A \
    --description "High availability interface for failover" \
    --groups $SECURITY_GROUP \
    --tag-specifications 'ResourceType=network-interface,Tags=[{Key=Name,Value=HA-ENI}]' \
    --query 'NetworkInterface.NetworkInterfaceId' --output text)

echo "Management ENI: $MGMT_ENI"
echo "Data ENI: $DATA_ENI"
echo "HA ENI: $HA_ENI"
```

**Configure Secondary Private IPs**:
```bash
# Add secondary private IP to management ENI
aws ec2 assign-private-ip-addresses \
    --network-interface-id $MGMT_ENI \
    --secondary-private-ip-address-count 2

# Get the assigned IPs
aws ec2 describe-network-interfaces --network-interface-ids $MGMT_ENI \
    --query 'NetworkInterfaces[0].PrivateIpAddresses[*].[PrivateIpAddress,Primary]' \
    --output table

# Associate EIP with secondary private IP
SECONDARY_IP=$(aws ec2 describe-network-interfaces --network-interface-ids $MGMT_ENI \
    --query 'NetworkInterfaces[0].PrivateIpAddresses[?Primary==`false`]|[0].PrivateIpAddress' \
    --output text)

aws ec2 associate-address \
    --allocation-id $EIP_2 \
    --network-interface-id $MGMT_ENI \
    --private-ip-address $SECONDARY_IP
```

### Step 4: Multi-Homed Instance Configuration
**Objective**: Create instance with multiple network interfaces

**Launch Multi-Homed Instance**:
```bash
# Launch instance with multiple ENIs
MULTI_INSTANCE=$(aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --count 1 \
    --instance-type t3.small \
    --key-name your-key-pair \
    --network-interfaces \
        "DeviceIndex=0,NetworkInterfaceId=$MGMT_ENI" \
        "DeviceIndex=1,NetworkInterfaceId=$DATA_ENI" \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Multi-Homed-Instance}]' \
    --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $MULTI_INSTANCE

# Verify network interfaces are attached
aws ec2 describe-instances --instance-ids $MULTI_INSTANCE \
    --query 'Reservations[0].Instances[0].NetworkInterfaces[*].[NetworkInterfaceId,SubnetId,PrivateIpAddress]' \
    --output table
```

**Configure Network Interface in Instance**:
```bash
# SSH to instance and configure second interface
MGMT_IP=$(aws ec2 describe-network-interfaces --network-interface-ids $MGMT_ENI \
    --query 'NetworkInterfaces[0].Association.PublicIp' --output text)

echo "SSH to multi-homed instance: ssh -i your-key.pem ec2-user@$MGMT_IP"

# Commands to run inside instance:
cat << 'EOF' > configure-eni.sh
#!/bin/bash
# Configure second network interface (eth1)

# Enable second interface
sudo ip link set dev eth1 up

# Get IP configuration from AWS metadata
ETH1_IP=$(curl -s http://169.254.169.254/latest/meta-data/network/interfaces/macs/$(cat /sys/class/net/eth1/address)/local-ipv4s)
ETH1_SUBNET=$(curl -s http://169.254.169.254/latest/meta-data/network/interfaces/macs/$(cat /sys/class/net/eth1/address)/subnet-ipv4-cidr-block)

# Configure IP address
sudo ip addr add $ETH1_IP/24 dev eth1

# Add route for second subnet
GATEWAY=$(echo $ETH1_SUBNET | cut -d'/' -f1 | awk -F. '{print $1"."$2"."$3".1"}')
sudo ip route add $ETH1_SUBNET dev eth1 src $ETH1_IP

echo "Configured eth1 with IP: $ETH1_IP"
echo "Added route for subnet: $ETH1_SUBNET"

# Verify configuration
ip addr show eth1
ip route | grep eth1
EOF

chmod +x configure-eni.sh
```

### Step 5: High Availability and Failover Scenarios
**Objective**: Implement ENI-based failover mechanisms

**ENI Failover Configuration**:
```bash
# Create script for ENI failover
cat << 'EOF' > eni-failover.sh
#!/bin/bash
# ENI Failover Script

SOURCE_INSTANCE=$1
TARGET_INSTANCE=$2
ENI_ID=$3

if [ $# -ne 3 ]; then
    echo "Usage: $0 <source-instance-id> <target-instance-id> <eni-id>"
    exit 1
fi

echo "Starting ENI failover..."
echo "Moving ENI $ENI_ID from $SOURCE_INSTANCE to $TARGET_INSTANCE"

# Detach ENI from source instance
echo "Detaching ENI from source instance..."
ATTACHMENT_ID=$(aws ec2 describe-network-interfaces --network-interface-ids $ENI_ID \
    --query 'NetworkInterfaces[0].Attachment.AttachmentId' --output text)

if [ "$ATTACHMENT_ID" != "None" ] && [ "$ATTACHMENT_ID" != "" ]; then
    aws ec2 detach-network-interface --attachment-id $ATTACHMENT_ID --force
    echo "Waiting for detachment..."
    aws ec2 wait network-interface-available --network-interface-ids $ENI_ID
fi

# Attach ENI to target instance
echo "Attaching ENI to target instance..."
aws ec2 attach-network-interface \
    --network-interface-id $ENI_ID \
    --instance-id $TARGET_INSTANCE \
    --device-index 1

echo "Waiting for attachment..."
sleep 10

# Verify attachment
aws ec2 describe-network-interfaces --network-interface-ids $ENI_ID \
    --query 'NetworkInterfaces[0].[NetworkInterfaceId,Attachment.InstanceId,Status]' \
    --output table

echo "ENI failover completed!"
EOF

chmod +x eni-failover.sh
```

**Test ENI Failover**:
```bash
# Create standby instance for failover testing
STANDBY_INSTANCE=$(aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --count 1 \
    --instance-type t3.micro \
    --key-name your-key-pair \
    --security-group-ids $SECURITY_GROUP \
    --subnet-id $PUBLIC_SUBNET_1A \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Standby-Instance}]' \
    --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids $STANDBY_INSTANCE

# Test failover by moving HA ENI
echo "Testing ENI failover..."
./eni-failover.sh $MULTI_INSTANCE $STANDBY_INSTANCE $HA_ENI

# Verify the move
aws ec2 describe-network-interfaces --network-interface-ids $HA_ENI \
    --query 'NetworkInterfaces[0].Attachment.[InstanceId,DeviceIndex,Status]' \
    --output table
```

### Step 6: EIP and ENI Monitoring and Management
**Objective**: Set up monitoring and alerting for network resources

**CloudWatch Monitoring**:
```bash
# Create CloudWatch alarms for EIP usage
aws cloudwatch put-metric-alarm \
    --alarm-name "Unused-EIP-Alarm" \
    --alarm-description "Alert when EIP is not associated with instance" \
    --metric-name "ElasticIPUnassociated" \
    --namespace "Custom/Billing" \
    --statistic Sum \
    --period 3600 \
    --threshold 1 \
    --comparison-operator GreaterThanOrEqualToThreshold \
    --evaluation-periods 1

# Create custom metric for ENI attachment status
aws cloudwatch put-metric-data \
    --namespace "Custom/Network" \
    --metric-data \
        MetricName=ENIAttached,Value=1,Unit=Count,Dimensions=Name=ENI,Value=$HA_ENI
```

**Cost Management**:
```bash
# Check EIP costs (unattached EIPs cost $0.005/hour)
aws ec2 describe-addresses \
    --query 'Addresses[?!InstanceId].[AllocationId,PublicIp,Tags[?Key==`Name`].Value|[0]]' \
    --output table

echo "Unattached EIPs incur charges of \$0.005 per hour"

# Calculate potential monthly cost for unattached EIPs
UNATTACHED_COUNT=$(aws ec2 describe-addresses --query 'length(Addresses[?!InstanceId])' --output text)
if [ "$UNATTACHED_COUNT" -gt 0 ]; then
    MONTHLY_COST=$(echo "$UNATTACHED_COUNT * 0.005 * 24 * 30" | bc -l)
    echo "Potential monthly cost for $UNATTACHED_COUNT unattached EIPs: \$$MONTHLY_COST"
fi
```

## Advanced Scenarios

### ENI-Based Network Appliance
```bash
# Create network appliance with multiple interfaces
APPLIANCE_SG=$(aws ec2 create-security-group \
    --group-name Network-Appliance-SG \
    --description "Security group for network appliance" \
    --vpc-id $VPC_ID \
    --query 'GroupId' --output text)

# Allow all traffic for appliance (adjust for production)
aws ec2 authorize-security-group-ingress --group-id $APPLIANCE_SG --protocol -1 --cidr 10.0.0.0/16

# Create appliance ENIs
EXTERNAL_ENI=$(aws ec2 create-network-interface \
    --subnet-id $PUBLIC_SUBNET_1A \
    --groups $APPLIANCE_SG \
    --description "External interface for network appliance" \
    --query 'NetworkInterface.NetworkInterfaceId' --output text)

INTERNAL_ENI=$(aws ec2 create-network-interface \
    --subnet-id $PRIVATE_SUBNET_1A \
    --groups $APPLIANCE_SG \
    --description "Internal interface for network appliance" \
    --source-dest-check false \
    --query 'NetworkInterface.NetworkInterfaceId' --output text)

# Note: source-dest-check disabled for routing/NAT functionality
```

### EIP DNS Configuration
```bash
# Set up reverse DNS for EIP (requires support case for custom PTR record)
EIP_IP=$(aws ec2 describe-addresses --allocation-ids $EIP_1 --query 'Addresses[0].PublicIp' --output text)

echo "To configure reverse DNS for $EIP_IP:"
echo "1. Create AWS Support case requesting PTR record"
echo "2. Provide desired hostname (e.g., webserver.example.com)"
echo "3. AWS will configure reverse DNS lookup"

# Forward DNS configuration (using Route 53)
HOSTED_ZONE_ID="Z1234567890ABC"  # Your hosted zone ID
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "webserver.example.com",
                "Type": "A",
                "TTL": 300,
                "ResourceRecords": [{"Value": "'$EIP_IP'"}]
            }
        }]
    }'
```

## Architecture Diagrams

### Multi-Homed Instance with ENIs
```
Internet
    |
[Internet Gateway]
    |
┌─────────────────────────────────────────────────────────────┐
│ VPC: 10.0.0.0/16                                           │
│                                                             │
│ ┌─────────────────┐    ┌─────────────────┐                 │
│ │ Public Subnet   │    │ Private Subnet  │                 │
│ │ 10.0.1.0/24     │    │ 10.0.11.0/24    │                 │
│ │                 │    │                 │                 │
│ │ [EIP] ←→ [ENI1] │    │      [ENI2]     │                 │
│ │         eth0 ←──┼────┼──→ Multi-Homed ←─┼─→ eth1         │
│ │                 │    │    Instance     │                 │
│ │ Management      │    │ Data Network    │                 │
│ │ Network         │    │                 │                 │
│ └─────────────────┘    └─────────────────┘                 │
└─────────────────────────────────────────────────────────────┘
```

### EIP Failover Scenario
```
Before Failover:
[EIP] ←→ [Primary Instance] (Running)
         [Backup Instance] (Standby)

During Failover:
[EIP] ←→ [Primary Instance] (Failed)
    ↓    [Backup Instance] (Standby)
    └──→ [Backup Instance] (Active)

After Failover:
         [Primary Instance] (Failed)
[EIP] ←→ [Backup Instance] (Active)
```

## Troubleshooting Guide

| Issue | Symptoms | Diagnosis | Solution |
|-------|----------|-----------|----------|
| EIP not accessible | Cannot reach public IP | Check security groups and NACLs | Verify inbound rules allow traffic |
| ENI attachment fails | Error during attachment | Check AZ compatibility | ENI and instance must be in same AZ |
| Multiple IPs not working | Secondary IPs unreachable | OS configuration needed | Configure additional IPs in OS |
| Failover doesn't work | Traffic still goes to failed instance | DNS/routing cache | Wait for DNS TTL or flush cache |
| High EIP costs | Unexpected billing | Unattached EIPs | Release unused EIPs or attach to instances |

## Key Takeaways
- EIPs provide static public IP addresses for dynamic cloud resources
- ENIs enable advanced networking scenarios like multi-homing and failover
- EIPs are free when attached, charged when idle
- ENIs can be moved between instances for high availability
- Multiple private IPs per ENI enable sophisticated network designs
- Source/destination checking must be disabled for routing appliances

## Best Practices
- **EIP Management**: Release unused EIPs to avoid charges
- **ENI Design**: Use ENIs for network appliances and failover scenarios
- **Monitoring**: Set up alerts for unattached EIPs
- **Documentation**: Tag all network resources consistently
- **Security**: Apply appropriate security groups to ENIs
- **High Availability**: Design for ENI-based failover mechanisms

## Additional Resources
- [Elastic IP Addresses](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)
- [Elastic Network Interfaces](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html)
- [Multiple IP Addresses](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/MultipleIP.html)

## Exam Tips
- EIPs are charged when not associated with running instances
- ENIs are AZ-specific and cannot move between AZs
- Maximum 5 EIPs per region by default (can request increase)
- Each ENI can have up to 15 private IP addresses
- Source/destination checking must be disabled for NAT/routing functions
- ENI failover enables rapid recovery for network appliances