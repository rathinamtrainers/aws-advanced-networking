# Topic 29: VPC Network Security and Advanced Access Control

## Prerequisites
- Completed Topic 28 (Advanced VPC Routing and Traffic Engineering)
- Understanding of network security concepts and threat models
- Knowledge of AWS security services and identity management
- Familiarity with security groups, NACLs, and VPC security features

## Learning Objectives
By the end of this topic, you will be able to:
- Design comprehensive network security architectures for VPCs
- Implement advanced security group and NACL strategies
- Configure VPC security monitoring and threat detection systems
- Deploy network segmentation and micro-segmentation solutions

## Theory

### VPC Network Security Architecture

#### Defense in Depth Strategy
```
Security Layer Model:
┌─────────────────────────────────────────────────────┐
│ Layer 7: Application Security (WAF, Authentication) │
├─────────────────────────────────────────────────────┤
│ Layer 6: Data Security (Encryption, DLP)           │
├─────────────────────────────────────────────────────┤
│ Layer 5: Identity & Access (IAM, SSO)              │
├─────────────────────────────────────────────────────┤
│ Layer 4: Network Security (Security Groups, NACLs) │
├─────────────────────────────────────────────────────┤
│ Layer 3: Perimeter Security (Firewalls, IPS)       │
├─────────────────────────────────────────────────────┤
│ Layer 2: Infrastructure Security (VPC, Subnets)    │
├─────────────────────────────────────────────────────┤
│ Layer 1: Physical Security (AWS Data Centers)      │
└─────────────────────────────────────────────────────┘
```

#### Network Security Components
- **Security Groups**: Stateful firewall at instance level
- **Network ACLs**: Stateless firewall at subnet level
- **VPC Flow Logs**: Network traffic monitoring and analysis
- **AWS WAF**: Web application firewall for Layer 7 protection
- **Network Firewall**: Managed firewall service for advanced filtering
- **GuardDuty**: Threat detection using machine learning

### Advanced Security Group Strategies

#### Micro-Segmentation Architecture
```
Application Tier Segmentation:
┌─────────────────┬─────────────────┬─────────────────┐
│   Web Tier SG   │   App Tier SG   │   DB Tier SG    │
├─────────────────┼─────────────────┼─────────────────┤
│ Port 80,443     │ Port 8080       │ Port 3306       │
│ Source: 0.0.0.0 │ Source: Web SG  │ Source: App SG  │
│ Protocol: TCP   │ Protocol: TCP   │ Protocol: TCP   │
└─────────────────┴─────────────────┴─────────────────┘
```

#### Dynamic Security Group Management
- **Tag-based Rules**: Automatic security group assignment based on tags
- **Service-based Segmentation**: Rules based on service discovery
- **Time-based Access**: Temporary access rules with automatic expiration
- **Conditional Access**: Context-aware security policies

### Network Access Control Lists (NACLs)

#### NACL vs Security Group Comparison
| Feature | Security Groups | Network ACLs |
|---------|----------------|--------------|
| **State** | Stateful | Stateless |
| **Level** | Instance | Subnet |
| **Rules** | Allow only | Allow and Deny |
| **Evaluation** | All rules | Ordered rules |
| **Default** | Deny all | Allow all |

#### Advanced NACL Patterns
```
NACL Rule Structure:
Rule # | Type  | Protocol | Port Range | Source/Dest    | Allow/Deny
100    | HTTP  | TCP      | 80         | 0.0.0.0/0     | Allow
110    | HTTPS | TCP      | 443        | 0.0.0.0/0     | Allow
120    | SSH   | TCP      | 22         | 10.0.0.0/8    | Allow
200    | ALL   | ALL      | ALL        | 192.168.1.0/24| Deny
32767  | ALL   | ALL      | ALL        | 0.0.0.0/0     | Deny
```

### VPC Security Monitoring

#### Flow Logs Analysis
```
Flow Log Fields:
version account-id interface-id srcaddr dstaddr srcport dstport 
protocol packets bytes windowstart windowend action flowlogstatus
```

#### Security Event Detection
- **Anomalous Traffic Patterns**: Unusual volume or destinations
- **Unauthorized Access Attempts**: Failed connection attempts
- **Data Exfiltration**: Large outbound data transfers
- **Lateral Movement**: East-west traffic patterns

## Lab Exercise: Comprehensive VPC Security Implementation

### Lab Duration: 360 minutes

### Step 1: Design Security Architecture
**Objective**: Plan comprehensive security strategy for multi-tier application

**Security Architecture Planning**:
```bash
# Create comprehensive security planning assessment
cat << 'EOF' > vpc-security-planning.sh
#!/bin/bash

echo "=== VPC Network Security Architecture Planning ==="
echo ""

echo "1. THREAT MODEL ANALYSIS"
echo "   External threats: DDoS/Web attacks/Data exfiltration"
echo "   Internal threats: Lateral movement/Privilege escalation"
echo "   Compliance requirements: SOC2/PCI DSS/HIPAA"
echo "   Data sensitivity: Public/Internal/Confidential/Restricted"
echo "   Business criticality: Critical/High/Medium/Low"
echo ""

echo "2. NETWORK SEGMENTATION STRATEGY"
echo "   Segmentation approach: Tier-based/Service-based/Zone-based"
echo "   Isolation requirements: Complete/Partial/Conditional"
echo "   Trust boundaries: DMZ/Internal/Secure zones"
echo "   Micro-segmentation: Application/Service/Container level"
echo "   Zero-trust implementation: Yes/No/Partial"
echo ""

echo "3. ACCESS CONTROL REQUIREMENTS"
echo "   Principle of least privilege: Strict/Moderate/Flexible"
echo "   Just-in-time access: Required/Optional/Not needed"
echo "   Multi-factor authentication: Required/Recommended"
echo "   Network access control: Mandatory/Advisory"
echo "   Audit and compliance: Full logging/Selective"
echo ""

echo "4. MONITORING AND DETECTION"
echo "   Real-time monitoring: Required/Preferred/Optional"
echo "   Threat detection: ML-based/Rule-based/Hybrid"
echo "   Incident response: Automated/Manual/Hybrid"
echo "   Security analytics: Cloud-native/Third-party"
echo "   Alert integration: SIEM/SOC/Internal tools"
echo ""

echo "5. SECURITY CONTROLS IMPLEMENTATION"
echo "   Firewall strategy: Distributed/Centralized/Hybrid"
echo "   IDS/IPS deployment: Required/Optional"
echo "   DLP implementation: Network/Endpoint/Both"
echo "   Encryption requirements: In-transit/At-rest/Both"
echo "   Certificate management: Automated/Manual"
echo ""

echo "6. OPERATIONAL SECURITY"
echo "   Security automation: High/Medium/Low"
echo "   Change management: Strict/Standard/Flexible"
echo "   Security testing: Continuous/Periodic/Ad-hoc"
echo "   Vulnerability management: Proactive/Reactive"
echo "   Disaster recovery: RPO/RTO requirements"

EOF

chmod +x vpc-security-planning.sh
./vpc-security-planning.sh
```

### Step 2: Implement Advanced Security Groups
**Objective**: Configure sophisticated security group architecture

