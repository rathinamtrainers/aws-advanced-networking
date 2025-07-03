# Topic 25: Hybrid Network Optimization and Performance Tuning

## Prerequisites
- Completed Topics 11-24 (Complete Hybrid Networking Foundation)
- Understanding of network performance metrics and monitoring
- Knowledge of AWS networking services and optimization techniques
- Familiarity with troubleshooting methodologies and tools

## Learning Objectives
By the end of this topic, you will be able to:
- Analyze and optimize hybrid network performance bottlenecks
- Implement advanced traffic engineering and path optimization
- Configure monitoring and alerting for hybrid network performance
- Design cost-optimized hybrid networking architectures

## Theory

### Hybrid Network Performance Fundamentals

#### Performance Metrics Overview
- **Latency**: Round-trip time between on-premises and AWS
- **Throughput**: Data transfer capacity (Mbps/Gbps)
- **Packet Loss**: Percentage of lost packets during transmission
- **Jitter**: Variation in packet arrival times
- **Availability**: Uptime percentage and mean time between failures

#### Performance Factors
- **Physical Distance**: Geographic distance affects latency
- **Network Path**: Internet vs dedicated connections (Direct Connect)
- **Bandwidth Allocation**: Available capacity vs utilization
- **Protocol Overhead**: IPSec, TCP, application protocol efficiency
- **Traffic Patterns**: Burst vs steady-state traffic characteristics

### Traffic Engineering for Hybrid Networks

#### Path Selection Optimization
```
Traffic Flow Decision Matrix:
┌─────────────────┬──────────────┬─────────────┬──────────────┐
│ Traffic Type    │ Path Choice  │ Latency     │ Cost         │
├─────────────────┼──────────────┼─────────────┼──────────────┤
│ Critical Apps   │ Direct Conn. │ Low (<50ms) │ High         │
│ Bulk Data       │ VPN/Internet │ Med (50-150)│ Low          │
│ Real-time       │ Direct Conn. │ Ultra Low   │ High         │
│ Backup/Archive  │ VPN/Internet │ High (>150) │ Very Low     │
└─────────────────┴──────────────┴─────────────┴──────────────┘
```

#### BGP Optimization Strategies

##### AS Path Prepending
```
Scenario: Prefer Direct Connect over VPN
Direct Connect: AS 65000 65001
VPN Backup:     AS 65000 65000 65000 65001 (prepended)
```

##### Local Preference Manipulation
```
BGP Local Preference Values:
- Direct Connect Primary: 200
- Direct Connect Secondary: 150
- VPN Primary: 100
- VPN Backup: 50
```

##### MED (Multi-Exit Discriminator) Tuning
```
Outbound Traffic Engineering:
AWS Region A → MED 100
AWS Region B → MED 200
(Lower MED preferred)
```

### Performance Optimization Strategies

#### Bandwidth Optimization

##### Traffic Shaping and QoS
```
QoS Priority Classes:
1. Real-time (Voice/Video): EF (Expedited Forwarding)
2. Critical Business: AF31 (Assured Forwarding)
3. Standard Business: AF21
4. Bulk Data: Best Effort
```

##### Connection Aggregation
```
ECMP (Equal Cost Multi-Path) Configuration:
┌─────────────┐    ┌──── Direct Connect 1 ────┐
│             │    │                          │
│ On-premises │────┤      (Load Balanced)     │──── AWS VPC
│             │    │                          │
└─────────────┘    └──── Direct Connect 2 ────┘
```

#### Protocol Optimization

##### TCP Window Scaling
```bash
# Optimize TCP window size for high-bandwidth, high-latency links
echo 'net.core.rmem_max = 67108864' >> /etc/sysctl.conf
echo 'net.core.wmem_max = 67108864' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_rmem = 4096 87380 67108864' >> /etc/sysctl.conf
echo 'net.ipv4.tcp_wmem = 4096 65536 67108864' >> /etc/sysctl.conf
```

##### IPSec Optimization
```
Optimized IPSec Parameters:
- IKE DH Group: 19 (256-bit ECC) or higher
- ESP Encryption: AES-256-GCM (hardware acceleration)
- ESP Authentication: AES-256-GCM (combined mode)
- PFS Group: 19 or higher
- SA Lifetime: Extended for stable connections
```

### Monitoring and Observability

#### CloudWatch Enhanced Monitoring

##### Custom Metrics Collection
```python
import boto3
import time
import subprocess

cloudwatch = boto3.client('cloudwatch')

def publish_network_metrics():
    # Measure latency to AWS
    result = subprocess.run(['ping', '-c', '4', 'ec2.amazonaws.com'], 
                          capture_output=True, text=True)
    
    # Parse ping results for latency
    avg_latency = parse_ping_latency(result.stdout)
    
    # Publish to CloudWatch
    cloudwatch.put_metric_data(
        Namespace='Hybrid/Network',
        MetricData=[
            {
                'MetricName': 'Latency',
                'Value': avg_latency,
                'Unit': 'Milliseconds',
                'Dimensions': [
                    {
                        'Name': 'ConnectionType',
                        'Value': 'DirectConnect'
                    }
                ]
            }
        ]
    )
```

