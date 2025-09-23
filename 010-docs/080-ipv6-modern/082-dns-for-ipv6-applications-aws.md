# Topic 82: DNS for IPv6 Applications in AWS

## Overview

DNS (Domain Name System) is critical for IPv6 applications, providing name resolution and service discovery. IPv6 introduces new DNS record types, resolution patterns, and configuration considerations. This topic covers comprehensive DNS configuration for IPv6 applications on AWS, including Route 53 setup, dual-stack resolution, and performance optimization.

## IPv6 DNS Fundamentals

### IPv6 DNS Record Types

| Record Type | Purpose | Example |
|-------------|---------|---------|
| AAAA | IPv6 address resolution | `example.com AAAA 2001:db8::1` |
| A | IPv4 address resolution | `example.com A 203.0.113.1` |
| PTR | Reverse DNS (IPv6) | `1.0.0.0.8.b.d.0.1.0.0.2.ip6.arpa PTR example.com` |
| SRV | Service discovery | `_http._tcp.example.com SRV 10 5 80 web.example.com` |
| CNAME | Canonical name | `www.example.com CNAME example.com` |

### IPv6 DNS Resolution Process

```
IPv6 DNS Resolution Flow:
1. Client queries for AAAA record
2. DNS resolver checks cache
3. If not cached, queries authoritative DNS
4. Returns IPv6 address(es)
5. Client attempts connection to IPv6 address
6. Fallback to A record if IPv6 fails (Happy Eyeballs)
```

### Dual-Stack DNS Behavior

```bash
# Dual-stack DNS queries
dig example.com AAAA    # IPv6 addresses
dig example.com A       # IPv4 addresses
dig example.com ANY     # All record types

# Client behavior with both records available
# 1. Query both A and AAAA records simultaneously
# 2. Prefer IPv6 if available and working
# 3. Fall back to IPv4 if IPv6 fails
# 4. Cache successful connection type
```

## Route 53 IPv6 Configuration

### Basic AAAA Record Setup

```bash
#!/bin/bash
# Create IPv6 DNS records in Route 53

HOSTED_ZONE_ID="Z1D633PJN98FT9"
DOMAIN="example.com"

# Create AAAA record for main domain
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [
            {
                "Action": "CREATE",
                "ResourceRecordSet": {
                    "Name": "'$DOMAIN'",
                    "Type": "AAAA",
                    "TTL": 300,
                    "ResourceRecords": [
                        {"Value": "2001:db8:1234::1"},
                        {"Value": "2001:db8:1234::2"}
                    ]
                }
            }
        ]
    }'

# Create AAAA record for www subdomain
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [
            {
                "Action": "CREATE",
                "ResourceRecordSet": {
                    "Name": "www.'$DOMAIN'",
                    "Type": "AAAA",
                    "TTL": 300,
                    "ResourceRecords": [
                        {"Value": "2001:db8:1234::1"},
                        {"Value": "2001:db8:1234::2"}
                    ]
                }
            }
        ]
    }'
```

### Alias Records for AWS Resources

```bash
#!/bin/bash
# Create IPv6 alias records for AWS load balancers

# Get ALB DNS name and hosted zone
ALB_DNS=$(aws elbv2 describe-load-balancers \
    --names my-dualstack-alb \
    --query 'LoadBalancers[0].DNSName' \
    --output text)

ALB_ZONE=$(aws elbv2 describe-load-balancers \
    --names my-dualstack-alb \
    --query 'LoadBalancers[0].CanonicalHostedZoneId' \
    --output text)

# Create alias record that includes both IPv4 and IPv6
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [
            {
                "Action": "CREATE",
                "ResourceRecordSet": {
                    "Name": "app.'$DOMAIN'",
                    "Type": "A",
                    "AliasTarget": {
                        "DNSName": "'$ALB_DNS'",
                        "HostedZoneId": "'$ALB_ZONE'",
                        "EvaluateTargetHealth": true
                    }
                }
            },
            {
                "Action": "CREATE",
                "ResourceRecordSet": {
                    "Name": "app.'$DOMAIN'",
                    "Type": "AAAA",
                    "AliasTarget": {
                        "DNSName": "'$ALB_DNS'",
                        "HostedZoneId": "'$ALB_ZONE'",
                        "EvaluateTargetHealth": true
                    }
                }
            }
        ]
    }'
```

### CloudFront IPv6 DNS Configuration

```bash
#!/bin/bash
# Configure DNS for CloudFront with IPv6 support

# Get CloudFront distribution details
CF_DOMAIN=$(aws cloudfront list-distributions \
    --query 'DistributionList.Items[?Comment==`IPv6 Demo Distribution`].DomainName' \
    --output text)

CF_ZONE="Z2FDTNDATAQYW2"  # CloudFront hosted zone ID

# Create alias records for CloudFront
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [
            {
                "Action": "CREATE",
                "ResourceRecordSet": {
                    "Name": "cdn.'$DOMAIN'",
                    "Type": "A",
                    "AliasTarget": {
                        "DNSName": "'$CF_DOMAIN'",
                        "HostedZoneId": "'$CF_ZONE'",
                        "EvaluateTargetHealth": false
                    }
                }
            },
            {
                "Action": "CREATE",
                "ResourceRecordSet": {
                    "Name": "cdn.'$DOMAIN'",
                    "Type": "AAAA",
                    "AliasTarget": {
                        "DNSName": "'$CF_DOMAIN'",
                        "HostedZoneId": "'$CF_ZONE'",
                        "EvaluateTargetHealth": false
                    }
                }
            }
        ]
    }'
```