**Advanced Security Group Implementation**:
```bash
# Create advanced security group architecture
cat << 'EOF' > implement-advanced-security-groups.sh
#!/bin/bash

# Check if VPC config exists from previous labs
if [ ! -f "advanced-routing-config.env" ]; then
    echo "Error: VPC configuration not found. Please run previous labs first."
    exit 1
fi

source advanced-routing-config.env

echo "=== Implementing Advanced Security Group Architecture ==="
echo ""

echo "1. CREATING TIERED SECURITY GROUPS"

# Web Tier Security Groups
WEB_SG_PUBLIC=$(aws ec2 create-security-group \
    --group-name Web-Tier-Public-SG \
    --description "Public web tier security group with internet access" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Web-Tier-Public-SG},{Key=Tier,Value=Web},{Key=Exposure,Value=Public}]" \
    --query 'GroupId' \
    --output text)

WEB_SG_PRIVATE=$(aws ec2 create-security-group \
    --group-name Web-Tier-Private-SG \
    --description "Private web tier security group for load balancer targets" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Web-Tier-Private-SG},{Key=Tier,Value=Web},{Key=Exposure,Value=Private}]" \
    --query 'GroupId' \
    --output text)

echo "✅ Web tier security groups created: $WEB_SG_PUBLIC, $WEB_SG_PRIVATE"

# Application Tier Security Groups
APP_SG_FRONTEND=$(aws ec2 create-security-group \
    --group-name App-Tier-Frontend-SG \
    --description "Application frontend services security group" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=App-Tier-Frontend-SG},{Key=Tier,Value=Application},{Key=Layer,Value=Frontend}]" \
    --query 'GroupId' \
    --output text)

APP_SG_BACKEND=$(aws ec2 create-security-group \
    --group-name App-Tier-Backend-SG \
    --description "Application backend services security group" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=App-Tier-Backend-SG},{Key=Tier,Value=Application},{Key=Layer,Value=Backend}]" \
    --query 'GroupId' \
    --output text)

APP_SG_API=$(aws ec2 create-security-group \
    --group-name App-Tier-API-SG \
    --description "Application API services security group" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=App-Tier-API-SG},{Key=Tier,Value=Application},{Key=Layer,Value=API}]" \
    --query 'GroupId' \
    --output text)

echo "✅ Application tier security groups created: $APP_SG_FRONTEND, $APP_SG_BACKEND, $APP_SG_API"

# Database Tier Security Groups
DB_SG_READ=$(aws ec2 create-security-group \
    --group-name Database-Read-SG \
    --description "Database read-only access security group" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Database-Read-SG},{Key=Tier,Value=Database},{Key=Access,Value=ReadOnly}]" \
    --query 'GroupId' \
    --output text)

DB_SG_WRITE=$(aws ec2 create-security-group \
    --group-name Database-Write-SG \
    --description "Database read-write access security group" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Database-Write-SG},{Key=Tier,Value=Database},{Key=Access,Value=ReadWrite}]" \
    --query 'GroupId' \
    --output text)

DB_SG_ADMIN=$(aws ec2 create-security-group \
    --group-name Database-Admin-SG \
    --description "Database administrative access security group" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Database-Admin-SG},{Key=Tier,Value=Database},{Key=Access,Value=Admin}]" \
    --query 'GroupId' \
    --output text)

echo "✅ Database tier security groups created: $DB_SG_READ, $DB_SG_WRITE, $DB_SG_ADMIN"

# Management and Monitoring Security Groups
MGMT_SG_BASTION=$(aws ec2 create-security-group \
    --group-name Management-Bastion-SG \
    --description "Bastion host security group for secure access" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Management-Bastion-SG},{Key=Tier,Value=Management},{Key=Purpose,Value=Bastion}]" \
    --query 'GroupId' \
    --output text)

MGMT_SG_MONITORING=$(aws ec2 create-security-group \
    --group-name Management-Monitoring-SG \
    --description "Monitoring and logging services security group" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Management-Monitoring-SG},{Key=Tier,Value=Management},{Key=Purpose,Value=Monitoring}]" \
    --query 'GroupId' \
    --output text)

echo "✅ Management security groups created: $MGMT_SG_BASTION, $MGMT_SG_MONITORING"

echo ""
echo "2. CONFIGURING SECURITY GROUP RULES"

# Web Tier Rules
echo "Configuring Web Tier rules..."

# Public Web SG - Internet access
aws ec2 authorize-security-group-ingress \
    --group-id $WEB_SG_PUBLIC \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0

aws ec2 authorize-security-group-ingress \
    --group-id $WEB_SG_PUBLIC \
    --protocol tcp \
    --port 443 \
    --cidr 0.0.0.0/0

# Private Web SG - Load balancer access only
aws ec2 authorize-security-group-ingress \
    --group-id $WEB_SG_PRIVATE \
    --protocol tcp \
    --port 80 \
    --source-group $WEB_SG_PUBLIC

aws ec2 authorize-security-group-ingress \
    --group-id $WEB_SG_PRIVATE \
    --protocol tcp \
    --port 443 \
    --source-group $WEB_SG_PUBLIC

echo "✅ Web tier rules configured"

# Application Tier Rules
echo "Configuring Application Tier rules..."

# Frontend App SG - Web tier access
aws ec2 authorize-security-group-ingress \
    --group-id $APP_SG_FRONTEND \
    --protocol tcp \
    --port 8080 \
    --source-group $WEB_SG_PRIVATE

# Backend App SG - Frontend access
aws ec2 authorize-security-group-ingress \
    --group-id $APP_SG_BACKEND \
    --protocol tcp \
    --port 8081 \
    --source-group $APP_SG_FRONTEND

# API SG - Frontend and Backend access
aws ec2 authorize-security-group-ingress \
    --group-id $APP_SG_API \
    --protocol tcp \
    --port 8082 \
    --source-group $APP_SG_FRONTEND

aws ec2 authorize-security-group-ingress \
    --group-id $APP_SG_API \
    --protocol tcp \
    --port 8082 \
    --source-group $APP_SG_BACKEND

echo "✅ Application tier rules configured"

# Database Tier Rules
echo "Configuring Database Tier rules..."

# Read-only database access from application tiers
aws ec2 authorize-security-group-ingress \
    --group-id $DB_SG_READ \
    --protocol tcp \
    --port 3306 \
    --source-group $APP_SG_FRONTEND

aws ec2 authorize-security-group-ingress \
    --group-id $DB_SG_READ \
    --protocol tcp \
    --port 3306 \
    --source-group $APP_SG_BACKEND

# Read-write database access from backend and API tiers
aws ec2 authorize-security-group-ingress \
    --group-id $DB_SG_WRITE \
    --protocol tcp \
    --port 3306 \
    --source-group $APP_SG_BACKEND

aws ec2 authorize-security-group-ingress \
    --group-id $DB_SG_WRITE \
    --protocol tcp \
    --port 3306 \
    --source-group $APP_SG_API

# Admin database access from management tier only
aws ec2 authorize-security-group-ingress \
    --group-id $DB_SG_ADMIN \
    --protocol tcp \
    --port 3306 \
    --source-group $MGMT_SG_BASTION

echo "✅ Database tier rules configured"

# Management Tier Rules
echo "Configuring Management Tier rules..."

# Bastion host - SSH access from specific IP ranges
aws ec2 authorize-security-group-ingress \
    --group-id $MGMT_SG_BASTION \
    --protocol tcp \
    --port 22 \
    --cidr 203.0.113.0/24  # Replace with your admin IP range

# Monitoring - Access from all internal tiers
for sg in $APP_SG_FRONTEND $APP_SG_BACKEND $APP_SG_API; do
    aws ec2 authorize-security-group-ingress \
        --group-id $MGMT_SG_MONITORING \
        --protocol tcp \
        --port 9090 \
        --source-group $sg
done

echo "✅ Management tier rules configured"

echo ""
echo "3. IMPLEMENTING DYNAMIC SECURITY GROUP MANAGEMENT"

cat << 'DYNAMIC_SG' > dynamic-security-group-manager.py
#!/usr/bin/env python3

import boto3
import json
from datetime import datetime, timedelta

class DynamicSecurityGroupManager:
    def __init__(self, vpc_id):
        self.vpc_id = vpc_id
        self.ec2 = boto3.client('ec2')
        self.ec2_resource = boto3.resource('ec2')
    
    def implement_time_based_access(self, sg_id, rules):
        """Implement time-based security group rules"""
        print(f"Implementing time-based access for SG: {sg_id}")
        
        current_hour = datetime.now().hour
        
        # Business hours: 6 AM to 10 PM
        if 6 <= current_hour <= 22:
            policy = "business_hours"
            self.apply_business_hours_rules(sg_id, rules)
        else:
            policy = "after_hours"
            self.apply_after_hours_rules(sg_id, rules)
        
        print(f"Applied {policy} policy to security group")
    
    def apply_business_hours_rules(self, sg_id, rules):
        """Apply business hours security rules"""
        business_rules = [
            {'port': 22, 'source': '10.0.0.0/8', 'description': 'SSH from corporate network'},
            {'port': 80, 'source': '0.0.0.0/0', 'description': 'HTTP from anywhere'},
            {'port': 443, 'source': '0.0.0.0/0', 'description': 'HTTPS from anywhere'},
            {'port': 3389, 'source': '10.0.0.0/8', 'description': 'RDP from corporate network'}
        ]
        
        print("Business Hours Rules:")
        for rule in business_rules:
            print(f"  Allow {rule['port']} from {rule['source']} - {rule['description']}")
    
    def apply_after_hours_rules(self, sg_id, rules):
        """Apply after hours security rules (more restrictive)"""
        after_hours_rules = [
            {'port': 22, 'source': '203.0.113.0/24', 'description': 'SSH from VPN only'},
            {'port': 80, 'source': '0.0.0.0/0', 'description': 'HTTP from anywhere'},
            {'port': 443, 'source': '0.0.0.0/0', 'description': 'HTTPS from anywhere'}
        ]
        
        print("After Hours Rules (Restricted):")
        for rule in after_hours_rules:
            print(f"  Allow {rule['port']} from {rule['source']} - {rule['description']}")
    
    def implement_conditional_access(self, sg_id, conditions):
        """Implement conditional access based on various factors"""
        print(f"\nImplementing conditional access for SG: {sg_id}")
        
        # Example conditions
        conditions_to_check = {
            'source_location': 'Check if source IP is from approved geographic region',
            'threat_level': 'Check current threat intelligence level',
            'user_authentication': 'Verify multi-factor authentication',
            'device_compliance': 'Check if device meets security policies'
        }
        
        print("Conditional Access Checks:")
        for condition, description in conditions_to_check.items():
            print(f"  {condition}: {description}")
    
    def implement_just_in_time_access(self, sg_id, duration_minutes=60):
        """Implement just-in-time access with automatic expiration"""
        print(f"\nImplementing JIT access for SG: {sg_id}")
        print(f"Access duration: {duration_minutes} minutes")
        
        # In practice, this would:
        # 1. Add temporary security group rule
        # 2. Schedule Lambda function to remove rule after duration
        # 3. Log access request and approval
        
        expiration_time = datetime.now() + timedelta(minutes=duration_minutes)
        print(f"Access will expire at: {expiration_time}")
        
        jit_config = {
            'sg_id': sg_id,
            'expiration': expiration_time.isoformat(),
            'approved_by': 'security-automation',
            'reason': 'Maintenance access request'
        }
        
        print("JIT Access Configuration:")
        print(json.dumps(jit_config, indent=2))
    
    def tag_based_security_management(self):
        """Implement tag-based security group management"""
        print("\nTag-Based Security Management:")
        
        tag_policies = {
            'Environment': {
                'Production': 'Strict security rules, limited access',
                'Staging': 'Moderate security rules, development access',
                'Development': 'Relaxed security rules, full access'
            },
            'DataClassification': {
                'Confidential': 'Highest security, encryption required',
                'Internal': 'Standard security, audit logging',
                'Public': 'Basic security, standard monitoring'
            },
            'Compliance': {
                'PCI': 'PCI DSS compliance rules',
                'HIPAA': 'HIPAA compliance rules',
                'SOC2': 'SOC2 compliance rules'
            }
        }
        
        for tag_key, tag_values in tag_policies.items():
            print(f"\n{tag_key} Tag Policies:")
            for value, description in tag_values.items():
                print(f"  {value}: {description}")

def main():
    import sys
    
    if len(sys.argv) < 2:
        print("Usage: python3 dynamic-security-group-manager.py <vpc-id>")
        sys.exit(1)
    
    vpc_id = sys.argv[1]
    manager = DynamicSecurityGroupManager(vpc_id)
    
    # Example usage
    example_sg_id = "sg-example123"
    example_rules = []
    
    manager.implement_time_based_access(example_sg_id, example_rules)
    manager.implement_conditional_access(example_sg_id, {})
    manager.implement_just_in_time_access(example_sg_id, 60)
    manager.tag_based_security_management()

if __name__ == "__main__":
    main()

DYNAMIC_SG

chmod +x dynamic-security-group-manager.py

echo "✅ Dynamic security group manager created"

# Save security group configuration
cat << CONFIG > security-groups-config.env
export WEB_SG_PUBLIC="$WEB_SG_PUBLIC"
export WEB_SG_PRIVATE="$WEB_SG_PRIVATE"
export APP_SG_FRONTEND="$APP_SG_FRONTEND"
export APP_SG_BACKEND="$APP_SG_BACKEND"
export APP_SG_API="$APP_SG_API"
export DB_SG_READ="$DB_SG_READ"
export DB_SG_WRITE="$DB_SG_WRITE"
export DB_SG_ADMIN="$DB_SG_ADMIN"
export MGMT_SG_BASTION="$MGMT_SG_BASTION"
export MGMT_SG_MONITORING="$MGMT_SG_MONITORING"
CONFIG

echo ""
echo "ADVANCED SECURITY GROUPS IMPLEMENTATION SUMMARY:"
echo "Security Groups Created:"
echo "  Web Tier: $WEB_SG_PUBLIC (Public), $WEB_SG_PRIVATE (Private)"
echo "  App Tier: $APP_SG_FRONTEND, $APP_SG_BACKEND, $APP_SG_API"
echo "  DB Tier: $DB_SG_READ, $DB_SG_WRITE, $DB_SG_ADMIN"
echo "  Management: $MGMT_SG_BASTION, $MGMT_SG_MONITORING"
echo ""
echo "Advanced Features:"
echo "  - Micro-segmentation by application layer"
echo "  - Principle of least privilege access"
echo "  - Dynamic security group management"
echo "  - Time-based and conditional access controls"
echo ""
echo "Configuration saved to security-groups-config.env"

EOF

chmod +x implement-advanced-security-groups.sh
./implement-advanced-security-groups.sh
```

