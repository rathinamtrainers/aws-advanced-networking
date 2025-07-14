# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is an AWS Advanced Networking training repository containing educational materials and documentation. The repository is structured as a learning resource for AWS networking concepts.

## Repository Structure

- `010-docs/` - Contains training documentation organized by topic modules
  - `010-syllabus.md` - Comprehensive 220-topic syllabus covering AWS networking from foundations to advanced topics
  - `020-foundations/` - Topics 1-10: VPC basics, subnets, gateways
  - `030-hybrid-networking/` - Topics 11-25: Direct Connect, VPN, Transit Gateway
  - `040-advanced-vpc/` - Topics 26-40: Endpoints, PrivateLink, DNS
  - Additional numbered directories for remaining modules (050-security/, 060-global-networking/, etc.)
- `README.md` - Basic repository description

## Content Architecture

The training content is organized into major learning modules:

1. **Foundations** (Topics 1-10): VPC basics, subnets, gateways, routing
2. **Hybrid Networking** (Topics 11-25): Direct Connect, VPN, Transit Gateway
3. **Advanced VPC** (Topics 26-40): Endpoints, PrivateLink, DNS, troubleshooting
4. **Security** (Topics 41-55): Firewalls, WAF, Shield, Zero Trust
5. **Global Networking** (Topics 56-65): Multi-region, edge services
6. **Load Balancing** (Topics 66-75): ALB, NLB, GWLB patterns
7. **IPv6 & Modern Design** (Topics 76-85): IPv6 implementation
8. **Monitoring & Optimization** (Topics 86-92): Performance, troubleshooting
9. **Advanced Routing** (Topics 93-112): Complex architectures
10. **Deep Dives** (Topics 113-187): Direct Connect, VPN, security, global patterns
11. **Automation & Migration** (Topics 188-220): IaC, real-world scenarios

## Development Notes

This repository currently contains only documentation and training materials. There are no build systems, package managers, or code execution requirements. All content is in Markdown format for educational purposes.

## File Naming and Organization

Content files follow a structured naming convention:
- Module directories: `XXX-module-name/` (e.g., `020-foundations/`)
- Topic files: `NNN-topic-name.md` where NNN matches the topic number in the syllabus
- Support files: `000-training-structure.md`, `001-lab-template.md`

## Working with This Repository

When working with this repository:
- Focus on documentation organization and clarity
- Maintain the structured learning progression in the syllabus
- Preserve the topic numbering system for reference consistency
- Follow the existing file naming conventions for new content
- All content should be educational and focused on AWS networking concepts