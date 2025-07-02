# Topic 10: Mastering VPC Flow Logs - Network Monitoring Essentials

## Prerequisites
- Completed Topics 1-9 (AWS Networking Foundations)
- Understanding of network protocols and traffic analysis
- Basic knowledge of CloudWatch and log analysis

## Learning Objectives
By the end of this topic, you will be able to:
- Configure VPC Flow Logs for comprehensive network monitoring
- Analyze flow log data for security and performance insights
- Set up automated alerting based on flow log patterns
- Implement cost-effective flow log strategies

## Theory

### VPC Flow Logs Overview

#### Definition
VPC Flow Logs capture information about IP traffic going to and from network interfaces in your VPC. Flow logs can help you monitor and troubleshoot connectivity issues.

#### Key Characteristics
- **Comprehensive Coverage**: Captures all network traffic metadata
- **Multiple Destinations**: CloudWatch Logs, S3, or Kinesis Data Firehose
- **Flexible Scope**: VPC, subnet, or network interface level
- **Rich Metadata**: Source, destination, ports, protocols, and more
- **Security Insights**: Accept/reject decisions and traffic patterns

### Flow Log Record Format

#### Default Format Fields
```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes windowstart windowend action flowlogstatus
```

#### Custom Format Options
```
${version} ${account-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${windowstart} ${windowend} ${action} ${flowlogstatus} ${vpc-id} ${subnet-id} ${instance-id} ${tcp-flags} ${type} ${pkt-srcaddr} ${pkt-dstaddr} ${region} ${az-id} ${sublocation-type} ${sublocation-id}
```

### Flow Log Scope Options

#### 1. VPC Level
- **Coverage**: All network interfaces in VPC
- **Use Case**: Comprehensive VPC monitoring
- **Cost**: Higher due to volume
- **Granularity**: Less specific to individual resources

#### 2. Subnet Level  
- **Coverage**: All network interfaces in subnet
- **Use Case**: Tier-specific monitoring (web, app, db)
- **Cost**: Moderate
- **Granularity**: Good balance of detail and cost

#### 3. Network Interface Level
- **Coverage**: Specific network interface
- **Use Case**: Critical instance monitoring
- **Cost**: Lower due to specificity
- **Granularity**: Very detailed for specific resources

### Flow Log Destinations

#### CloudWatch Logs
- **Pros**: Real-time analysis, CloudWatch Insights queries, alerting
- **Cons**: Higher cost for large volumes, retention costs
- **Best For**: Real-time monitoring and alerting

#### Amazon S3
- **Pros**: Cost-effective storage, integration with analytics tools
- **Cons**: Not real-time, requires additional processing
- **Best For**: Long-term retention and batch analysis

#### Kinesis Data Firehose
- **Pros**: Near real-time, automatic delivery to multiple destinations
- **Cons**: Additional complexity, streaming costs
- **Best For**: Real-time analytics pipelines

## Lab Exercise: Comprehensive Flow Logs Implementation

### Lab Duration: 180 minutes

### Step 1: Create Multi-Tier VPC for Flow Log Testing
**Objective**: Set up environment with different traffic patterns