### Step 3: Configure Advanced Network ACLs
**Objective**: Implement subnet-level security with sophisticated NACL policies

**Advanced NACL Configuration**:
```bash
# Configure advanced Network ACLs
cat << 'EOF' > configure-advanced-nacls.sh
#!/bin/bash

source advanced-routing-config.env
source security-groups-config.env

echo "=== Configuring Advanced Network ACLs ==="
echo ""

echo "1. CREATING TIER-SPECIFIC NETWORK ACLs"

# Web Tier NACL (Public Subnets)
WEB_NACL_ID=$(aws ec2 create-network-acl \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=network-acl,Tags=[{Key=Name,Value=Web-Tier-NACL},{Key=Tier,Value=Web}]" \
    --query 'NetworkAcl.NetworkAclId' \
    --output text)

echo "✅ Web Tier NACL created: $WEB_NACL_ID"

# Application Tier NACL (Private Subnets)
APP_NACL_ID=$(aws ec2 create-network-acl \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=network-acl,Tags=[{Key=Name,Value=App-Tier-NACL},{Key=Tier,Value=Application}]" \
    --query 'NetworkAcl.NetworkAclId' \
    --output text)

echo "✅ Application Tier NACL created: $APP_NACL_ID"

# Database Tier NACL (Highly Restricted)
DB_NACL_ID=$(aws ec2 create-network-acl \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=network-acl,Tags=[{Key=Name,Value=Database-Tier-NACL},{Key=Tier,Value=Database}]" \
    --query 'NetworkAcl.NetworkAclId' \
    --output text)

echo "✅ Database Tier NACL created: $DB_NACL_ID"

# Management Tier NACL (Controlled Access)
MGMT_NACL_ID=$(aws ec2 create-network-acl \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=network-acl,Tags=[{Key=Name,Value=Management-Tier-NACL},{Key=Tier,Value=Management}]" \
    --query 'NetworkAcl.NetworkAclId' \
    --output text)

echo "✅ Management Tier NACL created: $MGMT_NACL_ID"

echo ""
echo "2. CONFIGURING WEB TIER NACL RULES"

# Web Tier - Inbound Rules
# Allow HTTP from anywhere
aws ec2 create-network-acl-entry \
    --network-acl-id $WEB_NACL_ID \
    --rule-number 100 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=80,To=80 \
    --cidr-block 0.0.0.0/0

# Allow HTTPS from anywhere
aws ec2 create-network-acl-entry \
    --network-acl-id $WEB_NACL_ID \
    --rule-number 110 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=443,To=443 \
    --cidr-block 0.0.0.0/0

# Allow ephemeral ports for return traffic
aws ec2 create-network-acl-entry \
    --network-acl-id $WEB_NACL_ID \
    --rule-number 120 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=1024,To=65535 \
    --cidr-block 0.0.0.0/0

# Allow SSH from management network
aws ec2 create-network-acl-entry \
    --network-acl-id $WEB_NACL_ID \
    --rule-number 130 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=22,To=22 \
    --cidr-block 10.0.31.0/24

# Deny known bad actors (example)
aws ec2 create-network-acl-entry \
    --network-acl-id $WEB_NACL_ID \
    --rule-number 200 \
    --protocol tcp \
    --rule-action deny \
    --port-range From=1,To=65535 \
    --cidr-block 198.51.100.0/24

# Web Tier - Outbound Rules
# Allow HTTP to application tier
aws ec2 create-network-acl-entry \
    --network-acl-id $WEB_NACL_ID \
    --rule-number 100 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=8080,To=8080 \
    --cidr-block 10.0.11.0/24 \
    --egress

# Allow HTTPS to application tier
aws ec2 create-network-acl-entry \
    --network-acl-id $WEB_NACL_ID \
    --rule-number 110 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=8080,To=8080 \
    --cidr-block 10.0.12.0/24 \
    --egress

# Allow ephemeral ports for responses
aws ec2 create-network-acl-entry \
    --network-acl-id $WEB_NACL_ID \
    --rule-number 120 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=1024,To=65535 \
    --cidr-block 0.0.0.0/0 \
    --egress

echo "✅ Web Tier NACL rules configured"

echo ""
echo "3. CONFIGURING APPLICATION TIER NACL RULES"

# Application Tier - Inbound Rules
# Allow traffic from web tier
aws ec2 create-network-acl-entry \
    --network-acl-id $APP_NACL_ID \
    --rule-number 100 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=8080,To=8082 \
    --cidr-block 10.0.1.0/24

aws ec2 create-network-acl-entry \
    --network-acl-id $APP_NACL_ID \
    --rule-number 110 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=8080,To=8082 \
    --cidr-block 10.0.2.0/24

# Allow SSH from management
aws ec2 create-network-acl-entry \
    --network-acl-id $APP_NACL_ID \
    --rule-number 120 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=22,To=22 \
    --cidr-block 10.0.31.0/24

# Allow ephemeral ports
aws ec2 create-network-acl-entry \
    --network-acl-id $APP_NACL_ID \
    --rule-number 130 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=1024,To=65535 \
    --cidr-block 0.0.0.0/0

# Application Tier - Outbound Rules
# Allow to database tier
aws ec2 create-network-acl-entry \
    --network-acl-id $APP_NACL_ID \
    --rule-number 100 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=3306,To=3306 \
    --cidr-block 10.0.21.0/24 \
    --egress

aws ec2 create-network-acl-entry \
    --network-acl-id $APP_NACL_ID \
    --rule-number 110 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=3306,To=3306 \
    --cidr-block 10.0.22.0/24 \
    --egress

# Allow HTTPS for external API calls
aws ec2 create-network-acl-entry \
    --network-acl-id $APP_NACL_ID \
    --rule-number 120 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=443,To=443 \
    --cidr-block 0.0.0.0/0 \
    --egress

# Allow ephemeral ports
aws ec2 create-network-acl-entry \
    --network-acl-id $APP_NACL_ID \
    --rule-number 130 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=1024,To=65535 \
    --cidr-block 0.0.0.0/0 \
    --egress

echo "✅ Application Tier NACL rules configured"

echo ""
echo "4. CONFIGURING DATABASE TIER NACL RULES"

# Database Tier - Inbound Rules (Highly Restrictive)
# Allow MySQL from application tier only
aws ec2 create-network-acl-entry \
    --network-acl-id $DB_NACL_ID \
    --rule-number 100 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=3306,To=3306 \
    --cidr-block 10.0.11.0/24

aws ec2 create-network-acl-entry \
    --network-acl-id $DB_NACL_ID \
    --rule-number 110 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=3306,To=3306 \
    --cidr-block 10.0.12.0/24

# Allow SSH from management (for emergency access)
aws ec2 create-network-acl-entry \
    --network-acl-id $DB_NACL_ID \
    --rule-number 120 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=22,To=22 \
    --cidr-block 10.0.31.0/24

# Database Tier - Outbound Rules (Minimal)
# Allow ephemeral ports for responses only
aws ec2 create-network-acl-entry \
    --network-acl-id $DB_NACL_ID \
    --rule-number 100 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=1024,To=65535 \
    --cidr-block 10.0.11.0/24 \
    --egress

aws ec2 create-network-acl-entry \
    --network-acl-id $DB_NACL_ID \
    --rule-number 110 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=1024,To=65535 \
    --cidr-block 10.0.12.0/24 \
    --egress

echo "✅ Database Tier NACL rules configured"

echo ""
echo "5. ASSOCIATING NACLs WITH SUBNETS"

# Associate Web NACL with web subnets
aws ec2 replace-network-acl-association \
    --association-id $(aws ec2 describe-network-acls \
        --filters "Name=association.subnet-id,Values=$WEB_SUBNET_A" \
        --query 'NetworkAcls[0].Associations[0].NetworkAclAssociationId' \
        --output text) \
    --network-acl-id $WEB_NACL_ID

aws ec2 replace-network-acl-association \
    --association-id $(aws ec2 describe-network-acls \
        --filters "Name=association.subnet-id,Values=$WEB_SUBNET_B" \
        --query 'NetworkAcls[0].Associations[0].NetworkAclAssociationId' \
        --output text) \
    --network-acl-id $WEB_NACL_ID

# Associate App NACL with app subnets
aws ec2 replace-network-acl-association \
    --association-id $(aws ec2 describe-network-acls \
        --filters "Name=association.subnet-id,Values=$APP_SUBNET_A" \
        --query 'NetworkAcls[0].Associations[0].NetworkAclAssociationId' \
        --output text) \
    --network-acl-id $APP_NACL_ID

aws ec2 replace-network-acl-association \
    --association-id $(aws ec2 describe-network-acls \
        --filters "Name=association.subnet-id,Values=$APP_SUBNET_B" \
        --query 'NetworkAcls[0].Associations[0].NetworkAclAssociationId' \
        --output text) \
    --network-acl-id $APP_NACL_ID

# Associate DB NACL with database subnets
aws ec2 replace-network-acl-association \
    --association-id $(aws ec2 describe-network-acls \
        --filters "Name=association.subnet-id,Values=$DB_SUBNET_A" \
        --query 'NetworkAcls[0].Associations[0].NetworkAclAssociationId' \
        --output text) \
    --network-acl-id $DB_NACL_ID

aws ec2 replace-network-acl-association \
    --association-id $(aws ec2 describe-network-acls \
        --filters "Name=association.subnet-id,Values=$DB_SUBNET_B" \
        --query 'NetworkAcls[0].Associations[0].NetworkAclAssociationId' \
        --output text) \
    --network-acl-id $DB_NACL_ID

echo "✅ All NACL associations completed"

echo ""
echo "6. CREATING NACL MONITORING AND AUTOMATION"

cat << 'NACL_AUTOMATION' > nacl-automation-toolkit.py
#!/usr/bin/env python3

import boto3
import json
from datetime import datetime, timedelta

class NACLAutomationToolkit:
    def __init__(self, vpc_id):
        self.vpc_id = vpc_id
        self.ec2 = boto3.client('ec2')
        self.cloudwatch = boto3.client('cloudwatch')
    
    def implement_threat_response_nacl(self, threat_ip, duration_minutes=60):
        """Automatically block threat IPs using NACL rules"""
        print(f"Implementing threat response for IP: {threat_ip}")
        
        # Get all NACLs in VPC
        response = self.ec2.describe_network_acls(
            Filters=[
                {'Name': 'vpc-id', 'Values': [self.vpc_id]}
            ]
        )
        
        for nacl in response['NetworkAcls']:
            if nacl['IsDefault']:
                continue
                
            nacl_id = nacl['NetworkAclId']
            
            # Find next available rule number
            rule_numbers = [entry['RuleNumber'] for entry in nacl['Entries']]
            next_rule = 50  # Start with low priority for threat blocking
            while next_rule in rule_numbers:
                next_rule += 10
            
            print(f"Adding block rule {next_rule} to NACL {nacl_id}")
            
            # In practice, this would add the blocking rule
            threat_block_config = {
                'nacl_id': nacl_id,
                'rule_number': next_rule,
                'threat_ip': threat_ip,
                'action': 'deny',
                'expiration': (datetime.now() + timedelta(minutes=duration_minutes)).isoformat()
            }
            
            print(f"Threat block configuration: {json.dumps(threat_block_config, indent=2)}")
    
    def implement_adaptive_nacl_rules(self, traffic_patterns):
        """Implement adaptive NACL rules based on traffic patterns"""
        print("Implementing adaptive NACL rules based on traffic analysis")
        
        adaptive_rules = {
            'high_traffic_periods': {
                'description': 'Relax certain rules during high traffic',
                'rules': ['Increase ephemeral port ranges', 'Allow additional HTTP methods']
            },
            'low_traffic_periods': {
                'description': 'Stricter rules during low traffic',
                'rules': ['Reduce ephemeral port ranges', 'Block non-essential protocols']
            },
            'security_incidents': {
                'description': 'Emergency lockdown rules',
                'rules': ['Block all non-essential traffic', 'Allow only management access']
            }
        }
        
        for scenario, config in adaptive_rules.items():
            print(f"\n{scenario.upper()}:")
            print(f"  Description: {config['description']}")
            for rule in config['rules']:
                print(f"  - {rule}")
    
    def implement_compliance_nacl_templates(self):
        """Implement compliance-specific NACL templates"""
        print("\nCompliance NACL Templates:")
        
        compliance_templates = {
            'PCI_DSS': {
                'description': 'PCI DSS compliance requirements',
                'rules': [
                    'Block all non-essential ports',
                    'Restrict access to cardholder data environment',
                    'Implement strong access controls',
                    'Log all network access attempts'
                ]
            },
            'HIPAA': {
                'description': 'HIPAA compliance for healthcare data',
                'rules': [
                    'Encrypt all communications',
                    'Restrict access to PHI systems',
                    'Implement audit logging',
                    'Control administrative access'
                ]
            },
            'SOC2': {
                'description': 'SOC2 compliance controls',
                'rules': [
                    'Implement security monitoring',
                    'Control logical access',
                    'Maintain system boundaries',
                    'Monitor system changes'
                ]
            }
        }
        
        for compliance, config in compliance_templates.items():
            print(f"\n{compliance}:")
            print(f"  Description: {config['description']}")
            for rule in config['rules']:
                print(f"  - {rule}")
    
    def monitor_nacl_effectiveness(self):
        """Monitor NACL rule effectiveness"""
        print("\nNACL Effectiveness Monitoring:")
        
        monitoring_metrics = [
            'Number of blocked connections per rule',
            'False positive rate for blocking rules',
            'Response time impact of NACL processing',
            'Rule utilization and optimization opportunities'
        ]
        
        for metric in monitoring_metrics:
            print(f"- {metric}")
        
        print("\nRecommended CloudWatch Metrics:")
        cloudwatch_metrics = [
            'NetworkPacketsIn/Out by NACL',
            'NetworkBytesIn/Out by NACL', 
            'Custom metric for blocked connections',
            'Custom metric for rule performance'
        ]
        
        for metric in cloudwatch_metrics:
            print(f"- {metric}")

def main():
    import sys
    
    if len(sys.argv) < 2:
        print("Usage: python3 nacl-automation-toolkit.py <vpc-id>")
        sys.exit(1)
    
    vpc_id = sys.argv[1]
    toolkit = NACLAutomationToolkit(vpc_id)
    
    # Example usage
    toolkit.implement_threat_response_nacl('198.51.100.100', 120)
    toolkit.implement_adaptive_nacl_rules({})
    toolkit.implement_compliance_nacl_templates()
    toolkit.monitor_nacl_effectiveness()

if __name__ == "__main__":
    main()

NACL_AUTOMATION

chmod +x nacl-automation-toolkit.py

# Save NACL configuration
cat << CONFIG > nacls-config.env
export WEB_NACL_ID="$WEB_NACL_ID"
export APP_NACL_ID="$APP_NACL_ID"
export DB_NACL_ID="$DB_NACL_ID"
export MGMT_NACL_ID="$MGMT_NACL_ID"
CONFIG

echo ""
echo "ADVANCED NACL CONFIGURATION SUMMARY:"
echo "Network ACLs Created:"
echo "  Web Tier: $WEB_NACL_ID"
echo "  Application Tier: $APP_NACL_ID"
echo "  Database Tier: $DB_NACL_ID"
echo "  Management Tier: $MGMT_NACL_ID"
echo ""
echo "Advanced Features:"
echo "  - Tier-specific access controls"
echo "  - Threat response automation"
echo "  - Compliance templates"
echo "  - Adaptive rule management"
echo ""
echo "Configuration saved to nacls-config.env"

EOF

chmod +x configure-advanced-nacls.sh
./configure-advanced-nacls.sh
```