#### VPC Flow Logs Analysis

##### Flow Log Processing Pipeline
```
VPC Flow Logs → CloudWatch Logs → Lambda → ElasticSearch → Kibana
                     ↓
               CloudWatch Insights
```

##### Flow Log Queries for Performance Analysis
```sql
-- Top talkers analysis
fields @timestamp, srcaddr, dstaddr, bytes
| filter bytes > 1000000
| stats sum(bytes) by srcaddr, dstaddr
| sort by sum desc
| limit 10

-- Latency analysis (synthetic)
fields @timestamp, start, end
| filter end > start
| eval duration = end - start
| stats avg(duration), max(duration) by 5m
```

## Lab Exercise: Comprehensive Hybrid Network Optimization

### Lab Duration: 360 minutes

### Step 1: Baseline Performance Assessment
**Objective**: Establish current performance metrics and identify bottlenecks

**Performance Baseline Script**:
```bash
# Create comprehensive performance assessment
cat << 'EOF' > hybrid-performance-baseline.sh
#!/bin/bash

echo "=== Hybrid Network Performance Baseline Assessment ==="
echo ""

echo "1. NETWORK CONNECTIVITY TESTING"

# Test connectivity to various AWS endpoints
ENDPOINTS=(
    "ec2.us-east-1.amazonaws.com"
    "s3.us-east-1.amazonaws.com"
    "rds.us-east-1.amazonaws.com"
    "lambda.us-east-1.amazonaws.com"
)

for endpoint in "${ENDPOINTS[@]}"; do
    echo "Testing connectivity to $endpoint:"
    
    # Latency test
    echo "  Latency test (ping):"
    ping -c 10 $endpoint | tail -1
    
    # HTTP response time test
    echo "  HTTP response time:"
    curl -o /dev/null -s -w "    Connect: %{time_connect}s, Total: %{time_total}s\n" \
         https://$endpoint/ 2>/dev/null || echo "    HTTP test failed"
    
    # DNS resolution time
    echo "  DNS resolution time:"
    time nslookup $endpoint >/dev/null 2>&1
    
    echo ""
done

echo "2. BANDWIDTH TESTING"

# Install iperf3 if not available
if ! command -v iperf3 &> /dev/null; then
    echo "Installing iperf3..."
    if [[ "$OSTYPE" == "linux-gnu"* ]]; then
        sudo yum install -y iperf3 || sudo apt-get install -y iperf3
    elif [[ "$OSTYPE" == "darwin"* ]]; then
        brew install iperf3
    fi
fi

echo "Bandwidth testing instructions:"
echo "1. Launch an EC2 instance in AWS with iperf3 server"
echo "2. Run: iperf3 -s on the EC2 instance"
echo "3. Run: iperf3 -c <EC2_INSTANCE_IP> from on-premises"

echo ""
echo "3. ROUTE ANALYSIS"

echo "Current routing table:"
if [[ "$OSTYPE" == "linux-gnu"* ]]; then
    ip route
elif [[ "$OSTYPE" == "darwin"* ]]; then
    netstat -rn
else
    route print
fi

echo ""
echo "4. NETWORK INTERFACE STATISTICS"

if [[ "$OSTYPE" == "linux-gnu"* ]]; then
    echo "Interface statistics:"
    cat /proc/net/dev
    
    echo ""
    echo "TCP statistics:"
    cat /proc/net/snmp | grep Tcp
fi

echo ""
echo "5. CREATING PERFORMANCE MONITORING SCRIPT"

cat << 'MONITOR' > network-performance-monitor.sh
#!/bin/bash

LOG_FILE="network_performance_$(date +%Y%m%d_%H%M%S).log"

echo "Starting network performance monitoring..."
echo "Logging to: $LOG_FILE"

while true; do
    TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
    
    # Ping test to AWS
    PING_RESULT=$(ping -c 1 ec2.us-east-1.amazonaws.com 2>/dev/null | \
                  grep 'time=' | awk -F'time=' '{print $2}' | awk '{print $1}')
    
    # Interface statistics
    if [[ "$OSTYPE" == "linux-gnu"* ]]; then
        RX_BYTES=$(cat /proc/net/dev | grep eth0 | awk '{print $2}')
        TX_BYTES=$(cat /proc/net/dev | grep eth0 | awk '{print $10}')
    else
        RX_BYTES="N/A"
        TX_BYTES="N/A"
    fi
    
    # Log the metrics
    echo "$TIMESTAMP,Latency_ms,$PING_RESULT,RX_Bytes,$RX_BYTES,TX_Bytes,$TX_BYTES" >> $LOG_FILE
    
    sleep 60
done
MONITOR

chmod +x network-performance-monitor.sh

echo "✅ Performance baseline assessment completed"
echo "Monitoring script created: network-performance-monitor.sh"

EOF

chmod +x hybrid-performance-baseline.sh
./hybrid-performance-baseline.sh
```

