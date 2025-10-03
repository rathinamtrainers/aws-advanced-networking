# Lab: AWS Gateway Load Balancer (GWLB) Deep Dive - Hands-On

## Overview

AWS Gateway Load Balancer (GWLB) enables you to deploy, scale, and manage third-party virtual appliances such as firewalls, intrusion detection/prevention systems (IDS/IPS), and deep packet inspection systems. This hands-on lab demonstrates how to set up GWLB with network security appliances for inspecting traffic.

## Learning Objectives

By completing this lab, you will:
- Understand GWLB architecture and GENEVE protocol
- Deploy Gateway Load Balancer with target appliances
- Configure GWLB endpoints for traffic inspection
- Implement traffic routing through security appliances
- Test end-to-end traffic flow and inspection
- Monitor and troubleshoot GWLB deployments

## Prerequisites

- AWS account with appropriate permissions
- Understanding of VPC networking concepts
- Familiarity with EC2 instances
- Basic knowledge of routing and security groups
- AWS CLI configured

## Lab Architecture

```
                                Internet
                                    │
                                    │
                            ┌───────▼────────┐
                            │ Internet Gateway│
                            └───────┬────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │       Application VPC         │
                    │                               │
                    │  ┌─────────────────────────┐  │
                    │  │   Public Subnet         │  │
                    │  │   ┌────────────┐        │  │
                    │  │   │ Web Server │        │  │
                    │  │   └─────┬──────┘        │  │
                    │  └─────────┼───────────────┘  │
                    │            │                  │
                    │  ┌─────────▼───────────────┐  │
                    │  │  GWLB Endpoint (ENI)    │  │
                    │  └─────────┬───────────────┘  │
                    └────────────┼──────────────────┘
                                 │ (GENEVE)
                                 │
                    ┌────────────▼──────────────────┐
                    │    Security VPC               │
                    │                               │
                    │  ┌─────────────────────────┐  │
                    │  │ Gateway Load Balancer   │  │
                    │  └──────┬──────────┬───────┘  │
                    │         │          │          │
                    │  ┌──────▼────┐ ┌──▼────────┐  │
                    │  │ Firewall  │ │ Firewall  │  │
                    │  │ Instance  │ │ Instance  │  │
                    │  │    AZ1    │ │    AZ2    │  │
                    │  └───────────┘ └───────────┘  │
                    └───────────────────────────────┘

Traffic Flow:
1. Internet → IGW → Application VPC
2. Routing table sends traffic to GWLB Endpoint
3. GWLB Endpoint → Gateway Load Balancer (GENEVE encapsulation)
4. GWLB distributes to Firewall instances
5. Firewall inspects and returns traffic
6. GWLB Endpoint → Web Server
```

## Part 1: Setup Security VPC with GWLB

### Step 1: Create Security VPC

#### AWS Console UI Steps:

1. Open **VPC** console
2. Click **"Create VPC"**
3. **VPC Settings:**
   - Resources to create: **VPC only**
   - Name tag: `Security-VPC`
   - IPv4 CIDR: `10.1.0.0/16`
   - IPv6 CIDR: No IPv6 CIDR block
   - Tenancy: Default
4. Click **"Create VPC"**

#### AWS CLI Verification Command:
```bash
aws ec2 describe-vpcs \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=Security-VPC" \
  --query 'Vpcs[*].[VpcId,CidrBlock,State]' \
  --output table
```

---

### Step 2: Create Subnets for GWLB

#### AWS Console UI Steps:

1. In VPC console, click **"Subnets"** → **"Create subnet"**
2. **VPC:** Select `Security-VPC`
3. **Subnet 1 (AZ1):**
   - Subnet name: `Security-GWLB-Subnet-AZ1`
   - Availability Zone: `us-east-1a`
   - IPv4 CIDR: `10.1.1.0/24`
4. Click **"Add new subnet"**
5. **Subnet 2 (AZ2):**
   - Subnet name: `Security-GWLB-Subnet-AZ2`
   - Availability Zone: `us-east-1b`
   - IPv4 CIDR: `10.1.2.0/24`
