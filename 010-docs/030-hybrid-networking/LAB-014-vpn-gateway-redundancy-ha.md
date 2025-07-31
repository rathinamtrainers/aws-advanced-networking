# Lab 14: VPN Gateway Redundancy and High Availability

## Lab Overview
This lab demonstrates how to implement high availability and redundancy patterns for AWS VPN connections. Students will configure multiple VPN tunnels, implement failover mechanisms, and test disaster recovery scenarios to ensure continuous connectivity.

## Prerequisites
- AWS Account with VPN and VPC permissions
- Completed Lab 15: Site-to-Site VPN Configuration
- Understanding of BGP routing and failover concepts
- Basic knowledge of network monitoring

## Lab Duration
105 minutes

## Learning Objectives
By the end of this lab, you will:
- Implement VPN redundancy with multiple connections
- Configure BGP failover and path selection
- Set up monitoring and alerting for VPN health
- Test failover scenarios and recovery procedures
- Optimize VPN performance and reliability
- Implement automated failover responses

## Lab Architecture
```
                    ┌─────────────────┐
                    │   AWS Region    │
                    │                 │
    ┌──────────┐    │  ┌─────────────┐│    ┌──────────────┐
    │Primary   │────┼──│VPN Gateway 1││────│              │
    │On-Prem   │    │  └─────────────┘│    │              │
    │Router    │    │                 │    │     VPC      │
    │          │────┼──│VPN Gateway 2││────│              │
    └──────────┘    │  └─────────────┘│    │              │
                    │                 │    └──────────────┘
    ┌──────────┐    │  ┌─────────────┐│
    │Secondary │────┼──│VPN Gateway 3││
    │On-Prem   │    │  └─────────────┘│
    │Router    │    └─────────────────┘
    └──────────┘
```

## Lab Steps

### Step 1: Prepare the Environment
1. **Create High Availability VPC**
   - Navigate to **VPC Console**
   - Create VPC:
     - Name: `HA-VPN-VPC`
     - IPv4 CIDR: `10.200.0.0/16`
     - Enable DNS hostnames and resolution

2. **Create Multi-AZ Subnets**
   - Private Subnet AZ-A:
     - Name: `HA-Private-Subnet-1A`
     - CIDR: `10.200.1.0/24`
     - AZ: us-east-1a
   - Private Subnet AZ-B:
     - Name: `HA-Private-Subnet-1B`
     - CIDR: `10.200.2.0/24`
     - AZ: us-east-1b
   - Public Subnet AZ-A:
     - Name: `HA-Public-Subnet-1A`
     - CIDR: `10.200.10.0/24`
     - AZ: us-east-1a

3. **Create Virtual Private Gateways**
   - Create Primary VGW:
     - Name: `Primary-VGW`
     - ASN: Amazon default (64512)
     - Attach to HA-VPN-VPC
   - Create Secondary VGW:
     - Name: `Secondary-VGW`
     - ASN: 64513
     - Keep unattached initially

### Step 2: Set Up Multiple Customer Gateways
1. **Primary Customer Gateway**
   - Navigate to **VPC** → **Customer Gateways**
   - Create Customer Gateway:
     - Name: `Primary-Customer-Gateway`
     - BGP ASN: `65001`
     - IP Address: `203.0.113.10` (primary site public IP)
     - Device: Primary site router

2. **Secondary Customer Gateway**
   - Create second Customer Gateway:
     - Name: `Secondary-Customer-Gateway`
     - BGP ASN: `65002`
     - IP Address: `203.0.113.20` (secondary site public IP)
     - Device: Secondary site router

3. **Backup Customer Gateway**
   - Create third Customer Gateway for same site:
     - Name: `Backup-Customer-Gateway`
     - BGP ASN: `65001` (same as primary)
     - IP Address: `203.0.113.11` (backup router public IP)

### Step 3: Create Redundant VPN Connections
1. **Primary VPN Connection**
   - Navigate to **Site-to-Site VPN Connections**
   - Create VPN Connection:
     - Name: `Primary-VPN-Connection`
     - Virtual Private Gateway: Primary-VGW
     - Customer Gateway: Primary-Customer-Gateway
     - Routing: Dynamic (BGP)
     - Static IP prefixes: (leave empty for BGP)

2. **Secondary VPN Connection (Different Site)**
   - Create VPN Connection:
     - Name: `Secondary-VPN-Connection`
     - Virtual Private Gateway: Primary-VGW
     - Customer Gateway: Secondary-Customer-Gateway
     - Routing: Dynamic (BGP)