## Advanced DNS Configurations

### Health Checks with IPv6

```bash
#!/bin/bash
# Create Route 53 health checks for IPv6 endpoints

create_ipv6_health_check() {
    local endpoint_ipv6=$1
    local endpoint_name=$2

    HEALTH_CHECK_ID=$(aws route53 create-health-check \
        --caller-reference "ipv6-health-$(date +%s)" \
        --health-check-config '{
            "Type": "HTTPS",
            "ResourcePath": "/health",
            "FullyQualifiedDomainName": "'$endpoint_name'",
            "Port": 443,
            "RequestInterval": 30,
            "FailureThreshold": 3,
            "IPAddress": "'$endpoint_ipv6'"
        }' \
        --query 'HealthCheck.Id' \
        --output text)

    echo "Health check created: $HEALTH_CHECK_ID for $endpoint_ipv6"
    echo $HEALTH_CHECK_ID
}

# Create health checks for IPv6 endpoints
HEALTH_CHECK_1=$(create_ipv6_health_check "2001:db8:1234::1" "web1.example.com")
HEALTH_CHECK_2=$(create_ipv6_health_check "2001:db8:1234::2" "web2.example.com")

# Create weighted routing with health checks
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [
            {
                "Action": "CREATE",
                "ResourceRecordSet": {
                    "Name": "api.'$DOMAIN'",
                    "Type": "AAAA",
                    "SetIdentifier": "IPv6-Primary",
                    "Weight": 100,
                    "TTL": 60,
                    "ResourceRecords": [{"Value": "2001:db8:1234::1"}],
                    "HealthCheckId": "'$HEALTH_CHECK_1'"
                }
            },
            {
                "Action": "CREATE",
                "ResourceRecordSet": {
                    "Name": "api.'$DOMAIN'",
                    "Type": "AAAA",
                    "SetIdentifier": "IPv6-Secondary",
                    "Weight": 50,
                    "TTL": 60,
                    "ResourceRecords": [{"Value": "2001:db8:1234::2"}],
                    "HealthCheckId": "'$HEALTH_CHECK_2'"
                }
            }
        ]
    }'
```

### Geolocation-Based IPv6 Routing

```bash
#!/bin/bash
# Configure geolocation routing for IPv6

# Create geolocation records for different regions
aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [
            {
                "Action": "CREATE",
                "ResourceRecordSet": {
                    "Name": "global.'$DOMAIN'",
                    "Type": "AAAA",
                    "SetIdentifier": "US-East",
                    "GeoLocation": {
                        "CountryCode": "US",
                        "SubdivisionCode": "VA"
                    },
                    "TTL": 300,
                    "ResourceRecords": [{"Value": "2001:db8:us-east::1"}]
                }
            },
            {
                "Action": "CREATE",
                "ResourceRecordSet": {
                    "Name": "global.'$DOMAIN'",
                    "Type": "AAAA",
                    "SetIdentifier": "EU-West",
                    "GeoLocation": {
                        "ContinentCode": "EU"
                    },
                    "TTL": 300,
                    "ResourceRecords": [{"Value": "2001:db8:eu-west::1"}]
                }
            },
            {
                "Action": "CREATE",
                "ResourceRecordSet": {
                    "Name": "global.'$DOMAIN'",
                    "Type": "AAAA",
                    "SetIdentifier": "Default",
                    "GeoLocation": {
                        "CountryCode": "*"
                    },
                    "TTL": 300,
                    "ResourceRecords": [{"Value": "2001:db8:default::1"}]
                }
            }
        ]
    }'
```

### Latency-Based Routing

```bash
#!/bin/bash
# Configure latency-based routing for optimal IPv6 performance

regions=("us-east-1" "eu-west-1" "ap-southeast-1")
ipv6_endpoints=("2001:db8:us::1" "2001:db8:eu::1" "2001:db8:ap::1")

for i in "${!regions[@]}"; do
    region="${regions[$i]}"
    endpoint="${ipv6_endpoints[$i]}"

    aws route53 change-resource-record-sets \
        --hosted-zone-id $HOSTED_ZONE_ID \
        --change-batch '{
            "Changes": [
                {
                    "Action": "CREATE",
                    "ResourceRecordSet": {
                        "Name": "fast.'$DOMAIN'",
                        "Type": "AAAA",
                        "SetIdentifier": "'$region'",
                        "Region": "'$region'",
                        "TTL": 60,
                        "ResourceRecords": [{"Value": "'$endpoint'"}]
                    }
                }
            ]
        }'
done
```

