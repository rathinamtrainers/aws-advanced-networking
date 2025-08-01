# Lab 24: AWS Cloud WAN vs Transit Gateway Comparison

## Lab Overview
This comparative lab explores AWS Cloud WAN and Transit Gateway, helping students understand when to use each service for global network management. Students will implement both solutions and compare their capabilities, use cases, and management approaches.

## Prerequisites
- AWS Account with Cloud WAN and Transit Gateway permissions
- Understanding of global networking concepts
- Completed Transit Gateway lab (Lab 20)
- Basic knowledge of SD-WAN concepts

## Lab Duration
120 minutes

## Learning Objectives
By the end of this lab, you will:
- Compare Cloud WAN and Transit Gateway architectures
- Implement Cloud WAN global network design
- Configure network segmentation in both services
- Understand routing and policy differences
- Analyze cost models and scaling characteristics
- Make informed decisions on service selection
- Design hybrid architectures using both services

## Lab Architecture Comparison
```
Transit Gateway Architecture:
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Region A  │    │   Region B  │    │   Region C  │
│     TGW     │◄──►│     TGW     │◄──►│     TGW     │
│             │    │             │    │             │
│  ┌─────────┐│    │  ┌─────────┐│    │  ┌─────────┐│
│  │VPC│VPN││    │  │VPC│VPN││    │  │VPC│VPN││
│  └─────────┘│    │  └─────────┘│    │  └─────────┘│
└─────────────┘    └─────────────┘    └─────────────┘

Cloud WAN Architecture:
                ┌─────────────────┐
                │   Global Core    │
                │   Network       │
                └─────────────────┘
                         │
        ┌────────────────┼────────────────┐
        │                │                │
   ┌─────────┐      ┌─────────┐      ┌─────────┐
   │Region A │      │Region B │      │Region C │
   │Core Loc.│      │Core Loc.│      │Core Loc.│
   │  ┌───┐  │      │  ┌───┐  │      │  ┌───┐  │
   │  │VPC│  │      │  │VPC│  │      │  │VPC│  │
   │  │CPE│  │      │  │CPE│  │      │  │CPE│  │
   │  └───┘  │      │  └───┘  │      │  └───┘  │
   └─────────┘      └─────────┘      └─────────┘
```

## Lab Steps

### Step 1: Review Transit Gateway Architecture (Existing)
1. **Examine Current Transit Gateway Setup**
   - Review existing Transit Gateway from Lab 20
   - Document current architecture:
     - Number of attachments
     - Route table configuration
     - Cross-region peering setup
     - Bandwidth and routing patterns

2. **Document Transit Gateway Characteristics**
   - Regional service with inter-region peering
   - Hub-and-spoke within region
   - Manual route table management
   - BGP-based routing with on-premises
   - Per-attachment and data transfer pricing

### Step 2: Create Cloud WAN Global Network
1. **Enable Cloud WAN Service**
   - Navigate to **VPC Console** → **Cloud WAN**
   - Note: Cloud WAN may not be available in all regions
   - Choose supported region (us-east-1 recommended)

2. **Create Global Network**
   - Click **Create global network**
   - Configuration:
     - Name: `Lab-Global-Network`
     - Description: `Lab comparison between Cloud WAN and Transit Gateway`
     - Tags: Add appropriate tags

3. **Create Core Network**
   - In your global network, click **Create core network**
   - Configuration:
     - Name: `Lab-Core-Network`
     - Description: `Core network for global connectivity`
     - Policy document: We'll create this next