6. Click **"Create subnet"**

#### AWS CLI Verification Command:
```bash
aws ec2 describe-subnets \
  --region us-east-1 \
  --filters "Name=vpc-id,Values=<security-vpc-id>" \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

---

### Step 3: Create Security Groups for Firewall Appliances

#### AWS Console UI Steps:

1. Go to **EC2** → **Security Groups** → **"Create security group"**
2. **Basic details:**
   - Security group name: `Firewall-Appliance-SG`
   - Description: `Security group for firewall appliances`
   - VPC: Select `Security-VPC`
3. **Inbound rules:**
   - Type: **All traffic**
   - Source: **Custom** `10.0.0.0/8` (to allow traffic from application VPC)
4. **Outbound rules:**
   - Type: **All traffic**
   - Destination: **Anywhere-IPv4** `0.0.0.0/0`
5. Click **"Create security group"**

#### AWS CLI Verification Command:
```bash
aws ec2 describe-security-groups \
  --region us-east-1 \
  --filters "Name=group-name,Values=Firewall-Appliance-SG" \
  --query 'SecurityGroups[*].[GroupId,GroupName,VpcId]' \
  --output table
```

---

### Step 4: Launch Firewall Appliance Instances

#### AWS Console UI Steps:

**Instance 1 (AZ1):**

1. Go to **EC2** → **Instances** → **"Launch instances"**
2. **Name:** `Firewall-Appliance-AZ1`
3. **AMI:** Select **Amazon Linux 2023 AMI**
4. **Instance type:** `t3.medium`
5. **Key pair:** Create or select existing key pair
6. **Network settings:**
   - VPC: `Security-VPC`
   - Subnet: `Security-GWLB-Subnet-AZ1`
   - Auto-assign public IP: **Disable**
   - Security group: Select `Firewall-Appliance-SG`
7. **Advanced details:**
   - Scroll to **User data** and paste:

```bash
#!/bin/bash
# Enable IP forwarding
echo "net.ipv4.ip_forward = 1" >> /etc/sysctl.conf
sysctl -p

# Install iptables
yum install -y iptables-services
systemctl enable iptables
systemctl start iptables

# Configure iptables to log and forward traffic
iptables -F
iptables -t nat -F

# Log all forwarded traffic
iptables -A FORWARD -j LOG --log-prefix "GWLB-FIREWALL: " --log-level 4

# Accept all traffic (simple pass-through for demo)
iptables -A FORWARD -j ACCEPT
iptables -P FORWARD ACCEPT

# Save rules
service iptables save

# Install and configure GENEVE for GWLB
# Install required packages
yum install -y tcpdump nc

# Create startup script for GENEVE
cat > /usr/local/bin/gwlb-geneve.sh << 'EOF'
#!/bin/bash
# This script handles GENEVE encapsulated traffic from GWLB
# In production, use proper GENEVE handling with network appliance software

# Enable GENEVE module
modprobe geneve

# Log traffic
tcpdump -i any -n -v 'udp port 6081' -w /var/log/geneve.pcap &

echo "GWLB Firewall Appliance Started"
EOF

chmod +x /usr/local/bin/gwlb-geneve.sh
/usr/local/bin/gwlb-geneve.sh

# Create simple health check endpoint
mkdir -p /var/www/html
echo "OK" > /var/www/html/health.html
python3 -m http.server 80 --directory /var/www/html &
```

8. Click **"Launch instance"**

**Repeat for Instance 2 (AZ2):**
- Name: `Firewall-Appliance-AZ2`
- Subnet: `Security-GWLB-Subnet-AZ2`
- Same configuration as Instance 1

#### AWS CLI Verification Command:
```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=Firewall-Appliance-*" "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,Tags[?Key==`Name`].Value|[0],PrivateIpAddress,State.Name]' \
  --output table
