# Topic 21: AWS Global Accelerator for Hybrid Performance

## Prerequisites
- Completed Topics 11-20 (Hybrid Networking Foundation)
- Understanding of global network performance concepts
- Knowledge of anycast routing and edge locations
- Familiarity with load balancing and traffic optimization

## Learning Objectives
By the end of this topic, you will be able to:
- Design Global Accelerator architectures for improved performance
- Configure endpoint groups and traffic dials for optimal routing
- Implement health checks and failover mechanisms
- Monitor and optimize global application performance

## Theory

### AWS Global Accelerator Overview

#### Definition
AWS Global Accelerator is a networking service that improves the performance of applications by routing traffic through AWS's global network infrastructure using anycast IP addresses.

#### Key Characteristics
- **Anycast IP Addresses**: Two static IP addresses that route to optimal AWS edge locations
- **Global Network**: Traffic routed through AWS backbone for improved performance
- **Intelligent Routing**: Dynamic routing based on network conditions and health
- **Protocol Support**: TCP and UDP traffic optimization
- **DDoS Protection**: Built-in AWS Shield Standard protection

### Global Accelerator Architecture

#### Network Flow
```
Client → AWS Edge Location → AWS Global Network → Target Endpoints
   ↓           ↓                    ↓                ↓
Internet → Anycast IPs → Optimized Path → ALB/NLB/EC2/EIP
```

#### Core Components
- **Accelerator**: Top-level resource with anycast IP addresses
- **Listener**: Processes inbound connections on specified ports/protocols
- **Endpoint Groups**: Regional groupings of endpoints with traffic controls
- **Endpoints**: Target resources (ALB, NLB, EC2 instances, Elastic IPs)

### Performance Benefits

#### Latency Reduction
- **Edge Proximity**: Traffic enters AWS network at nearest edge location
- **Optimized Routing**: AWS backbone provides better paths than internet
- **Congestion Avoidance**: Bypass internet congestion points
- **Typical Improvement**: 60% reduction in latency for global users

#### Availability Improvement
- **Instant Failover**: Sub-minute failover to healthy endpoints
- **Health Monitoring**: Continuous endpoint health assessment
- **Traffic Distribution**: Intelligent load distribution across regions
- **DDoS Mitigation**: Protection against network-layer attacks

## Lab Exercise: Global Accelerator Implementation

### Lab Duration: 300 minutes

### Step 1: Design Global Accelerator Architecture
**Objective**: Plan global performance optimization strategy

**Architecture Planning**:
```bash
# Create Global Accelerator planning assessment
cat << 'EOF' > global-accelerator-planning.sh
#!/bin/bash

echo "=== AWS Global Accelerator Architecture Planning ==="
echo ""

echo "1. APPLICATION PERFORMANCE REQUIREMENTS"
echo "   Application type: Web/API/Gaming/IoT/Media"
echo "   Geographic user distribution: _______"
echo "   Latency sensitivity: High/Medium/Low"
echo "   Throughput requirements: _______ Mbps"
echo "   Acceptable latency: _______ ms"
echo ""

echo "2. ENDPOINT ARCHITECTURE"
echo "   Primary region: _______"
echo "   Secondary regions: _______"
echo "   Endpoint types: ALB/NLB/EC2/EIP"
echo "   Multi-region deployment: Yes/No"
echo "   Auto-scaling enabled: Yes/No"
echo ""

echo "3. TRAFFIC PATTERNS"
echo "   Peak traffic times: _______"
echo "   Regional traffic distribution: _______"
echo "   Protocol requirements: TCP/UDP/Both"
echo "   Port requirements: _______"
echo "   Session persistence needed: Yes/No"
echo ""

echo "4. AVAILABILITY REQUIREMENTS"
echo "   RTO (Recovery Time Objective): _______ minutes"
echo "   RPO (Recovery Point Objective): _______ minutes"
echo "   Failover strategy: Automatic/Manual"
echo "   Health check frequency: _______ seconds"
echo "   Minimum healthy endpoints: _______"
echo ""

echo "5. MONITORING AND OPTIMIZATION"
echo "   Performance baseline: _______ ms latency"
echo "   Monitoring requirements: Real-time/Periodic"
echo "   Alerting thresholds: _______"
echo "   Cost optimization priority: High/Medium/Low"
echo "   Performance reporting: Yes/No"
echo ""

echo "6. SECURITY CONSIDERATIONS"
echo "   DDoS protection: Standard/Advanced"
echo "   IP allowlisting: Required/Optional"
echo "   Geographic restrictions: Yes/No"
echo "   SSL termination: Edge/Origin"
echo "   Certificate management: ACM/External"

EOF

chmod +x global-accelerator-planning.sh
./global-accelerator-planning.sh
```

### Step 2: Set Up Multi-Region Infrastructure
**Objective**: Deploy application infrastructure across multiple regions