## DNS Server Configuration

### BIND9 IPv6 Configuration

```bash
# /etc/bind/named.conf.options
options {
    directory "/var/cache/bind";

    # Listen on both IPv4 and IPv6
    listen-on { any; };
    listen-on-v6 { any; };

    # Enable IPv6 for queries
    query-source address * port 53;
    query-source-v6 address * port 53;

    # DNS64 configuration for IPv6-only clients
    dns64 64:ff9b::/96 {
        clients { any; };
        mapped { !rfc1918; any; };
        exclude { 64:ff9b::/96; ::ffff:0000:0000/96; };
        suffix ::;
    };

    recursion yes;
    allow-recursion { any; };

    dnssec-enable yes;
    dnssec-validation yes;
};

# Zone configuration for IPv6
zone "example.com" {
    type master;
    file "/etc/bind/zones/example.com.zone";
    allow-transfer { any; };
};

# IPv6 reverse zone
zone "4.3.2.1.8.b.d.0.1.0.0.2.ip6.arpa" {
    type master;
    file "/etc/bind/zones/reverse-ipv6.zone";
};
```

### Zone File Configuration

```bash
# /etc/bind/zones/example.com.zone
$TTL 300
@   IN  SOA ns1.example.com. admin.example.com. (
            2024011501  ; Serial
            3600        ; Refresh
            1800        ; Retry
            1209600     ; Expire
            300         ; Minimum TTL
)

; Name servers
@       IN  NS      ns1.example.com.
@       IN  NS      ns2.example.com.

; IPv4 and IPv6 addresses for name servers
ns1     IN  A       203.0.113.1
ns1     IN  AAAA    2001:db8:1234::1
ns2     IN  A       203.0.113.2
ns2     IN  AAAA    2001:db8:1234::2

; Main domain - dual stack
@       IN  A       203.0.113.10
@       IN  AAAA    2001:db8:1234::10

; Web services
www     IN  A       203.0.113.10
www     IN  AAAA    2001:db8:1234::10
web1    IN  AAAA    2001:db8:1234::11
web2    IN  AAAA    2001:db8:1234::12

; API endpoints
api     IN  A       203.0.113.20
api     IN  AAAA    2001:db8:1234::20

; Mail servers
@       IN  MX  10  mail.example.com.
mail    IN  A       203.0.113.30
mail    IN  AAAA    2001:db8:1234::30

; Service discovery
_http._tcp  IN  SRV  10 5 80  web1.example.com.
_http._tcp  IN  SRV  10 5 80  web2.example.com.
_https._tcp IN  SRV  10 5 443 web1.example.com.
_https._tcp IN  SRV  10 5 443 web2.example.com.
```

### Reverse DNS Configuration

```bash
# /etc/bind/zones/reverse-ipv6.zone
$TTL 300
@   IN  SOA ns1.example.com. admin.example.com. (
            2024011501  ; Serial
            3600        ; Refresh
            1800        ; Retry
            1209600     ; Expire
            300         ; Minimum TTL
)

@       IN  NS      ns1.example.com.
@       IN  NS      ns2.example.com.

; PTR records for IPv6 addresses
; 2001:db8:1234::1 -> 1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.4.3.2.1.8.b.d.0.1.0.0.2.ip6.arpa
1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0     IN  PTR  ns1.example.com.
2.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0     IN  PTR  ns2.example.com.
0.1.0.0.0.0.0.0.0.0.0.0.0.0.0.0     IN  PTR  example.com.
1.1.0.0.0.0.0.0.0.0.0.0.0.0.0.0     IN  PTR  web1.example.com.
2.1.0.0.0.0.0.0.0.0.0.0.0.0.0.0     IN  PTR  web2.example.com.
```

## Application Integration

### Python DNS Configuration