```bash
# Create VPC for flow log testing
FLOWLOG_VPC=$(aws ec2 create-vpc --cidr-block 10.50.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=FlowLog-Test-VPC}]' \
    --query 'Vpc.VpcId' --output text)

aws ec2 modify-vpc-attribute --vpc-id $FLOWLOG_VPC --enable-dns-support
aws ec2 modify-vpc-attribute --vpc-id $FLOWLOG_VPC --enable-dns-hostnames

# Create subnets in different tiers
PUBLIC_SUBNET=$(aws ec2 create-subnet --vpc-id $FLOWLOG_VPC --cidr-block 10.50.1.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Public-Subnet},{Key=Tier,Value=Web}]' \
    --query 'Subnet.SubnetId' --output text)

PRIVATE_SUBNET=$(aws ec2 create-subnet --vpc-id $FLOWLOG_VPC --cidr-block 10.50.2.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=Private-Subnet},{Key=Tier,Value=App}]' \
    --query 'Subnet.SubnetId' --output text)

DB_SUBNET=$(aws ec2 create-subnet --vpc-id $FLOWLOG_VPC --cidr-block 10.50.3.0/24 --availability-zone us-east-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=DB-Subnet},{Key=Tier,Value=Database}]' \
    --query 'Subnet.SubnetId' --output text)

# Create Internet Gateway
IGW_ID=$(aws ec2 create-internet-gateway \
    --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=FlowLog-IGW}]' \
    --query 'InternetGateway.InternetGatewayId' --output text)

aws ec2 attach-internet-gateway --internet-gateway-id $IGW_ID --vpc-id $FLOWLOG_VPC

# Create NAT Gateway for private subnet internet access
EIP_ALLOC=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)

NAT_GW=$(aws ec2 create-nat-gateway --subnet-id $PUBLIC_SUBNET --allocation-id $EIP_ALLOC \
    --tag-specifications 'ResourceType=nat-gateway,Tags=[{Key=Name,Value=FlowLog-NAT}]' \
    --query 'NatGateway.NatGatewayId' --output text)

aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW

echo "VPC Infrastructure Created:"
echo "VPC: $FLOWLOG_VPC"
echo "Public Subnet: $PUBLIC_SUBNET"
echo "Private Subnet: $PRIVATE_SUBNET"
echo "DB Subnet: $DB_SUBNET"
echo "NAT Gateway: $NAT_GW"
```

### Step 2: Configure Route Tables
**Objective**: Set up proper routing for traffic flow

```bash
# Create route table for public subnet
PUBLIC_RT=$(aws ec2 create-route-table --vpc-id $FLOWLOG_VPC \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Public-RT}]' \
    --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $PUBLIC_RT --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $PUBLIC_RT --subnet-id $PUBLIC_SUBNET

# Create route table for private subnet
PRIVATE_RT=$(aws ec2 create-route-table --vpc-id $FLOWLOG_VPC \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Private-RT}]' \
    --query 'RouteTable.RouteTableId' --output text)

aws ec2 create-route --route-table-id $PRIVATE_RT --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW
aws ec2 associate-route-table --route-table-id $PRIVATE_RT --subnet-id $PRIVATE_SUBNET

# DB subnet uses main route table (no internet access)
echo "Route tables configured for different access patterns"
```

### Step 3: Create IAM Role for Flow Logs
**Objective**: Set up proper permissions for flow log delivery

```bash
# Create IAM role for flow logs
cat << 'EOF' > flowlogs-trust-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "vpc-flow-logs.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
FLOWLOGS_ROLE=$(aws iam create-role \
    --role-name FlowLogsDeliveryRole \
    --assume-role-policy-document file://flowlogs-trust-policy.json \
    --query 'Role.Arn' --output text)

# Create policy for CloudWatch Logs delivery
cat << 'EOF' > flowlogs-delivery-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents",
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
EOF

aws iam put-role-policy \
    --role-name FlowLogsDeliveryRole \
    --policy-name FlowLogsDeliveryRolePolicy \
    --policy-document file://flowlogs-delivery-policy.json

echo "IAM role created: $FLOWLOGS_ROLE"

# Wait for role to propagate
sleep 10
```

### Step 4: Create CloudWatch Log Groups
**Objective**: Set up destinations for flow log data

```bash
# Create log groups for different scopes
aws logs create-log-group --log-group-name /aws/vpc/flowlogs/vpc-level
aws logs create-log-group --log-group-name /aws/vpc/flowlogs/subnet-level
aws logs create-log-group --log-group-name /aws/vpc/flowlogs/eni-level

# Set retention policy (14 days for cost optimization)
aws logs put-retention-policy --log-group-name /aws/vpc/flowlogs/vpc-level --retention-in-days 14
aws logs put-retention-policy --log-group-name /aws/vpc/flowlogs/subnet-level --retention-in-days 14
aws logs put-retention-policy --log-group-name /aws/vpc/flowlogs/eni-level --retention-in-days 14

echo "CloudWatch Log Groups created with 14-day retention"
```

