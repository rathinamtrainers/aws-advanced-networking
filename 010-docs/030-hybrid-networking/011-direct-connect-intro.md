# Topic 11: AWS Direct Connect - Introduction and Use Cases

## Prerequisites
- Completed Foundation Module (Topics 1-10)
- Understanding of enterprise networking concepts
- Knowledge of BGP routing protocol basics
- Familiarity with dedicated network connections

## Learning Objectives
By the end of this topic, you will be able to:
- Understand AWS Direct Connect architecture and components
- Identify use cases and benefits of Direct Connect
- Compare Direct Connect with other connectivity options
- Plan Direct Connect implementations for various scenarios

## Theory

### AWS Direct Connect Overview

#### Definition
AWS Direct Connect is a cloud service solution that makes it easy to establish a dedicated network connection from your premises to AWS. You can establish private connectivity between AWS and your datacenter, office, or colocation environment.

#### Key Characteristics
- **Dedicated Connection**: Physical fiber connection to AWS
- **Consistent Performance**: Predictable bandwidth and latency
- **Cost Reduction**: Lower data transfer costs for high-volume workloads
- **Enhanced Security**: Private connection bypassing the public internet
- **Hybrid Architecture**: Seamless integration with on-premises infrastructure

### Direct Connect Components

#### 1. Direct Connect Location
- **Definition**: AWS-approved data centers where Direct Connect is available
- **Global Presence**: 100+ locations worldwide
- **Colocation Facilities**: Third-party data centers with AWS equipment
- **AWS Regions**: Direct connection to specific AWS regions

#### 2. Cross Connect
- **Physical Layer**: Fiber optic cable connecting customer equipment to AWS
- **Installation**: Managed by colocation facility
- **Redundancy**: Multiple cross connects for high availability
- **Bandwidth Options**: 1 Gbps to 100 Gbps connections

#### 3. Customer Gateway
- **Equipment**: Customer-owned router in Direct Connect location
- **BGP Support**: Must support BGP routing protocol
- **Interface**: Physical connection to AWS via cross connect
- **Configuration**: BGP sessions and routing configuration

#### 4. Virtual Interfaces (VIFs)
- **Logical Separation**: Multiple VIFs on single physical connection
- **Types**: Private VIF, Public VIF, Transit VIF
- **VLAN Tagging**: 802.1Q VLAN separation
- **Routing**: Independent routing tables per VIF

### Direct Connect Architecture

#### Physical Layer
```
Customer Premises → WAN/MPLS → Direct Connect Location → AWS Region
     [Router]    →  [Carrier] →    [Customer Router]  → [AWS Router]
                                         ↓
                                    [Cross Connect]
                                         ↓
                                    [AWS Equipment]
```

#### Logical Layer
```
Customer Network
       ↓
[Customer Gateway BGP Router]
       ↓
[Direct Connect Gateway] (Optional)
       ↓
[Virtual Private Gateway] or [Transit Gateway]
       ↓
[VPC Resources]
```

### Virtual Interface Types

#### 1. Private Virtual Interface (Private VIF)
- **Purpose**: Connect to VPC resources via Virtual Private Gateway
- **Access**: Private IP addresses only
- **Routing**: BGP routes to VPC CIDR blocks
- **Use Cases**: 
  - VPC resource access
  - Hybrid cloud architectures
  - Database connectivity
  - Application integration

#### 2. Public Virtual Interface (Public VIF)
- **Purpose**: Connect to AWS public services
- **Access**: Public IP addresses
- **Routing**: BGP routes to AWS public IP ranges
- **Use Cases**:
  - S3 bucket access
  - CloudFront distributions
  - AWS APIs
  - Public service integration

#### 3. Transit Virtual Interface (Transit VIF)
- **Purpose**: Connect to Transit Gateway
- **Access**: Multiple VPCs through single interface
- **Routing**: Dynamic routing via Transit Gateway
- **Use Cases**:
  - Multi-VPC connectivity
  - Complex hybrid architectures
  - Centralized routing management
  - Scalable network design

### Direct Connect Benefits

#### Performance Benefits
- **Consistent Bandwidth**: Dedicated capacity not shared with internet traffic
- **Lower Latency**: Direct path to AWS without internet routing
- **Higher Throughput**: Up to 100 Gbps per connection
- **Predictable Performance**: No internet congestion impact

