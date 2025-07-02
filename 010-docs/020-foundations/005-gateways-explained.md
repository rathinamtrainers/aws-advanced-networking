# Topic 5: Internet Gateway and NAT Gateway - Key Differences

## Prerequisites
- Completed Topics 1-4 (Foundations series)
- Understanding of public vs private subnet concepts
- Basic knowledge of network address translation

## Learning Objectives
By the end of this topic, you will be able to:
- Compare and contrast Internet Gateway and NAT Gateway functionality
- Implement proper gateway selection for different use cases
- Configure high availability NAT Gateway architectures
- Troubleshoot common gateway connectivity issues

## Theory

### Internet Gateway (IGW)

#### Definition and Purpose
Internet Gateway is a horizontally scaled, redundant, and highly available VPC component that allows communication between your VPC and the internet.

#### Key Characteristics
- **Horizontally Scaled**: Automatically scales to handle traffic
- **Highly Available**: No availability risk or bandwidth constraints
- **One per VPC**: Each VPC can have only one IGW attached
- **Bidirectional**: Allows both inbound and outbound internet traffic
- **No NAT Function**: Performs one-to-one NAT for instances with public IPs

#### How IGW Works
1. Instance with public IP sends traffic to internet
2. VPC router forwards traffic to IGW (via route table)
3. IGW performs 1:1 NAT (private IP ↔ public IP)
4. Traffic flows to internet destination
5. Return traffic follows reverse path

### NAT Gateway

#### Definition and Purpose
NAT Gateway enables instances in private subnets to connect to the internet or other AWS services, but prevents the internet from initiating connections to those instances.

#### Key Characteristics
- **Managed Service**: Fully managed by AWS
- **High Availability**: Redundant within single AZ
- **Bandwidth Scaling**: Scales up to 45 Gbps
- **Unidirectional**: Outbound traffic only
- **Port Address Translation**: Many private IPs → one public IP

#### How NAT Gateway Works
1. Private instance sends traffic to internet
2. VPC router forwards to NAT Gateway (via route table)
3. NAT Gateway translates source IP to its Elastic IP
4. Traffic flows to internet destination
5. Return traffic comes back to NAT Gateway
6. NAT Gateway forwards to original private instance

### Comparison Matrix

| Feature | Internet Gateway | NAT Gateway |
|---------|------------------|-------------|
| **Purpose** | Bidirectional internet access | Outbound-only internet access |
| **Target** | Public subnets | Private subnets |
| **IP Address** | 1:1 NAT (public IP required) | Port Address Translation |
| **Availability** | Highly available (regional) | Single AZ (need multiple for HA) |
| **Cost** | Free | Hourly charges + data processing |
| **Bandwidth** | No limits | Up to 45 Gbps |
| **Management** | Fully managed | Fully managed |
| **Security** | Allows inbound connections | Blocks inbound connections |

### NAT Gateway vs NAT Instance

| Feature | NAT Gateway | NAT Instance |
|---------|-------------|--------------|
| **Management** | Fully managed | Self-managed |
| **Performance** | Up to 45 Gbps | Depends on instance type |
| **Availability** | Single AZ redundancy | Manual HA setup required |
| **Security Groups** | Not supported | Supported as source/destination |
| **Port Forwarding** | Not supported | Supported |
| **Bastion Host** | Not supported | Can be configured |
| **Cost** | Higher operational cost | Lower cost, higher management |

## Lab Exercise: Implementing Gateway Strategies

### Lab Duration: 90 minutes

### Step 1: Create Test Environment
**Objective**: Set up infrastructure to test different gateway scenarios

**Architecture Setup**:
```bash
# Use existing VPC or create new one
VPC_ID="vpc-xxxxxxxxx"  # From previous labs

# Create test subnets if not exists
aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.50.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Test-Public-Subnet}]'

aws ec2 create-subnet --vpc-id $VPC_ID --cidr-block 10.0.51.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Test-Private-Subnet}]'
```

### Step 2: Internet Gateway Implementation
**Objective**: Configure and test Internet Gateway functionality

**AWS Console Steps**:
1. Verify Internet Gateway exists and is attached
2. Launch instance in public subnet with auto-assign public IP
3. Create security group allowing SSH and ICMP
4. Test bidirectional connectivity