### Step 5: Configure VPC Flow Logs at Different Scopes
**Objective**: Implement flow logs at VPC, subnet, and ENI levels

**VPC-Level Flow Logs**:
```bash
# Create VPC-level flow logs with default format
VPC_FLOWLOG=$(aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids $FLOWLOG_VPC \
    --traffic-type ALL \
    --log-destination-type cloud-watch-logs \
    --log-group-name /aws/vpc/flowlogs/vpc-level \
    --deliver-logs-permission-arn $FLOWLOGS_ROLE \
    --tag-specifications 'ResourceType=vpc-flow-log,Tags=[{Key=Name,Value=VPC-Level-FlowLog}]' \
    --query 'FlowLogIds[0]' --output text)

echo "VPC-level flow log created: $VPC_FLOWLOG"
```

**Subnet-Level Flow Logs with Custom Format**:
```bash
# Create subnet-level flow logs with custom format for detailed analysis
CUSTOM_FORMAT='${version} ${account-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${windowstart} ${windowend} ${action} ${flowlogstatus} ${vpc-id} ${subnet-id} ${instance-id} ${tcp-flags} ${type} ${pkt-srcaddr} ${pkt-dstaddr}'

SUBNET_FLOWLOG=$(aws ec2 create-flow-logs \
    --resource-type Subnet \
    --resource-ids $PRIVATE_SUBNET \
    --traffic-type ALL \
    --log-destination-type cloud-watch-logs \
    --log-group-name /aws/vpc/flowlogs/subnet-level \
    --deliver-logs-permission-arn $FLOWLOGS_ROLE \
    --log-format "$CUSTOM_FORMAT" \
    --tag-specifications 'ResourceType=vpc-flow-log,Tags=[{Key=Name,Value=Subnet-Level-FlowLog}]' \
    --query 'FlowLogIds[0]' --output text)

echo "Subnet-level flow log created: $SUBNET_FLOWLOG"
```

**S3 Destination Flow Logs**:
```bash
# Create S3 bucket for flow logs
S3_BUCKET="flowlogs-analysis-$(date +%s)"
aws s3 mb s3://$S3_BUCKET

# Create bucket policy for flow logs
cat << EOF > s3-bucket-policy.json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AWSLogDeliveryWrite",
      "Effect": "Allow",
      "Principal": {
        "Service": "delivery.logs.amazonaws.com"
      },
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::$S3_BUCKET/*"
    },
    {
      "Sid": "AWSLogDeliveryAclCheck",
      "Effect": "Allow",
      "Principal": {
        "Service": "delivery.logs.amazonaws.com"
      },
      "Action": "s3:GetBucketAcl",
      "Resource": "arn:aws:s3:::$S3_BUCKET"
    }
  ]
}
EOF

aws s3api put-bucket-policy --bucket $S3_BUCKET --policy file://s3-bucket-policy.json

# Create flow logs to S3
S3_FLOWLOG=$(aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids $FLOWLOG_VPC \
    --traffic-type REJECT \
    --log-destination-type s3 \
    --log-destination "arn:aws:s3:::$S3_BUCKET/rejected-traffic/" \
    --log-format "$CUSTOM_FORMAT" \
    --tag-specifications 'ResourceType=vpc-flow-log,Tags=[{Key=Name,Value=S3-Rejected-Traffic}]' \
    --query 'FlowLogIds[0]' --output text)

echo "S3 flow log created: $S3_FLOWLOG"
echo "S3 bucket: $S3_BUCKET"
```

### Step 6: Launch Test Instances and Generate Traffic
**Objective**: Create realistic traffic patterns for flow log analysis

