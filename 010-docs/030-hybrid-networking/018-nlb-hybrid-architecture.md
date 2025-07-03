# Topic 18: Network Load Balancer for Hybrid Architecture

## Prerequisites
- Completed Foundation Module (Topics 1-10)
- Understanding of load balancing concepts and Layer 4 protocols
- Knowledge of hybrid networking patterns (Topics 11-17)
- Familiarity with AWS load balancer types and use cases

## Learning Objectives
By the end of this topic, you will be able to:
- Design Network Load Balancer architectures for hybrid environments
- Configure NLB for cross-zone and cross-region scenarios
- Implement health checks and failover mechanisms
- Optimize NLB performance for hybrid workloads

## Theory

### Network Load Balancer Overview

#### Definition
AWS Network Load Balancer (NLB) is a Layer 4 load balancer that operates at the transport layer, distributing incoming traffic across multiple targets based on IP protocol data.

#### Key Characteristics
- **Ultra-high Performance**: Millions of requests per second
- **Static IP Support**: Fixed IP addresses for each Availability Zone
- **Source IP Preservation**: Maintains original client IP addresses
- **Cross-Zone Load Balancing**: Optional traffic distribution across AZs
- **Health Checks**: TCP, HTTP, and HTTPS health monitoring

### NLB in Hybrid Architectures

#### Hybrid Use Cases for NLB
1. **On-premises to AWS Load Balancing**: Distribute traffic from on-premises clients
2. **Cross-region Load Balancing**: Balance traffic across multiple AWS regions
3. **Hybrid Application Tiers**: Balance between on-premises and cloud resources
4. **Edge Computing**: Distribute traffic to edge locations and AWS

#### Architecture Patterns

##### Pattern 1: On-premises Client to AWS Services
```
On-premises Clients → Direct Connect/VPN → NLB → AWS Targets
                                          ↓
                               EC2 Instances / Containers
```

##### Pattern 2: Cross-region Hybrid Load Balancing
```
                    Route 53 (DNS-based)
                           ↓
              ┌─────────────────────────────┐
              ↓                             ↓
    Region A: NLB → AWS Targets    Region B: NLB → AWS Targets
              ↓                             ↓
     On-premises Targets           On-premises Targets
```

##### Pattern 3: Edge-to-Core Distribution
```
Edge Locations → CloudFront → NLB → Application Tier
      ↓                      ↓           ↓
Local Processing    Regional Processing  Core Processing
```

### NLB Advanced Features for Hybrid

#### Static IP Addresses
- **Benefit**: Simplified firewall rules and DNS configuration
- **Use Case**: Corporate firewall policies requiring fixed IPs
- **Implementation**: One static IP per Availability Zone
- **Considerations**: Additional cost for Elastic IP addresses

#### Cross-Zone Load Balancing
- **Default Behavior**: Traffic distributed only within same AZ
- **Cross-Zone Enabled**: Traffic distributed across all healthy targets
- **Cost Impact**: Cross-AZ data transfer charges apply
- **Use Case**: Optimal distribution across all targets

#### Connection Draining
- **Purpose**: Graceful handling of target deregistration
- **Mechanism**: Complete existing connections before removal
- **Timeout**: Configurable (0-3600 seconds)
- **Benefits**: Zero-downtime deployments and maintenance

## Lab Exercise: Comprehensive NLB Hybrid Architecture

### Lab Duration: 300 minutes

### Step 1: Design Hybrid NLB Architecture
**Objective**: Plan NLB deployment for hybrid connectivity

**Architecture Planning Script**:
```bash
# Create hybrid NLB planning assessment
cat << 'EOF' > nlb-hybrid-planning.sh
#!/bin/bash

echo "=== Network Load Balancer Hybrid Architecture Planning ==="
echo ""

echo "1. TRAFFIC PATTERNS ANALYSIS"
echo "   Primary traffic source: On-premises/Internet/Both"
echo "   Expected connections per second: _______"
echo "   Peak bandwidth requirements: _______ Gbps"
echo "   Geographic distribution of clients: _______"
echo "   Protocol requirements: TCP/UDP/Both"
echo ""

echo "2. TARGET ARCHITECTURE"
echo "   Target types: EC2/IP/Lambda/ALB"
echo "   Target distribution: Single AZ/Multi-AZ/Multi-Region"
echo "   On-premises targets: Yes/No"
echo "   Container targets: ECS/EKS/Both/None"
echo "   Auto Scaling integration: Yes/No"
echo ""

echo "3. CONNECTIVITY REQUIREMENTS"
echo "   VPC connectivity: Direct Connect/VPN/Both"
echo "   Internet-facing requirements: Yes/No"
echo "   Cross-region connectivity: Yes/No"
echo "   Source IP preservation: Required/Optional"
echo "   Static IP requirements: Yes/No"
echo ""

echo "4. HIGH AVAILABILITY DESIGN"
echo "   Multi-AZ deployment: Yes/No"
echo "   Cross-zone load balancing: Enabled/Disabled"
echo "   Health check strategy: TCP/HTTP/HTTPS"
echo "   Failover requirements: Automatic/Manual"
echo "   RTO/RPO targets: _______"
echo ""

echo "5. PERFORMANCE REQUIREMENTS"
echo "   Latency requirements: _______ ms"
echo "   Throughput requirements: _______ req/sec"
echo "   Concurrent connections: _______"
echo "   SSL termination: Yes/No"
echo "   Connection persistence: Required/Optional"
echo ""

echo "6. MONITORING AND LOGGING"
echo "   CloudWatch metrics: Standard/Enhanced"
echo "   Access logging: Required/Optional"
echo "   VPC Flow Logs: Enabled/Disabled"
echo "   Third-party monitoring: Yes/No"
echo "   Alerting requirements: _______"

EOF

chmod +x nlb-hybrid-planning.sh
./nlb-hybrid-planning.sh
```

### Step 2: Create VPC Infrastructure for Hybrid NLB
**Objective**: Set up VPC foundation for NLB deployment

