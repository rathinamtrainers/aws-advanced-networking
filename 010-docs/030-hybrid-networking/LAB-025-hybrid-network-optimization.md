# Lab 25: Hybrid Network Optimization - Performance and Cost

## Lab Overview
This comprehensive lab focuses on optimizing hybrid network architectures for performance, cost, and reliability. Students will implement monitoring, analyze traffic patterns, optimize routing, and implement cost-effective solutions for hybrid connectivity.

## Prerequisites
- AWS Account with full networking permissions
- Completed previous hybrid networking labs
- Understanding of network performance metrics
- Basic knowledge of cost optimization principles

## Lab Duration
105 minutes

## Learning Objectives
By the end of this lab, you will:
- Implement comprehensive network monitoring and analytics
- Optimize routing for performance and cost
- Configure traffic engineering and load balancing
- Implement automated optimization responses
- Analyze and optimize data transfer costs
- Create performance baselines and SLAs
- Design cost-effective hybrid architectures

## Lab Architecture
```
┌─────────────────────────────────────────────────────────────┐
│                    Optimization Dashboard                    │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │CloudWatch   │ │VPC Flow     │ │Custom       │           │
│  │Metrics      │ │Logs         │ │Metrics      │           │
│  └─────────────┘ └─────────────┘ └─────────────┘           │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                 Hybrid Network Infrastructure               │
│                                                            │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │ Direct       │    │ Transit      │    │ VPN          │ │
│  │ Connect      │◄───►│ Gateway      │◄───►│ Connections  │ │
│  └──────────────┘    └──────────────┘    └──────────────┘ │
│                              │                             │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │ VPC-A        │    │ VPC-B        │    │ VPC-C        │ │
│  │ Production   │    │ Development  │    │ Shared       │ │
│  └──────────────┘    └──────────────┘    └──────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## Lab Steps

### Step 1: Create Comprehensive Monitoring Infrastructure
1. **Set Up Base Hybrid Environment**
   - Create multi-VPC environment with Transit Gateway
   - Configure VPN and simulate Direct Connect
   - Deploy test applications across VPCs
   - Establish baseline connectivity

2. **Enable VPC Flow Logs**
   - Enable Flow Logs for all VPCs:
   ```bash
   # Enable flow logs for each VPC
   aws ec2 create-flow-logs \
     --resource-type VPC \
     --resource-ids vpc-12345678 \
     --traffic-type ALL \
     --log-destination s3://network-flow-logs-bucket/vpc-logs/ \
     --log-destination-type s3
   ```

3. **Configure CloudWatch Custom Metrics**
   - Create custom metrics collection script:
   ```python
   #!/usr/bin/env python3
   import boto3
   import time
   import subprocess
   import json
   
   cloudwatch = boto3.client('cloudwatch')
   
   def collect_network_metrics():
       """Collect custom network performance metrics"""
       
       # Test latency to different endpoints
       endpoints = [
           {'name': 'OnPrem-VPN', 'ip': '192.168.1.10'},
           {'name': 'VPC-A', 'ip': '10.100.1.10'},
           {'name': 'VPC-B', 'ip': '10.200.1.10'}
       ]
       
       for endpoint in endpoints:
           try:
               # Measure ping latency
               result = subprocess.run(['ping', '-c', '4', endpoint['ip']], 
                                     capture_output=True, text=True)
               
               if result.returncode == 0:
                   # Parse ping results for average latency
                   lines = result.stdout.split('\n')
                   for line in lines:
                       if 'avg' in line:
                           avg_latency = float(line.split('/')[4])
                           
                           # Send to CloudWatch
                           cloudwatch.put_metric_data(
                               Namespace='Custom/Network',
                               MetricData=[
                                   {
                                       'MetricName': 'Latency',
                                       'Dimensions': [
                                           {
                                               'Name': 'Endpoint',
                                               'Value': endpoint['name']
                                           }
                                       ],
                                       'Value': avg_latency,
                                       'Unit': 'Milliseconds'
                                   }
                               ]
                           )
           except Exception as e:
               print(f"Error testing {endpoint['name']}: {e}")
   
   if __name__ == "__main__":
       while True:
           collect_network_metrics()
           time.sleep(60)  # Collect every minute
   ```

### Step 2: Implement Traffic Analysis
1. **Set Up VPC Flow Logs Analysis**
   - Create Athena table for Flow Logs analysis:
   ```sql
   CREATE EXTERNAL TABLE vpc_flow_logs (
     version int,
     account string,
     interfaceid string,
     sourceaddress string,
     destinationaddress string,
     sourceport int,
     destinationport int,
     protocol int,
     numpackets int,
     numbytes bigint,
     windowstart int,
     windowend int,
     action string
   )
   PARTITIONED BY (dt string)
   STORED AS PARQUET
   LOCATION 's3://network-flow-logs-bucket/vpc-logs/'
   ```

2. **Create Traffic Analysis Queries**
   ```sql
   -- Top traffic flows by bytes
   SELECT 
     sourceaddress, 
     destinationaddress, 
     SUM(numbytes) as total_bytes,
     COUNT(*) as flow_count
   FROM vpc_flow_logs 
   WHERE dt = '2023-10-01'
     AND action = 'ACCEPT'
   GROUP BY sourceaddress, destinationaddress
   ORDER BY total_bytes DESC
   LIMIT 20;
   
   -- Cross-VPC traffic analysis
   SELECT 
     CASE 
       WHEN sourceaddress LIKE '10.100.%' THEN 'VPC-A'
       WHEN sourceaddress LIKE '10.200.%' THEN 'VPC-B' 
       WHEN sourceaddress LIKE '192.168.%' THEN 'OnPrem'
       ELSE 'Other'
     END as source_network,
     CASE 
       WHEN destinationaddress LIKE '10.100.%' THEN 'VPC-A'
       WHEN destinationaddress LIKE '10.200.%' THEN 'VPC-B'
       WHEN destinationaddress LIKE '192.168.%' THEN 'OnPrem'
       ELSE 'Other'
     END as dest_network,
     SUM(numbytes) as total_bytes
   FROM vpc_flow_logs 
   WHERE dt = '2023-10-01'
   GROUP BY source_network, dest_network
   ORDER BY total_bytes DESC;
   ```

### Step 3: Optimize Routing Performance
1. **Implement BGP Route Optimization**
   - Configure BGP communities for traffic engineering:
   ```bash
   # Example Quagga/FRR configuration
   router bgp 65001
    neighbor 169.254.21.2 remote-as 64512
    neighbor 169.254.21.2 route-map PREPEND-BACKUP out
    neighbor 169.254.22.2 remote-as 64512
    neighbor 169.254.22.2 route-map PRIMARY out
   
   route-map PRIMARY permit 10
    set local-preference 150
   
   route-map PREPEND-BACKUP permit 10
    set as-path prepend 65001 65001
    set local-preference 100
   ```

2. **Configure ECMP (Equal Cost Multi-Path)**
   - Enable multiple path routing where supported:
   ```bash
   # Configure multiple VPN tunnels for load balancing
   aws ec2 modify-vpc-attribute \
     --vpc-id vpc-12345678 \
     --enable-dns-hostnames \
     --enable-dns-support
   ```

3. **Implement Traffic Engineering**
   - Create route preferences based on application requirements:
   - Low-latency traffic → Direct Connect
   - Backup traffic → VPN
   - Development traffic → Lower cost paths

### Step 4: Cost Optimization Analysis
1. **Data Transfer Cost Analysis**
   - Create cost tracking script:
   ```python
   import boto3
   import json
   from datetime import datetime, timedelta
   
   def analyze_data_transfer_costs():
       """Analyze data transfer costs across services"""
       
       # Get VPN data transfer
       cloudwatch = boto3.client('cloudwatch')
       
       end_time = datetime.utcnow()
       start_time = end_time - timedelta(days=30)
       
       # Get VPN tunnel bytes
       vpn_metrics = cloudwatch.get_metric_statistics(
           Namespace='AWS/VPN',
           MetricName='TunnelIpAddress',
           Dimensions=[],
           StartTime=start_time,
           EndTime=end_time,
           Period=3600,
           Statistics=['Sum']
       )
       
       # Calculate approximate costs
       # VPN data transfer: $0.09/GB outbound
       # Direct Connect: varies by location, typically lower
       
       total_gb = sum(point['Sum'] for point in vpn_metrics['Datapoints']) / (1024**3)
       vpn_cost = total_gb * 0.09
       dx_cost = total_gb * 0.02  # Example Direct Connect rate
       
       print(f"Monthly data transfer: {total_gb:.2f} GB")
       print(f"VPN cost: ${vpn_cost:.2f}")
       print(f"Direct Connect cost: ${dx_cost:.2f}")
       print(f"Potential savings: ${vpn_cost - dx_cost:.2f}")
       
       return {
           'total_gb': total_gb,
           'vpn_cost': vpn_cost,
           'dx_cost': dx_cost,
           'savings': vpn_cost - dx_cost
       }
   ```

2. **Regional Optimization Analysis**
   - Analyze costs by region and optimize placement:
   ```python
   def analyze_regional_costs():
       """Analyze costs across regions for optimization"""
       
       regions = ['us-east-1', 'us-west-2', 'eu-west-1']
       
       for region in regions:
           # Analyze compute costs
           # Analyze data transfer costs
           # Factor in compliance requirements
           
           print(f"Region: {region}")
           # Implementation details for cost comparison
   ```

### Step 5: Performance Optimization
1. **Implement Network Performance Monitoring**
   - Create performance monitoring dashboard:
   ```bash
   # Create CloudWatch dashboard
   aws cloudwatch put-dashboard \
     --dashboard-name "Hybrid-Network-Performance" \
     --dashboard-body file://dashboard-config.json
   ```

2. **Optimize MTU Settings**
   - Test and configure optimal MTU:
   ```bash
   # Test MTU discovery
   ping -M do -s 1472 target-ip  # Should work (1500 MTU)
   ping -M do -s 1436 target-ip  # VPN optimal MTU
   
   # Configure interface MTU
   sudo ip link set dev eth0 mtu 1436
   
   # Make permanent in network configuration
   echo "MTU=1436" >> /etc/sysconfig/network-scripts/ifcfg-eth0
   ```

3. **TCP Optimization for Hybrid Networks**
   ```bash
   # Optimize TCP settings for hybrid networks
   cat >> /etc/sysctl.conf << EOF
   # TCP optimization for hybrid networks
   net.core.rmem_max = 134217728
   net.core.wmem_max = 134217728
   net.ipv4.tcp_rmem = 4096 65536 134217728
   net.ipv4.tcp_wmem = 4096 65536 134217728
   net.ipv4.tcp_congestion_control = bbr
   net.core.default_qdisc = fq
   EOF
   
   sysctl -p
   ```

### Step 6: Automated Optimization
1. **Create Auto-Scaling Based on Network Metrics**
   - Lambda function for automatic route adjustments:
   ```python
   import boto3
   import json
   
   def lambda_handler(event, context):
       """Automatically adjust routes based on performance"""
       
       ec2 = boto3.client('ec2')
       cloudwatch = boto3.client('cloudwatch')
       
       # Get current latency metrics
       metrics = cloudwatch.get_metric_statistics(
           Namespace='Custom/Network',
           MetricName='Latency',
           Dimensions=[{'Name': 'Endpoint', 'Value': 'Primary-Path'}],
           StartTime=datetime.utcnow() - timedelta(minutes=15),
           EndTime=datetime.utcnow(),
           Period=300,
           Statistics=['Average']
       )
       
       if metrics['Datapoints']:
           avg_latency = sum(p['Average'] for p in metrics['Datapoints']) / len(metrics['Datapoints'])
           
           # If latency is high, consider route changes
           if avg_latency > 100:  # 100ms threshold
               # Implement route failover logic
               print(f"High latency detected: {avg_latency}ms")
               # Add route modification code here
       
       return {'statusCode': 200, 'body': json.dumps('Optimization complete')}
   ```

2. **Implement Cost-based Route Selection**
   ```python
   def optimize_routes_for_cost():
       """Select routes based on cost optimization"""
       
       # Get current data transfer patterns
       # Calculate costs for different route options
       # Automatically adjust route preferences
       
       route_options = [
           {'name': 'Direct Connect', 'cost_per_gb': 0.02, 'latency': 20},
           {'name': 'VPN Primary', 'cost_per_gb': 0.09, 'latency': 25},
           {'name': 'VPN Backup', 'cost_per_gb': 0.09, 'latency': 35}
       ]
       
       # Implement cost-based routing logic
       optimal_route = min(route_options, key=lambda x: x['cost_per_gb'])
       print(f"Optimal route: {optimal_route['name']}")
   ```

### Step 7: Security Optimization
1. **Implement Network Micro-segmentation**
   - Use security groups and NACLs optimally:
   ```python
   def optimize_security_groups():
       """Optimize security groups for performance and security"""
       
       ec2 = boto3.client('ec2')
       
       # Analyze security group rules
       security_groups = ec2.describe_security_groups()
       
       for sg in security_groups['SecurityGroups']:
           # Check for overly permissive rules
           # Optimize rule order for performance
           # Remove redundant rules
           pass
   ```

2. **Implement Zero Trust Networking**
   - Configure identity-based network access:
   - Use AWS IAM for network resource access
   - Implement certificate-based authentication

### Step 8: Create Performance Baselines and SLAs
1. **Establish Performance Baselines**
   ```python
   def create_performance_baseline():
       """Establish performance baselines for monitoring"""
       
       baselines = {
           'latency': {
               'onprem_to_vpc_a': {'target': 25, 'threshold': 50},
               'vpc_a_to_vpc_b': {'target': 5, 'threshold': 15},
               'cross_region': {'target': 80, 'threshold': 150}
           },
           'bandwidth': {
               'vpn_utilization': {'target': 70, 'threshold': 90},
               'direct_connect': {'target': 60, 'threshold': 85}
           },
           'availability': {
               'target': 99.9,
               'threshold': 99.5
           }
       }
       
       return baselines
   ```

2. **Implement SLA Monitoring**
   - Create CloudWatch alarms for SLA violations
   - Set up automated notifications
   - Implement SLA reporting

### Step 9: Cost Optimization Recommendations
1. **Create Cost Optimization Engine**
   ```python
   def generate_cost_recommendations():
       """Generate cost optimization recommendations"""
       
       recommendations = []
       
       # Analyze current usage patterns
       current_usage = analyze_current_usage()
       
       # Check if Direct Connect would be more cost-effective
       if current_usage['monthly_gb'] > 5000:
           savings = calculate_dx_savings(current_usage['monthly_gb'])
           recommendations.append({
               'type': 'connectivity',
               'recommendation': 'Consider Direct Connect',
               'savings': savings,
               'implementation_time': '6-8 weeks'
           })
       
       # Check for underutilized resources
       if current_usage['vpn_utilization'] < 30:
           recommendations.append({
               'type': 'rightsizing',
               'recommendation': 'Consider smaller VPN connections',
               'savings': calculate_rightsizing_savings(),
               'implementation_time': '1 week'
           })
       
       return recommendations
   ```

### Step 10: Create Optimization Dashboard
1. **Build Comprehensive Dashboard**
   - Create CloudWatch dashboard with key metrics
   - Include cost tracking and optimization recommendations
   - Add performance trend analysis

2. **Implement Reporting**
   ```python
   def generate_monthly_report():
       """Generate monthly optimization report"""
       
       report = {
           'performance_metrics': get_performance_summary(),
           'cost_analysis': get_cost_summary(),
           'optimization_opportunities': generate_cost_recommendations(),
           'sla_compliance': calculate_sla_compliance(),
           'recommendations': get_optimization_recommendations()
       }
       
       # Send report via email or store in S3
       return report
   ```

## Lab Validation
Complete these verification steps:

1. **Monitoring Infrastructure**
   - [ ] VPC Flow Logs enabled and analyzing
   - [ ] Custom metrics collecting performance data
   - [ ] CloudWatch dashboards operational
   - [ ] Automated alerting configured

2. **Performance Optimization**
   - [ ] Routing optimized for performance
   - [ ] MTU settings optimized
   - [ ] TCP parameters tuned
   - [ ] Baseline performance established

3. **Cost Optimization**
   - [ ] Cost analysis completed
   - [ ] Optimization recommendations generated
   - [ ] Automated cost monitoring in place
   - [ ] ROI calculations documented

## Performance Optimization Checklist

### Network Layer Optimizations
- [ ] MTU discovery and optimization
- [ ] TCP window scaling configured
- [ ] Congestion control algorithm optimized
- [ ] Buffer sizes tuned for network conditions

### Routing Optimizations
- [ ] BGP path preferences configured
- [ ] ECMP enabled where possible
- [ ] Route summarization implemented
- [ ] Traffic engineering policies applied

### Security Optimizations
- [ ] Security groups optimized for performance
- [ ] NACL rules minimized and ordered efficiently
- [ ] IPSec parameters optimized
- [ ] Certificate-based authentication where possible

### Cost Optimizations
- [ ] Data transfer patterns analyzed
- [ ] Right-sized connectivity options
- [ ] Regional optimization completed
- [ ] Reserved capacity considered

## Troubleshooting Performance Issues

### High Latency
```bash
# Test network path
traceroute target-ip
mtr -r -c 100 target-ip

