# EVPN-VxLAN

This would be the labs for EVPN-VxLAN implementation for enterprise campus network. This section is the underlay walkthrough.

## How to make it work

### Labs Setup

The lab would required an Junos vQFX switches. Below is the lab environment setup

![Lab campus evpn](https://github.com/meorkamalmeorsulaiman/DC-Tech/blob/Campus-EVPN-VxLAN/Campus-EVPN-VxLAN/diagrams/Collapse%20Core%20Lab.png)

## Building the Underlay

`core01` and `core02` is a p2p peers and would establish iBGP as the underlay. You may choose any routing to exchange the loopback IP addresses to build the overlay connection.

### Point to Point Interfaces

core01

```
root@core01> show configuration interfaces xe-0/0/10 | display set 
set interfaces xe-0/0/10 description "Connected to core02"
set interfaces xe-0/0/10 mtu 9216
set interfaces xe-0/0/10 unit 0 family inet address 10.10.1.1/30

{master:0}
root@core01> show configuration interfaces xe-0/0/11 | display set    
set interfaces xe-0/0/11 description "Connected to core02"
set interfaces xe-0/0/11 mtu 9216
set interfaces xe-0/0/11 unit 0 family inet address 10.10.1.5/30

{master:0}
root@core01> show configuration interfaces lo0 | display set 
set interfaces lo0 unit 0 family inet address 10.10.0.1/32

{master:0}
root@core01> 
```

core02

```
root@core02> show configuration interfaces xe-0/0/10 | display set 
set interfaces xe-0/0/10 description "Connected to core01"
set interfaces xe-0/0/10 mtu 9216
set interfaces xe-0/0/10 unit 0 family inet address 10.10.1.2/30

{master:0}
root@core02> show configuration interfaces xe-0/0/11 | display set    
set interfaces xe-0/0/11 description "Connected to core01"
set interfaces xe-0/0/11 mtu 9216
set interfaces xe-0/0/11 unit 0 family inet address 10.10.1.6/30

{master:0}
root@core02> show configuration interfaces lo0 | display set 
set interfaces lo0 unit 0 family inet address 10.10.0.2/32

{master:0}
root@core02> 
```

### The Underlay ECMP and route filters

We will proceed with bringing up the iBGP session between these pair of switches, we going to turn on ECMP for the underlay then announce & receive `/32` loopback IP addresses. The filters are optionals but this is based on Juniper guides

core01

```
set policy-options policy-statement ECMP_POLICY then load-balance per-packet
set policy-options policy-statement ECMP_POLICY then accept
set policy-options policy-statement UNDERLAY_IMPORT term LOOPBACK from route-filter 10.10.0.2/32 exact
set policy-options policy-statement UNDERLAY_IMPORT term LOOPBACK then accept
set policy-options policy-statement UNDERLAY_IMPORT term DEFAULT then reject
set policy-options policy-statement UNDERLAY_EXPORT term LOOPBACK from route-filter 10.10.0.1/32 exact
set policy-options policy-statement UNDERLAY_EXPORT term LOOPBACK then accept
set policy-options policy-statement UNDERLAY_EXPORT term DEFAULT then reject
```

core02

```
set policy-options policy-statement ECMP_POLICY then load-balance per-packet
set policy-options policy-statement ECMP_POLICY then accept
set policy-options policy-statement UNDERLAY_IMPORT term LOOPBACK from route-filter 10.10.0.1/32 exact
set policy-options policy-statement UNDERLAY_IMPORT term LOOPBACK then accept
set policy-options policy-statement UNDERLAY_IMPORT term DEFAULT then reject
set policy-options policy-statement UNDERLAY_EXPORT term LOOPBACK from route-filter 10.10.0.2/32 exact
set policy-options policy-statement UNDERLAY_EXPORT term LOOPBACK then accept
set policy-options policy-statement UNDERLAY_EXPORT term DEFAULT then reject
```

### The Underlay iBGP Configuration

We set the router IDs, AS and turning on ECMP

core01

```
set routing-options router-id 10.10.1.1
set routing-options autonomous-system 65500
set routing-options forwarding-table export ECMP_POLICY

```

core02

```
set routing-options router-id 10.10.1.2
set routing-options autonomous-system 65500
set routing-options forwarding-table export ECMP_POLICY
```

### The iBGP Configuration

We create the underlay bgp group and also include bfd which is optional. Furthermore, we have a 2 point-to-point links between the switches thus we also going to establish 2 session between the pair.

core01

```
set protocols bgp group UNDERLAY type internal
set protocols bgp group UNDERLAY description "Connection for Underlay - BGP"
set protocols bgp group UNDERLAY import UNDERLAY_IMPORT
set protocols bgp group UNDERLAY family inet unicast
set protocols bgp group UNDERLAY export UNDERLAY_EXPORT
set protocols bgp group UNDERLAY bfd-liveness-detection minimum-interval 350
set protocols bgp group UNDERLAY bfd-liveness-detection multiplier 3
set protocols bgp group UNDERLAY multipath multiple-as
set protocols bgp group UNDERLAY neighbor 10.10.0.2 peer-as 65500
set protocols bgp group UNDERLAY neighbor 10.10.0.6 peer-as 65500
```

core02 

```
set protocols bgp group UNDERLAY type internal
set protocols bgp group UNDERLAY description "Connection for Underlay - BGP"
set protocols bgp group UNDERLAY import UNDERLAY_IMPORT
set protocols bgp group UNDERLAY family inet unicast
set protocols bgp group UNDERLAY export UNDERLAY_EXPORT
set protocols bgp group UNDERLAY bfd-liveness-detection minimum-interval 350
set protocols bgp group UNDERLAY bfd-liveness-detection multiplier 3
set protocols bgp group UNDERLAY multipath multiple-as
set protocols bgp group UNDERLAY neighbor 10.10.0.1 peer-as 65500
set protocols bgp group UNDERLAY neighbor 10.10.0.5
```

Once commited, validate that the peering are established

core01

```
root@core01# run show bgp summary group UNDERLAY      
Groups: 1 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0               
                       2          2          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.10.0.2             65500         64         63       0       0       27:10 1/1/1/0              0/0/0/0
10.10.0.6             65500         53         52       0       0       22:56 1/1/1/0              0/0/0/0

{master:0}[edit]
root@core01# run show bfd session brief 
                                                  Detect   Transmit
Address                  State     Interface      Time     Interval  Multiplier
10.10.0.2                Up        xe-0/0/10.0    1.050     0.350        3   
10.10.0.6                Up        xe-0/0/11.0    1.050     0.350        3   

2 sessions, 2 clients
Cumulative transmit rate 5.7 pps, cumulative receive rate 5.7 pps

{master:0}[edit]
root@core01# 

```

core02

```
root@core02# run show bgp summary group UNDErLAY   
Groups: 1 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0               
                       2          2          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.10.0.1             65500         65         64       0       0       28:03 1/1/1/0              0/0/0/0
10.10.0.5             65500         56         54       0       0       23:49 1/1/1/0              0/0/0/0

{master:0}[edit]
root@core02# run show bfd session brief 
                                                  Detect   Transmit
Address                  State     Interface      Time     Interval  Multiplier
10.10.0.1                Up        xe-0/0/10.0    1.050     0.350        3   
10.10.0.5                Up        xe-0/0/11.0    1.050     0.350        3   

2 sessions, 2 clients
Cumulative transmit rate 5.7 pps, cumulative receive rate 5.7 pps

{master:0}[edit]
root@core02# 
```

Moving on and validate the route received

core01

```
root@core01> show route table inet.0 10.10.1.2 

inet.0: 8 destinations, 9 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.10.1.2/32       *[BGP/170] 00:00:20, localpref 100
                      AS path: I, validation-state: unverified
                    > to 10.10.0.2 via xe-0/0/10.0
                      to 10.10.0.6 via xe-0/0/11.0
                    [BGP/170] 00:00:20, localpref 100
                      AS path: I, validation-state: unverified
                    > to 10.10.0.6 via xe-0/0/11.0

{master:0}
root@core01>
```

core02

```
root@core02> show route table inet.0 10.10.1.1 

inet.0: 8 destinations, 9 routes (8 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.10.1.1/32       *[BGP/170] 00:01:05, localpref 100
                      AS path: I, validation-state: unverified
                    > to 10.10.0.1 via xe-0/0/10.0
                      to 10.10.0.5 via xe-0/0/11.0
                    [BGP/170] 00:01:05, localpref 100
                      AS path: I, validation-state: unverified
                    > to 10.10.0.5 via xe-0/0/11.0

{master:0}
root@core02> 
```

That's for the underlay setup for this EVPN-VxLAN section. We will continue with the overlay setup in the next lab.