**VPC Infrastructure Setup**:
```bash
# Create VPC infrastructure for NLB
cat << 'EOF' > setup-nlb-vpc-infrastructure.sh
#!/bin/bash

echo "=== Setting Up VPC Infrastructure for Hybrid NLB ==="
echo ""

# Define VPC parameters
VPC_CIDR="10.0.0.0/16"
VPC_NAME="NLB-Hybrid-VPC"
REGION="us-east-1"

echo "1. CREATING VPC FOR NLB DEPLOYMENT"
echo "VPC CIDR: $VPC_CIDR"
echo "Region: $REGION"
echo ""

# Create VPC
VPC_ID=$(aws ec2 create-vpc \
    --cidr-block $VPC_CIDR \
    --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=$VPC_NAME}]" \
    --query 'Vpc.VpcId' \
    --output text)

echo "✅ VPC created: $VPC_ID"

# Enable DNS support
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames

echo "✅ DNS support enabled"

echo ""
echo "2. CREATING SUBNETS FOR MULTI-AZ DEPLOYMENT"

# Get availability zones
AZS=($(aws ec2 describe-availability-zones --region $REGION --query 'AvailabilityZones[0:3].ZoneName' --output text))

# Create public subnets for NLB
PUBLIC_SUBNET_IDS=()
for i in "${!AZS[@]}"; do
    AZ=${AZS[$i]}
    SUBNET_CIDR="10.0.$((i+1)).0/24"
    
    echo "Creating public subnet in $AZ: $SUBNET_CIDR"
    
    SUBNET_ID=$(aws ec2 create-subnet \
        --vpc-id $VPC_ID \
        --cidr-block $SUBNET_CIDR \
        --availability-zone $AZ \
        --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=NLB-Public-${AZ}},{Key=Type,Value=Public}]" \
        --query 'Subnet.SubnetId' \
        --output text)
    
    PUBLIC_SUBNET_IDS+=($SUBNET_ID)
    echo "✅ Public subnet created: $SUBNET_ID"
    
    # Enable auto-assign public IP
    aws ec2 modify-subnet-attribute \
        --subnet-id $SUBNET_ID \
        --map-public-ip-on-launch
done

# Create private subnets for targets
PRIVATE_SUBNET_IDS=()
for i in "${!AZS[@]}"; do
    AZ=${AZS[$i]}
    SUBNET_CIDR="10.0.$((i+10)).0/24"
    
    echo "Creating private subnet in $AZ: $SUBNET_CIDR"
    
    SUBNET_ID=$(aws ec2 create-subnet \
        --vpc-id $VPC_ID \
        --cidr-block $SUBNET_CIDR \
        --availability-zone $AZ \
        --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=NLB-Private-${AZ}},{Key=Type,Value=Private}]" \
        --query 'Subnet.SubnetId' \
        --output text)
    
    PRIVATE_SUBNET_IDS+=($SUBNET_ID)
    echo "✅ Private subnet created: $SUBNET_ID"
done

echo ""
echo "3. CREATING INTERNET GATEWAY"

# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
    --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=NLB-IGW}]" \
    --query 'InternetGateway.InternetGatewayId' \
    --output text)

echo "✅ Internet Gateway created: $IGW_ID"

# Attach to VPC
aws ec2 attach-internet-gateway \
    --internet-gateway-id $IGW_ID \
    --vpc-id $VPC_ID

echo "✅ Internet Gateway attached to VPC"

echo ""
echo "4. CREATING NAT GATEWAYS FOR PRIVATE SUBNETS"

# Create NAT Gateways in each public subnet
NAT_GW_IDS=()
for i in "${!PUBLIC_SUBNET_IDS[@]}"; do
    SUBNET_ID=${PUBLIC_SUBNET_IDS[$i]}
    AZ=${AZS[$i]}
    
    echo "Creating NAT Gateway in $AZ"
    
    # Allocate Elastic IP
    EIP_ALLOC_ID=$(aws ec2 allocate-address \
        --domain vpc \
        --tag-specifications "ResourceType=elastic-ip,Tags=[{Key=Name,Value=NLB-NAT-${AZ}}]" \
        --query 'AllocationId' \
        --output text)
    
    # Create NAT Gateway
    NAT_GW_ID=$(aws ec2 create-nat-gateway \
        --subnet-id $SUBNET_ID \
        --allocation-id $EIP_ALLOC_ID \
        --tag-specifications "ResourceType=nat-gateway,Tags=[{Key=Name,Value=NLB-NAT-${AZ}}]" \
        --query 'NatGateway.NatGatewayId' \
        --output text)
    
    NAT_GW_IDS+=($NAT_GW_ID)
    echo "✅ NAT Gateway created: $NAT_GW_ID"
done

echo ""
echo "5. CONFIGURING ROUTE TABLES"

# Create public route table
PUBLIC_RT_ID=$(aws ec2 create-route-table \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=NLB-Public-RT}]" \
    --query 'RouteTable.RouteTableId' \
    --output text)

echo "✅ Public route table created: $PUBLIC_RT_ID"

# Add route to Internet Gateway
aws ec2 create-route \
    --route-table-id $PUBLIC_RT_ID \
    --destination-cidr-block 0.0.0.0/0 \
    --gateway-id $IGW_ID

# Associate public subnets with public route table
for subnet_id in "${PUBLIC_SUBNET_IDS[@]}"; do
    aws ec2 associate-route-table \
        --route-table-id $PUBLIC_RT_ID \
        --subnet-id $subnet_id
    echo "✅ Associated public subnet $subnet_id with public route table"
done

# Create private route tables (one per AZ for NAT Gateway)
for i in "${!PRIVATE_SUBNET_IDS[@]}"; do
    SUBNET_ID=${PRIVATE_SUBNET_IDS[$i]}
    NAT_GW_ID=${NAT_GW_IDS[$i]}
    AZ=${AZS[$i]}
    
    # Create private route table
    PRIVATE_RT_ID=$(aws ec2 create-route-table \
        --vpc-id $VPC_ID \
        --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=NLB-Private-RT-${AZ}}]" \
        --query 'RouteTable.RouteTableId' \
        --output text)
    
    # Add route to NAT Gateway
    aws ec2 create-route \
        --route-table-id $PRIVATE_RT_ID \
        --destination-cidr-block 0.0.0.0/0 \
        --nat-gateway-id $NAT_GW_ID
    
    # Associate private subnet
    aws ec2 associate-route-table \
        --route-table-id $PRIVATE_RT_ID \
        --subnet-id $SUBNET_ID
    
    echo "✅ Private route table created and associated: $PRIVATE_RT_ID"
done

# Save configuration
cat << CONFIG > nlb-vpc-config.env
export VPC_ID="$VPC_ID"
export IGW_ID="$IGW_ID"
export PUBLIC_SUBNET_IDS=(${PUBLIC_SUBNET_IDS[@]})
export PRIVATE_SUBNET_IDS=(${PRIVATE_SUBNET_IDS[@]})
export NAT_GW_IDS=(${NAT_GW_IDS[@]})
export AZS=(${AZS[@]})
CONFIG

echo ""
echo "VPC INFRASTRUCTURE SUMMARY:"
echo "VPC ID: $VPC_ID"
echo "Public Subnets: ${PUBLIC_SUBNET_IDS[@]}"
echo "Private Subnets: ${PRIVATE_SUBNET_IDS[@]}"
echo "NAT Gateways: ${NAT_GW_IDS[@]}"
echo ""
echo "Configuration saved to nlb-vpc-config.env"

EOF

chmod +x setup-nlb-vpc-infrastructure.sh
./setup-nlb-vpc-infrastructure.sh
```

### Step 3: Deploy Target Infrastructure
**Objective**: Create target instances and services for NLB