### Step 2: Implement Traffic Engineering
**Objective**: Configure BGP optimization and traffic path control

**BGP Optimization Script**:
```bash
# Configure BGP optimization for hybrid networks
cat << 'EOF' > configure-bgp-optimization.sh
#!/bin/bash

echo "=== Configuring BGP Optimization for Hybrid Networks ==="
echo ""

echo "1. BGP PATH ANALYSIS"

# Create BGP analysis script for virtual gateway
cat << 'BGP_ANALYSIS' > analyze-bgp-paths.sh
#!/bin/bash

echo "BGP Path Analysis for Virtual Private Gateway"
echo "============================================"

# Note: These commands require AWS CLI and appropriate permissions

VGW_ID="$1"
if [ -z "$VGW_ID" ]; then
    echo "Usage: $0 <VGW_ID>"
    echo "Example: $0 vgw-1234567890abcdef0"
    exit 1
fi

echo "Analyzing BGP routes for VGW: $VGW_ID"
echo ""

# Get BGP route information
echo "1. BGP ROUTE TABLE:"
aws ec2 describe-route-tables \
    --filters "Name=route.gateway-id,Values=$VGW_ID" \
    --query 'RouteTables[*].Routes[?GatewayId==`'$VGW_ID'`]' \
    --output table

echo ""
echo "2. VPN CONNECTION STATUS:"
aws ec2 describe-vpn-connections \
    --filters "Name=vpn-gateway-id,Values=$VGW_ID" \
    --query 'VpnConnections[*].[VpnConnectionId,State,CustomerGatewayId]' \
    --output table

echo ""
echo "3. ROUTE PROPAGATION STATUS:"
aws ec2 describe-route-tables \
    --filters "Name=route.gateway-id,Values=$VGW_ID" \
    --query 'RouteTables[*].[RouteTableId,PropagatingVgws[0].GatewayId]' \
    --output table

BGP_ANALYSIS

chmod +x analyze-bgp-paths.sh

echo "✅ BGP analysis script created: analyze-bgp-paths.sh"

echo ""
echo "2. TRANSIT GATEWAY BGP OPTIMIZATION"

# Create Transit Gateway BGP optimization
cat << 'TGW_BGP' > optimize-tgw-bgp.sh
#!/bin/bash

TGW_ID="$1"
if [ -z "$TGW_ID" ]; then
    echo "Usage: $0 <TGW_ID>"
    exit 1
fi

echo "Optimizing BGP for Transit Gateway: $TGW_ID"
echo ""

echo "1. ANALYZING TGW ROUTE TABLES:"
aws ec2 describe-transit-gateway-route-tables \
    --filters "Name=transit-gateway-id,Values=$TGW_ID" \
    --query 'TransitGatewayRouteTables[*].[TransitGatewayRouteTableId,State]' \
    --output table

echo ""
echo "2. CHECKING ROUTE PROPAGATION:"
TGW_RT_ID=$(aws ec2 describe-transit-gateway-route-tables \
    --filters "Name=transit-gateway-id,Values=$TGW_ID" \
    --query 'TransitGatewayRouteTables[0].TransitGatewayRouteTableId' \
    --output text)

if [ "$TGW_RT_ID" != "None" ]; then
    aws ec2 get-transit-gateway-route-table-propagations \
        --transit-gateway-route-table-id $TGW_RT_ID \
        --query 'TransitGatewayRouteTablePropagations[*].[TransitGatewayAttachmentId,State]' \
        --output table
fi

echo ""
echo "3. BGP OPTIMIZATION RECOMMENDATIONS:"
echo "   - Use AS path prepending for backup paths"
echo "   - Configure MED values for traffic engineering"
echo "   - Implement BGP communities for path selection"
echo "   - Monitor BGP route advertisements"

TGW_BGP

chmod +x optimize-tgw-bgp.sh

echo "✅ Transit Gateway BGP optimization script created"

echo ""
echo "3. CREATING BGP MONITORING SCRIPT"

cat << 'BGP_MONITOR' > monitor-bgp-performance.sh
#!/bin/bash

LOG_FILE="bgp_performance_$(date +%Y%m%d_%H%M%S).log"

echo "Starting BGP performance monitoring..."
echo "Logging to: $LOG_FILE"

# Function to get route count
get_route_count() {
    local target_ip="$1"
    if command -v traceroute &> /dev/null; then
        traceroute $target_ip 2>/dev/null | wc -l
    else
        echo "N/A"
    fi
}

# Function to measure path quality
measure_path_quality() {
    local target="$1"
    local path_name="$2"
    
    # Ping test
    ping_result=$(ping -c 5 $target 2>/dev/null | tail -1)
    
    # Extract average latency
    avg_latency=$(echo $ping_result | grep -o 'avg = [0-9.]*' | cut -d' ' -f3)
    
    # Extract packet loss
    packet_loss=$(ping -c 10 $target 2>/dev/null | grep 'packet loss' | \
                  grep -o '[0-9]*% packet loss' | cut -d'%' -f1)
    
    timestamp=$(date '+%Y-%m-%d %H:%M:%S')
    echo "$timestamp,$path_name,$target,$avg_latency,$packet_loss" >> $LOG_FILE
}

# Monitor multiple paths
while true; do
    measure_path_quality "ec2.us-east-1.amazonaws.com" "Primary_Path"
    measure_path_quality "s3.us-east-1.amazonaws.com" "S3_Path"
    
    sleep 300  # 5-minute intervals
done

BGP_MONITOR

chmod +x monitor-bgp-performance.sh

echo "✅ BGP monitoring script created"

echo ""
echo "BGP OPTIMIZATION CONFIGURATION COMPLETED"
echo ""
echo "Available scripts:"
echo "1. analyze-bgp-paths.sh <VGW_ID> - Analyze VPN BGP paths"
echo "2. optimize-tgw-bgp.sh <TGW_ID> - Optimize Transit Gateway BGP"
echo "3. monitor-bgp-performance.sh - Monitor BGP performance"

EOF

chmod +x configure-bgp-optimization.sh
./configure-bgp-optimization.sh
```

