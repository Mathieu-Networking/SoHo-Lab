# Enterprise SOHO Network Implementation (Cisco Packet Tracer)

### 1. Project Overview
This project simulates a secure, scalable network for a small-to-medium business consisting of four distinct departments (HR, Legal, Sales, IT). The goal was to provide segmentation between departments for security while allowing all users to access the internet via a simulated ISP. The architecture features a multi-switch topology to demonstrate Layer 2 trunking and scalability.

**Key Technologies:** VLANs, Inter-VLAN Routing (ROAS), 802.1Q Trunking, DHCP, NAT/PAT, Extended ACLs.

### 2. Network Topology
![Network Topology Diagram](images/topology_screenshot.png)
*(Note: Replace this path with your actual screenshot file)*

### 3. Architecture & Addressing
The network uses a **Hub-and-Spoke** topology with a Cisco 2911 Router handling all Layer 3 routing decisions. I expanded the Layer 2 footprint by daisy-chaining a second switch (S2) to Switch 1 (S1) via an 802.1Q Trunk, allowing VLANs to span multiple physical devices.

**Addressing Schema (10.x.x.x Private Space):**

| VLAN ID | Department | Subnet | Gateway | DHCP Pool Start | Hosted On |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **10** | HR | 10.0.10.0/24 | 10.0.10.1 | .21 | Switch 1 |
| **20** | Legal | 10.0.20.0/24 | 10.0.20.1 | .21 | Switch 1 |
| **30** | Sales | 10.0.30.0/24 | 10.0.30.1 | .21 | Switch 1 |
| **40** | IT/Ops | 10.0.40.0/24 | 10.0.40.1 | .21 | Switch 2 |
| **WAN** | ISP Link | 203.0.113.0/30 | 203.0.113.2 | N/A | Router 1 |

### 4. Key Configurations

#### A. Multi-Switch Trunking (Layer 2 Expansion)
To support the new IT department on a separate physical switch, I configured an 802.1Q Trunk link between Switch 1 and Switch 2. This allows VLAN traffic to pass between switches before reaching the router.
```cisco
! Switch 1 Interface to Switch 2
interface GigabitEthernet0/2
 switchport mode trunk
! Switch 2 Interface to Switch 1
interface GigabitEthernet0/1
 switchport mode trunk
```
###B. Secure Segmentation (VLANs & ROAS)
I configured Router-on-a-Stick on the main router to handle traffic for all 4 VLANs.

```Cisco CLI
! Example: Router Configuration for new VLAN 40
interface GigabitEthernet0/0/1.40
 encapsulation dot1Q 40
 ip address 10.0.40.1 255.255.255.0
```
### C. Automated Addressing (DHCP)
The router acts as the DHCP server for all scopes. I utilized excluded-address to reserve the first 20 IPs for static devices (printers/servers) across all subnets.

```Cisco CLI
! Reserved .1 through .20 for static assignment
ip dhcp excluded-address 10.0.10.1 10.0.10.20
ip dhcp excluded-address 10.0.40.1 10.0.40.20

! Example Pool for VLAN 40
ip dhcp pool IT_POOL
 network 10.0.40.0 255.255.255.0
 default-router 10.0.40.1
 dns-server 8.8.8.8
```
### D. Internet Connectivity (NAT Overload)
To allow internal private IPs to access the public internet, I configured Port Address Translation (PAT). This maps all internal traffic (List 1) to the single public WAN IP (203.0.113.2).

```Cisco CLI
! NAT Configuration
access-list 1 permit 10.0.0.0 0.255.255.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload
```
#### E. Security (Access Control Lists)
I implemented an Extended ACL to block the Sales department from accessing sensitive HR financial servers.

```Cisco CLI
access-list 100 deny ip 10.0.30.0 0.0.0.255 10.0.10.0 0.0.0.255
access-list 100 permit ip any any
```
### 5. Troubleshooting & Challenges
**Issue:** Internal PCs could not access the Google Server (8.8.8.8), and the NAT translation table remained empty.

**Diagnosis:** Upon inspection (show ip interface brief), I realized the WAN interface (g0/0) was Up but had no IP address assigned. Because the router lacked an IP in the ISP subnet, it could not install the default gateway route, causing the traffic to drop before NAT could occur.

**Resolution:** Assigned the public IP 203.0.113.2 to the interface, flagged it as ip nat outside, and verified connectivity by checking the translation table (show ip nat translations).

6. Verification Proof
Evidence of NAT working:

Evidence of ACL blocking Sales->HR:  