### Step 4: Implement VPC Security Monitoring
**Objective**: Deploy comprehensive security monitoring and threat detection

**Security Monitoring Implementation**:
```bash
# Implement comprehensive VPC security monitoring
cat << 'EOF' > implement-vpc-security-monitoring.sh
#!/bin/bash

source advanced-routing-config.env

echo "=== Implementing Comprehensive VPC Security Monitoring ==="
echo ""

echo "1. ENABLING VPC FLOW LOGS"

# Create CloudWatch Log Group for VPC Flow Logs
LOG_GROUP_NAME="/aws/vpc/flowlogs"
aws logs create-log-group --log-group-name $LOG_GROUP_NAME

# Create IAM role for VPC Flow Logs
FLOW_LOGS_ROLE_POLICY=$(cat << 'POLICY'
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
POLICY
)

FLOW_LOGS_ROLE_ARN=$(aws iam create-role \
    --role-name VPC-FlowLogs-Role \
    --assume-role-policy-document "$FLOW_LOGS_ROLE_POLICY" \
    --query 'Role.Arn' \
    --output text)

# Attach policy to role
aws iam put-role-policy \
    --role-name VPC-FlowLogs-Role \
    --policy-name VPC-FlowLogs-Policy \
    --policy-document '{
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
    }'

# Enable VPC Flow Logs
FLOW_LOGS_ID=$(aws ec2 create-flow-logs \
    --resource-type VPC \
    --resource-ids $VPC_ID \
    --traffic-type ALL \
    --log-destination-type cloud-watch-logs \
    --log-group-name $LOG_GROUP_NAME \
    --deliver-logs-permission-arn $FLOW_LOGS_ROLE_ARN \
    --query 'FlowLogIds[0]' \
    --output text)

echo "✅ VPC Flow Logs enabled: $FLOW_LOGS_ID"

echo ""
echo "2. SETTING UP SECURITY MONITORING DASHBOARD"

# Create comprehensive security dashboard
SECURITY_DASHBOARD_BODY=$(cat << DASHBOARD
{
    "widgets": [
        {
            "type": "log",
            "x": 0,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "query": "SOURCE '$LOG_GROUP_NAME'\n| fields @timestamp, srcaddr, dstaddr, srcport, dstport, protocol, action\n| filter action = \"REJECT\"\n| stats count() by srcaddr\n| sort count desc\n| limit 20",
                "region": "us-east-1",
                "title": "Top Rejected Source IPs",
                "view": "table"
            }
        },
        {
            "type": "log",
            "x": 12,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "query": "SOURCE '$LOG_GROUP_NAME'\n| fields @timestamp, srcaddr, dstaddr, srcport, dstport, protocol, packets, bytes\n| filter bytes > 1000000\n| sort bytes desc\n| limit 10",
                "region": "us-east-1",
                "title": "High Data Transfer Connections",
                "view": "table"
            }
        },
        {
            "type": "log",
            "x": 0,
            "y": 6,
            "width": 24,
            "height": 6,
            "properties": {
                "query": "SOURCE '$LOG_GROUP_NAME'\n| fields @timestamp, srcaddr, dstaddr, dstport, action\n| filter action = \"REJECT\"\n| stats count() by dstport\n| sort count desc\n| limit 15",
                "region": "us-east-1",
                "title": "Most Targeted Ports (Rejected)",
                "view": "bar"
            }
        }
    ]
}
DASHBOARD
)

aws cloudwatch put-dashboard \
    --dashboard-name "VPC-Security-Monitoring" \
    --dashboard-body "$SECURITY_DASHBOARD_BODY"

echo "✅ Security monitoring dashboard created"

echo ""
echo "3. IMPLEMENTING THREAT DETECTION SYSTEM"

cat << 'THREAT_DETECTION' > implement-threat-detection.py
#!/usr/bin/env python3

import boto3
import json
from datetime import datetime, timedelta
import ipaddress

class VPCThreatDetectionSystem:
    def __init__(self, vpc_id):
        self.vpc_id = vpc_id
        self.logs_client = boto3.client('logs')
        self.ec2 = boto3.client('ec2')
        self.sns = boto3.client('sns')
        self.guardduty = boto3.client('guardduty')
    
    def analyze_flow_logs_for_threats(self, hours=1):
        """Analyze VPC Flow Logs for potential threats"""
        end_time = datetime.now()
        start_time = end_time - timedelta(hours=hours)
        
        print("THREAT ANALYSIS FROM VPC FLOW LOGS:")
        print("=" * 50)
        
        # Query for suspicious patterns
        suspicious_patterns = {
            'port_scanning': {
                'query': '''
                    fields @timestamp, srcaddr, dstaddr, dstport, action
                    | filter action = "REJECT"
                    | stats count() by srcaddr, dstaddr
                    | sort count desc
                    | limit 10
                ''',
                'description': 'Potential port scanning activity'
            },
            'data_exfiltration': {
                'query': '''
                    fields @timestamp, srcaddr, dstaddr, bytes, action
                    | filter action = "ACCEPT" and bytes > 10000000
                    | sort bytes desc
                    | limit 5
                ''',
                'description': 'Large data transfers (potential exfiltration)'
            },
            'bruteforce_attempts': {
                'query': '''
                    fields @timestamp, srcaddr, dstaddr, dstport, action
                    | filter dstport = 22 and action = "REJECT"
                    | stats count() by srcaddr
                    | sort count desc
                    | limit 10
                ''',
                'description': 'SSH brute force attempts'
            },
            'unusual_protocols': {
                'query': '''
                    fields @timestamp, srcaddr, dstaddr, protocol, action
                    | filter protocol not in [6, 17, 1]
                    | stats count() by protocol, srcaddr
                    | sort count desc
                ''',
                'description': 'Unusual protocol usage'
            }
        }
        
        for pattern_name, pattern_config in suspicious_patterns.items():
            print(f"\n{pattern_name.upper().replace('_', ' ')}:")
            print(f"Description: {pattern_config['description']}")
            print("Query ready for CloudWatch Logs Insights")
            
            # In production, this would execute the query and analyze results
            print("Sample detection logic:")
            if pattern_name == 'port_scanning':
                print("  - Alert if single IP attempts >100 different ports")
                print("  - Block IP if scan rate >50 ports/minute")
            elif pattern_name == 'data_exfiltration':
                print("  - Alert if data transfer >1GB to external IP")
                print("  - Investigate if multiple large transfers")
            elif pattern_name == 'bruteforce_attempts':
                print("  - Alert if >20 failed SSH attempts from single IP")
                print("  - Auto-block after 50 failed attempts")
    
    def implement_automated_response(self):
        """Implement automated threat response"""
        print("\nAUTOMATED THREAT RESPONSE SYSTEM:")
        print("=" * 50)
        
        response_actions = {
            'ip_blocking': {
                'trigger': 'Suspicious IP behavior detected',
                'action': 'Add IP to NACL deny rule',
                'duration': '1 hour (adjustable)',
                'escalation': 'Security team notification'
            },
            'instance_isolation': {
                'trigger': 'Compromised instance detected',
                'action': 'Modify security groups to isolate',
                'duration': 'Until manual review',
                'escalation': 'Immediate security team alert'
            },
            'traffic_redirection': {
                'trigger': 'DDoS attack detected',
                'action': 'Redirect traffic through AWS Shield',
                'duration': 'Duration of attack',
                'escalation': 'AWS Support engagement'
            },
            'emergency_lockdown': {
                'trigger': 'Critical security breach',
                'action': 'Block all non-essential traffic',
                'duration': 'Until incident resolved',
                'escalation': 'Executive team notification'
            }
        }
        
        for action_name, action_config in response_actions.items():
            print(f"\n{action_name.upper().replace('_', ' ')}:")
            for key, value in action_config.items():
                print(f"  {key.title()}: {value}")
    
    def setup_guardduty_integration(self):
        """Set up GuardDuty integration for advanced threat detection"""
        print("\nGUARDDUTY INTEGRATION:")
        print("=" * 50)
        
        guardduty_features = {
            'VPC_FLOW_LOGS': 'Analyze VPC Flow Logs for network threats',
            'DNS_LOGS': 'Monitor DNS queries for malicious domains',
            'S3_PROTECTION': 'Detect S3 bucket compromise',
            'MALWARE_PROTECTION': 'Scan EC2 instances for malware',
            'KUBERNETES_PROTECTION': 'Monitor EKS clusters'
        }
        
        print("GuardDuty Features for VPC Security:")
        for feature, description in guardduty_features.items():
            print(f"  {feature}: {description}")
        
        print("\nThreat Detection Categories:")
        threat_categories = [
            'Backdoor (C&C communication)',
            'Behavior (Anomalous API calls)', 
            'Cryptocurrency (Mining activity)',
            'Malware (Known malicious IPs)',
            'Recon (Port scanning, enumeration)',
            'Trojan (Trojan horse activity)',
            'UnauthorizedBehavior (Suspicious activities)'
        ]
        
        for category in threat_categories:
            print(f"  - {category}")
    
    def create_security_automation_lambda(self):
        """Create Lambda function template for security automation"""
        lambda_function = '''
import boto3
import json

def lambda_handler(event, context):
    """
    Security automation Lambda function
    Triggered by CloudWatch alarms or GuardDuty findings
    """
    
    ec2 = boto3.client('ec2')
    sns = boto3.client('sns')
    
    # Parse the security event
    event_type = event.get('source', '')
    
    if event_type == 'aws.guardduty':
        # Handle GuardDuty finding
        response = handle_guardduty_finding(event)
        
    elif event_type == 'aws.cloudwatch':
        # Handle CloudWatch alarm
        response = handle_cloudwatch_alarm(event)
        
    else:
        response = {'status': 'unknown_event_type'}
    
    return {
        'statusCode': 200,
        'body': json.dumps(response)
    }

def handle_guardduty_finding(event):
    """Handle GuardDuty security finding"""
    detail = event.get('detail', {})
    severity = detail.get('severity', 0)
    finding_type = detail.get('type', '')
    
    if severity >= 7.0:  # High severity
        # Implement immediate response
        return {'action': 'immediate_response', 'severity': 'high'}
    elif severity >= 4.0:  # Medium severity
        # Implement standard response
        return {'action': 'standard_response', 'severity': 'medium'}
    else:
        # Log and monitor
        return {'action': 'log_and_monitor', 'severity': 'low'}

def handle_cloudwatch_alarm(event):
    """Handle CloudWatch alarm"""
    alarm_name = event.get('AlarmName', '')
    
    if 'SuspiciousTraffic' in alarm_name:
        # Block suspicious IP
        return {'action': 'block_ip'}
    elif 'HighDataTransfer' in alarm_name:
        # Investigate data transfer
        return {'action': 'investigate_transfer'}
    
    return {'action': 'default_response'}
        '''
        
        with open('security-automation-lambda.py', 'w') as f:
            f.write(lambda_function)
        
        print("\nSECURITY AUTOMATION LAMBDA:")
        print("File: security-automation-lambda.py")
        print("Purpose: Automated response to security events")
        print("Triggers: GuardDuty findings, CloudWatch alarms")
    
    def implement_network_forensics(self):
        """Implement network forensics capabilities"""
        print("\nNETWORK FORENSICS CAPABILITIES:")
        print("=" * 50)
        
        forensics_tools = {
            'packet_capture': {
                'tool': 'VPC Traffic Mirroring',
                'purpose': 'Deep packet inspection',
                'use_case': 'Detailed traffic analysis'
            },
            'flow_analysis': {
                'tool': 'VPC Flow Logs',
                'purpose': 'Connection-level analysis',
                'use_case': 'Traffic pattern investigation'
            },
            'dns_analysis': {
                'tool': 'Route 53 Resolver Query Logs',
                'purpose': 'DNS query investigation',
                'use_case': 'Malicious domain detection'
            },
            'timeline_analysis': {
                'tool': 'CloudTrail + Flow Logs',
                'purpose': 'Event correlation',
                'use_case': 'Incident timeline reconstruction'
            }
        }
        
        for tool_name, tool_config in forensics_tools.items():
            print(f"\n{tool_name.upper().replace('_', ' ')}:")
            for key, value in tool_config.items():
                print(f"  {key.title()}: {value}")

def main():
    import sys
    
    if len(sys.argv) < 2:
        print("Usage: python3 implement-threat-detection.py <vpc-id>")
        sys.exit(1)
    
    vpc_id = sys.argv[1]
    system = VPCThreatDetectionSystem(vpc_id)
    
    system.analyze_flow_logs_for_threats()
    system.implement_automated_response()
    system.setup_guardduty_integration()
    system.create_security_automation_lambda()
    system.implement_network_forensics()

if __name__ == "__main__":
    main()

THREAT_DETECTION

chmod +x implement-threat-detection.py

echo "✅ Threat detection system implemented"

echo ""
echo "4. CREATING SECURITY ALERTING SYSTEM"

# Create SNS topic for security alerts
SECURITY_TOPIC_ARN=$(aws sns create-topic \
    --name VPC-Security-Alerts \
    --query 'TopicArn' \
    --output text)

# Create CloudWatch alarms for security events
aws cloudwatch put-metric-alarm \
    --alarm-name "VPC-High-Rejected-Traffic" \
    --alarm-description "Alert on high volume of rejected traffic" \
    --metric-name "PacketsDropCount" \
    --namespace "AWS/VPC" \
    --statistic "Sum" \
    --period 300 \
    --threshold 1000 \
    --comparison-operator "GreaterThanThreshold" \
    --evaluation-periods 2 \
    --alarm-actions $SECURITY_TOPIC_ARN

echo "✅ Security alerting system configured"

# Save monitoring configuration
cat << CONFIG > security-monitoring-config.env
export FLOW_LOGS_ID="$FLOW_LOGS_ID"
export LOG_GROUP_NAME="$LOG_GROUP_NAME"
export SECURITY_TOPIC_ARN="$SECURITY_TOPIC_ARN"
export FLOW_LOGS_ROLE_ARN="$FLOW_LOGS_ROLE_ARN"
CONFIG

echo ""
echo "VPC SECURITY MONITORING IMPLEMENTATION SUMMARY:"
echo "Components Deployed:"
echo "  VPC Flow Logs: $FLOW_LOGS_ID"
echo "  Log Group: $LOG_GROUP_NAME"
echo "  Security Dashboard: VPC-Security-Monitoring"
echo "  Security Alerts Topic: $SECURITY_TOPIC_ARN"
echo ""
echo "Monitoring Features:"
echo "  - Real-time threat detection"
echo "  - Automated response capabilities"
echo "  - Comprehensive security analytics"
echo "  - Network forensics tools"
echo ""
echo "Configuration saved to security-monitoring-config.env"

EOF

chmod +x implement-vpc-security-monitoring.sh
./implement-vpc-security-monitoring.sh
```

