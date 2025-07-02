# Topic 2: AWS Global Infrastructure - Regions, AZs, and Edge Locations

## Prerequisites
- Completed Topic 1: Introduction to AWS Networking
- Basic understanding of geographic networking concepts
- Access to AWS CLI and Console

## Learning Objectives
By the end of this topic, you will be able to:
- Explain AWS global infrastructure components and their relationships
- Choose appropriate AWS regions for workloads based on requirements
- Design multi-AZ architectures for high availability
- Understand edge locations and their role in content delivery

## Theory

### AWS Global Infrastructure Overview
AWS operates a globally distributed infrastructure designed for security, reliability, performance, and scalability.

### Components of AWS Global Infrastructure

#### 1. AWS Regions
**Definition**: Geographic areas containing multiple isolated Availability Zones

**Key Characteristics**:
- Completely independent and isolated
- Minimum of 3 AZs per region (typically)
- Connected by high-bandwidth, low-latency networking
- Data sovereignty compliance

**Current Count**: 31+ regions worldwide (as of 2024)

**Region Selection Criteria**:
- **Latency**: Distance to users
- **Compliance**: Data residency requirements  
- **Service Availability**: Not all services available in all regions
- **Cost**: Pricing varies by region

#### 2. Availability Zones (AZs)
**Definition**: Isolated locations within regions with independent power, cooling, and networking

**Key Characteristics**:
- Physically separated (typically 10+ miles apart)
- Connected via high-speed private links
- Single points of failure isolation
- Sub-millisecond latency between AZs in same region

**Naming Convention**: 
- Region code + letter (e.g., us-east-1a, us-east-1b)
- AZ IDs for consistent mapping (e.g., use1-az1)

#### 3. Edge Locations
**Definition**: AWS infrastructure for content caching and edge computing

**Types**:
- **CloudFront Edge Locations**: Content caching (400+ locations)
- **Regional Edge Caches**: Larger caches between origin and edge
- **AWS Local Zones**: Ultra-low latency for specific metros
- **AWS Wavelength**: 5G edge computing

### Networking Between Infrastructure Components

#### Inter-Region Connectivity
- All AWS regions connected via AWS backbone network
- Traffic can be routed between regions
- Cross-region replication available for many services
- Higher latency than intra-region communication

#### Intra-Region Connectivity  
- All AZs in region connected via redundant network links
- High bandwidth, low latency connections
- Synchronous replication possible between AZs
- Designed for real-time applications

## Lab Exercise: Exploring AWS Global Infrastructure

### Lab Duration: 60 minutes

### Step 1: Identify Available Regions and AZs
**Objective**: Understand your account's regional access and AZ distribution

**AWS Console Steps**:
1. Log into AWS Management Console
2. Note current region in top-right corner
3. Click region dropdown to see all available regions
4. Switch to **us-east-1** (N. Virginia)
5. Navigate to **EC2 Console**
6. In left panel, click **"EC2 Dashboard"**
7. Scroll down to see **"Service Health"** section
8. Click **"Availability Zones"** in left panel
9. Record AZ information for current region

**CLI Commands**:
```bash
# List all regions
aws ec2 describe-regions --query 'Regions[*].[RegionName,Endpoint]' --output table

# List AZs for current region
aws ec2 describe-availability-zones --query 'AvailabilityZones[*].[ZoneName,ZoneId,State]' --output table

# Get specific region's AZs
aws ec2 describe-availability-zones --region us-west-2 --query 'AvailabilityZones[*].[ZoneName,ZoneId,State]' --output table
```

**Expected Output**:
```
|  us-east-1a  |  use1-az4  |  available  |
|  us-east-1b  |  use1-az6  |  available  |
|  us-east-1c  |  use1-az1  |  available  |
|  us-east-1d  |  use1-az2  |  available  |
|  us-east-1e  |  use1-az3  |  available  |
|  us-east-1f  |  use1-az5  |  available  |
```

**Verification**: You can identify AZs and their consistent AZ IDs

### Step 2: Compare Region Characteristics
**Objective**: Analyze differences between regions

**AWS Console Steps**:
1. Create comparison table with these columns:
   - Region Name
   - Region Code  
   - Number of AZs
   - Sample Services Available
2. Check the following regions:
   - us-east-1 (N. Virginia)
   - us-west-2 (Oregon)
   - eu-west-1 (Ireland)
   - ap-southeast-1 (Singapore)
3. For each region:
   - Switch to the region
   - Count AZs in EC2 console
   - Check service availability in console navigation

**CLI Commands**:
```bash
# Check services available in specific regions
aws ec2 describe-regions --region-names us-east-1 us-west-2 eu-west-1

# Compare AZ counts
for region in us-east-1 us-west-2 eu-west-1 ap-southeast-1; do
  echo "Region: $region"
  aws ec2 describe-availability-zones --region $region --query 'length(AvailabilityZones)'
  echo "---"
done
```