```python
#!/usr/bin/env python3
# Python application with IPv6 DNS configuration

import socket
import dns.resolver
import dns.reversename
import ipaddress
from typing import List, Tuple, Optional

class IPv6DNSClient:
    def __init__(self, prefer_ipv6: bool = True):
        self.prefer_ipv6 = prefer_ipv6
        self.resolver = dns.resolver.Resolver()

        # Configure resolver for IPv6
        self.resolver.nameservers = [
            '2001:4860:4860::8888',  # Google DNS IPv6
            '2001:4860:4860::8844',  # Google DNS IPv6
            '8.8.8.8',               # Google DNS IPv4 fallback
            '8.8.4.4'                # Google DNS IPv4 fallback
        ]

    def resolve_dual_stack(self, hostname: str) -> Tuple[List[str], List[str]]:
        \"\"\"Resolve both IPv4 and IPv6 addresses for a hostname\"\"\"
        ipv4_addresses = []
        ipv6_addresses = []

        try:
            # Query AAAA records (IPv6)
            aaaa_result = self.resolver.resolve(hostname, 'AAAA')
            ipv6_addresses = [str(rdata) for rdata in aaaa_result]
        except dns.resolver.NXDOMAIN:
            print(f\"No AAAA records found for {hostname}\")
        except Exception as e:
            print(f\"Error resolving AAAA for {hostname}: {e}\")

        try:
            # Query A records (IPv4)
            a_result = self.resolver.resolve(hostname, 'A')
            ipv4_addresses = [str(rdata) for rdata in a_result]
        except dns.resolver.NXDOMAIN:
            print(f\"No A records found for {hostname}\")
        except Exception as e:
            print(f\"Error resolving A for {hostname}: {e}\")

        return ipv4_addresses, ipv6_addresses

    def get_preferred_addresses(self, hostname: str) -> List[str]:
        \"\"\"Get addresses in preferred order (IPv6 first if enabled)\"\"\"
        ipv4_addrs, ipv6_addrs = self.resolve_dual_stack(hostname)

        if self.prefer_ipv6:
            return ipv6_addrs + ipv4_addrs
        else:
            return ipv4_addrs + ipv6_addrs

    def reverse_lookup(self, ip_address: str) -> Optional[str]:
        \"\"\"Perform reverse DNS lookup\"\"\"
        try:
            ip_obj = ipaddress.ip_address(ip_address)
            reverse_name = dns.reversename.from_address(ip_address)
            result = self.resolver.resolve(reverse_name, 'PTR')
            return str(result[0])
        except Exception as e:
            print(f\"Reverse lookup failed for {ip_address}: {e}\")
            return None

    def check_srv_records(self, service: str, protocol: str, domain: str) -> List[dict]:
        \"\"\"Query SRV records for service discovery\"\"\"
        srv_name = f\"_{service}._{protocol}.{domain}\"

        try:
            result = self.resolver.resolve(srv_name, 'SRV')
            srv_records = []

            for rdata in result:
                srv_record = {
                    'priority': rdata.priority,
                    'weight': rdata.weight,
                    'port': rdata.port,
                    'target': str(rdata.target)
                }
                srv_records.append(srv_record)

            return sorted(srv_records, key=lambda x: (x['priority'], -x['weight']))
        except Exception as e:
            print(f\"SRV lookup failed for {srv_name}: {e}\")
            return []

    def test_connectivity(self, hostname: str, port: int = 80) -> dict:
        \"\"\"Test connectivity to both IPv4 and IPv6 addresses\"\"\"
        ipv4_addrs, ipv6_addrs = self.resolve_dual_stack(hostname)
        results = {'ipv4': [], 'ipv6': []}

        # Test IPv6 connectivity
        for addr in ipv6_addrs:
            try:
                sock = socket.socket(socket.AF_INET6, socket.SOCK_STREAM)
                sock.settimeout(5)
                start_time = time.time()
                sock.connect((addr, port))
                connect_time = time.time() - start_time
                sock.close()
                results['ipv6'].append({
                    'address': addr,
                    'status': 'success',
                    'time': connect_time
                })
            except Exception as e:
                results['ipv6'].append({
                    'address': addr,
                    'status': 'failed',
                    'error': str(e)
                })

        # Test IPv4 connectivity
        for addr in ipv4_addrs:
            try:
                sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
                sock.settimeout(5)
                start_time = time.time()
                sock.connect((addr, port))
                connect_time = time.time() - start_time
                sock.close()
                results['ipv4'].append({
                    'address': addr,
                    'status': 'success',
                    'time': connect_time
                })
            except Exception as e:
                results['ipv4'].append({
                    'address': addr,
                    'status': 'failed',
                    'error': str(e)
                })

        return results

# Usage example
if __name__ == \"__main__\":
    import time

    dns_client = IPv6DNSClient(prefer_ipv6=True)

    # Test dual-stack resolution
    hostname = \"example.com\"
    print(f\"Resolving {hostname}...\")
    ipv4_addrs, ipv6_addrs = dns_client.resolve_dual_stack(hostname)
    print(f\"IPv4 addresses: {ipv4_addrs}\")
    print(f\"IPv6 addresses: {ipv6_addrs}\")

    # Test connectivity
    print(f\"\\nTesting connectivity to {hostname}:80\")
    connectivity = dns_client.test_connectivity(hostname, 80)
    print(f\"IPv6 results: {connectivity['ipv6']}\")
    print(f\"IPv4 results: {connectivity['ipv4']}\")

    # Test service discovery
    print(f\"\\nChecking SRV records for HTTP service...\")
    srv_records = dns_client.check_srv_records('http', 'tcp', hostname)
    for record in srv_records:
        print(f\"  {record['target']}:{record['port']} (priority: {record['priority']}, weight: {record['weight']})\")
```

### Node.js DNS Configuration