### Step 5: Implement Network Segmentation and Micro-segmentation
**Objective**: Deploy advanced network segmentation strategies

**Network Segmentation Implementation**:
```bash
# Implement advanced network segmentation
cat << 'EOF' > implement-network-segmentation.sh
#!/bin/bash

source advanced-routing-config.env
source security-groups-config.env

echo "=== Implementing Advanced Network Segmentation ==="
echo ""

echo "1. DESIGNING MICRO-SEGMENTATION ARCHITECTURE"

cat << 'MICROSEGMENTATION' > design-microsegmentation.py
#!/usr/bin/env python3

import boto3
import json

class MicroSegmentationDesigner:
    def __init__(self, vpc_id):
        self.vpc_id = vpc_id
        self.ec2 = boto3.client('ec2')
    
    def design_zero_trust_architecture(self):
        """Design zero trust network architecture"""
        print("ZERO TRUST NETWORK ARCHITECTURE:")
        print("=" * 50)
        
        zero_trust_principles = {
            'never_trust_always_verify': {
                'description': 'Verify every connection attempt',
                'implementation': [
                    'Multi-factor authentication required',
                    'Device compliance verification',
                    'Continuous behavior monitoring',
                    'Session-based access controls'
                ]
            },
            'least_privilege_access': {
                'description': 'Minimum required access only',
                'implementation': [
                    'Just-in-time access provisioning',
                    'Role-based access controls',
                    'Time-limited permissions',
                    'Regular access reviews'
                ]
            },
            'assume_breach_mentality': {
                'description': 'Design for compromise scenarios',
                'implementation': [
                    'Network segmentation everywhere',
                    'Lateral movement prevention',
                    'Advanced threat detection',
                    'Rapid incident response'
                ]
            },
            'encrypt_everything': {
                'description': 'End-to-end encryption',
                'implementation': [
                    'Data in transit encryption',
                    'Data at rest encryption',
                    'Application-level encryption',
                    'Key management systems'
                ]
            }
        }
        
        for principle, details in zero_trust_principles.items():
            print(f"\n{principle.upper().replace('_', ' ')}:")
            print(f"  Description: {details['description']}")
            print("  Implementation:")
            for item in details['implementation']:
                print(f"    - {item}")
    
    def implement_application_microsegmentation(self):
        """Implement application-level micro-segmentation"""
        print("\nAPPLICATION MICRO-SEGMENTATION:")
        print("=" * 50)
        
        microseg_patterns = {
            'service_mesh': {
                'description': 'Service-to-service communication control',
                'technologies': ['AWS App Mesh', 'Istio', 'Consul Connect'],
                'benefits': [
                    'mTLS between services',
                    'Traffic policies enforcement',
                    'Observability and monitoring',
                    'Gradual rollout capabilities'
                ]
            },
            'container_segmentation': {
                'description': 'Container-level network isolation',
                'technologies': ['EKS Network Policies', 'Calico', 'Cilium'],
                'benefits': [
                    'Pod-to-pod communication control',
                    'Namespace isolation',
                    'Label-based policies',
                    'Runtime security'
                ]
            },
            'function_isolation': {
                'description': 'Serverless function network isolation',
                'technologies': ['Lambda VPC', 'VPC Endpoints', 'PrivateLink'],
                'benefits': [
                    'Function-level network controls',
                    'No internet access by default',
                    'Service-specific connectivity',
                    'Reduced attack surface'
                ]
            }
        }
        
        for pattern, details in microseg_patterns.items():
            print(f"\n{pattern.upper().replace('_', ' ')}:")
            print(f"  Description: {details['description']}")
            print(f"  Technologies: {', '.join(details['technologies'])}")
            print("  Benefits:")
            for benefit in details['benefits']:
                print(f"    - {benefit}")
    
    def design_compliance_segmentation(self):
        """Design compliance-driven segmentation"""
        print("\nCOMPLIANCE-DRIVEN SEGMENTATION:")
        print("=" * 50)
        
        compliance_zones = {
            'PCI_DSS_Zone': {
                'description': 'Payment card data processing zone',
                'requirements': [
                    'Isolated from other networks',
                    'Restricted access controls',
                    'Comprehensive logging',
                    'Regular security testing'
                ],
                'network_controls': [
                    'Dedicated subnets for CDE',
                    'Firewall rules at every boundary',
                    'Network access control',
                    'Secure remote access'
                ]
            },
            'HIPAA_Zone': {
                'description': 'Healthcare data processing zone',
                'requirements': [
                    'PHI data protection',
                    'Audit trail maintenance',
                    'Access controls',
                    'Transmission security'
                ],
                'network_controls': [
                    'Encrypted communications',
                    'Access monitoring',
                    'Network segmentation',
                    'Intrusion detection'
                ]
            },
            'SOC2_Zone': {
                'description': 'SOC2 compliance zone',
                'requirements': [
                    'Security controls',
                    'Availability controls',
                    'Processing integrity',
                    'Confidentiality controls'
                ],
                'network_controls': [
                    'Logical access controls',
                    'Network security monitoring',
                    'Change management',
                    'System boundaries'
                ]
            }
        }
        
        for zone, details in compliance_zones.items():
            print(f"\n{zone}:")
            print(f"  Description: {details['description']}")
            print("  Requirements:")
            for req in details['requirements']:
                print(f"    - {req}")
            print("  Network Controls:")
            for control in details['network_controls']:
                print(f"    - {control}")
    
    def implement_dynamic_segmentation(self):
        """Implement dynamic segmentation based on context"""
        print("\nDYNAMIC SEGMENTATION:")
        print("=" * 50)
        
        dynamic_factors = {
            'user_context': {
                'factors': ['Location', 'Device type', 'Time of access', 'Behavior patterns'],
                'actions': ['Adjust network access', 'Apply security policies', 'Enable monitoring']
            },
            'threat_level': {
                'factors': ['Current threat intelligence', 'Attack patterns', 'Vulnerability status'],
                'actions': ['Increase restrictions', 'Enable additional monitoring', 'Isolate systems']
            },
            'application_state': {
                'factors': ['Application health', 'Performance metrics', 'Error rates'],
                'actions': ['Traffic shaping', 'Failover routing', 'Load balancing']
            },
            'business_context': {
                'factors': ['Business hours', 'Maintenance windows', 'Critical operations'],
                'actions': ['Adjust access policies', 'Change monitoring levels', 'Update priorities']
            }
        }
        
        for context, details in dynamic_factors.items():
            print(f"\n{context.upper().replace('_', ' ')}:")
            print("  Factors:")
            for factor in details['factors']:
                print(f"    - {factor}")
            print("  Actions:")
            for action in details['actions']:
                print(f"    - {action}")

def main():
    import sys
    
    if len(sys.argv) < 2:
        print("Usage: python3 design-microsegmentation.py <vpc-id>")
        sys.exit(1)
    
    vpc_id = sys.argv[1]
    designer = MicroSegmentationDesigner(vpc_id)
    
    designer.design_zero_trust_architecture()
    designer.implement_application_microsegmentation()
    designer.design_compliance_segmentation()
    designer.implement_dynamic_segmentation()

if __name__ == "__main__":
    main()

MICROSEGMENTATION

chmod +x design-microsegmentation.py

echo "✅ Micro-segmentation design tool created"

echo ""
echo "2. IMPLEMENTING WORKLOAD ISOLATION"

cat << 'WORKLOAD_ISOLATION' > implement-workload-isolation.sh
#!/bin/bash

echo "=== Implementing Workload Isolation Strategies ==="
echo ""

echo "1. CREATING ISOLATED WORKLOAD ENVIRONMENTS"

# Create dedicated security groups for different workload types
WORKLOAD_CRITICAL_SG=$(aws ec2 create-security-group \
    --group-name Critical-Workload-SG \
    --description "Security group for critical business workloads" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Critical-Workload-SG},{Key=Classification,Value=Critical}]" \
    --query 'GroupId' \
    --output text)

WORKLOAD_STANDARD_SG=$(aws ec2 create-security-group \
    --group-name Standard-Workload-SG \
    --description "Security group for standard business workloads" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Standard-Workload-SG},{Key=Classification,Value=Standard}]" \
    --query 'GroupId' \
    --output text)

WORKLOAD_DEVELOPMENT_SG=$(aws ec2 create-security-group \
    --group-name Development-Workload-SG \
    --description "Security group for development workloads" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Development-Workload-SG},{Key=Classification,Value=Development}]" \
    --query 'GroupId' \
    --output text)

echo "✅ Workload isolation security groups created"

echo ""
echo "2. CONFIGURING WORKLOAD-SPECIFIC ACCESS CONTROLS"

# Critical workload - Highly restricted
aws ec2 authorize-security-group-ingress \
    --group-id $WORKLOAD_CRITICAL_SG \
    --protocol tcp \
    --port 443 \
    --source-group $WEB_SG_PRIVATE

# No SSH access to critical workloads from external sources
# Only internal management access
aws ec2 authorize-security-group-ingress \
    --group-id $WORKLOAD_CRITICAL_SG \
    --protocol tcp \
    --port 22 \
    --source-group $MGMT_SG_BASTION

echo "✅ Critical workload access controls configured"

# Standard workload - Moderate restrictions
aws ec2 authorize-security-group-ingress \
    --group-id $WORKLOAD_STANDARD_SG \
    --protocol tcp \
    --port 80 \
    --source-group $WEB_SG_PRIVATE

aws ec2 authorize-security-group-ingress \
    --group-id $WORKLOAD_STANDARD_SG \
    --protocol tcp \
    --port 443 \
    --source-group $WEB_SG_PRIVATE

aws ec2 authorize-security-group-ingress \
    --group-id $WORKLOAD_STANDARD_SG \
    --protocol tcp \
    --port 22 \
    --source-group $MGMT_SG_BASTION

echo "✅ Standard workload access controls configured"

# Development workload - More permissive for development needs
aws ec2 authorize-security-group-ingress \
    --group-id $WORKLOAD_DEVELOPMENT_SG \
    --protocol tcp \
    --port 80 \
    --cidr 10.0.0.0/16

aws ec2 authorize-security-group-ingress \
    --group-id $WORKLOAD_DEVELOPMENT_SG \
    --protocol tcp \
    --port 443 \
    --cidr 10.0.0.0/16

aws ec2 authorize-security-group-ingress \
    --group-id $WORKLOAD_DEVELOPMENT_SG \
    --protocol tcp \
    --port 22 \
    --cidr 10.0.31.0/24

# Allow development ports
aws ec2 authorize-security-group-ingress \
    --group-id $WORKLOAD_DEVELOPMENT_SG \
    --protocol tcp \
    --port 3000 \
    --cidr 10.0.0.0/16

aws ec2 authorize-security-group-ingress \
    --group-id $WORKLOAD_DEVELOPMENT_SG \
    --protocol tcp \
    --port 8000 \
    --cidr 10.0.0.0/16

echo "✅ Development workload access controls configured"

echo ""
echo "3. WORKLOAD ISOLATION SUMMARY"
echo "Critical Workloads: $WORKLOAD_CRITICAL_SG"
echo "  - HTTPS access from web tier only"
echo "  - SSH access from bastion only"
echo "  - No direct internet access"
echo "  - Comprehensive monitoring required"
echo ""
echo "Standard Workloads: $WORKLOAD_STANDARD_SG"
echo "  - HTTP/HTTPS access from web tier"
echo "  - SSH access from management"
echo "  - Standard monitoring"
echo ""
echo "Development Workloads: $WORKLOAD_DEVELOPMENT_SG"
echo "  - Broader access for development needs"
echo "  - Development ports enabled"
echo "  - Enhanced logging for debugging"

WORKLOAD_ISOLATION

chmod +x implement-workload-isolation.sh

echo "✅ Workload isolation implementation created"

echo ""
echo "NETWORK SEGMENTATION IMPLEMENTATION COMPLETED"
echo ""
echo "Segmentation Components:"
echo "1. Zero trust architecture design"
echo "2. Application micro-segmentation"
echo "3. Compliance-driven segmentation"
echo "4. Dynamic segmentation capabilities"
echo "5. Workload isolation strategies"
echo ""
echo "Available Tools:"
echo "- design-microsegmentation.py"
echo "- implement-workload-isolation.sh"

EOF

chmod +x implement-network-segmentation.sh
./implement-network-segmentation.sh
```