**Target Infrastructure Deployment**:
```bash
# Create target infrastructure for NLB
cat << 'EOF' > deploy-nlb-targets.sh
#!/bin/bash

source nlb-vpc-config.env

echo "=== Deploying Target Infrastructure for NLB ==="
echo ""

echo "1. CREATING SECURITY GROUPS"

# Security group for NLB targets
TARGET_SG_ID=$(aws ec2 create-security-group \
    --group-name NLB-Target-SG \
    --description "Security group for NLB targets" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=NLB-Target-SG}]" \
    --query 'GroupId' \
    --output text)

echo "✅ Target security group created: $TARGET_SG_ID"

# Allow HTTP traffic from anywhere (NLB preserves source IP)
aws ec2 authorize-security-group-ingress \
    --group-id $TARGET_SG_ID \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0

# Allow HTTPS traffic
aws ec2 authorize-security-group-ingress \
    --group-id $TARGET_SG_ID \
    --protocol tcp \
    --port 443 \
    --cidr 0.0.0.0/0

# Allow SSH for management
aws ec2 authorize-security-group-ingress \
    --group-id $TARGET_SG_ID \
    --protocol tcp \
    --port 22 \
    --cidr 10.0.0.0/16

# Allow health checks from NLB
aws ec2 authorize-security-group-ingress \
    --group-id $TARGET_SG_ID \
    --protocol tcp \
    --port 8080 \
    --cidr 10.0.0.0/16

echo "✅ Security group rules configured"

echo ""
echo "2. CREATING TARGET INSTANCES"

# Get latest Amazon Linux 2 AMI
AMI_ID=$(aws ec2 describe-images \
    --owners amazon \
    --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" "Name=state,Values=available" \
    --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
    --output text)

echo "Using AMI: $AMI_ID"

# User data script for web server setup
USER_DATA=$(cat << 'USERDATA'
#!/bin/bash
yum update -y
yum install -y httpd

# Create instance metadata webpage
INSTANCE_ID=$(curl -s http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s http://169.254.169.254/latest/meta-data/placement/availability-zone)
LOCAL_IP=$(curl -s http://169.254.169.254/latest/meta-data/local-ipv4)

cat << EOF > /var/www/html/index.html
<!DOCTYPE html>
<html>
<head>
    <title>NLB Target Server</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 40px; }
        .server-info { background: #f0f0f0; padding: 20px; border-radius: 5px; }
        .az-display { font-size: 24px; font-weight: bold; color: #d63384; }
    </style>
</head>
<body>
    <h1>Network Load Balancer Target</h1>
    <div class="server-info">
        <div class="az-display">Availability Zone: $AZ</div>
        <p><strong>Instance ID:</strong> $INSTANCE_ID</p>
        <p><strong>Local IP:</strong> $LOCAL_IP</p>
        <p><strong>Timestamp:</strong> $(date)</p>
    </div>
    <h2>Health Check Endpoint</h2>
    <p>Health check available at <a href="/health">http://$LOCAL_IP/health</a></p>
</body>
</html>
EOF

# Health check endpoint
cat << EOF > /var/www/html/health
OK
EOF

# Start and enable Apache
systemctl start httpd
systemctl enable httpd

# Health check service on port 8080
yum install -y nc
echo "while true; do echo -e 'HTTP/1.1 200 OK\r\n\r\nOK' | nc -l -p 8080 -q 1; done" > /opt/health-check.sh
chmod +x /opt/health-check.sh
nohup /opt/health-check.sh > /dev/null 2>&1 &

USERDATA
)

# Encode user data
USER_DATA_ENCODED=$(echo "$USER_DATA" | base64 -w 0)

# Launch instances in each private subnet
TARGET_INSTANCE_IDS=()
for i in "${!PRIVATE_SUBNET_IDS[@]}"; do
    SUBNET_ID=${PRIVATE_SUBNET_IDS[$i]}
    AZ=${AZS[$i]}
    
    echo "Launching instance in $AZ (subnet: $SUBNET_ID)"
    
    INSTANCE_ID=$(aws ec2 run-instances \
        --image-id $AMI_ID \
        --count 1 \
        --instance-type t3.micro \
        --security-group-ids $TARGET_SG_ID \
        --subnet-id $SUBNET_ID \
        --user-data "$USER_DATA_ENCODED" \
        --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=NLB-Target-${AZ}},{Key=Purpose,Value=NLB-Target}]" \
        --query 'Instances[0].InstanceId' \
        --output text)
    
    TARGET_INSTANCE_IDS+=($INSTANCE_ID)
    echo "✅ Instance launched: $INSTANCE_ID"
done

echo ""
echo "3. WAITING FOR INSTANCES TO BE READY"

# Wait for all instances to be running
for instance_id in "${TARGET_INSTANCE_IDS[@]}"; do
    echo "Waiting for instance $instance_id to be running..."
    aws ec2 wait instance-running --instance-ids $instance_id
    echo "✅ Instance $instance_id is running"
done

echo ""
echo "4. CREATING ADDITIONAL TARGETS (IP TARGETS)"

# Create Lambda function as IP target example
cat << 'LAMBDA_CODE' > nlb-lambda-target.py
import json
import socket

def lambda_handler(event, context):
    """
    Lambda function to serve as NLB target
    """
    
    # Get client information from event
    source_ip = event.get('requestContext', {}).get('identity', {}).get('sourceIp', 'unknown')
    
    # Get Lambda container info
    hostname = socket.gethostname()
    
    response_body = {
        'message': 'Hello from Lambda NLB Target',
        'hostname': hostname,
        'source_ip': source_ip,
        'timestamp': context.aws_request_id
    }
    
    return {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json'
        },
        'body': json.dumps(response_body)
    }
LAMBDA_CODE

echo "✅ Lambda function code created: nlb-lambda-target.py"

# Update configuration
cat << CONFIG >> nlb-vpc-config.env
export TARGET_SG_ID="$TARGET_SG_ID"
export TARGET_INSTANCE_IDS=(${TARGET_INSTANCE_IDS[@]})
export AMI_ID="$AMI_ID"
CONFIG

echo ""
echo "TARGET INFRASTRUCTURE SUMMARY:"
echo "Security Group: $TARGET_SG_ID"
echo "Target Instances: ${TARGET_INSTANCE_IDS[@]}"
echo "AMI Used: $AMI_ID"
echo ""
echo "Configuration updated in nlb-vpc-config.env"

EOF

chmod +x deploy-nlb-targets.sh
./deploy-nlb-targets.sh
```

### Step 4: Create Network Load Balancer
**Objective**: Deploy and configure NLB for hybrid architecture

