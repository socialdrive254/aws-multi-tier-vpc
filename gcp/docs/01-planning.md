# Planning & CIDR Design (GCP)

This document captures the design decisions made before any GCP resources were created. The goal is to think through the network layout on paper first, so the build phase is straightforward execution rather than guesswork.

## Cloud Platform Differences

This project was originally planned for AWS. GCP's networking model differs in two important ways:

- **VPCs are global.** In AWS, a VPC lives in one region. In GCP, a VPC spans all regions worldwide. This simplifies multi-region designs but changes how you think about scope.
- **Subnets are regional.** In AWS, a subnet lives in one Availability Zone. In GCP, a subnet spans all zones within a region. This means fewer subnets are needed — one per tier instead of one per tier per zone.

The security model also differs: AWS uses Security Groups (stateful, per-instance) and NACLs (stateless, per-subnet). GCP uses Firewall Rules (stateful, per-VPC, targeted using network tags). There is no NACL equivalent in GCP.

## Region

**europe-west1 (Belgium)** — chosen for mature service availability, broad documentation coverage, and acceptable latency from Nairobi. This is GCP's equivalent of AWS eu-west-1 (Ireland).

Zones used for high availability:
- europe-west1-b
- europe-west1-c

## VPC CIDR Block

GCP VPCs do not have an overall CIDR block the way AWS does. Instead, each subnet defines its own range independently. However, we are using the same addressing scheme as the AWS plan (10.0.0.0/16 space) so the two implementations remain consistent and could theoretically be connected later via VPN or interconnect without address conflicts.

## Subnet Layout

Three subnets, one per tier. Each subnet is regional (spanning all zones in europe-west1), so there is no need to duplicate per zone. High availability is achieved by launching instances in different zones within the same subnet.

| Subnet   | CIDR          | Region        | Purpose                       |
|----------|---------------|---------------|-------------------------------|
| public   | 10.0.1.0/24   | europe-west1  | Web servers (internet-facing) |
| private  | 10.0.11.0/24  | europe-west1  | Application servers           |
| isolated | 10.0.21.0/24  | europe-west1  | Database (no internet access) |

Each /24 provides 256 addresses. GCP reserves 4 per subnet (network address, gateway, two reserved by Google), leaving 252 usable — more than enough for this project.

## Addressing Scheme

Same scheme as the AWS plan for consistency:

- Public subnets use the 1-10 range (10.0.1.0/24)
- Private subnets use the 11-20 range (10.0.11.0/24)
- Isolated subnets use the 21-30 range (10.0.21.0/24)

Gaps between tiers leave room for expansion without disrupting the pattern.

## Traffic Flow Summary

- **Inbound internet traffic** reaches instances in the public subnet via their external IP addresses. GCP does not require an explicit Internet Gateway resource — internet access is built into the default routing if an instance has an external IP.
- **Outbound traffic from the private subnet** (for example, application servers pulling software updates) flows through Cloud NAT, which provides internet access without assigning external IPs to instances. The internet cannot initiate connections inward to private instances.
- **The isolated subnet** has no route to the internet in either direction. No external IPs are assigned, and no Cloud NAT is configured for this subnet. Database instances can only be reached by application servers in the private subnet.

## High Availability Design

Instances are deployed across two zones (europe-west1-b and europe-west1-c) within the same regional subnet. If one zone experiences a failure, instances in the other zone continue operating. This provides the same availability guarantee as the AWS multi-AZ design, with fewer resources to manage.

## Security Model (GCP-Specific)

GCP uses **firewall rules** applied at the VPC level, targeted to specific instances using **network tags**. For example:

- Instances tagged `web` receive a firewall rule allowing inbound HTTPS (port 443) from the internet
- Instances tagged `app` receive a rule allowing inbound traffic only from instances tagged `web`
- Instances tagged `db` receive a rule allowing inbound traffic only from instances tagged `app`

This tag-based approach replaces both AWS Security Groups and NACLs. Default behaviour in a custom VPC is deny-all — no traffic is allowed until you explicitly create firewall rules.

## Architecture Diagram

See [diagrams/architecture.png](../diagrams/architecture.png) for the visual representation of this design.

## Comparison: AWS vs GCP Implementation

| Concept              | AWS                          | GCP                              |
|----------------------|------------------------------|----------------------------------|
| VPC scope            | Regional                     | Global                           |
| Subnet scope         | Per Availability Zone        | Per Region (spans all zones)     |
| Number of subnets    | 6 (3 tiers x 2 AZs)         | 3 (3 tiers, regional)            |
| Internet access      | Internet Gateway (explicit)  | Built-in (external IP)           |
| NAT                  | NAT Gateway                  | Cloud NAT                        |
| Firewall (stateful)  | Security Groups              | Firewall Rules + network tags    |
| Firewall (stateless) | NACLs                        | No equivalent                    |
| Default traffic      | Allow-all in default VPC     | Deny-all in custom VPC           |

## Decisions Deferred to the Build Phase

- Exact machine types (likely e2-micro to stay within free tier)
- Database engine (likely Cloud SQL with PostgreSQL or MySQL)
- Whether to use OS Login or IAP Tunneling for administrative access (IAP preferred — no SSH keys or open ports needed)