#### Cost Benefits
- **Reduced Data Transfer Costs**: Lower egress charges for high-volume workloads
- **Predictable Pricing**: Fixed monthly costs for dedicated bandwidth
- **No Internet Gateway Charges**: Direct connection bypasses IGW costs
- **Bulk Transfer Savings**: Significant savings for large data migrations

#### Security Benefits
- **Private Connectivity**: Traffic doesn't traverse public internet
- **Isolation**: Dedicated physical connection
- **Compliance**: Meets regulatory requirements for data transmission
- **Encryption**: Optional encryption for additional security

#### Reliability Benefits
- **SLA**: 99.9% uptime SLA for dedicated connections
- **Redundancy**: Multiple connections for high availability
- **Fail-over**: Automatic routing to backup connections
- **Monitoring**: Enhanced monitoring and alerting capabilities

### Use Cases and Scenarios

#### 1. Large-Scale Data Transfer
```
Scenario: Media company needs to transfer 100TB of video content monthly
Solution: Direct Connect provides cost-effective bulk transfer
Benefits: 70% reduction in data transfer costs compared to internet
```

#### 2. Hybrid Cloud Architecture
```
Scenario: Enterprise wants seamless on-premises to cloud integration
Solution: Private VIF connecting corporate network to AWS VPCs
Benefits: Consistent network experience, security, and performance
```

#### 3. Disaster Recovery
```
Scenario: Financial institution requires reliable DR connectivity
Solution: Multiple Direct Connect connections for redundancy
Benefits: Guaranteed bandwidth for critical data replication
```

#### 4. Real-Time Applications
```
Scenario: Trading platform requires ultra-low latency
Solution: Direct Connect to AWS region with proximity to exchanges
Benefits: Predictable, low-latency connectivity for time-sensitive operations
```

#### 5. Compliance and Governance
```
Scenario: Healthcare organization with strict data privacy requirements
Solution: Private Direct Connect connection with encryption
Benefits: Data never traverses public internet, meets HIPAA requirements
```

## Lab Exercise: Direct Connect Planning and Design

### Lab Duration: 120 minutes

### Step 1: Assess Direct Connect Requirements
**Objective**: Analyze business requirements for Direct Connect implementation

**Requirements Analysis Framework**:
```bash
# Create requirements assessment script
cat << 'EOF' > dx-requirements-assessment.sh
#!/bin/bash

echo "=== AWS Direct Connect Requirements Assessment ==="
echo ""

echo "1. BANDWIDTH REQUIREMENTS"
echo "   Current internet bandwidth: _____ Mbps"
echo "   Peak AWS traffic volume: _____ Mbps"
echo "   Expected growth (12 months): _____ %"
echo "   Recommended DX bandwidth: _____ Mbps"
echo ""

echo "2. LATENCY REQUIREMENTS"
echo "   Current internet latency to AWS: _____ ms"
echo "   Application latency requirements: _____ ms"
echo "   Real-time application needs: Yes/No"
echo "   Latency-sensitive workloads: [List]"
echo ""

echo "3. COST ANALYSIS"
echo "   Current monthly data transfer costs: $ _____"
echo "   Monthly data volume to/from AWS: _____ GB"
echo "   Projected Direct Connect savings: $ _____"
echo "   Break-even timeline: _____ months"
echo ""

echo "4. SECURITY AND COMPLIANCE"
echo "   Data classification level: Public/Internal/Confidential"
echo "   Regulatory requirements: [List]"
echo "   Encryption requirements: Yes/No"
echo "   Network isolation needs: Yes/No"
echo ""

echo "5. AVAILABILITY REQUIREMENTS"
echo "   Business criticality: Low/Medium/High/Critical"
echo "   Acceptable downtime: _____ hours/month"
echo "   Redundancy requirements: Single/Dual/Multi-path"
echo "   Failover time requirements: _____ seconds"
echo ""

echo "6. TECHNICAL REQUIREMENTS"
echo "   On-premises router capability: Yes/No"
echo "   BGP routing experience: Yes/No"
echo "   VLAN support: Yes/No"
echo "   Monitoring and management tools: [List]"

EOF

chmod +x dx-requirements-assessment.sh
./dx-requirements-assessment.sh > dx-requirements.txt

echo "Requirements assessment saved to dx-requirements.txt"
```

### Step 2: Direct Connect Location Selection
**Objective**: Choose optimal Direct Connect location

