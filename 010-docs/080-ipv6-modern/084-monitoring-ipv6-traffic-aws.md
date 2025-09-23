# Topic 84: Monitoring IPv6 Traffic on AWS

## Overview

Effective monitoring of IPv6 traffic is crucial for maintaining network performance, security, and operational visibility. AWS provides comprehensive monitoring tools and services that support IPv6, enabling organizations to track traffic patterns, identify issues, and optimize network performance. This topic covers IPv6-specific monitoring strategies, tools, and implementation patterns.

## IPv6 Monitoring Fundamentals

### Key Differences from IPv4 Monitoring

| Aspect | IPv4 Monitoring | IPv6 Monitoring | Considerations |
|--------|-----------------|-----------------|----------------|
| Address Space | 32-bit addresses | 128-bit addresses | Larger log entries, different parsing |
| Traffic Patterns | NAT-concentrated | Direct end-to-end | More distributed traffic flows |
| Log Formats | Standard formats | Extended IPv6 formats | Tool compatibility required |
| Visualization | Established tools | IPv6-aware tools needed | Display and analysis differences |
| Aggregation | Easy summarization | Complex prefix handling | Hierarchical analysis required |

### IPv6-Specific Monitoring Challenges

1. **Address Representation**: Multiple valid formats for same address
2. **Privacy Extensions**: Temporary addresses complicate tracking
3. **Dual-Stack Correlation**: Linking IPv4 and IPv6 traffic from same source
4. **Tooling Gaps**: Legacy tools may not support IPv6 properly
5. **Scale Considerations**: Larger address space affects indexing and storage

## AWS Native Monitoring Services

### VPC Flow Logs for IPv6

```bash
#!/bin/bash
# Configure VPC Flow Logs for comprehensive IPv6 monitoring

configure_ipv6_flow_logs() {
    local vpc_id=$1
    local log_destination=$2

    echo "Configuring IPv6-aware VPC Flow Logs for VPC: $vpc_id"

    # Create CloudWatch Log Group
    aws logs create-log-group \
        --log-group-name "/aws/vpc/flowlogs/ipv6-monitoring" \
        --retention-in-days 30

    # Create IAM role for Flow Logs
    cat > flow-logs-trust-policy.json << EOF
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "vpc-flow-logs.amazonaws.com"
            },
            "Action": "sts:AssumeRole"
        }
    ]
}
EOF

    ROLE_ARN=$(aws iam create-role \
        --role-name VPCFlowLogsRole \
        --assume-role-policy-document file://flow-logs-trust-policy.json \
        --query 'Role.Arn' \
        --output text)

    # Attach policy for CloudWatch Logs access
    aws iam attach-role-policy \
        --role-name VPCFlowLogsRole \
        --policy-arn arn:aws:iam::aws:policy/service-role/VPCFlowLogsDeliveryRolePolicy

    # Create Flow Logs with custom format for IPv6
    aws ec2 create-flow-logs \
        --resource-type VPC \
        --resource-ids $vpc_id \
        --traffic-type ALL \
        --log-destination-type cloud-watch-logs \
        --log-group-name "/aws/vpc/flowlogs/ipv6-monitoring" \
        --deliver-logs-permission-arn $ROLE_ARN \
        --log-format '${version} ${account-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${windowstart} ${windowend} ${action} ${flowlogstatus} ${vpc-id} ${subnet-id} ${instance-id} ${tcp-flags} ${type} ${pkt-srcaddr} ${pkt-dstaddr} ${region} ${az-id} ${sublocation-type} ${sublocation-id} ${pkt-src-aws-service} ${pkt-dst-aws-service} ${flow-direction} ${traffic-path}'

    echo "Flow Logs created with enhanced IPv6 monitoring format"
    rm flow-logs-trust-policy.json
}

# Usage
configure_ipv6_flow_logs "vpc-12345678" "cloudwatch"
```

### CloudWatch Metrics for IPv6

