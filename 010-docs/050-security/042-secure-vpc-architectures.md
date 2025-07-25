# Topic 42: Designing Secure VPC Architectures

## Introduction

Designing secure VPC architectures requires understanding security principles, AWS networking components, and best practices for implementing defense-in-depth strategies. This comprehensive guide covers secure design patterns and implementation strategies.

## Core Security Principles

### Security by Design
- **Least Privilege**: Grant minimum necessary access
- **Defense in Depth**: Multiple layers of security controls
- **Zero Trust**: Never trust, always verify
- **Segregation**: Isolate workloads and networks
- **Monitoring**: Continuous visibility and logging

### AWS Well-Architected Security Pillar
1. **Identity and Access Management**
2. **Detective Controls**
3. **Infrastructure Protection**
4. **Data Protection in Transit and at Rest**
5. **Incident Response**

## Secure VPC Design Patterns

### Three-Tier Architecture

#### Architecture Overview
```
Internet → IGW → Public Subnet (Web) → Private Subnet (App) → Private Subnet (DB)
```

#### Security Layers
1. **Internet Gateway**: Controlled entry point
2. **Public Subnet**: Web tier with limited exposure
3. **Private Subnets**: Application and database tiers
4. **NAT Gateway**: Outbound internet access for private resources

#### Implementation Details
**Public Subnet (Web Tier)**:
- ALB/NLB for load balancing
- Auto Scaling Groups with minimum instances
- Security Groups allowing HTTP/HTTPS only
- WAF for application layer protection

**Private Subnet (Application Tier)**:
- Application servers with no direct internet access
- Security Groups allowing traffic only from web tier
- Connection to databases in separate subnet
- Outbound internet via NAT Gateway

**Private Subnet (Database Tier)**:
- RDS instances in DB subnet groups
- Security Groups allowing traffic only from app tier
- Encrypted storage and transit
- No internet connectivity

### DMZ (Perimeter Network) Design

#### Architecture Components
```
Internet → IGW → DMZ Subnet → Internal Subnets
```

#### DMZ Subnet Configuration
**Purpose**: Host internet-facing services securely
**Components**:
- Bastion hosts for administrative access
- Reverse proxies and load balancers
- Security appliances (firewalls, IDS/IPS)
- Jump boxes for secure connectivity

**Security Controls**:
```bash
# NACL for DMZ Subnet
Rule #100: ALLOW HTTP 80 from 0.0.0.0/0
Rule #110: ALLOW HTTPS 443 from 0.0.0.0/0
Rule #120: ALLOW SSH 22 from Admin_CIDR
Rule #130: ALLOW Established connections
Rule #32767: DENY ALL
```

### Hub and Spoke Architecture

#### Design Benefits
- **Centralized Security**: Security services in hub VPC
- **Shared Services**: Common resources in hub
- **Network Segmentation**: Isolated spoke VPCs
- **Traffic Inspection**: All traffic flows through hub

#### Implementation with Transit Gateway
```bash
# Hub VPC: Centralized security and shared services
Hub_VPC (10.0.0.0/16)
├── Security_Subnet (Firewalls, IDS/IPS)
├── Shared_Services_Subnet (DNS, AD, Monitoring)
└── Transit_Gateway_Subnet

# Spoke VPCs: Workload-specific
Prod_VPC (10.1.0.0/16) → TGW → Hub_VPC
Dev_VPC (10.2.0.0/16) → TGW → Hub_VPC
Test_VPC (10.3.0.0/16) → TGW → Hub_VPC
```

## Network Security Components

### Security Groups Strategy

#### Principle of Least Privilege
```bash
# Web Server Security Group
Name: WebServer-SG
Inbound:
  HTTP (80): ALB-SG
  HTTPS (443): ALB-SG
  SSH (22): Bastion-SG
Outbound:
  HTTPS (443): 0.0.0.0/0 (for updates)
  App (8080): AppServer-SG
```

#### Security Group Chaining
```bash
# Tiered security group references
Internet → ALB-SG → WebServer-SG → AppServer-SG → Database-SG
```

### Network ACLs Configuration

#### Subnet-Level Protection
```bash
# Web Subnet NACL
Rule #100: ALLOW HTTP 80 from 0.0.0.0/0
Rule #110: ALLOW HTTPS 443 from 0.0.0.0/0
Rule #120: ALLOW SSH 22 from 10.0.0.0/8
Rule #130: ALLOW Ephemeral 1024-65535 to 0.0.0.0/0
Rule #32767: DENY ALL

# Database Subnet NACL
Rule #100: ALLOW MySQL 3306 from App_Subnet_CIDR
Rule #110: ALLOW Ephemeral 1024-65535 to App_Subnet_CIDR
Rule #32767: DENY ALL
```

### VPC Endpoints Security

#### Interface Endpoints (PrivateLink)
```bash
# S3 VPC Endpoint Policy
{
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::company-bucket/*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalVpc": "vpc-12345678"
        }
      }
    }
  ]
}
```

#### Gateway Endpoints
- **S3 Gateway Endpoint**: Private access to S3
- **DynamoDB Gateway Endpoint**: Private access to DynamoDB
- **Route Table Integration**: Automatic routing

## Advanced Security Patterns

### Zero Trust Network Architecture

#### Implementation Components
1. **Identity Verification**: Every connection authenticated
2. **Device Verification**: Endpoint security validation
3. **Application Verification**: Application-level authentication
4. **Micro-Segmentation**: Granular network controls

