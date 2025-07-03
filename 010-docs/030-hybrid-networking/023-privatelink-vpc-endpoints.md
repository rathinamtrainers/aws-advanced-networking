# Topic 23: AWS PrivateLink and VPC Endpoints

## Prerequisites
- Completed Topics 11-22 (Hybrid Networking Foundation)
- Understanding of VPC networking concepts and routing
- Knowledge of AWS service architectures and APIs
- Familiarity with DNS resolution and private networking

## Learning Objectives
By the end of this topic, you will be able to:
- Design PrivateLink architectures for secure service access
- Configure VPC endpoints for AWS services and custom applications
- Implement cross-account and cross-region PrivateLink connectivity
- Troubleshoot PrivateLink connectivity and DNS resolution issues

## Theory

### AWS PrivateLink Overview

#### Definition
AWS PrivateLink provides private connectivity between VPCs, supported AWS services, and on-premises networks without exposing traffic to the public internet.

#### Key Benefits
- **Private Connectivity**: Traffic never leaves the AWS network
- **Simplified Network Management**: No VPC peering or transit gateways required
- **Enhanced Security**: Reduced attack surface by avoiding internet routing
- **Scalable Architecture**: Supports thousands of consuming VPCs

### VPC Endpoints Types

#### Interface Endpoints (Powered by PrivateLink)
- **Technology**: Elastic Network Interfaces with private IPs
- **Supported Services**: 100+ AWS services
- **DNS Resolution**: Private DNS names resolve to endpoint IPs
- **Security**: Controlled by security groups and endpoint policies

#### Gateway Endpoints
- **Supported Services**: S3 and DynamoDB only
- **Technology**: Route table entries, not ENIs
- **Cost**: No hourly charges, only data transfer
- **Limitations**: VPC-local access only

### PrivateLink Architecture Patterns

#### Pattern 1: Service Provider Architecture
```
Service Provider VPC:
    Application Load Balancer ← VPC Endpoint Service ← Network Load Balancer
                                        ↑
Service Consumer VPC:                   |
    EC2 Instances → Interface Endpoint ←┘
```

#### Pattern 2: Cross-Account Service Sharing
```
Account A (Provider):
    Custom Application → NLB → VPC Endpoint Service
                                    ↓
Account B (Consumer):
    Applications → Interface Endpoint → PrivateLink Connection
```

#### Pattern 3: Hybrid Access Pattern
```
On-premises → Direct Connect/VPN → VPC → Interface Endpoints → AWS Services
                                    ↓
                            No Internet Gateway Required
```

### VPC Endpoint Service Configuration

#### Service Provider Setup
- **Network Load Balancer**: Required for custom services
- **VPC Endpoint Service**: Exposes NLB through PrivateLink
- **Service Names**: Auto-generated or custom domain names
- **Acceptance Settings**: Manual or automatic connection acceptance

#### Service Consumer Setup
- **Interface Endpoints**: Created in consuming VPC subnets
- **DNS Configuration**: Private DNS names for seamless integration
- **Security Groups**: Control access to endpoint network interfaces
- **Endpoint Policies**: JSON policies for fine-grained access control

## Lab Exercise: Comprehensive PrivateLink Implementation

### Lab Duration: 360 minutes

### Step 1: Design PrivateLink Architecture
**Objective**: Plan comprehensive PrivateLink deployment strategy

