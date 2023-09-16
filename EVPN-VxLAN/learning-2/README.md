# Adding another VLAN and route it across VxLAN Tunnels

This section will expand the VLAN usage in the fabric by adding another VLAN `20` Below illustrate what we want to do in this section

![Adding another VLAN](https://github.com/meorkamalmeorsulaiman/DC-Tech/blob/EVPN-VxLAN/EVPN-VxLAN/diagrams/lab02/EVPN-VxLAN-2-Adding%20another%20VLAN.png)


## Lab walkthrough

We will follow the based configuration in the previous lab. However, we going to add another VLAN for Customer-A and attached it to a VNI `10020` on every leafs

```
vlan 20
  name Customer-A-VL20
  vn-segment 10020
```

Once added, we proceed with the typical fabric provision configs which includes:

- EVPN Advertisement
- Attach to VTEP
- Link it the our multicast group
- Create anycast gateway SVI

```
evpn
  vni 10010 l2
    rd auto
    route-target import auto
    route-target export auto
  vni 10020 l2
    rd auto
    route-target import auto
    route-target export auto
```

Below where we attached the vni to our VTEP for overlay VxLAN tunnel

```
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback1
  member vni 10010
    mcast-group 239.0.0.1
  member vni 10020
    mcast-group 239.0.0.1
```

Anycast gateway configs

```
interface Vlan20
  no shutdown
  ip address 10.10.20.254/24
  fabric forwarding mode anycast-gateway

```

At this point, you will be able have connectivity to the anycast gateway and also it is important to validate the EVPN advertisement

### Validate EVPN advertisement

You can validate using similar approach as previously done. Where we can first check the locally learn routes

```
leaf03(config)# show mac address-table vlan 20
Legend: 
        * - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
        age - seconds since last seen,+ - primary entry using vPC Peer-Link,
        (T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
*   20     0050.0000.0e00   dynamic  0         F      F    Eth1/3
```

Check the L2 routing information base for the new VLAN mac address

```
leaf03(config)# show l2route evpn mac evi 20

Flags -(Rmac):Router MAC (Stt):Static (L):Local (R):Remote (V):vPC link 
(Dup):Duplicate (Spl):Split (Rcv):Recv (AD):Auto-Delete (D):Del Pending
(S):Stale (C):Clear, (Ps):Peer Sync (O):Re-Originated (Nho):NH-Override
(Pf):Permanently-Frozen, (Orp): Orphan

Topology    Mac Address    Prod   Flags         Seq No     Next-Hops            
                  
----------- -------------- ------ ------------- ---------- ---------------------
------------------
20          0050.0000.0e00 Local  L,            0          Eth1/3               
                  
leaf03(config)# 

```

Then, validate whether the route advertise to both spines from the L2VNI table

```
leaf03(config)# show bgp l2vpn evpn 0050.0000.0e00
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.0.1.3:32787    (L2VNI 10020)
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.0000.0e00]:[0]:[0.0.0.0]/216,
 version 40
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    10.0.5.3 (metric 0) from 0.0.0.0 (10.0.1.3)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10020
      Extcommunity: RT:65500:10020 ENCAP:8

  Path-id 1 advertised to peers:
    10.0.0.1           10.0.0.2       
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.0000.0e00]:[32]:[10.10.20.1]/
272, version 35
Paths: (1 available, best #1)
Flags: (0x000102) (high32 00000000) on xmit-list, is not in l2rib/evpn

  Advertised path-id 1
  Path type: local, path is valid, is best path, no labeled nexthop
  AS-Path: NONE, path locally originated
    10.0.5.3 (metric 0) from 0.0.0.0 (10.0.1.3)
      Origin IGP, MED not set, localpref 100, weight 32768
      Received label 10020
      Extcommunity: RT:65500:10020 ENCAP:8 Router MAC:5005.0000.
1b08

  Path-id 1 advertised to peers:
    10.0.0.1           10.0.0.2       

leaf03(config)#
```

Validate that the route received from the remote leafs. You noticed that the rd generated from the `ROUTER ID` + `32767` + `VLAN ID` Routes advertise by remote peers are VPN-EVPN routes (RD) with route-target attached. Make sense to validate whether you received the vpn route before it's imported

```
leaf01(config)# show bgp l2vpn evpn rd 10.0.1.3:32787
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.0.1.3:32787
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.0000.0e00]:[0]:[0.0.0.0]/216,
 version 36
Paths: (2 available, best #1)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10020
  AS-Path: NONE, path sourced internal to AS
    10.0.5.3 (metric 81) from 10.0.0.1 (10.0.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10020
      Extcommunity: RT:65500:10020 ENCAP:8
      Originator: 10.0.1.3 Cluster list: 10.0.0.1 

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  AS-Path: NONE, path sourced internal to AS
    10.0.5.3 (metric 81) from 10.0.0.2 (10.0.0.2)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10020
      Extcommunity: RT:65500:10020 ENCAP:8
      Originator: 10.0.1.3 Cluster list: 10.0.0.2 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.0000.0e00]:[32]:[10.10.20.1]/
272, version 31
Paths: (2 available, best #2)
Flags: (0x000202) (high32 00000000) on xmit-list, is not in l2rib/evpn, is not i
n HW

  Path type: internal, path is valid, not best reason: Neighbor Address, no labe
led nexthop
  AS-Path: NONE, path sourced internal to AS
    10.0.5.3 (metric 81) from 10.0.0.2 (10.0.0.2)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10020
      Extcommunity: RT:65500:10020 ENCAP:8 Router MAC:5005.0000.
1b08
      Originator: 10.0.1.3 Cluster list: 10.0.0.2 

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop
             Imported to 1 destination(s)
             Imported paths list: L2-10020
  AS-Path: NONE, path sourced internal to AS
    10.0.5.3 (metric 81) from 10.0.0.1 (10.0.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10020
      Extcommunity: RT:65500:10020 ENCAP:8 Router MAC:5005.0000.

```

Validate that the route imported to the local L2VNI table, this is where the route-target effect - to import the VPN routes (RT).

```
leaf01(config)# show bgp l2vpn evpn VNi-id 10020
BGP routing table information for VRF default, address family L2VPN EVPN
BGP table version is 37, Local Router ID is 10.0.1.1
Status: s-suppressed, x-deleted, S-stale, d-dampened, h-history, *-valid, >-best
Path type: i-internal, e-external, c-confed, l-local, a-aggregate, r-redist, I-i
njected
Origin codes: i - IGP, e - EGP, ? - incomplete, | - multipath, & - backup, 2 - b
est2

   Network            Next Hop            Metric     LocPrf     Weight Path
Route Distinguisher: 10.0.1.1:32787    (L2VNI 10020)
*>i[2]:[0]:[0]:[48]:[0050.0000.0e00]:[0]:[0.0.0.0]/216
                      10.0.5.3                          100          0 i
*>i[2]:[0]:[0]:[48]:[0050.0000.0e00]:[32]:[10.10.20.1]/272
                      10.0.5.3                          100          0 i

leaf01(config)# 
```

You can really see it was imported:

```
leaf01(config)# show bgp l2vpn evpn 0050.0000.0e00
BGP routing table information for VRF default, address family L2VPN EVPN
Route Distinguisher: 10.0.1.1:32787    (L2VNI 10020)
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.0000.0e00]:[0]:[0.0.0.0]/216,
 version 37
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 10.0.1.3:32787:[2]:[0]:[0]:[48]:[0050.0000.0e00]:[0]:
[0.0.0.0]/216 
  AS-Path: NONE, path sourced internal to AS
    10.0.5.3 (metric 81) from 10.0.0.1 (10.0.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10020
      Extcommunity: RT:65500:10020 ENCAP:8
      Originator: 10.0.1.3 Cluster list: 10.0.0.1 

  Path-id 1 not advertised to any peer
BGP routing table entry for [2]:[0]:[0]:[48]:[0050.0000.0e00]:[32]:[10.10.20.1]/
272, version 32
Paths: (1 available, best #1)
Flags: (0x000212) (high32 00000000) on xmit-list, is in l2rib/evpn, is not in HW

  Advertised path-id 1
  Path type: internal, path is valid, is best path, no labeled nexthop, in rib
             Imported from 10.0.1.3:32787:[2]:[0]:[0]:[48]:[0050.0000.0e00]:[32]
:[10.10.20.1]/272 
  AS-Path: NONE, path sourced internal to AS
    10.0.5.3 (metric 81) from 10.0.0.1 (10.0.0.1)
      Origin IGP, MED not set, localpref 100, weight 0
      Received label 10020
      Extcommunity: RT:65500:10020 ENCAP:8 Router MAC:5005.0000.
1b08
      Originator: 10.0.1.3 Cluster list: 10.0.0.1 

  Path-id 1 not advertised to any peer
```

Validate that the it populated the L2 routing information base

```
leaf01(config)# show l2route evpn mac evi 20

Flags -(Rmac):Router MAC (Stt):Static (L):Local (R):Remote (V):vPC link 
(Dup):Duplicate (Spl):Split (Rcv):Recv (AD):Auto-Delete (D):Del Pending
(S):Stale (C):Clear, (Ps):Peer Sync (O):Re-Originated (Nho):NH-Override
(Pf):Permanently-Frozen, (Orp): Orphan

Topology    Mac Address    Prod   Flags         Seq No     Next-Hops            
                  
----------- -------------- ------ ------------- ---------- ---------------------
------------------
20          0050.0000.0e00 BGP    SplRcv        0          10.0.5.3 (Label: 1002
0)                
leaf01(config)# 
```

Then finally you validate the local CAM table populated:

```
leaf01(config)# show mac address-table vlan 20
Legend: 
	* - primary entry, G - Gateway MAC, (R) - Routed MAC, O - Overlay MAC
	age - seconds since last seen,+ - primary entry using vPC Peer-Link,
	(T) - True, (F) - False, C - ControlPlane MAC, ~ - vsan
   VLAN     MAC Address      Type      age     Secure NTFY Ports
---------+-----------------+--------+---------+------+----+------------------
C   20     0050.0000.0e00   dynamic  0         F      F    nve1(10.0.5.3)
```

At the time of writing, we only have 1 endpoint for `VLAN20` connected to `leaf03`. This is just to reenforce understanding. Next lab will be working on routing between vlan that we have created `VLAN10` <-> `VLAN20
`