**NLB Creation and Configuration**:
```bash
# Create Network Load Balancer
cat << 'EOF' > create-network-load-balancer.sh
#!/bin/bash

source nlb-vpc-config.env

echo "=== Creating Network Load Balancer ==="
echo ""

NLB_NAME="Hybrid-Network-LB"
echo "1. CREATING NETWORK LOAD BALANCER"
echo "Name: $NLB_NAME"
echo "Scheme: internet-facing"
echo "Type: network"
echo ""

# Create NLB
NLB_ARN=$(aws elbv2 create-load-balancer \
    --name $NLB_NAME \
    --scheme internet-facing \
    --type network \
    --ip-address-type ipv4 \
    --subnets ${PUBLIC_SUBNET_IDS[@]} \
    --tags Key=Name,Value=$NLB_NAME Key=Purpose,Value=Hybrid-Architecture \
    --query 'LoadBalancers[0].LoadBalancerArn' \
    --output text)

echo "✅ Network Load Balancer created: $NLB_ARN"

# Wait for NLB to be active
echo "Waiting for NLB to become active..."
aws elbv2 wait load-balancer-available --load-balancer-arns $NLB_ARN
echo "✅ NLB is now active"

# Get NLB DNS name
NLB_DNS=$(aws elbv2 describe-load-balancers \
    --load-balancer-arns $NLB_ARN \
    --query 'LoadBalancers[0].DNSName' \
    --output text)

echo "NLB DNS Name: $NLB_DNS"

echo ""
echo "2. ENABLING CROSS-ZONE LOAD BALANCING"

# Enable cross-zone load balancing
aws elbv2 modify-load-balancer-attributes \
    --load-balancer-arn $NLB_ARN \
    --attributes Key=load_balancing.cross_zone.enabled,Value=true

echo "✅ Cross-zone load balancing enabled"

echo ""
echo "3. CREATING TARGET GROUP FOR EC2 INSTANCES"

# Create target group for EC2 instances
TG_EC2_ARN=$(aws elbv2 create-target-group \
    --name NLB-EC2-Targets \
    --protocol TCP \
    --port 80 \
    --vpc-id $VPC_ID \
    --target-type instance \
    --health-check-protocol TCP \
    --health-check-port 8080 \
    --health-check-interval-seconds 30 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 2 \
    --tags Key=Name,Value=NLB-EC2-Targets \
    --query 'TargetGroups[0].TargetGroupArn' \
    --output text)

echo "✅ EC2 target group created: $TG_EC2_ARN"

# Register EC2 instances as targets
echo "Registering EC2 instances as targets..."
for instance_id in "${TARGET_INSTANCE_IDS[@]}"; do
    aws elbv2 register-targets \
        --target-group-arn $TG_EC2_ARN \
        --targets Id=$instance_id,Port=80
    echo "✅ Registered instance: $instance_id"
done

echo ""
echo "4. CREATING TARGET GROUP FOR IP TARGETS"

# Create target group for IP targets (for hybrid scenarios)
TG_IP_ARN=$(aws elbv2 create-target-group \
    --name NLB-IP-Targets \
    --protocol TCP \
    --port 80 \
    --vpc-id $VPC_ID \
    --target-type ip \
    --health-check-protocol HTTP \
    --health-check-path /health \
    --health-check-port 80 \
    --health-check-interval-seconds 30 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 2 \
    --tags Key=Name,Value=NLB-IP-Targets \
    --query 'TargetGroups[0].TargetGroupArn' \
    --output text)

echo "✅ IP target group created: $TG_IP_ARN"

# Get private IPs of instances for IP targeting
echo "Getting private IPs for IP target registration..."
PRIVATE_IPS=()
for instance_id in "${TARGET_INSTANCE_IDS[@]}"; do
    PRIVATE_IP=$(aws ec2 describe-instances \
        --instance-ids $instance_id \
        --query 'Reservations[0].Instances[0].PrivateIpAddress' \
        --output text)
    PRIVATE_IPS+=($PRIVATE_IP)
    echo "Instance $instance_id private IP: $PRIVATE_IP"
done

# Register private IPs as targets
for ip in "${PRIVATE_IPS[@]}"; do
    aws elbv2 register-targets \
        --target-group-arn $TG_IP_ARN \
        --targets Id=$ip,Port=80
    echo "✅ Registered IP target: $ip"
done

echo ""
echo "5. CREATING LISTENERS"

# Create listener for port 80 (EC2 targets)
LISTENER_80_ARN=$(aws elbv2 create-listener \
    --load-balancer-arn $NLB_ARN \
    --protocol TCP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=$TG_EC2_ARN \
    --tags Key=Name,Value=NLB-HTTP-Listener \
    --query 'Listeners[0].ListenerArn' \
    --output text)

echo "✅ HTTP listener created: $LISTENER_80_ARN"

# Create listener for port 8080 (IP targets)
LISTENER_8080_ARN=$(aws elbv2 create-listener \
    --load-balancer-arn $NLB_ARN \
    --protocol TCP \
    --port 8080 \
    --default-actions Type=forward,TargetGroupArn=$TG_IP_ARN \
    --tags Key=Name,Value=NLB-Alt-HTTP-Listener \
    --query 'Listeners[0].ListenerArn' \
    --output text)

echo "✅ Alternative HTTP listener created: $LISTENER_8080_ARN"

echo ""
echo "6. CONFIGURING ADVANCED ATTRIBUTES"

# Configure target group attributes for performance
aws elbv2 modify-target-group-attributes \
    --target-group-arn $TG_EC2_ARN \
    --attributes Key=preserve_client_ip.enabled,Value=true \
               Key=proxy_protocol_v2.enabled,Value=false \
               Key=deregistration_delay.timeout_seconds,Value=30 \
               Key=stickiness.enabled,Value=true \
               Key=stickiness.type,Value=source_ip

aws elbv2 modify-target-group-attributes \
    --target-group-arn $TG_IP_ARN \
    --attributes Key=preserve_client_ip.enabled,Value=true \
               Key=deregistration_delay.timeout_seconds,Value=30

echo "✅ Target group attributes configured"

# Update configuration file
cat << CONFIG >> nlb-vpc-config.env
export NLB_ARN="$NLB_ARN"
export NLB_DNS="$NLB_DNS"
export TG_EC2_ARN="$TG_EC2_ARN"
export TG_IP_ARN="$TG_IP_ARN"
export LISTENER_80_ARN="$LISTENER_80_ARN"
export LISTENER_8080_ARN="$LISTENER_8080_ARN"
export PRIVATE_IPS=(${PRIVATE_IPS[@]})
CONFIG

echo ""
echo "NETWORK LOAD BALANCER SUMMARY:"
echo "NLB ARN: $NLB_ARN"
echo "NLB DNS: $NLB_DNS"
echo "EC2 Target Group: $TG_EC2_ARN"
echo "IP Target Group: $TG_IP_ARN"
echo "HTTP Listener (80): $LISTENER_80_ARN"
echo "Alt HTTP Listener (8080): $LISTENER_8080_ARN"
echo ""
echo "Configuration updated in nlb-vpc-config.env"

EOF

chmod +x create-network-load-balancer.sh
./create-network-load-balancer.sh
```

### Step 5: Configure Hybrid Connectivity
**Objective**: Set up hybrid connectivity for NLB access