**Architecture Planning**:
```bash
# Create PrivateLink planning assessment
cat << 'EOF' > privatelink-planning.sh
#!/bin/bash

echo "=== AWS PrivateLink Architecture Planning ==="
echo ""

echo "1. SERVICE PROVIDER REQUIREMENTS"
echo "   Service type: Custom Application/AWS Service"
echo "   Service availability zones: _______"
echo "   Expected consumer VPCs: _______"
echo "   Cross-account access: Yes/No"
echo "   Multi-region access: Yes/No"
echo "   Load balancer requirements: NLB/ALB"
echo ""

echo "2. SERVICE CONSUMER REQUIREMENTS"
echo "   Number of consuming VPCs: _______"
echo "   Cross-account consumers: _______"
echo "   DNS integration requirements: Yes/No"
echo "   Private DNS resolution: Yes/No"
echo "   Custom domain names: Yes/No"
echo "   On-premises access: Yes/No"
echo ""

echo "3. AWS SERVICES ACCESS REQUIREMENTS"
echo "   Required AWS services: S3/DynamoDB/EC2/RDS/etc"
echo "   Gateway endpoints needed: S3/DynamoDB"
echo "   Interface endpoints needed: _______"
echo "   Service-specific configurations: _______"
echo "   Regional service access: Single/Multi-region"
echo ""

echo "4. SECURITY AND ACCESS CONTROL"
echo "   Endpoint policies required: Yes/No"
echo "   Security group restrictions: Yes/No"
echo "   Cross-account permissions: Yes/No"
echo "   Service-specific IAM policies: Yes/No"
echo "   Network ACL restrictions: Yes/No"
echo ""

echo "5. MONITORING AND COMPLIANCE"
echo "   VPC Flow Logs: Yes/No"
echo "   CloudTrail API logging: Yes/No"
echo "   Custom monitoring metrics: Yes/No"
echo "   Compliance requirements: SOC/PCI/HIPAA"
echo "   Cost optimization: Yes/No"
echo ""

echo "6. PERFORMANCE REQUIREMENTS"
echo "   Expected throughput: _______ Gbps"
echo "   Latency requirements: _______ ms"
echo "   Availability requirements: _______"
echo "   Disaster recovery: Yes/No"
echo "   Auto-scaling requirements: Yes/No"

EOF

chmod +x privatelink-planning.sh
./privatelink-planning.sh
```

### Step 2: Create VPC Infrastructure for PrivateLink
**Objective**: Set up provider and consumer VPC environments

**VPC Infrastructure Setup**:
```bash
# Create VPC infrastructure for PrivateLink lab
cat << 'EOF' > setup-privatelink-infrastructure.sh
#!/bin/bash

echo "=== Setting Up PrivateLink Infrastructure ==="
echo ""

echo "1. CREATING PROVIDER VPC"

# Create provider VPC
PROVIDER_VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.1.0.0/16 \
    --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=PrivateLink-Provider-VPC}]" \
    --query 'Vpc.VpcId' \
    --output text)

echo "✅ Provider VPC created: $PROVIDER_VPC_ID"

# Create provider subnets
PROVIDER_SUBNET_A_ID=$(aws ec2 create-subnet \
    --vpc-id $PROVIDER_VPC_ID \
    --cidr-block 10.1.1.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=PrivateLink-Provider-Subnet-A}]" \
    --query 'Subnet.SubnetId' \
    --output text)

PROVIDER_SUBNET_B_ID=$(aws ec2 create-subnet \
    --vpc-id $PROVIDER_VPC_ID \
    --cidr-block 10.1.2.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=PrivateLink-Provider-Subnet-B}]" \
    --query 'Subnet.SubnetId' \
    --output text)

echo "✅ Provider subnets created: $PROVIDER_SUBNET_A_ID, $PROVIDER_SUBNET_B_ID"

echo ""
echo "2. CREATING CONSUMER VPC"

# Create consumer VPC
CONSUMER_VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.2.0.0/16 \
    --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=PrivateLink-Consumer-VPC}]" \
    --query 'Vpc.VpcId' \
    --output text)

echo "✅ Consumer VPC created: $CONSUMER_VPC_ID"

# Create consumer subnets
CONSUMER_SUBNET_A_ID=$(aws ec2 create-subnet \
    --vpc-id $CONSUMER_VPC_ID \
    --cidr-block 10.2.1.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=PrivateLink-Consumer-Subnet-A}]" \
    --query 'Subnet.SubnetId' \
    --output text)

CONSUMER_SUBNET_B_ID=$(aws ec2 create-subnet \
    --vpc-id $CONSUMER_VPC_ID \
    --cidr-block 10.2.2.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=PrivateLink-Consumer-Subnet-B}]" \
    --query 'Subnet.SubnetId' \
    --output text)

echo "✅ Consumer subnets created: $CONSUMER_SUBNET_A_ID, $CONSUMER_SUBNET_B_ID"

echo ""
echo "3. CREATING SECURITY GROUPS"

# Provider security group
PROVIDER_SG_ID=$(aws ec2 create-security-group \
    --group-name PrivateLink-Provider-SG \
    --description "Security group for PrivateLink service provider" \
    --vpc-id $PROVIDER_VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=PrivateLink-Provider-SG}]" \
    --query 'GroupId' \
    --output text)

# Consumer security group
CONSUMER_SG_ID=$(aws ec2 create-security-group \
    --group-name PrivateLink-Consumer-SG \
    --description "Security group for PrivateLink service consumer" \
    --vpc-id $CONSUMER_VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=PrivateLink-Consumer-SG}]" \
    --query 'GroupId' \
    --output text)

echo "✅ Security groups created:"
echo "  Provider: $PROVIDER_SG_ID"
echo "  Consumer: $CONSUMER_SG_ID"

# Configure security group rules
aws ec2 authorize-security-group-ingress \
    --group-id $PROVIDER_SG_ID \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
    --group-id $PROVIDER_SG_ID \
    --protocol tcp \
    --port 443 \
    --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
    --group-id $CONSUMER_SG_ID \
    --protocol tcp \
    --port 80 \
    --cidr 10.2.0.0/16

aws ec2 authorize-security-group-ingress \
    --group-id $CONSUMER_SG_ID \
    --protocol tcp \
    --port 443 \
    --cidr 10.2.0.0/16

echo "✅ Security group rules configured"

# Save configuration
cat << CONFIG > privatelink-infrastructure-config.env
export PROVIDER_VPC_ID="$PROVIDER_VPC_ID"
export PROVIDER_SUBNET_A_ID="$PROVIDER_SUBNET_A_ID"
export PROVIDER_SUBNET_B_ID="$PROVIDER_SUBNET_B_ID"
export CONSUMER_VPC_ID="$CONSUMER_VPC_ID"
export CONSUMER_SUBNET_A_ID="$CONSUMER_SUBNET_A_ID"
export CONSUMER_SUBNET_B_ID="$CONSUMER_SUBNET_B_ID"
export PROVIDER_SG_ID="$PROVIDER_SG_ID"
export CONSUMER_SG_ID="$CONSUMER_SG_ID"
CONFIG

echo ""
echo "PrivateLink infrastructure setup complete"
echo "Configuration saved to privatelink-infrastructure-config.env"

EOF

chmod +x setup-privatelink-infrastructure.sh
./setup-privatelink-infrastructure.sh
```

