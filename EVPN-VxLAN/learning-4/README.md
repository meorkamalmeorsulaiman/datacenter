# Connecting the Fabric with External Network

We continue with tapping external network into the fabric before adding new tenant. Border leafs is where an external connected to the fabric. There are 2 types

## Lab walkthrough

We will follow the based configuration in the previous lab. However, we are going to add a external routers that going to announced `1.1.1.1` into the fabric. This configs is very straight forward to implement.

![LAB Diagram](https://github.com/meorkamalmeorsulaiman/DC-Tech/blob/EVPN-VxLAN/EVPN-VxLAN/diagrams/Labs%20Topo.png)

The border leaf has been configured same as the other leaf switches. However we added another routed port toward the external routers. We going to establish EBP Peering with the external routers

- Border leaf

```
interface Ethernet1/3
  description connected-to-INET-RTR1
  no switchport
  vrf member Customer-A
  ip address 10.10.101.1/24
  no shutdown
  vrf Customer-A
    address-family ipv4 unicast
      redistribute direct route-map TYPE-5-ROUTES
    neighbor 10.10.101.10
      remote-as 65502
      local-as 65501
      address-family ipv4 unicast
```

The route-map basically just to announce /24 routes into the fabric. This will help with silent host, since we have the /24 routes in EVPN

```
bleaf01(config-router-vrf-neighbor)# show route-map 
route-map TYPE-5-ROUTES, permit, sequence 10 
  Match clauses:
  Set clauses:
bleaf01(config-

```
Let's validate whether we learning something on the INET-RTR1

```
INET-RTR1#show ip bgp  
BGP table version is 82, local router ID is 10.10.101.10
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal, 
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter, 
              x best-external, a additional-path, c RIB-compressed, 
              t secondary path, L long-lived-stale,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>   1.1.1.0/30       0.0.0.0                  0         32768 ?
 *>   10.10.10.0/24    10.10.101.1              0             0 65501 65500 ?
 *>   10.10.10.1/32    10.10.101.1                            0 65501 65500 i
 *>   10.10.10.2/32    10.10.101.1                            0 65501 65500 i
 *>   10.10.20.0/24    10.10.101.1              0             0 65501 65500 ?
 *>   10.10.20.1/32    10.10.101.1                            0 65501 65500 i
 *>   10.10.90.0/30    10.10.101.1              0             0 65501 65500 ?
 *    10.10.101.0/24   10.10.101.1              0             0 65501 65500 ?
 *>                    0.0.0.0                  0         32768 ?
INET-RTR1#
```
Let's check the fabric from `leaf01` in the vrf table

```
leaf01(config)# show ip route vrf Customer-A 
IP Route Table for VRF "Customer-A"
'*' denotes best ucast next-hop
'**' denotes best mcast next-hop
'[x/y]' denotes [preference/metric]
'%<string>' in via output denotes VRF <string>

1.1.1.1/32, ubest/mbest: 1/0
    *via 10.0.5.5%default, [200/0], 00:00:17, bgp-65500, internal, tag 65501, segid: 100001 tunnelid: 0xa000505 encap: VXLAN
 
10.10.10.0/24, ubest/mbest: 1/0, attached
    *via 10.10.10.254, Vlan10, [0/0], 3d03h, direct
10.10.10.1/32, ubest/mbest: 1/0, attached
    *via 10.10.10.1, Vlan10, [190/0], 3d03h, hmm
10.10.10.2/32, ubest/mbest: 1/0
    *via 10.0.5.2%default, [200/0], 3d03h, bgp-65500, internal, tag 65500, segid: 100001 tunnelid: 0xa000502 encap: VXLAN
 
10.10.10.254/32, ubest/mbest: 1/0, attached
    *via 10.10.10.254, Vlan10, [0/0], 3d03h, local
10.10.20.0/24, ubest/mbest: 1/0, attached
    *via 10.10.20.254, Vlan20, [0/0], 3d03h, direct
10.10.20.1/32, ubest/mbest: 1/0
    *via 10.0.5.3%default, [200/0], 3d03h, bgp-65500, internal, tag 65500, segid: 100001 tunnelid: 0xa000503 encap: VXLAN
 
10.10.20.254/32, ubest/mbest: 1/0, attached
    *via 10.10.20.254, Vlan20, [0/0], 3d03h, local
10.10.90.0/30, ubest/mbest: 1/0, attached
    *via 10.10.90.2, Vlan90, [0/0], 1d01h, direct
10.10.90.2/32, ubest/mbest: 1/0, attached
    *via 10.10.90.2, Vlan90, [0/0], 1d01h, local
10.10.101.0/24, ubest/mbest: 1/0
    *via 10.0.5.5%default, [200/0], 00:13:33, bgp-65500, internal, tag 65500, segid: 100001 tunnelid: 0xa000505 encap: VXLAN
 

leaf01(config)# 
```

Let see in the EVPN

```
leaf01(config)# show bgp l2vpn evpn vrf Customer-A 
Route Distinguisher: 10.0.1.1:3    (L3VNI 100001)
<SNIPPED>
*>i[5]:[0]:[0]:[32]:[1.1.1.1]/224
                      10.0.5.5                 0        100          0 65501 65502 ?

leaf01(config)# 
```
We can see the there is type 5 route inserted which is IP subnet prefix...that's all for the section. Not really fancy. Hang on!! Below are the ping from endpoint `VLAN10`

```
root@ubuntu22-server:~# ping 1.1.1.1
PING 1.1.1.1 (1.1.1.1) 56(84) bytes of data.
64 bytes from 1.1.1.1: icmp_seq=1 ttl=253 time=8.76 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=253 time=6.37 ms
64 bytes from 1.1.1.1: icmp_seq=3 ttl=253 time=9.23 ms
64 bytes from 1.1.1.1: icmp_seq=4 ttl=253 time=7.96 ms
^C
--- 1.1.1.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3005ms
rtt min/avg/max/mdev = 6.374/8.083/9.234/1.086 ms
root@ubuntu22-server:~# 

```