```bash
# Create security groups with different access patterns
WEB_SG=$(aws ec2 create-security-group \
    --group-name Web-SG \
    --description "Web tier security group" \
    --vpc-id $FLOWLOG_VPC \
    --query 'GroupId' --output text)

APP_SG=$(aws ec2 create-security-group \
    --group-name App-SG \
    --description "App tier security group" \
    --vpc-id $FLOWLOG_VPC \
    --query 'GroupId' --output text)

DB_SG=$(aws ec2 create-security-group \
    --group-name DB-SG \
    --description "Database tier security group" \
    --vpc-id $FLOWLOG_VPC \
    --query 'GroupId' --output text)

# Configure security group rules
# Web tier: Allow HTTP/HTTPS from internet, SSH from anywhere
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $WEB_SG --protocol tcp --port 22 --cidr 0.0.0.0/0

# App tier: Allow from web tier only
aws ec2 authorize-security-group-ingress --group-id $APP_SG --protocol tcp --port 8080 --source-group $WEB_SG
aws ec2 authorize-security-group-ingress --group-id $APP_SG --protocol tcp --port 22 --source-group $WEB_SG

# DB tier: Allow from app tier only
aws ec2 authorize-security-group-ingress --group-id $DB_SG --protocol tcp --port 3306 --source-group $APP_SG
aws ec2 authorize-security-group-ingress --group-id $DB_SG --protocol tcp --port 22 --source-group $WEB_SG

# Launch instances in each tier
WEB_INSTANCE=$(aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --count 1 \
    --instance-type t3.micro \
    --key-name your-key-pair \
    --security-group-ids $WEB_SG \
    --subnet-id $PUBLIC_SUBNET \
    --associate-public-ip-address \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=Web-Server}]' \
    --user-data '#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Web Server - Flow Logs Test</h1>" > /var/www/html/index.html' \
    --query 'Instances[0].InstanceId' --output text)

APP_INSTANCE=$(aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --count 1 \
    --instance-type t3.micro \
    --key-name your-key-pair \
    --security-group-ids $APP_SG \
    --subnet-id $PRIVATE_SUBNET \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=App-Server}]' \
    --query 'Instances[0].InstanceId' --output text)

DB_INSTANCE=$(aws ec2 run-instances \
    --image-id ami-0abcdef1234567890 \
    --count 1 \
    --instance-type t3.micro \
    --key-name your-key-pair \
    --security-group-ids $DB_SG \
    --subnet-id $DB_SUBNET \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=DB-Server}]' \
    --query 'Instances[0].InstanceId' --output text)

# Wait for instances to be running
aws ec2 wait instance-running --instance-ids $WEB_INSTANCE $APP_INSTANCE $DB_INSTANCE

echo "Test instances launched:"
echo "Web Server: $WEB_INSTANCE"
echo "App Server: $APP_INSTANCE"
echo "DB Server: $DB_INSTANCE"
```

**Generate Test Traffic**:
```bash
# Get instance IPs for testing
WEB_PUBLIC_IP=$(aws ec2 describe-instances --instance-ids $WEB_INSTANCE \
    --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)

WEB_PRIVATE_IP=$(aws ec2 describe-instances --instance-ids $WEB_INSTANCE \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

APP_PRIVATE_IP=$(aws ec2 describe-instances --instance-ids $APP_INSTANCE \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

DB_PRIVATE_IP=$(aws ec2 describe-instances --instance-ids $DB_INSTANCE \
    --query 'Reservations[0].Instances[0].PrivateIpAddress' --output text)

# Create traffic generation script
cat << 'EOF' > generate-traffic.sh
#!/bin/bash

WEB_IP=$1
APP_IP=$2
DB_IP=$3

echo "Generating test traffic patterns..."

# Simulate legitimate web traffic
for i in {1..10}; do
    curl -s http://$WEB_IP/ > /dev/null
    sleep 1
done

# Simulate blocked traffic (will be rejected by security groups)
for i in {1..5}; do
    nc -z -w1 $APP_IP 22 2>/dev/null  # SSH attempt to app server
    nc -z -w1 $DB_IP 3306 2>/dev/null # MySQL attempt to DB server
    sleep 2
done

# Simulate internal traffic
ssh -o ConnectTimeout=1 -o StrictHostKeyChecking=no ec2-user@$WEB_IP "
    # From web server, try to connect to app server
    nc -z -w1 $APP_IP 8080
    # From web server, try unauthorized DB access (will be blocked)
    nc -z -w1 $DB_IP 3306
" 2>/dev/null

echo "Traffic generation completed"
EOF

chmod +x generate-traffic.sh

echo "Run traffic generation with:"
echo "./generate-traffic.sh $WEB_PUBLIC_IP $APP_PRIVATE_IP $DB_PRIVATE_IP"

# Generate some immediate traffic
./generate-traffic.sh $WEB_PUBLIC_IP $APP_PRIVATE_IP $DB_PRIVATE_IP
```