**Multi-Region Setup**:
```bash
# Create multi-region infrastructure for Global Accelerator
cat << 'EOF' > setup-multiregion-infrastructure.sh
#!/bin/bash

echo "=== Setting Up Multi-Region Infrastructure ==="
echo ""

# Define regions for deployment
PRIMARY_REGION="us-east-1"
SECONDARY_REGION="us-west-2"
TERTIARY_REGION="eu-west-1"

REGIONS=($PRIMARY_REGION $SECONDARY_REGION $TERTIARY_REGION)

echo "1. CREATING INFRASTRUCTURE IN MULTIPLE REGIONS"
echo "Primary Region: $PRIMARY_REGION"
echo "Secondary Region: $SECONDARY_REGION"
echo "Tertiary Region: $TERTIARY_REGION"
echo ""

# Function to create infrastructure in a region
create_regional_infrastructure() {
    local region=$1
    local region_name=$2
    
    echo "Creating infrastructure in $region ($region_name)..."
    
    # Create VPC
    VPC_ID=$(aws ec2 create-vpc \
        --cidr-block 10.$((RANDOM % 100)).0.0/16 \
        --region $region \
        --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=GA-VPC-$region_name}]" \
        --query 'Vpc.VpcId' \
        --output text)
    
    echo "✅ VPC created in $region: $VPC_ID"
    
    # Enable DNS support
    aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support --region $region
    aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames --region $region
    
    # Get AZs for the region
    AZS=($(aws ec2 describe-availability-zones --region $region --query 'AvailabilityZones[0:2].ZoneName' --output text))
    
    # Create subnets
    SUBNET_IDS=()
    for i in "${!AZS[@]}"; do
        SUBNET_ID=$(aws ec2 create-subnet \
            --vpc-id $VPC_ID \
            --cidr-block 10.$((RANDOM % 100)).$((i+1)).0/24 \
            --availability-zone ${AZS[$i]} \
            --region $region \
            --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=GA-Subnet-$region_name-${AZS[$i]}}]" \
            --query 'Subnet.SubnetId' \
            --output text)
        
        SUBNET_IDS+=($SUBNET_ID)
        
        # Enable auto-assign public IP
        aws ec2 modify-subnet-attribute \
            --subnet-id $SUBNET_ID \
            --map-public-ip-on-launch \
            --region $region
    done
    
    echo "✅ Subnets created in $region: ${SUBNET_IDS[@]}"
    
    # Create Internet Gateway
    IGW_ID=$(aws ec2 create-internet-gateway \
        --region $region \
        --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=GA-IGW-$region_name}]" \
        --query 'InternetGateway.InternetGatewayId' \
        --output text)
    
    # Attach IGW
    aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $VPC_ID --region $region
    
    # Create route table
    RT_ID=$(aws ec2 create-route-table \
        --vpc-id $VPC_ID \
        --region $region \
        --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=GA-RT-$region_name}]" \
        --query 'RouteTable.RouteTableId' \
        --output text)
    
    # Add route to IGW
    aws ec2 create-route --route-table-id $RT_ID --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID --region $region
    
    # Associate subnets with route table
    for subnet_id in "${SUBNET_IDS[@]}"; do
        aws ec2 associate-route-table --route-table-id $RT_ID --subnet-id $subnet_id --region $region
    done
    
    echo "✅ Networking configured in $region"
    
    # Create security group
    SG_ID=$(aws ec2 create-security-group \
        --group-name GA-Web-SG-$region_name \
        --description "Security group for Global Accelerator web servers" \
        --vpc-id $VPC_ID \
        --region $region \
        --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=GA-Web-SG-$region_name}]" \
        --query 'GroupId' \
        --output text)
    
    # Configure security group rules
    aws ec2 authorize-security-group-ingress \
        --group-id $SG_ID \
        --protocol tcp \
        --port 80 \
        --cidr 0.0.0.0/0 \
        --region $region
    
    aws ec2 authorize-security-group-ingress \
        --group-id $SG_ID \
        --protocol tcp \
        --port 443 \
        --cidr 0.0.0.0/0 \
        --region $region
    
    aws ec2 authorize-security-group-ingress \
        --group-id $SG_ID \
        --protocol tcp \
        --port 22 \
        --cidr 0.0.0.0/0 \
        --region $region
    
    echo "✅ Security group configured in $region: $SG_ID"
    
    # Store configuration
    eval "export ${region_name^^}_VPC_ID=$VPC_ID"
    eval "export ${region_name^^}_SUBNET_IDS=(${SUBNET_IDS[@]})"
    eval "export ${region_name^^}_SG_ID=$SG_ID"
    eval "export ${region_name^^}_IGW_ID=$IGW_ID"
}

# Create infrastructure in each region
create_regional_infrastructure $PRIMARY_REGION "primary"
create_regional_infrastructure $SECONDARY_REGION "secondary"
create_regional_infrastructure $TERTIARY_REGION "tertiary"

echo ""
echo "2. DEPLOYING APPLICATION LOAD BALANCERS"

# Function to create ALB in a region
create_regional_alb() {
    local region=$1
    local region_name=$2
    local vpc_var="${region_name^^}_VPC_ID"
    local subnet_var="${region_name^^}_SUBNET_IDS[@]"
    local sg_var="${region_name^^}_SG_ID"
    
    local vpc_id=${!vpc_var}
    local subnet_ids=(${!subnet_var})
    local sg_id=${!sg_var}
    
    echo "Creating ALB in $region ($region_name)..."
    
    # Create ALB
    ALB_ARN=$(aws elbv2 create-load-balancer \
        --name GA-ALB-$region_name \
        --subnets ${subnet_ids[@]} \
        --security-groups $sg_id \
        --region $region \
        --tags Key=Name,Value=GA-ALB-$region_name \
        --query 'LoadBalancers[0].LoadBalancerArn' \
        --output text)
    
    echo "✅ ALB created in $region: $ALB_ARN"
    
    # Create target group
    TG_ARN=$(aws elbv2 create-target-group \
        --name GA-TG-$region_name \
        --protocol HTTP \
        --port 80 \
        --vpc-id $vpc_id \
        --region $region \
        --health-check-path /health \
        --health-check-interval-seconds 10 \
        --healthy-threshold-count 2 \
        --unhealthy-threshold-count 3 \
        --tags Key=Name,Value=GA-TG-$region_name \
        --query 'TargetGroups[0].TargetGroupArn' \
        --output text)
    
    echo "✅ Target Group created in $region: $TG_ARN"
    
    # Create listener
    aws elbv2 create-listener \
        --load-balancer-arn $ALB_ARN \
        --protocol HTTP \
        --port 80 \
        --default-actions Type=forward,TargetGroupArn=$TG_ARN \
        --region $region
    
    echo "✅ Listener configured in $region"
    
    # Store ALB configuration
    eval "export ${region_name^^}_ALB_ARN=$ALB_ARN"
    eval "export ${region_name^^}_TG_ARN=$TG_ARN"
}

# Create ALBs in each region
create_regional_alb $PRIMARY_REGION "primary"
create_regional_alb $SECONDARY_REGION "secondary"  
create_regional_alb $TERTIARY_REGION "tertiary"

# Save configuration to file
cat << CONFIG > global-accelerator-config.env
# Primary Region Configuration
export PRIMARY_REGION="$PRIMARY_REGION"
export PRIMARY_VPC_ID="$PRIMARY_VPC_ID"
export PRIMARY_ALB_ARN="$PRIMARY_ALB_ARN"
export PRIMARY_TG_ARN="$PRIMARY_TG_ARN"

# Secondary Region Configuration
export SECONDARY_REGION="$SECONDARY_REGION"
export SECONDARY_VPC_ID="$SECONDARY_VPC_ID"
export SECONDARY_ALB_ARN="$SECONDARY_ALB_ARN"
export SECONDARY_TG_ARN="$SECONDARY_TG_ARN"

# Tertiary Region Configuration
export TERTIARY_REGION="$TERTIARY_REGION"
export TERTIARY_VPC_ID="$TERTIARY_VPC_ID"
export TERTIARY_ALB_ARN="$TERTIARY_ALB_ARN"
export TERTIARY_TG_ARN="$TERTIARY_TG_ARN"
CONFIG

echo ""
echo "MULTI-REGION INFRASTRUCTURE SETUP COMPLETE"
echo "Primary Region: $PRIMARY_REGION"
echo "Secondary Region: $SECONDARY_REGION"
echo "Tertiary Region: $TERTIARY_REGION"
echo ""
echo "Configuration saved to global-accelerator-config.env"

EOF

chmod +x setup-multiregion-infrastructure.sh
./setup-multiregion-infrastructure.sh
```

