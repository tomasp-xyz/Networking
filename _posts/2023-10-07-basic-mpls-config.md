---
layout: post
title: "Basic MPLS configuration [OSPF IGP+BGP]"
categories: misc
---

I've wanted to configure IP/MPLS based network for a while now, but never got around to it, because MPLS was simply too tough of a topic to handle. However, it is one of the basic requirements for ENARSI so basic knowledge of it is going to be mandatory to pass the certification so I might as well get to work :)

Note - this is a learning experience from https://www.rogerperkin.co.uk/ccie/mpls/cisco-mpls-tutorial/ - I must leave credit and a thanks for the fine work of Mr. Roger.

# Step 1: topology & config preparation
For this lab, I shall use 1 router (R3) to represent P router, 2 routers (R2 & R4) as PE and 2 routers (R1 & R5) as CPE

![image](https://github.com/tomasp-xyz/Networking/assets/108157159/315f2349-f91f-4711-ba61-4f7f9c0947c8)

Below is LLD of addressing in this topology (using /31s as I've seen them in MPLS network before and quite like the idea of addressing reservation):

| Router name | Loopback IP | g0/0 IP | g0/1 IP |
| ------------- | ------------- | ------------- | ------------- |
| R1 | 1.1.1.1 | 10.0.0.0/31 | N/A |
| R2 | 2.2.2.2 | 10.0.0.1/31 | 10.0.0.2/31 |
| R3 | 3.3.3.3 | 10.0.0.3/31 | 10.0.0.4/31 |
| R4 | 4.4.4.4 | 10.0.0.5/31 | 10.0.0.6/31 |
| R5 | 5.5.5.5 | 10.0.0.7/31 | N/A |

Basic config below. I shall be using interface-based OSPF as the underlay IGP.
<details>
  <summary>Click me for basic config</summary>
  
R1: 
```
en
conf t
int lo0
ip add 1.1.1.1
ip ospf 1 area 0
int g0/0
ip add 10.0.0.0 255.255.255.254
ip ospf 1 area 0
no shutd
end
```

R2:
```
en
conf t
int lo0
ip add 2.2.2.2
ip ospf 1 area 0
int g0/0
ip add 10.0.0.1 255.255.255.254
ip ospf 1 area 0
no shutd
int g0/1
ip add 10.0.0.2 255.255.255.254
ip ospf 1 area 0
no shutd
end
```

R3:
```
en
conf t
int lo0
ip add 3.3.3.3 255.255.255.255
ip ospf 1 area 0
int g0/0
ip add 10.0.0.3 255.255.255.254
ip ospf 1 area 0
no shutd
int g0/1
ip add 10.0.0.4 255.255.255.254
ip ospf 1 area 0
no shutd
end
```

R4:
```
en
conf t
int lo0
ip add 4.4.4.4 255.255.255.255
ip ospf 1 area 0
int g0/0
ip add 10.0.0.5 255.255.255.254
ip ospf 1 area 0
no shutd
int g0/1
ip add 10.0.0.6 255.255.255.254
ip ospf 1 area 0
no shutd
end
```

R5:
```
en
conf t
int lo0
ip add 5.5.5.5 255.255.255.255
ip ospf 1 area 0
int g0/0
ip add 10.0.0.7 255.255.255.254
ip ospf 1 area 0
no shutd
end
```
</details>

Alright, now we should have:
* Pingable physical interfaces
* Pingable loopbacks

Verify by running pings from R1 to R5. I'm using specific source addressing just to be precise, but we can even ping physical interface from loopbacks and vice versa, because all routes are known inside OSPF area 0.
`ping 5.5.5.5 so lo0`
```
R1#ping 5.5.5.5 so lo0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 5.5.5.5, timeout is 2 seconds:
Packet sent with a source address of 1.1.1.1
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/3/4 ms
```

`ping 10.0.0.7 so g0/0`
```
R1#ping 10.0.0.7 so g0/0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.7, timeout is 2 seconds:
Packet sent with a source address of 10.0.0.0
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/3/4 ms
```

# Step 2: Configure LDP on all the interfaces in the MPLS Core
This is a requirement to configure **only** MPLS core, which means PE and P routers. CPE routers should not have this configuration. 

I've chosen to configure this slightly different from the guide and used area 0 specifically in the configuration. Not sure of the consequences yet, but if anything doesn't work properly I'll fix it and try to figure out why something failed to work.

R2:
```
conf t
router ospf 1
mpls ldp autoconfig area 0
end
```

R3:
```
conf t
router ospf 1
mpls ldp autoconfig area 0
end
```

R4:
```
conf t
router ospf 1
mpls ldp autoconfig area 0
end
```

Pretty simple. We've got LDP sessions up and running almost immediately. Let's do verification using `show mpls interfaces detail`
R2:
```
R2#show mpls interfaces detail
Interface GigabitEthernet0/0:
        Type Unknown
        IP labeling enabled (ldp):
          IGP config
        LSP Tunnel labeling not enabled
        IP FRR labeling not enabled
        BGP labeling not enabled
        MPLS operational
        MTU = 1500
Interface GigabitEthernet0/1:
        Type Unknown
        IP labeling enabled (ldp):
          IGP config
        LSP Tunnel labeling not enabled
        IP FRR labeling not enabled
        BGP labeling not enabled
        MPLS operational
        MTU = 1500
```

R3:
```
R3#show mpls interfaces detail
Interface GigabitEthernet0/0:
        Type Unknown
        IP labeling enabled (ldp):
          IGP config
        LSP Tunnel labeling not enabled
        IP FRR labeling not enabled
        BGP labeling not enabled
        MPLS operational
        MTU = 1500
Interface GigabitEthernet0/1:
        Type Unknown
        IP labeling enabled (ldp):
          IGP config
        LSP Tunnel labeling not enabled
        IP FRR labeling not enabled
        BGP labeling not enabled
        MPLS operational
        MTU = 1500
```

R4:
```
R4#show mpls interfaces  detail
Interface GigabitEthernet0/0:
        Type Unknown
        IP labeling enabled (ldp):
          IGP config
        LSP Tunnel labeling not enabled
        IP FRR labeling not enabled
        BGP labeling not enabled
        MPLS operational
        MTU = 1500
Interface GigabitEthernet0/1:
        Type Unknown
        IP labeling enabled (ldp):
          IGP config
        LSP Tunnel labeling not enabled
        IP FRR labeling not enabled
        BGP labeling not enabled
        MPLS operational
        MTU = 1500
```

There might be an issue with the default MTU as MPLS is L2.5 protocol (fits between LL and IP layers in OSI) and each label adds 4 bytes to the packet. Book `MPLS Fundamentals` by **Luc De Ghein** explains it in greater detail starting at page 59.

Let's verify neighbor states using the `show mpls ldp neighbor` command on each core router as well.
R2:
```
R2#show mpls ldp neighbor
    Peer LDP Ident: 3.3.3.3:0; Local LDP Ident 2.2.2.2:0
        TCP connection: 3.3.3.3.22797 - 2.2.2.2.646
        State: Oper; Msgs sent/rcvd: 22/22; Downstream
        Up time: 00:09:40
        LDP discovery sources:
          GigabitEthernet0/1, Src IP addr: 10.0.0.3
        Addresses bound to peer LDP Ident:
          10.0.0.3        10.0.0.4        3.3.3.3
```

R3:
```
R3#show mpls ldp neighbor
    Peer LDP Ident: 2.2.2.2:0; Local LDP Ident 3.3.3.3:0
        TCP connection: 2.2.2.2.646 - 3.3.3.3.22797
        State: Oper; Msgs sent/rcvd: 22/23; Downstream
        Up time: 00:09:57
        LDP discovery sources:
          GigabitEthernet0/0, Src IP addr: 10.0.0.2
        Addresses bound to peer LDP Ident:
          10.0.0.1        10.0.0.2        2.2.2.2
    Peer LDP Ident: 4.4.4.4:0; Local LDP Ident 3.3.3.3:0
        TCP connection: 4.4.4.4.13065 - 3.3.3.3.646
        State: Oper; Msgs sent/rcvd: 23/23; Downstream
        Up time: 00:09:49
        LDP discovery sources:
          GigabitEthernet0/1, Src IP addr: 10.0.0.5
        Addresses bound to peer LDP Ident:
          10.0.0.5        10.0.0.6        4.4.4.4
```

R4:
```
R4#show mpls ldp neighbor
    Peer LDP Ident: 3.3.3.3:0; Local LDP Ident 4.4.4.4:0
        TCP connection: 3.3.3.3.646 - 4.4.4.4.13065
        State: Oper; Msgs sent/rcvd: 23/23; Downstream
        Up time: 00:10:10
        LDP discovery sources:
          GigabitEthernet0/0, Src IP addr: 10.0.0.4
        Addresses bound to peer LDP Ident:
          10.0.0.3        10.0.0.4        3.3.3.3
```

The conclusion I can draw from this output is that LDP uses TCP port 646 to establish LDP connection. I believe it uses UDP port 646 to send LDP Hello messages, so just to be sure, below I've got a PCAP of it. Rather interesting that LDP uses both multicast and unicast for Hellos, but unicast runs on TCP:
![image](https://github.com/tomasp-xyz/Networking/assets/108157159/37bf312e-7c2c-40d5-93ad-55b71623177d)

While UDP is used to send Hellos to multicast address:
![image](https://github.com/tomasp-xyz/Networking/assets/108157159/03d456a1-6715-4eb7-93d7-6020c1f97474)

We can also check the traceroute of R2 trying to reach R4 lo0. MPLS label 18 is used to reach R4
```
R2#trace 4.4.4.4
Type escape sequence to abort.
Tracing the route to 4.4.4.4
VRF info: (vrf in name/id, vrf out name/id)
  1 10.0.0.3 [MPLS: Label 18 Exp 0] 3 msec 2 msec 2 msec
  2 10.0.0.5 3 msec 2 msec *
```

```
R2#show mpls forwarding-table
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id     Switched      interface
16         No Label   1.1.1.1/32       0             Gi0/0      10.0.0.0
17         Pop Label  3.3.3.3/32       0             Gi0/1      10.0.0.3
18         18         4.4.4.4/32       0             Gi0/1      10.0.0.3
19         19         5.5.5.5/32       0             Gi0/1      10.0.0.3
20         Pop Label  10.0.0.4/31      0             Gi0/1      10.0.0.3
21         21         10.0.0.6/31      0             Gi0/1      10.0.0.3
```

We can see that any packet moving from R2 towards R4's 4.4.4.4 gets a label of 18 (labels 0 to 15 are reserved so they're not used). Meanwhile, R3 doesn't apply the label, but instead pops it, effectively removing MPLS label and allowing the IP packet to arrive to R4 as intended:
```
R3#show mpls forwarding-table
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id     Switched      interface
16         16         1.1.1.1/32       0             Gi0/0      10.0.0.2
17         Pop Label  2.2.2.2/32       0             Gi0/0      10.0.0.2
18         Pop Label  4.4.4.4/32       252           Gi0/1      10.0.0.5
19         19         5.5.5.5/32       0             Gi0/1      10.0.0.5
20         Pop Label  10.0.0.0/31      0             Gi0/0      10.0.0.2
21         Pop Label  10.0.0.6/31      0             Gi0/1      10.0.0.5
```

```
R4#show mpls forwarding-table
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id     Switched      interface
16         16         1.1.1.1/32       0             Gi0/0      10.0.0.4
17         17         2.2.2.2/32       0             Gi0/0      10.0.0.4
18         Pop Label  3.3.3.3/32       0             Gi0/0      10.0.0.4
19         No Label   5.5.5.5/32       0             Gi0/1      10.0.0.7
20         20         10.0.0.0/31      0             Gi0/0      10.0.0.4
21         Pop Label  10.0.0.2/31      0             Gi0/0      10.0.0.4
```

I was also curious how things look like from R1 and R5 perspective
```
R1#trace 5.5.5.5
Type escape sequence to abort.
Tracing the route to 5.5.5.5
VRF info: (vrf in name/id, vrf out name/id)
  1 10.0.0.1 3 msec 2 msec 2 msec
  2 10.0.0.3 [MPLS: Label 19 Exp 0] 4 msec 4 msec 4 msec
  3 10.0.0.5 [MPLS: Label 19 Exp 0] 3 msec 3 msec 3 msec
  4 10.0.0.7 5 msec 3 msec *
```

According to `show mpls forwarding-table` output, R2 is considered as an ingress MPLS PE router and it marks any packets coming from R1 and destined for R5 with label 19. This label is then forwarded onwards to R3, which doesn't seem to do anything with the label and simply switches it towards R4. Now, R4 sees that a packet with this label should be going out via interface g0/1 and that there should be no label applied for the packet.

In fact, while observing the pcap, we can see that R4's interface g0/0 (essentially facing R1) receives packet with MPLS label 19:
![image](https://github.com/tomasp-xyz/Networking/assets/108157159/f9fd05d8-5caf-4ff9-b100-0308c81b3928)

But it forwards this same packet without any label out of its interface g0/1
![image](https://github.com/tomasp-xyz/Networking/assets/108157159/397bc00c-cb5f-4534-afc5-89e617e5186f)

Therefore, this proves that the "No label" state in the MPLS forwarding table simply means that we're removing MPLS label when specified egress interface is used for packet forwarding.

I think a similar result would be visible if we were to use ISIS (or EIGRP?) as our underlay IGP, because it seems that all we needed to do was establish connectivity between all routers (and their loopback ifaces) and enable MPLS/LDP on P/PE routers.

# Step 3: MPLS BGP Configuration between R1 and R3
At this point, we're getting into L3 VPN territory according to the article, which indicates that there's going to be VRF configuration somewhere along the road. Let's try it.

R2 config
```
conf t
router bgp 65500
neighbor 4.4.4.4 remote-as 65500
neighbor 4.4.4.4 update-source lo0
no auto-summary
address-family vpnv4
neighbor 4.4.4.4 activate
end
```

R4 config
```
conf t
router bgp 65500
neighbor 2.2.2.2 remote-as 65500
neighbor 2.2.2.2 update-source lo0
no auto-summary
address-family vpnv4
neighbor 2.2.2.2 activate
end
```

Seems like we're doing iBGP basic configuration with AFI set to vpnv4. Due to the fact that we already have underlay configured, we pretty much immediately see neighbor UP. Confirmed using `show ip bgp summary`
R2
```
R2#show ip bgp summary
BGP router identifier 2.2.2.2, local AS number 65500
BGP table version is 1, main routing table version 1

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
4.4.4.4         4        65500       2       2        1    0    0 00:00:30        0
```

R4
```
R4#show bgp vpnv4 unicast all summary
BGP router identifier 4.4.4.4, local AS number 65500
BGP table version is 1, main routing table version 1

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
2.2.2.2         4        65500       5       5        1    0    0 00:01:04        0
```

_Note - I've used two different show commands, but they output the same information. I think the more specific way is better, but for this lab there is no factual difference, at least currently._

# Step 4: Add two more routers, create VRFs
Okay, so apparently we don't need to have CPEs (R1 and R5) in area 0 and instead we should set them to area 2 (or really any other area, as long as it connects to area 0 in some manner). Changing configuration here:

R1
```
conf t
no router ospf 1
int lo0
ip ospf 1 area 2
int g0/0
ip ospf 1 area 2
end
```

R5
```
conf t
no router ospf 1
int lo0
ip ospf 1 area 2
int g0/0
ip ospf 1 area 2
end
```

Also need to disable OSPF on R2 and R4 routers facing CPEs:

R2
```
conf t
int g0/0
no ip ospf 1 area 0
end
```

R4
```
conf t
int g0/1
no ip ospf 1 area 0
end
```

OK, done. Mistake fixed, moving on.

# Step 5: VRF’s
I am skipping the unnecessary configuration on the guide, because it configures addressing without VRFs first on PE routers, but then re-configures VRFs and addressing again. Not sure why, but when we apply VRF on Cisco IOS router's interface, the IP addressing must be reconfigured so this intermediate step is skipped here.

Configuring VRF and RD/RT on R1 first:

R1
```
conf t
ip vrf RED
rd 5:5
route-target both 5:5
int g0/0
ip vrf forwarding RED
ip add 10.0.0.1 255.255.255.254
end
```

Connectivity check:
```
R2#ping vrf RED 10.0.0.0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.0.0.0, timeout is 2 seconds:
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 1/1/2 ms
R2#show arp
R2#show arp vr
R2#show arp vrf RED
Protocol  Address          Age (min)  Hardware Addr   Type   Interface
Internet  10.0.0.0                0   5000.0001.0000  ARPA   GigabitEthernet0/0
Internet  10.0.0.1                -   5000.0002.0000  ARPA   GigabitEthernet0/0
```

Since we've got basic connectivity between CPE (R1) and PE (R2, VRF `RED` on interface g0/0), we can create a new OSPF process and use area 2 for it to establish adjacency:

R2
```
conf t
int g0/0
ip ospf 2 area 2
end
```

At this point, R2 has the loopback IP address of R1 in its VRF `RED` routing table
```
R2#show ip route vrf RED

Routing Table: RED
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR

Gateway of last resort is not set

      1.0.0.0/32 is subnetted, 1 subnets
O        1.1.1.1 [110/2] via 10.0.0.0, 00:01:54, GigabitEthernet0/0
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.0.0.0/31 is directly connected, GigabitEthernet0/0
L        10.0.0.1/32 is directly connected, GigabitEthernet0/0
```

We now mirror the process for R4 PE & R5 CPE routers. R3 (P) router is left unchanged.

R4
```
conf t
ip vrf RED
rd 5:5
route-target both 5:5
int g0/1
ip vrf forwarding RED
ip address 10.0.0.6 255.255.255.254
ip ospf 2 area 2
end
```

This part is done. Now we have route to lo0 of R5 again from R4's perspective, as long as we use VRF `RED` to check the routing table:
```
R4#show ip route vrf RED

Routing Table: RED
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR

Gateway of last resort is not set

      5.0.0.0/32 is subnetted, 1 subnets
O        5.5.5.5 [110/2] via 10.0.0.7, 00:00:25, GigabitEthernet0/1
      10.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        10.0.0.6/31 is directly connected, GigabitEthernet0/1
L        10.0.0.6/32 is directly connected, GigabitEthernet0/1
```

# Step 6: Route redistribution across MPLS core
According to the guide, we need to:
* Redistribute OSPF into MP-BGP on R2
* Redistribute MP-BGP into OSPF on R2
* Redistribute OSPF into MP-BGP on R4
* Redistribute MP-BGP into OSPF on R4

_Note - I've changed router names here, because I'm using slightly different naming convention, but this will work the exact same either way._

Let's go into R2 and start working on it. Redistribute OSPF into MP-BGP on R2:

R2
```
conf t
router bgp 65500
address-family ipv4 vrf RED
redistribute ospf 2
end
```

We now verify that route redistribution is working as intended with command `show ip bgp vpnv4 vrf RED`
```
R2#show ip bgp vpnv4 vrf RED
BGP table version is 3, local router ID is 2.2.2.2
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 1:1 (default for vrf RED)
 *>  1.1.1.1/32       10.0.0.0                 2         32768 ?
 *>  10.0.0.0/31      0.0.0.0                  0         32768 ?
```

And yep, it is working indeed - we've got route to R1's lo0 interface in our BGP table, which is located under VRF `RED`.

Let's go and do the same thing on R4 now:

R4
```
conf t
router bgp 65500
address-family ipv4 vrf RED
redistribute ospf 2
end
```

And same thing works on R4 as well, we can see that we've successfully redistributed (because of `?` marker next to `Path`) routes from OSPF to MP-BGP:
```
R4#show ip bgp vpnv4 vrf RED
BGP table version is 3, local router ID is 4.4.4.4
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 5:5 (default for vrf RED)
 *>  5.5.5.5/32       10.0.0.7                 2         32768 ?
 *>  10.0.0.6/31      0.0.0.0                  0         32768 ?
```

There seems to be one last step here - we need to go into OSPF process this time on both PE routers and redistribute BGP **subnets** into the OSPF process, which connects directly to CPE. We do this by using the following configuration commands:

R2
```
conf t
router ospf 2
redistribute bgp 65500 subnets
end
```

R4
```
conf t
router ospf 2
redistribute bgp 65500 subnets
end
```

And this seems to work! R5 has IA OSPF routes:
```
R5#show ip route ospf
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR

Gateway of last resort is not set

      1.0.0.0/32 is subnetted, 1 subnets
O IA     1.1.1.1 [110/3] via 10.0.0.6, 00:01:48, GigabitEthernet0/0
      10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
O IA     10.0.0.0/31 [110/2] via 10.0.0.6, 00:01:48, GigabitEthernet0/0
```

Same for R1:
```
R1#show ip route ospf
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR

Gateway of last resort is not set

      5.0.0.0/32 is subnetted, 1 subnets
O IA     5.5.5.5 [110/3] via 10.0.0.1, 00:00:44, GigabitEthernet0/0
      10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
O IA     10.0.0.6/31 [110/2] via 10.0.0.1, 00:00:44, GigabitEthernet0/0
```

And we can reach both CPE sides as intended:
```
R5#ping 1.1.1.1 so lo0
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 1.1.1.1, timeout is 2 seconds:
Packet sent with a source address of 5.5.5.5
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/3/5 ms
```

It seems that we now have some additional labels here too:
```
R1#trace 5.5.5.5 so lo0
Type escape sequence to abort.
Tracing the route to 5.5.5.5
VRF info: (vrf in name/id, vrf out name/id)
  1 10.0.0.1 1 msec 3 msec 2 msec
  2 10.0.0.3 [MPLS: Labels 18/20 Exp 0] 4 msec 4 msec 3 msec
  3 10.0.0.6 [MPLS: Label 20 Exp 0] 3 msec 4 msec 3 msec
  4 10.0.0.7 4 msec 4 msec *
```

```
R5#trace 1.1.1.1 so lo0
Type escape sequence to abort.
Tracing the route to 1.1.1.1
VRF info: (vrf in name/id, vrf out name/id)
  1 10.0.0.6 2 msec 1 msec 2 msec
  2 10.0.0.4 [MPLS: Labels 17/21 Exp 0] 3 msec 4 msec 3 msec
  3 10.0.0.1 [MPLS: Label 21 Exp 0] 4 msec 3 msec 4 msec
  4 10.0.0.0 4 msec 4 msec *
```

Pcaps prove it (this is from the perspective of R4's g0/0):
![image](https://github.com/tomasp-xyz/Networking/assets/108157159/5cb51326-d0a5-4281-b1ab-b5fe34f86433)

Below is a sample pcap from R2's g0/1:
![image](https://github.com/tomasp-xyz/Networking/assets/108157159/9c7ebb9b-52e0-4394-b317-127eea9aa23e)

What seems to be happening here is PE routers are applying two labels now - one is bottom of the stack label 21 and the other is for MPLS switching over core 18 when looking from R4 PE perspective. Label 18 is removed when it goes through P router R3 towards PE router R2. At this point, packet has only one MPLS label remaining - 21. On R2, this label is considered as `No Label` and therefore label is removed when it's switched towards CPE R1.

For completion, I checked MPLS forwarding tables on all P/PE routers to verify this.

R2
```
R2#show mpls forwarding-table
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id     Switched      interface
16         No Label   10.0.0.0/31[V]   1244          aggregate/RED
17         Pop Label  3.3.3.3/32       0             Gi0/1      10.0.0.3
18         18         4.4.4.4/32       0             Gi0/1      10.0.0.3
20         Pop Label  10.0.0.4/31      0             Gi0/1      10.0.0.3
21         No Label   1.1.1.1/32[V]    413344        Gi0/0      10.0.0.0
```

R3
```
R3#show mpls forwarding-table
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id     Switched      interface
17         Pop Label  2.2.2.2/32       441376        Gi0/0      10.0.0.2
18         Pop Label  4.4.4.4/32       424794        Gi0/1      10.0.0.5
```

R4
```
R4#show mpls forwarding-table
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop
Label      Label      or Tunnel Id     Switched      interface
17         17         2.2.2.2/32       0             Gi0/0      10.0.0.4
18         Pop Label  3.3.3.3/32       0             Gi0/0      10.0.0.4
20         No Label   5.5.5.5/32[V]    5918          Gi0/1      10.0.0.7
21         Pop Label  10.0.0.2/31      0             Gi0/0      10.0.0.4
23         No Label   10.0.0.6/31[V]   402876        aggregate/RED
```

There is a lot more here that needs to be understood. For completion sake, [V] indicates a [V]PN label, which was locally assigned by the PE.

Final configurations of the routers:
<details>
  <summary>Click me</summary>
  
R1
```
R1#show run
Building configuration...

Current configuration : 3042 bytes
!
! Last configuration change at 20:11:50 UTC Mon Oct 2 2023
!
version 15.6
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname R1
!
boot-start-marker
boot-end-marker
!
!
!
no aaa new-model
ethernet lmi ce
!
!
!
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
!
!
!
!
!
!
!
!
!
!
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
!
!
!
!
redundancy
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
interface Loopback0
 ip address 1.1.1.1 255.255.255.255
 ip ospf 1 area 2
!
interface GigabitEthernet0/0
 ip address 10.0.0.0 255.255.255.254
 ip ospf 1 area 2
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/1
 no ip address
 shutdown
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/2
 no ip address
 shutdown
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/3
 no ip address
 shutdown
 duplex auto
 speed auto
 media-type rj45
!
router ospf 1
!
ip forward-protocol nd
!
!
no ip http server
no ip http secure-server
!
!
!
!
control-plane
!
banner exec ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner incoming ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner login ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
!
line con 0
line aux 0
line vty 0 4
 login
 transport input none
!
no scheduler allocate
!
end
```

R2
```
R2#show run
Building configuration...

Current configuration : 3563 bytes
!
! Last configuration change at 21:01:50 UTC Mon Oct 2 2023
!
version 15.6
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname R2
!
boot-start-marker
boot-end-marker
!
!
!
no aaa new-model
ethernet lmi ce
!
!
!
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
!
!
!
!
!
!
!
ip vrf RED
 rd 5:5
 route-target export 5:5
 route-target import 5:5
!
!
!
!
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
!
!
!
!
redundancy
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
interface Loopback0
 ip address 2.2.2.2 255.255.255.255
 ip ospf 1 area 0
!
interface GigabitEthernet0/0
 ip vrf forwarding RED
 ip address 10.0.0.1 255.255.255.254
 ip ospf 2 area 2
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/1
 ip address 10.0.0.2 255.255.255.254
 ip ospf 1 area 0
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/2
 no ip address
 shutdown
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/3
 no ip address
 shutdown
 duplex auto
 speed auto
 media-type rj45
!
router ospf 2 vrf RED
 redistribute bgp 65500 subnets
!
router ospf 1
 mpls ldp autoconfig area 0
!
router bgp 65500
 bgp log-neighbor-changes
 neighbor 4.4.4.4 remote-as 65500
 neighbor 4.4.4.4 update-source Loopback0
 !
 address-family vpnv4
  neighbor 4.4.4.4 activate
  neighbor 4.4.4.4 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf RED
  redistribute ospf 2
 exit-address-family
!
ip forward-protocol nd
!
!
no ip http server
no ip http secure-server
!
!
!
!
control-plane
!
banner exec ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner incoming ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner login ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
!
line con 0
line aux 0
line vty 0 4
 login
 transport input none
!
no scheduler allocate
!
end
```

R3
```
R3#show run
Building configuration...

Current configuration : 3100 bytes
!
! Last configuration change at 19:26:19 UTC Mon Oct 2 2023
!
version 15.6
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname R3
!
boot-start-marker
boot-end-marker
!
!
!
no aaa new-model
ethernet lmi ce
!
!
!
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
!
!
!
!
!
!
!
!
!
!
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
!
!
!
!
redundancy
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
 ip ospf 1 area 0
!
interface GigabitEthernet0/0
 ip address 10.0.0.3 255.255.255.254
 ip ospf 1 area 0
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/1
 ip address 10.0.0.4 255.255.255.254
 ip ospf 1 area 0
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/2
 no ip address
 shutdown
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/3
 no ip address
 shutdown
 duplex auto
 speed auto
 media-type rj45
!
router ospf 1
 mpls ldp autoconfig area 0
!
ip forward-protocol nd
!
!
no ip http server
no ip http secure-server
!
!
!
!
control-plane
!
banner exec ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner incoming ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner login ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
!
line con 0
line aux 0
line vty 0 4
 login
 transport input none
!
no scheduler allocate
!
end
```

R4
```
R4#show run
Building configuration...

Current configuration : 3563 bytes
!
! Last configuration change at 20:45:47 UTC Mon Oct 2 2023
!
version 15.6
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname R4
!
boot-start-marker
boot-end-marker
!
!
!
no aaa new-model
ethernet lmi ce
!
!
!
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
!
!
!
!
!
!
!
ip vrf RED
 rd 5:5
 route-target export 5:5
 route-target import 5:5
!
!
!
!
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
!
!
!
!
redundancy
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
interface Loopback0
 ip address 4.4.4.4 255.255.255.255
 ip ospf 1 area 0
!
interface GigabitEthernet0/0
 ip address 10.0.0.5 255.255.255.254
 ip ospf 1 area 0
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/1
 ip vrf forwarding RED
 ip address 10.0.0.6 255.255.255.254
 ip ospf 2 area 2
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/2
 no ip address
 shutdown
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/3
 no ip address
 shutdown
 duplex auto
 speed auto
 media-type rj45
!
router ospf 2 vrf RED
 redistribute bgp 65500 subnets
!
router ospf 1
 mpls ldp autoconfig area 0
!
router bgp 65500
 bgp log-neighbor-changes
 neighbor 2.2.2.2 remote-as 65500
 neighbor 2.2.2.2 update-source Loopback0
 !
 address-family vpnv4
  neighbor 2.2.2.2 activate
  neighbor 2.2.2.2 send-community extended
 exit-address-family
 !
 address-family ipv4 vrf RED
  redistribute ospf 2
 exit-address-family
!
ip forward-protocol nd
!
!
no ip http server
no ip http secure-server
!
!
!
!
control-plane
!
banner exec ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner incoming ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner login ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
!
line con 0
line aux 0
line vty 0 4
 login
 transport input none
!
no scheduler allocate
!
end
```

R5
```
R5#show run
Building configuration...

Current configuration : 3042 bytes
!
! Last configuration change at 20:10:26 UTC Mon Oct 2 2023
!
version 15.6
service timestamps debug datetime msec
service timestamps log datetime msec
no service password-encryption
!
hostname R5
!
boot-start-marker
boot-end-marker
!
!
!
no aaa new-model
ethernet lmi ce
!
!
!
mmi polling-interval 60
no mmi auto-configure
no mmi pvc
mmi snmp-timeout 180
!
!
!
!
!
!
!
!
!
!
!
ip cef
no ipv6 cef
!
multilink bundle-name authenticated
!
!
!
!
!
redundancy
!
!
!
!
!
!
!
!
!
!
!
!
!
!
!
interface Loopback0
 ip address 5.5.5.5 255.255.255.255
 ip ospf 1 area 2
!
interface GigabitEthernet0/0
 ip address 10.0.0.7 255.255.255.254
 ip ospf 1 area 2
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/1
 no ip address
 shutdown
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/2
 no ip address
 shutdown
 duplex auto
 speed auto
 media-type rj45
!
interface GigabitEthernet0/3
 no ip address
 shutdown
 duplex auto
 speed auto
 media-type rj45
!
router ospf 1
!
ip forward-protocol nd
!
!
no ip http server
no ip http secure-server
!
!
!
!
control-plane
!
banner exec ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner incoming ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
banner login ^C
**************************************************************************
* IOSv is strictly limited to use for evaluation, demonstration and IOS  *
* education. IOSv is provided as-is and is not supported by Cisco's      *
* Technical Advisory Center. Any use or disclosure, in whole or in part, *
* of the IOSv Software or Documentation to any third party for any       *
* purposes is expressly prohibited except as otherwise authorized by     *
* Cisco in writing.                                                      *
**************************************************************************^C
!
line con 0
line aux 0
line vty 0 4
 login
 transport input none
!
no scheduler allocate
!
end
```

</details>

And that's a very basic look at how MPLS works from service provider perspective. Pretty interesting, many important technologies here to consider for future reference, but at this point we've got L3VPN services going for ourselves, which is already pretty nice. Next up, we need to figure out how to provide L2VPN services for the sake of services configuration completion :)