```javascript
// Node.js IPv6 DNS implementation
const dns = require('dns').promises;
const net = require('net');

class IPv6DNSResolver {
    constructor(options = {}) {
        this.preferIPv6 = options.preferIPv6 || true;
        this.timeout = options.timeout || 5000;

        // Configure DNS servers
        dns.setServers([
            '2001:4860:4860::8888',  // Google IPv6
            '2001:4860:4860::8844',  // Google IPv6
            '8.8.8.8',               // Google IPv4
            '8.8.4.4'                // Google IPv4
        ]);
    }

    async resolveDualStack(hostname) {
        try {
            const [ipv4Results, ipv6Results] = await Promise.allSettled([
                dns.resolve4(hostname),
                dns.resolve6(hostname)
            ]);

            const ipv4Addresses = ipv4Results.status === 'fulfilled' ? ipv4Results.value : [];
            const ipv6Addresses = ipv6Results.status === 'fulfilled' ? ipv6Results.value : [];

            return {
                ipv4: ipv4Addresses,
                ipv6: ipv6Addresses
            };
        } catch (error) {
            console.error(`Error resolving ${hostname}:`, error);
            return { ipv4: [], ipv6: [] };
        }
    }

    async getPreferredAddresses(hostname) {
        const { ipv4, ipv6 } = await this.resolveDualStack(hostname);

        if (this.preferIPv6) {
            return [...ipv6, ...ipv4];
        } else {
            return [...ipv4, ...ipv6];
        }
    }

    async reverseLookup(ipAddress) {
        try {
            const hostnames = await dns.reverse(ipAddress);
            return hostnames[0];
        } catch (error) {
            console.error(`Reverse lookup failed for ${ipAddress}:`, error);
            return null;
        }
    }

    async resolveSRV(service, protocol, domain) {
        const srvName = `_${service}._${protocol}.${domain}`;

        try {
            const records = await dns.resolveSrv(srvName);
            return records.sort((a, b) => {
                if (a.priority !== b.priority) {
                    return a.priority - b.priority;
                }
                return b.weight - a.weight;
            });
        } catch (error) {
            console.error(`SRV lookup failed for ${srvName}:`, error);
            return [];
        }
    }

    async testConnectivity(hostname, port = 80) {
        const { ipv4, ipv6 } = await this.resolveDualStack(hostname);
        const results = { ipv4: [], ipv6: [] };

        // Test IPv6 addresses
        for (const address of ipv6) {
            try {
                const startTime = Date.now();
                await this.testConnection(address, port, 6);
                const connectTime = Date.now() - startTime;
                results.ipv6.push({
                    address,
                    status: 'success',
                    time: connectTime
                });
            } catch (error) {
                results.ipv6.push({
                    address,
                    status: 'failed',
                    error: error.message
                });
            }
        }

        // Test IPv4 addresses
        for (const address of ipv4) {
            try {
                const startTime = Date.now();
                await this.testConnection(address, port, 4);
                const connectTime = Date.now() - startTime;
                results.ipv4.push({
                    address,
                    status: 'success',
                    time: connectTime
                });
            } catch (error) {
                results.ipv4.push({
                    address,
                    status: 'failed',
                    error: error.message
                });
            }
        }

        return results;
    }

    testConnection(address, port, family) {
        return new Promise((resolve, reject) => {
            const socket = new net.Socket();
            const timeout = setTimeout(() => {
                socket.destroy();
                reject(new Error('Connection timeout'));
            }, this.timeout);

            socket.connect(port, address, () => {
                clearTimeout(timeout);
                socket.destroy();
                resolve();
            });

            socket.on('error', (error) => {
                clearTimeout(timeout);
                reject(error);
            });
        });
    }

    async monitorDNSPerformance(hostname, iterations = 10) {
        const results = [];

        for (let i = 0; i < iterations; i++) {
            const startTime = Date.now();
            await this.resolveDualStack(hostname);
            const queryTime = Date.now() - startTime;

            results.push({
                iteration: i + 1,
                time: queryTime,
                timestamp: new Date().toISOString()
            });

            // Wait 1 second between queries
            await new Promise(resolve => setTimeout(resolve, 1000));
        }

        const avgTime = results.reduce((sum, r) => sum + r.time, 0) / results.length;
        const minTime = Math.min(...results.map(r => r.time));
        const maxTime = Math.max(...results.map(r => r.time));

        return {
            results,
            statistics: {
                average: avgTime,
                minimum: minTime,
                maximum: maxTime,
                total_queries: iterations
            }
        };
    }
}

// Usage example
async function main() {
    const resolver = new IPv6DNSResolver({ preferIPv6: true });

    const hostname = 'example.com';

    console.log(`Resolving ${hostname}...`);
    const addresses = await resolver.resolveDualStack(hostname);
    console.log('IPv4 addresses:', addresses.ipv4);
    console.log('IPv6 addresses:', addresses.ipv6);

    console.log(`\\nTesting connectivity to ${hostname}:80`);
    const connectivity = await resolver.testConnectivity(hostname, 80);
    console.log('IPv6 results:', connectivity.ipv6);
    console.log('IPv4 results:', connectivity.ipv4);

    console.log('\\nChecking SRV records for HTTP service...');
    const srvRecords = await resolver.resolveSRV('http', 'tcp', hostname);
    srvRecords.forEach(record => {
        console.log(`  ${record.name}:${record.port} (priority: ${record.priority}, weight: ${record.weight})`);
    });

    console.log('\\nMonitoring DNS performance...');
    const performance = await resolver.monitorDNSPerformance(hostname, 5);
    console.log('Performance statistics:', performance.statistics);
}

main().catch(console.error);
```