### Step 3: Create Global Accelerator
**Objective**: Deploy and configure Global Accelerator

**Global Accelerator Creation**:
```bash
# Create AWS Global Accelerator
cat << 'EOF' > create-global-accelerator.sh
#!/bin/bash

source global-accelerator-config.env

echo "=== Creating AWS Global Accelerator ==="
echo ""

GA_NAME="Production-Global-Accelerator"

echo "1. CREATING GLOBAL ACCELERATOR"
echo "Name: $GA_NAME"
echo "IP Address Type: IPv4"
echo "Enabled: true"
echo ""

# Create Global Accelerator
GA_ARN=$(aws globalaccelerator create-accelerator \
    --name $GA_NAME \
    --ip-address-type IPV4 \
    --enabled \
    --tags Key=Name,Value=$GA_NAME Key=Environment,Value=Production \
    --query 'Accelerator.AcceleratorArn' \
    --output text)

echo "✅ Global Accelerator created: $GA_ARN"

# Wait for accelerator to be deployed
echo "Waiting for Global Accelerator to be deployed (this may take several minutes)..."
aws globalaccelerator wait accelerator-deployed --accelerator-arn $GA_ARN

echo "✅ Global Accelerator is now deployed"

# Get accelerator details
GA_DETAILS=$(aws globalaccelerator describe-accelerator \
    --accelerator-arn $GA_ARN \
    --query 'Accelerator')

# Extract static IPs
STATIC_IPS=$(echo $GA_DETAILS | jq -r '.IpSets[0].IpAddresses[]' | tr '\n' ' ')

echo "Global Accelerator Static IPs: $STATIC_IPS"

echo ""
echo "2. CREATING LISTENERS"

# Create HTTP listener
HTTP_LISTENER_ARN=$(aws globalaccelerator create-listener \
    --accelerator-arn $GA_ARN \
    --protocol TCP \
    --port-ranges FromPort=80,ToPort=80 \
    --client-affinity NONE \
    --query 'Listener.ListenerArn' \
    --output text)

echo "✅ HTTP Listener created: $HTTP_LISTENER_ARN"

# Create HTTPS listener
HTTPS_LISTENER_ARN=$(aws globalaccelerator create-listener \
    --accelerator-arn $GA_ARN \
    --protocol TCP \
    --port-ranges FromPort=443,ToPort=443 \
    --client-affinity SOURCE_IP \
    --query 'Listener.ListenerArn' \
    --output text)

echo "✅ HTTPS Listener created: $HTTPS_LISTENER_ARN"

echo ""
echo "3. CREATING ENDPOINT GROUPS"

# Create endpoint group for primary region
PRIMARY_EG_ARN=$(aws globalaccelerator create-endpoint-group \
    --listener-arn $HTTP_LISTENER_ARN \
    --endpoint-group-region $PRIMARY_REGION \
    --traffic-dial-percentage 100 \
    --health-check-interval-seconds 10 \
    --health-check-path "/health" \
    --health-check-protocol HTTP \
    --health-check-port 80 \
    --threshold-count 3 \
    --endpoint-configurations EndpointId=$PRIMARY_ALB_ARN,Weight=100 \
    --query 'EndpointGroup.EndpointGroupArn' \
    --output text)

echo "✅ Primary endpoint group created: $PRIMARY_EG_ARN"

# Create endpoint group for secondary region
SECONDARY_EG_ARN=$(aws globalaccelerator create-endpoint-group \
    --listener-arn $HTTP_LISTENER_ARN \
    --endpoint-group-region $SECONDARY_REGION \
    --traffic-dial-percentage 50 \
    --health-check-interval-seconds 10 \
    --health-check-path "/health" \
    --health-check-protocol HTTP \
    --health-check-port 80 \
    --threshold-count 3 \
    --endpoint-configurations EndpointId=$SECONDARY_ALB_ARN,Weight=100 \
    --query 'EndpointGroup.EndpointGroupArn' \
    --output text)

echo "✅ Secondary endpoint group created: $SECONDARY_EG_ARN"

# Create endpoint group for tertiary region
TERTIARY_EG_ARN=$(aws globalaccelerator create-endpoint-group \
    --listener-arn $HTTP_LISTENER_ARN \
    --endpoint-group-region $TERTIARY_REGION \
    --traffic-dial-percentage 25 \
    --health-check-interval-seconds 10 \
    --health-check-path "/health" \
    --health-check-protocol HTTP \
    --health-check-port 80 \
    --threshold-count 3 \
    --endpoint-configurations EndpointId=$TERTIARY_ALB_ARN,Weight=100 \
    --query 'EndpointGroup.EndpointGroupArn' \
    --output text)

echo "✅ Tertiary endpoint group created: $TERTIARY_EG_ARN"

echo ""
echo "4. CONFIGURING HTTPS ENDPOINT GROUPS"

# Repeat for HTTPS listener with same configuration
PRIMARY_HTTPS_EG_ARN=$(aws globalaccelerator create-endpoint-group \
    --listener-arn $HTTPS_LISTENER_ARN \
    --endpoint-group-region $PRIMARY_REGION \
    --traffic-dial-percentage 100 \
    --health-check-interval-seconds 10 \
    --health-check-path "/health" \
    --health-check-protocol HTTP \
    --health-check-port 80 \
    --threshold-count 3 \
    --endpoint-configurations EndpointId=$PRIMARY_ALB_ARN,Weight=100 \
    --query 'EndpointGroup.EndpointGroupArn' \
    --output text)

SECONDARY_HTTPS_EG_ARN=$(aws globalaccelerator create-endpoint-group \
    --listener-arn $HTTPS_LISTENER_ARN \
    --endpoint-group-region $SECONDARY_REGION \
    --traffic-dial-percentage 50 \
    --health-check-interval-seconds 10 \
    --health-check-path "/health" \
    --health-check-protocol HTTP \
    --health-check-port 80 \
    --threshold-count 3 \
    --endpoint-configurations EndpointId=$SECONDARY_ALB_ARN,Weight=100 \
    --query 'EndpointGroup.EndpointGroupArn' \
    --output text)

echo "✅ HTTPS endpoint groups created"

# Update configuration file
cat << CONFIG >> global-accelerator-config.env

# Global Accelerator Configuration
export GA_ARN="$GA_ARN"
export GA_STATIC_IPS="$STATIC_IPS"
export HTTP_LISTENER_ARN="$HTTP_LISTENER_ARN"
export HTTPS_LISTENER_ARN="$HTTPS_LISTENER_ARN"
export PRIMARY_EG_ARN="$PRIMARY_EG_ARN"
export SECONDARY_EG_ARN="$SECONDARY_EG_ARN"
export TERTIARY_EG_ARN="$TERTIARY_EG_ARN"
CONFIG

echo ""
echo "GLOBAL ACCELERATOR CONFIGURATION COMPLETE"
echo ""
echo "Global Accelerator Details:"
echo "- ARN: $GA_ARN"
echo "- Static IPs: $STATIC_IPS"
echo "- HTTP Listener: $HTTP_LISTENER_ARN"
echo "- HTTPS Listener: $HTTPS_LISTENER_ARN"
echo ""
echo "Endpoint Groups:"
echo "- Primary ($PRIMARY_REGION): 100% traffic"
echo "- Secondary ($SECONDARY_REGION): 50% traffic"
echo "- Tertiary ($TERTIARY_REGION): 25% traffic"
echo ""
echo "Configuration updated in global-accelerator-config.env"

EOF

chmod +x create-global-accelerator.sh
./create-global-accelerator.sh
```