```

---

### Step 5: Create Target Group for GWLB

#### AWS Console UI Steps:

1. Go to **EC2** → **Target Groups** → **"Create target group"**
2. **Choose a target type:** **Instances**
3. **Target group name:** `GWLB-Firewall-TG`
4. **Protocol:** **GENEVE**
5. **Port:** `6081`
6. **VPC:** Select `Security-VPC`
7. **Health checks:**
   - Protocol: **HTTP**
   - Path: `/health.html`
   - Port: `80`
   - Healthy threshold: `2`
   - Unhealthy threshold: `2`
   - Timeout: `5 seconds`
   - Interval: `10 seconds`
8. Click **"Next"**
9. **Register targets:**
   - Select both firewall instances: `Firewall-Appliance-AZ1` and `Firewall-Appliance-AZ2`
   - Click **"Include as pending below"**
10. Click **"Create target group"**

#### AWS CLI Verification Command:
```bash
aws elbv2 describe-target-groups \
  --region us-east-1 \
  --names GWLB-Firewall-TG \
  --query 'TargetGroups[*].[TargetGroupName,Protocol,Port,VpcId]' \
  --output table
```

**Check target health:**
```bash
aws elbv2 describe-target-health \
  --region us-east-1 \
  --target-group-arn <target-group-arn> \
  --query 'TargetHealthDescriptions[*].[Target.Id,TargetHealth.State,TargetHealth.Reason]' \
  --output table
```

---

### Step 6: Create Gateway Load Balancer

#### AWS Console UI Steps:

1. Go to **EC2** → **Load Balancers** → **"Create Load Balancer"**
2. Select **"Gateway Load Balancer"** → **"Create"**
3. **Basic configuration:**
   - Load balancer name: `Security-GWLB`
   - IP address type: **IPv4**
4. **Network mapping:**
   - VPC: `Security-VPC`
   - Mappings: Select both AZs
     - ✓ `us-east-1a` → Select `Security-GWLB-Subnet-AZ1`
     - ✓ `us-east-1b` → Select `Security-GWLB-Subnet-AZ2`
5. **Listener:**
   - Default action: Forward to `GWLB-Firewall-TG`
6. **Tags:**
   - Key: `Name`, Value: `Security-GWLB`
7. Click **"Create load balancer"**

#### AWS CLI Verification Command:
```bash
aws elbv2 describe-load-balancers \
  --region us-east-1 \
  --names Security-GWLB \
  --query 'LoadBalancers[*].[LoadBalancerName,Type,State.Code,VpcId,LoadBalancerArn]' \
  --output table
```

---

## Part 2: Setup Application VPC with GWLB Endpoint

### Step 7: Create Application VPC

#### AWS Console UI Steps:

1. Go to **VPC** console → **"Create VPC"**
2. **VPC Settings:**
   - Resources to create: **VPC only**
   - Name tag: `Application-VPC`
   - IPv4 CIDR: `10.0.0.0/16`
3. Click **"Create VPC"**

#### AWS CLI Verification Command:
```bash
aws ec2 describe-vpcs \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=Application-VPC" \
  --query 'Vpcs[*].[VpcId,CidrBlock,State]' \
  --output table
```

---

### Step 8: Create Application Subnets

#### AWS Console UI Steps:

1. Click **"Subnets"** → **"Create subnet"**
2. **VPC:** Select `Application-VPC`
3. **Public Subnet:**
   - Subnet name: `App-Public-Subnet-AZ1`
   - Availability Zone: `us-east-1a`
   - IPv4 CIDR: `10.0.1.0/24`
4. Click **"Add new subnet"**
5. **GWLB Endpoint Subnet:**
   - Subnet name: `App-GWLB-Endpoint-Subnet-AZ1`
   - Availability Zone: `us-east-1a`
   - IPv4 CIDR: `10.0.2.0/24`
6. Click **"Create subnet"**

#### AWS CLI Verification Command:
```bash
aws ec2 describe-subnets \
  --region us-east-1 \
  --filters "Name=vpc-id,Values=<application-vpc-id>" \
  --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,Tags[?Key==`Name`].Value|[0]]' \
  --output table