### Step 3: Deploy Service Provider Infrastructure
**Objective**: Create NLB and VPC Endpoint Service for custom application

**Service Provider Setup**:
```bash
# Deploy service provider infrastructure
cat << 'EOF' > deploy-service-provider.sh
#!/bin/bash

source privatelink-infrastructure-config.env

echo "=== Deploying PrivateLink Service Provider ==="
echo ""

echo "1. LAUNCHING SERVICE PROVIDER INSTANCES"

# Get latest Amazon Linux 2 AMI
AMI_ID=$(aws ec2 describe-images \
    --owners amazon \
    --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
              "Name=state,Values=available" \
    --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
    --output text)

echo "Using AMI: $AMI_ID"

# Launch provider instances
PROVIDER_INSTANCE_A=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.micro \
    --subnet-id $PROVIDER_SUBNET_A_ID \
    --security-group-ids $PROVIDER_SG_ID \
    --associate-public-ip-address \
    --user-data '#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>PrivateLink Service Provider - Instance A</h1>" > /var/www/html/index.html
echo "<p>Timestamp: $(date)</p>" >> /var/www/html/index.html
echo "<p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>" >> /var/www/html/index.html' \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=PrivateLink-Provider-A}]" \
    --query 'Instances[0].InstanceId' \
    --output text)

PROVIDER_INSTANCE_B=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.micro \
    --subnet-id $PROVIDER_SUBNET_B_ID \
    --security-group-ids $PROVIDER_SG_ID \
    --associate-public-ip-address \
    --user-data '#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>PrivateLink Service Provider - Instance B</h1>" > /var/www/html/index.html
echo "<p>Timestamp: $(date)</p>" >> /var/www/html/index.html
echo "<p>Instance ID: $(curl -s http://169.254.169.254/latest/meta-data/instance-id)</p>" >> /var/www/html/index.html' \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=PrivateLink-Provider-B}]" \
    --query 'Instances[0].InstanceId' \
    --output text)

echo "✅ Provider instances launched:"
echo "  Instance A: $PROVIDER_INSTANCE_A"
echo "  Instance B: $PROVIDER_INSTANCE_B"

# Wait for instances to be running
echo "Waiting for instances to be running..."
aws ec2 wait instance-running --instance-ids $PROVIDER_INSTANCE_A $PROVIDER_INSTANCE_B
echo "✅ Instances are running"

echo ""
echo "2. CREATING NETWORK LOAD BALANCER"

# Create NLB
NLB_ARN=$(aws elbv2 create-load-balancer \
    --name PrivateLink-Service-NLB \
    --type network \
    --scheme internal \
    --ip-address-type ipv4 \
    --subnets $PROVIDER_SUBNET_A_ID $PROVIDER_SUBNET_B_ID \
    --tags Key=Name,Value=PrivateLink-Service-NLB \
    --query 'LoadBalancers[0].LoadBalancerArn' \
    --output text)

echo "✅ Network Load Balancer created: $NLB_ARN"

# Wait for NLB to be active
echo "Waiting for NLB to become active..."
aws elbv2 wait load-balancer-available --load-balancer-arns $NLB_ARN
echo "✅ NLB is active"

# Create target group
TG_ARN=$(aws elbv2 create-target-group \
    --name PrivateLink-Service-TG \
    --protocol TCP \
    --port 80 \
    --vpc-id $PROVIDER_VPC_ID \
    --target-type instance \
    --health-check-protocol TCP \
    --health-check-port 80 \
    --health-check-interval-seconds 30 \
    --healthy-threshold-count 2 \
    --unhealthy-threshold-count 2 \
    --tags Key=Name,Value=PrivateLink-Service-TG \
    --query 'TargetGroups[0].TargetGroupArn' \
    --output text)

echo "✅ Target group created: $TG_ARN"

# Register targets
aws elbv2 register-targets \
    --target-group-arn $TG_ARN \
    --targets Id=$PROVIDER_INSTANCE_A Id=$PROVIDER_INSTANCE_B

echo "✅ Targets registered"

# Create listener
LISTENER_ARN=$(aws elbv2 create-listener \
    --load-balancer-arn $NLB_ARN \
    --protocol TCP \
    --port 80 \
    --default-actions Type=forward,TargetGroupArn=$TG_ARN \
    --query 'Listeners[0].ListenerArn' \
    --output text)

echo "✅ Listener created: $LISTENER_ARN"

echo ""
echo "3. CREATING VPC ENDPOINT SERVICE"

# Create VPC endpoint service
SERVICE_NAME=$(aws ec2 create-vpc-endpoint-service-configuration \
    --network-load-balancer-arns $NLB_ARN \
    --acceptance-required \
    --tag-specifications "ResourceType=vpc-endpoint-service,Tags=[{Key=Name,Value=PrivateLink-Custom-Service}]" \
    --query 'ServiceConfiguration.ServiceName' \
    --output text)

echo "✅ VPC Endpoint Service created: $SERVICE_NAME"

# Save provider configuration
cat << CONFIG > privatelink-provider-config.env
export PROVIDER_INSTANCE_A="$PROVIDER_INSTANCE_A"
export PROVIDER_INSTANCE_B="$PROVIDER_INSTANCE_B"
export NLB_ARN="$NLB_ARN"
export TG_ARN="$TG_ARN"
export LISTENER_ARN="$LISTENER_ARN"
export SERVICE_NAME="$SERVICE_NAME"
CONFIG

echo ""
echo "SERVICE PROVIDER DEPLOYMENT SUMMARY:"
echo "Provider Instances: $PROVIDER_INSTANCE_A, $PROVIDER_INSTANCE_B"
echo "Network Load Balancer: $NLB_ARN"
echo "VPC Endpoint Service: $SERVICE_NAME"
echo ""
echo "Configuration saved to privatelink-provider-config.env"

EOF

chmod +x deploy-service-provider.sh
./deploy-service-provider.sh
```