### Step 4: Implement Traffic Management
**Objective**: Configure advanced traffic routing and optimization

**Traffic Management Configuration**:
```bash
# Configure traffic management for Global Accelerator
cat << 'EOF' > configure-traffic-management.sh
#!/bin/bash

source global-accelerator-config.env

echo "=== Configuring Global Accelerator Traffic Management ==="
echo ""

echo "1. IMPLEMENTING BLUE/GREEN DEPLOYMENT STRATEGY"

# Function to adjust traffic dial for blue/green deployments
adjust_traffic_dial() {
    local endpoint_group_arn=$1
    local traffic_percentage=$2
    local deployment_type=$3
    
    echo "Adjusting traffic dial for $deployment_type deployment..."
    echo "Endpoint Group: $endpoint_group_arn"
    echo "Traffic Percentage: $traffic_percentage%"
    
    aws globalaccelerator update-endpoint-group \
        --endpoint-group-arn $endpoint_group_arn \
        --traffic-dial-percentage $traffic_percentage
    
    echo "✅ Traffic dial updated to $traffic_percentage%"
}

# Blue/Green deployment example
echo "Blue/Green Deployment Scenario:"
echo "- Blue (Current): Primary region at 100%"
echo "- Green (New): Secondary region starting at 0%"
echo ""

# Start green deployment with 0% traffic
adjust_traffic_dial $SECONDARY_EG_ARN 0 "Green (Initial)"

# Gradually increase green traffic
for percentage in 10 25 50 75 100; do
    echo "Increasing green deployment to $percentage%..."
    adjust_traffic_dial $SECONDARY_EG_ARN $percentage "Green (Rollout)"
    
    # In real scenario, you would monitor metrics here
    echo "Monitor metrics for 5 minutes before next increase"
    echo "Check error rates, latency, and business metrics"
    sleep 2  # Simulated monitoring time
done

# Complete blue/green switchover
adjust_traffic_dial $PRIMARY_EG_ARN 0 "Blue (Deprecated)"
echo "✅ Blue/Green deployment complete - traffic switched to green"

echo ""
echo "2. IMPLEMENTING CANARY DEPLOYMENT STRATEGY"

# Canary deployment with multiple regions
echo "Canary Deployment Strategy:"
echo "- Primary: 90% traffic (stable)"
echo "- Canary: 10% traffic (new version)"
echo ""

# Set up canary deployment
adjust_traffic_dial $PRIMARY_EG_ARN 90 "Primary (Stable)"
adjust_traffic_dial $SECONDARY_EG_ARN 10 "Canary (New Version)"
adjust_traffic_dial $TERTIARY_EG_ARN 0 "Tertiary (Disabled)"

echo "✅ Canary deployment configured"

echo ""
echo "3. CONFIGURING ENDPOINT WEIGHTS"

# Update endpoint weights within endpoint groups
echo "Configuring endpoint weights for load distribution..."

# Primary region - single endpoint at full weight
aws globalaccelerator update-endpoint-group \
    --endpoint-group-arn $PRIMARY_EG_ARN \
    --endpoint-configurations EndpointId=$PRIMARY_ALB_ARN,Weight=255

# Secondary region - could have multiple endpoints with different weights
aws globalaccelerator update-endpoint-group \
    --endpoint-group-arn $SECONDARY_EG_ARN \
    --endpoint-configurations EndpointId=$SECONDARY_ALB_ARN,Weight=255

echo "✅ Endpoint weights configured"

echo ""
echo "4. IMPLEMENTING HEALTH CHECK OPTIMIZATION"

# Update health check settings for optimal performance
update_health_checks() {
    local endpoint_group_arn=$1
    local region_name=$2
    
    echo "Optimizing health checks for $region_name..."
    
    aws globalaccelerator update-endpoint-group \
        --endpoint-group-arn $endpoint_group_arn \
        --health-check-interval-seconds 10 \
        --health-check-path "/health" \
        --health-check-protocol HTTP \
        --health-check-port 80 \
        --threshold-count 2
    
    echo "✅ Health checks optimized for $region_name"
}

update_health_checks $PRIMARY_EG_ARN "Primary"
update_health_checks $SECONDARY_EG_ARN "Secondary"
update_health_checks $TERTIARY_EG_ARN "Tertiary"

echo ""
echo "5. CREATING TRAFFIC MANAGEMENT SCRIPTS"

# Create script for emergency traffic rerouting
cat << 'EMERGENCY_SCRIPT' > emergency-traffic-reroute.sh
#!/bin/bash

# Emergency traffic rerouting script

ENDPOINT_GROUP=$1
EMERGENCY_PERCENTAGE=${2:-100}

if [ -z "$ENDPOINT_GROUP" ]; then
    echo "Usage: $0 <endpoint-group-arn> [percentage]"
    echo "Example: $0 arn:aws:globalaccelerator::123456789012:accelerator/abc123/listener/def456/endpoint-group/ghi789 100"
    exit 1
fi

echo "EMERGENCY TRAFFIC REROUTING"
echo "Endpoint Group: $ENDPOINT_GROUP"
echo "Emergency Traffic: $EMERGENCY_PERCENTAGE%"

# Reroute traffic immediately
aws globalaccelerator update-endpoint-group \
    --endpoint-group-arn $ENDPOINT_GROUP \
    --traffic-dial-percentage $EMERGENCY_PERCENTAGE

echo "✅ Emergency traffic rerouting completed"
echo "Monitor application performance and adjust as needed"

EMERGENCY_SCRIPT

chmod +x emergency-traffic-reroute.sh

# Create script for gradual traffic shifting
cat << 'TRAFFIC_SHIFT_SCRIPT' > gradual-traffic-shift.sh
#!/bin/bash

# Gradual traffic shifting script for safe deployments

SOURCE_EG=$1
TARGET_EG=$2
SHIFT_INCREMENT=${3:-10}
WAIT_TIME=${4:-300}  # 5 minutes between shifts

if [ -z "$SOURCE_EG" ] || [ -z "$TARGET_EG" ]; then
    echo "Usage: $0 <source-endpoint-group> <target-endpoint-group> [increment] [wait-time]"
    exit 1
fi

echo "GRADUAL TRAFFIC SHIFTING"
echo "Source: $SOURCE_EG"
echo "Target: $TARGET_EG"
echo "Increment: $SHIFT_INCREMENT%"
echo "Wait Time: $WAIT_TIME seconds"

# Get current traffic percentages
SOURCE_CURRENT=$(aws globalaccelerator describe-endpoint-group \
    --endpoint-group-arn $SOURCE_EG \
    --query 'EndpointGroup.TrafficDialPercentage' \
    --output text)

TARGET_CURRENT=$(aws globalaccelerator describe-endpoint-group \
    --endpoint-group-arn $TARGET_EG \
    --query 'EndpointGroup.TrafficDialPercentage' \
    --output text)

echo "Current traffic - Source: $SOURCE_CURRENT%, Target: $TARGET_CURRENT%"

# Perform gradual shift
for ((i=$SHIFT_INCREMENT; i<=100; i+=$SHIFT_INCREMENT)); do
    SOURCE_NEW=$((100-i))
    TARGET_NEW=$i
    
    echo "Shifting traffic - Source: $SOURCE_NEW%, Target: $TARGET_NEW%"
    
    # Update source endpoint group
    aws globalaccelerator update-endpoint-group \
        --endpoint-group-arn $SOURCE_EG \
        --traffic-dial-percentage $SOURCE_NEW
    
    # Update target endpoint group
    aws globalaccelerator update-endpoint-group \
        --endpoint-group-arn $TARGET_EG \
        --traffic-dial-percentage $TARGET_NEW
    
    echo "✅ Traffic shifted. Waiting $WAIT_TIME seconds for monitoring..."
    sleep $WAIT_TIME
done

echo "✅ Gradual traffic shifting completed"

TRAFFIC_SHIFT_SCRIPT

chmod +x gradual-traffic-shift.sh

echo ""
echo "TRAFFIC MANAGEMENT CONFIGURATION COMPLETE"
echo ""
echo "Available Traffic Management Tools:"
echo "1. emergency-traffic-reroute.sh - Immediate traffic rerouting"
echo "2. gradual-traffic-shift.sh - Safe gradual traffic shifting"
echo ""
echo "Current Traffic Distribution:"
echo "- Primary Region: 90% (Blue/Stable)"
echo "- Secondary Region: 10% (Green/Canary)"
echo "- Tertiary Region: 0% (Disabled)"
echo ""
echo "Health Check Optimization:"
echo "- Interval: 10 seconds"
echo "- Threshold: 2 consecutive checks"
echo "- Protocol: HTTP"
echo "- Path: /health"

EOF

chmod +x configure-traffic-management.sh
./configure-traffic-management.sh
```

