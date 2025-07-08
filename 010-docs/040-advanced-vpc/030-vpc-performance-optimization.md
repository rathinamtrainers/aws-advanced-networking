# Topic 30: VPC Performance Optimization and Network Acceleration

## Prerequisites
- Completed Topic 29 (VPC Network Security and Advanced Access Control)
- Understanding of network performance concepts and bottlenecks
- Knowledge of AWS compute and storage services
- Familiarity with monitoring and optimization techniques

## Learning Objectives
By the end of this topic, you will be able to:
- Analyze and optimize VPC network performance bottlenecks
- Implement network acceleration techniques and enhanced networking
- Configure placement groups and instance optimization strategies
- Monitor and tune network performance for high-throughput applications

## Theory

### VPC Network Performance Fundamentals

#### Performance Metrics
- **Bandwidth**: Maximum data transfer rate (Gbps)
- **Latency**: Round-trip time for packet delivery (milliseconds)
- **Packet Per Second (PPS)**: Number of packets processed per second
- **Jitter**: Variation in packet delivery times
- **Throughput**: Actual data transfer achieved under load

#### Performance Factors
```
Network Performance Hierarchy:
┌─────────────────────────────────────────────────────┐
│ Application Layer (Protocol efficiency)            │
├─────────────────────────────────────────────────────┤
│ Instance Layer (CPU, Memory, Network)              │
├─────────────────────────────────────────────────────┤
│ Hypervisor Layer (SR-IOV, DPDK)                   │
├─────────────────────────────────────────────────────┤
│ Infrastructure Layer (Placement, AZ)               │
├─────────────────────────────────────────────────────┤
│ Physical Layer (Hardware, Network fabric)          │
└─────────────────────────────────────────────────────┘
```

### Enhanced Networking Technologies

#### SR-IOV (Single Root I/O Virtualization)
- **Purpose**: Hardware-level network virtualization
- **Benefits**: Lower latency, higher PPS, reduced CPU utilization
- **Support**: Available on most modern instance types
- **Implementation**: Enabled by default on supported instances

#### Elastic Network Adapter (ENA)
- **Technology**: High performance network interface
- **Bandwidth**: Up to 100 Gbps on supported instances
- **Features**: Enhanced networking, SR-IOV support
- **Requirements**: ENA-enabled AMIs and instance types

#### DPDK (Data Plane Development Kit)
- **Purpose**: Userspace packet processing
- **Benefits**: Bypass kernel networking stack
- **Use Cases**: High-frequency trading, NFV, packet processing
- **Implementation**: Application-level optimization

### Instance Performance Optimization

#### Instance Type Selection
```
Performance by Instance Family:
┌──────────────┬─────────────┬─────────────┬─────────────┐
│ Family       │ Bandwidth   │ PPS         │ Use Case    │
├──────────────┼─────────────┼─────────────┼─────────────┤
│ C5n          │ 100 Gbps    │ 14M PPS     │ HPC         │
│ M5n/M5dn     │ 100 Gbps    │ 14M PPS     │ General     │
│ R5n/R5dn     │ 100 Gbps    │ 14M PPS     │ Memory      │
│ C5/M5/R5     │ 25 Gbps     │ 6M PPS      │ Standard    │
│ T3/T4g       │ 5 Gbps      │ 1M PPS      │ Burstable   │
└──────────────┴─────────────┴─────────────┴─────────────┘
```

#### Placement Groups
- **Cluster**: Low latency, high bandwidth within single AZ
- **Partition**: Distributed across distinct hardware
- **Spread**: Maximum isolation across hardware

### Network Acceleration Techniques

#### Jumbo Frames (9000 MTU)
- **Standard MTU**: 1500 bytes
- **Jumbo MTU**: 9000 bytes
- **Benefits**: Reduced overhead, higher throughput
- **Requirements**: End-to-end support, same VPC/region

#### TCP Optimization
- **Window Scaling**: Larger TCP windows for high bandwidth
- **Congestion Control**: Optimized algorithms (BBR, CUBIC)
- **Buffer Tuning**: Socket buffer optimization
- **Keep-alive Settings**: Connection persistence optimization

## Lab Exercise: Comprehensive VPC Performance Optimization

### Lab Duration: 360 minutes

### Step 1: Baseline Performance Assessment
**Objective**: Establish current performance metrics and identify bottlenecks

**Performance Baseline Setup**:
```bash
# Create comprehensive performance baseline assessment
cat << 'EOF' > vpc-performance-baseline.sh
#!/bin/bash

echo "=== VPC Performance Baseline Assessment ==="
echo ""

echo "1. NETWORK PERFORMANCE REQUIREMENTS ANALYSIS"
echo "   Application type: Web/Database/HPC/Real-time"
echo "   Expected throughput: _______ Gbps"
echo "   Latency requirements: _______ ms"
echo "   Concurrent connections: _______"
echo "   Geographic distribution: Single/Multi-region"
echo "   Peak traffic patterns: _______ requests/second"
echo ""

echo "2. CURRENT INFRASTRUCTURE ASSESSMENT"
echo "   Instance types in use: _______"
echo "   Enhanced networking enabled: Yes/No"
echo "   Placement groups configured: Yes/No"
echo "   Network optimization applied: Yes/No"
echo "   Monitoring tools deployed: CloudWatch/Third-party"
echo ""

echo "3. PERFORMANCE BOTTLENECK IDENTIFICATION"
echo "   CPU utilization during peak: _______%"
echo "   Network utilization: _______%"
echo "   Memory utilization: _______%"
echo "   Storage I/O performance: _______"
echo "   Application response time: _______ ms"
echo ""

echo "4. OPTIMIZATION OPPORTUNITIES"
echo "   Instance type upgrade needed: Yes/No"
echo "   Enhanced networking benefits: High/Medium/Low"
echo "   Placement group potential: High/Medium/Low"
echo "   Network tuning required: Yes/No"
echo "   Application optimization needed: Yes/No"

EOF

chmod +x vpc-performance-baseline.sh
./vpc-performance-baseline.sh
```

### Step 2: Implement Enhanced Networking
**Objective**: Deploy high-performance networking capabilities