### Step 3: Implement Advanced Monitoring
**Objective**: Set up comprehensive monitoring and alerting

**Advanced Monitoring Setup**:
```bash
# Set up advanced monitoring for hybrid networks
cat << 'EOF' > setup-advanced-monitoring.sh
#!/bin/bash

echo "=== Setting Up Advanced Hybrid Network Monitoring ==="
echo ""

echo "1. CREATING CLOUDWATCH CUSTOM METRICS"

# Create Python script for custom metrics
cat << 'PYTHON_METRICS' > hybrid-network-metrics.py
#!/usr/bin/env python3

import boto3
import subprocess
import json
import time
import re
from datetime import datetime

class HybridNetworkMonitor:
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        self.namespace = 'Hybrid/Network'
    
    def measure_latency(self, target):
        """Measure latency using ping"""
        try:
            result = subprocess.run(
                ['ping', '-c', '4', target],
                capture_output=True,
                text=True,
                timeout=30
            )
            
            # Parse ping output for average latency
            if result.returncode == 0:
                match = re.search(r'avg = ([\d.]+)', result.stdout)
                if match:
                    return float(match.group(1))
            return None
            
        except subprocess.TimeoutExpired:
            return None
        except Exception as e:
            print(f"Error measuring latency to {target}: {e}")
            return None
    
    def measure_dns_resolution_time(self, hostname):
        """Measure DNS resolution time"""
        try:
            start_time = time.time()
            result = subprocess.run(
                ['nslookup', hostname],
                capture_output=True,
                timeout=10
            )
            end_time = time.time()
            
            if result.returncode == 0:
                return (end_time - start_time) * 1000  # Convert to milliseconds
            return None
            
        except subprocess.TimeoutExpired:
            return None
        except Exception as e:
            print(f"Error measuring DNS resolution for {hostname}: {e}")
            return None
    
    def get_interface_stats(self, interface='eth0'):
        """Get network interface statistics"""
        try:
            with open(f'/proc/net/dev', 'r') as f:
                for line in f:
                    if interface in line:
                        fields = line.split()
                        return {
                            'rx_bytes': int(fields[1]),
                            'tx_bytes': int(fields[9]),
                            'rx_packets': int(fields[2]),
                            'tx_packets': int(fields[10]),
                            'rx_errors': int(fields[3]),
                            'tx_errors': int(fields[11])
                        }
            return None
        except Exception as e:
            print(f"Error getting interface stats: {e}")
            return None
    
    def publish_metrics(self):
        """Publish all metrics to CloudWatch"""
        metrics = []
        
        # AWS service endpoints to monitor
        endpoints = {
            'EC2': 'ec2.us-east-1.amazonaws.com',
            'S3': 's3.us-east-1.amazonaws.com',
            'RDS': 'rds.us-east-1.amazonaws.com'
        }
        
        for service, endpoint in endpoints.items():
            # Latency metrics
            latency = self.measure_latency(endpoint)
            if latency:
                metrics.append({
                    'MetricName': 'Latency',
                    'Value': latency,
                    'Unit': 'Milliseconds',
                    'Dimensions': [
                        {'Name': 'Service', 'Value': service},
                        {'Name': 'ConnectionType', 'Value': 'Hybrid'}
                    ]
                })
            
            # DNS resolution time
            dns_time = self.measure_dns_resolution_time(endpoint)
            if dns_time:
                metrics.append({
                    'MetricName': 'DNSResolutionTime',
                    'Value': dns_time,
                    'Unit': 'Milliseconds',
                    'Dimensions': [
                        {'Name': 'Service', 'Value': service}
                    ]
                })
        
        # Interface statistics
        interface_stats = self.get_interface_stats()
        if interface_stats:
            for metric_name, value in interface_stats.items():
                unit = 'Bytes' if 'bytes' in metric_name else 'Count'
                metrics.append({
                    'MetricName': f'Interface{metric_name.title()}',
                    'Value': value,
                    'Unit': unit,
                    'Dimensions': [
                        {'Name': 'Interface', 'Value': 'eth0'}
                    ]
                })
        
        # Publish metrics in batches
        batch_size = 20
        for i in range(0, len(metrics), batch_size):
            batch = metrics[i:i+batch_size]
            try:
                self.cloudwatch.put_metric_data(
                    Namespace=self.namespace,
                    MetricData=batch
                )
                print(f"Published {len(batch)} metrics to CloudWatch")
            except Exception as e:
                print(f"Error publishing metrics: {e}")

def main():
    monitor = HybridNetworkMonitor()
    
    print("Starting hybrid network monitoring...")
    while True:
        try:
            monitor.publish_metrics()
            time.sleep(300)  # 5-minute intervals
        except KeyboardInterrupt:
            print("Monitoring stopped by user")
            break
        except Exception as e:
            print(f"Error in monitoring loop: {e}")
            time.sleep(60)

if __name__ == "__main__":
    main()

PYTHON_METRICS

chmod +x hybrid-network-metrics.py

echo "✅ Custom metrics script created: hybrid-network-metrics.py"

echo ""
echo "2. CREATING CLOUDWATCH DASHBOARD"

# Create dashboard configuration
cat << 'DASHBOARD' > hybrid-network-dashboard.json
{
    "widgets": [
        {
            "type": "metric",
            "x": 0,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "metrics": [
                    [ "Hybrid/Network", "Latency", "Service", "EC2" ],
                    [ ".", ".", ".", "S3" ],
                    [ ".", ".", ".", "RDS" ]
                ],
                "period": 300,
                "stat": "Average",
                "region": "us-east-1",
                "title": "Latency to AWS Services",
                "yAxis": {
                    "left": {
                        "min": 0
                    }
                }
            }
        },
        {
            "type": "metric",
            "x": 12,
            "y": 0,
            "width": 12,
            "height": 6,
            "properties": {
                "metrics": [
                    [ "AWS/VPN", "TunnelState", "VpnId", "vpn-example" ],
                    [ "AWS/DirectConnect", "ConnectionState", "ConnectionId", "dx-example" ]
                ],
                "period": 300,
                "stat": "Maximum",
                "region": "us-east-1",
                "title": "Connection Status"
            }
        },
        {
            "type": "metric",
            "x": 0,
            "y": 6,
            "width": 24,
            "height": 6,
            "properties": {
                "metrics": [
                    [ "Hybrid/Network", "InterfaceRxBytes", "Interface", "eth0" ],
                    [ ".", "InterfaceTxBytes", ".", "." ]
                ],
                "period": 300,
                "stat": "Average",
                "region": "us-east-1",
                "title": "Network Interface Throughput"
            }
        }
    ]
}
DASHBOARD

# Create dashboard
DASHBOARD_NAME="Hybrid-Network-Performance"
aws cloudwatch put-dashboard \
    --dashboard-name $DASHBOARD_NAME \
    --dashboard-body file://hybrid-network-dashboard.json

echo "✅ CloudWatch dashboard created: $DASHBOARD_NAME"

echo ""
echo "3. CREATING CLOUDWATCH ALARMS"

# High latency alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "Hybrid-Network-High-Latency" \
    --alarm-description "Alert when latency to AWS services is high" \
    --metric-name Latency \
    --namespace Hybrid/Network \
    --statistic Average \
    --period 300 \
    --threshold 100 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --alarm-actions "arn:aws:sns:us-east-1:123456789012:network-alerts"

# High error rate alarm
aws cloudwatch put-metric-alarm \
    --alarm-name "Hybrid-Network-High-Errors" \
    --alarm-description "Alert when network errors are high" \
    --metric-name InterfaceRxErrors \
    --namespace Hybrid/Network \
    --statistic Sum \
    --period 300 \
    --threshold 10 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 1

echo "✅ CloudWatch alarms created"

echo ""
echo "4. SETTING UP VPC FLOW LOGS ANALYSIS"

# Create Flow Logs analysis script
cat << 'FLOW_ANALYSIS' > analyze-flow-logs.py
#!/usr/bin/env python3

import boto3
import json
from datetime import datetime, timedelta

class FlowLogsAnalyzer:
    def __init__(self):
        self.logs_client = boto3.client('logs')
        self.log_group = '/aws/vpc/flowlogs'
    
    def analyze_top_talkers(self, hours=1):
        """Analyze top talking hosts in the last N hours"""
        end_time = datetime.now()
        start_time = end_time - timedelta(hours=hours)
        
        query = """
        fields @timestamp, srcaddr, dstaddr, bytes
        | filter bytes > 1000000
        | stats sum(bytes) as total_bytes by srcaddr, dstaddr
        | sort total_bytes desc
        | limit 20
        """
        
        return self.run_insights_query(query, start_time, end_time)
    
    def analyze_rejected_traffic(self, hours=1):
        """Analyze rejected traffic patterns"""
        end_time = datetime.now()
        start_time = end_time - timedelta(hours=hours)
        
        query = """
        fields @timestamp, srcaddr, dstaddr, srcport, dstport, action
        | filter action = "REJECT"
        | stats count() as reject_count by srcaddr, dstaddr, dstport
        | sort reject_count desc
        | limit 20
        """
        
        return self.run_insights_query(query, start_time, end_time)
    
    def run_insights_query(self, query, start_time, end_time):
        """Run CloudWatch Insights query"""
        try:
            response = self.logs_client.start_query(
                logGroupName=self.log_group,
                startTime=int(start_time.timestamp()),
                endTime=int(end_time.timestamp()),
                queryString=query
            )
            
            query_id = response['queryId']
            
            # Wait for query to complete
            import time
            while True:
                response = self.logs_client.get_query_results(queryId=query_id)
                if response['status'] == 'Complete':
                    return response['results']
                elif response['status'] == 'Failed':
                    print(f"Query failed: {response}")
                    return []
                time.sleep(2)
                
        except Exception as e:
            print(f"Error running query: {e}")
            return []

def main():
    analyzer = FlowLogsAnalyzer()
    
    print("Analyzing VPC Flow Logs...")
    print("=" * 50)
    
    print("\nTop Talkers (last hour):")
    top_talkers = analyzer.analyze_top_talkers()
    for result in top_talkers[:10]:
        print(f"  {result}")
    
    print("\nRejected Traffic (last hour):")
    rejected = analyzer.analyze_rejected_traffic()
    for result in rejected[:10]:
        print(f"  {result}")

if __name__ == "__main__":
    main()

FLOW_ANALYSIS

chmod +x analyze-flow-logs.py

echo "✅ Flow logs analysis script created"

echo ""
echo "ADVANCED MONITORING SETUP COMPLETED"
echo ""
echo "Monitoring components:"
echo "1. Custom metrics: hybrid-network-metrics.py"
echo "2. CloudWatch dashboard: $DASHBOARD_NAME"
echo "3. CloudWatch alarms for latency and errors"
echo "4. Flow logs analysis: analyze-flow-logs.py"
echo ""
echo "To start monitoring:"
echo "  python3 hybrid-network-metrics.py"

EOF

chmod +x setup-advanced-monitoring.sh
./setup-advanced-monitoring.sh
```