### Step 5: Implement Performance Monitoring
**Objective**: Set up comprehensive monitoring and alerting

**Performance Monitoring Setup**:
```bash
# Set up Global Accelerator performance monitoring
cat << 'EOF' > setup-ga-monitoring.sh
#!/bin/bash

source global-accelerator-config.env

echo "=== Setting Up Global Accelerator Performance Monitoring ==="
echo ""

echo "1. CREATING CLOUDWATCH DASHBOARD"

# Create comprehensive CloudWatch dashboard
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
                    [ "AWS/GlobalAccelerator", "NewFlowCount", "Accelerator", "'$(echo $GA_ARN | cut -d'/' -f2)'" ],
                    [ ".", "ActiveFlowCount", ".", "." ]
                ],
                "period": 300,
                "stat": "Sum",
                "region": "us-west-2",
                "title": "Global Accelerator Flow Metrics"
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
                    [ "AWS/GlobalAccelerator", "ProcessedBytesIn", "Accelerator", "'$(echo $GA_ARN | cut -d'/' -f2)'" ],
                    [ ".", "ProcessedBytesOut", ".", "." ]
                ],
                "period": 300,
                "stat": "Sum",
                "region": "us-west-2",
                "title": "Data Processing Metrics"
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
                    [ "AWS/GlobalAccelerator", "AwsApiLatency", "Accelerator", "'$(echo $GA_ARN | cut -d'/' -f2)'", "EndpointGroup", "'$(echo $PRIMARY_EG_ARN | cut -d'/' -f2)'" ],
                    [ "...", "'$(echo $SECONDARY_EG_ARN | cut -d'/' -f2)'" ],
                    [ "...", "'$(echo $TERTIARY_EG_ARN | cut -d'/' -f2)'" ]
                ],
                "period": 300,
                "stat": "Average",
                "region": "us-west-2",
                "title": "Endpoint Group Latency"
            }
        }
    ]
}'

aws cloudwatch put-dashboard \
    --dashboard-name "GlobalAccelerator-Performance" \
    --dashboard-body "$DASHBOARD_BODY" \
    --region us-west-2

echo "✅ CloudWatch dashboard created: GlobalAccelerator-Performance"

echo ""
echo "2. CREATING CLOUDWATCH ALARMS"

GA_NAME=$(echo $GA_ARN | cut -d'/' -f2)

# High latency alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "GA-HighLatency" \
    --alarm-description "Alert when Global Accelerator latency is high" \
    --metric-name AwsApiLatency \
    --namespace AWS/GlobalAccelerator \
    --statistic Average \
    --period 300 \
    --threshold 200 \
    --comparison-operator GreaterThanThreshold \
    --dimensions Name=Accelerator,Value=$GA_NAME \
    --evaluation-periods 2 \
    --region us-west-2

# Low flow count alarm (potential issue)
aws cloudwatch put-metric-alarm \
    --alarm-name "GA-LowFlowCount" \
    --alarm-description "Alert when flow count drops significantly" \
    --metric-name ActiveFlowCount \
    --namespace AWS/GlobalAccelerator \
    --statistic Average \
    --period 300 \
    --threshold 10 \
    --comparison-operator LessThanThreshold \
    --dimensions Name=Accelerator,Value=$GA_NAME \
    --evaluation-periods 3 \
    --region us-west-2

echo "✅ CloudWatch alarms created"

echo ""
echo "3. SETTING UP FLOW LOGS (VPC FLOW LOGS FOR ENDPOINTS)"

# Enable VPC Flow Logs for endpoint VPCs to monitor traffic patterns
enable_vpc_flow_logs() {
    local vpc_id=$1
    local region=$2
    local region_name=$3
    
    echo "Enabling VPC Flow Logs for $region_name ($vpc_id)..."
    
    # Create CloudWatch log group
    aws logs create-log-group \
        --log-group-name "/aws/vpc/flowlogs/$region_name" \
        --region $region 2>/dev/null || true
    
    # Create IAM role for Flow Logs (simplified - use existing role in production)
    echo "VPC Flow Logs setup requires IAM role configuration"
    echo "Please configure VPC Flow Logs manually or use existing IAM role"
}

enable_vpc_flow_logs $PRIMARY_VPC_ID $PRIMARY_REGION "primary"
enable_vpc_flow_logs $SECONDARY_VPC_ID $SECONDARY_REGION "secondary"
enable_vpc_flow_logs $TERTIARY_VPC_ID $TERTIARY_REGION "tertiary"

echo ""
echo "4. CREATING PERFORMANCE TESTING SCRIPT"

cat << 'PERF_TEST' > ga-performance-test.sh
#!/bin/bash

# Global Accelerator performance testing script

GA_STATIC_IPS=($GA_STATIC_IPS)
TEST_DURATION=${1:-60}  # Default 60 seconds
CONCURRENT_CONNECTIONS=${2:-10}

echo "=== Global Accelerator Performance Test ==="
echo "Static IPs: ${GA_STATIC_IPS[@]}"
echo "Test Duration: $TEST_DURATION seconds"
echo "Concurrent Connections: $CONCURRENT_CONNECTIONS"
echo ""

# Function to test latency to a specific IP
test_latency() {
    local ip=$1
    local samples=10
    
    echo "Testing latency to $ip..."
    
    # Ping test
    if command -v ping >/dev/null 2>&1; then
        echo "Ping results:"
        ping -c $samples $ip | tail -1
    fi
    
    # HTTP response time test
    if command -v curl >/dev/null 2>&1; then
        echo "HTTP response time:"
        for i in $(seq 1 5); do
            RESPONSE_TIME=$(curl -w "%{time_total}" -s -o /dev/null http://$ip/ 2>/dev/null)
            echo "  Request $i: ${RESPONSE_TIME}s"
        done
    fi
    
    echo ""
}

# Test each static IP
for ip in "${GA_STATIC_IPS[@]}"; do
    test_latency $ip
done

# Load testing with curl (if available)
if command -v curl >/dev/null 2>&1; then
    echo "Running load test..."
    
    PRIMARY_IP=${GA_STATIC_IPS[0]}
    
    echo "Testing concurrent connections to $PRIMARY_IP"
    
    # Run concurrent curl requests
    for i in $(seq 1 $CONCURRENT_CONNECTIONS); do
        {
            curl -s -w "Connection $i: %{time_total}s\n" -o /dev/null http://$PRIMARY_IP/
        } &
    done
    
    wait
    echo "✅ Load test completed"
fi

echo ""
echo "Performance test completed"
echo "Check CloudWatch metrics for detailed analysis"

PERF_TEST

chmod +x ga-performance-test.sh

echo ""
echo "5. CREATING REGIONAL PERFORMANCE COMPARISON"

cat << 'COMPARISON_SCRIPT' > regional-performance-comparison.sh
#!/bin/bash

# Compare performance across regions

source global-accelerator-config.env

echo "=== Regional Performance Comparison ==="
echo ""

# Get ALB DNS names from each region
PRIMARY_ALB_DNS=$(aws elbv2 describe-load-balancers \
    --load-balancer-arns $PRIMARY_ALB_ARN \
    --region $PRIMARY_REGION \
    --query 'LoadBalancers[0].DNSName' \
    --output text)

SECONDARY_ALB_DNS=$(aws elbv2 describe-load-balancers \
    --load-balancer-arns $SECONDARY_ALB_ARN \
    --region $SECONDARY_REGION \
    --query 'LoadBalancers[0].DNSName' \
    --output text)

TERTIARY_ALB_DNS=$(aws elbv2 describe-load-balancers \
    --load-balancer-arns $TERTIARY_ALB_ARN \
    --region $TERTIARY_REGION \
    --query 'LoadBalancers[0].DNSName' \
    --output text)

# Test direct access to each region
test_direct_access() {
    local alb_dns=$1
    local region=$2
    
    echo "Testing direct access to $region ($alb_dns)..."
    
    if command -v curl >/dev/null 2>&1; then
        for i in {1..5}; do
            RESPONSE_TIME=$(curl -w "%{time_total}" -s -o /dev/null http://$alb_dns/ 2>/dev/null)
            echo "  Direct to $region: ${RESPONSE_TIME}s"
        done
    fi
    echo ""
}

echo "Direct Regional Access Performance:"
test_direct_access $PRIMARY_ALB_DNS $PRIMARY_REGION
test_direct_access $SECONDARY_ALB_DNS $SECONDARY_REGION
test_direct_access $TERTIARY_ALB_DNS $TERTIARY_REGION

echo "Global Accelerator Performance:"
test_direct_access ${GA_STATIC_IPS[0]} "Global-Accelerator"

echo "Performance comparison completed"
echo "Global Accelerator should show improved and consistent performance"

COMPARISON_SCRIPT

chmod +x regional-performance-comparison.sh

echo ""
echo "PERFORMANCE MONITORING SETUP COMPLETE"
echo ""
echo "Monitoring Components:"
echo "- CloudWatch Dashboard: GlobalAccelerator-Performance"
echo "- CloudWatch Alarms: GA-HighLatency, GA-LowFlowCount"
echo "- Performance Test Script: ga-performance-test.sh"
echo "- Regional Comparison Script: regional-performance-comparison.sh"
echo ""
echo "Usage Examples:"
echo "1. Run performance test: ./ga-performance-test.sh 120 20"
echo "2. Compare regional performance: ./regional-performance-comparison.sh"
echo "3. View dashboard: AWS Console → CloudWatch → Dashboards → GlobalAccelerator-Performance"
echo ""
echo "Static IPs for testing: $GA_STATIC_IPS"

EOF

chmod +x setup-ga-monitoring.sh
./setup-ga-monitoring.sh
```

