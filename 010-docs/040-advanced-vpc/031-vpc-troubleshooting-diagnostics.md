# Topic 31: VPC Troubleshooting and Advanced Diagnostics

## Prerequisites
- Completed Topic 30 (VPC Performance Optimization and Network Acceleration)
- Understanding of network troubleshooting methodologies
- Knowledge of AWS networking services and monitoring tools
- Familiarity with packet analysis and network debugging

## Learning Objectives
By the end of this topic, you will be able to:
- Implement systematic VPC troubleshooting methodologies
- Use advanced AWS diagnostic tools and techniques
- Analyze VPC Flow Logs and network traffic patterns
- Resolve complex connectivity and performance issues

## Theory

### VPC Troubleshooting Methodology

#### Systematic Approach
```
Troubleshooting Framework:
┌─────────────────────────────────────────────────────┐
│ 1. Problem Definition & Scope                      │
├─────────────────────────────────────────────────────┤
│ 2. Information Gathering & Baseline                │
├─────────────────────────────────────────────────────┤
│ 3. Layer-by-Layer Analysis (OSI Model)             │
├─────────────────────────────────────────────────────┤
│ 4. Component Isolation & Testing                   │
├─────────────────────────────────────────────────────┤
│ 5. Root Cause Analysis                             │
├─────────────────────────────────────────────────────┤
│ 6. Solution Implementation & Verification          │
└─────────────────────────────────────────────────────┘
```

#### Common VPC Issues Categories
- **Connectivity Issues**: Cannot reach destination
- **Performance Issues**: Slow response times or low throughput  
- **Security Issues**: Blocked traffic or unauthorized access
- **DNS Issues**: Name resolution failures
- **Routing Issues**: Traffic taking wrong paths

### AWS Diagnostic Tools

#### VPC Reachability Analyzer
- **Purpose**: Test connectivity between network resources
- **Analysis**: Layer 3 and 4 connectivity verification
- **Output**: Detailed path analysis and blocking points
- **Use Cases**: Troubleshoot security group and NACL issues

#### VPC Flow Logs
- **Data Sources**: VPC, Subnet, or ENI level
- **Information**: Source/destination IPs, ports, protocols, actions
- **Analysis**: Traffic patterns, security violations, performance
- **Integration**: CloudWatch Logs, S3, Kinesis Data Firehose

#### AWS X-Ray
- **Tracing**: End-to-end request tracing
- **Performance**: Latency analysis and bottleneck identification
- **Integration**: Application-level network diagnostics
- **Visualization**: Service map and performance insights

### Network Layer Troubleshooting

#### Layer 3 (Network) Issues
```
Common Layer 3 Problems:
┌──────────────────┬─────────────────┬─────────────────┐
│ Issue            │ Symptoms        │ Diagnostic      │
├──────────────────┼─────────────────┼─────────────────┤
│ Route Missing    │ No connectivity │ Route tables    │
│ Wrong Route      │ Traffic routing │ Traceroute      │
│ MTU Mismatch     │ Fragmentation   │ Ping with size  │
│ IP Conflicts     │ Intermittent    │ ARP tables      │
└──────────────────┴─────────────────┴─────────────────┘
```

#### Layer 4 (Transport) Issues
```
Common Layer 4 Problems:
┌──────────────────┬─────────────────┬─────────────────┐
│ Issue            │ Symptoms        │ Diagnostic      │
├──────────────────┼─────────────────┼─────────────────┤
│ Port Blocked     │ Connection fail │ Telnet/NC test  │
│ Security Groups  │ Access denied   │ SG rule check  │
│ NACLs            │ One-way block   │ NACL analysis   │
│ Load Balancer    │ Health checks   │ Target health   │
└──────────────────┴─────────────────┴─────────────────┘
```

## Lab Exercise: Comprehensive VPC Troubleshooting Implementation

### Lab Duration: 360 minutes

### Step 1: Build Troubleshooting Lab Environment
**Objective**: Create a complex environment with intentional issues for troubleshooting practice

**Troubleshooting Lab Setup**:
```bash
# Create comprehensive troubleshooting lab environment
cat << 'EOF' > setup-troubleshooting-lab.sh
#!/bin/bash

echo "=== Setting Up VPC Troubleshooting Lab Environment ==="
echo ""

echo "1. CREATING TROUBLESHOOTING VPC WITH INTENTIONAL ISSUES"

# Create VPC for troubleshooting lab
TROUBLE_VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.100.0.0/16 \
    --tag-specifications "ResourceType=vpc,Tags=[{Key=Name,Value=Troubleshooting-Lab-VPC},{Key=Purpose,Value=Diagnostics}]" \
    --query 'Vpc.VpcId' \
    --output text)

echo "✅ Troubleshooting VPC created: $TROUBLE_VPC_ID"

# Create subnets with various configurations
PUBLIC_SUBNET_ID=$(aws ec2 create-subnet \
    --vpc-id $TROUBLE_VPC_ID \
    --cidr-block 10.100.1.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=Trouble-Public-Subnet}]" \
    --query 'Subnet.SubnetId' \
    --output text)

PRIVATE_SUBNET_ID=$(aws ec2 create-subnet \
    --vpc-id $TROUBLE_VPC_ID \
    --cidr-block 10.100.2.0/24 \
    --availability-zone us-east-1b \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=Trouble-Private-Subnet}]" \
    --query 'Subnet.SubnetId' \
    --output text)

ISOLATED_SUBNET_ID=$(aws ec2 create-subnet \
    --vpc-id $TROUBLE_VPC_ID \
    --cidr-block 10.100.3.0/24 \
    --availability-zone us-east-1a \
    --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=Trouble-Isolated-Subnet}]" \
    --query 'Subnet.SubnetId' \
    --output text)

echo "✅ Troubleshooting subnets created"

echo ""
echo "2. CREATING SECURITY GROUPS WITH VARIOUS ISSUES"

# Create security groups with intentional misconfigurations
OPEN_SG_ID=$(aws ec2 create-security-group \
    --group-name Open-Security-Group \
    --description "Overly permissive security group for troubleshooting" \
    --vpc-id $TROUBLE_VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Open-SG},{Key=Issue,Value=Overly-Permissive}]" \
    --query 'GroupId' \
    --output text)

RESTRICTIVE_SG_ID=$(aws ec2 create-security-group \
    --group-name Restrictive-Security-Group \
    --description "Overly restrictive security group for troubleshooting" \
    --vpc-id $TROUBLE_VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Restrictive-SG},{Key=Issue,Value=Too-Restrictive}]" \
    --query 'GroupId' \
    --output text)

MISCONFIGURED_SG_ID=$(aws ec2 create-security-group \
    --group-name Misconfigured-Security-Group \
    --description "Misconfigured security group for troubleshooting" \
    --vpc-id $TROUBLE_VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Misconfigured-SG},{Key=Issue,Value=Wrong-Ports}]" \
    --query 'GroupId' \
    --output text)

echo "✅ Security groups with various issues created"

echo ""
echo "3. CONFIGURING INTENTIONAL MISCONFIGURATIONS"

# Open SG - Too permissive (security issue)
aws ec2 authorize-security-group-ingress \
    --group-id $OPEN_SG_ID \
    --protocol -1 \
    --cidr 0.0.0.0/0

# Restrictive SG - No rules (connectivity issue)
# Don't add any rules to create connectivity problems

# Misconfigured SG - Wrong ports
aws ec2 authorize-security-group-ingress \
    --group-id $MISCONFIGURED_SG_ID \
    --protocol tcp \
    --port 8080 \
    --cidr 10.100.0.0/16  # App expects port 80, but we opened 8080

aws ec2 authorize-security-group-ingress \
    --group-id $MISCONFIGURED_SG_ID \
    --protocol tcp \
    --port 23 \
    --cidr 10.100.0.0/16  # Telnet instead of SSH

echo "✅ Intentional security group misconfigurations applied"

echo ""
echo "4. CREATING NETWORK ACLS WITH ISSUES"

# Create overly restrictive NACL
RESTRICTIVE_NACL_ID=$(aws ec2 create-network-acl \
    --vpc-id $TROUBLE_VPC_ID \
    --tag-specifications "ResourceType=network-acl,Tags=[{Key=Name,Value=Restrictive-NACL},{Key=Issue,Value=Blocks-Traffic}]" \
    --query 'NetworkAcl.NetworkAclId' \
    --output text)

# Add only very restrictive rules
aws ec2 create-network-acl-entry \
    --network-acl-id $RESTRICTIVE_NACL_ID \
    --rule-number 100 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=22,To=22 \
    --cidr-block 10.100.1.0/24

# Missing return traffic rule (stateless issue)
aws ec2 create-network-acl-entry \
    --network-acl-id $RESTRICTIVE_NACL_ID \
    --rule-number 100 \
    --protocol tcp \
    --rule-action allow \
    --port-range From=22,To=22 \
    --cidr-block 10.100.1.0/24 \
    --egress

echo "✅ Restrictive NACL with stateless issues created"

echo ""
echo "5. CREATING ROUTING ISSUES"

# Create Internet Gateway but don't attach (routing issue)
IGW_ID=$(aws ec2 create-internet-gateway \
    --tag-specifications "ResourceType=internet-gateway,Tags=[{Key=Name,Value=Unattached-IGW},{Key=Issue,Value=Not-Attached}]" \
    --query 'InternetGateway.InternetGatewayId' \
    --output text)

# Don't attach to VPC to create connectivity issue
echo "✅ Internet Gateway created but not attached (intentional issue)"

# Create custom route table with missing routes
INCOMPLETE_RT_ID=$(aws ec2 create-route-table \
    --vpc-id $TROUBLE_VPC_ID \
    --tag-specifications "ResourceType=route-table,Tags=[{Key=Name,Value=Incomplete-Route-Table},{Key=Issue,Value=Missing-Routes}]" \
    --query 'RouteTable.RouteTableId' \
    --output text)

# Associate with public subnet but don't add Internet route
aws ec2 associate-route-table \
    --subnet-id $PUBLIC_SUBNET_ID \
    --route-table-id $INCOMPLETE_RT_ID

echo "✅ Route table with missing routes created"

echo ""
echo "6. CREATING TEST INSTANCES WITH ISSUES"

# Get Amazon Linux 2 AMI
AMI_ID=$(aws ec2 describe-images \
    --owners amazon \
    --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
              "Name=state,Values=available" \
    --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
    --output text)

# Instance in public subnet with overly open security group
PUBLIC_INSTANCE=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.micro \
    --subnet-id $PUBLIC_SUBNET_ID \
    --security-group-ids $OPEN_SG_ID \
    --associate-public-ip-address \
    --user-data '#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
echo "<h1>Public Instance - Security Issue</h1>" > /var/www/html/index.html' \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=Public-Instance-Security-Issue}]" \
    --query 'Instances[0].InstanceId' \
    --output text)

# Instance in private subnet with restrictive security group
PRIVATE_INSTANCE=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.micro \
    --subnet-id $PRIVATE_SUBNET_ID \
    --security-group-ids $RESTRICTIVE_SG_ID \
    --user-data '#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
echo "<h1>Private Instance - Connectivity Issue</h1>" > /var/www/html/index.html' \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=Private-Instance-Connectivity-Issue}]" \
    --query 'Instances[0].InstanceId' \
    --output text)

# Instance with misconfigured security group
MISC_INSTANCE=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t3.micro \
    --subnet-id $ISOLATED_SUBNET_ID \
    --security-group-ids $MISCONFIGURED_SG_ID \
    --user-data '#!/bin/bash
yum update -y
yum install -y httpd openssh-server
systemctl start httpd
systemctl start sshd
echo "<h1>Misconfigured Instance - Wrong Ports</h1>" > /var/www/html/index.html' \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=Misconfigured-Instance}]" \
    --query 'Instances[0].InstanceId' \
    --output text)

echo "✅ Test instances with various issues created"

# Save troubleshooting lab configuration
cat << CONFIG > troubleshooting-lab-config.env
export TROUBLE_VPC_ID="$TROUBLE_VPC_ID"
export PUBLIC_SUBNET_ID="$PUBLIC_SUBNET_ID"
export PRIVATE_SUBNET_ID="$PRIVATE_SUBNET_ID"
export ISOLATED_SUBNET_ID="$ISOLATED_SUBNET_ID"
export OPEN_SG_ID="$OPEN_SG_ID"
export RESTRICTIVE_SG_ID="$RESTRICTIVE_SG_ID"
export MISCONFIGURED_SG_ID="$MISCONFIGURED_SG_ID"
export RESTRICTIVE_NACL_ID="$RESTRICTIVE_NACL_ID"
export IGW_ID="$IGW_ID"
export INCOMPLETE_RT_ID="$INCOMPLETE_RT_ID"
export PUBLIC_INSTANCE="$PUBLIC_INSTANCE"
export PRIVATE_INSTANCE="$PRIVATE_INSTANCE"
export MISC_INSTANCE="$MISC_INSTANCE"
CONFIG

echo ""
echo "TROUBLESHOOTING LAB ENVIRONMENT SUMMARY:"
echo "VPC with intentional issues: $TROUBLE_VPC_ID"
echo ""
echo "Issues Created:"
echo "1. Security Group Issues:"
echo "   - Overly permissive SG: $OPEN_SG_ID"
echo "   - Too restrictive SG: $RESTRICTIVE_SG_ID" 
echo "   - Wrong ports SG: $MISCONFIGURED_SG_ID"
echo ""
echo "2. Network ACL Issues:"
echo "   - Stateless traffic blocking: $RESTRICTIVE_NACL_ID"
echo ""
echo "3. Routing Issues:"
echo "   - Unattached Internet Gateway: $IGW_ID"
echo "   - Missing routes: $INCOMPLETE_RT_ID"
echo ""
echo "4. Instance Issues:"
echo "   - Security risk: $PUBLIC_INSTANCE"
echo "   - Connectivity problems: $PRIVATE_INSTANCE"
echo "   - Port misconfigurations: $MISC_INSTANCE"
echo ""
echo "Configuration saved to troubleshooting-lab-config.env"

EOF

chmod +x setup-troubleshooting-lab.sh
./setup-troubleshooting-lab.sh
```

