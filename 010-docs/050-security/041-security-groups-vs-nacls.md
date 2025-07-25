# Topic 41: Security Groups vs Network ACLs: Key Differences

## Introduction

Understanding the differences between Security Groups and Network Access Control Lists (NACLs) is fundamental to designing secure AWS network architectures. Both provide network-level security but operate at different layers and have distinct characteristics.

## Security Groups Overview

### What are Security Groups?
- **Instance-level firewall** that controls traffic to and from EC2 instances
- **Stateful** - automatically allows return traffic for established connections
- **Default deny** - only explicitly allowed traffic is permitted
- Applied to **Elastic Network Interfaces (ENIs)**

### Security Group Characteristics
- **Stateful Operation**: Return traffic is automatically allowed
- **Allow Rules Only**: Cannot create explicit deny rules
- **Protocol Support**: TCP, UDP, ICMP, ESP, AH, and GRE
- **Source/Destination**: Can reference other security groups, IP addresses, or CIDR blocks
- **Instance Assignment**: Multiple security groups can be assigned to one instance
- **Rule Evaluation**: All rules are evaluated before allowing traffic

### Security Group Rules Structure
```
Type: HTTP
Protocol: TCP
Port Range: 80
Source: 0.0.0.0/0
```

## Network ACLs Overview

### What are Network ACLs?
- **Subnet-level firewall** that controls traffic entering and leaving subnets
- **Stateless** - must explicitly allow both inbound and outbound traffic
- **Default allow** for default NACL, **default deny** for custom NACLs
- Applied to **subnets**

### Network ACL Characteristics
- **Stateless Operation**: Must explicitly allow return traffic
- **Allow and Deny Rules**: Can create both allow and deny rules
- **Rule Numbers**: Rules processed in numerical order (lowest first)
- **Protocol Support**: TCP, UDP, ICMP, ESP, AH, GRE, and custom protocols
- **Subnet Association**: One NACL per subnet, but one NACL can be associated with multiple subnets
- **Rule Evaluation**: Rules processed in order until a match is found

### Network ACL Rules Structure
```
Rule #: 100
Type: HTTP
Protocol: TCP
Port Range: 80
Source: 0.0.0.0/0
Allow/Deny: ALLOW
```

## Key Differences Comparison

| Aspect | Security Groups | Network ACLs |
|--------|----------------|--------------|
| **Level** | Instance (ENI) | Subnet |
| **State** | Stateful | Stateless |
| **Rules** | Allow only | Allow and Deny |
| **Rule Processing** | All rules evaluated | First match wins |
| **Default Behavior** | Deny all | Default NACL allows all |
| **Return Traffic** | Automatic | Must be explicitly allowed |
| **Rule Limits** | 60 inbound, 60 outbound | 20 rules per NACL |
| **Association** | Multiple per instance | One per subnet |

## Use Cases and Best Practices

### When to Use Security Groups
- **Primary security control** for EC2 instances
- **Application-specific rules** (web servers, databases)
- **Dynamic security** referencing other security groups
- **Granular control** at the instance level
- **Simplified management** with stateful connections

### When to Use Network ACLs
- **Additional layer of security** (defense in depth)
- **Subnet-level controls** for broad traffic filtering
- **Explicit deny rules** for specific threats
- **Compliance requirements** for network segmentation
- **Blocking specific IP addresses** or ranges

## Security Layering Strategy

### Defense in Depth Approach
```
Internet → IGW → Route Table → NACL → Security Group → EC2 Instance
```

1. **Network ACL**: First line of defense at subnet level
2. **Security Group**: Instance-specific protection
3. **Instance Firewall**: OS-level firewall (optional)
4. **Application Security**: Application-level controls

### Recommended Configuration Pattern
- **NACLs**: Use for broad subnet-level filtering and explicit denies
- **Security Groups**: Use for granular application-specific rules
- **Combine Both**: Implement layered security approach

## Common Configuration Examples

### Web Server Security
**Network ACL (Subnet Level)**:
```
Rule #100: Allow HTTP (80) from 0.0.0.0/0
Rule #110: Allow HTTPS (443) from 0.0.0.0/0
Rule #120: Allow SSH (22) from Admin_CIDR
Rule #32767: Deny All (implicit)
```

**Security Group (Instance Level)**:
```
HTTP (80): 0.0.0.0/0
HTTPS (443): 0.0.0.0/0
SSH (22): Admin_Security_Group
```

### Database Server Security
**Network ACL**:
```
Rule #100: Allow MySQL (3306) from Web_Subnet_CIDR
Rule #32767: Deny All
```

**Security Group**:
```
MySQL (3306): Web_Server_Security_Group
SSH (22): Admin_Security_Group
```

## Troubleshooting Security Issues

### Common Problems
1. **Stateless NACL Issues**: Forgetting to allow return traffic
2. **Rule Order**: NACL rules processed in wrong order
3. **Overlapping Rules**: Conflicting security group rules
4. **Default Behavior**: Misunderstanding default allow vs deny

### Debugging Steps
1. **Check Security Groups**: Verify allow rules exist
2. **Verify NACLs**: Ensure both inbound and outbound rules
3. **Rule Precedence**: Check NACL rule numbering
4. **Flow Logs**: Use VPC Flow Logs for traffic analysis
5. **Reachability Analyzer**: Use AWS Reachability Analyzer

## Monitoring and Auditing

### VPC Flow Logs Integration
- **Action Field**: ACCEPT or REJECT indicates which layer blocked traffic
- **Security Group Logs**: Track accepted/rejected traffic
- **NACL Analysis**: Identify blocked traffic patterns

### CloudTrail Monitoring
- **Security Group Changes**: Track modifications to rules
- **NACL Modifications**: Monitor subnet-level changes
- **Association Changes**: Track resource associations

## Cost Implications

### Security Groups
- **No Additional Cost**: Included with EC2 instances
- **Rule Limits**: 60 rules per direction (can request increase)

### Network ACLs
- **No Additional Cost**: Included with VPC
- **Rule Limits**: 20 rules per NACL (can request increase)
- **Processing Overhead**: Minimal impact on performance

## Advanced Configurations

### Security Group Chaining
```bash
# Web tier references app tier security group
Web_SG → App_SG (port 8080)
App_SG → DB_SG (port 3306)
```

### NACL for Compliance
```bash
# Explicit deny for prohibited traffic
Rule #50: DENY TCP 23 (Telnet) from 0.0.0.0/0
Rule #60: DENY TCP 21 (FTP) from 0.0.0.0/0
Rule #100: ALLOW HTTPS 443 from 0.0.0.0/0
```

## Lab Exercise

### Scenario Setup
1. Create VPC with public and private subnets
2. Launch web server in public subnet
3. Launch database server in private subnet
4. Configure layered security with NACLs and Security Groups

### Implementation Steps
1. **Custom NACL**: Create restrictive subnet-level rules
2. **Security Groups**: Configure application-specific rules
3. **Testing**: Verify connectivity and security
4. **Monitoring**: Enable Flow Logs and analyze traffic

## Exam Tips

1. **Remember Stateful vs Stateless**: Key differentiator
2. **Rule Evaluation**: All vs first match processing
3. **Default Behavior**: Security Groups deny, default NACL allows
4. **Use Cases**: Instance-level vs subnet-level security
5. **Troubleshooting**: Check both layers when connectivity fails

## Conclusion

Security Groups and Network ACLs provide complementary network security controls in AWS. Understanding their differences and proper implementation ensures robust, layered security for your AWS infrastructure. Use Security Groups for granular, stateful instance protection and Network ACLs for broader, stateless subnet-level filtering.