**Location Analysis**:
```bash
# Get Direct Connect locations
aws directconnect describe-locations \
    --query 'locations[*].[locationCode,locationName,region]' \
    --output table > dx-locations.txt

echo "Available Direct Connect locations saved to dx-locations.txt"

# Analyze location options
cat << 'EOF' > analyze-dx-locations.sh
#!/bin/bash

echo "=== Direct Connect Location Analysis ==="
echo ""

# Example analysis for US East region
echo "REGION: US East (N. Virginia)"
echo "Primary locations:"
echo "  - Ashburn, VA (Multiple facilities)"
echo "  - Washington DC Metro"
echo "  - New York Metro"
echo ""

echo "SELECTION CRITERIA:"
echo "1. Geographic proximity to on-premises location"
echo "2. Carrier availability and pricing"
echo "3. Facility redundancy and reliability"
echo "4. AWS region connectivity options"
echo "5. Future expansion capabilities"
echo ""

echo "RECOMMENDED APPROACH:"
echo "1. Primary: Closest location to headquarters"
echo "2. Secondary: Different location for redundancy"
echo "3. Tertiary: Consider cross-region for DR"

EOF

chmod +x analyze-dx-locations.sh
./analyze-dx-locations.sh
```

### Step 3: Bandwidth Planning and Sizing
**Objective**: Determine appropriate Direct Connect bandwidth

**Bandwidth Calculation**:
```bash
# Create bandwidth planning tool
cat << 'EOF' > dx-bandwidth-planning.sh
#!/bin/bash

echo "=== Direct Connect Bandwidth Planning ==="
echo ""

# Input parameters (example values)
CURRENT_PEAK_MBPS=500
GROWTH_RATE=0.25  # 25% annual growth
SAFETY_MARGIN=0.30  # 30% safety margin
MONTHS_TO_PLAN=24

# Calculate projected bandwidth
PROJECTED_MBPS=$(echo "scale=0; $CURRENT_PEAK_MBPS * (1 + $GROWTH_RATE)^($MONTHS_TO_PLAN/12)" | bc -l)
REQUIRED_MBPS=$(echo "scale=0; $PROJECTED_MBPS * (1 + $SAFETY_MARGIN)" | bc -l)

echo "Current peak bandwidth: ${CURRENT_PEAK_MBPS} Mbps"
echo "Projected peak (${MONTHS_TO_PLAN} months): ${PROJECTED_MBPS} Mbps"
echo "Required with safety margin: ${REQUIRED_MBPS} Mbps"
echo ""

# Direct Connect bandwidth options
echo "AWS Direct Connect bandwidth options:"
echo "- 1 Gbps (1,000 Mbps)"
echo "- 10 Gbps (10,000 Mbps)"
echo "- 100 Gbps (100,000 Mbps)"
echo ""

# Recommendation logic
if [ $REQUIRED_MBPS -le 800 ]; then
    echo "RECOMMENDATION: 1 Gbps Direct Connect"
    echo "- Provides headroom for growth"
    echo "- Cost-effective for current needs"
elif [ $REQUIRED_MBPS -le 8000 ]; then
    echo "RECOMMENDATION: 10 Gbps Direct Connect"
    echo "- Accommodates projected growth"
    echo "- Better price-per-Mbps ratio"
else
    echo "RECOMMENDATION: Multiple 10 Gbps or 100 Gbps Direct Connect"
    echo "- High-capacity requirements"
    echo "- Consider LAG (Link Aggregation Group)"
fi

echo ""
echo "ADDITIONAL CONSIDERATIONS:"
echo "- Burstable vs sustained traffic patterns"
echo "- Peak vs average utilization"
echo "- Backup connectivity requirements"
echo "- Multi-region traffic distribution"

EOF

chmod +x dx-bandwidth-planning.sh
./dx-bandwidth-planning.sh
```

### Step 4: Cost Analysis and ROI Calculation
**Objective**: Analyze Direct Connect cost benefits