## Global Accelerator Advanced Features

### Traffic Management
- **Traffic Dials**: Control percentage of traffic to endpoint groups
- **Endpoint Weights**: Fine-tune load distribution within endpoint groups
- **Health Checks**: Automated endpoint health monitoring and failover
- **Blue/Green Deployments**: Safe deployment strategies with instant rollback

### Performance Optimization
- **Anycast IPs**: Static IP addresses that route to optimal edge locations
- **AWS Backbone**: Traffic routed through AWS global network infrastructure
- **TCP Optimization**: Congestion control and connection management
- **Geographic Routing**: Intelligent routing based on client location

### Security and Availability
- **DDoS Protection**: Built-in AWS Shield Standard protection
- **Health-based Routing**: Automatic traffic rerouting during outages
- **Multi-region Failover**: Sub-minute failover to healthy regions
- **Source IP Preservation**: Maintain original client IP addresses

## Use Cases for Global Accelerator

### Gaming Applications
- **Low Latency**: Critical for real-time gaming experiences
- **Global Reach**: Support for worldwide player base
- **UDP Support**: Optimized for gaming protocols
- **Consistent Performance**: Predictable latency and jitter

### IoT and Edge Computing
- **Device Connectivity**: Reliable connections for IoT devices
- **Edge Optimization**: Routing to nearest processing centers
- **Protocol Support**: TCP and UDP for various device types
- **Scale**: Support for millions of concurrent connections