```python
#!/usr/bin/env python3
# Custom CloudWatch metrics for IPv6 traffic monitoring

import boto3
import json
import ipaddress
from datetime import datetime, timedelta
from collections import defaultdict

class IPv6TrafficMonitor:
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        self.logs = boto3.client('logs')
        self.ec2 = boto3.client('ec2')

    def parse_flow_logs(self, log_group_name: str, hours: int = 1) -> dict:
        """Parse VPC Flow Logs for IPv6 traffic analysis"""
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(hours=hours)

        query = """
        fields @timestamp, srcaddr, dstaddr, srcport, dstport, protocol, packets, bytes, action
        | filter srcaddr like /^2/ or dstaddr like /^2/
        | stats sum(bytes) as total_bytes, sum(packets) as total_packets, count() as flow_count by srcaddr, dstaddr, action
        | sort total_bytes desc
        | limit 1000
        """

        response = self.logs.start_query(
            logGroupName=log_group_name,
            startTime=int(start_time.timestamp()),
            endTime=int(end_time.timestamp()),
            queryString=query
        )

        return response['queryId']

    def analyze_ipv6_patterns(self, results: list) -> dict:
        """Analyze IPv6 traffic patterns"""
        stats = {
            'total_flows': 0,
            'total_bytes': 0,
            'total_packets': 0,
            'top_sources': defaultdict(int),
            'top_destinations': defaultdict(int),
            'protocol_distribution': defaultdict(int),
            'action_counts': defaultdict(int),
            'geographic_distribution': defaultdict(int)
        }

        for result in results:
            stats['total_flows'] += 1

            bytes_count = int(result.get('total_bytes', 0))
            packets_count = int(result.get('total_packets', 0))

            stats['total_bytes'] += bytes_count
            stats['total_packets'] += packets_count

            srcaddr = result.get('srcaddr', '')
            dstaddr = result.get('dstaddr', '')
            action = result.get('action', 'UNKNOWN')

            # Track top sources and destinations
            if self.is_ipv6(srcaddr):
                stats['top_sources'][srcaddr] += bytes_count
            if self.is_ipv6(dstaddr):
                stats['top_destinations'][dstaddr] += bytes_count

            stats['action_counts'][action] += 1

        return stats

    def is_ipv6(self, addr: str) -> bool:
        """Check if address is IPv6"""
        try:
            ip = ipaddress.ip_address(addr)
            return ip.version == 6
        except:
            return False

    def publish_ipv6_metrics(self, stats: dict):
        """Publish IPv6 traffic metrics to CloudWatch"""
        timestamp = datetime.utcnow()

        metrics = [
            {
                'MetricName': 'IPv6TotalBytes',
                'Value': stats['total_bytes'],
                'Unit': 'Bytes',
                'Timestamp': timestamp
            },
            {
                'MetricName': 'IPv6TotalPackets',
                'Value': stats['total_packets'],
                'Unit': 'Count',
                'Timestamp': timestamp
            },
            {
                'MetricName': 'IPv6TotalFlows',
                'Value': stats['total_flows'],
                'Unit': 'Count',
                'Timestamp': timestamp
            },
            {
                'MetricName': 'IPv6AcceptedFlows',
                'Value': stats['action_counts'].get('ACCEPT', 0),
                'Unit': 'Count',
                'Timestamp': timestamp
            },
            {
                'MetricName': 'IPv6RejectedFlows',
                'Value': stats['action_counts'].get('REJECT', 0),
                'Unit': 'Count',
                'Timestamp': timestamp
            }
        ]

        # Calculate IPv6 adoption percentage
        total_internet_flows = stats['total_flows']
        if total_internet_flows > 0:
            ipv6_percentage = (stats['total_flows'] / total_internet_flows) * 100
            metrics.append({
                'MetricName': 'IPv6AdoptionPercentage',
                'Value': ipv6_percentage,
                'Unit': 'Percent',
                'Timestamp': timestamp
            })

        self.cloudwatch.put_metric_data(
            Namespace='AWS/VPC/IPv6',
            MetricData=metrics
        )

        print(f"Published {len(metrics)} IPv6 metrics to CloudWatch")

    def create_ipv6_dashboard(self, dashboard_name: str = "IPv6-Traffic-Monitoring"):
        """Create CloudWatch dashboard for IPv6 monitoring"""
        dashboard_body = {
            "widgets": [
                {
                    "type": "metric",
                    "x": 0,
                    "y": 0,
                    "width": 12,
                    "height": 6,
                    "properties": {
                        "metrics": [
                            ["AWS/VPC/IPv6", "IPv6TotalBytes"],
                            [".", "IPv6TotalPackets"]
                        ],
                        "period": 300,
                        "stat": "Sum",
                        "region": "us-east-1",
                        "title": "IPv6 Traffic Volume",
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
                            ["AWS/VPC/IPv6", "IPv6AcceptedFlows"],
                            [".", "IPv6RejectedFlows"]
                        ],
                        "period": 300,
                        "stat": "Sum",
                        "region": "us-east-1",
                        "title": "IPv6 Flow Status",
                        "view": "timeSeries",
                        "stacked": False
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
                            ["AWS/VPC/IPv6", "IPv6AdoptionPercentage"]
                        ],
                        "period": 300,
                        "stat": "Average",
                        "region": "us-east-1",
                        "title": "IPv6 Adoption Rate",
                        "yAxis": {
                            "left": {
                                "min": 0,
                                "max": 100
                            }
                        }
                    }
                }
            ]
        }

        self.cloudwatch.put_dashboard(
            DashboardName=dashboard_name,
            DashboardBody=json.dumps(dashboard_body)
        )

        print(f"Created IPv6 monitoring dashboard: {dashboard_name}")

    def setup_ipv6_alarms(self):
        """Set up CloudWatch alarms for IPv6 traffic monitoring"""

        # High IPv6 traffic alarm
        self.cloudwatch.put_metric_alarm(
            AlarmName='High-IPv6-Traffic',
            ComparisonOperator='GreaterThanThreshold',
            EvaluationPeriods=2,
            MetricName='IPv6TotalBytes',
            Namespace='AWS/VPC/IPv6',
            Period=300,
            Statistic='Sum',
            Threshold=1000000000.0,  # 1GB
            ActionsEnabled=True,
            AlarmDescription='High IPv6 traffic detected',
            Unit='Bytes'
        )

        # High IPv6 rejection rate alarm
        self.cloudwatch.put_metric_alarm(
            AlarmName='High-IPv6-Rejection-Rate',
            ComparisonOperator='GreaterThanThreshold',
            EvaluationPeriods=3,
            MetricName='IPv6RejectedFlows',
            Namespace='AWS/VPC/IPv6',
            Period=300,
            Statistic='Sum',
            Threshold=1000.0,
            ActionsEnabled=True,
            AlarmDescription='High IPv6 rejection rate detected'
        )

        # Low IPv6 adoption alarm
        self.cloudwatch.put_metric_alarm(
            AlarmName='Low-IPv6-Adoption',
            ComparisonOperator='LessThanThreshold',
            EvaluationPeriods=5,
            MetricName='IPv6AdoptionPercentage',
            Namespace='AWS/VPC/IPv6',
            Period=3600,
            Statistic='Average',
            Threshold=10.0,
            ActionsEnabled=True,
            AlarmDescription='IPv6 adoption rate below threshold'
        )

        print("IPv6 monitoring alarms configured")

# Usage example
def main():
    monitor = IPv6TrafficMonitor()

    # Set up monitoring infrastructure
    monitor.create_ipv6_dashboard()
    monitor.setup_ipv6_alarms()

    # Analyze recent traffic (this would typically run on schedule)
    query_id = monitor.parse_flow_logs("/aws/vpc/flowlogs/ipv6-monitoring", hours=1)
    print(f"Started flow log analysis query: {query_id}")

if __name__ == "__main__":
    main()
```