**Sample Comparison Table**:
| Region | Code | AZs | Key Services |
|--------|------|-----|--------------|
| N. Virginia | us-east-1 | 6 | All services, lowest cost |
| Oregon | us-west-2 | 4 | Most services, renewable energy |
| Ireland | eu-west-1 | 3 | GDPR compliance, EU data residency |
| Singapore | ap-southeast-1 | 3 | APAC access, local compliance |

### Step 3: Explore Edge Locations
**Objective**: Understand edge infrastructure for content delivery

**AWS Console Steps**:
1. Navigate to **CloudFront Console**
2. Click **"Create Distribution"** (don't actually create)
3. Scroll down to see **"Price Class"** options
4. Note the geographic coverage options:
   - All edge locations (best performance)
   - North America and Europe only
   - North America, Europe, Asia, Middle East, and Africa
5. Cancel the creation process
6. Visit AWS Global Infrastructure page externally
7. Review edge location map

**Key Observations**:
- 400+ edge locations worldwide
- Regional edge caches in major regions  
- Price classes allow cost optimization
- Edge locations serve both CloudFront and Route 53

### Step 4: Test Cross-Region Latency
**Objective**: Measure network performance between regions

**AWS Console Steps**:
1. Launch small EC2 instance in **us-east-1**
2. Launch small EC2 instance in **us-west-2**  
3. SSH to us-east-1 instance
4. Test latency to us-west-2 instance:

```bash
# From us-east-1 instance, test to us-west-2
ping [us-west-2-instance-private-ip]

# Test public connectivity
ping google.com

# Use traceroute to see path
traceroute [us-west-2-instance-public-ip]
```

**Expected Results**:
- Cross-region latency: ~60-80ms (us-east-1 to us-west-2)
- Internet latency: varies based on destination
- Traceroute shows AWS backbone network usage

**Cleanup**: Terminate both test instances

## Architecture Diagrams

### Global Infrastructure Overview
```
[Internet Users Worldwide]
         |
[400+ Edge Locations] ← CloudFront, Route 53
         |
[Regional Edge Caches]
         |
[AWS Backbone Network]
         |
┌─────────────────────────────────────┐
│ Region 1 (us-east-1)                │
│ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐    │
│ │AZ-a │ │AZ-b │ │AZ-c │ │AZ-d │    │
│ └─────┘ └─────┘ └─────┘ └─────┘    │
└─────────────────────────────────────┘
         |
┌─────────────────────────────────────┐
│ Region 2 (us-west-2)                │  
│ ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐    │
│ │AZ-a │ │AZ-b │ │AZ-c │ │AZ-d │    │
│ └─────┘ └─────┘ └─────┘ └─────┘    │
└─────────────────────────────────────┘
```

### Multi-AZ Architecture Pattern
```
Region: us-east-1
┌─────────────────────────────────────────┐
│ VPC (10.0.0.0/16)                       │
│                                         │
│ ┌─────────────┐ ┌─────────────┐         │
│ │   AZ-1a     │ │   AZ-1b     │         │
│ │ 10.0.1.0/24 │ │ 10.0.2.0/24 │         │
│ │             │ │             │         │
│ │ [Web Tier]  │ │ [Web Tier]  │         │
│ │ [App Tier]  │ │ [App Tier]  │         │
│ │ [DB Standby]│ │ [DB Primary]│         │
│ └─────────────┘ └─────────────┘         │
└─────────────────────────────────────────┘
```

## Troubleshooting
| Issue | Cause | Solution |
|-------|-------|----------|
| Service not available in region | Regional service limitations | Choose different region or wait for service expansion |
| High cross-region latency | Geographic distance | Use CloudFront or Regional services |
| AZ capacity constraints | High demand in specific AZ | Launch instances in different AZ |
| Inconsistent AZ naming | Account-specific AZ mapping | Use AZ IDs instead of AZ names |

## Key Takeaways
- AWS infrastructure provides global reach with local presence
- Regions are completely isolated for compliance and disaster recovery
- AZs within regions are connected by high-speed, low-latency links
- Edge locations bring AWS services closer to end users
- Always design for multiple AZs within a region for high availability

## Best Practices
- **Region Selection**: Consider latency, compliance, and service availability
- **Multi-AZ Design**: Always deploy across multiple AZs for production workloads
- **Data Locality**: Keep data close to users and comply with regulations
- **Cost Optimization**: Use appropriate regions and edge configurations
- **Disaster Recovery**: Consider cross-region backup and failover strategies

## Additional Resources
- [AWS Global Infrastructure](https://aws.amazon.com/about-aws/global-infrastructure/)
- [AWS Regional Services List](https://aws.amazon.com/about-aws/global-infrastructure/regional-product-services/)
- [CloudFront Edge Locations](https://aws.amazon.com/cloudfront/features/)
- [AWS Well-Architected Framework - Reliability Pillar](https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/)

## Exam Tips
- Know the difference between regions, AZs, and edge locations
- Understand that regions are completely isolated
- Remember minimum 3 AZs per region (with some exceptions)
- AZ IDs provide consistent mapping across accounts
- Edge locations serve CloudFront and Route 53 (not all AWS services)
- Some services are global (IAM, CloudFront, Route 53), others are regional or AZ-specific