## Monitoring and Troubleshooting

### DNS Performance Monitoring

```bash
#!/bin/bash
# Monitor IPv6 DNS performance and resolution

monitor_dns_performance() {
    local hostname=$1
    local iterations=${2:-10}

    echo \"Monitoring DNS performance for $hostname\"
    echo \"=========================================\"

    # Test AAAA record resolution times
    echo \"IPv6 (AAAA) Record Resolution Times:\"
    for i in $(seq 1 $iterations); do
        start_time=$(date +%s.%N)
        dig +short $hostname AAAA > /dev/null
        end_time=$(date +%s.%N)
        query_time=$(echo \"$end_time - $start_time\" | bc)
        echo \"Query $i: ${query_time}s\"
    done

    echo \"\"

    # Test A record resolution times
    echo \"IPv4 (A) Record Resolution Times:\"
    for i in $(seq 1 $iterations); do
        start_time=$(date +%s.%N)
        dig +short $hostname A > /dev/null
        end_time=$(date +%s.%N)
        query_time=$(echo \"$end_time - $start_time\" | bc)
        echo \"Query $i: ${query_time}s\"
    done

    echo \"\"

    # Test different DNS servers
    echo \"Comparing DNS Server Performance:\"
    dns_servers=(\"8.8.8.8\" \"1.1.1.1\" \"208.67.222.222\")

    for server in \"${dns_servers[@]}\"; do
        echo \"Testing $server:\"
        start_time=$(date +%s.%N)
        dig @$server +short $hostname AAAA > /dev/null
        end_time=$(date +%s.%N)
        query_time=$(echo \"$end_time - $start_time\" | bc)
        echo \"  AAAA query time: ${query_time}s\"
    done
}

# DNS troubleshooting function
troubleshoot_dns() {
    local hostname=$1

    echo \"DNS Troubleshooting for $hostname\"
    echo \"=================================\"

    # Check if domain exists
    echo \"1. Checking if domain exists...\"
    if dig +short $hostname SOA > /dev/null; then
        echo \"   ✅ Domain exists\"
    else
        echo \"   ❌ Domain does not exist or is not configured\"
        return 1
    fi

    # Check A records
    echo \"2. Checking A records...\"
    A_RECORDS=$(dig +short $hostname A)
    if [ ! -z \"$A_RECORDS\" ]; then
        echo \"   ✅ A records found:\"
        echo \"$A_RECORDS\" | sed 's/^/      /'
    else
        echo \"   ❌ No A records found\"
    fi

    # Check AAAA records
    echo \"3. Checking AAAA records...\"
    AAAA_RECORDS=$(dig +short $hostname AAAA)
    if [ ! -z \"$AAAA_RECORDS\" ]; then
        echo \"   ✅ AAAA records found:\"
        echo \"$AAAA_RECORDS\" | sed 's/^/      /'
    else
        echo \"   ❌ No AAAA records found\"
    fi

    # Check name servers
    echo \"4. Checking name servers...\"
    NS_RECORDS=$(dig +short $hostname NS)
    if [ ! -z \"$NS_RECORDS\" ]; then
        echo \"   ✅ Name servers found:\"
        echo \"$NS_RECORDS\" | sed 's/^/      /'

        # Test each name server
        for ns in $NS_RECORDS; do
            echo \"   Testing $ns:\"
            if dig @$ns +short $hostname AAAA > /dev/null 2>&1; then
                echo \"      ✅ $ns responds to AAAA queries\"
            else
                echo \"      ❌ $ns does not respond to AAAA queries\"
            fi
        done
    else
        echo \"   ❌ No name servers found\"
    fi

    # Check reverse DNS
    echo \"5. Checking reverse DNS...\"
    if [ ! -z \"$AAAA_RECORDS\" ]; then
        for ipv6 in $AAAA_RECORDS; do
            reverse=$(dig +short -x $ipv6)
            if [ ! -z \"$reverse\" ]; then
                echo \"   ✅ $ipv6 -> $reverse\"
            else
                echo \"   ❌ No reverse DNS for $ipv6\"
            fi
        done
    fi

    # Check DNSSEC
    echo \"6. Checking DNSSEC...\"
    if dig +dnssec +short $hostname AAAA | grep -q \"RRSIG\"; then
        echo \"   ✅ DNSSEC signatures present\"
    else
        echo \"   ⚠️  No DNSSEC signatures found\"
    fi
}

# Usage
monitor_dns_performance \"example.com\" 5
troubleshoot_dns \"example.com\"
```

### CloudWatch DNS Metrics