## VPC Network Security Summary

### Defense in Depth Architecture

#### Multi-Layer Security Model
1. **Physical Security**: AWS data center security
2. **Infrastructure Security**: VPC isolation and subnets  
3. **Perimeter Security**: Internet gateways and firewalls
4. **Network Security**: Security groups and NACLs
5. **Identity & Access**: IAM and authentication
6. **Data Security**: Encryption and data loss prevention
7. **Application Security**: WAF and application controls

### Advanced Security Group Strategies

#### Micro-Segmentation Benefits
- **Granular Control**: Instance-level security policies
- **Least Privilege**: Minimum required access only
- **Dynamic Management**: Tag-based and automated rules
- **Compliance Support**: Audit trails and policy enforcement

#### Security Group Best Practices
- **Layered Defense**: Multiple security groups per instance
- **Principle of Least Privilege**: Minimum required ports/protocols
- **Source Specificity**: Reference other security groups vs CIDR blocks
- **Regular Auditing**: Automated compliance checking

### Network ACL Advanced Implementation

#### NACL Design Patterns
- **Tier-based Control**: Different NACLs per application tier
- **Threat Response**: Automated blocking of malicious IPs
- **Compliance Templates**: Pre-configured compliance rules
- **Emergency Procedures**: Rapid lockdown capabilities