3. **Backup VPN Connection (Same Site)**
   - Create VPN Connection:
     - Name: `Backup-VPN-Connection`
     - Virtual Private Gateway: Primary-VGW
     - Customer Gateway: Backup-Customer-Gateway
     - Routing: Dynamic (BGP)

### Step 4: Configure BGP Path Preferences
1. **Download All VPN Configurations**
   - Download configuration for each VPN connection
   - Note the BGP session parameters for each
   - Document tunnel endpoint IPs and pre-shared keys

2. **Configure Primary Path Preference**
   - For Primary VPN Connection:
     - Set higher local preference (150)
     - Shorter AS-PATH if possible
     - Configure as preferred route
   
3. **Configure Backup Path Preference**
   - For Secondary VPN Connection:
     - Set medium local preference (100)
     - Configure as secondary route
   - For Backup VPN Connection:
     - Set lower local preference (50)
     - Configure as tertiary route

### Step 5: Simulate On-Premises Configuration
1. **Create Simulation Environment**
   - Create separate VPC: `OnPrem-HA-Simulation`
   - CIDR: `192.168.0.0/16`
   - Create public subnets in multiple AZs
   - Launch EC2 instances to simulate routers

2. **Configure Primary Router Simulation**
   - Launch instance with Quagga/FRR:
   ```bash
   #!/bin/bash
   yum update -y
   yum install -y quagga
   systemctl enable zebra bgpd
   echo 'net.ipv4.ip_forward = 1' >> /etc/sysctl.conf
   sysctl -p
   ```

3. **Configure BGP on Simulated Routers**
   - Set up BGP sessions with AWS
   - Configure route advertisements
   - Implement path preferences with BGP attributes

### Step 6: Configure Route Tables for HA
1. **Enable Route Propagation**
   - Navigate to VPC Route Tables
   - For each private subnet route table:
     - Enable route propagation from Primary-VGW
     - Verify routes are being learned from all VPN connections

2. **Configure Route Priorities**
   - Monitor learned routes in VPC route table
   - Verify that BGP path selection works correctly
   - Document active vs backup route preferences

3. **Set Up Route Table Monitoring**
   - Create CloudWatch custom metric for route counts
   - Set up alarms for route table changes
   - Monitor for routing loops or inconsistencies

### Step 7: Implement Health Checks and Monitoring
1. **VPN Connection Health Checks**
   - Create CloudWatch dashboard for VPN metrics:
     - TunnelState for all connections
     - BGP session status
     - Data transfer metrics
     - Packet loss and latency

2. **Custom Health Monitoring**
   - Deploy monitoring instances in VPC
   - Configure synthetic transactions to on-premises
   - Set up end-to-end connectivity monitoring
   ```bash
   # Example monitoring script
   #!/bin/bash
   ONPREM_IP="192.168.1.10"
   while true; do
     if ping -c 1 -W 3 $ONPREM_IP > /dev/null; then
       aws cloudwatch put-metric-data --namespace "Custom/VPN" \
         --metric-data MetricName=Connectivity,Value=1
     else
       aws cloudwatch put-metric-data --namespace "Custom/VPN" \
         --metric-data MetricName=Connectivity,Value=0
     fi
     sleep 60
   done
   ```

3. **Set Up Alerting**
   - Create SNS topic for VPN alerts
   - Configure CloudWatch alarms:
     - VPN tunnel down
     - BGP session failure
     - High packet loss
     - Connectivity failure

### Step 8: Test Failover Scenarios
1. **Primary Tunnel Failure Test**
   - Simulate primary tunnel failure
   - Monitor automatic failover to secondary tunnel
   - Measure failover time and impact
   - Document recovery procedures

2. **Complete Site Failure Test**
   - Simulate entire primary site failure
   - Test failover to secondary site
   - Verify application continuity
   - Measure RTO (Recovery Time Objective)

3. **BGP Convergence Testing**
   - Monitor BGP route convergence times
   - Test with different failure scenarios
   - Optimize BGP timers for faster convergence

### Step 9: Implement Automated Responses
1. **Lambda-based Failover Automation**
   - Create Lambda function for automated responses:
   ```python
   import boto3
   import json
   
   def lambda_handler(event, context):
       # Parse CloudWatch alarm
       message = json.loads(event['Records'][0]['Sns']['Message'])
       
       if message['AlarmName'] == 'VPN-Primary-Down':
           # Trigger automated failover procedures
           ec2 = boto3.client('ec2')
           
           # Update route tables if needed
           # Send notifications
           # Log incident
           
       return {'statusCode': 200}
   ```