### Step 3: Design Cloud WAN Policy
1. **Create Network Policy Document**
   ```json
   {
     "version": "1.0",
     "core-network-configuration": {
       "vpn-ecmp-support": true,
       "asn-ranges": ["64512-65534"],
       "edge-locations": [
         {
           "location": "us-east-1",
           "asn": 64512
         },
         {
           "location": "us-west-2", 
           "asn": 64513
         },
         {
           "location": "eu-west-1",
           "asn": 64514
         }
       ]
     },
     "segments": [
       {
         "name": "production",
         "description": "Production workloads segment",
         "require-attachment-acceptance": false,
         "edge-locations": ["us-east-1", "us-west-2", "eu-west-1"]
       },
       {
         "name": "development", 
         "description": "Development workloads segment",
         "require-attachment-acceptance": false,
         "edge-locations": ["us-east-1", "us-west-2"]
       },
       {
         "name": "shared-services",
         "description": "Shared services segment",
         "require-attachment-acceptance": false,
         "edge-locations": ["us-east-1", "us-west-2", "eu-west-1"]
       }
     ],
     "segment-actions": [
       {
         "action": "share",
         "mode": "attachment-route",
         "segment": "shared-services",
         "share-with": ["production", "development"]
       }
     ],
     "attachment-policies": [
       {
         "rule-number": 100,
         "condition-logic": "or",
         "conditions": [
           {
             "type": "tag-value",
             "operator": "equals",
             "key": "Environment",
             "value": "Production"
           }
         ],
         "action": {
           "association-method": "constant",
           "segment": "production"
         }
       },
       {
         "rule-number": 200,
         "condition-logic": "or", 
         "conditions": [
           {
             "type": "tag-value",
             "operator": "equals",
             "key": "Environment", 
             "value": "Development"
           }
         ],
         "action": {
           "association-method": "constant",
           "segment": "development"
         }
       },
       {
         "rule-number": 300,
         "condition-logic": "or",
         "conditions": [
           {
             "type": "tag-value",
             "operator": "equals",
             "key": "Environment",
             "value": "Shared"
           }
         ],
         "action": {
           "association-method": "constant", 
           "segment": "shared-services"
         }
       }
     ]
   }
   ```

2. **Apply Policy to Core Network**
   - In Core Network details, click **Create policy**
   - Paste the policy document
   - Click **Create policy**
   - Wait for policy to be applied and core network to be available

### Step 4: Create VPCs for Cloud WAN Testing
1. **Create Production VPC (US East)**
   - Navigate to **VPC Console**
   - Create VPC:
     - Name: `CloudWAN-Prod-US-East`
     - CIDR: `10.10.0.0/16`
     - Tags: `Environment = Production`

2. **Create Development VPC (US West)**
   - Switch to **US West 2** region
   - Create VPC:
     - Name: `CloudWAN-Dev-US-West`
     - CIDR: `10.20.0.0/16`
     - Tags: `Environment = Development`

3. **Create Shared Services VPC (EU West)**
   - Switch to **EU West 1** region
   - Create VPC:
     - Name: `CloudWAN-Shared-EU-West`
     - CIDR: `10.30.0.0/16`
     - Tags: `Environment = Shared`

4. **Create Subnets in Each VPC**
   - Create private subnets in each VPC
   - Configure route tables
   - Add test instances for connectivity testing

### Step 5: Attach VPCs to Cloud WAN
1. **Create VPC Attachments**
   - In each region, navigate to **Cloud WAN**
   - Click **Create attachment**
   - For each VPC:
     - Attachment type: VPC
     - Core network: Select your core network
     - VPC: Select the appropriate VPC
     - Subnet: Select subnet for attachment
     - Tags: Ensure Environment tag is set correctly

2. **Verify Attachments and Segmentation**
   - Check that VPCs are attached to correct segments
   - Verify policy-based association worked
   - Monitor attachment status until "Available"

### Step 6: Compare Management Approaches
1. **Transit Gateway Management**
   - Review route table complexity
   - Document manual route propagation steps
   - Note cross-region peering requirements
   - Analyze attachment-by-attachment configuration

2. **Cloud WAN Management**
   - Review policy-driven configuration
   - Note automatic segment assignment
   - Observe centralized global view
   - Document simplified multi-region management

### Step 7: Test Connectivity Patterns
1. **Test Transit Gateway Connectivity**
   ```bash
   # From TGW environment, test:
   # - Intra-region VPC to VPC
   # - Cross-region VPC to VPC  
   # - On-premises to VPC
   
   # Document latency and routing paths
   traceroute target-ip
   ping -c 10 target-ip
   ```