```

---

### Step 9: Create and Attach Internet Gateway

#### AWS Console UI Steps:

1. Go to **VPC** → **Internet Gateways** → **"Create internet gateway"**
2. **Name tag:** `App-IGW`
3. Click **"Create internet gateway"**
4. Select the IGW → **Actions** → **"Attach to VPC"**
5. Select `Application-VPC` → **"Attach internet gateway"**

#### AWS CLI Verification Command:
```bash
aws ec2 describe-internet-gateways \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=App-IGW" \
  --query 'InternetGateways[*].[InternetGatewayId,Attachments[0].VpcId,Attachments[0].State]' \
  --output table
```

---

### Step 10: Create GWLB Endpoint Service

#### AWS Console UI Steps:

1. Go to **VPC** → **Endpoint Services** → **"Create endpoint service"**
2. **Load balancer type:** **Gateway Load Balancer**
3. **Available load balancers:** Select `Security-GWLB`
4. **Acceptance required:** Uncheck (for this lab)
5. **Tags:**
   - Key: `Name`, Value: `GWLB-Endpoint-Service`
6. Click **"Create"**
7. **IMPORTANT:** Copy the **Service name** (e.g., `com.amazonaws.vpce.us-east-1.vpce-svc-xxxxx`)

#### AWS CLI Verification Command:
```bash
aws ec2 describe-vpc-endpoint-service-configurations \
  --region us-east-1 \
  --query 'ServiceConfigurations[?GatewayLoadBalancerArns[0]!=`null`].[ServiceId,ServiceName,ServiceState]' \
  --output table
```

---

### Step 11: Create GWLB Endpoint in Application VPC

#### AWS Console UI Steps:

1. Go to **VPC** → **Endpoints** → **"Create endpoint"**
2. **Service category:** **Other endpoint services**
3. **Service name:** Paste the service name from Step 10
4. Click **"Verify service"** (should show "Service name verified")
5. **VPC:** Select `Application-VPC`
6. **Subnets:**
   - Select `us-east-1a` → `App-GWLB-Endpoint-Subnet-AZ1`
7. **Tags:**
   - Key: `Name`, Value: `App-GWLB-Endpoint`
8. Click **"Create endpoint"**
9. Wait for status to become **"Available"**

#### AWS CLI Verification Command:
```bash
aws ec2 describe-vpc-endpoints \
  --region us-east-1 \
  --filters "Name=vpc-endpoint-type,Values=GatewayLoadBalancer" \
  --query 'VpcEndpoints[*].[VpcEndpointId,ServiceName,State,VpcId]' \
  --output table
```

---

### Step 12: Configure Route Tables for Traffic Inspection

#### Create Route Tables:

**1. IGW Route Table (Edge routing):**

AWS Console UI Steps:
1. Go to **VPC** → **Route Tables** → **"Create route table"**
2. **Name:** `App-IGW-RouteTable`
3. **VPC:** `Application-VPC`
4. Click **"Create route table"**
5. **Add routes:**
   - Click **"Edit routes"** → **"Add route"**
   - Destination: `10.0.1.0/24` (public subnet CIDR)
   - Target: Select **Gateway Load Balancer Endpoint** → Select `App-GWLB-Endpoint`
   - Click **"Save changes"**
6. **Associate with IGW:**
   - Go to **"Edge associations"** tab
   - Click **"Edit edge associations"**
   - Select `App-IGW`
   - Click **"Save changes"**

**2. Public Subnet Route Table:**

AWS Console UI Steps:
1. **"Create route table"**
2. **Name:** `App-Public-RouteTable`
3. **VPC:** `Application-VPC`
4. Click **"Create route table"**
5. **Add routes:**
   - Click **"Edit routes"** → **"Add route"**
   - Destination: `0.0.0.0/0`
   - Target: Select **Gateway Load Balancer Endpoint** → Select `App-GWLB-Endpoint`
   - Click **"Save changes"**
6. **Subnet associations:**
   - Go to **"Subnet associations"** tab
   - Click **"Edit subnet associations"**
   - Select `App-Public-Subnet-AZ1`
   - Click **"Save associations"**

**3. GWLB Endpoint Route Table:**

AWS Console UI Steps:
1. **"Create route table"**
2. **Name:** `App-GWLB-Endpoint-RouteTable`
3. **VPC:** `Application-VPC`
4. Click **"Create route table"**
5. **Add routes:**
   - Click **"Edit routes"** → **"Add route"**
   - Destination: `0.0.0.0/0`
   - Target: Select **Internet Gateway** → Select `App-IGW`
   - Click **"Save changes"**
6. **Subnet associations:**
   - Go to **"Subnet associations"** tab
   - Click **"Edit subnet associations"**
   - Select `App-GWLB-Endpoint-Subnet-AZ1`
   - Click **"Save associations"**

#### AWS CLI Verification Command:
```bash
# List all route tables in Application VPC
aws ec2 describe-route-tables \
  --region us-east-1 \
  --filters "Name=vpc-id,Values=<application-vpc-id>" \
  --query 'RouteTables[*].[RouteTableId,Tags[?Key==`Name`].Value|[0],Routes[*].[DestinationCidrBlock,GatewayId,VpcEndpointId]|[0]]' \
  --output table
