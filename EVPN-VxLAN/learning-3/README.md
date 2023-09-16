# Routing between VLAN or Inter-VxLAn or IRB

This section will expand the VLAN usage in the fabric by adding another VLAN `20` Below illustrate what we want to do in this section


## Lab walkthrough

All the switches now have both `VLAN10` and `VLAN20`, we require for both of them to be routed in between. Similar like Inter-vlan routing but in EVPN-VxLAN it called Intergrated Routing and Bridging (IRB). It require an L3VNI for routing that will be use to bridge between L2VNI attached to a VTEP. There is another method of routing between VxLAN, but we will talk about symmetric IRB.

First you have to create a new VLAN and attached to a VNI

```
vlan 10
  name Customer-A-VL10
  vn-segment 10010
vlan 20
  name Customer-A-VL20
  vn-segment 10020
vlan 100
  name Customer-A-L3VNI
  vn-segment 100001
vlan 101
  vn-segment 10001
```

I have put some naming which we will see later see it relation. Moving on, we create an new VRF what will import and export IP and EVPN

```
vrf context Customer-A
  vni 100001
  rd auto
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn
router bgp 65500
  vrf Customer-A
    address-family ipv4 unicast
```

Once configured associate all the VLANs into the new IP VRF

```
interface Vlan10
  no shutdown
  vrf member Customer-A
  ip address 10.10.10.254/24
  fabric forwarding mode anycast-gateway
interface Vlan20
  no shutdown
  vrf member Customer-A
  ip address 10.10.20.254/24
  fabric forwarding mode anycast-gateway
```
Then create the L3 IRB that bridges between the L2VNI

```
interface Vlan100
  no shutdown
  vrf member Customer-A
  ip forward
```

Finally associate the VTEP with the new VRF

```
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  member vni 10010
    mcast-group 239.0.0.1
  member vni 10020
    mcast-group 239.0.0.1
  member vni 100001 associate-vrf
```

All of these configs are applied to all of the leafs

## What happen here?

At this point I also clueless, but what I can tell it generate IPv4 Unicast and EVPN BGP Updates. So we look at the originating leaf which host's VLAN10 and it endpoint MAC is `0050.0000.0c00`.

Look at the local L2VNI table and the local mac exist

```
leaf01(config)# show bgp l2vpn evpn vni-id 10010
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 43, Local Router ID is 10.0.1.1
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 10.0.1.1:32777    (L2VNI 10010)
*>l[2]:[0]:[0]:[48]:[0050.0000.0c00]:[0]:[0.0.0.0]/216
                      10.0.5.1                          100      32768 i
*>i[2]:[0]:[0]:[48]:[0050.0000.0d00]:[0]:[0.0.0.0]/216
                      10.0.5.2                          100          0 i
*>l[2]:[0]:[0]:[48]:[0050.0000.0c00]:[32]:[10.10.10.1]/272
                      10.0.5.1                          100      32768 i
*>i[2]:[0]:[0]:[48]:[0050.0000.0d00]:[32]:[10.10.10.2]/272
                      10.0.5.2                          100          0 i
```

We also can see that the local L3VNI doesn't have the locally learn MAC endpoint

```
leaf01(config)# show bgp l2vpn evpn vni-id 100001
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 43, Local Router ID is 10.0.1.1
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 10.0.1.1:3    (L3VNI 100001)
*>i[2]:[0]:[0]:[48]:[0050.0000.0d00]:[32]:[10.10.10.2]/272
                      10.0.5.2                          100          0 i
*>i[2]:[0]:[0]:[48]:[0050.0000.0e00]:[32]:[10.10.20.1]/272
                      10.0.5.3                          100          0 i

leaf01(config)# 
```

Hmm..weird. But at the remote-end we see it - `leaf03`

```
leaf03(config)# show bgp l2vpn evpn vni-id 100001
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 42, Local Router ID is 10.0.1.3
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 10.0.1.3:3    (L3VNI 100001)
*>i[2]:[0]:[0]:[48]:[0050.0000.0c00]:[32]:[10.10.10.1]/272
                      10.0.5.1                          100          0 i
*>i[2]:[0]:[0]:[48]:[0050.0000.0d00]:[32]:[10.10.10.2]/272
                      10.0.5.2                          100          0 i

leaf03(config)# 
```

So how it get's transport. If we look at the EVPN-VPN routes on `leaf03` received from `leaf01`. There are route-target append `Extcommunity: RT:65500:10010 RT:65500:100001 ENCAP:8 Router MAC:5003.0000.`