```python
#!/usr/bin/env python3
# CloudWatch monitoring for Route 53 DNS queries

import boto3
from datetime import datetime, timedelta
import json

class Route53Monitor:
    def __init__(self):
        self.route53 = boto3.client('route53')
        self.cloudwatch = boto3.client('cloudwatch')

    def get_hosted_zone_metrics(self, hosted_zone_id, hours=24):
        \"\"\"Get Route 53 query metrics for a hosted zone\"\"\"
        end_time = datetime.utcnow()
        start_time = end_time - timedelta(hours=hours)

        metrics = ['QueryCount', 'UniqueLocations', 'ResponseTime']
        results = {}

        for metric in metrics:
            try:
                response = self.cloudwatch.get_metric_statistics(
                    Namespace='AWS/Route53',
                    MetricName=metric,
                    Dimensions=[
                        {
                            'Name': 'HostedZoneId',
                            'Value': hosted_zone_id
                        }
                    ],
                    StartTime=start_time,
                    EndTime=end_time,
                    Period=3600,
                    Statistics=['Sum', 'Average', 'Maximum']
                )
                results[metric] = response['Datapoints']
            except Exception as e:
                print(f\"Error getting {metric}: {e}\")
                results[metric] = []

        return results

    def analyze_query_patterns(self, hosted_zone_id):
        \"\"\"Analyze DNS query patterns for IPv6 vs IPv4\"\"\"
        print(f\"Analyzing query patterns for hosted zone: {hosted_zone_id}\")
        print(\"=\" * 60)

        # Get zone information
        zone_info = self.route53.get_hosted_zone(Id=hosted_zone_id)
        zone_name = zone_info['HostedZone']['Name']

        print(f\"Zone: {zone_name}\")

        # Get metrics
        metrics = self.get_hosted_zone_metrics(hosted_zone_id)

        for metric_name, datapoints in metrics.items():
            if datapoints:
                latest = sorted(datapoints, key=lambda x: x['Timestamp'])[-1]
                print(f\"{metric_name}:\")
                print(f\"  Latest Value: {latest.get('Sum', latest.get('Average', 0))}\")
                print(f\"  Timestamp: {latest['Timestamp']}\")
                print()

    def create_dns_dashboard(self, hosted_zone_id, dashboard_name):
        \"\"\"Create CloudWatch dashboard for DNS monitoring\"\"\"
        dashboard_body = {
            \"widgets\": [
                {
                    \"type\": \"metric\",
                    \"x\": 0,
                    \"y\": 0,
                    \"width\": 12,
                    \"height\": 6,
                    \"properties\": {
                        \"metrics\": [
                            [\"AWS/Route53\", \"QueryCount\", \"HostedZoneId\", hosted_zone_id]
                        ],
                        \"period\": 300,
                        \"stat\": \"Sum\",
                        \"region\": \"us-east-1\",
                        \"title\": \"DNS Query Count\"
                    }
                },
                {
                    \"type\": \"metric\",
                    \"x\": 0,
                    \"y\": 6,
                    \"width\": 12,
                    \"height\": 6,
                    \"properties\": {
                        \"metrics\": [
                            [\"AWS/Route53\", \"ResponseTime\", \"HostedZoneId\", hosted_zone_id]
                        ],
                        \"period\": 300,
                        \"stat\": \"Average\",
                        \"region\": \"us-east-1\",
                        \"title\": \"DNS Response Time\"
                    }
                }
            ]
        }

        self.cloudwatch.put_dashboard(
            DashboardName=dashboard_name,
            DashboardBody=json.dumps(dashboard_body)
        )

        print(f\"Dashboard '{dashboard_name}' created successfully\")

    def setup_dns_alarms(self, hosted_zone_id):
        \"\"\"Set up CloudWatch alarms for DNS issues\"\"\"

        # High query count alarm
        self.cloudwatch.put_metric_alarm(
            AlarmName=f\"Route53-HighQueryCount-{hosted_zone_id}\",
            ComparisonOperator='GreaterThanThreshold',
            EvaluationPeriods=2,
            MetricName='QueryCount',
            Namespace='AWS/Route53',
            Period=300,
            Statistic='Sum',
            Threshold=10000.0,
            ActionsEnabled=True,
            AlarmDescription='High DNS query count detected',
            Dimensions=[
                {
                    'Name': 'HostedZoneId',
                    'Value': hosted_zone_id
                }
            ]
        )

        # High response time alarm
        self.cloudwatch.put_metric_alarm(
            AlarmName=f\"Route53-HighResponseTime-{hosted_zone_id}\",
            ComparisonOperator='GreaterThanThreshold',
            EvaluationPeriods=3,
            MetricName='ResponseTime',
            Namespace='AWS/Route53',
            Period=300,
            Statistic='Average',
            Threshold=1000.0,
            ActionsEnabled=True,
            AlarmDescription='High DNS response time detected',
            Dimensions=[
                {
                    'Name': 'HostedZoneId',
                    'Value': hosted_zone_id
                }
            ]
        )

        print(\"DNS monitoring alarms created\")

# Usage
if __name__ == \"__main__\":
    import sys
    if len(sys.argv) != 2:
        print(\"Usage: python3 route53_monitor.py <hosted-zone-id>\")
        sys.exit(1)

    hosted_zone_id = sys.argv[1]
    monitor = Route53Monitor()
    monitor.analyze_query_patterns(hosted_zone_id)
    monitor.create_dns_dashboard(hosted_zone_id, f\"DNS-Monitor-{hosted_zone_id}\")
    monitor.setup_dns_alarms(hosted_zone_id)
```