```

---

## Part 3: Deploy and Test Application

### Step 13: Create Security Group for Web Server

#### AWS Console UI Steps:

1. Go to **EC2** → **Security Groups** → **"Create security group"**
2. **Security group name:** `WebServer-SG`
3. **Description:** `Security group for web server`
4. **VPC:** Select `Application-VPC`
5. **Inbound rules:**
   - Type: **HTTP**, Port: `80`, Source: **Anywhere-IPv4** `0.0.0.0/0`
   - Type: **SSH**, Port: `22`, Source: **My IP** (your IP)
6. **Outbound rules:**
   - Type: **All traffic**, Destination: **Anywhere-IPv4** `0.0.0.0/0`
7. Click **"Create security group"**

#### AWS CLI Verification Command:
```bash
aws ec2 describe-security-groups \
  --region us-east-1 \
  --filters "Name=group-name,Values=WebServer-SG" \
  --query 'SecurityGroups[*].[GroupId,GroupName,VpcId]' \
  --output table
```

---

### Step 14: Launch Web Server Instance

#### AWS Console UI Steps:

1. Go to **EC2** → **Instances** → **"Launch instances"**
2. **Name:** `WebServer`
3. **AMI:** **Amazon Linux 2023 AMI**
4. **Instance type:** `t3.micro`
5. **Key pair:** Create or select existing
6. **Network settings:**
   - VPC: `Application-VPC`
   - Subnet: `App-Public-Subnet-AZ1`
   - Auto-assign public IP: **Enable**
   - Security group: Select `WebServer-SG`
7. **Advanced details → User data:**

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd

# Create simple web page
cat > /var/www/html/index.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>GWLB Demo</title>
</head>
<body>
    <h1>Gateway Load Balancer Demo</h1>
    <p>This traffic is being inspected by GWLB!</p>
    <p>Server IP: <span id="ip"></span></p>
    <script>
        fetch('http://169.254.169.254/latest/meta-data/local-ipv4')
            .then(response => response.text())
            .then(ip => document.getElementById('ip').textContent = ip);
    </script>
</body>
</html>
EOF

# Log access
echo "Web server started at $(date)" >> /var/log/gwlb-demo.log
```

8. Click **"Launch instance"**

#### AWS CLI Verification Command:
```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=WebServer" "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].[InstanceId,PublicIpAddress,PrivateIpAddress,State.Name]' \
  --output table
```

---

## Part 4: Testing and Validation

### Step 15: Test Traffic Flow Through GWLB

#### Test 1: Access Web Server from Internet

**Get the public IP of web server:**
```bash
aws ec2 describe-instances \
  --region us-east-1 \
  --filters "Name=tag:Name,Values=WebServer" \
  --query 'Reservations[*].Instances[*].PublicIpAddress' \
  --output text
```