**Cost Comparison**:
```bash
# Create cost analysis tool
cat << 'EOF' > dx-cost-analysis.sh
#!/bin/bash

echo "=== Direct Connect Cost Analysis ==="
echo ""

# Input parameters (example values)
MONTHLY_DATA_OUT_GB=10000  # 10TB monthly egress
INTERNET_COST_PER_GB=0.09  # Standard internet data transfer cost
DX_PORT_COST=216.00        # 1Gbps Direct Connect port cost
DX_DATA_COST_PER_GB=0.02   # Direct Connect data transfer cost

# Calculate costs
INTERNET_MONTHLY_COST=$(echo "scale=2; $MONTHLY_DATA_OUT_GB * $INTERNET_COST_PER_GB" | bc -l)
DX_DATA_MONTHLY_COST=$(echo "scale=2; $MONTHLY_DATA_OUT_GB * $DX_DATA_COST_PER_GB" | bc -l)
DX_TOTAL_MONTHLY_COST=$(echo "scale=2; $DX_PORT_COST + $DX_DATA_MONTHLY_COST" | bc -l)

MONTHLY_SAVINGS=$(echo "scale=2; $INTERNET_MONTHLY_COST - $DX_TOTAL_MONTHLY_COST" | bc -l)
ANNUAL_SAVINGS=$(echo "scale=2; $MONTHLY_SAVINGS * 12" | bc -l)

echo "CURRENT INTERNET COSTS:"
echo "Monthly data transfer (${MONTHLY_DATA_OUT_GB} GB): \$${INTERNET_MONTHLY_COST}"
echo "Annual data transfer cost: \$$(echo "scale=2; $INTERNET_MONTHLY_COST * 12" | bc -l)"
echo ""

echo "DIRECT CONNECT COSTS:"
echo "Monthly port fee (1 Gbps): \$${DX_PORT_COST}"
echo "Monthly data transfer (${MONTHLY_DATA_OUT_GB} GB): \$${DX_DATA_MONTHLY_COST}"
echo "Total monthly Direct Connect cost: \$${DX_TOTAL_MONTHLY_COST}"
echo "Annual Direct Connect cost: \$$(echo "scale=2; $DX_TOTAL_MONTHLY_COST * 12" | bc -l)"
echo ""

echo "SAVINGS ANALYSIS:"
echo "Monthly savings: \$${MONTHLY_SAVINGS}"
echo "Annual savings: \$${ANNUAL_SAVINGS}"

# Break-even analysis
if (( $(echo "$MONTHLY_SAVINGS > 0" | bc -l) )); then
    echo "✅ Direct Connect provides immediate cost savings"
    ROI_PERCENTAGE=$(echo "scale=1; $ANNUAL_SAVINGS / ($DX_TOTAL_MONTHLY_COST * 12) * 100" | bc -l)
    echo "Annual ROI: ${ROI_PERCENTAGE}%"
else
    # Calculate break-even data volume
    BREAKEVEN_GB=$(echo "scale=0; $DX_PORT_COST / ($INTERNET_COST_PER_GB - $DX_DATA_COST_PER_GB)" | bc -l)
    echo "❌ Current volume too low for cost savings"
    echo "Break-even data volume: ${BREAKEVEN_GB} GB/month"
fi

echo ""
echo "OTHER BENEFITS TO CONSIDER:"
echo "- Consistent performance and lower latency"
echo "- Enhanced security and compliance"
echo "- Reduced internet gateway costs"
echo "- Improved reliability and SLA"

EOF

chmod +x dx-cost-analysis.sh
./dx-cost-analysis.sh
```

### Step 5: Direct Connect Architecture Design
**Objective**: Design comprehensive Direct Connect solution