## Best Practices and Optimization

### DNS Caching Strategies

```bash
#!/bin/bash
# Optimize DNS caching for IPv6 applications

optimize_dns_caching() {
    echo \"DNS Caching Optimization for IPv6\"
    echo \"==================================\"

    # Configure systemd-resolved for optimal IPv6 DNS
    cat > /etc/systemd/resolved.conf << EOF
[Resolve]
DNS=2001:4860:4860::8888 2001:4860:4860::8844 8.8.8.8 8.8.4.4
FallbackDNS=1.1.1.1 1.0.0.1
Domains=~.
DNSSEC=yes
DNSOverTLS=yes
Cache=yes
CacheFromLocalhost=no
DNSStubListener=yes
MulticastDNS=yes
LLMNR=yes
EOF

    # Restart systemd-resolved
    systemctl restart systemd-resolved

    # Configure nscd for aggressive caching
    cat > /etc/nscd.conf << EOF
# Enable aggressive DNS caching
enable-cache            hosts           yes
positive-time-to-live   hosts           3600
negative-time-to-live   hosts           60
suggested-size          hosts           211
check-files             hosts           yes
persistent              hosts           yes
shared                  hosts           yes
max-db-size             hosts           33554432

# IPv6 specific optimizations
enable-cache            ipv6            yes
positive-time-to-live   ipv6            3600
negative-time-to-live   ipv6            60
EOF

    # Start nscd
    systemctl enable nscd
    systemctl restart nscd

    echo \"DNS caching optimizations applied\"
}

# Application-level DNS caching
configure_application_dns() {
    echo \"Application DNS Configuration Tips:\"
    echo \"==================================\"
    echo \"1. Use connection pooling with DNS resolution caching\"
    echo \"2. Implement Happy Eyeballs (RFC 8305) for dual-stack\"
    echo \"3. Cache successful IP addresses and prefer them\"
    echo \"4. Monitor DNS resolution times and failures\"
    echo \"5. Use TTL-aware caching to respect DNS record TTLs\"
    echo \"6. Implement DNS-over-HTTPS (DoH) or DNS-over-TLS (DoT)\"
}

optimize_dns_caching
configure_application_dns
```

### Performance Tuning

```yaml
# DNS Performance Tuning Guidelines
DNS_Performance_Optimization:
  Record_Configuration:
    TTL_Strategy:
      - Short_TTL_During_Changes: 60-300_seconds
      - Normal_Operations: 3600_seconds
      - Static_Records: 86400_seconds

    Record_Types:
      - Use_AAAA_Records: "Required for IPv6"
      - Implement_SRV_Records: "For service discovery"
      - Configure_CNAME_Carefully: "Avoid CNAME chains"

  Client_Optimization:
    Resolution_Strategy:
      - Parallel_Queries: "Query A and AAAA simultaneously"
      - Connection_Attempt_Timing: "IPv6 first, IPv4 after 300ms"
      - Failure_Detection: "Fast failure detection and fallback"

    Caching_Strategy:
      - Respect_TTL_Values: "Honor DNS record TTLs"
      - Negative_Caching: "Cache NXDOMAIN responses"
      - Connection_Caching: "Cache successful connections"

  Monitoring:
    Key_Metrics:
      - Query_Response_Time: "< 100ms target"
      - Cache_Hit_Ratio: "> 80% target"
      - Failure_Rate: "< 1% target"
      - IPv6_vs_IPv4_Usage: "Track adoption"

    Alerting:
      - High_Response_Times: "> 500ms"
      - High_Failure_Rates: "> 5%"
      - DNS_Server_Failures: "Immediate alert"
```

## Conclusion

DNS configuration for IPv6 applications requires careful planning and implementation across multiple layers:

### Key Takeaways

1. **Dual-Stack DNS**: Configure both A and AAAA records for maximum compatibility
2. **Route 53 Integration**: Leverage AWS Route 53 for scalable, reliable IPv6 DNS
3. **Health Checks**: Implement health checks for IPv6 endpoints
4. **Performance Monitoring**: Monitor DNS resolution times and patterns
5. **Application Integration**: Implement Happy Eyeballs and intelligent fallback
6. **Caching Optimization**: Use appropriate TTLs and caching strategies
7. **Security**: Implement DNSSEC and secure DNS protocols

### Best Practices Summary

- Use parallel A and AAAA queries for optimal performance
- Implement proper error handling and fallback mechanisms
- Monitor IPv6 adoption rates and performance metrics
- Cache DNS results appropriately while respecting TTLs
- Configure reverse DNS (PTR records) for IPv6 addresses
- Use service discovery (SRV records) for scalable architectures
- Test DNS configuration thoroughly across different scenarios

Proper DNS configuration is essential for successful IPv6 deployment, ensuring applications can discover and connect to services efficiently while maintaining backward compatibility with IPv4-only clients.