2. **Route Table Automation**
   - Implement automatic route table updates
   - Configure route preference changes
   - Set up rollback procedures

3. **Notification Automation**
   - Configure automated incident tickets
   - Set up escalation procedures
   - Implement status page updates

### Step 10: Performance Optimization
1. **Bandwidth Aggregation**
   - Configure ECMP (Equal Cost Multi-Path) if supported
   - Test load balancing across multiple tunnels
   - Monitor bandwidth utilization

2. **Latency Optimization**
   - Measure latency across different paths
   - Configure traffic steering based on performance
   - Implement jitter buffers for voice/video traffic

3. **MTU Optimization**
   - Test different MTU sizes
   - Configure path MTU discovery
   - Optimize for different application types

### Step 11: Security Hardening
1. **IPSec Parameter Optimization**
   - Configure strong encryption algorithms
   - Set up perfect forward secrecy
   - Implement regular key rotation

2. **Access Control**
   - Implement source-based routing restrictions
   - Configure firewall rules for VPN traffic
   - Set up intrusion detection

3. **Audit and Compliance**
   - Enable VPN connection logging
   - Configure compliance monitoring
   - Document security procedures

## Lab Validation
Complete these verification steps:

1. **Redundancy Verification**
   - [ ] Multiple VPN connections established
   - [ ] BGP sessions active on all connections
   - [ ] Route preferences configured correctly
   - [ ] Failover mechanisms working

2. **Monitoring Setup**
   - [ ] CloudWatch dashboards created
   - [ ] Alarms configured and tested
   - [ ] Automated responses functioning
   - [ ] Health checks operational

3. **Failover Testing**
   - [ ] Primary tunnel failure handled gracefully
   - [ ] Site-level failover working
   - [ ] Recovery procedures documented
   - [ ] Performance metrics within targets

## Troubleshooting Common Issues

### BGP Convergence Problems
- Check BGP timer configurations
- Verify AS-PATH and local preference settings
- Monitor for BGP route flapping
- Review BGP authentication settings

### Slow Failover Times
- Optimize BGP keepalive and hold timers
- Configure BFD (Bidirectional Forwarding Detection)
- Review health check intervals
- Check for DNS caching issues

### Route Selection Issues
- Verify BGP path attributes
- Check for conflicting static routes
- Monitor route propagation delays
- Review multi-path configurations

### Performance Degradation
- Monitor tunnel utilization and congestion
- Check for MTU/fragmentation issues
- Review encryption overhead
- Analyze traffic patterns

## Cost Optimization
1. **Connection Optimization**
   - Monitor actual usage across connections
   - Right-size number of VPN connections
   - Consider Direct Connect for high volume

2. **Data Transfer Optimization**
   - Implement traffic compression where possible
   - Optimize routing to minimize cross-AZ transfers
   - Use CloudFront for static content

## Disaster Recovery Planning
1. **RTO/RPO Targets**
   - Define recovery time objectives
   - Set recovery point objectives
   - Document acceptable downtime

2. **Runbooks**
   - Create detailed failover procedures
   - Document rollback processes
   - Maintain emergency contact lists

3. **Testing Schedule**
   - Regular failover testing (monthly)
   - Annual disaster recovery exercises
   - Documentation updates

## Clean Up
To avoid ongoing charges:

1. **Remove VPN Connections**
   - Delete all VPN connections
   - Remove customer gateways
   - Detach and delete virtual private gateways

2. **Clean Up Monitoring**
   - Remove CloudWatch alarms and dashboards
   - Delete Lambda functions
   - Remove SNS topics and subscriptions

3. **Remove Infrastructure**
   - Terminate EC2 instances
   - Delete VPCs and subnets
   - Remove security groups

## Key Takeaways
- Multiple VPN connections provide redundancy and increased bandwidth
- BGP path attributes control traffic routing and failover behavior
- Monitoring and alerting are critical for detecting failures
- Automated responses reduce recovery time
- Regular testing ensures failover procedures work correctly
- Documentation is essential for incident response

## Next Steps
- Proceed to Lab 16: VPN Tunneling and IPSec Deep Dive
- Implement production disaster recovery procedures
- Explore AWS Transit Gateway for simpler HA architectures
- Consider AWS Global Accelerator for improved performance