### VPC Security Monitoring

#### Comprehensive Monitoring Strategy
- **VPC Flow Logs**: Network traffic analysis
- **CloudWatch Dashboards**: Real-time security metrics
- **GuardDuty Integration**: ML-based threat detection
- **Automated Response**: Lambda-based security automation

#### Threat Detection Capabilities
- **Anomaly Detection**: Unusual traffic patterns
- **Signature-based Detection**: Known attack patterns
- **Behavioral Analysis**: User and entity behavior analytics
- **Threat Intelligence**: External threat feeds integration

## Key Takeaways
- Network security requires multiple layers of defense
- Security groups and NACLs provide complementary protection
- Micro-segmentation enables granular access control
- Automated monitoring and response are essential for scale
- Compliance requirements drive segmentation strategies
- Zero trust principles enhance overall security posture

## Additional Resources
- [VPC Security Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-best-practices.html)
- [AWS GuardDuty User Guide](https://docs.aws.amazon.com/guardduty/latest/ug/)
- [VPC Flow Logs User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)

## Exam Tips
- Security groups are stateful, NACLs are stateless
- Security groups default deny, can only create allow rules
- NACLs support both allow and deny rules
- Security groups evaluate all rules, NACLs process in order
- VPC Flow Logs capture metadata, not packet contents
- GuardDuty uses ML for threat detection
- Defense in depth requires multiple security layers
- Principle of least privilege should guide all access decisions