### Step 4: Cost Optimization Analysis
**Objective**: Analyze and optimize hybrid network costs

**Cost Optimization Script**:
```bash
# Analyze and optimize hybrid network costs
cat << 'EOF' > optimize-hybrid-costs.sh
#!/bin/bash

echo "=== Hybrid Network Cost Optimization Analysis ==="
echo ""

echo "1. DATA TRANSFER COST ANALYSIS"

# Create cost analysis script
cat << 'COST_ANALYSIS' > analyze-network-costs.py
#!/usr/bin/env python3

import boto3
import json
from datetime import datetime, timedelta

class NetworkCostAnalyzer:
    def __init__(self):
        self.ce_client = boto3.client('ce')  # Cost Explorer
        self.cloudwatch = boto3.client('cloudwatch')
    
    def get_data_transfer_costs(self, days=30):
        """Get data transfer costs for the last N days"""
        end_date = datetime.now().date()
        start_date = end_date - timedelta(days=days)
        
        try:
            response = self.ce_client.get_cost_and_usage(
                TimePeriod={
                    'Start': start_date.strftime('%Y-%m-%d'),
                    'End': end_date.strftime('%Y-%m-%d')
                },
                Granularity='DAILY',
                Metrics=['BlendedCost'],
                GroupBy=[
                    {
                        'Type': 'DIMENSION',
                        'Key': 'SERVICE'
                    }
                ],
                Filter={
                    'Dimensions': {
                        'Key': 'SERVICE',
                        'Values': [
                            'Amazon Elastic Compute Cloud - Compute',
                            'Amazon Virtual Private Cloud',
                            'AWS Direct Connect',
                            'Amazon CloudFront'
                        ]
                    }
                }
            )
            
            return response['ResultsByTime']
            
        except Exception as e:
            print(f"Error getting cost data: {e}")
            return []
    
    def analyze_data_transfer_patterns(self):
        """Analyze data transfer patterns for optimization"""
        print("Data Transfer Cost Optimization Recommendations:")
        print("=" * 50)
        
        recommendations = [
            {
                'category': 'Direct Connect',
                'recommendation': 'Use Direct Connect for consistent high-volume traffic',
                'savings': 'Up to 80% vs internet data transfer',
                'threshold': '>1TB/month cross-region or internet traffic'
            },
            {
                'category': 'VPC Endpoints',
                'recommendation': 'Use VPC Endpoints for AWS services',
                'savings': 'Eliminate internet gateway data charges',
                'threshold': 'High S3/DynamoDB usage'
            },
            {
                'category': 'CloudFront',
                'recommendation': 'Use CloudFront for global content delivery',
                'savings': '25-50% vs direct S3 delivery',
                'threshold': 'Global user base'
            },
            {
                'category': 'Cross-AZ Traffic',
                'recommendation': 'Minimize cross-AZ data transfer',
                'savings': '$0.01-0.02/GB saved',
                'threshold': 'High inter-AZ communication'
            }
        ]
        
        for rec in recommendations:
            print(f"\n{rec['category']}:")
            print(f"  Recommendation: {rec['recommendation']}")
            print(f"  Potential Savings: {rec['savings']}")
            print(f"  Apply When: {rec['threshold']}")
    
    def generate_cost_report(self):
        """Generate comprehensive cost report"""
        costs = self.get_data_transfer_costs()
        
        print("\nCost Analysis Report:")
        print("=" * 30)
        
        total_cost = 0
        service_costs = {}
        
        for day_data in costs:
            date = day_data['TimePeriod']['Start']
            for group in day_data['Groups']:
                service = group['Keys'][0]
                cost = float(group['Metrics']['BlendedCost']['Amount'])
                
                if service not in service_costs:
                    service_costs[service] = 0
                service_costs[service] += cost
                total_cost += cost
        
        print(f"Total Network-Related Costs (last 30 days): ${total_cost:.2f}")
        print("\nBreakdown by Service:")
        for service, cost in sorted(service_costs.items(), key=lambda x: x[1], reverse=True):
            percentage = (cost / total_cost * 100) if total_cost > 0 else 0
            print(f"  {service}: ${cost:.2f} ({percentage:.1f}%)")

def main():
    analyzer = NetworkCostAnalyzer()
    analyzer.generate_cost_report()
    analyzer.analyze_data_transfer_patterns()

if __name__ == "__main__":
    main()

COST_ANALYSIS

chmod +x analyze-network-costs.py

echo "✅ Cost analysis script created: analyze-network-costs.py"

echo ""
echo "2. BANDWIDTH OPTIMIZATION RECOMMENDATIONS"

cat << 'BANDWIDTH_OPT' > bandwidth-optimization.md
# Bandwidth Optimization Strategies

## 1. Connection Type Selection

### Direct Connect vs VPN Cost Comparison
```
Traffic Volume | Direct Connect | VPN (Internet) | Savings
1TB/month     | $100-150      | $200-300      | 40-50%
10TB/month    | $400-600      | $2000-3000    | 75-80%
100TB/month   | $2000-3000    | $20000-30000  | 85-90%
```

### Decision Matrix
- **< 1TB/month**: VPN over internet
- **1-10TB/month**: Consider Direct Connect
- **> 10TB/month**: Direct Connect recommended

## 2. Traffic Engineering Optimizations

### Path Selection Rules
1. **Critical Applications**: Direct Connect primary, VPN backup
2. **Bulk Data Transfer**: Lowest cost path (often VPN)
3. **Real-time Applications**: Lowest latency path (Direct Connect)
4. **Backup Traffic**: VPN or internet (lowest priority)

### QoS Implementation
```
Priority 1: Voice/Video (< 1% of bandwidth)
Priority 2: Critical Business Apps (20-30%)
Priority 3: Standard Business Apps (40-50%)
Priority 4: Bulk Data/Backup (remaining)
```

## 3. Compression and Caching

### Data Compression
- **Application-level**: 20-60% reduction
- **Transport-level**: 10-30% reduction
- **Content compression**: 50-90% for text/HTML

### Caching Strategies
- **CloudFront**: Reduce origin data transfer
- **Local caching**: Reduce WAN traffic
- **Database caching**: Reduce query traffic

## 4. Architecture Optimizations

### Regional Strategy
- **Data locality**: Keep data close to users
- **Multi-region**: Consider replication costs
- **Edge computing**: Process data at the edge

### Service Selection
- **VPC Endpoints**: Eliminate internet charges
- **AWS PrivateLink**: Secure, cost-effective service access
- **Transit Gateway**: Centralized connectivity

BANDWIDTH_OPT

echo "✅ Bandwidth optimization guide created"

echo ""
echo "3. AUTOMATED COST MONITORING"

# Create cost monitoring script
cat << 'COST_MONITOR' > monitor-network-costs.py
#!/usr/bin/env python3

import boto3
import json
from datetime import datetime, timedelta

def create_cost_budget():
    """Create a budget for network costs"""
    budgets_client = boto3.client('budgets')
    
    account_id = boto3.client('sts').get_caller_identity()['Account']
    
    budget = {
        'BudgetName': 'Hybrid-Network-Monthly-Budget',
        'BudgetLimit': {
            'Amount': '1000.00',  # Adjust as needed
            'Unit': 'USD'
        },
        'TimeUnit': 'MONTHLY',
        'BudgetType': 'COST',
        'CostFilters': {
            'Service': [
                'Amazon Virtual Private Cloud',
                'AWS Direct Connect',
                'Amazon Elastic Compute Cloud - Compute'
            ]
        }
    }
    
    notifications = [
        {
            'Notification': {
                'NotificationType': 'ACTUAL',
                'ComparisonOperator': 'GREATER_THAN',
                'Threshold': 80.0,
                'ThresholdType': 'PERCENTAGE'
            },
            'Subscribers': [
                {
                    'SubscriptionType': 'EMAIL',
                    'Address': 'admin@example.com'  # Update with real email
                }
            ]
        }
    ]
    
    try:
        response = budgets_client.create_budget(
            AccountId=account_id,
            Budget=budget,
            NotificationsWithSubscribers=notifications
        )
        print("Budget created successfully")
        return response
    except Exception as e:
        print(f"Error creating budget: {e}")
        return None

def main():
    print("Setting up cost monitoring...")
    create_cost_budget()
    print("Cost budget created with 80% threshold alert")

if __name__ == "__main__":
    main()

COST_MONITOR

chmod +x monitor-network-costs.py

echo "✅ Cost monitoring script created"

echo ""
echo "COST OPTIMIZATION ANALYSIS COMPLETED"
echo ""
echo "Available tools:"
echo "1. analyze-network-costs.py - Analyze current costs"
echo "2. bandwidth-optimization.md - Optimization strategies"
echo "3. monitor-network-costs.py - Set up cost monitoring"
echo ""
echo "Run: python3 analyze-network-costs.py for current cost analysis"

EOF

chmod +x optimize-hybrid-costs.sh
./optimize-hybrid-costs.sh
```