**Architecture Planning**:
```bash
# Create architecture design template
cat << 'EOF' > dx-architecture-design.sh
#!/bin/bash

echo "=== Direct Connect Architecture Design ==="
echo ""

echo "1. CONNECTION ARCHITECTURE"
echo ""
echo "   Primary Direct Connect:"
echo "   ├── Location: [Primary DX Location]"
echo "   ├── Bandwidth: [1Gbps/10Gbps/100Gbps]"
echo "   ├── BGP ASN: [Customer ASN]"
echo "   └── Redundancy: [Active/Standby]"
echo ""
echo "   Secondary Direct Connect (Optional):"
echo "   ├── Location: [Secondary DX Location]"
echo "   ├── Bandwidth: [Same/Different]"
echo "   ├── BGP ASN: [Same/Different]"
echo "   └── Redundancy: [Active/Active]"
echo ""

echo "2. VIRTUAL INTERFACE DESIGN"
echo ""
echo "   Private VIF #1:"
echo "   ├── VLAN ID: [100-4094]"
echo "   ├── BGP ASN: [AWS ASN]"
echo "   ├── Customer IP: [169.254.x.x/30]"
echo "   ├── AWS IP: [169.254.x.x/30]"
echo "   └── Target: [VGW/TGW]"
echo ""
echo "   Public VIF #1 (Optional):"
echo "   ├── VLAN ID: [100-4094]"
echo "   ├── BGP ASN: [AWS ASN]"
echo "   ├── Customer IP: [Public IP/30]"
echo "   ├── AWS IP: [Public IP/30]"
echo "   └── Purpose: [S3/CloudFront/APIs]"
echo ""

echo "3. ROUTING DESIGN"
echo ""
echo "   BGP Configuration:"
echo "   ├── Customer routes advertised to AWS"
echo "   ├── AWS routes received from AWS"
echo "   ├── Route filtering and preferences"
echo "   └── Load balancing across connections"
echo ""

echo "4. REDUNDANCY AND FAILOVER"
echo ""
echo "   High Availability Options:"
echo "   ├── Multiple Direct Connect locations"
echo "   ├── Multiple connections per location"
echo "   ├── VPN backup for Direct Connect"
echo "   └── BGP route preferences and failover"
echo ""

echo "5. SECURITY CONSIDERATIONS"
echo ""
echo "   Security Measures:"
echo "   ├── BGP authentication"
echo "   ├── Route filtering"
echo "   ├── Optional encryption (IPSec/MACsec)"
echo "   └── Network segmentation"

EOF

chmod +x dx-architecture-design.sh
./dx-architecture-design.sh > dx-architecture.txt

echo "Architecture design template saved to dx-architecture.txt"
```

### Step 6: Implementation Roadmap
**Objective**: Create phased implementation plan

**Implementation Planning**:
```bash
# Create implementation roadmap
cat << 'EOF' > dx-implementation-roadmap.sh
#!/bin/bash

echo "=== Direct Connect Implementation Roadmap ==="
echo ""

echo "PHASE 1: PLANNING AND PREPARATION (Weeks 1-4)"
echo "├── Requirements gathering and analysis"
echo "├── Direct Connect location selection"
echo "├── Carrier selection and pricing"
echo "├── Network design and architecture"
echo "├── Security and compliance review"
echo "└── Project timeline and resource planning"
echo ""

echo "PHASE 2: PROCUREMENT AND SETUP (Weeks 5-8)"
echo "├── Direct Connect connection order"
echo "├── Carrier circuit provisioning"
echo "├── Customer router procurement/configuration"
echo "├── Colocation space setup (if needed)"
echo "├── Cross connect installation"
echo "└── Initial physical connectivity testing"
echo ""

echo "PHASE 3: LOGICAL CONFIGURATION (Weeks 9-10)"
echo "├── Virtual Interface creation"
echo "├── BGP session establishment"
echo "├── Route advertisement and testing"
echo "├── Virtual Private Gateway/Transit Gateway setup"
echo "├── Initial connectivity validation"
echo "└── Performance baseline establishment"
echo ""

echo "PHASE 4: INTEGRATION AND TESTING (Weeks 11-12)"
echo "├── Application connectivity testing"
echo "├── Performance optimization"
echo "├── Failover testing and validation"
echo "├── Security testing and validation"
echo "├── Monitoring and alerting setup"
echo "└── Documentation and procedures"
echo ""

echo "PHASE 5: PRODUCTION CUTOVER (Weeks 13-14)"
echo "├── Final pre-production testing"
echo "├── Change management approval"
echo "├── Gradual traffic migration"
echo "├── Full production cutover"
echo "├── Post-implementation monitoring"
echo "└── Project closure and lessons learned"
echo ""

echo "CONSIDERATIONS:"
echo "• Lead times vary by location and carrier"
echo "• Cross connect installation typically takes 2-5 business days"
echo "• BGP configuration and testing requires coordination"
echo "• Pilot testing recommended before full deployment"
echo "• Backup connectivity should remain until fully validated"

EOF

chmod +x dx-implementation-roadmap.sh
./dx-implementation-roadmap.sh
```

## Direct Connect vs Other Connectivity Options

### Comparison Matrix

| Factor | Direct Connect | Site-to-Site VPN | Internet Gateway | AWS PrivateLink |
|--------|---------------|------------------|------------------|-----------------|
| **Bandwidth** | Up to 100 Gbps | Up to 1.25 Gbps | Variable | Service dependent |
| **Latency** | Consistent, low | Variable | Variable | Low |
| **Cost** | High fixed + low transfer | Low fixed + transfer | Transfer only | Hourly + data |
| **Setup Time** | Weeks/months | Minutes/hours | Immediate | Hours |
| **Security** | Private connection | Encrypted tunnel | Public internet | Private connection |
| **Reliability** | 99.9% SLA | Best effort | Best effort | 99.9% SLA |
| **Use Case** | High volume, low latency | Secure connectivity | General access | Service connectivity |