### Step 4: Create VPC Endpoints for AWS Services
**Objective**: Set up interface and gateway endpoints for AWS services

**AWS Service Endpoints Setup**:
```bash
# Create VPC endpoints for AWS services
cat << 'EOF' > create-aws-service-endpoints.sh
#!/bin/bash

source privatelink-infrastructure-config.env

echo "=== Creating VPC Endpoints for AWS Services ==="
echo ""

echo "1. CREATING GATEWAY ENDPOINTS"

# S3 Gateway Endpoint
S3_ENDPOINT_ID=$(aws ec2 create-vpc-endpoint \
    --vpc-id $CONSUMER_VPC_ID \
    --service-name com.amazonaws.us-east-1.s3 \
    --route-table-ids $(aws ec2 describe-route-tables \
        --filters "Name=vpc-id,Values=$CONSUMER_VPC_ID" \
        --query 'RouteTables[0].RouteTableId' \
        --output text) \
    --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=S3-Gateway-Endpoint}]" \
    --query 'VpcEndpoint.VpcEndpointId' \
    --output text)

echo "✅ S3 Gateway Endpoint created: $S3_ENDPOINT_ID"

# DynamoDB Gateway Endpoint
DYNAMODB_ENDPOINT_ID=$(aws ec2 create-vpc-endpoint \
    --vpc-id $CONSUMER_VPC_ID \
    --service-name com.amazonaws.us-east-1.dynamodb \
    --route-table-ids $(aws ec2 describe-route-tables \
        --filters "Name=vpc-id,Values=$CONSUMER_VPC_ID" \
        --query 'RouteTables[0].RouteTableId' \
        --output text) \
    --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=DynamoDB-Gateway-Endpoint}]" \
    --query 'VpcEndpoint.VpcEndpointId' \
    --output text)

echo "✅ DynamoDB Gateway Endpoint created: $DYNAMODB_ENDPOINT_ID"

echo ""
echo "2. CREATING INTERFACE ENDPOINTS"

# EC2 Interface Endpoint
EC2_ENDPOINT_ID=$(aws ec2 create-vpc-endpoint \
    --vpc-id $CONSUMER_VPC_ID \
    --service-name com.amazonaws.us-east-1.ec2 \
    --vpc-endpoint-type Interface \
    --subnet-ids $CONSUMER_SUBNET_A_ID $CONSUMER_SUBNET_B_ID \
    --security-group-ids $CONSUMER_SG_ID \
    --private-dns-enabled \
    --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=EC2-Interface-Endpoint}]" \
    --query 'VpcEndpoint.VpcEndpointId' \
    --output text)

echo "✅ EC2 Interface Endpoint created: $EC2_ENDPOINT_ID"

# RDS Interface Endpoint
RDS_ENDPOINT_ID=$(aws ec2 create-vpc-endpoint \
    --vpc-id $CONSUMER_VPC_ID \
    --service-name com.amazonaws.us-east-1.rds \
    --vpc-endpoint-type Interface \
    --subnet-ids $CONSUMER_SUBNET_A_ID $CONSUMER_SUBNET_B_ID \
    --security-group-ids $CONSUMER_SG_ID \
    --private-dns-enabled \
    --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=RDS-Interface-Endpoint}]" \
    --query 'VpcEndpoint.VpcEndpointId' \
    --output text)

echo "✅ RDS Interface Endpoint created: $RDS_ENDPOINT_ID"

# CloudWatch Interface Endpoint
CLOUDWATCH_ENDPOINT_ID=$(aws ec2 create-vpc-endpoint \
    --vpc-id $CONSUMER_VPC_ID \
    --service-name com.amazonaws.us-east-1.monitoring \
    --vpc-endpoint-type Interface \
    --subnet-ids $CONSUMER_SUBNET_A_ID $CONSUMER_SUBNET_B_ID \
    --security-group-ids $CONSUMER_SG_ID \
    --private-dns-enabled \
    --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=CloudWatch-Interface-Endpoint}]" \
    --query 'VpcEndpoint.VpcEndpointId' \
    --output text)

echo "✅ CloudWatch Interface Endpoint created: $CLOUDWATCH_ENDPOINT_ID"

echo ""
echo "3. CREATING ENDPOINT POLICIES"

# Create restrictive endpoint policy for S3
cat << 'POLICY' > s3-endpoint-policy.json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": "*",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::example-bucket",
                "arn:aws:s3:::example-bucket/*"
            ],
            "Condition": {
                "StringEquals": {
                    "aws:PrincipalVpc": "$CONSUMER_VPC_ID"
                }
            }
        }
    ]
}
POLICY

echo "✅ S3 endpoint policy created"

# Save AWS endpoints configuration
cat << CONFIG > aws-endpoints-config.env
export S3_ENDPOINT_ID="$S3_ENDPOINT_ID"
export DYNAMODB_ENDPOINT_ID="$DYNAMODB_ENDPOINT_ID"
export EC2_ENDPOINT_ID="$EC2_ENDPOINT_ID"
export RDS_ENDPOINT_ID="$RDS_ENDPOINT_ID"
export CLOUDWATCH_ENDPOINT_ID="$CLOUDWATCH_ENDPOINT_ID"
CONFIG

echo ""
echo "AWS SERVICE ENDPOINTS CREATED:"
echo "S3 Gateway Endpoint: $S3_ENDPOINT_ID"
echo "DynamoDB Gateway Endpoint: $DYNAMODB_ENDPOINT_ID"
echo "EC2 Interface Endpoint: $EC2_ENDPOINT_ID"
echo "RDS Interface Endpoint: $RDS_ENDPOINT_ID"
echo "CloudWatch Interface Endpoint: $CLOUDWATCH_ENDPOINT_ID"
echo ""
echo "Configuration saved to aws-endpoints-config.env"

EOF

chmod +x create-aws-service-endpoints.sh
./create-aws-service-endpoints.sh
```