### Step 2: Implement VPC Diagnostic Tools
**Objective**: Deploy comprehensive diagnostic and analysis tools

**Diagnostic Tools Implementation**:
```bash
# Implement comprehensive VPC diagnostic tools
cat << 'EOF' > implement-diagnostic-tools.sh
#!/bin/bash

echo "=== Implementing VPC Diagnostic Tools ==="
echo ""

echo "1. CREATING VPC CONNECTIVITY ANALYZER"

cat << 'CONNECTIVITY_ANALYZER' > vpc-connectivity-analyzer.py
#!/usr/bin/env python3

import boto3
import json
import time
from datetime import datetime

class VPCConnectivityAnalyzer:
    def __init__(self, vpc_id):
        self.vpc_id = vpc_id
        self.ec2 = boto3.client('ec2')
        self.results = {}
    
    def analyze_reachability(self, source_id, destination_id, destination_port=80):
        """Use VPC Reachability Analyzer to test connectivity"""
        print(f"ANALYZING REACHABILITY: {source_id} → {destination_id}:{destination_port}")
        print("=" * 60)
        
        try:
            # Create network insights path
            response = self.ec2.create_network_insights_path(
                Source=source_id,
                Destination=destination_id,
                Protocol='tcp',
                DestinationPort=destination_port,
                TagSpecifications=[
                    {
                        'ResourceType': 'network-insights-path',
                        'Tags': [
                            {'Key': 'Name', 'Value': f'Analysis-{source_id}-to-{destination_id}'}
                        ]
                    }
                ]
            )
            
            path_id = response['NetworkInsightsPath']['NetworkInsightsPathId']
            print(f"✅ Network insights path created: {path_id}")
            
            # Start analysis
            analysis_response = self.ec2.start_network_insights_analysis(
                NetworkInsightsPathId=path_id
            )
            
            analysis_id = analysis_response['NetworkInsightsAnalysis']['NetworkInsightsAnalysisId']
            print(f"✅ Analysis started: {analysis_id}")
            
            # Wait for analysis to complete
            print("Waiting for analysis to complete...")
            while True:
                status_response = self.ec2.describe_network_insights_analyses(
                    NetworkInsightsAnalysisIds=[analysis_id]
                )
                
                status = status_response['NetworkInsightsAnalyses'][0]['Status']
                print(f"Analysis status: {status}")
                
                if status == 'succeeded':
                    break
                elif status == 'failed':
                    print("❌ Analysis failed")
                    return None
                
                time.sleep(10)
            
            # Get results
            analysis_result = status_response['NetworkInsightsAnalyses'][0]
            
            print("\nANALYSIS RESULTS:")
            print("-" * 30)
            
            if analysis_result['NetworkPathFound']:
                print("✅ Network path found - Connectivity should work")
                
                if 'ForwardPath' in analysis_result:
                    print("\nForward Path Components:")
                    for component in analysis_result['ForwardPath']['Components']:
                        component_type = component.get('ComponentType', 'Unknown')
                        component_id = component.get('ComponentId', 'N/A')
                        print(f"  - {component_type}: {component_id}")
                
            else:
                print("❌ Network path not found - Connectivity blocked")
                
                if 'Explanations' in analysis_result:
                    print("\nBlocking Factors:")
                    for explanation in analysis_result['Explanations']:
                        direction = explanation.get('Direction', 'Unknown')
                        explanation_code = explanation.get('ExplanationCode', 'Unknown')
                        print(f"  - {direction}: {explanation_code}")
                        
                        if 'SecurityGroup' in explanation:
                            sg_id = explanation['SecurityGroup']['GroupId']
                            print(f"    Security Group: {sg_id}")
                        
                        if 'Acl' in explanation:
                            acl_id = explanation['Acl']['Id']
                            print(f"    Network ACL: {acl_id}")
            
            self.results[f"{source_id}-{destination_id}"] = analysis_result
            
            # Cleanup
            self.ec2.delete_network_insights_analysis(NetworkInsightsAnalysisId=analysis_id)
            self.ec2.delete_network_insights_path(NetworkInsightsPathId=path_id)
            
            return analysis_result
            
        except Exception as e:
            print(f"Error in reachability analysis: {e}")
            return None
    
    def analyze_security_groups(self, instance_id):
        """Analyze security group configurations for an instance"""
        print(f"\nSECURITY GROUP ANALYSIS FOR: {instance_id}")
        print("=" * 50)
        
        try:
            # Get instance details
            response = self.ec2.describe_instances(InstanceIds=[instance_id])
            instance = response['Reservations'][0]['Instances'][0]
            
            security_groups = instance['SecurityGroups']
            
            for sg in security_groups:
                sg_id = sg['GroupId']
                sg_name = sg['GroupName']
                
                print(f"\nSecurity Group: {sg_name} ({sg_id})")
                print("-" * 40)
                
                # Get detailed SG rules
                sg_details = self.ec2.describe_security_groups(GroupIds=[sg_id])
                sg_data = sg_details['SecurityGroups'][0]
                
                print("Inbound Rules:")
                if sg_data['IpPermissions']:
                    for rule in sg_data['IpPermissions']:
                        self.print_security_group_rule(rule, 'Inbound')
                else:
                    print("  No inbound rules (all traffic blocked)")
                
                print("\nOutbound Rules:")
                if sg_data['IpPermissionsEgress']:
                    for rule in sg_data['IpPermissionsEgress']:
                        self.print_security_group_rule(rule, 'Outbound')
                else:
                    print("  No outbound rules (all traffic blocked)")
                
                # Security analysis
                self.analyze_security_group_risks(sg_data)
                
        except Exception as e:
            print(f"Error analyzing security groups: {e}")
    
    def print_security_group_rule(self, rule, direction):
        """Print formatted security group rule"""
        protocol = rule.get('IpProtocol', 'All')
        
        if protocol == '-1':
            protocol = 'All'
            port_range = 'All'
        else:
            from_port = rule.get('FromPort', 0)
            to_port = rule.get('ToPort', 0)
            if from_port == to_port:
                port_range = str(from_port)
            else:
                port_range = f"{from_port}-{to_port}"
        
        # Sources/Destinations
        sources = []
        
        for ip_range in rule.get('IpRanges', []):
            cidr = ip_range['CidrIp']
            sources.append(cidr)
        
        for sg_ref in rule.get('UserIdGroupPairs', []):
            sg_id = sg_ref['GroupId']
            sources.append(f"SG:{sg_id}")
        
        for prefix in rule.get('PrefixListIds', []):
            prefix_id = prefix['PrefixListId']
            sources.append(f"PL:{prefix_id}")
        
        sources_str = ', '.join(sources) if sources else 'None'
        
        print(f"  {protocol:8} {port_range:15} {sources_str}")
    
    def analyze_security_group_risks(self, sg_data):
        """Analyze security group for potential risks"""
        risks = []
        
        for rule in sg_data['IpPermissions']:
            # Check for overly permissive rules
            for ip_range in rule.get('IpRanges', []):
                if ip_range['CidrIp'] == '0.0.0.0/0':
                    protocol = rule.get('IpProtocol', 'All')
                    if protocol == '-1':
                        risks.append("⚠️  CRITICAL: All traffic from anywhere (0.0.0.0/0)")
                    elif rule.get('FromPort') == 22:
                        risks.append("⚠️  HIGH: SSH access from anywhere")
                    elif rule.get('FromPort') == 3389:
                        risks.append("⚠️  HIGH: RDP access from anywhere")
                    else:
                        risks.append(f"⚠️  MEDIUM: Port {rule.get('FromPort', 'All')} from anywhere")
        
        if risks:
            print("\nSecurity Risks Identified:")
            for risk in risks:
                print(f"  {risk}")
        else:
            print("\n✅ No obvious security risks detected")
    
    def analyze_network_acls(self, subnet_id):
        """Analyze Network ACL configurations for a subnet"""
        print(f"\nNETWORK ACL ANALYSIS FOR SUBNET: {subnet_id}")
        print("=" * 50)
        
        try:
            # Get subnet details
            response = self.ec2.describe_subnets(SubnetIds=[subnet_id])
            subnet = response['Subnets'][0]
            
            # Get associated Network ACL
            nacl_response = self.ec2.describe_network_acls(
                Filters=[
                    {'Name': 'association.subnet-id', 'Values': [subnet_id]}
                ]
            )
            
            nacl = nacl_response['NetworkAcls'][0]
            nacl_id = nacl['NetworkAclId']
            
            print(f"Network ACL: {nacl_id}")
            print("-" * 30)
            
            # Analyze inbound rules
            print("Inbound Rules:")
            inbound_rules = [entry for entry in nacl['Entries'] if not entry['Egress']]
            inbound_rules.sort(key=lambda x: x['RuleNumber'])
            
            for rule in inbound_rules:
                self.print_nacl_rule(rule)
            
            print("\nOutbound Rules:")
            outbound_rules = [entry for entry in nacl['Entries'] if entry['Egress']]
            outbound_rules.sort(key=lambda x: x['RuleNumber'])
            
            for rule in outbound_rules:
                self.print_nacl_rule(rule)
            
            # Check for common NACL issues
            self.analyze_nacl_issues(nacl)
            
        except Exception as e:
            print(f"Error analyzing Network ACLs: {e}")
    
    def print_nacl_rule(self, rule):
        """Print formatted NACL rule"""
        rule_number = rule['RuleNumber']
        protocol = rule['Protocol']
        action = rule['RuleAction']
        cidr = rule['CidrBlock']
        
        # Convert protocol number to name
        if protocol == '-1':
            protocol_name = 'All'
            port_range = 'All'
        elif protocol == '6':
            protocol_name = 'TCP'
            port_range = f"{rule.get('PortRange', {}).get('From', 'All')}-{rule.get('PortRange', {}).get('To', 'All')}"
        elif protocol == '17':
            protocol_name = 'UDP'
            port_range = f"{rule.get('PortRange', {}).get('From', 'All')}-{rule.get('PortRange', {}).get('To', 'All')}"
        elif protocol == '1':
            protocol_name = 'ICMP'
            port_range = 'All'
        else:
            protocol_name = f"Protocol-{protocol}"
            port_range = 'All'
        
        print(f"  {rule_number:5} {action:6} {protocol_name:8} {port_range:15} {cidr}")
    
    def analyze_nacl_issues(self, nacl):
        """Analyze NACL for common issues"""
        issues = []
        
        entries = nacl['Entries']
        
        # Check for missing return traffic rules
        inbound_rules = [e for e in entries if not e['Egress'] and e['RuleAction'] == 'allow']
        outbound_rules = [e for e in entries if e['Egress'] and e['RuleAction'] == 'allow']
        
        if inbound_rules and not outbound_rules:
            issues.append("⚠️  Missing outbound rules for return traffic")
        
        # Check for conflicting rules
        rule_numbers = [e['RuleNumber'] for e in entries]
        if len(rule_numbers) != len(set(rule_numbers)):
            issues.append("⚠️  Duplicate rule numbers detected")
        
        # Check for overly restrictive configuration
        allow_rules = [e for e in entries if e['RuleAction'] == 'allow']
        if len(allow_rules) < 2:  # At least inbound and outbound
            issues.append("⚠️  Very restrictive NACL - may block legitimate traffic")
        
        if issues:
            print("\nPotential NACL Issues:")
            for issue in issues:
                print(f"  {issue}")
        else:
            print("\n✅ No obvious NACL issues detected")

def main():
    import sys
    
    if len(sys.argv) < 2:
        print("Usage: python3 vpc-connectivity-analyzer.py <vpc-id> [source-instance] [dest-instance]")
        sys.exit(1)
    
    vpc_id = sys.argv[1]
    analyzer = VPCConnectivityAnalyzer(vpc_id)
    
    if len(sys.argv) >= 4:
        source_instance = sys.argv[2]
        dest_instance = sys.argv[3]
        
        # Perform reachability analysis
        analyzer.analyze_reachability(source_instance, dest_instance)
    
    # If instances provided, analyze their configurations
    if len(sys.argv) >= 3:
        instance_id = sys.argv[2]
        analyzer.analyze_security_groups(instance_id)
        
        # Get instance subnet for NACL analysis
        try:
            response = analyzer.ec2.describe_instances(InstanceIds=[instance_id])
            subnet_id = response['Reservations'][0]['Instances'][0]['SubnetId']
            analyzer.analyze_network_acls(subnet_id)
        except:
            print("Could not retrieve subnet information for NACL analysis")

if __name__ == "__main__":
    main()

CONNECTIVITY_ANALYZER

chmod +x vpc-connectivity-analyzer.py

echo "✅ VPC connectivity analyzer created"

echo ""
echo "2. CREATING VPC FLOW LOGS ANALYZER"

cat << 'FLOW_ANALYZER' > vpc-flow-logs-analyzer.py
#!/usr/bin/env python3

import boto3
import json
from datetime import datetime, timedelta
import ipaddress

class VPCFlowLogsAnalyzer:
    def __init__(self, log_group_name):
        self.log_group_name = log_group_name
        self.logs_client = boto3.client('logs')
        self.ec2 = boto3.client('ec2')
    
    def analyze_rejected_traffic(self, hours=1):
        """Analyze rejected traffic patterns"""
        print("REJECTED TRAFFIC ANALYSIS:")
        print("=" * 40)
        
        end_time = datetime.now()
        start_time = end_time - timedelta(hours=hours)
        
        query = """
        fields @timestamp, srcaddr, dstaddr, srcport, dstport, protocol, action
        | filter action = "REJECT"
        | stats count() as reject_count by srcaddr, dstaddr, dstport
        | sort reject_count desc
        | limit 20
        """
        
        results = self.run_insights_query(query, start_time, end_time)
        
        if results:
            print("Top Rejected Connections:")
            print(f"{'Source IP':<15} {'Dest IP':<15} {'Port':<6} {'Count':<8}")
            print("-" * 50)
            
            for result in results:
                fields = {item['field']: item['value'] for item in result}
                srcaddr = fields.get('srcaddr', 'N/A')
                dstaddr = fields.get('dstaddr', 'N/A')
                dstport = fields.get('dstport', 'N/A')
                count = fields.get('reject_count', '0')
                
                print(f"{srcaddr:<15} {dstaddr:<15} {dstport:<6} {count:<8}")
        else:
            print("No rejected traffic found in the specified time period")
    
    def analyze_top_talkers(self, hours=1):
        """Analyze top talking hosts by data volume"""
        print("\nTOP TALKERS ANALYSIS:")
        print("=" * 30)
        
        end_time = datetime.now()
        start_time = end_time - timedelta(hours=hours)
        
        query = """
        fields @timestamp, srcaddr, dstaddr, bytes, action
        | filter action = "ACCEPT"
        | stats sum(bytes) as total_bytes by srcaddr, dstaddr
        | sort total_bytes desc
        | limit 15
        """
        
        results = self.run_insights_query(query, start_time, end_time)
        
        if results:
            print("Top Data Transfers:")
            print(f"{'Source IP':<15} {'Dest IP':<15} {'Bytes':<12}")
            print("-" * 45)
            
            for result in results:
                fields = {item['field']: item['value'] for item in result}
                srcaddr = fields.get('srcaddr', 'N/A')
                dstaddr = fields.get('dstaddr', 'N/A')
                total_bytes = int(fields.get('total_bytes', 0))
                
                # Convert bytes to human readable format
                if total_bytes > 1024**3:
                    bytes_str = f"{total_bytes/1024**3:.2f} GB"
                elif total_bytes > 1024**2:
                    bytes_str = f"{total_bytes/1024**2:.2f} MB"
                elif total_bytes > 1024:
                    bytes_str = f"{total_bytes/1024:.2f} KB"
                else:
                    bytes_str = f"{total_bytes} B"
                
                print(f"{srcaddr:<15} {dstaddr:<15} {bytes_str:<12}")
        else:
            print("No traffic data found in the specified time period")
    
    def analyze_port_usage(self, hours=1):
        """Analyze most commonly used ports"""
        print("\nPORT USAGE ANALYSIS:")
        print("=" * 25)
        
        end_time = datetime.now()
        start_time = end_time - timedelta(hours=hours)
        
        query = """
        fields @timestamp, dstport, protocol, action
        | filter action = "ACCEPT" and dstport != "-"
        | stats count() as connection_count by dstport, protocol
        | sort connection_count desc
        | limit 15
        """
        
        results = self.run_insights_query(query, start_time, end_time)
        
        if results:
            print("Most Used Ports:")
            print(f"{'Port':<6} {'Protocol':<10} {'Connections':<12}")
            print("-" * 30)
            
            for result in results:
                fields = {item['field']: item['value'] for item in result}
                dstport = fields.get('dstport', 'N/A')
                protocol = self.protocol_number_to_name(fields.get('protocol', '0'))
                count = fields.get('connection_count', '0')
                
                print(f"{dstport:<6} {protocol:<10} {count:<12}")
        else:
            print("No port usage data found")
    
    def detect_security_anomalies(self, hours=1):
        """Detect potential security anomalies"""
        print("\nSECURITY ANOMALY DETECTION:")
        print("=" * 35)
        
        anomalies = []
        
        # Check for port scanning
        end_time = datetime.now()
        start_time = end_time - timedelta(hours=hours)
        
        port_scan_query = """
        fields @timestamp, srcaddr, dstaddr, dstport, action
        | filter action = "REJECT"
        | stats count_distinct(dstport) as unique_ports by srcaddr, dstaddr
        | sort unique_ports desc
        | limit 10
        """
        
        port_scan_results = self.run_insights_query(port_scan_query, start_time, end_time)
        
        for result in port_scan_results:
            fields = {item['field']: item['value'] for item in result}
            unique_ports = int(fields.get('unique_ports', 0))
            if unique_ports > 10:  # Potential port scan
                srcaddr = fields.get('srcaddr', 'N/A')
                dstaddr = fields.get('dstaddr', 'N/A')
                anomalies.append(f"Potential port scan: {srcaddr} → {dstaddr} ({unique_ports} ports)")
        
        # Check for unusual data volumes
        large_transfer_query = """
        fields @timestamp, srcaddr, dstaddr, bytes
        | filter bytes > 100000000
        | stats count() as large_transfers, sum(bytes) as total_bytes by srcaddr, dstaddr
        | sort total_bytes desc
        | limit 5
        """
        
        large_transfer_results = self.run_insights_query(large_transfer_query, start_time, end_time)
        
        for result in large_transfer_results:
            fields = {item['field']: item['value'] for item in result}
            total_bytes = int(fields.get('total_bytes', 0))
            if total_bytes > 1024**3:  # > 1GB
                srcaddr = fields.get('srcaddr', 'N/A')
                dstaddr = fields.get('dstaddr', 'N/A')
                gb_transferred = total_bytes / 1024**3
                anomalies.append(f"Large data transfer: {srcaddr} → {dstaddr} ({gb_transferred:.2f} GB)")
        
        if anomalies:
            print("Detected Anomalies:")
            for anomaly in anomalies:
                print(f"  ⚠️  {anomaly}")
        else:
            print("✅ No security anomalies detected")
    
    def protocol_number_to_name(self, protocol_num):
        """Convert protocol number to name"""
        protocol_map = {
            '1': 'ICMP',
            '6': 'TCP', 
            '17': 'UDP',
            '47': 'GRE',
            '50': 'ESP',
            '51': 'AH'
        }
        return protocol_map.get(str(protocol_num), f"Protocol-{protocol_num}")
    
    def run_insights_query(self, query, start_time, end_time):
        """Run CloudWatch Logs Insights query"""
        try:
            response = self.logs_client.start_query(
                logGroupName=self.log_group_name,
                startTime=int(start_time.timestamp()),
                endTime=int(end_time.timestamp()),
                queryString=query
            )
            
            query_id = response['queryId']
            
            # Wait for query completion
            import time
            while True:
                response = self.logs_client.get_query_results(queryId=query_id)
                if response['status'] == 'Complete':
                    return response['results']
                elif response['status'] == 'Failed':
                    print(f"Query failed: {response}")
                    return []
                time.sleep(2)
                
        except Exception as e:
            print(f"Error running insights query: {e}")
            return []
    
    def generate_summary_report(self, hours=24):
        """Generate comprehensive summary report"""
        print(f"\nVPC FLOW LOGS SUMMARY REPORT (Last {hours} hours)")
        print("=" * 60)
        print(f"Report generated: {datetime.now()}")
        print(f"Log group: {self.log_group_name}")
        print("")
        
        self.analyze_rejected_traffic(hours)
        self.analyze_top_talkers(hours)
        self.analyze_port_usage(hours)
        self.detect_security_anomalies(hours)

def main():
    import sys
    
    if len(sys.argv) < 2:
        print("Usage: python3 vpc-flow-logs-analyzer.py <log-group-name> [hours]")
        print("Example: python3 vpc-flow-logs-analyzer.py /aws/vpc/flowlogs 24")
        sys.exit(1)
    
    log_group_name = sys.argv[1]
    hours = int(sys.argv[2]) if len(sys.argv) > 2 else 1
    
    analyzer = VPCFlowLogsAnalyzer(log_group_name)
    analyzer.generate_summary_report(hours)

if __name__ == "__main__":
    main()

FLOW_ANALYZER

chmod +x vpc-flow-logs-analyzer.py

echo "✅ VPC Flow Logs analyzer created"

echo ""
echo "DIAGNOSTIC TOOLS IMPLEMENTATION COMPLETED"
echo ""
echo "Available Tools:"
echo "1. vpc-connectivity-analyzer.py - Comprehensive connectivity analysis"
echo "2. vpc-flow-logs-analyzer.py - Flow logs analysis and anomaly detection"
echo ""
echo "Usage Examples:"
echo "python3 vpc-connectivity-analyzer.py <vpc-id> <instance-id>"
echo "python3 vpc-connectivity-analyzer.py <vpc-id> <source-instance> <dest-instance>"
echo "python3 vpc-flow-logs-analyzer.py /aws/vpc/flowlogs 24"

EOF

chmod +x implement-diagnostic-tools.sh
./implement-diagnostic-tools.sh
```