### Application Load Balancer Monitoring

```bash
#!/bin/bash
# Monitor IPv6 traffic through Application Load Balancer

monitor_alb_ipv6() {
    local alb_name=$1

    echo "Monitoring IPv6 traffic for ALB: $alb_name"
    echo "========================================="

    # Get ALB ARN
    ALB_ARN=$(aws elbv2 describe-load-balancers \
        --names $alb_name \
        --query 'LoadBalancers[0].LoadBalancerArn' \
        --output text)

    # Check if ALB supports dual-stack
    IP_ADDRESS_TYPE=$(aws elbv2 describe-load-balancers \
        --load-balancer-arns $ALB_ARN \
        --query 'LoadBalancers[0].IpAddressType' \
        --output text)

    echo "ALB IP Address Type: $IP_ADDRESS_TYPE"

    if [ "$IP_ADDRESS_TYPE" = "dualstack" ]; then
        echo "✅ ALB configured for dual-stack (IPv4 + IPv6)"
    else
        echo "⚠️  ALB not configured for IPv6 (current: $IP_ADDRESS_TYPE)"
    fi

    # Create custom metrics for IPv6 vs IPv4 traffic
    cat << EOF
Custom CloudWatch Metrics to Create:

1. IPv6 Request Percentage:
   - Parse ALB access logs
   - Count requests by IP version
   - Calculate IPv6 adoption rate

2. IPv6 Response Time Comparison:
   - Monitor response times by IP version
   - Compare IPv6 vs IPv4 performance
   - Identify performance differences

3. IPv6 Error Rate Analysis:
   - Track HTTP errors by IP version
   - Monitor IPv6-specific issues
   - Alert on IPv6 availability problems

Sample CloudWatch Query for ALB Logs:
fields @timestamp, client_ip, request_processing_time, response_processing_time, elb_status_code
| filter client_ip like /^2/
| stats avg(request_processing_time) as avg_request_time,
        avg(response_processing_time) as avg_response_time,
        count() as ipv6_requests by bin(5m)
EOF

    # Enable ALB access logs for detailed analysis
    S3_BUCKET="my-alb-logs-bucket"
    aws elbv2 modify-load-balancer-attributes \
        --load-balancer-arn $ALB_ARN \
        --attributes Key=access_logs.s3.enabled,Value=true \
                    Key=access_logs.s3.bucket,Value=$S3_BUCKET \
                    Key=access_logs.s3.prefix,Value="ipv6-monitoring"

    echo "ALB access logs enabled for IPv6 monitoring"
}

# Usage
monitor_alb_ipv6 "my-dualstack-alb"
```

## Advanced Monitoring Implementations

### Custom IPv6 Traffic Analyzer