## Performance Optimization Summary

### Key Performance Optimization Areas

#### 1. Bandwidth Management
- **QoS Implementation**: Prioritize critical traffic
- **Traffic Shaping**: Control bandwidth allocation
- **Compression**: Reduce data transfer volumes
- **Caching**: Minimize redundant transfers

#### 2. Path Optimization
- **BGP Tuning**: Optimize route selection
- **ECMP**: Use multiple paths for load distribution
- **Traffic Engineering**: Direct traffic based on requirements
- **Failover Optimization**: Fast convergence for backup paths

#### 3. Protocol Optimization
- **TCP Tuning**: Optimize for high-bandwidth, high-latency links
- **IPSec Optimization**: Use efficient encryption algorithms
- **MTU Optimization**: Reduce fragmentation
- **Keep-alive Tuning**: Optimize connection persistence

### Monitoring and Alerting Best Practices

#### Key Metrics to Monitor
- **Latency**: Round-trip time measurements
- **Throughput**: Actual vs available bandwidth utilization
- **Packet Loss**: Error rates and retransmissions
- **Availability**: Connection uptime and failover events
- **Cost**: Data transfer costs and budget alerts

#### Alerting Thresholds
- **Latency**: > 100ms for critical applications
- **Packet Loss**: > 0.1% for production traffic
- **Utilization**: > 80% of available bandwidth
- **Error Rate**: > 0.01% for connection errors

