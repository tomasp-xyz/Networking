---
layout: post
title: "Hierarchical DMVPN Phase 3 configuration"
categories: dmvpn, nhrp, routing
---

DMVPN is listed as one of CCNP ENARSI exam topics (section `2.3 Configure and verify DMVPN`). In order to understand some concepts, specifically how NHRP works and how automatic neighbor resolution occurs, as well as to solidify general knowledge when it comes to configuring IPsec on Cisco IOS-based routers, I've decided to try and tackle the topic by following Cisco article and by creating a lab out of it:

https://www.cisco.com/c/en/us/support/docs/security/dynamic-multipoint-vpn-dmvpn/211292-Configure-Phase-3-Hierarchical-DMVPN-wit.html

# Step 1 - topology setup
I'll be using the same topology as it was described in the article with the following differences:
- different addressing scheme
- OSPF instead of EIGRP
- fine tuning NHRP configuration

![image](https://github.com/tomasp-xyz/Networking/assets/108157159/f4d88c60-7788-4177-a829-ac06a236a746)


Addressing scheme

| Router name | Tunnel 1 IP | Tunnel 2 IP | g0/0 IP | g0/1 IP | g0/2 IP |
| ------------- | ------------- | ------------- | ------------- | ------------- | ------------- |
| Hub 0 (main) | 172.20.30.6/29 | N/A | 10.20.30.0/31 | 192.168.30.0/31 | N/A |
| Hub 1 | 172.20.30.1/29 | 172.25.1.1/24 | 10.0.0.1/31 | N/A | 10.0.0.2/31 |
| Hub 2 | 172.20.30.2/29 | 172.25.2.1/24 | N/A | 192.168.30.1/31 | 192.168.30.2/31 |
| Spoke 1 | 172.25.1.2/24 | N/A | N/A | N/A | 10.0.0.3/31 |
| Spoke 2 | 172.25.2.2/24 | N/A | N/A | N/A | 192.168.30.3/31 |

# Step 2 - basic configuration 
To start things off, I need to setup everything as described in the addressing scheme. There is no NHRP, OSPF, DHCP or IPsec configuration as of right now - I'm simply trying to bring devices online and find out if point-to-point connectivity is working as intended. Configuration below.

<details>
  <summary>Click me for basic config - Hub 0 (main)</summary>
  
```  
ena
conf t
hostname HUB0_MAIN
int g0/0
ip address 10.20.30.0 255.255.255.254
no shutd
description "TO_HUB1_G0/0"
!
int g0/1
ip address 192.168.30.0 255.255.255.254
no shutd
description "TO_HUB2_G0/1"
!
int lo0
ip address 1.1.1.1 255.255.255.255
description "LO0_FOR_ROUTING"
!
int tun1
ip address 172.20.30.6 255.255.255.248
description "MGRE_FOR_HUBS"
no shutd
bandwidth 1000000
ip mtu 1460
ip tcp adjust-mss 1420
tunnel mode gre multipoint
tunnel source lo0
tunnel key 1234
!
```

</details>

<details>
  <summary>Click me for basic config - Hub 1</summary>

  ```
ena
conf t
hostname HUB1
int g0/0
ip address 10.20.30.1 255.255.255.254
no shutd
description "TO_HUB0_G0/0"
!
int g0/2
ip address 10.0.0.2 255.255.255.254
no shutd
description "TO_SPOKE1_G0/2"
!
int lo0
ip address 11.11.11.11 255.255.255.255
description "LO0_FOR_ROUTING"
!
int tun1
ip address 172.20.30.1 255.255.255.248
description "MGRE_HUB1_TUN1"
no shutd
bandwidth 1000000
ip mtu 1460
ip tcp adjust-mss 1420
tunnel mode gre multipoint
tunnel source lo0
tunnel key 1234
!
int tun2
ip address 172.25.1.1 255.255.255.0
description "MGRE_FOR_SPOKES"
no shutd
bandwidth 1000000
ip tcp adjust-mss 1420
tunnel mode gre multipoint
tunnel source lo0
tunnel key 4321
!
```

</details>
  
<details>
  <summary>Click me for basic config - Hub 2</summary>

  ```
ena
conf t
hostname HUB2
int g0/1
ip address 192.168.30.1 255.255.255.254
no shutd
description "TO_HUB0_G0/1"
!
int g0/2
ip address 192.168.30.2 255.255.255.254
no shutd
description "TO_SPOKE2_G0/2"
!
int lo0
ip address 22.22.22.22 255.255.255.255
description "LO0_FOR_ROUTING"
!
int tun1
ip address 172.20.30.2 255.255.255.248
description "MGRE_HUB2_TUN1"
no shutd
bandwidth 1000000
ip mtu 1460
ip tcp adjust-mss 1420
tunnel mode gre multipoint
tunnel source lo0
tunnel key 1234
!
int tun2
ip address 172.25.2.1 255.255.255.0
description "MGRE_FOR_SPOKES_@HUB2"
no shutd
bandwidth 1000000
ip mtu 1460
ip tcp adjust-mss 1420
tunnel mode gre multipoint
tunnel source lo0
tunnel key 4321
!
```

</details>

<details>
  <summary>Click me for basic config - Spoke 1</summary>

```
enab
conf t
hostname SPOKE1
int g0/2
ip address 10.0.0.3 255.255.255.254
no shutd
description "TO_HUB1_G0/2"
!
int g0/3
ip address 192.168.0.1 255.255.255.0
no shutd
description "SPOKE1_LAN"
!
int tun1
ip address 172.25.1.2 255.255.255.0
description "MGRE_TO_HUB1"
no shutd
bandwidth 1000000
ip mtu 1460
ip tcp adjust-mss 1420
tunnel mode gre multipoint
tunnel source g0/2
tunnel key 4321
!
```

</details>

<details>
  <summary>Click me for basic config - Spoke 2</summary>

  ```
enab
conf t
hostname SPOKE2
int g0/2
ip address 192.168.30.3 255.255.255.254
no shutd
description "TO_HUB2_G0/2"
!
int g0/3
ip address 192.168.99.1 255.255.255.0
no shutd
description "SPOKE2_LAN"
!
int tun1
ip address 172.25.2.2 255.255.255.0
description "MGRE_TO_HUB2"
no shutd
bandwidth 1000000
ip mtu 1460
ip tcp adjust-mss 1420
tunnel mode gre multipoint
tunnel source g0/2
tunnel key 4321
!
```

</details>

Do note that at this point we won't have GRE connectivity at this point and only P2P links will come up once routers fill in their respective MAC to IP entries in the ARP table.

# Step 3 - OSPF & NHRP config
Dynamic routing is needed to exchange information about non-P2P interfaces, loopbacks and LANs between spokes/hubs. What I didn't know and actually learned specifically thanks to the hierarchical DMVPN config example is that NHRP is mandatory in order to bring mGRE tunnels online. If NHRP configuration does not exist then we can still get connectivity between mGRE tunnels with the help of either:
- static route pointing specifically to tunnel IP address via _<specific>_ interface
- or a more specific route than the directly connected interface (due to RIB logic for best route selection, aka `route > AD > metric`)

This is also why I combined both OSPF and NHRP configuration into a single section. Without either of these components connectivity simply will not be possible between mGRE interfaces of our hubs. Just to be clear - if we do not fulfill one of the conditions named above (and even then it's not recommended to do that because extra config makes our dynamic multipoint VPN not so dynamic due to additional config overhead on top of making things more complex) then the following error messages can be seen when running `debug ip packet` command:
```
*Apr  1 18:38:03.169: IP: s=172.20.30.6 (local), d=172.20.30.1 (Tunnel1), len 10                                                            0, sending
*Apr  1 18:38:03.170: IP: s=172.20.30.6 (local), d=172.20.30.1 (Tunnel1), len 10                                                            0, output feature, TCP Adjust MSS(58), rtype 1, forus FALSE, sendself FALSE, mtu                                                             0, fwdchk FALSE
*Apr  1 18:38:03.171: IP: s=172.20.30.6 (local), d=172.20.30.1 (Tunnel1), len 10                                                            0, encapsulation failed.
*Apr  1 18:38:05.169: IP: s=172.20.30.6 (local), d=172.20.30.1 (Tunnel1), len 10                                                            0, sending
*Apr  1 18:38:05.170: IP: s=172.20.30.6 (local), d=172.20.30.1 (Tunnel1), len 10                                                            0, output feature, TCP Adjust MSS(58), rtype 1, forus FALSE, sendself FALSE, mtu                                                             0, fwdchk FALSE
*Apr  1 18:38:05.171: IP: s=172.20.30.6 (local), d=172.20.30.1 (Tunnel1), len 10                                                            0, encapsulation failed.
Success rate is 0 percent (0/5)
```

To avoid this issue, I included the following configuration on my hub routers:

**HUB 0 - MAIN**
```
int tun1
 ip nhrp authentication dmvpn
 ip nhrp map multicast dynamic
 ip nhrp network-id 1
 ip nhrp holdtime 10
 ip nhrp shortcut
 ip nhrp redirect
 keepalive 10 3
!
router ospf 1
 redistribute connected subnets
 network 10.20.30.0 0.0.0.1 area 0
 network 192.168.30.0 0.0.0.1 area 0
```

**HUB 1**
```
int tun1
 ip nhrp authentication dmvpn
 ip nhrp network-id 1
 ip nhrp holdtime 10
 ip nhrp nhs 172.20.30.6 nbma 1.1.1.1 multicast
 ip nhrp shortcut
 ip nhrp redirect
 keepalive 10 3
!
router ospf 1
 redistribute connected subnets
 network 10.20.30.0 0.0.0.1 area 0
```

**HUB 2**
```
int tun1
 ip nhrp authentication dmvpn
 ip nhrp network-id 1
 ip nhrp holdtime 10
 ip nhrp nhs 172.20.30.6 nbma 1.1.1.1 multicast
 ip nhrp shortcut
 ip nhrp redirect
 keepalive 10 3
!
router ospf 1
 redistribute connected subnets
 network 192.168.30.0 0.0.0.1 area 0
```

Now I would like to break this configuration down, because this is probably the most important part of this entire lab. Starting with simpler things, OSPF configuration is more or less identical across every router, only difference being the initial network configuration statement. The logic is as follows:
1. HUB0 router establishes neighborship with HUB1/HUB2 routers with a single network statement
2. Once neighborship is up (FULL/DR state), all remaining subnets are redistributed between each router

OSPF configuration could be improved further by setting its mode to P2P, but I won't be delving into OSPF part at this point since it works well enough in this case. However, the NHRP part was crucial. Starting with main (central) hub configuration, the following lines were added under `Tunnel1` interface:
- `ip nhrp authentication dmvpn` - authentication string that will be used within single logical NBMA network, optional config
- `ip nhrp map multicast dynamic` - this command ensures that downstream hub routers' IP addresses are automatically added to multicast destination list whenever they register with the main hub; useful in case we want broadcast/multicast traffic to get forwarded towards specific NBMAs across the mGRE tunnel
- `ip nhrp network-id 1` - somewhat confusing, but it seems that this command has local significance and does not have to match the other end; useful only when we have a hub with multiple mGRE interfaces serving multiple DMVPN networks
- `ip nhrp holdtime 10` - timer for dynamic NHRP entries expiration; I opted to use relatively low timer for faster expiration in case I decide to shut down a physical link between hubs/spokes
- `ip nhrp shortcut` - this is the command that makes DMVPN Phase 3 work - it optimizes traffic flow and, if possible, ensures that all data traffic bypasses the central hub between spokes (or two other hubs); online sources suggest that this command should not exist on a hub and instead it should be used on spokes only, but I'm leaving it in on all hubs for now since this is not exactly typical DMVPN configuration due to hierarchy shenanigans (this commands tells spokes to accept traffic redirected by the hub)
- `ip nhrp redirect` - this command forces the central hub to redirect spoke-to-spoke communication by using a better path, if such path is available; we don't need to use this command on spoke routers
- `keepalive 10 3` - my own idea to generate some extra traffic on top of typical OSPF keepalives; this is a built-in Cisco IOS command for tunnel interfaces to send keepalive packets every 10 seconds and if 3 consecutive keepalives are lost then the GRE tunnel will be brought down - can be useful to force GRE tunnel to perform auto up/down action by using some additional EEM scripts

### Some additional details from routers
#### OSPF status & routes

```
HUB0_MAIN#show ip ospf  neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
22.22.22.22       1   FULL/DR         00:00:38    192.168.30.1    GigabitEthernet0/1
11.11.11.11       1   FULL/DR         00:00:31    10.20.30.1      GigabitEthernet0/0

HUB0_MAIN#show ip ro | b Gate
Gateway of last resort is not set

      1.0.0.0/32 is subnetted, 1 subnets
C        1.1.1.1 is directly connected, Loopback0
      10.0.0.0/8 is variably subnetted, 3 subnets, 2 masks
O E2     10.0.0.2/31 [110/20] via 10.20.30.1, 02:05:07, GigabitEthernet0/0
C        10.20.30.0/31 is directly connected, GigabitEthernet0/0
L        10.20.30.0/32 is directly connected, GigabitEthernet0/0
      11.0.0.0/32 is subnetted, 1 subnets
O E2     11.11.11.11 [110/20] via 10.20.30.1, 02:05:07, GigabitEthernet0/0
      22.0.0.0/32 is subnetted, 1 subnets
O E2     22.22.22.22 [110/20] via 192.168.30.1, 01:09:37, GigabitEthernet0/1
      172.20.0.0/16 is variably subnetted, 2 subnets, 2 masks
C        172.20.30.0/29 is directly connected, Tunnel1
L        172.20.30.6/32 is directly connected, Tunnel1
      172.25.0.0/24 is subnetted, 2 subnets
O E2     172.25.1.0 [110/20] via 10.20.30.1, 02:05:07, GigabitEthernet0/0
O E2     172.25.2.0 [110/20] via 192.168.30.1, 01:09:37, GigabitEthernet0/1
      192.168.30.0/24 is variably subnetted, 3 subnets, 2 masks
C        192.168.30.0/31 is directly connected, GigabitEthernet0/1
L        192.168.30.0/32 is directly connected, GigabitEthernet0/1
O E2     192.168.30.2/31
           [110/20] via 192.168.30.1, 01:09:37, GigabitEthernet0/1

```

```
HUB1#show ip ospf ne

Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           1   FULL/BDR        00:00:38    10.20.30.0      GigabitEthernet0/0

HUB1#show ip route | b Gate
Gateway of last resort is not set

      1.0.0.0/32 is subnetted, 1 subnets
O E2     1.1.1.1 [110/20] via 10.20.30.0, 02:05:26, GigabitEthernet0/0
      10.0.0.0/8 is variably subnetted, 4 subnets, 2 masks
C        10.0.0.2/31 is directly connected, GigabitEthernet0/2
L        10.0.0.2/32 is directly connected, GigabitEthernet0/2
C        10.20.30.0/31 is directly connected, GigabitEthernet0/0
L        10.20.30.1/32 is directly connected, GigabitEthernet0/0
      11.0.0.0/32 is subnetted, 1 subnets
C        11.11.11.11 is directly connected, Loopback0
      22.0.0.0/32 is subnetted, 1 subnets
O E2     22.22.22.22 [110/20] via 10.20.30.0, 01:09:46, GigabitEthernet0/0
      172.20.0.0/16 is variably subnetted, 2 subnets, 2 masks
C        172.20.30.0/29 is directly connected, Tunnel1
L        172.20.30.1/32 is directly connected, Tunnel1
      172.25.0.0/16 is variably subnetted, 3 subnets, 2 masks
C        172.25.1.0/24 is directly connected, Tunnel2
L        172.25.1.1/32 is directly connected, Tunnel2
O E2     172.25.2.0/24 [110/20] via 10.20.30.0, 01:09:46, GigabitEthernet0/0
      192.168.30.0/31 is subnetted, 2 subnets
O        192.168.30.0 [110/2] via 10.20.30.0, 01:09:56, GigabitEthernet0/0
O E2     192.168.30.2 [110/20] via 10.20.30.0, 01:09:46, GigabitEthernet0/0

```

```
HUB2#show ip ospf neighbor

Neighbor ID     Pri   State           Dead Time   Address         Interface
1.1.1.1           1   FULL/BDR        00:00:31    192.168.30.0    GigabitEthernet0/1

HUB2#show ip route | b Gate
Gateway of last resort is not set

      1.0.0.0/32 is subnetted, 1 subnets
O E2     1.1.1.1 [110/20] via 192.168.30.0, 01:10:10, GigabitEthernet0/1
      10.0.0.0/31 is subnetted, 2 subnets
O E2     10.0.0.2 [110/20] via 192.168.30.0, 01:10:10, GigabitEthernet0/1
O        10.20.30.0 [110/2] via 192.168.30.0, 01:10:10, GigabitEthernet0/1
      11.0.0.0/32 is subnetted, 1 subnets
O E2     11.11.11.11 [110/20] via 192.168.30.0, 01:10:10, GigabitEthernet0/1
      22.0.0.0/32 is subnetted, 1 subnets
C        22.22.22.22 is directly connected, Loopback0
      172.20.0.0/16 is variably subnetted, 2 subnets, 2 masks
C        172.20.30.0/29 is directly connected, Tunnel1
L        172.20.30.2/32 is directly connected, Tunnel1
      172.25.0.0/16 is variably subnetted, 3 subnets, 2 masks
O E2     172.25.1.0/24 [110/20] via 192.168.30.0, 01:10:10, GigabitEthernet0/1
C        172.25.2.0/24 is directly connected, Tunnel2
L        172.25.2.1/32 is directly connected, Tunnel2
      192.168.30.0/24 is variably subnetted, 4 subnets, 2 masks
C        192.168.30.0/31 is directly connected, GigabitEthernet0/1
L        192.168.30.1/32 is directly connected, GigabitEthernet0/1
C        192.168.30.2/31 is directly connected, GigabitEthernet0/2
L        192.168.30.2/32 is directly connected, GigabitEthernet0/2

```

#### NHRP
```
HUB0_MAIN#show ip nhrp
172.20.30.1/32 via 172.20.30.1
   Tunnel1 created 01:15:50, expire 00:00:07
   Type: dynamic, Flags: unique registered nhop
   NBMA address: 11.11.11.11
172.20.30.2/32 via 172.20.30.2
   Tunnel1 created 00:37:48, expire 00:00:07
   Type: dynamic, Flags: unique registered nhop
   NBMA address: 22.22.22.22
HUB0_MAIN#show dmvpn
Legend: Attrb --> S - Static, D - Dynamic, I - Incomplete
        N - NATed, L - Local, X - No Socket
        T1 - Route Installed, T2 - Nexthop-override
        C - CTS Capable
        # Ent --> Number of NHRP entries with same NBMA peer
        NHS Status: E --> Expecting Replies, R --> Responding, W --> Waiting
        UpDn Time --> Up or Down Time for a Tunnel
==========================================================================

Interface: Tunnel1, IPv4 NHRP Details
Type:Hub, NHRP Peers:2,

 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1 11.11.11.11         172.20.30.1    UP 01:15:52     D
     1 22.22.22.22         172.20.30.2    UP 00:35:48     D

```

```
HUB1#show ip nhrp
172.20.30.6/32 via 172.20.30.6
   Tunnel1 created 01:15:46, never expire
   Type: static, Flags: used
   NBMA address: 1.1.1.1
HUB1#show dmvpn
Legend: Attrb --> S - Static, D - Dynamic, I - Incomplete
        N - NATed, L - Local, X - No Socket
        T1 - Route Installed, T2 - Nexthop-override
        C - CTS Capable
        # Ent --> Number of NHRP entries with same NBMA peer
        NHS Status: E --> Expecting Replies, R --> Responding, W --> Waiting
        UpDn Time --> Up or Down Time for a Tunnel
==========================================================================

Interface: Tunnel1, IPv4 NHRP Details
Type:Spoke, NHRP Peers:1,

 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1 1.1.1.1             172.20.30.6    UP 01:16:08     S
```

```
HUB2#show ip nhrp
172.20.30.6/32 via 172.20.30.6
   Tunnel1 created 00:36:21, never expire
   Type: static, Flags: used
   NBMA address: 1.1.1.1
HUB2#show dmvpn
Legend: Attrb --> S - Static, D - Dynamic, I - Incomplete
        N - NATed, L - Local, X - No Socket
        T1 - Route Installed, T2 - Nexthop-override
        C - CTS Capable
        # Ent --> Number of NHRP entries with same NBMA peer
        NHS Status: E --> Expecting Replies, R --> Responding, W --> Waiting
        UpDn Time --> Up or Down Time for a Tunnel
==========================================================================

Interface: Tunnel1, IPv4 NHRP Details
Type:Spoke, NHRP Peers:1,

 # Ent  Peer NBMA Addr Peer Tunnel Add State  UpDn Tm Attrb
 ----- --------------- --------------- ----- -------- -----
     1 1.1.1.1             172.20.30.6    UP 00:36:21     S

```

To conclude everything up until this point, by establishing OSPF neighborships via P2P interfaces and redistributing all locally connected routes between main hub & downstream hubs, each router is able to reach each other's loopback IP addresses which results in NHRP tables getting populated with each router respective loopback IP address (NBMAs). Due to this, we are now able to tunnel data via GRE interfaces.

# Step 4 - IPsec configuration on hubs

Quickly configuring IPsec here to protect traffic that's being transported via mGRE tunnels. I skipped IPsec config initially to simplify hub configuration, but at this point not much else besides IPsec has to be configured on all hubs. Quite a lot of IPsec configuration overlaps. For **hub 1** and **hub 2**, the following configuration should do the trick:

```

crypto isakmp policy 1
 encr aes 256
 hash sha512
 authentication pre-share
 group 16
 lifetime 3600

crypto isakmp key ike_auth address 0.0.0.0
crypto ipsec transform-set sec_DMVPN ah-sha512-hmac esp-aes 256
 mode transport
crypto ipsec transform-set sec_HUB-p2p ah-sha512-hmac esp-aes 256
 mode transport
crypto ipsec profile hub_profile
 set transform-set sec_DMVPN
crypto ipsec profile sec_HUB-p2p
 set transform-set sec_HUB-p2p
int 1
 tunnel protection ipsec profile hub_profile
int 2
 tunnel protection ipsec profile sec_HUB-p2p

```

This should establish IPsec connection to the central hub and also protect any traffic coming in from spokes. The following IPsec configuration is for central hub only:

```

crypto isakmp policy 1
 encr aes 256
 hash sha512
 authentication pre-share
 group 16
 lifetime 3600
crypto isakmp key ike_auth address 0.0.0.0
crypto ipsec transform-set sec_DMVPN ah-sha512-hmac esp-aes 256
 mode transport
crypto ipsec profile hub_profile
 set transform-set sec_DMVPN
 responder-only
int tun1
 tunnel protection ipsec profile hub_profile

```

Verifying ISAKMP and IPsec status between central and regional hubs:

```
HUB0_MAIN#show crypto session
Crypto session current status

Interface: Tunnel1
Session status: UP-ACTIVE
Peer: 11.11.11.11 port 500
  Session ID: 0
  IKEv1 SA: local 1.1.1.1/500 remote 11.11.11.11/500 Active
  IPSEC FLOW: permit 47 host 1.1.1.1 host 11.11.11.11
        Active SAs: 4, origin: crypto map

Interface: Tunnel1
Session status: UP-ACTIVE
Peer: 22.22.22.22 port 500
  Session ID: 0
  IKEv1 SA: local 1.1.1.1/500 remote 22.22.22.22/500 Active
  IPSEC FLOW: permit 47 host 1.1.1.1 host 22.22.22.22
        Active SAs: 4, origin: crypto map

HUB0_MAIN#show crypto ipsec sa

interface: Tunnel1
    Crypto map tag: Tunnel1-head-0, local addr 1.1.1.1

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (1.1.1.1/255.255.255.255/47/0)
   remote ident (addr/mask/prot/port): (11.11.11.11/255.255.255.255/47/0)
   current_peer 11.11.11.11 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 320, #pkts encrypt: 320, #pkts digest: 320
    #pkts decaps: 320, #pkts decrypt: 320, #pkts verify: 320
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #send errors 0, #recv errors 0

------- some output was removed to save space -------

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (1.1.1.1/255.255.255.255/47/0)
   remote ident (addr/mask/prot/port): (22.22.22.22/255.255.255.255/47/0)
   current_peer 22.22.22.22 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 673, #pkts encrypt: 673, #pkts digest: 673
    #pkts decaps: 674, #pkts decrypt: 674, #pkts verify: 674
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #send errors 0, #recv errors 0


```

Okay, so we're seeing that protection between regional hubs and central hub is working, packets are being encrypted/decrypted. Now, let's proceed with spokes configuration, this time configuring everything at once.

# Step 5 - Spokes setup

--------- to be continued later on ----------
