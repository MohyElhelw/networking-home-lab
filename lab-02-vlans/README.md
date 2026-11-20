# Lab 2 – VLANs, Inter-VLAN Routing, DHCP Relay, and a Management VLAN

Builds on [Lab 1](../lab-01-basic-lan/). The flat 192.168.1.0/24 network is split into separate VLANs for staff, students, servers, and switch management. A single router link connects them using router-on-a-stick, and one DHCP server in the server VLAN hands out addresses to every VLAN through DHCP relay.

![Topology](screenshots/topology.png)

## VLAN and addressing plan

| VLAN | Name | Network | Gateway (R1) | Devices | Switch port | Addressing |
|---|---|---|---|---|---|---|
| 10 | staff | 192.168.10.0/24 | 192.168.10.1 | PC-00, PC-01 | Fa0/2, Fa0/3 | DHCP (relay) |
| 20 | students | 192.168.20.0/24 | 192.168.20.1 | Laptop0 | Fa0/1 | DHCP (relay) |
| 30 | servers | 192.168.30.0/24 | 192.168.30.1 | Server0 (.10) | Fa0/10 | Static |
| 99 | management | 192.168.99.0/24 | 192.168.99.1 | S1 management IP (.11) | — | Static |

The S1 Gig0/1 to R1 Gig0/0 link is an **802.1Q trunk** carrying all VLANs.

## How it works
- **Access ports** (Fa0/1–3, Fa0/10) each belong to one VLAN. End devices don't know VLANs exist; the switch port decides.
- The **trunk** (Gig0/1) carries every VLAN over one cable. The switch adds an 802.1Q tag to each frame so the router knows which VLAN it came from.
- **Router-on-a-stick:** R1's single port is split into subinterfaces (G0/0.10, .20, .30, .99). Each one reads frames with its VLAN tag and acts as that VLAN's default gateway.
- **DHCP relay:** DHCP requests are broadcasts, and routers don't forward broadcasts. `ip helper-address` on G0/0.10 and G0/0.20 forwards them to the DHCP server in VLAN 30, and the server picks the pool that matches the subinterface the request came from.
- **Management VLAN:** the switch's management IP was moved off the default VLAN 1 to VLAN 99, and VLAN 1's interface was shut down.

## Configuration

### Switch (S1)
```
vlan 10
 name staff
vlan 20
 name students
vlan 30
 name servers
vlan 99
 name management
!
interface range fa0/2 - 3
 switchport mode access
 switchport access vlan 10
interface fa0/1
 switchport mode access
 switchport access vlan 20
interface fa0/10
 switchport mode access
 switchport access vlan 30
interface g0/1
 switchport mode trunk
!
interface vlan 1
 no ip address
 shutdown
interface vlan 99
 ip address 192.168.99.11 255.255.255.0
 no shutdown
ip default-gateway 192.168.99.1
```

### Router (R1)
```
interface g0/0
 no ip address
 no shutdown
interface g0/0.10
 encapsulation dot1Q 10
 ip address 192.168.10.1 255.255.255.0
 ip helper-address 192.168.30.10
interface g0/0.20
 encapsulation dot1Q 20
 ip address 192.168.20.1 255.255.255.0
 ip helper-address 192.168.30.10
interface g0/0.30
 encapsulation dot1Q 30
 ip address 192.168.30.1 255.255.255.0
interface g0/0.99
 encapsulation dot1Q 99
 ip address 192.168.99.1 255.255.255.0
```

### Server0
Static IP 192.168.30.10/24, gateway 192.168.30.1, DNS 192.168.30.10.

| DHCP pool | Gateway | DNS | Start IP | Mask | Max users |
|---|---|---|---|---|---|
| serverPool | 192.168.30.1 | 192.168.30.10 | 192.168.30.101 | 255.255.255.0 | 154 |
| vlan10 | 192.168.10.1 | 192.168.30.10 | 192.168.10.101 | 255.255.255.0 | 154 |
| vlan20 | 192.168.20.1 | 192.168.30.10 | 192.168.20.101 | 255.255.255.0 | 154 |

Line and enable passwords carried over from Lab 1 and are not included here.

## Build steps and verification

| Step | What I did | Test | Result |
|---|---|---|---|
| 1 | Created VLANs 10 and 20 | `show vlan brief` | Both VLANs active |
| 2 | Assigned access ports | PC-01 → PC-00 (same VLAN) | ✅ Replies |
| 2 | | PC-01 → Laptop0 (different VLAN, no router yet) | ❌ Timed out, as expected; VLANs isolate traffic |
| 3 | Made Gig0/1 a trunk | `show interfaces trunk` | Gig0/1 trunking, 802.1q, VLANs 1, 10, 20 active |
| 4 | Added router subinterfaces | `show ip interface brief` on R1 | G0/0.10 and G0/0.20 up/up |
| 5 | Gave clients static IPs in their VLAN's network | Laptop0 → 192.168.20.1 (own gateway) | ✅ TTL=255 |
| 5 | | Laptop0 → PC-01 (192.168.10.101) | ✅ TTL=127, so the traffic was routed through R1 |
| 6A | Moved the server to VLAN 30 | Laptop0 → 192.168.30.10 | ✅ TTL=127 |
| 6B | Set up DHCP relay and per-VLAN pools; switched clients to DHCP | `ipconfig /all` on a PC | 192.168.10.102, gateway .10.1, DHCP server 192.168.30.10 |
| 6B | | `ipconfig /all` on Laptop0 | 192.168.20.101, gateway .20.1, DHCP server 192.168.30.10 |
| 7 | Moved switch management to VLAN 99 | `show ip interface brief` on S1 | Vlan99 up/up at .11; Vlan1 administratively down |
| 7 | | `telnet 192.168.99.11` from PC-01 | ✅ Password prompt, then `S1>` |

### Screenshots
![VLANs](screenshots/show-vlan-brief.png)
![Trunk](screenshots/show-interfaces-trunk.png)
![Ping blocked before routing](screenshots/ping-blocked.png)
![Ping routed between VLANs](screenshots/ping-routed.png)
![DHCP across VLANs](screenshots/dhcp-relay.png)
![Telnet over management VLAN](screenshots/telnet-mgmt.png)

## What I learned
- One VLAN = one subnet. Devices in different VLANs can't talk without a router.
- Access ports carry one untagged VLAN; trunks carry many VLANs using 802.1Q tags. A trunk carries all VLANs by default.
- Router-on-a-stick: subinterfaces with `encapsulation dot1Q <vlan>` let one physical port be the gateway for several VLANs.
- TTL drops by 1 for each router hop. 128 → 127 proved traffic was routed between VLANs.
- DHCP needs `ip helper-address` to cross VLANs, because routers don't forward broadcasts.
- Troubleshooting: when a ping that should fail succeeds, check which device actually has the IP (`ipconfig`, `arp -a`, `show mac address-table`) before trusting the result.
- Managing a switch on VLAN 1 is bad practice; a dedicated management VLAN keeps it separate from user traffic.

## Next steps
- Shut down unused switch ports.
- Replace Telnet with SSH.
- Later lab: use ACLs to control which VLANs can reach each other (for example, block students from staff).