### Step 5: Create Interface Endpoint for Custom Service
**Objective**: Connect to custom PrivateLink service from consumer VPC

**Custom Service Endpoint Setup**:
```bash
# Create interface endpoint for custom service
cat << 'EOF' > create-custom-service-endpoint.sh
#!/bin/bash

source privatelink-infrastructure-config.env
source privatelink-provider-config.env

echo "=== Creating Interface Endpoint for Custom Service ==="
echo ""

echo "1. CREATING INTERFACE ENDPOINT"
echo "Service Name: $SERVICE_NAME"

# Create interface endpoint for custom service
CUSTOM_ENDPOINT_ID=$(aws ec2 create-vpc-endpoint \
    --vpc-id $CONSUMER_VPC_ID \
    --service-name $SERVICE_NAME \
    --vpc-endpoint-type Interface \
    --subnet-ids $CONSUMER_SUBNET_A_ID $CONSUMER_SUBNET_B_ID \
    --security-group-ids $CONSUMER_SG_ID \
    --private-dns-enabled \
    --tag-specifications "ResourceType=vpc-endpoint,Tags=[{Key=Name,Value=Custom-Service-Endpoint}]" \
    --query 'VpcEndpoint.VpcEndpointId' \
    --output text)

echo "✅ Custom service interface endpoint created: $CUSTOM_ENDPOINT_ID"

echo ""
echo "2. WAITING FOR ENDPOINT TO BE AVAILABLE"
echo "Note: This may take several minutes..."

# Wait for endpoint to be available
aws ec2 wait vpc-endpoint-available --vpc-endpoint-ids $CUSTOM_ENDPOINT_ID
echo "✅ Endpoint is available"

echo ""
echo "3. ACCEPTING ENDPOINT CONNECTION"

# Get connection ID
CONNECTION_ID=$(aws ec2 describe-vpc-endpoint-connections \
    --filters "Name=service-name,Values=$SERVICE_NAME" \
    --query 'VpcEndpointConnections[0].VpcEndpointConnectionId' \
    --output text)

echo "Connection ID: $CONNECTION_ID"

# Accept the connection
aws ec2 accept-vpc-endpoint-connections \
    --service-id $(aws ec2 describe-vpc-endpoint-service-configurations \
        --filters "Name=service-name,Values=$SERVICE_NAME" \
        --query 'ServiceConfigurations[0].ServiceId' \
        --output text) \
    --vpc-endpoint-ids $CUSTOM_ENDPOINT_ID

echo "✅ Endpoint connection accepted"

echo ""
echo "4. GETTING ENDPOINT DNS NAMES"

# Get endpoint DNS names
ENDPOINT_DNS=$(aws ec2 describe-vpc-endpoints \
    --vpc-endpoint-ids $CUSTOM_ENDPOINT_ID \
    --query 'VpcEndpoints[0].DnsEntries[0].DnsName' \
    --output text)

echo "✅ Endpoint DNS Name: $ENDPOINT_DNS"

# Save custom endpoint configuration
cat << CONFIG > custom-endpoint-config.env
export CUSTOM_ENDPOINT_ID="$CUSTOM_ENDPOINT_ID"
export CONNECTION_ID="$CONNECTION_ID"
export ENDPOINT_DNS="$ENDPOINT_DNS"
CONFIG

echo ""
echo "CUSTOM SERVICE ENDPOINT SUMMARY:"
echo "Endpoint ID: $CUSTOM_ENDPOINT_ID"
echo "Connection ID: $CONNECTION_ID"
echo "DNS Name: $ENDPOINT_DNS"
echo ""
echo "Configuration saved to custom-endpoint-config.env"

EOF

chmod +x create-custom-service-endpoint.sh
./create-custom-service-endpoint.sh
```