### Step 7: Analyze Flow Log Data
**Objective**: Extract insights from flow log data using CloudWatch Insights

**Wait for Flow Log Data**:
```bash
echo "Waiting for flow log data to appear (5-10 minutes)..."
sleep 300  # Wait 5 minutes for data collection
```

**CloudWatch Insights Queries**:
```bash
# Create comprehensive analysis queries
cat << 'EOF' > flowlog-queries.txt
# 1. Top Talkers (Most Active Source IPs)
fields @timestamp, srcaddr, dstaddr, srcport, dstport, protocol, bytes
| filter action = "ACCEPT"
| stats sum(bytes) as total_bytes by srcaddr
| sort total_bytes desc
| limit 10

# 2. Rejected Connections (Security Analysis)
fields @timestamp, srcaddr, dstaddr, srcport, dstport, protocol
| filter action = "REJECT"
| stats count() as rejection_count by srcaddr, dstaddr, dstport
| sort rejection_count desc
| limit 20

# 3. Protocol Distribution
fields @timestamp, protocol, action
| filter action = "ACCEPT"
| stats count() as connection_count by protocol
| sort connection_count desc

# 4. High Volume Connections
fields @timestamp, srcaddr, dstaddr, srcport, dstport, bytes, packets
| filter action = "ACCEPT"
| filter bytes > 1000000  # More than 1MB
| sort bytes desc
| limit 10

# 5. Time-based Traffic Analysis
fields @timestamp, srcaddr, dstaddr, bytes
| filter action = "ACCEPT"
| bin(@timestamp, 5m) as time_window
| stats sum(bytes) as total_bytes by time_window
| sort time_window desc

# 6. Internal vs External Traffic
fields @timestamp, srcaddr, dstaddr, bytes
| filter action = "ACCEPT"
| filter srcaddr like /^10\./ or srcaddr like /^172\./ or srcaddr like /^192\.168\./
| stats sum(bytes) as internal_bytes, count() as internal_connections
| limit 1

# 7. Port Scanning Detection
fields @timestamp, srcaddr, dstaddr, dstport
| filter action = "REJECT"
| stats count_distinct(dstport) as unique_ports by srcaddr
| filter unique_ports > 5
| sort unique_ports desc

# 8. Database Access Patterns (Port 3306)
fields @timestamp, srcaddr, dstaddr, action
| filter dstport = 3306
| stats count() as mysql_attempts by srcaddr, action
| sort mysql_attempts desc

EOF

echo "CloudWatch Insights queries saved to flowlog-queries.txt"
echo "Use these queries in CloudWatch Logs Insights console"
```

**Automated Analysis Script**:
```bash
# Create automated analysis script
cat << 'EOF' > analyze-flowlogs.sh
#!/bin/bash

LOG_GROUP="/aws/vpc/flowlogs/vpc-level"
START_TIME=$(date -d '30 minutes ago' +%s)
END_TIME=$(date +%s)

echo "=== VPC Flow Logs Analysis Report ==="
echo "Time Range: $(date -d @$START_TIME) to $(date -d @$END_TIME)"
echo ""

# Top source IPs by data volume
echo "=== Top Source IPs by Data Volume ==="
aws logs start-query \
    --log-group-name $LOG_GROUP \
    --start-time $START_TIME \
    --end-time $END_TIME \
    --query-string 'fields @timestamp, srcaddr, bytes | filter action = "ACCEPT" | stats sum(bytes) as total_bytes by srcaddr | sort total_bytes desc | limit 5' \
    --query 'queryId' --output text > /tmp/query1.id

sleep 10

aws logs get-query-results --query-id $(cat /tmp/query1.id) \
    --query 'results[*][1].value' --output table

echo ""

# Rejected connections analysis
echo "=== Top Rejected Connections ==="
aws logs start-query \
    --log-group-name $LOG_GROUP \
    --start-time $START_TIME \
    --end-time $END_TIME \
    --query-string 'fields @timestamp, srcaddr, dstaddr, dstport | filter action = "REJECT" | stats count() as rejections by srcaddr, dstport | sort rejections desc | limit 5' \
    --query 'queryId' --output text > /tmp/query2.id

sleep 10

aws logs get-query-results --query-id $(cat /tmp/query2.id) \
    --query 'results[*][1:3].[value]' --output table

EOF

chmod +x analyze-flowlogs.sh
echo "Run automated analysis with: ./analyze-flowlogs.sh"
```