**Hybrid Connectivity Configuration**:
```bash
# Configure hybrid connectivity for NLB
cat << 'EOF' > configure-hybrid-connectivity.sh
#!/bin/bash

source nlb-vpc-config.env

echo "=== Configuring Hybrid Connectivity for NLB ==="
echo ""

echo "1. CREATING VIRTUAL PRIVATE GATEWAY"

# Create VGW for hybrid connectivity
VGW_ID=$(aws ec2 create-vpn-gateway \
    --type ipsec.1 \
    --amazon-side-asn 64512 \
    --tag-specifications "ResourceType=vpn-gateway,Tags=[{Key=Name,Value=NLB-Hybrid-VGW}]" \
    --query 'VpnGateway.VpnGatewayId' \
    --output text)

echo "✅ Virtual Private Gateway created: $VGW_ID"

# Attach VGW to VPC
aws ec2 attach-vpn-gateway \
    --vpn-gateway-id $VGW_ID \
    --vpc-id $VPC_ID

echo "Waiting for VGW attachment..."
aws ec2 wait vpn-gateway-attached --vpn-gateway-ids $VGW_ID
echo "✅ VGW attached to VPC"

echo ""
echo "2. CREATING CUSTOMER GATEWAY (SIMULATED)"

# Create customer gateway for on-premises simulation
CUSTOMER_PUBLIC_IP="203.0.113.10"  # Example public IP
CUSTOMER_ASN="65001"

CGW_ID=$(aws ec2 create-customer-gateway \
    --type ipsec.1 \
    --public-ip $CUSTOMER_PUBLIC_IP \
    --bgp-asn $CUSTOMER_ASN \
    --device-name "NLB-Hybrid-CGW" \
    --tag-specifications "ResourceType=customer-gateway,Tags=[{Key=Name,Value=NLB-Hybrid-CGW}]" \
    --query 'CustomerGateway.CustomerGatewayId' \
    --output text)

echo "✅ Customer Gateway created: $CGW_ID"

echo ""
echo "3. CREATING VPN CONNECTION"

# Create VPN connection
VPN_ID=$(aws ec2 create-vpn-connection \
    --type ipsec.1 \
    --customer-gateway-id $CGW_ID \
    --vpn-gateway-id $VGW_ID \
    --options StaticRoutesOnly=false \
    --tag-specifications "ResourceType=vpn-connection,Tags=[{Key=Name,Value=NLB-Hybrid-VPN}]" \
    --query 'VpnConnection.VpnConnectionId' \
    --output text)

echo "✅ VPN Connection created: $VPN_ID"

# Wait for VPN to become available
echo "Waiting for VPN connection to become available..."
aws ec2 wait vpn-connection-available --vpn-connection-ids $VPN_ID
echo "✅ VPN connection is available"

echo ""
echo "4. CONFIGURING ROUTE PROPAGATION"

# Get route table IDs for the VPC
ROUTE_TABLE_IDS=$(aws ec2 describe-route-tables \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query 'RouteTables[*].RouteTableId' \
    --output text)

# Enable route propagation for VGW
for rt_id in $ROUTE_TABLE_IDS; do
    echo "Enabling route propagation on route table: $rt_id"
    aws ec2 enable-vgw-route-propagation \
        --route-table-id $rt_id \
        --gateway-id $VGW_ID
    echo "✅ Route propagation enabled on $rt_id"
done

echo ""
echo "5. CREATING DIRECT CONNECT GATEWAY (SIMULATION)"

# Create Direct Connect Gateway for enterprise connectivity
DXGW_NAME="NLB-Hybrid-DXGW"

echo "Creating Direct Connect Gateway: $DXGW_NAME"
echo "Note: This is a simulation - actual DX setup requires physical connectivity"

# In a real scenario, you would create a DX Gateway like this:
cat << 'DX_SETUP'

# Create Direct Connect Gateway
DXGW_ID=$(aws directconnect create-direct-connect-gateway \
    --name $DXGW_NAME \
    --query 'directConnectGateway.directConnectGatewayId' \
    --output text)

# Associate with Virtual Private Gateway
aws directconnect create-direct-connect-gateway-association \
    --direct-connect-gateway-id $DXGW_ID \
    --gateway-id $VGW_ID

DX_SETUP

echo "✅ Direct Connect Gateway configuration template created"

echo ""
echo "6. CONFIGURING SECURITY GROUPS FOR HYBRID ACCESS"

# Create security group for hybrid NLB access
HYBRID_SG_ID=$(aws ec2 create-security-group \
    --group-name NLB-Hybrid-Access-SG \
    --description "Security group for hybrid NLB access" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=NLB-Hybrid-Access-SG}]" \
    --query 'GroupId' \
    --output text)

echo "✅ Hybrid access security group created: $HYBRID_SG_ID"

# Allow access from on-premises networks
ONPREM_CIDR="10.1.0.0/16"  # Example on-premises CIDR

# Allow HTTP from on-premises
aws ec2 authorize-security-group-ingress \
    --group-id $HYBRID_SG_ID \
    --protocol tcp \
    --port 80 \
    --cidr $ONPREM_CIDR

# Allow HTTPS from on-premises
aws ec2 authorize-security-group-ingress \
    --group-id $HYBRID_SG_ID \
    --protocol tcp \
    --port 443 \
    --cidr $ONPREM_CIDR

# Allow custom application ports
aws ec2 authorize-security-group-ingress \
    --group-id $HYBRID_SG_ID \
    --protocol tcp \
    --port 8080 \
    --cidr $ONPREM_CIDR

echo "✅ Hybrid access security group rules configured"

echo ""
echo "7. SETTING UP ROUTE 53 FOR HYBRID DNS"

# Create private hosted zone for hybrid DNS resolution
HOSTED_ZONE_ID=$(aws route53 create-hosted-zone \
    --name "hybrid.internal" \
    --vpc VPCRegion=us-east-1,VPCId=$VPC_ID \
    --caller-reference "nlb-hybrid-$(date +%s)" \
    --hosted-zone-config Comment="Private zone for NLB hybrid access" PrivateZone=true \
    --query 'HostedZone.Id' \
    --output text)

echo "✅ Private hosted zone created: $HOSTED_ZONE_ID"

# Create A record for NLB
NLB_IPS=$(aws elbv2 describe-load-balancers \
    --load-balancer-arns $NLB_ARN \
    --query 'LoadBalancers[0].AvailabilityZones[*].LoadBalancerAddresses[*].IpAddress' \
    --output text)

# Create record set for NLB
cat << RECORD_SET > nlb-record-set.json
{
    "Comment": "A record for NLB hybrid access",
    "Changes": [
        {
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "app.hybrid.internal",
                "Type": "A",
                "TTL": 300,
                "ResourceRecords": [
                    $(echo "$NLB_IPS" | awk '{for(i=1;i<=NF;i++) print "{\"Value\":\"" $i "\"},"}'  | sed '$ s/,$//')
                ]
            }
        }
    ]
}
RECORD_SET

aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch file://nlb-record-set.json

echo "✅ DNS record created for NLB: app.hybrid.internal"

# Update configuration
cat << CONFIG >> nlb-vpc-config.env
export VGW_ID="$VGW_ID"
export CGW_ID="$CGW_ID"
export VPN_ID="$VPN_ID"
export HYBRID_SG_ID="$HYBRID_SG_ID"
export HOSTED_ZONE_ID="$HOSTED_ZONE_ID"
export ONPREM_CIDR="$ONPREM_CIDR"
CONFIG

echo ""
echo "HYBRID CONNECTIVITY SUMMARY:"
echo "VGW ID: $VGW_ID"
echo "Customer Gateway: $CGW_ID"
echo "VPN Connection: $VPN_ID"
echo "Hybrid Security Group: $HYBRID_SG_ID"
echo "Private Hosted Zone: $HOSTED_ZONE_ID"
echo "NLB Hybrid DNS: app.hybrid.internal"
echo ""
echo "Configuration updated in nlb-vpc-config.env"

EOF

chmod +x configure-hybrid-connectivity.sh
./configure-hybrid-connectivity.sh
```