**CLI Implementation**:
```bash
# Get subnet IDs
PUBLIC_SUBNET=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=Test-Public-Subnet" --query 'Subnets[0].SubnetId' --output text)

# Create security group for public instance
PUBLIC_SG=$(aws ec2 create-security-group --group-name Public-Test-SG --description "Public Instance Security Group" --vpc-id $VPC_ID --query 'GroupId' --output text)

# Allow SSH and ping
aws ec2 authorize-security-group-ingress --group-id $PUBLIC_SG --protocol tcp --port 22 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $PUBLIC_SG --protocol icmp --port -1 --cidr 0.0.0.0/0

# Launch public instance
aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --count 1 \
    --instance-type t3.micro \
    --key-name your-key-pair \
    --security-group-ids $PUBLIC_SG \
    --subnet-id $PUBLIC_SUBNET \
    --associate-public-ip-address \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Public-Test-Instance}]'
```

**Testing IGW Functionality**:
```bash
# From your local machine
ping [public-instance-public-ip]

# SSH to public instance
ssh -i your-key.pem ec2-user@[public-instance-public-ip]

# From public instance, test outbound connectivity
curl -I https://aws.amazon.com
ping 8.8.8.8
```

### Step 3: NAT Gateway Implementation and Testing
**Objective**: Configure NAT Gateway and test private subnet connectivity

**CLI Implementation**:
```bash
# Get private subnet ID
PRIVATE_SUBNET=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=Test-Private-Subnet" --query 'Subnets[0].SubnetId' --output text)

# Allocate Elastic IP for NAT Gateway
EIP_ALLOCATION=$(aws ec2 allocate-address --domain vpc --tag-specifications 'ResourceType=elastic-ip,Tags=[{Key=Name,Value=Test-NAT-EIP}]' --query 'AllocationId' --output text)

# Create NAT Gateway
NAT_GW=$(aws ec2 create-nat-gateway \
    --subnet-id $PUBLIC_SUBNET \
    --allocation-id $EIP_ALLOCATION \
    --tag-specifications 'ResourceType=nat-gateway,Tags=[{Key=Name,Value=Test-NAT-Gateway}]' \
    --query 'NatGateway.NatGatewayId' --output text)

# Wait for NAT Gateway to be available
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW

# Create route table for private subnet
PRIVATE_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Test-Private-RT}]' \
    --query 'RouteTable.RouteTableId' --output text)

# Add route to NAT Gateway
aws ec2 create-route --route-table-id $PRIVATE_RT --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW

# Associate private subnet with route table
aws ec2 associate-route-table --route-table-id $PRIVATE_RT --subnet-id $PRIVATE_SUBNET
```

**Launch Private Instance**:
```bash
# Create security group for private instance
PRIVATE_SG=$(aws ec2 create-security-group --group-name Private-Test-SG --description "Private Instance Security Group" --vpc-id $VPC_ID --query 'GroupId' --output text)

# Allow SSH from public subnet
aws ec2 authorize-security-group-ingress --group-id $PRIVATE_SG --protocol tcp --port 22 --cidr 10.0.50.0/24

# Launch private instance (no public IP)
aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --count 1 \
    --instance-type t3.micro \
    --key-name your-key-pair \
    --security-group-ids $PRIVATE_SG \
    --subnet-id $PRIVATE_SUBNET \
    --no-associate-public-ip-address \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Private-Test-Instance}]'
```

**Testing NAT Gateway Functionality**:
```bash
# SSH to public instance first
ssh -i your-key.pem ec2-user@[public-instance-ip]

# From public instance, SSH to private instance
ssh -i your-key.pem ec2-user@[private-instance-private-ip]

# From private instance, test outbound connectivity
curl -I https://aws.amazon.com  # Should work
ping 8.8.8.8                    # Should work

# From your local machine, try to reach private instance directly
ping [private-instance-private-ip]  # Should fail (no route)
```

### Step 4: Performance and Cost Analysis
**Objective**: Compare performance characteristics and costs

**Performance Testing**:
```bash
# On private instance, test download speed through NAT Gateway
wget -O /dev/null http://speedtest.wdc01.softlayer.com/downloads/test100.zip

# On public instance, test direct download speed
wget -O /dev/null http://speedtest.wdc01.softlayer.com/downloads/test100.zip

# Monitor NAT Gateway metrics in CloudWatch
aws cloudwatch get-metric-statistics \
    --namespace AWS/NATGateway \
    --metric-name BytesOutToDestination \
    --dimensions Name=NatGatewayId,Value=$NAT_GW \
    --start-time $(date -d '1 hour ago' -u +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Sum
```

