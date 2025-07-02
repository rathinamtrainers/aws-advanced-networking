# Foundations Module (Topics 1-10)

## Completed Topics

### ✅ Topic 1: Introduction to AWS Networking - A Complete Guide
**File**: `001-intro-aws-networking.md`
- AWS networking service overview
- Shared responsibility model
- Console navigation lab
- Key takeaways and exam tips

### ✅ Topic 2: AWS Global Infrastructure - Regions, AZs, and Edge Locations  
**File**: `002-global-infrastructure.md`
- Regions, AZs, and edge locations explained
- Infrastructure comparison lab
- Cross-region latency testing
- High availability design patterns

### ✅ Topic 3: Understanding Virtual Private Cloud (VPC) Basics
**File**: `003-vpc-basics.md`
- Custom VPC creation and configuration
- CIDR planning and subnet design
- DNS resolution and reserved IPs
- Complete VPC setup lab

### ✅ Topic 4: Subnets Explained - Public, Private, and Isolated
**File**: `004-subnets-explained.md`
- Three subnet types and use cases
- Multi-tier architecture implementation
- Route table configuration
- Security group integration

### ✅ Topic 5: Internet Gateway and NAT Gateway - Key Differences
**File**: `005-gateways-explained.md`
- IGW vs NAT Gateway comparison
- High availability NAT architecture
- Performance testing and cost analysis
- Troubleshooting gateway issues

## Remaining Topics (To Be Created)

### Topic 6: Deep Dive into VPC Route Tables
**Status**: Pending
**Key Content**:
- Route table hierarchy and precedence
- Custom vs main route tables
- Route propagation and longest prefix matching
- Advanced routing scenarios lab

### Topic 7: Elastic IP and Elastic Network Interface (ENI) Explained
**Status**: Pending  
**Key Content**:
- EIP allocation and association
- ENI creation and attachment
- Multi-homed instances configuration
- Failover scenarios with ENI

### Topic 8: VPC Peering - Concepts, Use Cases, and Demo
**Status**: Pending
**Key Content**:
- VPC peering limitations and requirements
- Cross-region and cross-account peering
- Route table configuration for peering
- Hands-on peering implementation

### Topic 9: Transit Gateway vs VPC Peering - Which to Choose?
**Status**: Pending
**Key Content**:
- Architecture comparison matrix
- Cost analysis and scaling considerations
- Migration strategies
- Decision framework lab

### Topic 10: Mastering VPC Flow Logs - Network Monitoring Essentials
**Status**: Pending
**Key Content**:
- Flow log configuration and formats
- CloudWatch integration and analysis
- Security monitoring use cases
- Log analysis and troubleshooting lab

## Module Learning Path

1. **Start Here**: Topics 1-3 provide essential foundation
2. **Build Skills**: Topics 4-5 add practical networking concepts  
3. **Advanced Features**: Topics 6-10 cover specialized scenarios
4. **Next Module**: Proceed to Hybrid Networking (Topics 11-25)

## Lab Environment Requirements

- AWS Account with VPC, EC2, and CloudWatch permissions
- AWS CLI configured with appropriate credentials
- Basic command line familiarity
- Understanding of IP networking concepts

## Common Prerequisites for All Labs

```bash
# Set default region
export AWS_DEFAULT_REGION=us-east-1

# Verify AWS CLI access
aws sts get-caller-identity

# Create lab environment variables
export LAB_PREFIX="aws-networking-lab"
export LAB_CIDR="10.0.0.0/16"
```