### Step 6: Implement Advanced NLB Features
**Objective**: Configure advanced NLB features for production use

**Advanced Features Configuration**:
```bash
# Configure advanced NLB features
cat << 'EOF' > configure-advanced-nlb-features.sh
#!/bin/bash

source nlb-vpc-config.env

echo "=== Configuring Advanced NLB Features ==="
echo ""

echo "1. CONFIGURING SSL/TLS TERMINATION"

# Create certificate for HTTPS listener (using ACM)
DOMAIN_NAME="nlb.example.com"

echo "Creating SSL certificate for $DOMAIN_NAME"
echo "Note: In production, use a real domain and DNS validation"

# Request certificate (this would require domain validation in real scenario)
cat << 'SSL_CONFIG'

# Request ACM certificate
CERT_ARN=$(aws acm request-certificate \
    --domain-name $DOMAIN_NAME \
    --validation-method DNS \
    --subject-alternative-names "*.nlb.example.com" \
    --tags Key=Name,Value=NLB-SSL-Certificate \
    --query 'CertificateArn' \
    --output text)

# Create HTTPS listener
LISTENER_443_ARN=$(aws elbv2 create-listener \
    --load-balancer-arn $NLB_ARN \
    --protocol TLS \
    --port 443 \
    --certificates CertificateArn=$CERT_ARN \
    --default-actions Type=forward,TargetGroupArn=$TG_EC2_ARN \
    --ssl-policy ELBSecurityPolicy-TLS-1-2-2017-01 \
    --query 'Listeners[0].ListenerArn' \
    --output text)

SSL_CONFIG

echo "✅ SSL/TLS configuration template created"

echo ""
echo "2. IMPLEMENTING HEALTH CHECK OPTIMIZATION"

# Configure optimized health checks for EC2 targets
aws elbv2 modify-target-group \
    --target-group-arn $TG_EC2_ARN \
    --health-check-protocol TCP \
    --health-check-port 8080 \
    --health-check-interval-seconds 10 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 2

echo "✅ EC2 target group health checks optimized"

# Configure HTTP health checks for IP targets
aws elbv2 modify-target-group \
    --target-group-arn $TG_IP_ARN \
    --health-check-protocol HTTP \
    --health-check-path "/health" \
    --health-check-port "traffic-port" \
    --health-check-interval-seconds 15 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 3 \
    --matcher HttpCode=200

echo "✅ IP target group health checks optimized"

echo ""
echo "3. CONFIGURING CONNECTION DRAINING"

# Set optimal deregistration delay
aws elbv2 modify-target-group-attributes \
    --target-group-arn $TG_EC2_ARN \
    --attributes Key=deregistration_delay.timeout_seconds,Value=60

aws elbv2 modify-target-group-attributes \
    --target-group-arn $TG_IP_ARN \
    --attributes Key=deregistration_delay.timeout_seconds,Value=30

echo "✅ Connection draining configured"

echo ""
echo "4. ENABLING ACCESS LOGGING"

# Create S3 bucket for access logs
BUCKET_NAME="nlb-access-logs-$(date +%s)"
ACCOUNT_ID=$(aws sts get-caller-identity --query 'Account' --output text)
REGION=$(aws configure get region)

aws s3api create-bucket \
    --bucket $BUCKET_NAME \
    --region $REGION

echo "✅ S3 bucket created: $BUCKET_NAME"

# Configure bucket policy for ELB access
ELB_ACCOUNT_ID=$(aws elbv2 describe-account-attributes \
    --attribute-names access-logs.s3.enabled \
    --query 'AccountAttributes[0].AttributeValue' \
    --output text 2>/dev/null || echo "027434742980")  # us-east-1 ELB account ID

cat << BUCKET_POLICY > bucket-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::$ELB_ACCOUNT_ID:root"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::$BUCKET_NAME/nlb-access-logs/AWSLogs/$ACCOUNT_ID/*"
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "delivery.logs.amazonaws.com"
            },
            "Action": "s3:PutObject",
            "Resource": "arn:aws:s3:::$BUCKET_NAME/nlb-access-logs/AWSLogs/$ACCOUNT_ID/*",
            "Condition": {
                "StringEquals": {
                    "s3:x-amz-acl": "bucket-owner-full-control"
                }
            }
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "delivery.logs.amazonaws.com"
            },
            "Action": "s3:GetBucketAcl",
            "Resource": "arn:aws:s3:::$BUCKET_NAME"
        }
    ]
}
BUCKET_POLICY

aws s3api put-bucket-policy \
    --bucket $BUCKET_NAME \
    --policy file://bucket-policy.json

# Enable access logging
aws elbv2 modify-load-balancer-attributes \
    --load-balancer-arn $NLB_ARN \
    --attributes Key=access_logs.s3.enabled,Value=true \
               Key=access_logs.s3.bucket,Value=$BUCKET_NAME \
               Key=access_logs.s3.prefix,Value=nlb-access-logs

echo "✅ Access logging enabled to S3 bucket: $BUCKET_NAME"

echo ""
echo "5. CONFIGURING FLOW STICKINESS"

# Enable flow hash stickiness for consistent routing
aws elbv2 modify-target-group-attributes \
    --target-group-arn $TG_EC2_ARN \
    --attributes Key=stickiness.enabled,Value=true \
               Key=stickiness.type,Value=source_ip

echo "✅ Flow stickiness enabled for EC2 targets"

echo ""
echo "6. SETTING UP CLOUDWATCH ENHANCED MONITORING"

# Create CloudWatch dashboard for NLB monitoring
DASHBOARD_NAME="NLB-Hybrid-Monitoring"

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
                    [ "AWS/NetworkELB", "ActiveFlowCount", "LoadBalancer", "'$(echo $NLB_ARN | cut -d'/' -f2-)'" ],
                    [ ".", "NewFlowCount", ".", "." ],
                    [ ".", "ProcessedBytes", ".", "." ]
                ],
                "period": 300,
                "stat": "Average",
                "region": "'$REGION'",
                "title": "NLB Connection Metrics"
            }
        },
        {
            "type": "metric",
            "x": 12,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "metrics": [
                    [ "AWS/NetworkELB", "HealthyHostCount", "TargetGroup", "'$(echo $TG_EC2_ARN | cut -d'/' -f2-)'" ],
                    [ ".", "UnHealthyHostCount", ".", "." ]
                ],
                "period": 300,
                "stat": "Average",
                "region": "'$REGION'",
                "title": "Target Health Status"
            }
        },
        {
            "type": "metric",
            "x": 0,
            "y": 6,
            "width": 24,
            "height": 6,
            "properties": {
                "metrics": [
                    [ "AWS/NetworkELB", "TargetResponseTime", "LoadBalancer", "'$(echo $NLB_ARN | cut -d'/' -f2-)'" ]
                ],
                "period": 300,
                "stat": "Average",
                "region": "'$REGION'",
                "title": "Target Response Time"
            }
        }
    ]
}'

aws cloudwatch put-dashboard \
    --dashboard-name $DASHBOARD_NAME \
    --dashboard-body "$DASHBOARD_BODY"

echo "✅ CloudWatch dashboard created: $DASHBOARD_NAME"

# Create CloudWatch alarms
aws cloudwatch put-metric-alarm \
    --alarm-name "NLB-UnhealthyTargets" \
    --alarm-description "Alert when NLB has unhealthy targets" \
    --metric-name UnHealthyHostCount \
    --namespace AWS/NetworkELB \
    --statistic Maximum \
    --period 60 \
    --threshold 1 \
    --comparison-operator GreaterThanOrEqualToThreshold \
    --dimensions Name=TargetGroup,Value=$(echo $TG_EC2_ARN | cut -d'/' -f2-) \
    --evaluation-periods 2

echo "✅ CloudWatch alarms configured"

echo ""
echo "7. IMPLEMENTING AUTO SCALING INTEGRATION"

# Create Auto Scaling Group for NLB targets
cat << 'ASG_CONFIG' > create-asg-for-nlb.sh
#!/bin/bash

# Create launch template
LAUNCH_TEMPLATE_ID=$(aws ec2 create-launch-template \
    --launch-template-name NLB-Target-Template \
    --launch-template-data '{
        "ImageId": "'$AMI_ID'",
        "InstanceType": "t3.micro",
        "SecurityGroupIds": ["'$TARGET_SG_ID'"],
        "UserData": "'$(echo "$USER_DATA" | base64 -w 0)'",
        "TagSpecifications": [
            {
                "ResourceType": "instance",
                "Tags": [
                    {"Key": "Name", "Value": "NLB-ASG-Instance"},
                    {"Key": "Purpose", "Value": "NLB-Target"}
                ]
            }
        ]
    }' \
    --query 'LaunchTemplate.LaunchTemplateId' \
    --output text)

# Create Auto Scaling Group
aws autoscaling create-auto-scaling-group \
    --auto-scaling-group-name NLB-Target-ASG \
    --launch-template LaunchTemplateId=$LAUNCH_TEMPLATE_ID,Version='$Latest' \
    --min-size 2 \
    --max-size 6 \
    --desired-capacity 3 \
    --target-group-arns $TG_EC2_ARN \
    --vpc-zone-identifier $(IFS=,; echo "${PRIVATE_SUBNET_IDS[*]}") \
    --health-check-type ELB \
    --health-check-grace-period 300 \
    --tags Key=Name,Value=NLB-Target-ASG,PropagateAtLaunch=true

echo "✅ Auto Scaling Group created for NLB targets"

ASG_CONFIG

chmod +x create-asg-for-nlb.sh

echo "✅ Auto Scaling integration script created"

# Update configuration
cat << CONFIG >> nlb-vpc-config.env
export BUCKET_NAME="$BUCKET_NAME"
export DASHBOARD_NAME="$DASHBOARD_NAME"
CONFIG

echo ""
echo "ADVANCED FEATURES CONFIGURATION COMPLETE:"
echo "- SSL/TLS termination template created"
echo "- Health checks optimized"
echo "- Connection draining configured"
echo "- Access logging enabled to: $BUCKET_NAME"
echo "- Flow stickiness enabled"
echo "- CloudWatch monitoring dashboard: $DASHBOARD_NAME"
echo "- Auto Scaling integration script created"
echo ""
echo "Configuration updated in nlb-vpc-config.env"

EOF

chmod +x configure-advanced-nlb-features.sh
./configure-advanced-nlb-features.sh
```