### Financial Services
- **Trading Applications**: Ultra-low latency for trading systems
- **Global Markets**: Access to worldwide financial markets
- **Reliability**: High availability for mission-critical applications
- **Security**: DDoS protection and secure connectivity

### Media and Content Delivery
- **Live Streaming**: Optimized delivery for real-time content
- **Global Audience**: Worldwide content distribution
- **Large File Transfer**: Optimized for high-throughput applications
- **Failover**: Automatic routing during content delivery issues

## Best Practices

### Design Principles
- **Multi-region Architecture**: Deploy across multiple AWS regions
- **Health Check Optimization**: Configure appropriate health check parameters
- **Traffic Management**: Use traffic dials for safe deployments
- **Monitoring**: Implement comprehensive performance monitoring

### Performance Optimization
- **Endpoint Selection**: Choose optimal endpoint types and locations
- **Health Check Tuning**: Balance responsiveness with stability
- **Client Affinity**: Configure based on application requirements
- **Regular Testing**: Continuous performance testing and optimization

### Operational Excellence
- **Automated Deployments**: Use traffic dials for blue/green deployments
- **Monitoring and Alerting**: Proactive monitoring of performance metrics
- **Disaster Recovery**: Multi-region failover procedures
- **Cost Optimization**: Monitor usage and optimize traffic distribution