**Enhanced Networking Implementation**:
```bash
# Implement enhanced networking and optimization
cat << 'EOF' > implement-enhanced-networking.sh
#!/bin/bash

# Check if VPC config exists from previous labs
if [ ! -f "advanced-routing-config.env" ]; then
    echo "Error: VPC configuration not found. Please run previous labs first."
    exit 1
fi

source advanced-routing-config.env

echo "=== Implementing Enhanced Networking ==="
echo ""

echo "1. CREATING PERFORMANCE-OPTIMIZED INSTANCES"

# Get latest Amazon Linux 2 AMI with ENA support
AMI_ID=$(aws ec2 describe-images \
    --owners amazon \
    --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
              "Name=state,Values=available" \
              "Name=ena-support,Values=true" \
    --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
    --output text)

echo "Using ENA-enabled AMI: $AMI_ID"

# Create placement group for low latency
PLACEMENT_GROUP_NAME="HighPerf-Cluster-PG"
aws ec2 create-placement-group \
    --group-name $PLACEMENT_GROUP_NAME \
    --strategy cluster \
    --tag-specifications "ResourceType=placement-group,Tags=[{Key=Name,Value=$PLACEMENT_GROUP_NAME},{Key=Purpose,Value=HighPerformance}]"

echo "✅ Placement group created: $PLACEMENT_GROUP_NAME"

# Create security group for performance testing
PERF_SG_ID=$(aws ec2 create-security-group \
    --group-name Performance-Testing-SG \
    --description "Security group for performance testing instances" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=Performance-Testing-SG}]" \
    --query 'GroupId' \
    --output text)

# Configure security group for performance testing
aws ec2 authorize-security-group-ingress \
    --group-id $PERF_SG_ID \
    --protocol tcp \
    --port 22 \
    --cidr 10.0.0.0/16

aws ec2 authorize-security-group-ingress \
    --group-id $PERF_SG_ID \
    --protocol tcp \
    --port 5001 \
    --cidr 10.0.0.0/16

aws ec2 authorize-security-group-ingress \
    --group-id $PERF_SG_ID \
    --protocol udp \
    --port 5001 \
    --cidr 10.0.0.0/16

# Allow all traffic within security group for testing
aws ec2 authorize-security-group-ingress \
    --group-id $PERF_SG_ID \
    --protocol -1 \
    --source-group $PERF_SG_ID

echo "✅ Performance testing security group configured: $PERF_SG_ID"

echo ""
echo "2. LAUNCHING HIGH-PERFORMANCE INSTANCES"

# Launch source instance (C5n.large for high performance)
SOURCE_INSTANCE=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type c5n.large \
    --key-name my-key-pair \
    --security-group-ids $PERF_SG_ID \
    --subnet-id $APP_SUBNET_A \
    --placement "GroupName=$PLACEMENT_GROUP_NAME" \
    --ena-support \
    --user-data '#!/bin/bash
yum update -y
yum install -y iperf3 htop tcpdump wireshark-cli
echo "net.core.rmem_max = 67108864" >> /etc/sysctl.conf
echo "net.core.wmem_max = 67108864" >> /etc/sysctl.conf
echo "net.ipv4.tcp_rmem = 4096 87380 67108864" >> /etc/sysctl.conf
echo "net.ipv4.tcp_wmem = 4096 65536 67108864" >> /etc/sysctl.conf
echo "net.core.netdev_max_backlog = 5000" >> /etc/sysctl.conf
sysctl -p' \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=Performance-Source},{Key=Role,Value=Source}]" \
    --query 'Instances[0].InstanceId' \
    --output text 2>/dev/null || echo "Launch may require key pair setup")

# Launch target instance
TARGET_INSTANCE=$(aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type c5n.large \
    --key-name my-key-pair \
    --security-group-ids $PERF_SG_ID \
    --subnet-id $APP_SUBNET_B \
    --placement "GroupName=$PLACEMENT_GROUP_NAME" \
    --ena-support \
    --user-data '#!/bin/bash
yum update -y
yum install -y iperf3 htop tcpdump wireshark-cli
echo "net.core.rmem_max = 67108864" >> /etc/sysctl.conf
echo "net.core.wmem_max = 67108864" >> /etc/sysctl.conf
echo "net.ipv4.tcp_rmem = 4096 87380 67108864" >> /etc/sysctl.conf
echo "net.ipv4.tcp_wmem = 4096 65536 67108864" >> /etc/sysctl.conf
echo "net.core.netdev_max_backlog = 5000" >> /etc/sysctl.conf
sysctl -p' \
    --tag-specifications "ResourceType=instance,Tags=[{Key=Name,Value=Performance-Target},{Key=Role,Value=Target}]" \
    --query 'Instances[0].InstanceId' \
    --output text 2>/dev/null || echo "Launch may require key pair setup")

echo "✅ Performance testing instances launched"
echo "  Source Instance: $SOURCE_INSTANCE"
echo "  Target Instance: $TARGET_INSTANCE"

echo ""
echo "3. CREATING PERFORMANCE TESTING SCRIPTS"

cat << 'PERF_TEST' > comprehensive-performance-test.sh
#!/bin/bash

echo "=== Comprehensive Network Performance Testing ==="
echo ""

SOURCE_IP="$1"
TARGET_IP="$2"

if [ -z "$SOURCE_IP" ] || [ -z "$TARGET_IP" ]; then
    echo "Usage: $0 <source-ip> <target-ip>"
    echo "Get IPs from: aws ec2 describe-instances"
    exit 1
fi

echo "Testing network performance between:"
echo "  Source: $SOURCE_IP"
echo "  Target: $TARGET_IP"
echo ""

echo "1. BASIC CONNECTIVITY TEST"
ping -c 5 $TARGET_IP

echo ""
echo "2. BANDWIDTH TEST WITH IPERF3"
echo "Starting iperf3 server on target (run manually on target instance):"
echo "  iperf3 -s -p 5001"
echo ""
echo "Running iperf3 client test (TCP):"
iperf3 -c $TARGET_IP -p 5001 -t 30 -P 4

echo ""
echo "Running iperf3 client test (UDP):"
iperf3 -c $TARGET_IP -p 5001 -u -b 10G -t 30

echo ""
echo "3. LATENCY TEST"
echo "RTT latency test:"
ping -c 100 $TARGET_IP | tail -1

echo ""
echo "4. PACKET LOSS TEST"
echo "Packet loss assessment:"
ping -c 1000 $TARGET_IP | grep "packet loss"

echo ""
echo "5. NETWORK INTERFACE STATISTICS"
echo "Interface statistics before/after test:"
cat /proc/net/dev | grep eth0

PERF_TEST

chmod +x comprehensive-performance-test.sh

echo "✅ Performance testing script created"

echo ""
echo "4. CREATING NETWORK OPTIMIZATION SCRIPT"

cat << 'NET_OPTIMIZE' > optimize-network-performance.py
#!/usr/bin/env python3

import subprocess
import sys
import json

class NetworkOptimizer:
    def __init__(self):
        self.optimizations = []
    
    def check_ena_support(self):
        """Check if ENA is supported and enabled"""
        print("CHECKING ENA SUPPORT:")
        print("=" * 30)
        
        try:
            # Check if ENA module is loaded
            result = subprocess.run(['lsmod'], capture_output=True, text=True)
            if 'ena' in result.stdout:
                print("✅ ENA driver is loaded")
                self.optimizations.append("ENA driver active")
            else:
                print("❌ ENA driver not found")
                self.optimizations.append("ENA driver missing")
            
            # Check network interface
            result = subprocess.run(['ethtool', '-i', 'eth0'], capture_output=True, text=True)
            if 'ena' in result.stdout:
                print("✅ ENA interface detected")
            else:
                print("❌ ENA interface not detected")
                
        except Exception as e:
            print(f"Error checking ENA: {e}")
    
    def check_sr_iov_support(self):
        """Check SR-IOV support"""
        print("\nCHECKING SR-IOV SUPPORT:")
        print("=" * 30)
        
        try:
            # Check if SR-IOV is available
            result = subprocess.run(['lspci', '-vvv'], capture_output=True, text=True)
            if 'SR-IOV' in result.stdout:
                print("✅ SR-IOV capabilities detected")
                self.optimizations.append("SR-IOV supported")
            else:
                print("❌ SR-IOV not detected")
                
        except Exception as e:
            print(f"Error checking SR-IOV: {e}")
    
    def optimize_tcp_settings(self):
        """Optimize TCP settings for high performance"""
        print("\nOPTIMIZING TCP SETTINGS:")
        print("=" * 30)
        
        tcp_optimizations = {
            'net.core.rmem_max': '134217728',
            'net.core.wmem_max': '134217728',
            'net.ipv4.tcp_rmem': '4096 65536 134217728',
            'net.ipv4.tcp_wmem': '4096 65536 134217728',
            'net.core.netdev_max_backlog': '5000',
            'net.ipv4.tcp_congestion_control': 'bbr',
            'net.ipv4.tcp_mtu_probing': '1',
            'net.ipv4.tcp_window_scaling': '1'
        }
        
        print("Recommended TCP optimizations:")
        for setting, value in tcp_optimizations.items():
            print(f"  {setting} = {value}")
            
        print("\nTo apply these settings:")
        print("  1. Add to /etc/sysctl.conf")
        print("  2. Run: sysctl -p")
        print("  3. Restart networking service")
        
        self.optimizations.append("TCP settings optimized")
    
    def check_jumbo_frames(self):
        """Check and recommend jumbo frames configuration"""
        print("\nCHECKING JUMBO FRAMES:")
        print("=" * 30)
        
        try:
            result = subprocess.run(['ip', 'link', 'show', 'eth0'], capture_output=True, text=True)
            if 'mtu 9000' in result.stdout:
                print("✅ Jumbo frames (9000 MTU) enabled")
                self.optimizations.append("Jumbo frames enabled")
            elif 'mtu 1500' in result.stdout:
                print("⚠️  Standard MTU (1500) detected")
                print("   Consider enabling jumbo frames for better performance")
                print("   Command: sudo ip link set dev eth0 mtu 9000")
                self.optimizations.append("Jumbo frames recommended")
            
        except Exception as e:
            print(f"Error checking MTU: {e}")
    
    def analyze_cpu_affinity(self):
        """Analyze CPU affinity for network interrupts"""
        print("\nANALYZING CPU AFFINITY:")
        print("=" * 30)
        
        print("Network interrupt optimization recommendations:")
        print("  1. Bind network interrupts to specific CPU cores")
        print("  2. Isolate application threads from interrupt handling")
        print("  3. Use NUMA topology awareness")
        print("  4. Configure receive packet steering (RPS)")
        
        cpu_optimization_script = '''
# Example CPU affinity optimization
echo 2 > /proc/irq/24/smp_affinity  # Bind to CPU 1
echo 4 > /proc/irq/25/smp_affinity  # Bind to CPU 2
echo rps_cpus=f > /sys/class/net/eth0/queues/rx-0/rps_cpus
        '''
        
        print("\nSample optimization commands:")
        print(cpu_optimization_script)
        
        self.optimizations.append("CPU affinity analysis complete")
    
    def dpdk_recommendations(self):
        """Provide DPDK implementation recommendations"""
        print("\nDPDK OPTIMIZATION RECOMMENDATIONS:")
        print("=" * 40)
        
        dpdk_benefits = [
            "Userspace packet processing",
            "Kernel bypass for low latency",
            "Poll mode drivers (PMD)",
            "Lock-free ring buffers",
            "NUMA awareness",
            "CPU affinity control"
        ]
        
        print("DPDK Benefits:")
        for benefit in dpdk_benefits:
            print(f"  - {benefit}")
        
        print("\nDPDK Implementation Steps:")
        implementation_steps = [
            "Install DPDK libraries",
            "Configure hugepages",
            "Bind network interfaces to DPDK",
            "Develop DPDK application",
            "Optimize for specific workload"
        ]
        
        for i, step in enumerate(implementation_steps, 1):
            print(f"  {i}. {step}")
        
        print("\nDPDK Use Cases:")
        use_cases = [
            "High-frequency trading",
            "Network function virtualization",
            "Packet processing applications",
            "Real-time analytics",
            "Low-latency messaging"
        ]
        
        for use_case in use_cases:
            print(f"  - {use_case}")
    
    def generate_optimization_report(self):
        """Generate comprehensive optimization report"""
        print("\nOPTIMIZATION SUMMARY REPORT:")
        print("=" * 40)
        
        for i, optimization in enumerate(self.optimizations, 1):
            print(f"{i}. {optimization}")
        
        print(f"\nTotal optimizations analyzed: {len(self.optimizations)}")
        
        print("\nNext Steps:")
        next_steps = [
            "Apply recommended TCP optimizations",
            "Enable jumbo frames if applicable",
            "Configure CPU affinity for interrupts",
            "Consider DPDK for ultra-low latency needs",
            "Monitor performance improvements"
        ]
        
        for i, step in enumerate(next_steps, 1):
            print(f"  {i}. {step}")

def main():
    optimizer = NetworkOptimizer()
    
    print("Network Performance Optimization Analysis")
    print("=" * 50)
    
    optimizer.check_ena_support()
    optimizer.check_sr_iov_support()
    optimizer.optimize_tcp_settings()
    optimizer.check_jumbo_frames()
    optimizer.analyze_cpu_affinity()
    optimizer.dpdk_recommendations()
    optimizer.generate_optimization_report()

if __name__ == "__main__":
    main()

NET_OPTIMIZE

chmod +x optimize-network-performance.py

echo "✅ Network optimization script created"

# Save performance configuration
cat << CONFIG > performance-config.env
export PLACEMENT_GROUP_NAME="$PLACEMENT_GROUP_NAME"
export PERF_SG_ID="$PERF_SG_ID"
export SOURCE_INSTANCE="$SOURCE_INSTANCE"
export TARGET_INSTANCE="$TARGET_INSTANCE"
export AMI_ID="$AMI_ID"
CONFIG

echo ""
echo "ENHANCED NETWORKING IMPLEMENTATION SUMMARY:"
echo "Placement Group: $PLACEMENT_GROUP_NAME"
echo "Security Group: $PERF_SG_ID"
echo "AMI with ENA: $AMI_ID"
echo ""
echo "Performance Testing Tools:"
echo "  - comprehensive-performance-test.sh"
echo "  - optimize-network-performance.py"
echo ""
echo "Configuration saved to performance-config.env"

EOF

chmod +x implement-enhanced-networking.sh
./implement-enhanced-networking.sh
```