### Step 7: Test and Validate NLB Performance
**Objective**: Test NLB functionality and performance

**NLB Testing and Validation**:
```bash
# Create comprehensive NLB testing script
cat << 'EOF' > test-nlb-performance.sh
#!/bin/bash

source nlb-vpc-config.env

echo "=== Testing and Validating NLB Performance ==="
echo ""

echo "1. BASIC CONNECTIVITY TESTS"

echo "Testing NLB DNS resolution..."
nslookup $NLB_DNS

echo ""
echo "Testing HTTP connectivity to NLB..."
for i in {1..5}; do
    echo "Request $i:"
    curl -s -I http://$NLB_DNS/ | head -1
    sleep 1
done

echo ""
echo "Testing load balancing distribution..."
for i in {1..10}; do
    RESPONSE=$(curl -s http://$NLB_DNS/ | grep "Availability Zone" | head -1)
    echo "Request $i: $RESPONSE"
    sleep 0.5
done

echo ""
echo "2. TARGET HEALTH VALIDATION"

# Check target health for EC2 targets
echo "EC2 Target Group Health:"
aws elbv2 describe-target-health \
    --target-group-arn $TG_EC2_ARN \
    --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State,TargetHealth.Description]' \
    --output table

echo ""
echo "IP Target Group Health:"
aws elbv2 describe-target-health \
    --target-group-arn $TG_IP_ARN \
    --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State,TargetHealth.Description]' \
    --output table

echo ""
echo "3. PERFORMANCE TESTING"

# Test throughput and latency
echo "Testing NLB performance with multiple concurrent connections..."

# Install testing tools if not available
if ! command -v ab >/dev/null 2>&1; then
    echo "Installing Apache Bench for performance testing..."
    sudo yum install -y httpd-tools 2>/dev/null || \
    sudo apt-get install -y apache2-utils 2>/dev/null || \
    echo "Please install Apache Bench (ab) manually"
fi

if command -v ab >/dev/null 2>&1; then
    echo "Running Apache Bench performance test..."
    ab -n 1000 -c 10 http://$NLB_DNS/
else
    echo "Apache Bench not available, using curl for basic testing..."
    
    echo "Testing response times:"
    for i in {1..10}; do
        RESPONSE_TIME=$(curl -w "%{time_total}" -s -o /dev/null http://$NLB_DNS/)
        echo "Request $i: ${RESPONSE_TIME}s"
    done
fi

echo ""
echo "4. FAILOVER TESTING"

# Test target failover behavior
echo "Testing target failover (simulating target failure)..."

# Get one target instance ID for failover test
FIRST_INSTANCE=${TARGET_INSTANCE_IDS[0]}
echo "Stopping instance $FIRST_INSTANCE to simulate failure..."

# Stop instance
aws ec2 stop-instances --instance-ids $FIRST_INSTANCE
echo "Instance stop initiated. Monitoring health checks..."

# Monitor health status
for i in {1..20}; do
    HEALTHY_COUNT=$(aws elbv2 describe-target-health \
        --target-group-arn $TG_EC2_ARN \
        --query 'TargetHealthDescriptions[?TargetHealth.State==`healthy`] | length(@)' \
        --output text)
    
    echo "Iteration $i: $HEALTHY_COUNT healthy targets"
    
    if [ "$HEALTHY_COUNT" -lt "${#TARGET_INSTANCE_IDS[@]}" ]; then
        echo "✅ Target failure detected by health checks"
        break
    fi
    
    sleep 10
done

# Test connectivity during failover
echo "Testing connectivity during failover..."
for i in {1..5}; do
    curl -s http://$NLB_DNS/ | grep "Instance ID" || echo "Request failed"
    sleep 2
done

# Restart the instance
echo "Restarting instance $FIRST_INSTANCE..."
aws ec2 start-instances --instance-ids $FIRST_INSTANCE

echo ""
echo "5. CROSS-ZONE LOAD BALANCING VERIFICATION"

# Verify cross-zone load balancing behavior
echo "Verifying cross-zone load balancing..."

# Test requests and track AZ distribution
declare -A AZ_COUNT
TOTAL_REQUESTS=50

for i in $(seq 1 $TOTAL_REQUESTS); do
    AZ=$(curl -s http://$NLB_DNS/ | grep "Availability Zone" | sed 's/.*Zone: \([^<]*\).*/\1/')
    if [ ! -z "$AZ" ]; then
        AZ_COUNT[$AZ]=$((${AZ_COUNT[$AZ]:-0} + 1))
    fi
done

echo "Request distribution across Availability Zones:"
for az in "${!AZ_COUNT[@]}"; do
    PERCENTAGE=$(echo "scale=2; ${AZ_COUNT[$az]} * 100 / $TOTAL_REQUESTS" | bc)
    echo "$az: ${AZ_COUNT[$az]} requests ($PERCENTAGE%)"
done

echo ""
echo "6. MONITORING METRICS VALIDATION"

# Check CloudWatch metrics
echo "Recent NLB CloudWatch metrics:"

NLB_NAME=$(echo $NLB_ARN | cut -d'/' -f2-)

# Get active flow count
aws cloudwatch get-metric-statistics \
    --namespace AWS/NetworkELB \
    --metric-name ActiveFlowCount \
    --dimensions Name=LoadBalancer,Value=$NLB_NAME \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Maximum \
    --query 'Datapoints[*].[Timestamp,Maximum]' \
    --output table

echo ""
echo "7. SECURITY VALIDATION"

# Test security group rules
echo "Testing security group access controls..."

# Test allowed ports
for port in 80 8080; do
    echo "Testing port $port access:"
    if timeout 5 bash -c "</dev/tcp/$NLB_DNS/$port"; then
        echo "✅ Port $port is accessible"
    else
        echo "❌ Port $port is not accessible"
    fi
done

# Test blocked ports
echo "Testing blocked port 22 (should fail):"
if timeout 3 bash -c "</dev/tcp/$NLB_DNS/22" 2>/dev/null; then
    echo "⚠️  Port 22 is unexpectedly accessible"
else
    echo "✅ Port 22 is properly blocked"
fi

echo ""
echo "8. DNS AND HYBRID CONNECTIVITY VALIDATION"

# Test internal DNS resolution
echo "Testing internal DNS resolution..."
if command -v dig >/dev/null 2>&1; then
    dig @169.254.169.253 app.hybrid.internal
else
    nslookup app.hybrid.internal 169.254.169.253
fi

echo ""
echo "9. GENERATE PERFORMANCE REPORT"

# Create performance summary
cat << REPORT > nlb-performance-report.txt
NLB Performance Test Report
===========================
Generated: $(date)

NLB Configuration:
- ARN: $NLB_ARN
- DNS: $NLB_DNS
- Cross-zone LB: Enabled
- Access Logs: Enabled

Target Groups:
- EC2 Targets: $TG_EC2_ARN
- IP Targets: $TG_IP_ARN

Test Results:
- Basic connectivity: $(curl -s -o /dev/null -w "%{http_code}" http://$NLB_DNS/ || echo "Failed")
- Response time: $(curl -w "%{time_total}" -s -o /dev/null http://$NLB_DNS/)s
- Healthy targets: $(aws elbv2 describe-target-health --target-group-arn $TG_EC2_ARN --query 'TargetHealthDescriptions[?TargetHealth.State==`healthy`] | length(@)' --output text)

Recommendations:
- Monitor target health regularly
- Set up automated scaling policies
- Implement proper health check endpoints
- Configure appropriate timeout values
- Monitor CloudWatch metrics for performance trends

REPORT

echo "✅ Performance report generated: nlb-performance-report.txt"

echo ""
echo "NLB TESTING COMPLETED"
echo "Check nlb-performance-report.txt for detailed results"

EOF

chmod +x test-nlb-performance.sh
./test-nlb-performance.sh
```