### Decision Framework

```bash
# Create decision framework tool
cat << 'EOF' > connectivity-decision-framework.sh
#!/bin/bash

echo "=== AWS Connectivity Decision Framework ==="
echo ""

echo "Choose DIRECT CONNECT when:"
echo "✓ Monthly data transfer > 1TB consistently"
echo "✓ Require consistent, low latency connectivity"
echo "✓ Need dedicated bandwidth guarantees"
echo "✓ Have compliance requirements for private connectivity"
echo "✓ Can justify higher upfront costs with volume savings"
echo "✓ Long-term commitment to AWS (12+ months)"
echo ""

echo "Choose SITE-TO-SITE VPN when:"
echo "✓ Need secure connectivity immediately"
echo "✓ Lower bandwidth requirements (< 1 Gbps)"
echo "✓ Variable or unpredictable traffic patterns"
echo "✓ Cost optimization is primary concern"
echo "✓ Temporary or proof-of-concept deployments"
echo "✓ Backup connectivity for Direct Connect"
echo ""

echo "Choose INTERNET GATEWAY when:"
echo "✓ Public services access is sufficient"
echo "✓ No specific security or performance requirements"
echo "✓ Minimal setup time required"
echo "✓ Cost optimization for low-volume workloads"
echo ""

echo "Consider HYBRID APPROACH when:"
echo "✓ Need both primary and backup connectivity"
echo "✓ Different applications have different requirements"
echo "✓ Gradual migration to Direct Connect"
echo "✓ Geographic distribution requires multiple options"

EOF

chmod +x connectivity-decision-framework.sh
./connectivity-decision-framework.sh
```

## Best Practices Summary

### Planning Best Practices
- **Requirements Analysis**: Thoroughly assess bandwidth, latency, and security needs
- **Location Selection**: Choose locations based on proximity and redundancy
- **Capacity Planning**: Plan for growth and peak utilization
- **Cost Analysis**: Calculate ROI including soft benefits

### Design Best Practices
- **Redundancy**: Implement multiple connections for high availability
- **BGP Design**: Use proper BGP attributes for optimal routing
- **Security**: Implement appropriate filtering and authentication
- **Monitoring**: Set up comprehensive monitoring and alerting

### Implementation Best Practices
- **Phased Approach**: Implement in phases with testing at each stage
- **Testing**: Comprehensive testing before production cutover
- **Documentation**: Maintain detailed configuration documentation
- **Training**: Ensure operations team is trained on Direct Connect management

## Troubleshooting Common Issues

| Issue | Symptoms | Diagnosis | Solution |
|-------|----------|-----------|----------|
| BGP session down | No connectivity | Check physical layer | Verify cross connect and BGP config |
| High latency | Poor performance | Monitor connection | Check routing and carrier path |
| Partial connectivity | Some resources unreachable | Review route tables | Verify BGP advertisements |
| Cost overruns | Higher than expected bills | Analyze usage patterns | Optimize routing and usage |

## Key Takeaways
- Direct Connect provides dedicated, high-performance connectivity to AWS
- Multiple virtual interface types support different connectivity needs
- Cost benefits depend on data volume and bandwidth requirements
- Proper planning and design are critical for successful implementation
- Redundancy and monitoring are essential for production deployments

## Additional Resources
- [AWS Direct Connect Documentation](https://docs.aws.amazon.com/directconnect/)
- [Direct Connect Getting Started Guide](https://docs.aws.amazon.com/directconnect/latest/UserGuide/getting_started.html)
- [Direct Connect Best Practices](https://docs.aws.amazon.com/directconnect/latest/UserGuide/best-practices.html)

## Exam Tips
- Understand the difference between VIF types (Private, Public, Transit)
- Know Direct Connect bandwidth options and limitations
- Remember that Direct Connect does not provide encryption by default
- BGP is required for all Direct Connect connections
- Cross connects are physical fiber connections managed by colocation facility
- Direct Connect Gateway enables connection to multiple regions
- LAG (Link Aggregation Group) allows multiple connections as single logical connection