```python
#!/usr/bin/env python3
# Advanced IPv6 traffic analysis and reporting

import boto3
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
import ipaddress
from datetime import datetime, timedelta
import re
import gzip
import json

class IPv6TrafficAnalyzer:
    def __init__(self):
        self.s3 = boto3.client('s3')
        self.cloudwatch = boto3.client('cloudwatch')
        self.ec2 = boto3.client('ec2')

    def analyze_alb_logs(self, bucket_name: str, prefix: str, hours: int = 24) -> pd.DataFrame:
        """Analyze ALB access logs for IPv6 traffic patterns"""

        # List log files from the last N hours
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(hours=hours)

        log_files = []
        paginator = self.s3.get_paginator('list_objects_v2')

        for page in paginator.paginate(Bucket=bucket_name, Prefix=prefix):
            if 'Contents' in page:
                for obj in page['Contents']:
                    obj_time = obj['LastModified'].replace(tzinfo=None)
                    if start_time <= obj_time <= end_time:
                        log_files.append(obj['Key'])

        # Parse log files
        all_records = []
        for log_file in log_files[:50]:  # Limit for demo
            try:
                response = self.s3.get_object(Bucket=bucket_name, Key=log_file)

                # Handle gzipped logs
                if log_file.endswith('.gz'):
                    content = gzip.decompress(response['Body'].read()).decode('utf-8')
                else:
                    content = response['Body'].read().decode('utf-8')

                records = self.parse_alb_log_lines(content)
                all_records.extend(records)

            except Exception as e:
                print(f"Error processing {log_file}: {e}")

        return pd.DataFrame(all_records)

    def parse_alb_log_lines(self, content: str) -> list:
        """Parse ALB log lines into structured data"""
        records = []

        # ALB log format regex (simplified)
        log_pattern = re.compile(
            r'(\S+) (\S+) (\S+) (\S+):(\d+) (\S+):(\d+) '
            r'(\S+) (\S+) (\S+) (\d+) (\d+) (\d+) (\d+) '
            r'"(\S+) (\S+) (\S+)" "([^"]*)" (\S+) (\S+) '
            r'(\S+) "([^"]*)" "([^"]*)" (\S+) (\S+) (\S+) (\S+)'
        )

        for line in content.strip().split('\\n'):
            if not line:
                continue

            match = log_pattern.match(line)
            if match:
                groups = match.groups()

                record = {
                    'timestamp': groups[0],
                    'elb_name': groups[1],
                    'client_ip': groups[2],
                    'client_port': groups[3],
                    'target_ip': groups[5],
                    'target_port': groups[6],
                    'request_processing_time': float(groups[7]) if groups[7] != '-1' else 0,
                    'target_processing_time': float(groups[8]) if groups[8] != '-1' else 0,
                    'response_processing_time': float(groups[9]) if groups[9] != '-1' else 0,
                    'elb_status_code': int(groups[10]),
                    'target_status_code': groups[11],
                    'received_bytes': int(groups[12]),
                    'sent_bytes': int(groups[13]),
                    'request_verb': groups[14],
                    'request_url': groups[15],
                    'request_proto': groups[16],
                    'user_agent': groups[17],
                    'ssl_cipher': groups[18],
                    'ssl_protocol': groups[19]
                }

                # Determine IP version
                try:
                    ip = ipaddress.ip_address(record['client_ip'])
                    record['ip_version'] = ip.version
                    record['is_ipv6'] = ip.version == 6

                    if ip.version == 6:
                        record['ipv6_type'] = self.classify_ipv6_address(ip)
                except:
                    record['ip_version'] = 0
                    record['is_ipv6'] = False

                records.append(record)

        return records

    def classify_ipv6_address(self, ip: ipaddress.IPv6Address) -> str:
        """Classify IPv6 address type"""
        if ip.is_loopback:
            return 'loopback'
        elif ip.is_link_local:
            return 'link_local'
        elif ip.is_site_local:
            return 'site_local'
        elif ip.is_multicast:
            return 'multicast'
        elif ip.is_private:
            return 'private'
        elif ip.is_global:
            return 'global'
        else:
            return 'other'

    def generate_ipv6_report(self, df: pd.DataFrame) -> dict:
        """Generate comprehensive IPv6 traffic report"""

        if df.empty:
            return {'error': 'No data available for analysis'}

        report = {
            'summary': {},
            'performance': {},
            'geographic': {},
            'security': {},
            'trends': {}
        }

        # Summary statistics
        total_requests = len(df)
        ipv6_requests = len(df[df['is_ipv6'] == True])
        ipv4_requests = total_requests - ipv6_requests

        report['summary'] = {
            'total_requests': total_requests,
            'ipv6_requests': ipv6_requests,
            'ipv4_requests': ipv4_requests,
            'ipv6_percentage': (ipv6_requests / total_requests * 100) if total_requests > 0 else 0,
            'time_range': {
                'start': df['timestamp'].min() if 'timestamp' in df else 'N/A',
                'end': df['timestamp'].max() if 'timestamp' in df else 'N/A'
            }
        }

        # Performance comparison
        if ipv6_requests > 0 and ipv4_requests > 0:
            ipv6_data = df[df['is_ipv6'] == True]
            ipv4_data = df[df['is_ipv6'] == False]

            report['performance'] = {
                'ipv6_avg_response_time': ipv6_data['target_processing_time'].mean(),
                'ipv4_avg_response_time': ipv4_data['target_processing_time'].mean(),
                'ipv6_avg_request_size': ipv6_data['received_bytes'].mean(),
                'ipv4_avg_request_size': ipv4_data['received_bytes'].mean(),
                'ipv6_avg_response_size': ipv6_data['sent_bytes'].mean(),
                'ipv4_avg_response_size': ipv4_data['sent_bytes'].mean(),
                'ipv6_error_rate': len(ipv6_data[ipv6_data['elb_status_code'] >= 400]) / len(ipv6_data) * 100,
                'ipv4_error_rate': len(ipv4_data[ipv4_data['elb_status_code'] >= 400]) / len(ipv4_data) * 100
            }

        # IPv6 address type distribution
        if ipv6_requests > 0:
            ipv6_df = df[df['is_ipv6'] == True]
            if 'ipv6_type' in ipv6_df.columns:
                ipv6_types = ipv6_df['ipv6_type'].value_counts().to_dict()
                report['ipv6_address_types'] = ipv6_types

        # Top IPv6 clients
        if ipv6_requests > 0:
            top_ipv6_clients = df[df['is_ipv6'] == True]['client_ip'].value_counts().head(10).to_dict()
            report['top_ipv6_clients'] = top_ipv6_clients

        return report

    def create_visualizations(self, df: pd.DataFrame, output_dir: str = './ipv6_reports/'):
        """Create visualization charts for IPv6 traffic analysis"""

        import os
        os.makedirs(output_dir, exist_ok=True)

        plt.style.use('seaborn')

        # 1. IPv6 vs IPv4 traffic distribution
        plt.figure(figsize=(10, 6))
        ip_version_counts = df['ip_version'].value_counts()
        labels = ['IPv4', 'IPv6'] if 4 in ip_version_counts.index and 6 in ip_version_counts.index else ['IPv4'] if 4 in ip_version_counts.index else ['IPv6']
        sizes = [ip_version_counts.get(4, 0), ip_version_counts.get(6, 0)]
        sizes = [s for s in sizes if s > 0]

        plt.pie(sizes, labels=labels, autopct='%1.1f%%', startangle=90)
        plt.title('IPv4 vs IPv6 Traffic Distribution')
        plt.savefig(f'{output_dir}/ipv4_vs_ipv6_distribution.png')
        plt.close()

        # 2. Response time comparison
        if len(df[df['is_ipv6'] == True]) > 0 and len(df[df['is_ipv6'] == False]) > 0:
            plt.figure(figsize=(12, 6))

            ipv6_times = df[df['is_ipv6'] == True]['target_processing_time']
            ipv4_times = df[df['is_ipv6'] == False]['target_processing_time']

            plt.hist([ipv4_times, ipv6_times], bins=50, alpha=0.7, label=['IPv4', 'IPv6'])
            plt.xlabel('Response Time (seconds)')
            plt.ylabel('Frequency')
            plt.title('Response Time Distribution: IPv4 vs IPv6')
            plt.legend()
            plt.savefig(f'{output_dir}/response_time_comparison.png')
            plt.close()

        # 3. Traffic over time
        if 'timestamp' in df.columns:
            df['timestamp'] = pd.to_datetime(df['timestamp'])
            df['hour'] = df['timestamp'].dt.floor('H')

            hourly_traffic = df.groupby(['hour', 'is_ipv6']).size().unstack(fill_value=0)

            plt.figure(figsize=(15, 6))
            if True in hourly_traffic.columns:
                plt.plot(hourly_traffic.index, hourly_traffic[True], label='IPv6', marker='o')
            if False in hourly_traffic.columns:
                plt.plot(hourly_traffic.index, hourly_traffic[False], label='IPv4', marker='s')

            plt.xlabel('Time')
            plt.ylabel('Request Count')
            plt.title('Traffic Trends Over Time')
            plt.legend()
            plt.xticks(rotation=45)
            plt.tight_layout()
            plt.savefig(f'{output_dir}/traffic_trends.png')
            plt.close()

        print(f"Visualizations saved to {output_dir}")

# Usage example
def main():
    analyzer = IPv6TrafficAnalyzer()

    # Analyze ALB logs
    bucket_name = "my-alb-logs-bucket"
    prefix = "ipv6-monitoring/"

    try:
        df = analyzer.analyze_alb_logs(bucket_name, prefix, hours=24)

        if not df.empty:
            report = analyzer.generate_ipv6_report(df)

            print("IPv6 Traffic Analysis Report")
            print("=" * 40)
            print(json.dumps(report, indent=2, default=str))

            analyzer.create_visualizations(df)
        else:
            print("No log data found for analysis")

    except Exception as e:
        print(f"Error during analysis: {e}")

if __name__ == "__main__":
    main()
```