### Step 6: Test PrivateLink Connectivity
**Objective**: Verify PrivateLink functionality and troubleshoot issues

**Connectivity Testing**:
```bash
# Test PrivateLink connectivity
cat << 'EOF' > test-privatelink-connectivity.sh
#!/bin/bash

source privatelink-infrastructure-config.env
source custom-endpoint-config.env

echo "=== Testing PrivateLink Connectivity ==="
echo ""

echo "1. LAUNCHING CONSUMER TEST INSTANCE"

# Get latest Amazon Linux 2 AMI
AMI_ID=$(aws ec2 describe-images \
    --owners amazon \
    --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
              "Name=state,Values=available" \
    --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
    --output text)

# Launch consumer test instance
CONSUMER_INSTANCE=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.micro \
    --subnet-id $CONSUMER_SUBNET_A_ID \
    --security-group-ids $CONSUMER_SG_ID \
    --user-data '#!/bin/bash
yum update -y
yum install -y curl dig tcpdump' \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=PrivateLink-Consumer-Test}]" \
    --query 'Instances[0].InstanceId' \
    --output text)

echo "✅ Consumer test instance launched: $CONSUMER_INSTANCE"

# Wait for instance to be running
echo "Waiting for instance to be running..."
aws ec2 wait instance-running --instance-ids $CONSUMER_INSTANCE
echo "✅ Instance is running"

echo ""
echo "2. TESTING DNS RESOLUTION"

# Create DNS test script
cat << 'DNS_TEST' > dns-test.sh
#!/bin/bash

echo "Testing DNS resolution for PrivateLink endpoints..."
echo ""

echo "Custom Service Endpoint:"
dig $ENDPOINT_DNS
echo ""

echo "AWS Service Endpoints:"
dig ec2.us-east-1.amazonaws.com
echo ""
dig rds.us-east-1.amazonaws.com
echo ""

echo "S3 Service (Gateway Endpoint - no DNS entry):"
echo "Gateway endpoints use route table entries, not DNS"
echo ""

DNS_TEST

echo "✅ DNS test script created"

echo ""
echo "3. TESTING HTTP CONNECTIVITY"

# Create connectivity test script
cat << 'CONN_TEST' > connectivity-test.sh
#!/bin/bash

echo "Testing HTTP connectivity to custom service..."
echo ""

echo "Testing via endpoint DNS name:"
curl -I http://$ENDPOINT_DNS/ || echo "Connection failed"
echo ""

echo "Testing with verbose output:"
curl -v http://$ENDPOINT_DNS/ || echo "Connection failed"
echo ""

echo "Testing multiple requests to verify load balancing:"
for i in {1..5}; do
    echo "Request $i:"
    curl -s http://$ENDPOINT_DNS/ | grep "Instance"
done

CONN_TEST

echo "✅ Connectivity test script created"

echo ""
echo "4. CREATING MONITORING SCRIPT"

# Create monitoring script
cat << 'MONITOR' > monitor-privatelink.sh
#!/bin/bash

echo "PrivateLink Monitoring Dashboard"
echo "================================"
echo ""

echo "1. VPC ENDPOINTS STATUS"
aws ec2 describe-vpc-endpoints \
    --vpc-endpoint-ids $CUSTOM_ENDPOINT_ID \
    --query 'VpcEndpoints[0].[VpcEndpointId,State,ServiceName]' \
    --output table

echo ""
echo "2. ENDPOINT CONNECTIONS"
aws ec2 describe-vpc-endpoint-connections \
    --filters "Name=vpc-endpoint-id,Values=$CUSTOM_ENDPOINT_ID" \
    --query 'VpcEndpointConnections[*].[VpcEndpointId,VpcEndpointState,CreationTimestamp]' \
    --output table

echo ""
echo "3. NETWORK LOAD BALANCER HEALTH"
aws elbv2 describe-target-health \
    --target-group-arn $TG_ARN \
    --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State,TargetHealth.Description]' \
    --output table

echo ""
echo "4. DNS ENTRIES"
aws ec2 describe-vpc-endpoints \
    --vpc-endpoint-ids $CUSTOM_ENDPOINT_ID \
    --query 'VpcEndpoints[0].DnsEntries[*].[DnsName,HostedZoneId]' \
    --output table

MONITOR

chmod +x monitor-privatelink.sh

echo "✅ Monitoring script created"

# Save test configuration
cat << CONFIG > test-config.env
export CONSUMER_INSTANCE="$CONSUMER_INSTANCE"
CONFIG

echo ""
echo "PRIVATELINK TESTING SETUP COMPLETE"
echo ""
echo "Test Components:"
echo "Consumer Instance: $CONSUMER_INSTANCE"
echo "DNS Test Script: dns-test.sh"
echo "Connectivity Test Script: connectivity-test.sh"
echo "Monitoring Script: monitor-privatelink.sh"
echo ""
echo "To run tests:"
echo "1. ./dns-test.sh - Test DNS resolution"
echo "2. ./connectivity-test.sh - Test HTTP connectivity"
echo "3. ./monitor-privatelink.sh - Monitor PrivateLink status"

EOF

chmod +x test-privatelink-connectivity.sh
./test-privatelink-connectivity.sh
```