### Step 8: Create Flow Log Alerts and Dashboards
**Objective**: Set up monitoring and alerting based on flow log patterns

**Create CloudWatch Alarms**:
```bash
# Create metric filter for rejected connections
aws logs put-metric-filter \
    --log-group-name /aws/vpc/flowlogs/vpc-level \
    --filter-name "RejectedConnections" \
    --filter-pattern '[version, account, eni, source, destination, srcport, destport, protocol, packets, bytes, windowstart, windowend, action="REJECT", flowlogstatus]' \
    --metric-transformations \
        metricName=RejectedConnectionCount,metricNamespace=VPC/FlowLogs,metricValue=1

# Create alarm for high rejection rate
aws cloudwatch put-metric-alarm \
    --alarm-name "High-Rejected-Connections" \
    --alarm-description "Alert when rejected connections exceed threshold" \
    --metric-name RejectedConnectionCount \
    --namespace VPC/FlowLogs \
    --statistic Sum \
    --period 300 \
    --threshold 10 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2

# Create metric filter for high data transfer
aws logs put-metric-filter \
    --log-group-name /aws/vpc/flowlogs/vpc-level \
    --filter-name "HighDataTransfer" \
    --filter-pattern '[version, account, eni, source, destination, srcport, destport, protocol, packets, bytes>1000000, windowstart, windowend, action, flowlogstatus]' \
    --metric-transformations \
        metricName=HighDataTransferCount,metricNamespace=VPC/FlowLogs,metricValue=1

echo "CloudWatch alarms created for flow log monitoring"
```

**Create Flow Log Dashboard**:
```bash
# Create CloudWatch dashboard
cat << 'EOF' > flowlog-dashboard.json
{
  "widgets": [
    {
      "type": "metric",
      "properties": {
        "metrics": [
          ["VPC/FlowLogs", "RejectedConnectionCount"],
          [".", "HighDataTransferCount"]
        ],
        "period": 300,
        "stat": "Sum",
        "region": "us-east-1",
        "title": "Flow Log Security Metrics",
        "yAxis": {
          "left": {
            "min": 0
          }
        }
      }
    },
    {
      "type": "log",
      "properties": {
        "query": "SOURCE '/aws/vpc/flowlogs/vpc-level'\n| fields @timestamp, srcaddr, dstaddr, action\n| filter action = \"REJECT\"\n| stats count() by srcaddr\n| sort count() desc\n| limit 10",
        "region": "us-east-1",
        "title": "Top Sources of Rejected Traffic",
        "view": "table"
      }
    }
  ]
}
EOF

aws cloudwatch put-dashboard \
    --dashboard-name "VPC-Flow-Logs-Security" \
    --dashboard-body file://flowlog-dashboard.json

echo "CloudWatch dashboard created: VPC-Flow-Logs-Security"
```

## Cost Optimization Strategies

