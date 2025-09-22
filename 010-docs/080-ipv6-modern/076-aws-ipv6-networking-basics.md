# Topic 76: AWS IPv6 Networking Basics

## Overview

IPv6 (Internet Protocol version 6) provides a vastly expanded address space compared to IPv4, addressing the growing demand for IP addresses in cloud environments. AWS has comprehensive IPv6 support across its networking services, enabling modern application architectures that can scale globally while maintaining security and performance.

## IPv6 Address Structure

### Address Format
- **Length**: 128 bits (vs 32 bits for IPv4)
- **Notation**: Eight groups of four hexadecimal digits separated by colons
- **Example**: `2001:0db8:85a3:0000:0000:8a2e:0370:7334`
- **Compressed**: `2001:db8:85a3::8a2e:370:7334` (omitting leading zeros and consecutive zero groups)

### Address Types in AWS
1. **Global Unicast**: Publicly routable addresses (2000::/3)
2. **Link-Local**: Auto-configured addresses for local subnet communication (fe80::/10)
3. **Unique Local**: Private addresses for internal communication (fc00::/7)

## AWS IPv6 Implementation

### Regional IPv6 CIDR Blocks
- AWS allocates `/56` IPv6 CIDR blocks from Amazon's global pool
- Each region has dedicated IPv6 address ranges
- Blocks are globally unique and publicly routable
- No overlap between regions or customers

### VPC IPv6 Support
```
VPC IPv6 CIDR: /56 (e.g., 2001:db8:1234::/56)
├── Subnet 1: /64 (2001:db8:1234:0001::/64)
├── Subnet 2: /64 (2001:db8:1234:0002::/64)
└── Subnet N: /64 (2001:db8:1234:000N::/64)
```

### Key Differences from IPv4

| Aspect | IPv4 | IPv6 |
|--------|------|------|
| Address Space | 32-bit (4.3B addresses) | 128-bit (340 undecillion) |
| NAT Required | Yes (for private networks) | No (global addressing) |
| DHCP | DHCPv4 | SLAAC or DHCPv6 |
| Address Assignment | Manual/DHCP | Auto-configuration |
| Fragmentation | Router + Host | Host only |
| Header Size | Variable (20-60 bytes) | Fixed (40 bytes) |

## AWS Services with IPv6 Support

### Compute Services
- **EC2**: Dual-stack and IPv6-only instances
- **ECS/Fargate**: IPv6 task networking
- **Lambda**: IPv6 VPC connectivity
- **Elastic Beanstalk**: IPv6 environment support

### Networking Services
- **VPC**: Native IPv6 CIDR assignment
- **ELB**: Application and Network Load Balancers
- **CloudFront**: IPv6 edge locations
- **Route 53**: AAAA record support
- **Transit Gateway**: IPv6 routing

### Storage and Database
- **S3**: IPv6 API endpoints
- **RDS**: IPv6 database connectivity
- **ElastiCache**: IPv6 cluster access
- **EFS**: IPv6 mount targets

## IPv6 Address Assignment in AWS

### Automatic Assignment (SLAAC)
```bash
# EC2 instance automatically receives:
# 1. Link-local address (fe80::/10)
# 2. Global unicast address from subnet CIDR
# 3. DNS configuration via Router Advertisement

# Example instance addresses:
ip -6 addr show
# inet6 2001:db8:1234:1::a3f/128 scope global
# inet6 fe80::1234:5678:9abc:def0/64 scope link
```

### Manual Assignment
```bash
# Assign specific IPv6 address to ENI
aws ec2 assign-ipv6-addresses \
    --network-interface-id eni-12345678 \
    --ipv6-addresses 2001:db8:1234:1::100
```

## IPv6 Routing Behavior

### Default Route Configuration
```bash
# IPv6 default route via Internet Gateway
ip -6 route show
# default via fe80::1 dev eth0 proto ra metric 1024
# 2001:db8:1234:1::/64 dev eth0 proto kernel metric 256
```

### Route Table Entries
- `::/0` - Default route (equivalent to 0.0.0.0/0 in IPv4)
- `2001:db8:1234::/56` - VPC local traffic
- Specific subnet routes automatically configured

## Security Considerations

### Network ACLs and Security Groups
```json
{
  "SecurityGroupRules": [
    {
      "IpProtocol": "tcp",
      "FromPort": 443,
      "ToPort": 443,
      "Ipv6Ranges": [
        {
          "CidrIpv6": "::/0",
          "Description": "HTTPS from anywhere IPv6"
        }
      ]
    }
  ]
}
```

### ICMPv6 Requirements
- **Neighbor Discovery**: Essential for IPv6 operation
- **Path MTU Discovery**: Required for proper fragmentation
- **Router Solicitation/Advertisement**: Auto-configuration

```json
{
  "IpProtocol": "icmpv6",
  "FromPort": -1,
  "ToPort": -1,
  "Ipv6Ranges": [{"CidrIpv6": "::/0"}]
}
```

## Performance and Scalability

### Address Space Benefits
- No NAT overhead for outbound connections
- Direct end-to-end connectivity
- Simplified routing tables
- Reduced CPU utilization on NAT devices

### Potential Challenges
- Larger packet headers (40 vs 20 bytes minimum)
- Learning curve for operations teams
- Legacy application compatibility
- Monitoring and logging adjustments

## AWS IPv6 Limitations

### Current Restrictions
- VPC Flow Logs: IPv6 support varies by destination
- Some third-party integrations may lack IPv6 support
- NAT Gateway: IPv6 traffic not supported (IPv6 is internet-routable by design)
- VPC Peering: Limited cross-region IPv6 support

### Regional Availability
```bash
# Check IPv6 support by region
aws ec2 describe-regions --query 'Regions[?Ipv6Support==`true`]'
```

## Cost Implications

### IPv6 is Free in AWS
- No additional charges for IPv6 addresses
- No data transfer premium for IPv6 traffic
- Eliminates NAT Gateway costs for outbound traffic
- Potential savings on Elastic IP address usage

## Migration Considerations

### Dual-Stack Approach
1. **Phase 1**: Enable IPv6 alongside existing IPv4
2. **Phase 2**: Update applications for IPv6 awareness
3. **Phase 3**: Optimize for IPv6-primary operation
4. **Phase 4**: Consider IPv6-only for new workloads

### Application Readiness Checklist
- [ ] Library support for IPv6 socket operations
- [ ] Database connection string compatibility
- [ ] Load balancer health check configuration
- [ ] Monitoring and alerting system updates
- [ ] DNS record management (AAAA records)
- [ ] Security policy updates

## Next Steps

In the following topics, we'll explore:
- Practical IPv6 enablement in VPC (Topic 77)
- IPv6-only architecture patterns (Topic 78)
- Security best practices for IPv6 (Topic 79)
- Real-world deployment scenarios (Topics 80-85)

## Key Takeaways

1. **Abundant Address Space**: IPv6 eliminates address scarcity concerns
2. **Simplified Architecture**: No NAT required for global connectivity
3. **AWS Native Support**: Comprehensive IPv6 integration across services
4. **Cost Effective**: No additional charges for IPv6 usage
5. **Future Ready**: Essential for modern cloud applications at scale
6. **Security Focus**: Requires updated security group and NACL configurations
7. **Dual-Stack Strategy**: Recommended approach for gradual migration