### Step 3: Create Systematic Troubleshooting Framework
**Objective**: Implement structured troubleshooting methodologies

**Troubleshooting Framework Implementation**:
```bash
# Create systematic troubleshooting framework
cat << 'EOF' > create-troubleshooting-framework.sh
#!/bin/bash

echo "=== Creating Systematic Troubleshooting Framework ==="
echo ""

echo "1. CREATING TROUBLESHOOTING DECISION TREE"

cat << 'DECISION_TREE' > troubleshooting-decision-tree.py
#!/usr/bin/env python3

import boto3
import json

class TroubleshootingDecisionTree:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
        self.troubleshooting_steps = []
    
    def start_troubleshooting(self, problem_description):
        """Start systematic troubleshooting process"""
        print("VPC TROUBLESHOOTING DECISION TREE")
        print("=" * 40)
        print(f"Problem: {problem_description}")
        print("")
        
        # Categorize the problem
        problem_category = self.categorize_problem(problem_description)
        print(f"Problem Category: {problem_category}")
        print("")
        
        # Execute appropriate troubleshooting flow
        if problem_category == "connectivity":
            self.troubleshoot_connectivity()
        elif problem_category == "performance":
            self.troubleshoot_performance()
        elif problem_category == "security":
            self.troubleshoot_security()
        elif problem_category == "dns":
            self.troubleshoot_dns()
        else:
            self.troubleshoot_general()
    
    def categorize_problem(self, description):
        """Categorize the problem based on description"""
        description = description.lower()
        
        connectivity_keywords = ['connect', 'reach', 'timeout', 'refused', 'unreachable']
        performance_keywords = ['slow', 'latency', 'performance', 'bandwidth', 'throughput']
        security_keywords = ['blocked', 'denied', 'unauthorized', 'forbidden']
        dns_keywords = ['dns', 'resolve', 'name', 'domain']
        
        if any(keyword in description for keyword in connectivity_keywords):
            return "connectivity"
        elif any(keyword in description for keyword in performance_keywords):
            return "performance"
        elif any(keyword in description for keyword in security_keywords):
            return "security"
        elif any(keyword in description for keyword in dns_keywords):
            return "dns"
        else:
            return "general"
    
    def troubleshoot_connectivity(self):
        """Systematic connectivity troubleshooting"""
        print("CONNECTIVITY TROUBLESHOOTING FLOW:")
        print("-" * 40)
        
        steps = [
            {
                'step': 1,
                'description': 'Verify basic instance health',
                'checks': [
                    'Instance state is running',
                    'Instance has private IP assigned',
                    'Instance security groups attached',
                    'Instance in correct subnet'
                ],
                'commands': [
                    'aws ec2 describe-instances --instance-ids <instance-id>',
                    'aws ec2 describe-instance-status --instance-ids <instance-id>'
                ]
            },
            {
                'step': 2,
                'description': 'Check security group rules',
                'checks': [
                    'Required ports are open',
                    'Source restrictions are correct',
                    'Outbound rules allow return traffic',
                    'No conflicting rules'
                ],
                'commands': [
                    'aws ec2 describe-security-groups --group-ids <sg-id>',
                    'python3 vpc-connectivity-analyzer.py <vpc-id> <instance-id>'
                ]
            },
            {
                'step': 3,
                'description': 'Check Network ACL rules',
                'checks': [
                    'NACL allows required traffic',
                    'Return traffic rules exist',
                    'Rule numbers are correct',
                    'No blocking deny rules'
                ],
                'commands': [
                    'aws ec2 describe-network-acls --filters "Name=association.subnet-id,Values=<subnet-id>"'
                ]
            },
            {
                'step': 4,
                'description': 'Verify routing configuration',
                'checks': [
                    'Route to destination exists',
                    'Route table associated with subnet',
                    'Internet gateway attached (if needed)',
                    'NAT gateway functioning (if needed)'
                ],
                'commands': [
                    'aws ec2 describe-route-tables --filters "Name=association.subnet-id,Values=<subnet-id>"',
                    'aws ec2 describe-internet-gateways',
                    'aws ec2 describe-nat-gateways'
                ]
            },
            {
                'step': 5,
                'description': 'Test actual connectivity',
                'checks': [
                    'Ping test (if ICMP allowed)',
                    'Port connectivity test',
                    'Application-specific test',
                    'Bidirectional connectivity'
                ],
                'commands': [
                    'ping <destination-ip>',
                    'telnet <destination-ip> <port>',
                    'nc -zv <destination-ip> <port>'
                ]
            }
        ]
        
        for step in steps:
            print(f"\nStep {step['step']}: {step['description']}")
            print("Checks:")
            for check in step['checks']:
                print(f"  □ {check}")
            print("Commands:")
            for command in step['commands']:
                print(f"  $ {command}")
            
            self.troubleshooting_steps.append(step)
    
    def troubleshoot_performance(self):
        """Systematic performance troubleshooting"""
        print("PERFORMANCE TROUBLESHOOTING FLOW:")
        print("-" * 40)
        
        steps = [
            {
                'step': 1,
                'description': 'Baseline performance measurement',
                'checks': [
                    'Current latency measurements',
                    'Throughput tests',
                    'Packet loss assessment',
                    'Jitter analysis'
                ],
                'tools': [
                    'ping for latency',
                    'iperf3 for throughput',
                    'mtr for path analysis',
                    'traceroute for routing'
                ]
            },
            {
                'step': 2,
                'description': 'Instance and network configuration',
                'checks': [
                    'Instance type supports required performance',
                    'Enhanced networking enabled',
                    'Placement group configuration',
                    'Network interface optimization'
                ],
                'tools': [
                    'Instance type specifications',
                    'ENA driver status',
                    'SR-IOV support verification'
                ]
            },
            {
                'step': 3,
                'description': 'Network path analysis',
                'checks': [
                    'Optimal routing path',
                    'No unnecessary hops',
                    'MTU configuration',
                    'Bandwidth limitations'
                ],
                'tools': [
                    'traceroute/tracepath',
                    'MTU discovery',
                    'Bandwidth testing'
                ]
            }
        ]
        
        for step in steps:
            print(f"\nStep {step['step']}: {step['description']}")
            print("Checks:")
            for check in step['checks']:
                print(f"  □ {check}")
            print("Tools:")
            for tool in step['tools']:
                print(f"  • {tool}")
    
    def troubleshoot_security(self):
        """Systematic security troubleshooting"""
        print("SECURITY TROUBLESHOOTING FLOW:")
        print("-" * 35)
        
        print("1. Identify Security Controls:")
        print("  □ Security Groups blocking traffic")
        print("  □ Network ACLs denying access")
        print("  □ IAM policies restricting actions")
        print("  □ VPC endpoints access policies")
        print("")
        print("2. Analyze Access Patterns:")
        print("  □ Review VPC Flow Logs for REJECT actions")
        print("  □ Check CloudTrail for permission denials")
        print("  □ Verify source IP restrictions")
        print("  □ Validate time-based access controls")
    
    def troubleshoot_dns(self):
        """Systematic DNS troubleshooting"""
        print("DNS TROUBLESHOOTING FLOW:")
        print("-" * 30)
        
        print("1. DNS Configuration Verification:")
        print("  □ VPC DNS resolution enabled")
        print("  □ VPC DNS hostnames enabled")
        print("  □ Route 53 Resolver configuration")
        print("  □ Custom DNS servers configured")
        print("")
        print("2. DNS Resolution Testing:")
        print("  □ Forward DNS resolution")
        print("  □ Reverse DNS resolution")
        print("  □ Private hosted zone queries")
        print("  □ External DNS queries")
    
    def troubleshoot_general(self):
        """General troubleshooting approach"""
        print("GENERAL TROUBLESHOOTING APPROACH:")
        print("-" * 40)
        
        print("1. Problem Scoping:")
        print("  □ When did the issue start?")
        print("  □ What changed recently?")
        print("  □ Is it affecting all users/services?")
        print("  □ Are there error messages?")
        print("")
        print("2. Layer-by-Layer Analysis:")
        print("  □ Physical/Infrastructure (AWS)")
        print("  □ Network (Routing, Security)")
        print("  □ Transport (TCP/UDP)")
        print("  □ Application (Service-specific)")
    
    def generate_troubleshooting_report(self):
        """Generate troubleshooting report"""
        print("\nTROUBLESHOoting SESSION SUMMARY:")
        print("=" * 40)
        print(f"Steps completed: {len(self.troubleshooting_steps)}")
        print(f"Report generated: {datetime.now()}")
        
        for i, step in enumerate(self.troubleshooting_steps, 1):
            print(f"\n{i}. {step['description']}")
            print(f"   Status: [  ] Completed")

def main():
    import sys
    
    if len(sys.argv) < 2:
        print("Usage: python3 troubleshooting-decision-tree.py '<problem-description>'")
        print("Example: python3 troubleshooting-decision-tree.py 'Cannot connect to web server'")
        sys.exit(1)
    
    problem_description = sys.argv[1]
    
    troubleshooter = TroubleshootingDecisionTree()
    troubleshooter.start_troubleshooting(problem_description)
    troubleshooter.generate_troubleshooting_report()

if __name__ == "__main__":
    main()

DECISION_TREE

chmod +x troubleshooting-decision-tree.py

echo "✅ Troubleshooting decision tree created"

echo ""
echo "2. CREATING AUTOMATED DIAGNOSTIC SCRIPT"

cat << 'AUTO_DIAG' > automated-vpc-diagnostics.sh
#!/bin/bash

echo "=== Automated VPC Diagnostics ==="
echo ""

VPC_ID="$1"
INSTANCE_ID="$2"

if [ -z "$VPC_ID" ]; then
    echo "Usage: $0 <vpc-id> [instance-id]"
    echo "Example: $0 vpc-12345678 i-12345678"
    exit 1
fi

echo "Running automated diagnostics for VPC: $VPC_ID"
if [ -n "$INSTANCE_ID" ]; then
    echo "Focusing on instance: $INSTANCE_ID"
fi
echo ""

echo "1. VPC BASIC INFORMATION"
echo "========================"
aws ec2 describe-vpcs --vpc-ids $VPC_ID --query 'Vpcs[0].[VpcId,CidrBlock,State,DhcpOptionsId]' --output table

echo ""
echo "2. SUBNETS IN VPC"
echo "================="
aws ec2 describe-subnets --filters "Name=vpc-id,Values=$VPC_ID" --query 'Subnets[*].[SubnetId,CidrBlock,AvailabilityZone,MapPublicIpOnLaunch]' --output table

echo ""
echo "3. INTERNET GATEWAYS"
echo "===================="
aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=$VPC_ID" --query 'InternetGateways[*].[InternetGatewayId,State,Attachments[0].State]' --output table

echo ""
echo "4. NAT GATEWAYS"
echo "==============="
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=$VPC_ID" --query 'NatGateways[*].[NatGatewayId,State,SubnetId,PublicIp]' --output table

echo ""
echo "5. ROUTE TABLES"
echo "==============="
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=$VPC_ID" --query 'RouteTables[*].[RouteTableId,Routes[0].DestinationCidrBlock,Routes[0].GatewayId,Associations[0].SubnetId]' --output table

echo ""
echo "6. SECURITY GROUPS"
echo "=================="
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=$VPC_ID" --query 'SecurityGroups[*].[GroupId,GroupName,Description]' --output table

echo ""
echo "7. NETWORK ACLS"
echo "==============="
aws ec2 describe-network-acls --filters "Name=vpc-id,Values=$VPC_ID" --query 'NetworkAcls[*].[NetworkAclId,IsDefault,Associations[0].SubnetId]' --output table

if [ -n "$INSTANCE_ID" ]; then
    echo ""
    echo "8. INSTANCE-SPECIFIC DIAGNOSTICS"
    echo "================================="
    
    echo "Instance Details:"
    aws ec2 describe-instances --instance-ids $INSTANCE_ID --query 'Reservations[0].Instances[0].[InstanceId,State.Name,SubnetId,VpcId,SecurityGroups[0].GroupId]' --output table
    
    echo ""
    echo "Instance Security Groups:"
    aws ec2 describe-instances --instance-ids $INSTANCE_ID --query 'Reservations[0].Instances[0].SecurityGroups[*].[GroupId,GroupName]' --output table
    
    echo ""
    echo "Instance Network Interfaces:"
    aws ec2 describe-instances --instance-ids $INSTANCE_ID --query 'Reservations[0].Instances[0].NetworkInterfaces[*].[NetworkInterfaceId,PrivateIpAddress,Association.PublicIp]' --output table
fi

echo ""
echo "9. VPC ENDPOINTS"
echo "================"
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=$VPC_ID" --query 'VpcEndpoints[*].[VpcEndpointId,ServiceName,State]' --output table

echo ""
echo "10. FLOW LOGS STATUS"
echo "===================="
aws ec2 describe-flow-logs --filter "Name=resource-id,Values=$VPC_ID" --query 'FlowLogs[*].[FlowLogId,FlowLogStatus,LogDestination]' --output table

echo ""
echo "DIAGNOSTIC SUMMARY"
echo "=================="
echo "✅ Basic VPC information collected"
echo "✅ Network infrastructure analyzed"
echo "✅ Security configurations reviewed"
if [ -n "$INSTANCE_ID" ]; then
    echo "✅ Instance-specific details gathered"
fi
echo ""
echo "Next Steps:"
echo "1. Review any 'detached' or 'failed' states above"
echo "2. Check for missing routes or security group rules"
echo "3. Use vpc-connectivity-analyzer.py for detailed analysis"
echo "4. Review VPC Flow Logs for traffic patterns"

AUTO_DIAG

chmod +x automated-vpc-diagnostics.sh

echo "✅ Automated diagnostic script created"

echo ""
echo "TROUBLESHOOTING FRAMEWORK IMPLEMENTATION COMPLETED"
echo ""
echo "Available Framework Components:"
echo "1. troubleshooting-decision-tree.py - Systematic problem categorization"
echo "2. automated-vpc-diagnostics.sh - Comprehensive VPC health check"
echo ""
echo "Usage Examples:"
echo "python3 troubleshooting-decision-tree.py 'Cannot connect to database'"
echo "./automated-vpc-diagnostics.sh vpc-12345678"
echo "./automated-vpc-diagnostics.sh vpc-12345678 i-12345678"

EOF

chmod +x create-troubleshooting-framework.sh
./create-troubleshooting-framework.sh
```