2. **Test Cloud WAN Connectivity**
   ```bash
   # From Cloud WAN environment, test:
   # - Production to Shared Services (should work)
   # - Development to Shared Services (should work)
   # - Production to Development (should be blocked)
   
   # Test from each VPC
   ping prod-instance-ip
   ping dev-instance-ip  
   ping shared-instance-ip
   ```

3. **Document Connectivity Matrix**
   ```
   Source\Destination  | Prod | Dev | Shared | OnPrem
   -------------------|------|-----|--------|--------
   Production         |  ✓   |  ✗  |   ✓    |   ✓
   Development        |  ✗   |  ✓  |   ✓    |   ✓
   Shared Services    |  ✓   |  ✓  |   ✓    |   ✓
   On-Premises        |  ✓   |  ✓  |   ✓    |   ✓
   ```

### Step 8: Implement Hybrid Scenarios
1. **Connect On-Premises to Cloud WAN**
   - Create Site-to-Site VPN connection
   - Attach to Cloud WAN core network
   - Configure appropriate segment association
   - Test connectivity from on-premises to each segment

2. **Connect On-Premises to Transit Gateway**
   - Use existing VPN from previous labs
   - Compare configuration complexity
   - Test connectivity patterns

3. **Mixed Architecture Scenario**
   - Connect Cloud WAN to Transit Gateway
   - Route specific traffic through each service
   - Document integration patterns

### Step 9: Performance and Cost Analysis
1. **Performance Comparison**
   - Measure latency between regions:
   ```bash
   # Create performance test script
   #!/bin/bash
   echo "Testing Cloud WAN Performance"
   for i in {1..100}; do
     ping -c 1 cloudwan-target-ip | grep "time="
   done | awk -F'time=' '{print $2}' | awk '{print $1}' | sort -n
   
   echo "Testing Transit Gateway Performance"
   for i in {1..100}; do
     ping -c 1 tgw-target-ip | grep "time="
   done | awk -F'time=' '{print $2}' | awk '{print $1}' | sort -n
   ```

2. **Cost Analysis**
   - Compare pricing models:
   ```
   Transit Gateway Costs:
   - TGW per hour: $0.05/hour per TGW
   - Attachment per hour: $0.05/hour per attachment
   - Data processing: $0.02/GB
   - Inter-region peering: $0.05/hour + data transfer
   
   Cloud WAN Costs:
   - Core Network Edge: $0.35/hour per edge location
   - Attachments: $0.05/hour per attachment
   - Data processing: $0.02/GB
   - No additional inter-region charges
   ```

3. **Scaling Analysis**
   - Document scaling characteristics:
   ```
   Transit Gateway Scaling:
   - Up to 5,000 attachments per TGW
   - Multiple TGWs per region possible
   - Manual route table management
   - Regional peering required
   
   Cloud WAN Scaling:
   - Unlimited attachments per core network
   - Global policy management
   - Automatic route management
   - Native multi-region support
   ```

### Step 10: Advanced Features Comparison
1. **Security Features**
   - Compare security group integration
   - Analyze network segmentation capabilities
   - Review monitoring and logging differences

2. **Automation and Infrastructure as Code**
   - Create CloudFormation templates for both:
   ```yaml
   # Transit Gateway CloudFormation snippet
   TransitGateway:
     Type: AWS::EC2::TransitGateway
     Properties:
       AmazonSideAsn: 64512
       Description: Lab Transit Gateway
   
   # Cloud WAN CloudFormation snippet  
   GlobalNetwork:
     Type: AWS::NetworkManager::GlobalNetwork
     Properties:
       Description: Lab Global Network
   
   CoreNetwork:
     Type: AWS::NetworkManager::CoreNetwork
     Properties:
       GlobalNetworkId: !Ref GlobalNetwork
       PolicyDocument: !Ref PolicyDocument
   ```

3. **Integration Capabilities**
   - Compare integration with other AWS services
   - Review third-party integration options
   - Analyze API and SDK support

## Lab Validation
Complete these verification steps:

1. **Architecture Comparison**
   - [ ] Both Transit Gateway and Cloud WAN deployed
   - [ ] Connectivity tested in both environments
   - [ ] Performance differences documented
   - [ ] Cost analysis completed