### Flow Log Cost Analysis
```bash
# Calculate flow log costs
cat << 'EOF' > flowlog-cost-calculator.sh
#!/bin/bash

echo "=== VPC Flow Logs Cost Calculator ==="
echo ""

# Get log group sizes
vpc_size=$(aws logs describe-log-groups --log-group-name-prefix "/aws/vpc/flowlogs/vpc-level" \
    --query 'logGroups[0].storedBytes' --output text 2>/dev/null || echo "0")

subnet_size=$(aws logs describe-log-groups --log-group-name-prefix "/aws/vpc/flowlogs/subnet-level" \
    --query 'logGroups[0].storedBytes' --output text 2>/dev/null || echo "0")

# Convert bytes to GB
vpc_gb=$(echo "scale=4; $vpc_size / 1024 / 1024 / 1024" | bc -l)
subnet_gb=$(echo "scale=4; $subnet_size / 1024 / 1024 / 1024" | bc -l)

# CloudWatch Logs pricing (us-east-1)
ingestion_cost_per_gb=0.50
storage_cost_per_gb=0.03

vpc_ingestion_cost=$(echo "scale=2; $vpc_gb * $ingestion_cost_per_gb" | bc -l)
vpc_storage_cost=$(echo "scale=2; $vpc_gb * $storage_cost_per_gb" | bc -l)

subnet_ingestion_cost=$(echo "scale=2; $subnet_gb * $ingestion_cost_per_gb" | bc -l)
subnet_storage_cost=$(echo "scale=2; $subnet_gb * $storage_cost_per_gb" | bc -l)

echo "VPC-Level Flow Logs:"
echo "  Data stored: ${vpc_gb} GB"
echo "  Ingestion cost: \$${vpc_ingestion_cost}"
echo "  Storage cost: \$${vpc_storage_cost}"
echo ""

echo "Subnet-Level Flow Logs:"
echo "  Data stored: ${subnet_gb} GB"
echo "  Ingestion cost: \$${subnet_ingestion_cost}"
echo "  Storage cost: \$${subnet_storage_cost}"
echo ""

total_cost=$(echo "scale=2; $vpc_ingestion_cost + $vpc_storage_cost + $subnet_ingestion_cost + $subnet_storage_cost" | bc -l)
echo "Total estimated cost: \$${total_cost}"
echo ""

echo "Cost Optimization Recommendations:"
echo "- Use S3 for long-term retention (cheaper storage)"
echo "- Filter logs to specific traffic types (REJECT only)"
echo "- Use shorter retention periods for detailed logs"
echo "- Consider subnet-level vs VPC-level based on requirements"

EOF

chmod +x flowlog-cost-calculator.sh
./flowlog-cost-calculator.sh
```

## Security Use Cases

### Security Monitoring Queries
```bash
# Create security-focused analysis queries
cat << 'EOF' > security-analysis-queries.txt
# 1. Potential Port Scanning Detection
fields @timestamp, srcaddr, dstaddr, dstport, action
| filter action = "REJECT"
| stats count_distinct(dstport) as unique_ports, count() as total_attempts by srcaddr
| filter unique_ports > 10 or total_attempts > 50
| sort total_attempts desc

# 2. Unusual Protocol Usage
fields @timestamp, srcaddr, protocol, action
| filter action = "ACCEPT"
| filter protocol != "6" and protocol != "17" and protocol != "1"  # Not TCP, UDP, or ICMP
| stats count() as unusual_protocol_count by srcaddr, protocol
| sort unusual_protocol_count desc

# 3. High Volume Data Exfiltration Detection
fields @timestamp, srcaddr, dstaddr, bytes, action
| filter action = "ACCEPT"
| filter not (dstaddr like /^10\./ or dstaddr like /^172\./ or dstaddr like /^192\.168\./)
| stats sum(bytes) as total_external_bytes by srcaddr
| filter total_external_bytes > 100000000  # More than 100MB
| sort total_external_bytes desc

# 4. Database Access from Unauthorized Sources
fields @timestamp, srcaddr, dstaddr, dstport, action
| filter dstport in [3306, 5432, 1433, 1521]  # MySQL, PostgreSQL, SQL Server, Oracle
| filter action = "REJECT"
| stats count() as unauthorized_db_attempts by srcaddr, dstport
| sort unauthorized_db_attempts desc

# 5. Suspicious SSH Activity
fields @timestamp, srcaddr, dstaddr, dstport, action
| filter dstport = 22
| stats count() as ssh_attempts, count_distinct(dstaddr) as unique_targets by srcaddr
| filter ssh_attempts > 20 or unique_targets > 5
| sort ssh_attempts desc

EOF

echo "Security analysis queries saved to security-analysis-queries.txt"
```