## PrivateLink Architecture Patterns Summary

### Interface Endpoints vs Gateway Endpoints

#### Interface Endpoints
- **Technology**: Elastic Network Interfaces (ENIs) with private IP addresses
- **Supported Services**: 100+ AWS services (EC2, RDS, Lambda, etc.)
- **Cost Model**: Hourly charges + data transfer charges
- **DNS Integration**: Private DNS names resolve to endpoint IPs
- **Security**: Security groups and endpoint policies

#### Gateway Endpoints
- **Technology**: Route table entries (no ENIs required)
- **Supported Services**: S3 and DynamoDB only
- **Cost Model**: No hourly charges, only data transfer within region
- **DNS Integration**: No private DNS, uses service DNS names
- **Security**: Endpoint policies only

### PrivateLink Security Features

#### Endpoint Policies
- **IAM-style Policies**: JSON policies for resource-level access control
- **Principal-based**: Control which AWS accounts, users, or roles can access
- **Action-based**: Specify allowed API actions
- **Resource-based**: Limit access to specific resources

#### Security Groups
- **Network-level Control**: Control traffic to/from endpoint ENIs
- **Port-specific**: Allow specific ports and protocols
- **Source-based**: Restrict access from specific IP ranges or security groups

### Cross-Account PrivateLink Patterns