### Step 3: Configure Jumbo Frames and MTU Optimization
**Objective**: Optimize frame sizes for maximum throughput

**MTU Optimization Implementation**:
```bash
# Configure MTU optimization and jumbo frames
cat << 'EOF' > configure-mtu-optimization.sh
#!/bin/bash

echo "=== Configuring MTU Optimization and Jumbo Frames ==="
echo ""

echo "1. MTU ANALYSIS AND RECOMMENDATIONS"

cat << 'MTU_ANALYSIS' > analyze-mtu-requirements.py
#!/usr/bin/env python3

import subprocess
import json

class MTUAnalyzer:
    def __init__(self):
        self.current_mtu = self.get_current_mtu()
        self.recommendations = []
    
    def get_current_mtu(self):
        """Get current MTU setting"""
        try:
            result = subprocess.run(['ip', 'link', 'show', 'eth0'], 
                                  capture_output=True, text=True)
            for line in result.stdout.split('\n'):
                if 'mtu' in line:
                    mtu = line.split('mtu')[1].split()[0]
                    return int(mtu)
        except:
            return 1500  # Default MTU
    
    def analyze_mtu_benefits(self):
        """Analyze benefits of different MTU sizes"""
        print("MTU SIZE ANALYSIS:")
        print("=" * 30)
        
        mtu_scenarios = {
            1500: {
                'description': 'Standard Ethernet MTU',
                'overhead': 'Higher packet overhead',
                'compatibility': 'Universal compatibility',
                'performance': 'Standard performance',
                'use_case': 'Internet traffic, mixed workloads'
            },
            9000: {
                'description': 'Jumbo frames',
                'overhead': 'Reduced packet overhead',
                'compatibility': 'Same VPC/region only',
                'performance': 'Up to 30% throughput improvement',
                'use_case': 'High bandwidth applications'
            },
            1450: {
                'description': 'VPN-optimized MTU',
                'overhead': 'Accounts for tunnel overhead',
                'compatibility': 'VPN and tunnel traffic',
                'performance': 'Optimized for encrypted traffic',
                'use_case': 'VPN connections, encrypted tunnels'
            }
        }
        
        print(f"Current MTU: {self.current_mtu}")
        print("")
        
        for mtu_size, details in mtu_scenarios.items():
            print(f"MTU {mtu_size}: {details['description']}")
            print(f"  Overhead: {details['overhead']}")
            print(f"  Compatibility: {details['compatibility']}")
            print(f"  Performance: {details['performance']}")
            print(f"  Use Case: {details['use_case']}")
            print("")
            
            if mtu_size != self.current_mtu:
                self.recommendations.append(f"Consider MTU {mtu_size} for {details['use_case']}")
    
    def test_mtu_connectivity(self, target_ip, mtu_size):
        """Test MTU connectivity to target"""
        print(f"TESTING MTU {mtu_size} TO {target_ip}:")
        
        # Calculate payload size (subtract IP and ICMP headers)
        payload_size = mtu_size - 28
        
        try:
            result = subprocess.run([
                'ping', '-c', '3', '-M', 'do', '-s', str(payload_size), target_ip
            ], capture_output=True, text=True)
            
            if result.returncode == 0:
                print(f"  ✅ MTU {mtu_size} supported")
                return True
            else:
                print(f"  ❌ MTU {mtu_size} not supported")
                return False
        except:
            print(f"  ❌ MTU test failed")
            return False
    
    def calculate_performance_impact(self):
        """Calculate performance impact of MTU changes"""
        print("PERFORMANCE IMPACT CALCULATION:")
        print("=" * 40)
        
        # Example calculation for 1GB transfer
        data_size_gb = 1
        data_size_bytes = data_size_gb * 1024 * 1024 * 1024
        
        scenarios = {
            1500: {'header_size': 54, 'payload': 1446},  # Ethernet + IP + TCP
            9000: {'header_size': 54, 'payload': 8946}   # Jumbo frame
        }
        
        for mtu, details in scenarios.items():
            packets_needed = data_size_bytes // details['payload']
            total_headers = packets_needed * details['header_size']
            overhead_percentage = (total_headers / data_size_bytes) * 100
            
            print(f"MTU {mtu}:")
            print(f"  Packets needed: {packets_needed:,}")
            print(f"  Header overhead: {total_headers:,} bytes ({overhead_percentage:.2f}%)")
            print(f"  Efficiency: {100-overhead_percentage:.2f}%")
            print("")
    
    def provide_optimization_recommendations(self):
        """Provide MTU optimization recommendations"""
        print("MTU OPTIMIZATION RECOMMENDATIONS:")
        print("=" * 40)
        
        optimization_strategies = {
            'intra_vpc_traffic': {
                'recommendation': 'Use 9000 MTU (jumbo frames)',
                'rationale': 'Maximum efficiency for internal traffic',
                'implementation': 'Configure on all instances in same VPC'
            },
            'internet_traffic': {
                'recommendation': 'Keep 1500 MTU',
                'rationale': 'Ensure compatibility with internet routers',
                'implementation': 'Use standard MTU for internet-facing traffic'
            },
            'vpn_traffic': {
                'recommendation': 'Use 1450 MTU or lower',
                'rationale': 'Account for VPN tunnel overhead',
                'implementation': 'Test optimal size for specific VPN setup'
            },
            'mixed_workloads': {
                'recommendation': 'Use path MTU discovery',
                'rationale': 'Automatically optimize for each path',
                'implementation': 'Enable TCP path MTU discovery'
            }
        }
        
        for strategy, details in optimization_strategies.items():
            print(f"{strategy.upper().replace('_', ' ')}:")
            print(f"  Recommendation: {details['recommendation']}")
            print(f"  Rationale: {details['rationale']}")
            print(f"  Implementation: {details['implementation']}")
            print("")
    
    def generate_mtu_script(self):
        """Generate MTU configuration script"""
        print("MTU CONFIGURATION SCRIPT:")
        print("=" * 30)
        
        script_content = '''#!/bin/bash

# MTU Configuration Script
echo "Configuring optimal MTU settings..."

# Backup current configuration
ip link show eth0 > /tmp/mtu_backup.txt

# Test current connectivity
ping -c 3 8.8.8.8 > /tmp/connectivity_test_before.txt

# Configure jumbo frames for intra-VPC traffic
echo "Setting MTU to 9000 for jumbo frames..."
sudo ip link set dev eth0 mtu 9000

# Verify configuration
NEW_MTU=$(ip link show eth0 | grep -o 'mtu [0-9]*' | cut -d' ' -f2)
echo "New MTU: $NEW_MTU"

# Test connectivity after change
ping -c 3 8.8.8.8 > /tmp/connectivity_test_after.txt

# Make persistent (add to network configuration)
echo "Making MTU setting persistent..."
echo "MTU=9000" >> /etc/sysconfig/network-scripts/ifcfg-eth0

echo "MTU configuration complete"
echo "Reboot required for full persistence"
        '''
        
        with open('configure_mtu.sh', 'w') as f:
            f.write(script_content)
        
        print("MTU configuration script saved to: configure_mtu.sh")
        print("Make executable with: chmod +x configure_mtu.sh")

def main():
    analyzer = MTUAnalyzer()
    
    analyzer.analyze_mtu_benefits()
    analyzer.calculate_performance_impact()
    analyzer.provide_optimization_recommendations()
    analyzer.generate_mtu_script()
    
    print(f"\nTotal recommendations: {len(analyzer.recommendations)}")
    for i, rec in enumerate(analyzer.recommendations, 1):
        print(f"{i}. {rec}")

if __name__ == "__main__":
    main()

MTU_ANALYSIS

chmod +x analyze-mtu-requirements.py

echo "✅ MTU analysis tool created"

echo ""
echo "2. CREATING JUMBO FRAMES IMPLEMENTATION SCRIPT"

cat << 'JUMBO_FRAMES' > implement-jumbo-frames.sh
#!/bin/bash

echo "=== Implementing Jumbo Frames Configuration ==="
echo ""

echo "1. PRE-IMPLEMENTATION CHECKS"

# Check current MTU
CURRENT_MTU=$(ip link show eth0 | grep -o 'mtu [0-9]*' | cut -d' ' -f2)
echo "Current MTU: $CURRENT_MTU"

# Check if instance supports jumbo frames
INSTANCE_TYPE=$(curl -s http://169.254.169.254/latest/meta-data/instance-type)
echo "Instance Type: $INSTANCE_TYPE"

# Check if we're in a VPC (required for jumbo frames)
VPC_ID=$(curl -s http://169.254.169.254/latest/meta-data/network/interfaces/macs/$(curl -s http://169.254.169.254/latest/meta-data/mac)/vpc-id)
echo "VPC ID: $VPC_ID"

if [ -z "$VPC_ID" ]; then
    echo "❌ Jumbo frames require VPC deployment"
    exit 1
fi

echo ""
echo "2. CONNECTIVITY TESTING BEFORE JUMBO FRAMES"

# Test connectivity with standard MTU
echo "Testing connectivity with current MTU..."
ping -c 3 -M do -s 1472 8.8.8.8 > /tmp/mtu_test_1500.txt
if [ $? -eq 0 ]; then
    echo "✅ Standard MTU (1500) connectivity confirmed"
else
    echo "❌ Standard MTU connectivity issues"
fi

echo ""
echo "3. IMPLEMENTING JUMBO FRAMES"

# Backup current network configuration
echo "Backing up current network configuration..."
cp /etc/sysconfig/network-scripts/ifcfg-eth0 /tmp/ifcfg-eth0.backup 2>/dev/null || echo "Network config backup not needed"

# Set jumbo frames
echo "Configuring jumbo frames (MTU 9000)..."
sudo ip link set dev eth0 mtu 9000

# Verify the change
NEW_MTU=$(ip link show eth0 | grep -o 'mtu [0-9]*' | cut -d' ' -f2)
echo "New MTU: $NEW_MTU"

if [ "$NEW_MTU" = "9000" ]; then
    echo "✅ Jumbo frames successfully configured"
else
    echo "❌ Jumbo frames configuration failed"
    exit 1
fi

echo ""
echo "4. CONNECTIVITY TESTING AFTER JUMBO FRAMES"

# Test jumbo frame connectivity
echo "Testing jumbo frame connectivity..."
ping -c 3 -M do -s 8972 127.0.0.1 > /tmp/mtu_test_9000.txt
if [ $? -eq 0 ]; then
    echo "✅ Jumbo frames connectivity confirmed"
else
    echo "⚠️  Jumbo frames local test issues (may be normal)"
fi

# Test standard connectivity still works
ping -c 3 8.8.8.8 > /tmp/connectivity_test_jumbo.txt
if [ $? -eq 0 ]; then
    echo "✅ Standard internet connectivity maintained"
else
    echo "❌ Internet connectivity issues after jumbo frames"
fi

echo ""
echo "5. MAKING CONFIGURATION PERSISTENT"

# Make MTU setting persistent
echo "Making jumbo frames persistent..."

# For Amazon Linux 2
if [ -f /etc/sysconfig/network-scripts/ifcfg-eth0 ]; then
    echo "Adding MTU=9000 to network configuration..."
    grep -q "MTU=" /etc/sysconfig/network-scripts/ifcfg-eth0 && \
        sed -i 's/MTU=.*/MTU=9000/' /etc/sysconfig/network-scripts/ifcfg-eth0 || \
        echo "MTU=9000" >> /etc/sysconfig/network-scripts/ifcfg-eth0
fi

# For Ubuntu/Debian systems
if [ -f /etc/netplan/50-cloud-init.yaml ]; then
    echo "Note: For Ubuntu systems, configure MTU in netplan"
    echo "Add 'mtu: 9000' under the ethernet interface configuration"
fi

echo ""
echo "6. PERFORMANCE OPTIMIZATION RECOMMENDATIONS"

echo "Additional optimizations for jumbo frames:"
echo "  1. Ensure all instances in communication path support jumbo frames"
echo "  2. Configure application buffers to utilize larger frames"
echo "  3. Monitor network performance improvements"
echo "  4. Test with your specific workload patterns"

echo ""
echo "JUMBO FRAMES IMPLEMENTATION COMPLETED"
echo "Reboot recommended to ensure persistence across restarts"

JUMBO_FRAMES

chmod +x implement-jumbo-frames.sh

echo "✅ Jumbo frames implementation script created"

echo ""
echo "3. CREATING MTU PATH DISCOVERY TOOL"

cat << 'PATH_MTU' > path-mtu-discovery.py
#!/usr/bin/env python3

import subprocess
import socket
import struct
import sys

class PathMTUDiscovery:
    def __init__(self):
        self.results = {}
    
    def discover_path_mtu(self, target_host, max_mtu=9000, min_mtu=576):
        """Discover optimal MTU for path to target host"""
        print(f"DISCOVERING PATH MTU TO {target_host}:")
        print("=" * 50)
        
        # Binary search for optimal MTU
        low = min_mtu
        high = max_mtu
        optimal_mtu = min_mtu
        
        while low <= high:
            mid = (low + high) // 2
            
            if self.test_mtu_size(target_host, mid):
                optimal_mtu = mid
                low = mid + 1
                print(f"  ✅ MTU {mid} works")
            else:
                high = mid - 1
                print(f"  ❌ MTU {mid} fails")
        
        self.results[target_host] = optimal_mtu
        print(f"\nOptimal MTU for {target_host}: {optimal_mtu}")
        return optimal_mtu
    
    def test_mtu_size(self, target_host, mtu_size):
        """Test if specific MTU size works for target host"""
        # Calculate payload size (subtract IP and ICMP headers)
        payload_size = mtu_size - 28
        
        try:
            result = subprocess.run([
                'ping', '-c', '1', '-W', '2', '-M', 'do', 
                '-s', str(payload_size), target_host
            ], capture_output=True, text=True, timeout=5)
            
            return result.returncode == 0
        except:
            return False
    
    def test_multiple_destinations(self, destinations):
        """Test MTU for multiple destinations"""
        print("TESTING MULTIPLE DESTINATIONS:")
        print("=" * 40)
        
        for dest in destinations:
            try:
                optimal_mtu = self.discover_path_mtu(dest)
                print(f"{dest}: {optimal_mtu} bytes")
            except Exception as e:
                print(f"{dest}: Error - {e}")
            print("")
    
    def generate_recommendations(self):
        """Generate MTU recommendations based on discovered values"""
        print("MTU RECOMMENDATIONS:")
        print("=" * 30)
        
        if not self.results:
            print("No MTU discovery results available")
            return
        
        min_mtu = min(self.results.values())
        max_mtu = max(self.results.values())
        
        print(f"Minimum MTU across all paths: {min_mtu}")
        print(f"Maximum MTU across all paths: {max_mtu}")
        
        if min_mtu == max_mtu:
            print(f"Recommendation: Use MTU {min_mtu} for all traffic")
        else:
            print(f"Recommendation: Use MTU {min_mtu} for universal compatibility")
            print(f"Alternative: Use per-destination MTU optimization")
        
        # Generate configuration
        print("\nConfiguration commands:")
        print(f"sudo ip link set dev eth0 mtu {min_mtu}")
        
        if min_mtu >= 9000:
            print("✅ Jumbo frames fully supported")
        elif min_mtu >= 1500:
            print("✅ Standard frames supported")
        else:
            print("⚠️  Reduced MTU required")

def main():
    discovery = PathMTUDiscovery()
    
    # Common test destinations
    test_destinations = [
        '8.8.8.8',      # Google DNS
        '1.1.1.1',      # Cloudflare DNS
        'amazon.com',   # AWS service
    ]
    
    if len(sys.argv) > 1:
        # Use provided destination
        test_destinations = sys.argv[1:]
    
    discovery.test_multiple_destinations(test_destinations)
    discovery.generate_recommendations()

if __name__ == "__main__":
    main()

PATH_MTU

chmod +x path-mtu-discovery.py

echo "✅ Path MTU discovery tool created"

echo ""
echo "MTU OPTIMIZATION CONFIGURATION COMPLETED"
echo ""
echo "Available Tools:"
echo "1. analyze-mtu-requirements.py - Comprehensive MTU analysis"
echo "2. implement-jumbo-frames.sh - Jumbo frames implementation"
echo "3. path-mtu-discovery.py - Path MTU discovery tool"
echo ""
echo "Usage Examples:"
echo "python3 analyze-mtu-requirements.py"
echo "./implement-jumbo-frames.sh"
echo "python3 path-mtu-discovery.py [target-host]"

EOF

chmod +x configure-mtu-optimization.sh
./configure-mtu-optimization.sh
```