## Flow Log Best Practices

### Implementation Guidelines
```bash
# Best practices implementation script
cat << 'EOF' > flowlog-best-practices.sh
#!/bin/bash

echo "=== VPC Flow Logs Best Practices ==="
echo ""

echo "1. Scope Selection:"
echo "   ✓ Use VPC-level for comprehensive monitoring"
echo "   ✓ Use subnet-level for tier-specific analysis"
echo "   ✓ Use ENI-level for critical instance monitoring"
echo ""

echo "2. Destination Strategy:"
echo "   ✓ CloudWatch Logs: Real-time analysis and alerting"
echo "   ✓ S3: Cost-effective long-term storage"
echo "   ✓ Kinesis Firehose: Streaming analytics pipelines"
echo ""

echo "3. Cost Optimization:"
echo "   ✓ Set appropriate retention periods"
echo "   ✓ Use traffic filtering (REJECT only for security)"
echo "   ✓ Consider S3 for historical data"
echo "   ✓ Monitor log volume and costs regularly"
echo ""

echo "4. Security Monitoring:"
echo "   ✓ Set up alerts for rejected connections"
echo "   ✓ Monitor for port scanning patterns"
echo "   ✓ Track unusual protocol usage"
echo "   ✓ Detect potential data exfiltration"
echo ""

echo "5. Performance Analysis:"
echo "   ✓ Monitor bandwidth utilization"
echo "   ✓ Identify top talkers and applications"
echo "   ✓ Track connection patterns"
echo "   ✓ Analyze latency indicators"
echo ""

echo "6. Compliance and Auditing:"
echo "   ✓ Enable flow logs for compliance requirements"
echo "   ✓ Maintain proper retention for audit trails"
echo "   ✓ Document flow log configuration"
echo "   ✓ Regular review of flow log policies"

EOF

chmod +x flowlog-best-practices.sh
./flowlog-best-practices.sh
```

## Troubleshooting Flow Logs

| Issue | Symptoms | Diagnosis | Solution |
|-------|----------|-----------|----------|
| No flow log data | Empty log groups | Check IAM permissions | Verify flow logs role has proper permissions |
| Incomplete data | Missing traffic | Check flow log scope | Ensure flow logs cover required resources |
| High costs | Unexpected charges | Review log volume | Optimize retention and filtering |
| Delayed data | Old timestamps | Check log delivery | Verify log delivery configuration |
| Missing rejected traffic | No REJECT entries | Security group rules | Ensure traffic is actually being rejected |

## Key Takeaways
- Flow logs provide comprehensive network traffic visibility
- Multiple destination options optimize for different use cases
- Custom formats enable detailed analysis requirements
- Proper filtering reduces costs while maintaining security visibility
- CloudWatch Insights enables powerful log analysis
- Automated alerting detects security and performance issues
- Cost optimization requires balancing detail with expenses

## Best Practices Summary
- **Scope Planning**: Choose appropriate scope based on monitoring needs
- **Cost Management**: Use retention policies and destination optimization
- **Security Focus**: Monitor rejected traffic and unusual patterns
- **Performance Insights**: Track bandwidth and connection patterns
- **Automation**: Set up alerts and automated analysis
- **Compliance**: Maintain proper retention for audit requirements

## Additional Resources
- [VPC Flow Logs Documentation](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)
- [CloudWatch Logs Insights](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/AnalyzingLogData.html)
- [Flow Logs Troubleshooting](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-troubleshooting.html)

## Exam Tips
- Flow logs capture metadata, not packet contents
- Default capture window is approximately 10-15 minutes
- Flow logs don't capture DHCP, DNS resolver, Windows license activation
- Custom format provides additional fields for analysis
- Flow logs can be configured at VPC, subnet, or ENI level
- Rejected traffic appears as action=REJECT in flow logs
- Flow logs are eventual consistency - not real-time