#### Service Sharing Architecture
```
Account A (Service Provider):
    Application → NLB → VPC Endpoint Service
                        ↓
Account B (Service Consumer):
    Applications → Interface Endpoint → Account A Service
```

#### Permission Model
- **Service Provider**: Controls which accounts can create connections
- **Service Consumer**: Creates interface endpoints and manages local access
- **Connection Approval**: Manual or automatic acceptance of connections

## Best Practices for PrivateLink

### Design Principles
- **Service-oriented Architecture**: Use PrivateLink for microservices communication
- **Security by Default**: Implement restrictive endpoint policies
- **Cost Optimization**: Use gateway endpoints for S3/DynamoDB when possible
- **Multi-AZ Deployment**: Deploy endpoints across multiple availability zones

### Security Considerations
- **Least Privilege**: Implement minimal required permissions in endpoint policies
- **Network Segmentation**: Use security groups to control endpoint access
- **Monitoring**: Enable VPC Flow Logs and CloudTrail for endpoint access
- **DNS Security**: Validate DNS resolution to ensure traffic flows through endpoints

### Performance Optimization
- **Regional Endpoints**: Use endpoints in the same region as consumers
- **Load Balancing**: Use Network Load Balancers for custom services
- **Connection Pooling**: Implement connection reuse for better performance
- **Monitoring**: Track endpoint performance metrics and optimize accordingly

## Key Takeaways
- PrivateLink provides secure, private connectivity without internet exposure
- Interface endpoints support 100+ AWS services with ENI-based connectivity
- Gateway endpoints provide cost-effective access to S3 and DynamoDB
- VPC Endpoint Services enable sharing of custom applications across accounts
- Endpoint policies provide fine-grained access control
- Proper DNS configuration ensures seamless application integration
- Monitoring and troubleshooting are essential for production deployments

## Additional Resources
- [AWS PrivateLink User Guide](https://docs.aws.amazon.com/vpc/latest/privatelink/)
- [VPC Endpoints Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-endpoints.html)
- [PrivateLink Pricing](https://aws.amazon.com/privatelink/pricing/)

## Exam Tips
- PrivateLink keeps traffic within AWS network backbone
- Interface endpoints use ENIs, gateway endpoints use route tables
- S3 and DynamoDB support both interface and gateway endpoints
- Gateway endpoints have no hourly charges
- VPC Endpoint Services require Network Load Balancers
- Endpoint policies are separate from IAM policies
- Cross-account PrivateLink requires connection approval
- Private DNS makes PrivateLink transparent to applications
- PrivateLink supports cross-region connectivity via Transit Gateway