### Real-time IPv6 Monitoring

```bash
#!/bin/bash
# Real-time IPv6 traffic monitoring script

setup_realtime_monitoring() {
    echo "Setting up real-time IPv6 monitoring"
    echo "===================================="

    # Install monitoring tools
    if ! command -v tcpdump &> /dev/null; then
        echo "Installing tcpdump..."
        sudo apt-get update && sudo apt-get install -y tcpdump
    fi

    if ! command -v nfcapd &> /dev/null; then
        echo "Installing nfcapd for NetFlow monitoring..."
        sudo apt-get install -y nfcapd
    fi

    # Create monitoring directory
    sudo mkdir -p /var/log/ipv6-monitoring
    sudo chmod 755 /var/log/ipv6-monitoring

    cat << 'EOF' > /usr/local/bin/ipv6-monitor.sh
#!/bin/bash
# Real-time IPv6 traffic monitoring

LOG_DIR="/var/log/ipv6-monitoring"
INTERFACE="eth0"

monitor_ipv6_traffic() {
    echo "Starting IPv6 traffic monitoring on $INTERFACE"

    # Monitor IPv6 traffic with tcpdump
    sudo tcpdump -i $INTERFACE -n ip6 -c 1000 -w $LOG_DIR/ipv6-$(date +%Y%m%d-%H%M%S).pcap &
    TCPDUMP_PID=$!

    # Real-time analysis
    sudo tcpdump -i $INTERFACE -n ip6 -l | while read line; do
        timestamp=$(date '+%Y-%m-%d %H:%M:%S')
        echo "[$timestamp] $line" >> $LOG_DIR/ipv6-realtime.log

        # Extract source and destination
        src=$(echo "$line" | grep -oE '2[0-9a-f:]+' | head -1)
        dst=$(echo "$line" | grep -oE '2[0-9a-f:]+' | tail -1)

        if [ ! -z "$src" ] && [ ! -z "$dst" ]; then
            # Send to CloudWatch (requires AWS CLI and proper IAM permissions)
            aws cloudwatch put-metric-data \
                --namespace "Custom/IPv6/RealTime" \
                --metric-data MetricName=IPv6Connection,Value=1,Unit=Count,Timestamp=$(date -u +%Y-%m-%dT%H:%M:%S) \
                2>/dev/null || true
        fi
    done

    # Clean up
    kill $TCPDUMP_PID 2>/dev/null || true
}

# Function to analyze captured traffic
analyze_captured_traffic() {
    local pcap_file=$1

    if [ ! -f "$pcap_file" ]; then
        echo "PCAP file not found: $pcap_file"
        return 1
    fi

    echo "Analyzing IPv6 traffic in $pcap_file"
    echo "==================================="

    # Basic statistics
    echo "Total IPv6 packets:"
    tcpdump -r $pcap_file -n ip6 | wc -l

    echo ""
    echo "Top 10 IPv6 source addresses:"
    tcpdump -r $pcap_file -n ip6 2>/dev/null | \
        grep -oE 'IP6 2[0-9a-f:]+' | \
        sed 's/IP6 //' | \
        sort | uniq -c | sort -nr | head -10

    echo ""
    echo "Top 10 IPv6 destination addresses:"
    tcpdump -r $pcap_file -n ip6 2>/dev/null | \
        grep -oE '> 2[0-9a-f:]+' | \
        sed 's/> //' | \
        sort | uniq -c | sort -nr | head -10

    echo ""
    echo "Protocol distribution:"
    tcpdump -r $pcap_file -n ip6 2>/dev/null | \
        grep -oE '(TCP|UDP|ICMPv6)' | \
        sort | uniq -c | sort -nr
}

# Start monitoring
monitor_ipv6_traffic
EOF

    chmod +x /usr/local/bin/ipv6-monitor.sh

    # Create systemd service for continuous monitoring
    cat << EOF > /etc/systemd/system/ipv6-monitor.service
[Unit]
Description=IPv6 Traffic Monitor
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/ipv6-monitor.sh
Restart=always
RestartSec=10
User=root

[Install]
WantedBy=multi-user.target
EOF

    sudo systemctl daemon-reload
    sudo systemctl enable ipv6-monitor.service

    echo "Real-time IPv6 monitoring setup complete"
    echo "Start with: sudo systemctl start ipv6-monitor.service"
}

# Performance monitoring function
monitor_ipv6_performance() {
    echo "IPv6 Performance Monitoring"
    echo "==========================="

    # Network interface statistics
    echo "Network Interface IPv6 Statistics:"
    ip -6 route show | head -10

    echo ""
    echo "IPv6 neighbor table:"
    ip -6 neigh show | head -10

    echo ""
    echo "IPv6 socket statistics:"
    ss -6 -tuln | head -10

    # System performance impact
    echo ""
    echo "System Performance Metrics:"

    # CPU usage for IPv6 processing
    top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print "CPU Usage: " 100 - $1 "%"}'

    # Memory usage
    free -h | grep "Mem:" | awk '{print "Memory Usage: " $3 "/" $2}'

    # Network interface throughput
    cat /proc/net/dev | grep eth0 | awk '{print "RX Bytes: " $2 ", TX Bytes: " $10}'
}

# Usage
setup_realtime_monitoring
monitor_ipv6_performance
```

