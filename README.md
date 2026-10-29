# AWS VPC Peering: Private Network Connectivity Lab

> **EWB Cloud Labs — AWS Solutions Architect Associate Project**  
> **Student:** SHIDESH PATIL | **Date:** 29 October 2026 | **AWS Region:** ap-south-1 (Mumbai)

---

## 📌 Project Overview
This project establishes and proves secure, bidirectional private connectivity between two isolated Virtual Private Clouds (**VPC A** and **VPC B**) using **AWS VPC Peering**, custom route tables, security groups, and an administrative Bastion Host. 

Neither private application server is assigned a public IP or exposed to the public internet.

---

## 🏗️ Architecture & IP Addressing Plan

| Resource Name | Subnet / VPC | IPv4 CIDR / Private IP | Public IPv4 | Purpose |
| :--- | :--- | :--- | :--- | :--- |
| **ewb-vpc-a** | VPC A | `10.10.0.0/16` | N/A | Primary VPC with public ingress |
| **ewb-a-public** | VPC A Subnet | `10.10.1.0/24` | Auto-assign Enabled | Public subnet for administration |
| **ewb-a-private** | VPC A Subnet | `10.10.2.0/24` | Auto-assign Disabled | Private workload subnet |
| **ewb-vpc-b** | VPC B | `10.20.0.0/16` | N/A | Secondary VPC (No IGW) |
| **ewb-b-private** | VPC B Subnet | `10.20.1.0/24` | Auto-assign Disabled | Private workload subnet |
| **ewb-bastion** | ewb-a-public | `10.10.1.10` | Auto-assigned | SSH Jump host for administrators |
| **ewb-app-a** | ewb-a-private | `10.10.2.10` | None | Private Application Server A |
| **ewb-app-b** | ewb-b-private | `10.20.1.10` | None | Private Application Server B |
| **ewb-igw-a** | Attached to VPC A | N/A | N/A | Internet Gateway for Bastion host |
| **ewb-a-b-peer** | VPC Peering Link | Requester: VPC A / Accepter: VPC B | N/A | Non-transitive private link |

---

## 🔒 Security Group Configuration

* **`ewb-sg-bastion` (VPC A):** Inbound SSH (TCP port 22) allowed strictly from administrator's local public IP (`/32`).
* **`ewb-sg-app-a` (VPC A):**
  * SSH (TCP port 22) allowed only from Bastion private IP (`10.10.1.10/32`).
  * Custom ICMP Echo Request allowed only from App-B (`10.20.1.10/32`).
* **`ewb-sg-app-b` (VPC B):**
  * SSH (TCP port 22) allowed only from Bastion private IP (`10.10.1.10/32`).
  * Custom ICMP Echo Request allowed only from App-A (`10.10.2.10/32`).

---

## 🛣️ Route Tables & Peering Configuration

* **`ewb-rt-a-public`:** `0.0.0.0/0` $\rightarrow$ `ewb-igw-a`, `10.20.0.0/16` $\rightarrow$ `pcx-xxxx` (so Bastion can SSH to App-B).
* **`ewb-rt-a-private`:** `10.20.0.0/16` $\rightarrow$ `pcx-xxxx` (App-A to VPC B).
* **`ewb-rt-b-private`:** `10.10.0.0/16` $\rightarrow$ `pcx-xxxx` (Return route back to VPC A).

---

## 🧪 Test Verification & Results

### 1. Baseline Test (Pre-Peering Failure)
Before establishing the peering connection and route tables, ICMP traffic from App-A (`10.10.2.10`) to App-B (`10.20.1.10`) was tested:
* **Result:** `ping -c 4 10.20.1.10` $\rightarrow$ **100% packet loss** (No route / dropped).

### 2. Forward & Reverse Path Verification (Post-Peering)
After activating `ewb-a-b-peer` and configuring bidirectional route entries:
* **Forward Ping (App-A $\rightarrow$ App-B):** `4 packets transmitted, 4 received, 0% packet loss`.
* **Reverse Ping (App-B $\rightarrow$ App-A):** `4 packets transmitted, 4 received, 0% packet loss`.

### 3. SSH Jump Authentication (Zero Private Key Leaks)
Private host access was completed via PuTTY + Pageant SSH Agent Forwarding:
* Connected to Bastion (`10.10.1.10`) $\rightarrow$ Jumped to App-A (`10.10.2.10`) and App-B (`10.20.1.10`) without copying `.pem` or `.ppk` keys to remote hosts.

### 4. Controlled Failure & Recovery Test
* Temporarily deleted inbound ICMP Echo Request rule on `ewb-sg-app-b`.
* Ping from App-A immediately failed (100% loss).
* Restored inbound ICMP rule on `ewb-sg-app-b`.
* Ping resumed immediately with 0% packet loss, proving security group enforcement.

---

## 💡 Viva / Architectural Questions & Answers

1. **Why is the return route needed?**
   * IP routing is hop-by-hop. Even if VPC A routes traffic forward to VPC B, VPC B's route table must explicitly contain a destination route pointing `10.10.0.0/16` to `pcx-xxxx` so response packets know how to return.
2. **Why is the App-B SSH source the bastion private IP?**
   * Because SSH connection forwarding through the Bastion terminates the laptop's external TCP handshake at the Bastion and initiates a new TCP connection from the Bastion's private IP (`10.10.1.10`) to App-B.
3. **Why does Active peering not guarantee a successful ping?**
   * Active peering only provisions the physical/virtual network link. Communication still requires explicit route table entries in both directions and security group rules allowing inbound ICMP traffic.
4. **Can VPC B use VPC A's internet gateway over peering?**
   * No. AWS VPC Peering does **not** support edge-to-edge routing or transitive routing; an internet gateway attached to VPC A cannot be used by peered VPC B.

---

## 🧹 Cleanup
All project resources (EC2 instances, Peering connections, custom Route Tables, Subnets, Internet Gateways, and VPCs) were terminated and removed to prevent unnecessary AWS charges.
