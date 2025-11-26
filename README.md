# Enterprise SOHO Network Implementation (Cisco Packet Tracer)

### 1. Project Overview
This project simulates a secure, scalable network for a small-to-medium business consisting of three distinct departments (HR, Legal, Sales). The goal was to provide segmentation between departments for security while allowing all users to access the internet via a simulated ISP.

**Key Technologies:** VLANs, Inter-VLAN Routing (ROAS), DHCP, NAT/PAT, Extended ACLs.

### 2. Network Topology
![Network Topology Diagram](Reservedfile . png


### 3. Architecture & Addressing
The network uses a **Hub-and-Spoke** topology with a Cisco 2911 Router handling all Layer 3 routing decisions. I utilized the **10.x.x.x** private address space to prevent VPN conflicts and allow for future scalability.

| VLAN ID | Department | Subnet | Gateway | DHCP Range |
| :--- | :--- | :--- | :--- | :--- |
| **10** | HR | 10.0.10.0/24 | 10.0.10.1 | .11 - .254 |
| **20** | Legal | 10.0.20.0/24 | 10.0.20.1 | .11 - .254 |
| **30** | Sales | 10.0.30.0/24 | 10.0.30.1 | .11 - .254 |
| **WAN** | ISP Link | 203.0.113.0/30 | 203.0.113.2 | N/A |

### 4. Key Configurations

#### A. Secure Segmentation (VLANs & ROAS)
I configured **Router-on-a-Stick** to allow the router to handle traffic between VLANs using 802.1Q encapsulation.
```cisco
! Router Configuration for HR VLAN
interface GigabitEthernet0/0/1.10
 encapsulation dot1Q 10
 ip address 10.0.10.1 255.255.255.0
```

### B. Automated Addressing (DHCP)
To minimize administrative overhead, I configured the router to act as the DHCP server. I utilized excluded-address to reserve the first 10 IPs for static devices (printers/servers).

```ip dhcp excluded-address 10.0.10.1 10.0.10.10
ip dhcp pool HR_POOL
 network 10.0.10.0 255.255.255.0
 default-router 10.0.10.1
 dns-server 8.8.8.8
```

### C. Internet Connectivity (NAT Overload)
To allow internal private IPs to access the public internet, I configured Port Address Translation (PAT). This maps all internal traffic to the single public WAN IP (203.0.113.2).
```
! NAT Configuration
access-list 1 permit 10.0.0.0 0.255.255.255
ip nat inside source list 1 interface GigabitEthernet0/0 overload
```
### D. Security (Access Control Lists)
I implemented an Extended ACL to block the Sales department from accessing sensitive HR financial servers, while still allowing Sales to access the Internet.
```
access-list 100 deny ip 10.0.30.0 0.0.0.255 10.0.10.0 0.0.0.255
access-list 100 permit ip any any
```
### 5. Troubleshooting & Challenges
Issue: Internal PCs could not access the Google Server (8.8.8.8), and the NAT translation table remained empty.

Diagnosis: Upon inspection (show ip interface brief), I realized the WAN interface (g0/0) was Up but had no IP address assigned. Because the router lacked an IP in the ISP subnet, it could not install the default gateway route, causing the traffic to drop before NAT could occur.

Resolution: Assigned the public IP 203.0.113.2 to the interface, flagged it as ip nat outside, and verified connectivity by checking the translation table (show ip nat translations).

### 6. Verification Proof
Evidence of NAT working:

Evidence of ACL blocking Sales->HR