## Security Monitoring for IPv6

### IPv6 Security Event Detection

```python
#!/usr/bin/env python3
# IPv6 security monitoring and threat detection

import boto3
import json
import ipaddress
from datetime import datetime, timedelta
import re
from collections import defaultdict, deque

class IPv6SecurityMonitor:
    def __init__(self):
        self.cloudwatch = boto3.client('cloudwatch')
        self.logs = boto3.client('logs')
        self.sns = boto3.client('sns')

        # Security patterns to detect
        self.attack_patterns = {
            'port_scan': re.compile(r'rapid connections from same source'),
            'address_scan': re.compile(r'sequential IPv6 addresses'),
            'dos_attempt': re.compile(r'high volume from single source'),
            'suspicious_prefix': re.compile(r'(fc00|fe80|ff00):')
        }

        # Tracking state
        self.connection_tracker = defaultdict(lambda: deque(maxlen=100))
        self.alert_threshold = {
            'connections_per_minute': 100,
            'unique_ports_per_source': 50,
            'bytes_per_minute': 10000000  # 10MB
        }

    def analyze_security_events(self, flow_logs: list) -> list:
        """Analyze flow logs for IPv6 security events"""
        security_events = []

        for log_entry in flow_logs:
            try:
                # Parse log entry
                fields = log_entry.split()
                if len(fields) < 14:
                    continue

                srcaddr = fields[3]
                dstaddr = fields[4]
                srcport = fields[5]
                dstport = fields[6]
                protocol = fields[7]
                packets = int(fields[8])
                bytes_count = int(fields[9])
                action = fields[12]

                # Check if source is IPv6
                if not self.is_ipv6_address(srcaddr):
                    continue

                # Detect potential security issues
                events = self.detect_security_threats(
                    srcaddr, dstaddr, srcport, dstport,
                    protocol, packets, bytes_count, action
                )

                security_events.extend(events)

            except Exception as e:
                print(f"Error analyzing log entry: {e}")
                continue

        return security_events

    def is_ipv6_address(self, addr: str) -> bool:
        """Check if address is IPv6"""
        try:
            ip = ipaddress.ip_address(addr)
            return ip.version == 6
        except:
            return False

    def detect_security_threats(self, srcaddr: str, dstaddr: str, srcport: str,
                              dstport: str, protocol: str, packets: int,
                              bytes_count: int, action: str) -> list:
        """Detect various IPv6 security threats"""
        events = []
        timestamp = datetime.utcnow()

        # Track connections from source
        self.connection_tracker[srcaddr].append({
            'timestamp': timestamp,
            'dstaddr': dstaddr,
            'dstport': dstport,
            'protocol': protocol,
            'packets': packets,
            'bytes': bytes_count,
            'action': action
        })

        recent_connections = [
            conn for conn in self.connection_tracker[srcaddr]
            if (timestamp - conn['timestamp']).seconds < 60
        ]

        # 1. Port scanning detection
        if len(recent_connections) > self.alert_threshold['connections_per_minute']:
            unique_ports = len(set(conn['dstport'] for conn in recent_connections))
            if unique_ports > self.alert_threshold['unique_ports_per_source']:
                events.append({
                    'type': 'port_scan',
                    'severity': 'high',
                    'source': srcaddr,
                    'description': f'Potential port scan: {unique_ports} ports in 1 minute',
                    'details': {
                        'connections': len(recent_connections),
                        'unique_ports': unique_ports,
                        'targets': list(set(conn['dstaddr'] for conn in recent_connections))
                    }
                })

        # 2. High volume traffic (potential DDoS)
        total_bytes = sum(conn['bytes'] for conn in recent_connections)
        if total_bytes > self.alert_threshold['bytes_per_minute']:
            events.append({
                'type': 'high_volume',
                'severity': 'medium',
                'source': srcaddr,
                'description': f'High volume traffic: {total_bytes} bytes in 1 minute',
                'details': {
                    'bytes_per_minute': total_bytes,
                    'connections': len(recent_connections)
                }
            })

        # 3. Suspicious IPv6 address patterns
        if self.is_suspicious_ipv6_pattern(srcaddr):
            events.append({
                'type': 'suspicious_address',
                'severity': 'medium',
                'source': srcaddr,
                'description': 'Suspicious IPv6 address pattern',
                'details': {
                    'address_type': self.classify_ipv6_address(srcaddr)
                }
            })

        # 4. Rejected connections analysis
        rejected_count = sum(1 for conn in recent_connections if conn['action'] == 'REJECT')
        if rejected_count > 20:  # High rejection rate
            events.append({
                'type': 'high_rejection_rate',
                'severity': 'low',
                'source': srcaddr,
                'description': f'High rejection rate: {rejected_count} rejected connections',
                'details': {
                    'rejected_connections': rejected_count,
                    'total_connections': len(recent_connections)
                }
            })

        return events

    def is_suspicious_ipv6_pattern(self, addr: str) -> bool:
        """Check for suspicious IPv6 address patterns"""
        try:
            ip = ipaddress.IPv6Address(addr)

            # Check for suspicious patterns
            if ip.is_link_local or ip.is_multicast:
                return True

            # Check for sequential scanning patterns
            addr_int = int(ip)
            if addr_int & 0xFFFF == 0:  # Ends with all zeros
                return True

            # Check for common attack prefixes
            suspicious_prefixes = [
                '2001:db8:',  # Documentation prefix
                'fc00:',      # Unique local
                'fe80:',      # Link local
                'ff00:',      # Multicast
            ]

            for prefix in suspicious_prefixes:
                if addr.startswith(prefix):
                    return True

            return False

        except:
            return False

    def classify_ipv6_address(self, addr: str) -> str:
        """Classify IPv6 address type for security analysis"""
        try:
            ip = ipaddress.IPv6Address(addr)

            if ip.is_loopback:
                return 'loopback'
            elif ip.is_link_local:
                return 'link_local'
            elif ip.is_site_local:
                return 'site_local'
            elif ip.is_multicast:
                return 'multicast'
            elif ip.is_private:
                return 'private'
            elif ip.is_global:
                return 'global'
            else:
                return 'unknown'
        except:
            return 'invalid'

    def send_security_alert(self, events: list, topic_arn: str):
        """Send security alerts via SNS"""
        if not events:
            return

        high_severity_events = [e for e in events if e['severity'] == 'high']

        if high_severity_events:
            message = {
                'alert_type': 'IPv6_Security_Event',
                'timestamp': datetime.utcnow().isoformat(),
                'high_severity_count': len(high_severity_events),
                'total_events': len(events),
                'events': high_severity_events[:5]  # Limit to first 5 events
            }

            try:
                self.sns.publish(
                    TopicArn=topic_arn,
                    Subject='IPv6 Security Alert - High Severity Events Detected',
                    Message=json.dumps(message, indent=2, default=str)
                )
                print(f"Security alert sent for {len(high_severity_events)} high severity events")
            except Exception as e:
                print(f"Error sending security alert: {e}")

    def create_security_dashboard(self, dashboard_name: str = "IPv6-Security-Monitor"):
        """Create security monitoring dashboard"""
        dashboard_body = {
            "widgets": [
                {
                    "type": "metric",
                    "x": 0,
                    "y": 0,
                    "width": 12,
                    "height": 6,
                    "properties": {
                        "metrics": [
                            ["AWS/VPC/IPv6Security", "PortScanAttempts"],
                            [".", "HighVolumeTraffic"],
                            [".", "SuspiciousAddresses"]
                        ],
                        "period": 300,
                        "stat": "Sum",
                        "region": "us-east-1",
                        "title": "IPv6 Security Events"
                    }
                },
                {
                    "type": "log",
                    "x": 0,
                    "y": 6,
                    "width": 24,
                    "height": 6,
                    "properties": {
                        "query": "SOURCE '/aws/vpc/flowlogs' | fields @timestamp, srcaddr, dstaddr, action\n| filter srcaddr like /^2/\n| filter action = \"REJECT\"\n| stats count() by srcaddr\n| sort count desc\n| limit 20",
                        "region": "us-east-1",
                        "title": "Top IPv6 Sources with Rejected Connections",
                        "view": "table"
                    }
                }
            ]
        }

        self.cloudwatch.put_dashboard(
            DashboardName=dashboard_name,
            DashboardBody=json.dumps(dashboard_body)
        )

        print(f"IPv6 security dashboard created: {dashboard_name}")

# Usage example
def main():
    monitor = IPv6SecurityMonitor()

    # Create security monitoring dashboard
    monitor.create_security_dashboard()

    # Example: Analyze sample flow logs for security events
    sample_logs = [
        "2 123456789012 eni-12345678 2001:db8:bad::1 2001:db8:target::1 12345 80 6 10 1000 1627812000 1627812060 ACCEPT OK",
        "2 123456789012 eni-12345678 2001:db8:bad::1 2001:db8:target::1 12346 443 6 5 500 1627812001 1627812061 REJECT OK",
        # Add more sample logs for testing
    ]

    security_events = monitor.analyze_security_events(sample_logs)

    if security_events:
        print(f"Detected {len(security_events)} security events:")
        for event in security_events:
            print(json.dumps(event, indent=2, default=str))

if __name__ == "__main__":
    main()
```