2. **Feature Analysis**
   - [ ] Segmentation working in Cloud WAN
   - [ ] Route propagation working in Transit Gateway
   - [ ] Policy-based management demonstrated
   - [ ] Hybrid connectivity tested

3. **Decision Framework**
   - [ ] Use case mapping completed
   - [ ] Pros and cons documented
   - [ ] Migration considerations identified
   - [ ] Hybrid scenarios explored

## Decision Framework

### When to Choose Transit Gateway
```
✓ Regional hub-and-spoke requirements
✓ Existing TGW investments and expertise
✓ Fine-grained route control needed
✓ Specific compliance requirements
✓ Cost optimization for single-region
✓ Integration with existing network appliances
```

### When to Choose Cloud WAN
```
✓ Global network requirements
✓ Network segmentation needs
✓ Policy-driven management preferred
✓ Simplified multi-region connectivity
✓ SD-WAN-like capabilities needed
✓ Centralized network operations
```

### Hybrid Scenarios
```
✓ Use TGW for regional hubs
✓ Use Cloud WAN for global backbone
✓ Connect TGW to Cloud WAN as needed
✓ Migrate gradually from TGW to Cloud WAN
✓ Maintain both for different use cases
```

## Migration Considerations

### TGW to Cloud WAN Migration
1. **Assessment Phase**
   - Inventory existing TGW attachments
   - Analyze routing requirements
   - Identify segmentation needs
   - Plan migration timeline

2. **Design Phase**
   - Create Cloud WAN policy
   - Design segment architecture
   - Plan attachment migration
   - Design hybrid connectivity

3. **Implementation Phase**
   - Deploy Cloud WAN in parallel
   - Migrate attachments gradually
   - Test connectivity thoroughly
   - Update routing as needed

4. **Optimization Phase**
   - Remove redundant TGW resources
   - Optimize policies
   - Monitor performance
   - Document new architecture

## Troubleshooting Common Issues

### Cloud WAN Policy Issues
```bash
# Check policy syntax
aws networkmanager get-core-network-policy \
  --core-network-id core-network-id

# Verify segment associations
aws networkmanager list-attachments \
  --core-network-id core-network-id
```

### Connectivity Issues
```bash
# Check route tables in TGW
aws ec2 describe-transit-gateway-route-tables

# Check Cloud WAN routes
aws networkmanager get-route-analysis \
  --global-network-id global-network-id \
  --source-transit-gateway-attachment-arn source-arn \
  --destination-transit-gateway-attachment-arn dest-arn
```

### Performance Problems
```bash
# Monitor CloudWatch metrics for both services
# TGW metrics: PacketDropCount, BytesIn, BytesOut
# Cloud WAN metrics: Similar metrics per attachment
```

## Cost Optimization Strategies

### Transit Gateway Optimization
- Right-size attachments
- Optimize route propagation
- Use Direct Connect for high volume
- Monitor cross-region traffic

### Cloud WAN Optimization  
- Optimize edge location selection
- Right-size attachment requirements
- Use policy-based routing efficiently
- Monitor global data flows

## Clean Up
To avoid ongoing charges:

1. **Cloud WAN Cleanup**
   - Delete VPC attachments
   - Delete core network
   - Delete global network
   - Remove test VPCs

2. **Transit Gateway Cleanup**
   - Remove unnecessary attachments
   - Delete unused route tables
   - Remove cross-region peering
   - Clean up test resources

## Key Takeaways
- Cloud WAN provides global network management with policy-driven configuration
- Transit Gateway excels at regional hub-and-spoke architectures
- Cloud WAN simplifies multi-region connectivity and segmentation
- Both services can coexist in hybrid architectures
- Migration planning requires careful analysis of requirements
- Cost models differ significantly between the services
- Policy-based management in Cloud WAN reduces operational complexity

## Next Steps
- Move to Advanced VPC Networking labs (Topics 26-40)
- Explore production Cloud WAN implementations
- Consider AWS Network Manager for monitoring
- Design enterprise-grade global networks
- Implement automated network provisioning