#### AWS Implementation
```bash
# Micro-segmentation with Security Groups
Each workload gets dedicated security group
Explicit rules for inter-service communication
Application-level authentication required
Continuous monitoring and validation
```

### Isolated Workload Pattern

#### Complete Network Isolation
```bash
# Isolated VPC Design
Isolated_VPC:
  - No Internet Gateway
  - No NAT Gateway
  - VPC Endpoints for AWS services
  - PrivateLink for external services
  - Direct Connect for on-premises
```

#### Use Cases
- **Highly Regulated Workloads**: Financial, healthcare
- **Sensitive Data Processing**: PII, PHI handling
- **Compliance Requirements**: Air-gapped environments
- **Research and Development**: Intellectual property protection

## Data Protection Strategies

### Encryption in Transit

#### VPC-Level Encryption
```bash
# TLS/SSL Termination
ALB → TLS 1.2/1.3 → Backend encryption
NLB → TCP passthrough → Application TLS
CloudFront → Edge TLS → Origin encryption
```

#### VPN and Direct Connect
```bash
# Site-to-Site VPN
IPSec encryption for all traffic
BGP for dynamic routing
Redundant tunnels for high availability

# Direct Connect + MACsec
Layer 2 encryption for Direct Connect
Hardware-based encryption
End-to-end security
```

### Encryption at Rest

#### EBS Encryption
```bash
# Default encryption enabled
All new volumes encrypted by default
Customer-managed or AWS-managed keys
Snapshots automatically encrypted
```

#### RDS Encryption
```bash
# Database encryption
Encrypted storage with KMS
Encrypted automated backups
Encrypted read replicas
TDE for SQL Server
```

## Monitoring and Compliance

### VPC Flow Logs

#### Security Monitoring
```bash
# Flow Log Analysis
Source/Destination IP patterns
Protocol and port analysis
Rejected connection attempts
Anomaly detection
```

#### Log Destinations
- **CloudWatch Logs**: Real-time monitoring
- **S3**: Long-term storage and analysis
- **Kinesis Data Firehose**: Stream processing

### AWS Config Rules

#### Network Compliance
```bash
# Security Group Rules Monitoring
vpc-sg-open-only-to-authorized-ports
vpc-network-acl-unused-check
ec2-security-group-attached-to-eni
root-access-key-check
```

### GuardDuty Integration

#### Threat Detection
- **Malicious IP Communication**: Known threat indicators
- **Cryptocurrency Mining**: Unauthorized mining activity
- **DNS Data Exfiltration**: Suspicious DNS patterns
- **Port Scanning**: Network reconnaissance attempts

## Incident Response Architecture

### Isolation Capabilities

#### Automated Response
```bash
# Lambda function for incident response
def isolate_instance(instance_id):
    # Create isolation security group
    isolation_sg = create_isolation_sg()
    
    # Replace instance security groups
    replace_security_groups(instance_id, isolation_sg)
    
    # Update NACLs for subnet isolation
    isolate_subnet(get_instance_subnet(instance_id))
```

#### Manual Isolation
- **Security Group Replacement**: Immediate network isolation
- **NACL Updates**: Subnet-level blocking
- **Route Table Modification**: Traffic redirection
- **Instance Termination**: Last resort isolation

### Forensics Preparation

#### Evidence Preservation
```bash
# Snapshot creation for forensics
create_ebs_snapshot(instance_id)
enable_detailed_monitoring(instance_id)
export_vpc_flow_logs(vpc_id, time_range)
preserve_cloudtrail_logs(account_id, time_range)
```

## Cost Optimization for Security

### Efficient Security Services

#### NAT Gateway vs NAT Instance
```bash
# NAT Gateway (Managed)
Pros: High availability, managed scaling
Cons: Higher cost for low traffic
Use case: Production environments

# NAT Instance (Self-managed)
Pros: Lower cost, customizable
Cons: Management overhead, single AZ
Use case: Development environments
```

#### VPC Endpoint Cost Analysis
```bash
# Interface Endpoint Pricing
$0.01 per hour per endpoint
$0.01 per GB processed
Cost-effective for high API usage

# Gateway Endpoint Pricing
No additional charges
Route table routing only
Always cost-effective
```

## Implementation Checklist

### Pre-Deployment Security
- [ ] Network design review and approval
- [ ] Security group rules documented
- [ ] NACL configurations validated
- [ ] Encryption strategies defined
- [ ] Monitoring and logging configured

### Post-Deployment Validation
- [ ] Security testing performed
- [ ] Penetration testing completed
- [ ] Compliance requirements verified
- [ ] Incident response procedures tested
- [ ] Regular security assessments scheduled

## Best Practices Summary

1. **Start with Least Privilege**: Deny by default, allow explicitly
2. **Implement Defense in Depth**: Multiple security layers
3. **Use Managed Services**: Leverage AWS security services
4. **Monitor Continuously**: Real-time threat detection
5. **Automate Responses**: Rapid incident containment
6. **Regular Reviews**: Periodic security assessments
7. **Document Everything**: Maintain security documentation
8. **Train Teams**: Security awareness and procedures

## Conclusion

Secure VPC architectures require careful planning, proper implementation of security controls, and continuous monitoring. By following these patterns and best practices, you can build robust, secure network infrastructures that protect your workloads while maintaining operational efficiency and compliance requirements.