## Operational Best Practices

### IPv6 Monitoring Checklist

```yaml
IPv6_Monitoring_Best_Practices:
  Infrastructure_Monitoring:
    VPC_Flow_Logs:
      - enable_for_all_vpcs: "Monitor all IPv6 traffic"
      - custom_log_format: "Include IPv6-specific fields"
      - appropriate_retention: "Balance cost vs compliance"
      - real_time_analysis: "Use CloudWatch Insights"

    Load_Balancer_Monitoring:
      - enable_access_logs: "Detailed request analysis"
      - dual_stack_metrics: "Compare IPv4/IPv6 performance"
      - health_check_monitoring: "IPv6 endpoint health"
      - geographic_analysis: "IPv6 adoption by region"

  Application_Monitoring:
    Performance_Metrics:
      - response_time_comparison: "IPv6 vs IPv4 latency"
      - throughput_analysis: "Bandwidth utilization"
      - error_rate_tracking: "IPv6-specific errors"
      - connection_success_rate: "Happy Eyeballs effectiveness"

    User_Experience:
      - client_ip_distribution: "IPv6 adoption rates"
      - device_type_analysis: "IPv6 support by device"
      - geographic_distribution: "Regional IPv6 usage"
      - time_based_patterns: "Peak usage analysis"

  Security_Monitoring:
    Threat_Detection:
      - suspicious_traffic_patterns: "Scanning, DoS attempts"
      - source_reputation: "Known bad IPv6 ranges"
      - connection_anomalies: "Unusual traffic flows"
      - protocol_compliance: "IPv6 standard violations"

    Compliance_Monitoring:
      - data_retention_policies: "Log retention requirements"
      - privacy_considerations: "IPv6 address privacy"
      - audit_trail_maintenance: "Change tracking"
      - incident_response_procedures: "IPv6-specific playbooks"

  Operational_Excellence:
    Automation:
      - automated_alerting: "Threshold-based notifications"
      - self_healing_systems: "Automatic remediation"
      - capacity_planning: "Growth projection"
      - cost_optimization: "Monitoring cost management"

    Documentation:
      - monitoring_procedures: "Step-by-step guides"
      - troubleshooting_playbooks: "IPv6-specific issues"
      - escalation_procedures: "Incident response"
      - knowledge_base: "Common issues and solutions"
```