### Step 4: Implement Performance Troubleshooting Tools
**Objective**: Create specialized tools for network performance diagnosis

**Performance Troubleshooting Tools**:
```bash
# Create specialized performance troubleshooting tools
cat << 'EOF' > implement-performance-troubleshooting.sh
#!/bin/bash

echo "=== Implementing Performance Troubleshooting Tools ==="
echo ""

echo "1. CREATING NETWORK PERFORMANCE DIAGNOSTIC TOOL"

cat << 'PERF_DIAG' > network-performance-diagnostics.py
#!/usr/bin/env python3

import subprocess
import psutil
import time
import json
from datetime import datetime

class NetworkPerformanceDiagnostics:
    def __init__(self):
        self.results = {}
        self.baseline_metrics = None
    
    def run_comprehensive_diagnostics(self, target_host=None):
        """Run comprehensive network performance diagnostics"""
        print("NETWORK PERFORMANCE DIAGNOSTICS")
        print("=" * 40)
        print(f"Diagnostic started: {datetime.now()}")
        
        if target_host:
            print(f"Target host: {target_host}")
        print("")
        
        # Gather baseline metrics
        self.baseline_metrics = self.get_baseline_metrics()
        
        # Run diagnostic tests
        self.check_interface_status()
        self.check_enhanced_networking()
        self.analyze_network_buffers()
        self.check_tcp_settings()
        
        if target_host:
            self.test_connectivity(target_host)
            self.measure_latency(target_host)
            self.test_throughput(target_host)
        
        self.generate_performance_report()
    
    def get_baseline_metrics(self):
        """Get baseline network metrics"""
        try:
            net_io = psutil.net_io_counters()
            return {
                'bytes_sent': net_io.bytes_sent,
                'bytes_recv': net_io.bytes_recv,
                'packets_sent': net_io.packets_sent,
                'packets_recv': net_io.packets_recv,
                'errin': net_io.errin,
                'errout': net_io.errout,
                'dropin': net_io.dropin,
                'dropout': net_io.dropout
            }
        except Exception as e:
            print(f"Error getting baseline metrics: {e}")
            return {}
    
    def check_interface_status(self):
        """Check network interface status and configuration"""
        print("1. NETWORK INTERFACE STATUS:")
        print("-" * 30)
        
        try:
            # Get interface information
            result = subprocess.run(['ip', 'link', 'show'], capture_output=True, text=True)
            
            interfaces = []
            current_interface = None
            
            for line in result.stdout.split('\n'):
                if ': ' in line and 'state' in line.upper():
                    parts = line.split(': ')
                    if len(parts) >= 2:
                        interface_name = parts[1].split('@')[0]
                        state = 'UP' if 'state UP' in line else 'DOWN'
                        
                        # Get MTU
                        mtu = 'Unknown'
                        if 'mtu' in line:
                            mtu_part = line.split('mtu')[1].split()[0]
                            mtu = mtu_part
                        
                        interfaces.append({
                            'name': interface_name,
                            'state': state,
                            'mtu': mtu
                        })
            
            print(f"{'Interface':<12} {'State':<8} {'MTU':<8}")
            print("-" * 30)
            
            for interface in interfaces:
                print(f"{interface['name']:<12} {interface['state']:<8} {interface['mtu']:<8}")
                
                # Check for issues
                if interface['state'] == 'DOWN' and interface['name'] != 'lo':
                    self.results['interface_issues'] = f"Interface {interface['name']} is DOWN"
                
                if interface['name'] == 'eth0' and interface['mtu'] != '9000':
                    self.results['mtu_optimization'] = f"MTU is {interface['mtu']}, consider jumbo frames (9000)"
        
        except Exception as e:
            print(f"Error checking interface status: {e}")
    
    def check_enhanced_networking(self):
        """Check enhanced networking status"""
        print("\n2. ENHANCED NETWORKING STATUS:")
        print("-" * 35)
        
        try:
            # Check for ENA driver
            ena_result = subprocess.run(['lsmod'], capture_output=True, text=True)
            if 'ena' in ena_result.stdout:
                print("✅ ENA driver loaded")
                self.results['ena_status'] = 'enabled'
            else:
                print("❌ ENA driver not found")
                self.results['ena_status'] = 'disabled'
                self.results['ena_issue'] = 'ENA driver not loaded - performance may be suboptimal'
            
            # Check SR-IOV support
            sriov_result = subprocess.run(['lspci', '-vvv'], capture_output=True, text=True)
            if 'SR-IOV' in sriov_result.stdout:
                print("✅ SR-IOV capabilities detected")
                self.results['sriov_status'] = 'supported'
            else:
                print("❌ SR-IOV not detected")
                self.results['sriov_status'] = 'not_supported'
        
        except Exception as e:
            print(f"Error checking enhanced networking: {e}")
    
    def analyze_network_buffers(self):
        """Analyze network buffer configurations"""
        print("\n3. NETWORK BUFFER ANALYSIS:")
        print("-" * 32)
        
        buffer_files = {
            'rmem_default': '/proc/sys/net/core/rmem_default',
            'rmem_max': '/proc/sys/net/core/rmem_max',
            'wmem_default': '/proc/sys/net/core/wmem_default',
            'wmem_max': '/proc/sys/net/core/wmem_max',
            'netdev_max_backlog': '/proc/sys/net/core/netdev_max_backlog'
        }
        
        optimal_values = {
            'rmem_default': 262144,
            'rmem_max': 67108864,
            'wmem_default': 262144,
            'wmem_max': 67108864,
            'netdev_max_backlog': 5000
        }
        
        print(f"{'Parameter':<20} {'Current':<12} {'Optimal':<12} {'Status':<8}")
        print("-" * 55)
        
        buffer_issues = []
        
        for param, file_path in buffer_files.items():
            try:
                with open(file_path, 'r') as f:
                    current_value = int(f.read().strip())
                
                optimal_value = optimal_values[param]
                status = "✅ OK" if current_value >= optimal_value else "⚠️  LOW"
                
                print(f"{param:<20} {current_value:<12} {optimal_value:<12} {status:<8}")
                
                if current_value < optimal_value:
                    buffer_issues.append(f"{param}: {current_value} < {optimal_value}")
            
            except Exception as e:
                print(f"{param:<20} {'Error':<12} {'N/A':<12} {'❌ ERR':<8}")
        
        if buffer_issues:
            self.results['buffer_issues'] = buffer_issues
    
    def check_tcp_settings(self):
        """Check TCP optimization settings"""
        print("\n4. TCP CONFIGURATION:")
        print("-" * 25)
        
        tcp_files = {
            'tcp_window_scaling': '/proc/sys/net/ipv4/tcp_window_scaling',
            'tcp_timestamps': '/proc/sys/net/ipv4/tcp_timestamps', 
            'tcp_sack': '/proc/sys/net/ipv4/tcp_sack',
            'tcp_congestion_control': '/proc/sys/net/ipv4/tcp_congestion_control'
        }
        
        optimal_values = {
            'tcp_window_scaling': '1',
            'tcp_timestamps': '1',
            'tcp_sack': '1',
            'tcp_congestion_control': 'bbr'
        }
        
        tcp_issues = []
        
        for param, file_path in tcp_files.items():
            try:
                with open(file_path, 'r') as f:
                    current_value = f.read().strip()
                
                optimal_value = optimal_values[param]
                status = "✅ OK" if current_value == optimal_value else "⚠️  SUBOPTIMAL"
                
                print(f"{param:<25} {current_value:<15} {status}")
                
                if current_value != optimal_value:
                    tcp_issues.append(f"{param}: {current_value} (recommend: {optimal_value})")
            
            except Exception as e:
                print(f"{param:<25} Error reading")
        
        if tcp_issues:
            self.results['tcp_issues'] = tcp_issues
    
    def test_connectivity(self, target_host):
        """Test basic connectivity to target host"""
        print(f"\n5. CONNECTIVITY TEST TO {target_host}:")
        print("-" * 40)
        
        try:
            # Ping test
            ping_result = subprocess.run(
                ['ping', '-c', '4', target_host],
                capture_output=True, text=True, timeout=10
            )
            
            if ping_result.returncode == 0:
                print("✅ Ping successful")
                
                # Extract packet loss and average latency
                output = ping_result.stdout
                for line in output.split('\n'):
                    if 'packet loss' in line:
                        packet_loss = line.split()[5]
                        print(f"Packet loss: {packet_loss}")
                        
                        if '0%' not in packet_loss:
                            self.results['connectivity_issues'] = f"Packet loss detected: {packet_loss}"
                    
                    if 'avg' in line and 'min' in line:
                        latency_parts = line.split('=')[1].strip().split('/')
                        avg_latency = latency_parts[1]
                        print(f"Average latency: {avg_latency} ms")
                        
                        if float(avg_latency) > 100:
                            self.results['latency_issues'] = f"High latency: {avg_latency} ms"
            else:
                print("❌ Ping failed")
                self.results['connectivity_issues'] = f"Cannot ping {target_host}"
        
        except subprocess.TimeoutExpired:
            print("❌ Ping timeout")
            self.results['connectivity_issues'] = f"Ping timeout to {target_host}"
        except Exception as e:
            print(f"❌ Ping error: {e}")
    
    def measure_latency(self, target_host):
        """Measure detailed latency metrics"""
        print(f"\n6. LATENCY MEASUREMENT TO {target_host}:")
        print("-" * 40)
        
        try:
            # Multiple ping tests for statistical analysis
            ping_result = subprocess.run(
                ['ping', '-c', '20', target_host],
                capture_output=True, text=True, timeout=30
            )
            
            if ping_result.returncode == 0:
                output = ping_result.stdout
                
                latencies = []
                for line in output.split('\n'):
                    if 'time=' in line:
                        time_part = line.split('time=')[1].split()[0]
                        latencies.append(float(time_part))
                
                if latencies:
                    min_latency = min(latencies)
                    max_latency = max(latencies)
                    avg_latency = sum(latencies) / len(latencies)
                    
                    # Calculate jitter (variation)
                    jitter = max_latency - min_latency
                    
                    print(f"Min latency: {min_latency:.2f} ms")
                    print(f"Max latency: {max_latency:.2f} ms")
                    print(f"Avg latency: {avg_latency:.2f} ms")
                    print(f"Jitter: {jitter:.2f} ms")
                    
                    # Performance assessment
                    if avg_latency < 10:
                        print("✅ Excellent latency")
                    elif avg_latency < 50:
                        print("✅ Good latency")
                    elif avg_latency < 100:
                        print("⚠️  Acceptable latency")
                    else:
                        print("❌ Poor latency")
                        self.results['latency_issues'] = f"High average latency: {avg_latency:.2f} ms"
                    
                    if jitter > 20:
                        print("⚠️  High jitter detected")
                        self.results['jitter_issues'] = f"High jitter: {jitter:.2f} ms"
        
        except Exception as e:
            print(f"Error measuring latency: {e}")
    
    def test_throughput(self, target_host):
        """Test network throughput"""
        print(f"\n7. THROUGHPUT TEST TO {target_host}:")
        print("-" * 35)
        
        print("Note: For accurate throughput testing:")
        print("1. Run iperf3 server on target: iperf3 -s")
        print("2. Run iperf3 client from here: iperf3 -c <target> -t 30")
        print("3. Use multiple parallel streams: iperf3 -c <target> -P 4")
        print("")
        print("Example commands:")
        print(f"  iperf3 -c {target_host} -t 30")
        print(f"  iperf3 -c {target_host} -P 4 -t 30")
        print(f"  iperf3 -c {target_host} -u -b 1G")  # UDP test
    
    def generate_performance_report(self):
        """Generate comprehensive performance report"""
        print("\n" + "=" * 60)
        print("PERFORMANCE DIAGNOSTIC REPORT")
        print("=" * 60)
        print(f"Report generated: {datetime.now()}")
        
        if not self.results:
            print("\n✅ No performance issues detected")
            return
        
        print(f"\nIssues found: {len(self.results)}")
        
        for category, issue in self.results.items():
            print(f"\n{category.upper().replace('_', ' ')}:")
            if isinstance(issue, list):
                for item in issue:
                    print(f"  - {item}")
            else:
                print(f"  - {issue}")
        
        print("\nRECOMMENDATIONS:")
        
        if 'ena_issue' in self.results:
            print("  1. Enable Enhanced Networking (ENA) for better performance")
        
        if 'buffer_issues' in self.results:
            print("  2. Optimize network buffer sizes:")
            print("     sudo sysctl -w net.core.rmem_max=67108864")
            print("     sudo sysctl -w net.core.wmem_max=67108864")
        
        if 'tcp_issues' in self.results:
            print("  3. Optimize TCP settings:")
            print("     sudo sysctl -w net.ipv4.tcp_congestion_control=bbr")
        
        if 'mtu_optimization' in self.results:
            print("  4. Consider enabling jumbo frames (MTU 9000)")
        
        if 'latency_issues' in self.results:
            print("  5. Investigate network path and consider placement groups")

def main():
    import sys
    
    target_host = sys.argv[1] if len(sys.argv) > 1 else None
    
    diagnostics = NetworkPerformanceDiagnostics()
    diagnostics.run_comprehensive_diagnostics(target_host)

if __name__ == "__main__":
    main()

PERF_DIAG

chmod +x network-performance-diagnostics.py

echo "✅ Network performance diagnostics tool created"

echo ""
echo "2. CREATING LATENCY TROUBLESHOOTING TOOL"

cat << 'LATENCY_TOOL' > latency-troubleshooting-tool.sh
#!/bin/bash

echo "=== Latency Troubleshooting Tool ==="
echo ""

TARGET_HOST="$1"
if [ -z "$TARGET_HOST" ]; then
    echo "Usage: $0 <target-host>"
    echo "Example: $0 8.8.8.8"
    exit 1
fi

echo "Analyzing latency to: $TARGET_HOST"
echo "Timestamp: $(date)"
echo ""

echo "1. BASIC LATENCY TEST"
echo "====================="
ping -c 10 $TARGET_HOST

echo ""
echo "2. DETAILED LATENCY ANALYSIS"
echo "============================="

# Collect detailed ping statistics
echo "Running extended ping test (100 packets)..."
PING_RESULT=$(ping -c 100 $TARGET_HOST 2>/dev/null)

if [ $? -eq 0 ]; then
    echo "$PING_RESULT" | tail -1
    
    # Extract and analyze statistics
    PACKET_LOSS=$(echo "$PING_RESULT" | grep "packet loss" | awk '{print $6}' | sed 's/%//')
    
    if [ "${PACKET_LOSS%.*}" -gt 0 ]; then
        echo "⚠️  WARNING: Packet loss detected (${PACKET_LOSS}%)"
    else
        echo "✅ No packet loss detected"
    fi
    
    # Extract latency statistics
    LATENCY_LINE=$(echo "$PING_RESULT" | grep "min/avg/max")
    if [ -n "$LATENCY_LINE" ]; then
        MIN_LATENCY=$(echo $LATENCY_LINE | cut -d'/' -f4)
        AVG_LATENCY=$(echo $LATENCY_LINE | cut -d'/' -f5)
        MAX_LATENCY=$(echo $LATENCY_LINE | cut -d'/' -f6)
        
        echo "Latency Statistics:"
        echo "  Minimum: ${MIN_LATENCY} ms"
        echo "  Average: ${AVG_LATENCY} ms"
        echo "  Maximum: ${MAX_LATENCY} ms"
        
        # Calculate jitter
        JITTER=$(echo "$MAX_LATENCY - $MIN_LATENCY" | bc 2>/dev/null || echo "N/A")
        echo "  Jitter: ${JITTER} ms"
        
        # Performance assessment
        AVG_INT=${AVG_LATENCY%.*}
        if [ "$AVG_INT" -lt 10 ]; then
            echo "✅ Excellent latency performance"
        elif [ "$AVG_INT" -lt 50 ]; then
            echo "✅ Good latency performance"
        elif [ "$AVG_INT" -lt 100 ]; then
            echo "⚠️  Acceptable latency performance"
        else
            echo "❌ Poor latency performance - investigate network path"
        fi
    fi
else
    echo "❌ Unable to reach target host"
fi

echo ""
echo "3. NETWORK PATH ANALYSIS"
echo "========================"

if command -v traceroute &> /dev/null; then
    echo "Tracing route to $TARGET_HOST:"
    traceroute $TARGET_HOST 2>/dev/null | head -15
elif command -v tracepath &> /dev/null; then
    echo "Tracing path to $TARGET_HOST:"
    tracepath $TARGET_HOST 2>/dev/null | head -15
else
    echo "Traceroute/tracepath not available"
fi

echo ""
echo "4. MTU PATH DISCOVERY"
echo "===================="

echo "Testing MTU sizes..."
for mtu in 1500 1450 1400 1350; do
    payload_size=$((mtu - 28))  # Subtract IP and ICMP headers
    if ping -c 1 -M do -s $payload_size $TARGET_HOST >/dev/null 2>&1; then
        echo "✅ MTU $mtu: Supported"
        break
    else
        echo "❌ MTU $mtu: Failed"
    fi
done

echo ""
echo "5. LATENCY TROUBLESHOOTING RECOMMENDATIONS"
echo "=========================================="

echo "Based on the analysis above:"
echo ""

if [ -n "$AVG_LATENCY" ]; then
    AVG_INT=${AVG_LATENCY%.*}
    
    if [ "$AVG_INT" -gt 100 ]; then
        echo "HIGH LATENCY DETECTED:"
        echo "  • Check if target is geographically distant"
        echo "  • Verify network path is optimal"
        echo "  • Consider using placement groups for AWS resources"
        echo "  • Check for network congestion"
    fi
    
    if [ -n "$JITTER" ] && [ "$JITTER" != "N/A" ]; then
        JITTER_INT=${JITTER%.*}
        if [ "$JITTER_INT" -gt 20 ]; then
            echo "HIGH JITTER DETECTED:"
            echo "  • Network path instability"
            echo "  • Possible congestion or routing issues"
            echo "  • Check QoS configuration"
        fi
    fi
fi

if [ -n "$PACKET_LOSS" ] && [ "${PACKET_LOSS%.*}" -gt 0 ]; then
    echo "PACKET LOSS DETECTED:"
    echo "  • Check network interface errors"
    echo "  • Verify security group and NACL rules"
    echo "  • Look for network congestion"
    echo "  • Check intermediate router capacity"
fi

echo ""
echo "NEXT STEPS:"
echo "  1. Run: python3 network-performance-diagnostics.py $TARGET_HOST"
echo "  2. Check VPC Flow Logs for rejected packets"
echo "  3. Verify security group and NACL configurations"
echo "  4. Consider using VPC Reachability Analyzer"

LATENCY_TOOL

chmod +x latency-troubleshooting-tool.sh

echo "✅ Latency troubleshooting tool created"

echo ""
echo "PERFORMANCE TROUBLESHOOTING TOOLS COMPLETED"
echo ""
echo "Available Tools:"
echo "1. network-performance-diagnostics.py - Comprehensive performance analysis"
echo "2. latency-troubleshooting-tool.sh - Detailed latency diagnosis"
echo ""
echo "Usage Examples:"
echo "python3 network-performance-diagnostics.py 8.8.8.8"
echo "./latency-troubleshooting-tool.sh 8.8.8.8"

EOF

chmod +x implement-performance-troubleshooting.sh
./implement-performance-troubleshooting.sh
```

