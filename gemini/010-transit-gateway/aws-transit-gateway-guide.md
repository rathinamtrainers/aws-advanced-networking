# AWS Transit Gateway: Training and Lab Guide

## Table of Contents
1.  **Introduction to AWS Transit Gateway**
    *   What is it?
    *   Why use it? Problems it solves.
2.  **Core Concepts**
    *   Transit Gateway (TGW)
    *   Attachments (VPC, VPN, Direct Connect)
    *   TGW Route Tables
    *   Associations
    *   Propagations
3.  **VPC Peering vs. Transit Gateway**
4.  **Lab Guide: Migrating from VPC Peering to Transit Gateway**
    *   Lab Overview & Architecture
    *   Prerequisites
    *   **Part 1:** Build the Initial Environment with VPC Peering
    *   **Part 2:** Deploy Transit Gateway and Migrate
    *   **Part 3:** Test and Decommission
5.  **Advanced Concepts**
    *   Inter-Region Peering
    *   Multicast
6.  **Cleanup**

---

### 1. Introduction to AWS Transit Gateway

#### What is it?
AWS Transit Gateway is a fully managed, highly available, and scalable cloud router that simplifies network connectivity between Amazon Virtual Private Clouds (VPCs), on-premises data centers, and other AWS accounts. It acts as a central hub where traffic is routed between all connected networks, known as "spokes."

#### Why use it? Problems it solves.
Before Transit Gateway, connecting many VPCs required a complex mesh of **VPC Peering** connections. If you had 10 VPCs that all needed to communicate, you would need 45 peering connections (`n * (n-1) / 2`). This model is complex to manage, scale, and troubleshoot.

**Transit Gateway solves this by:**
*   **Simplifying Connectivity:** It establishes a "hub and spoke" topology. Each VPC (spoke) connects to the central Transit Gateway (hub) just once.
*   **Centralized Routing:** It provides a central point to manage and control routing policies between your networks.
*   **Eliminating Transitive Peering Limitations:** Standard VPC Peering is not transitive (if VPC A is peered with B, and B is peered with C, A cannot talk to C through B). Transit Gateway allows for this transitive routing by default.
*   **Connecting to On-Premises:** It consolidates connections from on-premises networks (via AWS Direct Connect or Site-to-Site VPN) to a single gateway.

---

### 2. Core Concepts

*   **Transit Gateway (TGW):** The regional cloud router resource itself.
*   **Attachments:** The connection from a network to the TGW.
    *   **VPC Attachment:** Connects a VPC in the same region to the TGW.
    *   **VPN Attachment:** Connects an on-premises network via a Site-to-Site VPN connection.
    *   **Direct Connect Attachment:** Connects an on-premises network via a Direct Connect Gateway.
    *   **Peering Attachment:** Connects to another Transit Gateway in a different AWS region.