## NLB Best Practices for Hybrid Architectures

### Design Principles
- **Static IP Requirements**: Use NLB when fixed IP addresses are required
- **High Performance**: Leverage NLB's ultra-high performance capabilities
- **Source IP Preservation**: Maintain client IP addresses for security and logging
- **Cross-Zone Distribution**: Enable for optimal target utilization

### Security Considerations
- **Security Groups**: Configure appropriate ingress rules for NLB subnets
- **SSL/TLS Termination**: Implement certificate management and cipher policies
- **VPC Flow Logs**: Enable for traffic analysis and security monitoring
- **Access Logging**: Monitor and analyze access patterns

### Performance Optimization
- **Health Check Tuning**: Optimize interval and threshold settings
- **Connection Draining**: Configure appropriate timeout values
- **Target Distribution**: Use cross-zone load balancing for even distribution
- **Auto Scaling Integration**: Implement dynamic scaling based on metrics

### Monitoring and Alerting
- **CloudWatch Metrics**: Monitor connection counts, response times, and target health
- **Custom Dashboards**: Create comprehensive monitoring views
- **Automated Alerts**: Set up proactive alerting for health issues
- **Performance Baselines**: Establish baseline metrics for comparison

## Key Takeaways
- NLB provides Layer 4 load balancing with ultra-high performance
- Static IP support simplifies hybrid connectivity and firewall rules
- Cross-zone load balancing optimizes target utilization across AZs
- Health checks and connection draining ensure high availability
- Integration with Auto Scaling enables dynamic capacity management
- Comprehensive monitoring ensures optimal performance and availability

## Additional Resources
- [AWS Network Load Balancer User Guide](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/)
- [NLB Best Practices](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/network-load-balancer-best-practices.html)
- [ELB Monitoring Guide](https://docs.aws.amazon.com/elasticloadbalancing/latest/network/load-balancer-monitoring.html)

## Exam Tips
- NLB operates at Layer 4 (transport layer)
- Supports static IP addresses (one per AZ)
- Preserves source IP addresses by default
- Cross-zone load balancing incurs data transfer charges
- Supports TCP, UDP, and TLS protocols
- Health checks can be TCP, HTTP, or HTTPS
- Integration with Auto Scaling provides dynamic scaling
- Flow hash algorithm provides connection stickiness
- Maximum targets per NLB: 1000 per Availability Zone