## VPC Troubleshooting and Diagnostics Summary

### Systematic Troubleshooting Methodology

#### Structured Approach
1. **Problem Definition**: Clearly define symptoms and scope
2. **Information Gathering**: Collect baseline metrics and configurations
3. **Layer Analysis**: Work through OSI layers systematically
4. **Component Isolation**: Test individual components
5. **Root Cause Analysis**: Identify the underlying issue
6. **Solution Implementation**: Apply fix and verify resolution

### AWS Diagnostic Tools

#### VPC Reachability Analyzer
- **Purpose**: Test connectivity between resources
- **Analysis**: Layer 3 and 4 path verification
- **Output**: Detailed blocking point identification
- **Integration**: Automated troubleshooting workflows

#### VPC Flow Logs Analysis
- **Traffic Patterns**: Identify normal vs anomalous behavior
- **Security Events**: Detect rejected connections and attacks
- **Performance Issues**: Analyze throughput and latency patterns
- **Compliance**: Audit network access and communications

### Common VPC Issues and Solutions

#### Connectivity Problems
```
Issue Category          Diagnostic Steps              Common Solutions
Security Groups         Check inbound/outbound rules  Add required ports/sources
Network ACLs           Verify stateless rules        Add return traffic rules
Routing                Check route tables            Add missing routes
DNS Resolution         Test name resolution          Configure VPC DNS settings
```