*   **TGW Route Table:** Similar to a VPC route table, this controls where traffic is routed. Each attachment is associated with exactly one TGW route table.
*   **Association:** The process of linking an attachment to a TGW route table. This determines the routing behavior for traffic *coming from* that attachment.
*   **Propagation:** The process where routes from an attachment (like a VPC's CIDR block) are automatically added (propagated) to a TGW route table. This allows other connected networks to learn the route to that VPC.

---

### 3. VPC Peering vs. Transit Gateway

| Feature | VPC Peering | AWS Transit Gateway |
| :--- | :--- | :--- |
| **Topology** | Mesh (Point-to-Point) | Hub and Spoke |
| **Scalability** | Difficult. Limited to 125 active peers per VPC. | High. Scales to thousands of VPCs. |
| **Transitive Routing**| Not supported. | **Yes**, this is its primary function. |
| **Management** | Decentralized. Each connection is managed individually. | Centralized. Routing is managed at the TGW. |
| **On-Premises** | Requires separate VPN/DX to each VPC. | Single VPN/DX connection to the TGW. |
| **Cost** | Data processing charges only. | Hourly charge per attachment + data processing charges. |

---

### 4. Lab Guide: Migrating from VPC Peering to Transit Gateway

#### Lab Overview & Architecture

We will build a common scenario where two VPCs are connected via VPC Peering. We will then deploy a Transit Gateway, reroute traffic through it, and finally decommission the old peering connection, completing the migration with zero downtime.

*   **Initial State:** `VPC-A` <--> `VPC Peering` <--> `VPC-B`
*   **Final State:** `VPC-A` <--> `Transit Gateway` <--> `VPC-B`

#### Prerequisites
*   An AWS Account with permissions to create VPCs, EC2 instances, and Transit Gateways.
*   Familiarity with the AWS Management Console.

---

#### **Part 1: Build the Initial Environment with VPC Peering**

In this part, we set up two VPCs, launch a test instance in each, and connect them with VPC Peering.

**Step 1: Create VPC-A and its resources**
1.  Navigate to the **VPC Dashboard**.
2.  Click **Create VPC**.
3.  Select **VPC and more**.
4.  **Name tag:** `VPC-A`
5.  **IPv4 CIDR block:** `10.1.0.0/16`
6.  **Availability Zones (AZs):** 1
7.  **Public subnets:** 1
8.  **Private subnets:** 0
9.  **VPC endpoints:** None
10. Click **Create VPC**.
11. **Launch an EC2 Instance:**
    *   Navigate to the **EC2 Dashboard**.
    *   Click **Launch instances**.
    *   **Name:** `EC2-A`
    *   **AMI:** Amazon Linux 2 (or newer)
    *   **Instance type:** `t2.micro`
    *   **Key pair:** Create or select an existing one.
    *   **Network settings:**
        *   **VPC:** `VPC-A`
        *   **Subnet:** Select the public subnet in `VPC-A`.
        *   **Auto-assign public IP:** Enable
        *   **Security Group:** Create a new one named `SG-A`. Add a rule for **SSH (port 22)** from your IP and an **All ICMP - IPv4** rule from `10.2.0.0/16` (this is for `VPC-B`).
    *   Click **Launch instance**.

**Step 2: Create VPC-B and its resources**
1.  Repeat the process above with the following details:
    *   **Name tag:** `VPC-B`
    *   **IPv4 CIDR block:** `10.2.0.0/16`
    *   **EC2 Name:** `EC2-B`
    *   **Security Group:** `SG-B`. Add a rule for **SSH (port 22)** from your IP and an **All ICMP - IPv4** rule from `10.1.0.0/16` (for `VPC-A`).

**Step 3: Create and Configure VPC Peering**
1.  In the **VPC Dashboard**, go to **Peering connections**.
2.  Click **Create peering connection**.
3.  **Name:** `Peer-A-to-B`
4.  **VPC (Requester):** `VPC-A`
5.  **VPC (Accepter):** `VPC-B`
6.  Click **Create peering connection**.
7.  Select the new peering connection (it will be in a "Pending Acceptance" state) and go to **Actions -> Accept request**.
8.  **Update Route Tables:**
    *   Go to **Route Tables**.
    *   Select the main route table for `VPC-A`.
    *   Click **Routes -> Edit routes**.
    *   Add route: **Destination:** `10.2.0.0/16`, **Target:** `Peering Connection` -> `Peer-A-to-B`.
    *   Save changes.
    *   Select the main route table for `VPC-B`.
    *   Click **Routes -> Edit routes**.
    *   Add route: **Destination:** `10.1.0.0/16`, **Target:** `Peering Connection` -> `Peer-A-to-B`.
    *   Save changes.

**Step 4: Test Peering Connectivity**
1.  Connect to `EC2-A` via SSH or Session Manager.
2.  Get the Private IP address of `EC2-B` from the console (e.g., `10.2.x.x`).
3.  From `EC2-A`, run: `ping <Private IP of EC2-B>`
4.  You should see a successful ping response. This confirms the peering is working.

---

#### **Part 2: Deploy Transit Gateway and Migrate**

Now we introduce the TGW and shift traffic away from the peering connection.

**Step 1: Create the Transit Gateway**
1.  In the **VPC Dashboard**, go to **Transit gateways**.
2.  Click **Create transit gateway**.
3.  **Name tag:** `My-TGW`
4.  Leave all other options as default (e.g., Amazon side ASN, DNS support, etc.).
5.  Click **Create transit gateway**. It will take a few minutes to become available.

**Step 2: Create Transit Gateway Attachments**
1.  Go to **Transit gateway attachments**.
2.  Click **Create transit gateway attachment**.
3.  **Transit gateway ID:** Select `My-TGW`.
4.  **Attachment type:** `VPC`
5.  **Attachment name tag:** `TGW-Attach-VPC-A`
6.  **VPC ID:** Select `VPC-A`.
7.  **Subnet IDs:** Select the subnet in `VPC-A`.
8.  Click **Create attachment**.
9.  Repeat the process for `VPC-B`:
    *   **Attachment name tag:** `TGW-Attach-VPC-B`
    *   **VPC ID:** Select `VPC-B`.
    *   **Subnet IDs:** Select the subnet in `VPC-B`.
    *   Click **Create attachment**.

**Step 3: Verify Route Propagation**
1.  Go to **Transit gateway route tables**.
2.  Select the default route table for `My-TGW`.
3.  Click the **Propagations** tab. You should see that both attachments are propagating their routes.
4.  Click the **Routes** tab. You should see routes for `10.1.0.0/16` and `10.2.0.0/16`, both pointing to their respective VPC attachments. This means the TGW knows how to reach both VPCs.

**Step 4: The "Switchover" - Update VPC Route Tables**
This is the critical migration step. We will change the VPC routes to point to the TGW instead of the peering connection.
1.  Go to **Route Tables**.
2.  Select the main route table for `VPC-A`.
3.  Click **Routes -> Edit routes**.
4.  **Find the existing route:** Destination `10.2.0.0/16` pointing to the peering connection.
5.  **Change the Target:**
    *   From `Peering Connection` to `Transit Gateway`.
    *   Select `My-TGW`.
6.  Click **Save changes**.
7.  Repeat for `VPC-B`'s route table:
    *   Select the main route table for `VPC-B`.
    *   Click **Routes -> Edit routes**.
    *   Change the target for destination `10.1.0.0/16` from the peering connection to `My-TGW`.
    *   Click **Save changes**.

The migration is now complete. Traffic between the VPCs will now flow through the Transit Gateway.

---

#### **Part 3: Test and Decommission**

**Step 1: Test TGW Connectivity**
1.  Go back to your SSH session on `EC2-A`.
2.  Run the ping command again: `ping <Private IP of EC2-B>`
3.  The ping should still be successful, but this time the packets are being routed through the Transit Gateway. You have successfully migrated without downtime.

**Step 2: Decommission the VPC Peering Connection**
Now that traffic is flowing through the TGW, the peering connection is redundant and can be safely removed.
1.  In the **VPC Dashboard**, go to **Peering connections**.
2.  Select `Peer-A-to-B`.
3.  Click **Actions -> Delete peering connection**.
4.  Confirm the deletion.

---

### 5. Advanced Concepts (Briefly)

*   **Inter-Region Peering:** You can create a `Peering Attachment` on your TGW to connect it to another TGW in a different AWS Region, extending your network globally.
*   **Multicast:** TGW supports multicast routing, which is useful for applications that send a single stream of data to many users simultaneously (e.g., video streaming, stock market data feeds).

---

### 6. Cleanup

To avoid ongoing charges, delete the resources you created in the following order:
1.  **EC2 Instances:** Terminate `EC2-A` and `EC2-B`.
2.  **Transit Gateway Attachments:** Delete `TGW-Attach-VPC-A` and `TGW-Attach-VPC-B`.
3.  **Transit Gateway:** Delete `My-TGW`.
4.  **VPC Peering Connection:** (Should already be deleted).
5.  **VPCs:** Delete `VPC-A` and `VPC-B`. This will also remove associated subnets, route tables, and internet gateways.
6.  **Security Groups:** Delete `SG-A` and `SG-B` if they were not deleted with the VPCs.
7.  **Key Pairs:** Delete if you created a new one for the lab.
