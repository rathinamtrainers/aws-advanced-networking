# Lab 11: AWS Direct Connect Introduction and Use Cases

## Lab Overview
This lab provides hands-on experience with AWS Direct Connect concepts, exploring the service through the AWS Console and understanding its use cases compared to internet-based connectivity.

## Prerequisites
- AWS Account with appropriate permissions
- Basic understanding of VPC concepts
- Access to AWS Direct Connect console

## Lab Duration
45 minutes

## Learning Objectives
By the end of this lab, you will:
- Navigate the AWS Direct Connect console
- Understand Direct Connect components
- Compare Direct Connect vs internet connectivity
- Explore Direct Connect partner locations
- Understand pricing models

## Lab Steps

### Step 1: Explore the Direct Connect Console
1. **Access Direct Connect Service**
   - Sign in to AWS Management Console
   - Navigate to **Services** → **Networking & Content Delivery** → **Direct Connect**
   - Take note of the current region in the top-right corner

2. **Explore the Dashboard**
   - Review the Direct Connect dashboard
   - Note the sections: Connections, Virtual Interfaces, Direct Connect Gateways
   - Click on each section to understand the interface (they will be empty initially)

### Step 2: Understand Direct Connect Locations
1. **View Available Locations**
   - In the left navigation, click **Direct Connect locations**
   - Filter by your preferred region
   - Select 2-3 locations near your geographic area
   - Note the available speeds and partners for each location

2. **Analyze Location Details**
   - Click on a specific location
   - Review available partners
   - Note the different connection speeds available (1Gbps, 10Gbps, 100Gbps)
   - Document the physical address and facility information

### Step 3: Explore Connection Types
1. **Dedicated Connections**
   - Click **Connections** in the left navigation
   - Click **Create connection** (don't actually create one)
   - Review the dedicated connection options:
     - Port speeds available
     - Partner locations
     - Estimated setup time
   - Click **Cancel** when done exploring

2. **Hosted Connections**
   - Research hosted connection options through partners
   - Compare bandwidth options (50Mbps to 10Gbps)
   - Note the difference in setup process vs dedicated connections

### Step 4: Virtual Interface (VIF) Concepts
1. **Explore VIF Types**
   - Navigate to **Virtual interfaces**
   - Click **Create virtual interface** (don't complete)
   - Review the three VIF types:
     - **Private VIF**: Access VPC resources
     - **Public VIF**: Access AWS public services
     - **Transit VIF**: Connect to Transit Gateway
   - Note the configuration options for each type
   - Cancel the creation process

### Step 5: Direct Connect Gateway Exploration
1. **Gateway Concepts**
   - Navigate to **Direct Connect gateways**
   - Click **Create Direct Connect gateway** (don't complete)
   - Understand how gateways enable multi-region connectivity
   - Review ASN (Autonomous System Number) requirements
   - Cancel the creation

### Step 6: Cost Analysis
1. **Pricing Calculator**
   - Open AWS Pricing Calculator in a new tab
   - Add Direct Connect service
   - Configure a sample setup:
     - 1Gbps dedicated connection
     - US East (N. Virginia) region
     - 1TB data transfer per month
   - Compare with equivalent data transfer over internet

2. **Cost Comparison Exercise**
   - Calculate monthly costs for:
     - Port hours (connection fees)
     - Data transfer fees
     - Compare with NAT Gateway + internet transfer costs
   - Document the breakeven point for your use case

### Step 7: Use Case Analysis
1. **Network Performance Comparison**
   - Open CloudWatch in a new tab
   - Review available Direct Connect metrics:
     - Connection state
     - Bandwidth utilization
     - Packet loss
     - Latency
   - Note how these would differ from internet connectivity

2. **Security Benefits**
   - Document security advantages:
     - Private connection to AWS
     - Bypass internet routing
     - Predictable bandwidth
     - Reduced data exposure

## Lab Questions
Answer these questions based on your exploration:

1. **Location Analysis**
   - Which Direct Connect location is closest to your organization?
   - What connection speeds are available at that location?
   - Which partners operate at that facility?

2. **Cost Comparison**
   - At what monthly data transfer volume does Direct Connect become cost-effective compared to internet transfer?
   - What are the fixed monthly costs regardless of usage?

3. **Use Case Matching**
   - For each scenario, recommend Direct Connect or internet connectivity:
     - Occasional file uploads to S3 (< 100GB/month)
     - Real-time data replication (> 10TB/month)
     - Backup to AWS (scheduled, 1TB weekly)
     - Mission-critical application with strict latency requirements

4. **Technical Requirements**
   - What BGP ASN would you need for a Private VIF?
   - How many VLANs can you configure on a single physical connection?
   - What's the maximum number of prefixes you can advertise?

## Clean Up
No resources were created in this lab, so no cleanup is required.

## Key Takeaways
- Direct Connect provides dedicated network connectivity to AWS
- Multiple connection types serve different bandwidth and cost requirements
- Virtual Interfaces (VIFs) enable different types of AWS service access
- Cost benefits increase with higher data transfer volumes
- Location and partner selection impact implementation timeline

## Next Steps
- Proceed to Lab 12: Setting Up AWS Direct Connect
- Review your organization's network requirements
- Contact Direct Connect partners for pricing and availability