```
leaf03(config)# show bgp l2vpn evpn rd 10.0.1.1:32777
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.0.1.1:32777
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.0000.0c00]:[0]:[0.0.0.0]/216,
 version 16
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

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.0000.0c00]:[32]:[10.10.10.1]/
272, version 36
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  AS-Path: NONE, path sourced internal to AS
    10.0.5.1 (metric 81) from 10.0.0.2 (10.0.0.2)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010 100001
      Extcommunity: RT:65500:10010 RT:65500:100001 ENCAP:8 Router MAC:5003.0000.
1b08
      Originator: 10.0.1.1 Cluster list: 10.0.0.2 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 3 destination(s)
             Imported paths list: Customer-A L3-100001 L2-10010
  AS-Path: NONE, path sourced internal to AS
    10.0.5.1 (metric 81) from 10.0.0.1 (10.0.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010 100001
      Extcommunity: RT:65500:10010 RT:65500:100001 ENCAP:8 Router MAC:5003.0000.
1b08
      Originator: 10.0.1.1 Cluster list: 10.0.0.1 

  Path-id 1 not advertised to any peer

leaf03(config)# 
```

So here the gist, the VPN routes get imported to L3VNI tables on `leaf03` that learnt from `leaf01`. It's even imported into IP VRF `Imported paths list: Customer-A L3-100001 L2-10010`

```
leaf03(config)# show bgp l2vpn evpn vni-id 100001 detail 
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.0.1.3:3    (L3VNI 100001)
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.0000.0c00]:[32]:[10.10.10.1]/
272, version 38
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 10.0.1.1:32777:[2]:[0]:[0]:[48]:[0050.0000.0c00]:[32]
:[10.10.10.1]/272 
  AS-Path: NONE, path sourced internal to AS
    10.0.5.1 (metric 81) from 10.0.0.1 (10.0.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010 100001
      Extcommunity: RT:65500:10010 RT:65500:100001 ENCAP:8 Router MAC:5003.0000.
1b08
      Originator: 10.0.1.1 Cluster list: 10.0.0.1 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.0000.0d00]:[32]:[10.10.10.2]/
272, version 34
Paths: (1 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported from 10.0.1.2:32777:[2]:[0]:[0]:[48]:[0050.0000.0d00]:[32]
:[10.10.10.2]/272 
  AS-Path: NONE, path sourced internal to AS
    10.0.5.2 (metric 81) from 10.0.0.1 (10.0.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10010 100001
      Extcommunity: RT:65500:10010 RT:65500:100001 ENCAP:8 Router MAC:5004.0000.
1b08
      Originator: 10.0.1.2 Cluster list: 10.0.0.1 

  Path-id 1 not advertised to any peer
```

Above shows the route get imported into L3VNI table `Imported from 10.0.1.2:32777:[2]:[0]:[0]:[48]:[0050.0000.0d00]:[32]
:[10.10.10.2]/272` and import into IP VRF as it also importing EVPN routes

```
leaf03(config)# show bgp l2vpn evpn vni-id 100001 vrf Customer-A 
Route Distinguisher: 10.0.1.3:3    (L3VNI 100001)
*>i[2]:[0]:[0]:[48]:[0050.0000.0c00]:[32]:[10.10.10.1]/272
                      10.0.5.1                          100          0 i
*>i[2]:[0]:[0]:[48]:[0050.0000.0d00]:[32]:[10.10.10.2]/272
                      10.0.5.2                          100          0 i
```

Which then populated the routing table of that VRF

```
leaf03(config)# show ip route vrf Customer-A 
IP Route Table for VRF "Customer-A"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

10.10.10.0/24, ubest/mbest: 1/0, attached
    *via 10.10.10.254, Vlan10, [0/0], 01:41:32, direct
10.10.10.1/32, ubest/mbest: 1/0
    *via 10.0.5.1%default, [200/0], 01:30:40, bgp-65500, internal, tag 65500, se
gid: 100001 tunnelid: 0xa000501 encap: VXLAN
```

Finally it's populate the L2RIB via EVPN-L2VNI table. So the MAC and IP was tagged with L2VNI, L3VNI and IP VRF route-target. These route-target in-turn used to inject into different EVPN table:

- IP VRF - EVPN table
- L2VNI - EVPN table
- L3VNI - EVPN table

The L2VNI - EVPN table is normally use when shared MAC within IP subnet. Inter-VxLAN routing use additional table and route-target to learn the MAC and IP binding from different IP subnet. Easy to understand BGP? What's next??? We play around with multi-tenancy. This lab shown that remote route land into specific routing context `Customer-A` which is isolated from the common routing table. Full `leaf01` and `leaf02` are here: [Symmetric IRB](https://github.com/meorkamalmeorsulaiman/DC-Tech/tree/EVPN-VxLAN/EVPN-VxLAN/configs/IRB-02)
