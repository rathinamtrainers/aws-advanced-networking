# Topic 1: Introduction to AWS Networking - A Complete Guide

## Prerequisites
- Basic understanding of networking concepts (IP, subnets, routing)
- AWS account with administrative access
- AWS CLI installed and configured

## Learning Objectives
By the end of this topic, you will be able to:
- Understand AWS networking fundamentals and core services
- Identify key AWS networking components and their relationships
- Explain the AWS Shared Responsibility Model for networking
- Navigate AWS networking services in the console

## Theory

### AWS Networking Overview
AWS networking provides the foundation for all cloud infrastructure, enabling secure, scalable, and highly available connections between your resources and users worldwide.

### Core AWS Networking Services

#### 1. Amazon Virtual Private Cloud (VPC)
- **Purpose**: Isolated network environment in AWS cloud
- **Key Features**: 
  - Logically isolated network
  - Complete control over IP addressing
  - Subnet creation across Availability Zones
  - Route table management

#### 2. Elastic Load Balancing (ELB)
- **Application Load Balancer (ALB)**: Layer 7 load balancing
- **Network Load Balancer (NLB)**: Layer 4 load balancing  
- **Gateway Load Balancer (GWLB)**: Layer 3 gateway and load balancer

#### 3. Amazon Route 53
- **Purpose**: Scalable DNS web service
- **Features**: Domain registration, DNS routing, health checking

#### 4. AWS Direct Connect
- **Purpose**: Dedicated network connection to AWS
- **Benefits**: Consistent network performance, reduced bandwidth costs

#### 5. AWS Transit Gateway
- **Purpose**: Central hub for connecting VPCs and on-premises networks
- **Benefits**: Simplified network architecture, easier management

### AWS Shared Responsibility Model for Networking

#### AWS Responsibilities
- Physical network infrastructure
- Network controls and security of the host operating system
- Hypervisor patching and security
- Physical separation of customer environments

#### Customer Responsibilities  
- VPC configuration and security groups
- Network ACLs and routing tables
- Operating system patches and updates
- Application-level security

## Lab Exercise: Exploring AWS Networking Services

### Lab Duration: 45 minutes

### Step 1: Navigate AWS Networking Console
**Objective**: Familiarize yourself with AWS networking service locations

**AWS Console Steps**:
1. Log into AWS Management Console
2. Navigate to **VPC Dashboard**
   - Click on "Services" → "VPC" under Networking & Content Delivery
   - Observe the VPC Dashboard overview
3. Explore **EC2 Dashboard** networking sections
   - Navigate to "Services" → "EC2"
   - In left panel, expand "Network & Security"
   - Review Security Groups, Key Pairs, Elastic IPs
4. Visit **Route 53 Console**
   - Navigate to "Services" → "Route 53"
   - Review Hosted Zones and Health Checks sections

**Expected Output**: Familiarity with console navigation

**Verification**: You can locate each major networking service in the AWS Console

### Step 2: Review Default VPC Configuration
**Objective**: Understand default AWS networking setup

**AWS Console Steps**:
1. In VPC Console, click **"Your VPCs"**
2. Identify the default VPC (usually named "default")
3. Record the following information:
   - VPC ID: `vpc-xxxxxxxxx`
   - CIDR Block: Usually `172.31.0.0/16`
   - State: Available
4. Click **"Subnets"** in left panel
5. Count default subnets (typically one per AZ)
6. Click **"Internet Gateways"**
7. Verify default IGW is attached to default VPC

**CLI Commands**:
```bash
# List all VPCs
aws ec2 describe-vpcs --query 'Vpcs[?IsDefault==`true`]'

# List default subnets
aws ec2 describe-subnets --filters "Name=default-for-az,Values=true"

# List Internet Gateways
aws ec2 describe-internet-gateways
```

**Expected Output**:
```json
{
    "Vpcs": [
        {
            "CidrBlock": "172.31.0.0/16",
            "VpcId": "vpc-12345678",
            "State": "available",
            "IsDefault": true
        }
    ]
}
```

**Verification**: Default VPC exists with public subnets in multiple AZs

### Step 3: Explore Networking Service Integration
**Objective**: See how networking services work together

**AWS Console Steps**:
1. Navigate to **EC2 Console** → **Load Balancers**
2. Note available load balancer types
3. Go to **Route 53** → **Health Checks**
4. Observe health check configuration options
5. Return to **VPC Console** → **VPC Peering Connections**
6. Review peering connection states and options

**Key Observations**:
- Load balancers integrate with target groups and health checks
- Route 53 can route traffic based on health checks
- VPC peering enables private connectivity between VPCs

## Architecture Diagram
```
Internet
    |
[Internet Gateway]
    |
[Default VPC - 172.31.0.0/16]
    |
+---[Subnet AZ-a]---[Subnet AZ-b]---[Subnet AZ-c]
    |                  |               |
[EC2 Instances]    [EC2 Instances]  [EC2 Instances]
    |                  |               |
[Security Groups]  [Security Groups] [Security Groups]
```

## Troubleshooting
| Issue | Cause | Solution |
|-------|-------|----------|
| Cannot access VPC Console | Insufficient permissions | Check IAM permissions for EC2/VPC services |
| Default VPC missing | Accidentally deleted | Contact AWS Support or create new VPC |
| Services not visible | Wrong AWS region | Check region selector in top-right corner |

## Key Takeaways
- AWS networking is foundational to all cloud architecture
- Default VPC provides immediate connectivity but may not suit all use cases
- Multiple networking services work together to provide comprehensive solutions
- Shared responsibility model defines clear boundaries between AWS and customer duties

## Best Practices
- Always review default VPC configuration before deploying production workloads
- Use multiple Availability Zones for high availability
- Implement proper security group and NACL configurations
- Monitor network performance and costs regularly

## Additional Resources
- [AWS VPC User Guide](https://docs.aws.amazon.com/vpc/latest/userguide/)
- [AWS Networking Best Practices](https://aws.amazon.com/architecture/well-architected/)
- [AWS Shared Responsibility Model](https://aws.amazon.com/compliance/shared-responsibility-model/)

## Exam Tips
- Understand the difference between default and custom VPCs
- Know which networking services are regional vs global
- Remember that security groups are stateful, NACLs are stateless
- AWS manages the underlying physical network infrastructure