# Check for DNS issues
dig target-hostname

# Monitor packet loss
ping -f -c 1000 target-ip
```

### Low Bandwidth
```bash
# Test bandwidth
iperf3 -c target-ip -P 4 -t 60

# Check for congestion
ss -i  # Check congestion window

# Monitor network utilization
iftop -i eth0
```

### Cost Issues
```bash
# Analyze data transfer
aws logs filter-log-events \
  --log-group-name VPCFlowLogs \
  --filter-pattern "{ $.action = \"ACCEPT\" }"

# Monitor service costs
aws ce get-cost-and-usage \
  --time-period Start=2023-10-01,End=2023-10-31 \
  --granularity MONTHLY \
  --metrics UnblendedCost
```

## Key Optimization Strategies

### 1. Right-sizing Connectivity
- Monitor actual bandwidth usage
- Choose appropriate connection types
- Consider burst vs sustained traffic patterns

### 2. Traffic Engineering
- Route latency-sensitive traffic optimally
- Use backup paths for non-critical traffic
- Implement quality of service (QoS) where possible

### 3. Cost Management
- Regularly review data transfer patterns
- Consider reserved capacity for predictable workloads
- Optimize regional data placement

### 4. Performance Monitoring
- Establish performance baselines
- Monitor key metrics continuously
- Set up proactive alerting

## Clean Up
To avoid ongoing charges:

1. **Stop Monitoring Services**
   - Disable custom metric collection scripts
   - Delete CloudWatch dashboards and alarms
   - Stop VPC Flow Logs collection

2. **Remove Optimization Infrastructure**
   - Delete Lambda functions
   - Remove S3 buckets for logs
   - Clean up Athena tables

## Key Takeaways
- Continuous monitoring is essential for optimization
- Performance and cost optimization often require trade-offs
- Automation enables proactive optimization responses
- Regular analysis and adjustment improve efficiency over time
- Baseline establishment is crucial for measuring improvements
- Hybrid network optimization requires holistic approach considering all components

## Next Steps
- Implement optimizations in production environment
- Establish regular optimization review cycles
- Consider advanced networking services like AWS Cloud WAN
- Explore machine learning for predictive optimization
- Document optimization playbooks for your organization