### Step 4: Implement Application-Level Optimization
**Objective**: Optimize applications for maximum network performance

**Application Optimization Implementation**:
```bash
# Implement application-level network optimization
cat << 'EOF' > implement-application-optimization.sh
#!/bin/bash

echo "=== Implementing Application-Level Network Optimization ==="
echo ""

echo "1. CREATING APPLICATION PERFORMANCE ANALYZER"

cat << 'APP_ANALYZER' > application-performance-analyzer.py
#!/usr/bin/env python3

import psutil
import socket
import threading
import time
import json
from datetime import datetime

class ApplicationPerformanceAnalyzer:
    def __init__(self):
        self.metrics = {}
        self.monitoring = False
    
    def analyze_network_connections(self):
        """Analyze current network connections"""
        print("NETWORK CONNECTIONS ANALYSIS:")
        print("=" * 40)
        
        connections = psutil.net_connections(kind='inet')
        
        # Categorize connections
        connection_stats = {
            'tcp_established': 0,
            'tcp_listen': 0,
            'tcp_time_wait': 0,
            'udp_connections': 0
        }
        
        port_usage = {}
        
        for conn in connections:
            if conn.type == socket.SOCK_STREAM:  # TCP
                if conn.status == 'ESTABLISHED':
                    connection_stats['tcp_established'] += 1
                elif conn.status == 'LISTEN':
                    connection_stats['tcp_listen'] += 1
                elif conn.status == 'TIME_WAIT':
                    connection_stats['tcp_time_wait'] += 1
                    
                if conn.laddr:
                    port = conn.laddr.port
                    port_usage[port] = port_usage.get(port, 0) + 1
                    
            elif conn.type == socket.SOCK_DGRAM:  # UDP
                connection_stats['udp_connections'] += 1
        
        print("Connection Statistics:")
        for stat, count in connection_stats.items():
            print(f"  {stat.replace('_', ' ').title()}: {count}")
        
        print("\nTop 10 Ports by Connection Count:")
        sorted_ports = sorted(port_usage.items(), key=lambda x: x[1], reverse=True)[:10]
        for port, count in sorted_ports:
            print(f"  Port {port}: {count} connections")
        
        return connection_stats, port_usage
    
    def analyze_socket_buffers(self):
        """Analyze socket buffer configurations"""
        print("\nSOCKET BUFFER ANALYSIS:")
        print("=" * 30)
        
        try:
            # Get current socket buffer sizes
            with open('/proc/sys/net/core/rmem_default', 'r') as f:
                rmem_default = int(f.read().strip())
            with open('/proc/sys/net/core/rmem_max', 'r') as f:
                rmem_max = int(f.read().strip())
            with open('/proc/sys/net/core/wmem_default', 'r') as f:
                wmem_default = int(f.read().strip())
            with open('/proc/sys/net/core/wmem_max', 'r') as f:
                wmem_max = int(f.read().strip())
            
            print("Current Socket Buffer Sizes:")
            print(f"  Receive buffer default: {rmem_default:,} bytes")
            print(f"  Receive buffer maximum: {rmem_max:,} bytes")
            print(f"  Send buffer default: {wmem_default:,} bytes")
            print(f"  Send buffer maximum: {wmem_max:,} bytes")
            
            # Recommendations
            recommended = {
                'rmem_default': 262144,    # 256KB
                'rmem_max': 67108864,      # 64MB
                'wmem_default': 262144,    # 256KB
                'wmem_max': 67108864       # 64MB
            }
            
            print("\nRecommended High-Performance Settings:")
            for param, value in recommended.items():
                current_value = locals()[param]
                status = "✅ OK" if current_value >= value else "⚠️  UPGRADE"
                print(f"  {param}: {value:,} bytes {status}")
                
        except Exception as e:
            print(f"Error reading socket buffer settings: {e}")
    
    def monitor_real_time_performance(self, duration=60):
        """Monitor real-time network performance"""
        print(f"\nREAL-TIME MONITORING ({duration} seconds):")
        print("=" * 40)
        
        start_time = time.time()
        initial_stats = psutil.net_io_counters()
        
        print("Starting network monitoring...")
        print("Timestamp,Bytes_Sent,Bytes_Recv,Packets_Sent,Packets_Recv,Errors")
        
        while time.time() - start_time < duration:
            current_stats = psutil.net_io_counters()
            
            # Calculate rates
            elapsed = time.time() - start_time
            bytes_sent_rate = (current_stats.bytes_sent - initial_stats.bytes_sent) / elapsed
            bytes_recv_rate = (current_stats.bytes_recv - initial_stats.bytes_recv) / elapsed
            
            timestamp = datetime.now().strftime("%H:%M:%S")
            print(f"{timestamp},{current_stats.bytes_sent},{current_stats.bytes_recv},"
                  f"{current_stats.packets_sent},{current_stats.packets_recv},"
                  f"{current_stats.errin + current_stats.errout}")
            
            time.sleep(5)
        
        final_stats = psutil.net_io_counters()
        total_elapsed = time.time() - start_time
        
        print(f"\nSummary ({total_elapsed:.1f} seconds):")
        print(f"  Average send rate: {(final_stats.bytes_sent - initial_stats.bytes_sent) / total_elapsed / 1024 / 1024:.2f} MB/s")
        print(f"  Average receive rate: {(final_stats.bytes_recv - initial_stats.bytes_recv) / total_elapsed / 1024 / 1024:.2f} MB/s")
        print(f"  Total errors: {(final_stats.errin + final_stats.errout) - (initial_stats.errin + initial_stats.errout)}")
    
    def optimize_application_settings(self):
        """Provide application-specific optimization recommendations"""
        print("\nAPPLICATION OPTIMIZATION RECOMMENDATIONS:")
        print("=" * 50)
        
        optimizations = {
            'web_servers': {
                'description': 'HTTP/HTTPS servers (Apache, Nginx)',
                'optimizations': [
                    'Enable HTTP/2 and HTTP/3',
                    'Configure connection pooling',
                    'Use sendfile() for static content',
                    'Enable gzip compression',
                    'Optimize keep-alive settings'
                ]
            },
            'databases': {
                'description': 'Database servers (MySQL, PostgreSQL)',
                'optimizations': [
                    'Increase connection pool size',
                    'Configure query result caching',
                    'Use connection multiplexing',
                    'Optimize buffer pool sizes',
                    'Enable query optimization'
                ]
            },
            'messaging': {
                'description': 'Message queues (RabbitMQ, Kafka)',
                'optimizations': [
                    'Batch message processing',
                    'Configure producer/consumer buffers',
                    'Use persistent connections',
                    'Optimize serialization',
                    'Configure partition strategies'
                ]
            },
            'real_time': {
                'description': 'Real-time applications',
                'optimizations': [
                    'Use UDP for low-latency data',
                    'Implement custom protocols',
                    'Minimize system calls',
                    'Use memory-mapped files',
                    'Configure CPU affinity'
                ]
            }
        }
        
        for app_type, details in optimizations.items():
            print(f"\n{app_type.upper().replace('_', ' ')}:")
            print(f"  Description: {details['description']}")
            print("  Optimizations:")
            for opt in details['optimizations']:
                print(f"    - {opt}")
    
    def generate_tuning_script(self):
        """Generate system tuning script for network performance"""
        print("\nGENERATING SYSTEM TUNING SCRIPT:")
        print("=" * 40)
        
        tuning_script = '''#!/bin/bash

# High-Performance Network Tuning Script
echo "Applying high-performance network tuning..."

# TCP Buffer Optimization
echo "Configuring TCP buffers..."
echo "net.core.rmem_default = 262144" >> /etc/sysctl.conf
echo "net.core.rmem_max = 67108864" >> /etc/sysctl.conf
echo "net.core.wmem_default = 262144" >> /etc/sysctl.conf
echo "net.core.wmem_max = 67108864" >> /etc/sysctl.conf
echo "net.ipv4.tcp_rmem = 4096 262144 67108864" >> /etc/sysctl.conf
echo "net.ipv4.tcp_wmem = 4096 262144 67108864" >> /etc/sysctl.conf

# TCP Congestion Control
echo "Configuring TCP congestion control..."
echo "net.ipv4.tcp_congestion_control = bbr" >> /etc/sysctl.conf
echo "net.core.default_qdisc = fq" >> /etc/sysctl.conf

# Network Interface Optimization
echo "Optimizing network interface..."
echo "net.core.netdev_max_backlog = 5000" >> /etc/sysctl.conf
echo "net.core.netdev_budget = 600" >> /etc/sysctl.conf

# TCP Connection Optimization
echo "Optimizing TCP connections..."
echo "net.ipv4.tcp_window_scaling = 1" >> /etc/sysctl.conf
echo "net.ipv4.tcp_timestamps = 1" >> /etc/sysctl.conf
echo "net.ipv4.tcp_sack = 1" >> /etc/sysctl.conf
echo "net.ipv4.tcp_no_metrics_save = 1" >> /etc/sysctl.conf

# Apply settings
sysctl -p

echo "Network tuning complete. Reboot recommended."
        '''
        
        with open('network_tuning.sh', 'w') as f:
            f.write(tuning_script)
        
        print("Network tuning script saved to: network_tuning.sh")
        print("Make executable with: chmod +x network_tuning.sh")
        print("Run with: sudo ./network_tuning.sh")

def main():
    analyzer = ApplicationPerformanceAnalyzer()
    
    print("Application Performance Analysis")
    print("=" * 40)
    
    analyzer.analyze_network_connections()
    analyzer.analyze_socket_buffers()
    analyzer.optimize_application_settings()
    analyzer.generate_tuning_script()
    
    # Optional real-time monitoring
    import sys
    if len(sys.argv) > 1 and sys.argv[1] == '--monitor':
        duration = int(sys.argv[2]) if len(sys.argv) > 2 else 60
        analyzer.monitor_real_time_performance(duration)

if __name__ == "__main__":
    main()

APP_ANALYZER

chmod +x application-performance-analyzer.py

echo "✅ Application performance analyzer created"

echo ""
echo "2. CREATING LOAD TESTING FRAMEWORK"

cat << 'LOAD_TEST' > create-load-testing-framework.sh
#!/bin/bash

echo "=== Creating Load Testing Framework ==="
echo ""

echo "1. INSTALLING LOAD TESTING TOOLS"

# Install various load testing tools
sudo yum install -y httpd-tools  # for ab (Apache Bench)

# Check if wrk is available
if ! command -v wrk &> /dev/null; then
    echo "Installing wrk load testing tool..."
    sudo yum install -y git gcc
    git clone https://github.com/wg/wrk.git
    cd wrk && make && sudo cp wrk /usr/local/bin/
    cd .. && rm -rf wrk
fi

echo "✅ Load testing tools installed"

echo ""
echo "2. CREATING COMPREHENSIVE LOAD TEST SCRIPT"

cat << 'LOAD_TEST_SCRIPT' > comprehensive-load-test.sh
#!/bin/bash

TARGET_URL="$1"
CONCURRENT_USERS="$2"
TEST_DURATION="$3"

if [ -z "$TARGET_URL" ] || [ -z "$CONCURRENT_USERS" ] || [ -z "$TEST_DURATION" ]; then
    echo "Usage: $0 <target-url> <concurrent-users> <test-duration-seconds>"
    echo "Example: $0 http://example.com 100 60"
    exit 1
fi

echo "=== Comprehensive Load Testing ==="
echo "Target: $TARGET_URL"
echo "Concurrent Users: $CONCURRENT_USERS"
echo "Duration: $TEST_DURATION seconds"
echo ""

echo "1. APACHE BENCH (AB) TESTING"
echo "Starting ab test..."
ab -n $((CONCURRENT_USERS * 100)) -c $CONCURRENT_USERS $TARGET_URL > ab_results.txt
echo "✅ AB test completed - results in ab_results.txt"

echo ""
echo "2. WRK TESTING"
if command -v wrk &> /dev/null; then
    echo "Starting wrk test..."
    wrk -t$CONCURRENT_USERS -c$CONCURRENT_USERS -d${TEST_DURATION}s $TARGET_URL > wrk_results.txt
    echo "✅ WRK test completed - results in wrk_results.txt"
else
    echo "⚠️  WRK not available, skipping"
fi

echo ""
echo "3. CURL PERFORMANCE TESTING"
echo "Running curl performance tests..."

for i in {1..10}; do
    curl -w "@curl-format.txt" -o /dev/null -s $TARGET_URL
done > curl_results.txt

echo "✅ Curl tests completed - results in curl_results.txt"

echo ""
echo "4. GENERATING PERFORMANCE REPORT"

cat << 'REPORT' > performance_report.txt
Performance Test Report
=====================

Test Configuration:
- Target URL: $TARGET_URL
- Concurrent Users: $CONCURRENT_USERS
- Test Duration: $TEST_DURATION seconds
- Test Time: $(date)

Results Summary:
REPORT

if [ -f ab_results.txt ]; then
    echo "" >> performance_report.txt
    echo "Apache Bench Results:" >> performance_report.txt
    grep -E "(Requests per second|Time per request|Transfer rate)" ab_results.txt >> performance_report.txt
fi

if [ -f wrk_results.txt ]; then
    echo "" >> performance_report.txt
    echo "WRK Results:" >> performance_report.txt
    tail -n 10 wrk_results.txt >> performance_report.txt
fi

echo "✅ Performance report generated: performance_report.txt"

LOAD_TEST_SCRIPT

chmod +x comprehensive-load-test.sh

echo "✅ Load testing framework created"

echo ""
echo "3. CREATING CURL FORMAT FILE FOR DETAILED METRICS"

cat << 'CURL_FORMAT' > curl-format.txt
     time_namelookup:  %{time_namelookup}\n
        time_connect:  %{time_connect}\n
     time_appconnect:  %{time_appconnect}\n
    time_pretransfer:  %{time_pretransfer}\n
       time_redirect:  %{time_redirect}\n
  time_starttransfer:  %{time_starttransfer}\n
                     ----------\n
          time_total:  %{time_total}\n
CURL_FORMAT

echo "✅ Curl format file created"

LOAD_TEST

chmod +x create-load-testing-framework.sh

echo "✅ Load testing framework setup script created"

echo ""
echo "APPLICATION OPTIMIZATION IMPLEMENTATION COMPLETED"
echo ""
echo "Available Tools:"
echo "1. application-performance-analyzer.py - Application analysis tool"
echo "2. create-load-testing-framework.sh - Load testing setup"
echo "3. comprehensive-load-test.sh - Load testing execution"
echo ""
echo "Usage Examples:"
echo "python3 application-performance-analyzer.py"
echo "python3 application-performance-analyzer.py --monitor 120"
echo "./create-load-testing-framework.sh"
echo "./comprehensive-load-test.sh http://example.com 100 60"

EOF

chmod +x implement-application-optimization.sh
./implement-application-optimization.sh
```

