# EVPN-VxLAN

## Introduction

This will be the notes for EVPN-VxLAN learning. EVPN-VxLAN is a separate technology. EVPN is a subset of l2vpn which carries ethernet information while is an overlay ethernet technology that encapsulate ethernet into a UDP and send it over an IP. Enough, hebat tak aku sembang.

So, let get things simpler. Typical network that run's ethernet will need to a segment or network which is called a LAN. There is additional value which we usually call tag or 802.1q tag. This tag allow us to separate or isolate network into VLAN. So you separate a network using a tag. However, VLAN is just a broadcast domain that never get routed. At some point you want to extend your network with the same IP scheme.

What we usually do is to trunk it, which extending the broadcast area. It is not good, why? Create unnecessary traffic, the reliance of STP to avoid loop. It takes time or a guaranteed time of convergence for STP to put it back into forwarding state or if it in complete loop. One of your switchport will be in blocking state. Rugi boss!! There are more reasons but we keep simple.

So what VxLAN do here? It allow us to transport the same VLAN across routed link. Give an example, if you have 2 houses and both want to use `192.168.100.0/24` subnet. You can build a VxLAN tunnel between your house across the internet and terminate on your broadband.

Then, what the hack is EVPN? Well, it is a protocol that learn mac addresses or ethernet information. EVPN is a subset of BGP l2vpn AFI. So you can route mac address between switch over IP. Erm...doesn't seems convincing. Normal switch learn mac address by looking at frames that it's received. The source mac address coming into a port an stored it into a mac table or cam table. So, to learn the entire subnets it require a method called flood and learn. A host flood it's mac via ARP or GARP etc. All the switches populate it's cam table. Instead of flood and learn, EVPN allow switch to share those mac addresses locally and transport it to a remote switch using BGP. That's the gist of EVPN and famous word of EVPN this technique called Control-plane learning.

Combining these two, we can achieve small footprint to mac learning - Control-plane learning and scale up your VLAN. More benefit, will discussed while looking we going thru the lab

## How it works

When a end-host tries to sent a traffic toward it's destination. The traffic will be forwarded to the remote leaf using a tunnel called VTEP (Virutal Tunnel Endpoint). VTEP is the overlay tunnels that created for VxLAN to forward traffic. Figure below illustrated the packet forwarding in EVPN-VxLAN fabric.

![Unicast Traffic](https://github.com/meorkamalmeorsulaiman/DC-Tech/blob/EVPN-VxLAN/EVPN-VxLAN/diagrams/EVPN-VxLAN-Unicast%20Traffic.jpg)

These tunnels create between leafs switches, it can be p2p tunnels and dynamic. That's it...This is base of EVPN-VxLAN.

## BUM Traffic?

So VLAN also need to work with BUM traffic right? Broadcast, Unknown Unicast and Multicast. Then how this fabric forward these traffic? Well, it's either you full-mesh all the leafs VTEPs to one another or you use more scalable option with multicast.

![Overlay](https://github.com/meorkamalmeorsulaiman/DC-Tech/blob/EVPN-VxLAN/EVPN-VxLAN/diagrams/EVPN-VxLAN-Overlay.jpg)

Figure above illustrate the overlay networking of EVPN-VxLAN. What happen is that the spines switches act as a hub to all the leafs. The use of multicast help to forward all the BUM traffic from originating leaf to all the remaining leafs. A little bit about multicast traffic, it is sent by someone to a group of people. In this case, **the originating leaf is the someone and the remote leafs is the group of people** Below illustrates the BUM traffic originating from `10.10.10.1` and get's spread across the fabric.

![BUM Traffic](https://github.com/meorkamalmeorsulaiman/DC-Tech/blob/EVPN-VxLAN/EVPN-VxLAN/diagrams/EVPN-VxLAN-BUM%20Traffic.jpg)

__*Note: The service leafs `sleaf[X]` doesn't as the scope for this notes only for traffic forward between leafs.*__


## BGP-EVPN?

So we have talked about VxLAN for the most part until now... The question raised, why or what is the purpose of BGP here? Hmm... It's not much but it's very much changing how typical switch work. Remember in the earlier part of the notes where have talked about control-plane learning. So here again, trying to keep the fundamentals correct! Normal switch populate mac addresses by flood and learn mechanic. When a incoming frame reaches the switch port, the switch will tangkap and store it mac address. Then, forward to either all port excluding the originating port or specific port. Switches forward frames using mac address, a layer 2 field. When we use BGP-EVPN, the mac addresses are shared using BGP. How? Below illustrate to give some clarity

![Control-plane](https://github.com/meorkamalmeorsulaiman/DC-Tech/blob/EVPN-VxLAN/EVPN-VxLAN/diagrams/EVPN-VxLAN-Control-Plan%20Learning.jpg)

When a frame hit the VTEP, it captures it and sent's relevant info toward other BGP peers. Above shows the mac advertises to the spines and then to other remote leafs in the fabric. Example of remotely learned mac address by a leaf in the fabric:

```
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
The spines is act as a BGP route-reflectors between the leafs in the fabric. If you recall a MPLS L3VPN, where the core routers `P` only run MPLS and the `PE` that does all the clever jobs. The same approach brought into data center with all the hard work's run by the leafs and the spines only act as a hub connecting others. Cool!!

This control-plane learning change how normal switch learn mac addresses - **flood and learn** vs **control-plane learning**. Even more gangster, a feature called `ARP Suppresion` which suppress ARP frames. When a switch received an ARP destined to `FFFF.FFFF.FFFF` the switch will intercept. Validate it's ARP suppression cache table for known IP hosts and response on behave. This method convert's broadcast to unicast traffic which is very efficient. In case there are no cache, then it will flood and learn - BUM traffic.