**Cost Analysis**:
```bash
# Calculate NAT Gateway costs
echo "NAT Gateway Hourly Cost: $0.045 (us-east-1)"
echo "Data Processing Cost: $0.045 per GB"
echo "Elastic IP Cost: $0.005 per hour (when associated)"

# Compare with NAT Instance costs
echo "t3.nano instance: ~$3.50/month"
echo "Data transfer: Same as NAT Gateway"
echo "Management overhead: Additional operational cost"
```

## Advanced Configurations

### High Availability NAT Gateway Setup
```bash
# Create NAT Gateway in each AZ
for az in us-east-1a us-east-1b us-east-1c; do
    # Get public subnet for AZ
    PUBLIC_SUBNET_AZ=$(aws ec2 describe-subnets \
        --filters "Name=availability-zone,Values=$az" "Name=tag:Type,Values=Public" \
        --query 'Subnets[0].SubnetId' --output text)
    
    # Allocate EIP
    EIP_AZ=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
    
    # Create NAT Gateway
    NAT_GW_AZ=$(aws ec2 create-nat-gateway \
        --subnet-id $PUBLIC_SUBNET_AZ \
        --allocation-id $EIP_AZ \
        --query 'NatGateway.NatGatewayId' --output text)
    
    echo "Created NAT Gateway $NAT_GW_AZ in $az"
done
```

### Gateway Monitoring and Alerting
```bash
# Create CloudWatch alarm for NAT Gateway
aws cloudwatch put-metric-alarm \
    --alarm-name "NAT-Gateway-High-Data-Transfer" \
    --alarm-description "NAT Gateway high data transfer" \
    --metric-name BytesOutToDestination \
    --namespace AWS/NATGateway \
    --statistic Sum \
    --period 300 \
    --threshold 1000000000 \
    --comparison-operator GreaterThanThreshold \
    --dimensions Name=NatGatewayId,Value=$NAT_GW \
    --evaluation-periods 2
```

## Troubleshooting

| Issue | Symptoms | Cause | Solution |
|-------|----------|-------|---------|
| Public instance no internet | Cannot reach internet | Missing IGW route | Add 0.0.0.0/0 → IGW route |
| Private instance no internet | Outbound connections fail | Missing NAT route | Add 0.0.0.0/0 → NAT Gateway route |
| High NAT costs | Unexpected charges | Cross-AZ data transfer | Use NAT Gateway in same AZ as instances |
| NAT Gateway timeout | Connection timeouts | NAT Gateway overloaded | Scale to multiple NAT Gateways |
| Cannot reach private instance | SSH connection refused | No bastion host | Use bastion host in public subnet |

## Best Practices

### Internet Gateway
- One IGW per VPC (AWS limitation)
- No bandwidth constraints or availability concerns
- Use for resources requiring bidirectional internet access
- Secure with security groups and NACLs

### NAT Gateway
- Deploy one NAT Gateway per AZ for high availability
- Route private subnets to NAT Gateway in same AZ
- Monitor costs and data transfer patterns
- Use NAT instances for advanced features (port forwarding, bastion host)
- Consider VPC endpoints for AWS service access without internet

### Cost Optimization
- Use VPC endpoints for AWS services instead of NAT Gateway
- Implement data lifecycle policies to reduce transfer
- Monitor and alert on unusual data transfer patterns
- Consider NAT instances for lower-cost, lower-scale scenarios

## Key Takeaways
- IGW enables bidirectional internet access for public subnets
- NAT Gateway provides outbound-only internet access for private subnets
- NAT Gateway is managed, secure, and scalable but has costs
- High availability requires multiple NAT Gateways across AZs
- Route tables determine which gateway is used
- VPC endpoints can reduce NAT Gateway costs for AWS services

## Additional Resources
- [Internet Gateway Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Internet_Gateway.html)
- [NAT Gateway Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html)
- [VPC Endpoints](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-endpoints.html)

## Exam Tips
- IGW is free, NAT Gateway has hourly and data processing charges
- IGW is highly available across entire region, NAT Gateway is AZ-specific
- NAT Gateway cannot be used as bastion host (no inbound connections)
- Multiple NAT Gateways needed for high availability
- VPC can have only one IGW, but multiple NAT Gateways
- Security groups cannot be applied to IGW or NAT Gateway