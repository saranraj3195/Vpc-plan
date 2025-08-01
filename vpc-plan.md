# AWS VPC Architecture – Planning Document

---

## 1. Objectives & Scope

**Goal**: Design a highly available, secure, and well‑segmented AWS Virtual Private Cloud (VPC) with:

- 3 Public subnets (one per Availability Zone)  
- 3 Private (application) subnets (one per AZ)  
- 3 Database (isolated) subnets (one per AZ)

**Key Features:**

- Internet traffic supported via **Internet Gateway (IGW)**
- **Outbound-only** traffic from private layer via **NAT Gateways**
- **No internet access** for database subnets

**Environment Assumptions:**

- Multi-AZ deployment across three zones (e.g. AZ-A, AZ-B, AZ-C)
- Single VPC with IPv4 CIDR range (e.g. `10.0.0.0/16`)
- Security and resilience prioritized

---

## 2. High-Level Architecture

### VPC Configuration

- **CIDR block**: Single IPv4 (e.g. `10.0.0.0/16`)
- **DNS**: Hostnames and resolution enabled

### Subnet Layers

- **Public Subnets**:  
  - Internet access (inbound/outbound)  
  - Hosts NAT Gateways, Load Balancers, or Bastion Hosts

- **Private (App) Subnets**:  
  - No inbound internet  
  - Outbound internet via NAT Gateway in same AZ

- **Database Subnets**:  
  - Fully isolated (no outbound internet)  
  - Used for RDS or other data services

### Gateways

- **1 Internet Gateway (IGW)**: Attached to the VPC
- **3 NAT Gateways**: One per AZ, in respective public subnet

### Routing

- **Public Route Tables**: Default route `0.0.0.0/0` → IGW  
- **Private Route Tables**: Default route → NAT Gateway (same AZ)  
- **DB Subnets**: Local routing only (no `0.0.0.0/0`)

---

## 3. Subnet & CIDR Planning

| Tier / Subnet         | AZ    | Suggested CIDR   | Purpose                                 |
|------------------------|--------|------------------|------------------------------------------|
| Public Subnet A       | AZ-A  | 10.0.1.0/24      | NAT Gateway, Load Balancer, Bastion      |
| Public Subnet B       | AZ-B  | 10.0.2.0/24      |                                          |
| Public Subnet C       | AZ-C  | 10.0.3.0/24      |                                          |
| Private App Subnet A  | AZ-A  | 10.0.11.0/24     | Application servers (outbound via NAT)   |
| Private App Subnet B  | AZ-B  | 10.0.12.0/24     |                                          |
| Private App Subnet C  | AZ-C  | 10.0.13.0/24     |                                          |
| DB Subnet A           | AZ-A  | 10.0.21.0/24     | RDS / Database layer                     |
| DB Subnet B           | AZ-B  | 10.0.22.0/24     |                                          |
| DB Subnet C           | AZ-C  | 10.0.23.0/24     |                                          |

> Recommend dedicating contiguous /24 ranges to ease routing and growth planning.

---

## 4. Routing & Gateway Strategy

- **IGW** attached to the VPC.  
  All public subnets have routes directing `0.0.0.0/0` → IGW.
  
- **NAT Gateways**:  
  - One per Availability Zone  
  - Placed in the public subnet of that AZ  
  - Private subnets route outbound internet through their AZ's NAT Gateway  
  - Avoids cross‑AZ data charges and improves availability

- **Database Subnets**:  
  - No outbound internet route  
  - Only communicate internally or with the app tier


---

## 5. Security & Access Controls

### Security Groups (SGs):

**Public Tier SG**:
- Inbound:
  - HTTP/HTTPS open (`0.0.0.0/0`)
  - Optional: SSH from restricted admin IPs
- Outbound:
  - Default allow (or restricted as per policy)

**App Tier SG**:
- Inbound: Only from Public Tier SG (e.g., Load Balancer)
- Outbound: Permitted to Database Tier SG

**Database Tier SG**:
- Inbound: Only from App Tier SG
- Outbound: Usually none (or as required)

---

## 6. Additional Design Considerations

- **RDS DB Subnet Group**:  
  Group all three DB subnets for multi-AZ failover – required for high availability.

- **VPC Endpoints**:  
  Use for services like S3 or DynamoDB to avoid NAT traffic and reduce transfer costs.

- **VPC Flow Logs**:  
  Enable for auditing and monitoring network activity.

- **DNS Support**:  
  Ensure DNS hostnames and resolution are enabled for internal service discovery.

- **IPv6 Support** (if required):  
  Use Egress-Only Internet Gateway for IPv6 outbound from private subnets.

- **Architecture Diagrams**:  
  Use tools like draw.io, Lucidchart, or Creately for visual representation.

---

## 7. Summary

- A three-tier VPC architecture with:
  - 3 Public subnets  
  - 3 Private subnets  
  - 3 DB subnets  
  - Spread across 3 AZs

- Internet access provided by:
  - 1 IGW
  - 3 NAT Gateways (1 per AZ for HA & cost-efficiency)

- Database tier is fully isolated (no internet route)

- Security Groups follow least-privilege design

- Enhancements: VPC Endpoints, Flow Logs, DNS, Diagrams