### Cost Optimization Strategies

#### Connection Selection
- **Direct Connect**: For consistent high-volume traffic (>1TB/month)
- **VPN**: For variable or low-volume traffic
- **Hybrid Approach**: Primary/backup with cost-based routing

#### Data Transfer Optimization
- **Regional Strategy**: Keep data close to consumers
- **VPC Endpoints**: Eliminate internet data transfer charges
- **CloudFront**: Optimize global content delivery
- **Compression**: Reduce transfer volumes

## Key Takeaways
- Performance optimization requires continuous monitoring and tuning
- BGP optimization enables intelligent traffic path selection
- Cost optimization balances performance requirements with budget constraints
- Automated monitoring and alerting are essential for production environments
- Regular performance assessments identify optimization opportunities
- Multi-path architectures provide both performance and resilience benefits

## Additional Resources
- [AWS Network Performance Best Practices](https://docs.aws.amazon.com/whitepapers/latest/aws-vpc-connectivity-options/network-performance.html)
- [BGP Optimization Guide](https://docs.aws.amazon.com/directconnect/latest/UserGuide/routing-and-bgp.html)
- [AWS Cost Optimization](https://aws.amazon.com/aws-cost-management/aws-cost-optimization/)

## Exam Tips
- Understand the trade-offs between cost, performance, and reliability
- Know when to use Direct Connect vs VPN based on traffic patterns
- BGP path selection follows: Weight > Local Preference > AS Path > MED
- VPC Endpoints eliminate data transfer charges for AWS services
- CloudWatch custom metrics enable detailed performance monitoring
- Cost budgets and alerts prevent unexpected charges
- Quality of Service (QoS) prioritizes critical traffic flows
- TCP window scaling improves performance on high-latency links