## Conclusion

Effective IPv6 traffic monitoring on AWS requires a comprehensive approach that addresses the unique characteristics of IPv6:

### Key Takeaways

1. **Enhanced Visibility**: IPv6 monitoring provides insights into modern internet traffic patterns
2. **Performance Optimization**: Compare IPv6 vs IPv4 performance to optimize user experience
3. **Security Awareness**: Monitor for IPv6-specific attack patterns and threats
4. **Adoption Tracking**: Measure IPv6 adoption rates and identify optimization opportunities
5. **Operational Excellence**: Implement automated monitoring and alerting for IPv6 infrastructure

### Implementation Recommendations

- **Start with VPC Flow Logs**: Enable comprehensive IPv6 traffic logging
- **Implement Custom Metrics**: Track IPv6-specific KPIs and adoption rates
- **Set Up Automated Alerting**: Monitor for performance and security issues
- **Create Dashboards**: Visualize IPv6 traffic patterns and trends
- **Regular Analysis**: Perform periodic deep-dive analysis of IPv6 traffic
- **Security Focus**: Monitor for IPv6-specific attack patterns
- **Performance Benchmarking**: Compare IPv6 vs IPv4 application performance

The monitoring strategies and tools outlined here provide a solid foundation for maintaining visibility into IPv6 traffic, ensuring optimal performance, and detecting security threats in modern cloud environments.