## Key Takeaways
- Global Accelerator improves application performance using AWS global network
- Anycast IPs provide static endpoints that route to optimal locations
- Traffic dials enable safe deployment strategies and traffic management
- Multi-region deployment provides high availability and disaster recovery
- Comprehensive monitoring ensures optimal performance and quick issue resolution
- Built-in DDoS protection enhances security for global applications

## Additional Resources
- [AWS Global Accelerator User Guide](https://docs.aws.amazon.com/global-accelerator/latest/dg/)
- [Global Accelerator Best Practices](https://docs.aws.amazon.com/global-accelerator/latest/dg/best-practices.html)
- [Performance Monitoring Guide](https://docs.aws.amazon.com/global-accelerator/latest/dg/cloudwatch-monitoring.html)

## Exam Tips
- Global Accelerator uses anycast IP addresses (2 static IPs per accelerator)
- Traffic dials control percentage of traffic to endpoint groups (0-100%)
- Endpoint weights control distribution within endpoint groups (0-255)
- Health checks support HTTP, HTTPS, and TCP protocols
- Client affinity can be NONE or SOURCE_IP
- Supports ALB, NLB, EC2 instances, and Elastic IP addresses as endpoints
- Built-in DDoS protection with AWS Shield Standard
- Billing based on accelerator running time and data transfer