#### Performance Problems
```
Issue Category          Diagnostic Tools              Optimization Steps
High Latency           Ping, traceroute, MTR         Placement groups, routing
Low Throughput         iperf3, bandwidth tests       Enhanced networking, MTU
Packet Loss            Extended ping tests           Interface tuning, buffers
Jitter/Variation       Statistical analysis          QoS, traffic shaping
```

### Automated Diagnostic Framework

#### Decision Tree Logic
- **Problem Categorization**: Connectivity, performance, security, DNS
- **Systematic Testing**: Layer-by-layer verification
- **Issue Identification**: Automated root cause analysis
- **Solution Recommendations**: Actionable remediation steps

#### Performance Analysis
- **Baseline Measurement**: Establish performance benchmarks
- **Configuration Analysis**: Network settings optimization
- **Real-time Monitoring**: Continuous performance tracking
- **Trend Analysis**: Historical performance patterns

## Key Takeaways
- Systematic troubleshooting prevents missing root causes
- AWS provides powerful diagnostic tools for network analysis
- VPC Flow Logs are essential for traffic pattern analysis
- Performance issues often stem from configuration suboptimization
- Automated diagnostics improve troubleshooting efficiency
- Documentation of troubleshooting steps aids future resolution
- Proactive monitoring prevents many issues from becoming critical

## Additional Resources
- [VPC Reachability Analyzer](https://docs.aws.amazon.com/vpc/latest/reachability/)
- [VPC Flow Logs User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)
- [Network Troubleshooting](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/troubleshooting-network.html)

## Exam Tips
- VPC Reachability Analyzer tests Layer 3/4 connectivity
- VPC Flow Logs capture connection metadata, not packet contents
- Security groups are stateful, NACLs are stateless
- Route table associations are at the subnet level
- DNS resolution requires VPC settings enablement
- Performance issues often relate to instance type limitations
- Systematic troubleshooting follows OSI layer progression
- Documentation and baselines are crucial for effective troubleshooting