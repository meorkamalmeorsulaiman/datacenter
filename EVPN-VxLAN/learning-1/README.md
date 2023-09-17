# EVPN-VxLAN


## How to make it work

### Labs Setup

![EVPN-VxLAN](https://github.com/meorkamalmeorsulaiman/DC-Tech/blob/EVPN-VxLAN/EVPN-VxLAN/configs/initials-01/initials-01.png)

The full configs available here: [Intial configuration](https://github.com/meorkamalmeorsulaiman/DC-Tech/tree/EVPN-VxLAN/EVPN-VxLAN/configs/initials-01)

### Features required

Spines

```
nv overlay evpn
feature ospf
feature bgp
feature pim
```

Leaf

```
nv overlay evpn
feature ospf
feature bgp
feature pim
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay
```

### Underlay

- You'll need an underlay networking
- This are the underlaying network infrastructure that will be use to create the overlay VxLAN and EVPN
- Ensure that your loopback reachable from each nodes in the fabric
- The loopback will be use for iBGP, Anycast and VxLAN tunnels
- We use:
  - P2P spine01 `10.0.2.0/24`
  - P2P2 spine02 `10.0.3.0/24`
  - Overlay Loopback spines `10.0.0.0/24`
  - Overlay Loopback leafs `10.0.1.0/24`
  - VTEP Addresses `10.0.5.0/24`
  - Ancast RP spines `10.0.0.254/32` 
- Underlay routing protocol would be OSPF `100` and BGP AS `65500` for BGP-EVPN

### VNI Mapping

- Build the VLAN and VxLAN mapping
- VxLAN use VNI which mapped with the true VLAN
- These should be configured on all leafs switches

```
vlan 10
  name Customer-A-VL10
  vn-segment 10010
```

### Create NVE interface

- This will be the VTEPs and only on the leafs

```
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  member vni 10010
    mcast-group 239.0.0.1
```

### Established BGP-EVPN

 - Established iBGP-EVPN
 - Leafs

 ```
router bgp 65500
  router-id 10.0.1.1
  neighbor 10.0.0.1
    remote-as 65500
    update-source loopback0
    address-family l2vpn evpn
      send-community extended
  neighbor 10.0.0.2
    remote-as 65500
    update-source loopback0
    address-family l2vpn evpn
      send-community extended

```
- Spines will be the route-reflector

```
router bgp 65500
  router-id 10.0.0.1
  neighbor 10.0.1.0/24
    remote-as 65500
    update-source loopback0
    address-family l2vpn evpn
      send-community extended
      route-reflector-client
```
- Validate the neighborship on spines and leafs

```
spine01# show bgp l2vpn evpn summary 
BGP summary information for VRF default, address family L2VPN EVPN
BGP router identifier 10.0.0.1, local AS number 65500
BGP table version is 28, L2VPN EVPN config peers 5, capable peers 4
4 network entries and 4 paths using 976 bytes of memory
BGP attribute entries [1/172], BGP AS path entries [0/0]
BGP community entries [0/0], BGP clusterlist entries [0/0]

Neighbor        V    AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.0.1.1        4 65500      68      68       28    0    0 00:57:30 2         
10.0.1.2        4 65500      68      68       28    0    0 00:57:30 2         
10.0.1.3        4 65500      65      68       28    0    0 00:57:31 0         
10.0.1.4        4 65500      65      68       28    0    0 00:57:31 0      
spine01# 
```

## Generate BGP Advertisement

- Only on the leafs

```
evpn
  vni 10010 l2
    rd auto
    route-target import auto
    route-target export auto
```

### Multicast Routing

- Multicast use for BUM traffic. Let's be clear that ethernet still rely on FFFF
- There are 2 role, undelay multicast for ARP, DHCP, ND etc and Layer 3 multicast generate by host would used the overlay multicast routing with Anycast-RP
- Leafs

```
ip pim rp-address 10.0.0.254 group-list 224.0.0.0/4
ip pim ssm range 232.0.0.0/8


interface loopback0
  ip pim sparse-mode

interface loopback1
  ip pim sparse-mode

interface Ethernet1/1
  ip pim sparse-mode

interface Ethernet1/2
  ip pim sparse-mode
```

- Spines

```
ip pim rp-address 10.0.0.254 group-list 224.0.0.0/4
ip pim ssm range 232.0.0.0/8
ip pim anycast-rp 10.0.0.254 10.0.0.1
ip pim anycast-rp 10.0.0.254 10.0.0.2


interface loopback0
  ip pim sparse-mode

interface loopback1
  ip pim sparse-mode

interface Ethernet1/1
  ip pim sparse-mode

interface Ethernet1/2
  ip pim sparse-mode

interface Ethernet1/3
  ip pim sparse-mode

interface Ethernet1/4
  ip pim sparse-mode

```

- Validate the multicast routing with `show ip mroute` the S,G will be created once the multicast source starting the stream
- Validate anycast cluster on spines:

```
spine01# show ip pim rp 
PIM RP Status Information for VRF "default"
BSR disabled
Auto-RP disabled
BSR RP Candidate policy: None
BSR RP policy: None
Auto-RP Announce policy: None
Auto-RP Discovery policy: None

Anycast-RP 10.0.0.254 members:
  10.0.0.1*  10.0.0.2  

RP: 10.0.0.254*, (0), 
 uptime: 00:57:32   priority: 255, 
 RP-source: (local),  
 group ranges:
 224.0.0.0/4   
spine01# 

```

### Validate the EVPN 
- There are 2 part where you have to validate EVPN working correctly:
  - Local leaf
  - Remote leaf
- The items for each leafs devices into 3:
  - Local mac learning `show l2route evpn mac evi [VLAN] `
  - Local mac inserted EVPN instance `show bgp l2vpn evpn vni-id [VNI]`
  - Local EVPN instance advertise to spines `show bgp l2vpn evpn [MAC]` or `show bgp l2vpn evpn neighbors [spines IP] advertised-routes`
  - Remote leaf received advertisement `show bgp l2vpn evpn [MAC]` ` show bgp l2vpn evpn rd [local RD] [MAC]`
  - Remote leaf installed into local EVPN instance `show bgp l2vpn evp vni-id [VNI]`
  - Remote leaf receive MAC remotely `show l2route evpn mac evi [VLAN]` or `show l2route mac topology [VLAN]`


- Ensure that you have configured access port toward the client and anycast gateway to the vlan SVI on all the leafs

```
interface Vlan10
  no shutdown
  ip address 10.10.10.254/24
  fabric forwarding mode anycast-gateway
fabric forwarding anycast-gateway-mac 0000.2222.3333
```

- Validate the MAC of client has been produce to EVPN instance in the local leaf

```
leaf01# show l2route evpn mac evi 10

Flags -(Rmac):Router MAC (Stt):Static (L):Local (R):Remote (V):vPC link 
(Dup):Duplicate (Spl):Split (Rcv):Recv (AD):Auto-Delete (D):Del Pending
(S):Stale (C):Clear, (Ps):Peer Sync (O):Re-Originated (Nho):NH-Override
(Pf):Permanently-Frozen, (Orp): Orphan

Topology    Mac Address    Prod   Flags         Seq No     Next-Hops            
                  
----------- -------------- ------ ------------- ---------- ---------------------
------------------
10          0050.0000.0c00 Local  L,            0          Eth1/3    
```

- Validate that the MAC installed in the BGP EVPN AFI

```
leaf01# show bgp l2vpn evpn vni-id 10010
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 21, Local Router ID is 10.0.1.1
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 10.0.1.1:32777    (L2VNI 10010)
*>l[2]:[0]:[0]:[48]:[0050.0000.0c00]:[0]:[0.0.0.0]/216
                      10.0.5.1                          100      32768 i
*>l[2]:[0]:[0]:[48]:[0050.0000.0c00]:[32]:[10.10.10.1]/248
                      10.0.5.1                          100      32768 i
```

- Check the route advertise to leaf02 - Check in the originating leaf
- RD derived from VNI config under evpn instance
- /216 specifies the bit count of the prefix
- /248 bit count for mac-IP
- There are 2 updates here, MAC and MAC-IP

```
leaf01# show bgp l2vpn evpn 0050.0000.0c00
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.0.1.1:32777    (L2VNI 10010)
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.0000.0c00]:[0]:[0.0.0.0]/216,
 version 13
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    10.0.5.1 (metric 0) from 0.0.0.0 (10.0.1.1)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10010
      Extcommunity: RT:65500:10010 ENCAP:8

  Path-id 1 advertised to peers:
    10.0.0.1           10.0.0.2       
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.0000.0c00]:[32]:[10.10.10.1]/
248, version 17
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    10.0.5.1 (metric 0) from 0.0.0.0 (10.0.1.1)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10010
      Extcommunity: RT:65500:10010 ENCAP:8

  Path-id 1 advertised to peers:
    10.0.0.1           10.0.0.2 
```

- Move to remote leaf - leaf02
- Check whether we received the both of the updates /216 & /272

```
leaf03# show bgp l2vpn evpn 0050.0000.0c00
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.0.1.1:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.0000.0c00]:[0]:[0.0.0.0]/216,
 version 21
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  AS-Path: NONE, path sourced internal to AS
    10.0.5.1 (metric 81) from 10.0.0.2 (10.0.0.2)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010
      Extcommunity: RT:65500:10010 ENCAP:8
      Originator: 10.0.1.1 Cluster list: 10.0.0.2 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10010
  AS-Path: NONE, path sourced internal to AS
    10.0.5.1 (metric 81) from 10.0.0.1 (10.0.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010
      Extcommunity: RT:65500:10010 ENCAP:8
      Originator: 10.0.1.1 Cluster list: 10.0.0.1
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.0000.0c00]:[32]:[10.10.10.1]/
248, version 27
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10010
  AS-Path: NONE, path sourced internal to AS
    10.0.5.1 (metric 81) from 10.0.0.1 (10.0.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010
      Extcommunity: RT:65500:10010 ENCAP:8
      Originator: 10.0.1.1 Cluster list: 10.0.0.1 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  AS-Path: NONE, path sourced internal to AS
    10.0.5.1 (metric 81) from 10.0.0.2 (10.0.0.2)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010
      Extcommunity: RT:65500:10010 ENCAP:8
      Originator: 10.0.1.1 Cluster list: 10.0.0.2 

```

- Check the update correctly install in the L2VNI EVPN instance `10010` on the remote leafs
- Check if the route correctly import into local instance
```
leaf03# show bgp l2vpn evp vni-id 10010
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 32, Local Router ID is 10.0.1.3
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 10.0.1.3:32777    (L2VNI 10010)
*>i[2]:[0]:[0]:[48]:[0050.0000.0c00]:[0]:[0.0.0.0]/216
                      10.0.5.1                          100          0 i
*>i[2]:[0]:[0]:[48]:[0050.0000.0c00]:[32]:[10.10.10.1]/248
                      10.0.5.1                          100          0 i
leaf03# show bgp l2vpn evpn rd 10.0.1.3:32777 0050.0000.0c00
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.0.1.3:32777    (L2VNI 10010)
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.0000.0c00]:[0]:[0.0.0.0]/216,
 version 23
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 10.0.1.1:32777:[2]:[0]:[0]:[48]:[0050.0000.0c00]:[0]:
[0.0.0.0]/216 
  AS-Path: NONE, path sourced internal to AS
    10.0.5.1 (metric 81) from 10.0.0.1 (10.0.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010
      Extcommunity: RT:65500:10010 ENCAP:8
      Originator: 10.0.1.1 Cluster list: 10.0.0.1 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.0000.0c00]:[32]:[10.10.10.1]/
248, version 28
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 10.0.1.1:32777:[2]:[0]:[0]:[48]:[0050.0000.0c00]:[32]
:[10.10.10.1]/248 
  AS-Path: NONE, path sourced internal to AS
    10.0.5.1 (metric 81) from 10.0.0.1 (10.0.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010
      Extcommunity: RT:65500:10010 ENCAP:8
      Originator: 10.0.1.1 Cluster list: 10.0.0.1 

  Path-id 1 not advertised to any peer
```
- Check the mac received remotely - leaf02

```
leaf03# show l2route evpn mac evi 10

Flags -(Rmac):Router MAC (Stt):Static (L):Local (R):Remote (V):vPC link 
(Dup):Duplicate (Spl):Split (Rcv):Recv (AD):Auto-Delete (D):Del Pending
(S):Stale (C):Clear, (Ps):Peer Sync (O):Re-Originated (Nho):NH-Override
(Pf):Permanently-Frozen, (Orp): Orphan

Topology    Mac Address    Prod   Flags         Seq No     Next-Hops            
                  
----------- -------------- ------ ------------- ---------- ---------------------
------------------
10          0050.0000.0c00 BGP    SplRcv        0          10.0.5.1 (Label: 1001


leaf03# show l2route mac topology 10

Flags -(Rmac):Router MAC (Stt):Static (L):Local (R):Remote (V):vPC link 
(Dup):Duplicate (Spl):Split (Rcv):Recv (AD):Auto-Delete (D):Del Pending
(S):Stale (C):Clear, (Ps):Peer Sync (O):Re-Originated (Nho):NH-Override
(Pf):Permanently-Frozen, (Orp): Orphan

Topology    Mac Address    Prod   Flags         Seq No     Next-Hops            
                  
----------- -------------- ------ ------------- ---------- ---------------------
------------------
10          0050.0000.0c00 BGP    SplRcv        0          10.0.5.1 (Label: 1001

```