## VPC Performance Optimization Summary

### Enhanced Networking Technologies

#### SR-IOV and ENA Benefits
- **Lower Latency**: Hardware-level network virtualization
- **Higher PPS**: Up to 14 million packets per second
- **Reduced CPU**: Offload network processing to hardware
- **Better Bandwidth**: Up to 100 Gbps on supported instances

#### Instance Performance Characteristics
```
Performance Tiers:
- Ultra High: C5n, M5n, R5n (100 Gbps, 14M PPS)
- High: C5, M5, R5 (25 Gbps, 6M PPS) 
- Standard: C4, M4, R4 (10 Gbps, 2M PPS)
- Burstable: T3, T4g (5 Gbps, 1M PPS)
```

### Network Acceleration Techniques

#### Jumbo Frames (9000 MTU)
- **Efficiency Gain**: Up to 30% throughput improvement
- **Reduced Overhead**: Fewer packets for same data volume
- **Requirements**: Same VPC/region, end-to-end support

#### TCP Optimization
- **Buffer Tuning**: Larger socket buffers for high bandwidth
- **Congestion Control**: BBR algorithm for better performance
- **Window Scaling**: Support for large TCP windows

### Placement Groups and Optimization

#### Cluster Placement Groups
- **Low Latency**: Sub-millisecond latency between instances
- **High Bandwidth**: Full bisection bandwidth
- **Use Cases**: HPC, distributed databases, real-time applications

#### Performance Monitoring
- **Baseline Measurement**: Establish performance baselines
- **Continuous Monitoring**: Real-time performance tracking
- **Bottleneck Identification**: CPU, network, or application limits

## Key Takeaways
- Enhanced networking provides significant performance benefits
- Instance type selection impacts maximum network performance
- Jumbo frames improve efficiency for high-throughput applications
- Placement groups enable ultra-low latency communication
- Application-level optimization is crucial for maximum performance
- Comprehensive monitoring enables performance optimization
- Load testing validates optimization effectiveness

## Additional Resources
- [Enhanced Networking on Linux](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/enhanced-networking.html)
- [Placement Groups](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/placement-groups.html)
- [Network Performance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/general-purpose-instances.html#general-purpose-network-performance)

## Exam Tips
- SR-IOV provides hardware-level network virtualization
- ENA enables enhanced networking capabilities
- Jumbo frames require same VPC/region support
- Cluster placement groups provide lowest latency
- Enhanced networking is enabled by default on supported instances
- Network performance scales with instance size
- DPDK enables userspace packet processing
- TCP optimization is crucial for high-bandwidth applications