**Test from your browser or curl:**
```bash
curl http://<web-server-public-ip>
```

Expected: You should see the web page content.

#### Test 2: Verify Traffic is Going Through GWLB

**Check firewall logs on Firewall Appliance:**

SSH to one of the firewall instances (you'll need Session Manager or a bastion):

```bash
# Check iptables logs
sudo tail -f /var/log/messages | grep "GWLB-FIREWALL"

# Check GENEVE traffic
sudo tcpdump -i any -n 'udp port 6081' -v
```

You should see traffic being logged.

#### Test 3: Verify Target Health

```bash
# Check GWLB target health
aws elbv2 describe-target-health \
  --region us-east-1 \
  --target-group-arn <target-group-arn>
```

Expected: Both targets should show "healthy"

#### Test 4: Check VPC Endpoint Status

```bash
aws ec2 describe-vpc-endpoints \
  --region us-east-1 \
  --vpc-endpoint-ids <gwlb-endpoint-id> \
  --query 'VpcEndpoints[*].[VpcEndpointId,State,ServiceName]' \
  --output table
```

Expected: State should be "available"

---

## Monitoring and Troubleshooting

### CloudWatch Metrics

**Monitor GWLB metrics:**

1. Go to **CloudWatch** → **Metrics** → **All metrics**
2. Select **GatewayELB**
3. View metrics:
   - `ActiveFlowCount` - Number of active flows
   - `ProcessedBytes` - Bytes processed
   - `HealthyHostCount` - Number of healthy targets
   - `UnHealthyHostCount` - Number of unhealthy targets

**CLI Command:**
```bash
aws cloudwatch get-metric-statistics \
  --namespace AWS/GatewayELB \
  --metric-name HealthyHostCount \
  --dimensions Name=LoadBalancer,Value=<gwlb-arn-suffix> Name=TargetGroup,Value=<tg-arn-suffix> \
  --start-time $(date -u -d '10 minutes ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 300 \
  --statistics Average \
  --region us-east-1
```

### VPC Flow Logs

**Enable Flow Logs for troubleshooting:**

```bash
# Create flow log for Application VPC
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids <application-vpc-id> \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/application-vpc-flowlogs \
  --region us-east-1
```

### Common Issues and Solutions

| Issue | Symptom | Solution |
|-------|---------|----------|
| Web server not accessible | Connection timeout | Check security groups, route tables, IGW attachment |
| Traffic not inspected | No logs on firewall | Verify GWLB endpoint routing, check route table associations |
| Unhealthy targets | Targets show unhealthy | Check health check port (80), verify HTTP service running |
| GENEVE traffic not received | No GENEVE packets | Ensure security group allows UDP 6081 from GWLB |

---

## Understanding GENEVE Protocol

**GENEVE (Generic Network Virtualization Encapsulation):**

- Protocol used by GWLB to encapsulate traffic
- Uses UDP port 6081
- Adds metadata for traffic flow tracking
- Preserves original packet information

**Packet Structure:**
```
┌──────────────────────────────────────┐
│         Outer IP Header              │ (GWLB ↔ Appliance)
├──────────────────────────────────────┤
│         UDP Header (Port 6081)       │
├──────────────────────────────────────┤
│         GENEVE Header                │
│  - VNI (Virtual Network Identifier)  │
│  - Flow Hash                         │
│  - Metadata                          │
├──────────────────────────────────────┤
│         Original IP Packet           │ (Client ↔ Server)
│  - Original IP Header                │
│  - Original Payload                  │
└──────────────────────────────────────┘
```

---

## Advanced Configuration

### Adding More Firewall Rules

SSH to firewall appliance and add custom rules:

```bash
# Block specific IP
sudo iptables -I FORWARD -s <malicious-ip> -j DROP

# Rate limiting
sudo iptables -A FORWARD -p tcp --dport 80 -m limit --limit 100/s -j ACCEPT
sudo iptables -A FORWARD -p tcp --dport 80 -j DROP

# Log specific traffic
sudo iptables -A FORWARD -p tcp --dport 443 -j LOG --log-prefix "HTTPS-TRAFFIC: "

# Save rules
sudo service iptables save
```

### Multi-AZ High Availability

To add second AZ:

1. Create subnets in `us-east-1b` for both VPCs
2. Launch firewall instance in AZ2
3. Add AZ2 to GWLB network mapping
4. Create GWLB endpoint in AZ2
5. Update route tables for AZ2 subnets

---

## Cleanup

**Delete resources in reverse order:**

```bash
# 1. Terminate EC2 instances
aws ec2 terminate-instances \
  --instance-ids <web-server-id> <firewall-1-id> <firewall-2-id> \
  --region us-east-1

# Wait for termination
aws ec2 wait instance-terminated \
  --instance-ids <instance-ids> \
  --region us-east-1

# 2. Delete GWLB Endpoint
aws ec2 delete-vpc-endpoints \
  --vpc-endpoint-ids <gwlb-endpoint-id> \
  --region us-east-1

# 3. Delete GWLB Endpoint Service
aws ec2 delete-vpc-endpoint-service-configurations \
  --service-ids <service-id> \
  --region us-east-1

# 4. Delete Gateway Load Balancer
aws elbv2 delete-load-balancer \
  --load-balancer-arn <gwlb-arn> \
  --region us-east-1

# Wait 2 minutes, then delete target group
sleep 120
aws elbv2 delete-target-group \
  --target-group-arn <tg-arn> \
  --region us-east-1

# 5. Detach and delete IGW
aws ec2 detach-internet-gateway \
  --internet-gateway-id <igw-id> \
  --vpc-id <application-vpc-id> \
  --region us-east-1

aws ec2 delete-internet-gateway \
  --internet-gateway-id <igw-id> \
  --region us-east-1

# 6. Delete subnets
aws ec2 delete-subnet --subnet-id <subnet-id> --region us-east-1

# 7. Delete security groups (wait for ENIs to be deleted)
sleep 60
aws ec2 delete-security-group --group-id <sg-id> --region us-east-1

# 8. Delete VPCs
aws ec2 delete-vpc --vpc-id <application-vpc-id> --region us-east-1
aws ec2 delete-vpc --vpc-id <security-vpc-id> --region us-east-1

echo "Cleanup complete!"
```

---

## Key Takeaways

1. **GWLB Architecture:**
   - GWLB operates at Layer 3
   - Uses GENEVE protocol (UDP 6081) for encapsulation
   - Transparent to applications and end users

2. **Traffic Flow:**
   - Traffic is intercepted via routing tables
   - GWLB Endpoint encapsulates traffic with GENEVE
   - Security appliances inspect and return traffic
   - Original connection is preserved (stateful)

3. **High Availability:**
   - Deploy across multiple AZs
   - GWLB automatically distributes traffic
   - Health checks ensure only healthy appliances receive traffic

4. **Use Cases:**
   - Network firewalls (stateful inspection)
   - IDS/IPS systems
   - DDoS protection appliances
   - Deep packet inspection
   - Third-party security appliances

5. **Best Practices:**
   - Always deploy in multiple AZs
   - Monitor target health continuously
   - Use VPC Flow Logs for troubleshooting
   - Implement proper health checks
   - Test failover scenarios

---

## Additional Resources

- [AWS Gateway Load Balancer Documentation](https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/)
- [GENEVE Protocol RFC](https://datatracker.ietf.org/doc/html/rfc8926)
- [GWLB Best Practices](https://docs.aws.amazon.com/elasticloadbalancing/latest/gateway/gateway-load-balancer-best-practices.html)
- [Partner Appliances](https://aws.amazon.com/elasticloadbalancing/gateway-load-balancer/partners/)

## Next Steps

- Explore commercial security appliances (Palo Alto, Fortinet, etc.)
- Implement centralized inspection for multiple VPCs
- Set up automated scaling for appliances
- Integrate with AWS Security Hub
- Configure advanced routing scenarios

---

**Congratulations! You've successfully completed the GWLB Deep Dive lab!**
