---
layout: post
title: "EIGRP stub area and route manipulation"
categories: routing
---

_Content idea taken from 101 Labs book by Paul Browning & Farai Tafa_

As I begin working (in a more serious manner anyway) on ENARSI, a proper description of each hands-on exercise is necessary. I've already done some of these labs before, but never really described them or analyzed them in a proper manner, which means that quite a few details were left out in the process.

To start things off, I'll be using EVE-NG to simulate the network and the following topology:

![image](https://github.com/tomasp-xyz/Networking/assets/108157159/1774c3da-cfbe-44e9-ac1f-f425093df676)

Same addressing will be used for the routers that I'll be building. Let's get into configuration and begin by looking through the task list in order to see what needs to be done.

# Task 1: Configure hostnames and IP addresses on all routers

Very simple task. Let's go through the topology from R1 to R4 and get it up and running. One note that I want to leave here - interface names will not match as my virtual topology uses gig interfaces as opposed to serial interfaces.

Config below:

<details>
  <summary>Click me for task 1 config</summary>

R1
```
en
conf t
host R1
int g0/0
ip add 10.0.0.1 255.255.255.252
no shutd
int g0/1
ip add 150.1.1.1 255.255.255.0
no shutd
int g0/2
ip add 10.0.0.5 255.255.255.252
no shutd
end
``` 

R2
```
en
conf t
int g0/0
ip add 10.0.0.2 255.255.255.252
no shutd
int g0/3
ip add 150.2.2.2 255.255.255.0
no shutd
int g0/1
ip add 10.0.0.9 255.255.255.252
no shutd
end
```

R3
```
en
conf t
host R3
int g0/1
ip add 10.0.0.10 255.255.255.252
no shutd
int g0/3
ip add 150.3.3.3 255.255.255.0
no shutd
int g0/2
ip add 10.0.0.6 255.255.255.252
no shutd
end
```

R4
```
en
conf t
host R4
int g0/1
ip add 150.1.1.4 255.255.255.0
no shutd
end
```

</details>

Task done.

# Task 2: EIGRP AS1 and R4 stub router config

There are some specifics in this task so it is advised to read the full task description carefully. Let's break it down:
* Configure EIGRP for AS 1 as shown in the topology. 
* R4 should be configured as an EIGRP stub router. 
* R4 should NEVER advertise any routes. 
* In addition to this, ensure that router R4 will only ever receive a default route from R1 even if external routes are redistributed into EIGRP 1.

Let's go step by step. First off - basic AS configuration for R1 and R4 interfaces facing each other. I'll be using only specific interface IPs to activate the EIGRP process on specific interfaces.

R1

```
conf t
router eigrp 1
no auto-summary
netw 150.1.1.1 0.0.0.0
end
```

R4

```
conf t
router eigrp 1
no auto-summary
netw 150.1.1.4 0.0.0.0
end
```

Next, we need to configure R4 as EIGRP stub router. This configuration should suffice under EIGRP AS1 process:
`eigrp stub receive-only`

We verify it from the neighbor R1 perspective:

```
R1#show ip eigrp neighbors  detail
EIGRP-IPv4 Neighbors for AS(1)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
                                                   (sec)         (ms)       Cnt Num
0   150.1.1.4               Gi0/1                    11 00:00:03    3   100  0  5
   Version 20.0/2.0, Retrans: 0, Retries: 0
   Topology-ids from peer - 0
   Topologies advertised to peer:   base

   Receive-Only Peer Advertising (No) Routes
   Suppressing queries
Max Nbrs: 0, Current Nbrs: 0
```

It will also make it so that R4 only receives routes, but never advertises anything from itself. There is one last condition that we need to deal with in this task:
* In addition to this, ensure that router R4 will only ever receive a default route from R1 even if external routes are redistributed into EIGRP 1.

Let's start by making R4 receive a default route. For this to work, we need to configure R1 to advertise what's called a summary address from the interface which is facing R4:

R1

```
conf t
int g0/1
ip summary-address eigrp 1 0.0.0.0/0
```

Lastly, there's this condition: 
_even if external routes are redistributed into EIGRP 1._

What this means is that we need to filter routes by ensuring that ONLY default route is advertised towards R4, but not any other more specific route. I will do this later on as it _seems_ that we don't really need to do any extra configuration at the moment (but this is likely to change). 

_UPDATE - I've revisited this point before finishing the lab and it seems that we don't really have to do any extra configuration here, because even if I were to redistribute a more specific connected or static route towards R4, it simply doesn't appear and we can only see default route in the routing table. Not sure if this is needed at all for the lab, but in real world we might need to do additional route filtering on ASBR (this is the correct name for R1, right?)._

For verification, let's check the routing table on R4:

```
R4#show ip route
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override, p - overrides from PfR

Gateway of last resort is 150.1.1.1 to network 0.0.0.0

D*    0.0.0.0/0 [90/2816] via 150.1.1.1, 00:01:14, GigabitEthernet0/1
      150.1.0.0/16 is variably subnetted, 2 subnets, 2 masks
C        150.1.1.0/24 is directly connected, GigabitEthernet0/1
L        150.1.1.4/32 is directly connected, GigabitEthernet0/1
```

We've got a default route via EIGRP. Moving on for now.

# Task 3: Configure AS2

This is very straightforward as we don't have any extra conditions to configure. Let's apply basic EIGRP configuration to the routers in AS2.

For the most part the commands are the same - we activate EIGRP AS2 on specific interfaces (which should belong in this AS).

R1

```
conf t
router eigrp 2
no auto-summary
netw 10.0.0.1 0.0.0.0
netw 10.0.0.5 0.0.0.0
end
```

R2

```
conf t
router eigrp 2
no auto-summary
netw 10.0.0.2 0.0.0.0
netw 10.0.0.9 0.0.0.0
netw 150.2.2.2 0.0.0.0
end
```

R3

```
conf t
router eigrp 2
no auto-summary
netw 10.0.0.6 0.0.0.0
netw 10.0.0.10 0.0.0.0
netw 150.3.3.3 0.0.0.0
end
```

We should see adjacencies coming up here. Verify neighbor status with `show ip eigrp neighbors`

```
R1#show ip eigrp neighbors
EIGRP-IPv4 Neighbors for AS(1)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
                                                   (sec)         (ms)       Cnt Num
0   150.1.1.4               Gi0/1                    13 00:06:51  546  4914  0  10
EIGRP-IPv4 Neighbors for AS(2)
H   Address                 Interface              Hold Uptime   SRTT   RTO  Q  Seq
                                                   (sec)         (ms)       Cnt Num
1   10.0.0.6                Gi0/2                    13 00:01:59    5   100  0  6
0   10.0.0.2                Gi0/0                    12 00:02:17    2   100  0  10
```

Check from R1 if we can reach all of the 150.x.x.x networks:

```
R1#ping 150.2.2.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 150.2.2.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/3/7 ms
R1#ping 150.3.3.3
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 150.3.3.3, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/2/2 ms
```

Seems good.

# Task 4: R4 reachability to AS2

This task once again has multiple requests. Let's break them down:
* Configure EIGRP so that R4 can reach all other routers in the network and vice-versa.
* Ensure that only the 150.1.1.0/24 is allowed into the topology table for EIGRP 2. 
* Verify your configuration and also ping to and from R4 from the 150.2.2.0/24 and 150.3.3.0/24 subnets.

First part of this task makes it sound as if we should use some type of redistribution, because R2 and R3 currently are not aware of 150.1.1.0/24 subnet. What could be done is connected routes redistribution on R1 in AS2, but the second condition tells us that if we do it that way, we must ensure that other R1 connected routes are not advertised in AS2 as well. 

Here we can use a route map and prefix list combination to fulfill both conditions at the same time. Let's get a prefix list going on R1:

```
ip prefix-list G0-1 permit 150.1.1.0/24
```

Next up, let's make a route map to permit ONLY this route for redistribution.

```
route-map G0-1 permit 10
match ip address prefix-list G0-1
route-map G0-1 deny 20
```

To verify that this works, I'm also going to create an additional loopback interface on R1. This interface should not be reachable in AS2 if everything is working as intended. Config for the loopback:

```
int lo0
ip add 1.1.1.1 255.255.255.255
```

Let's try to redistribute connected routes from R1 to AS2 now. Config:

```
router eigrp 2
redistribute connected route-map G0-1
```

Seems like it worked - neither R2 nor R3 can reach 1.1.1.1 and these routers have only a single route towards R1 - 150.1.1.0/24:

```
R2#show ip ro e
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

      10.0.0.0/8 is variably subnetted, 5 subnets, 2 masks
D        10.0.0.4/30 [90/3072] via 10.0.0.10, 00:15:04, GigabitEthernet0/1
                     [90/3072] via 10.0.0.1, 00:15:04, GigabitEthernet0/0
      150.1.0.0/24 is subnetted, 1 subnets
D EX     150.1.1.0 [170/3072] via 10.0.0.1, 00:02:14, GigabitEthernet0/0
      150.3.0.0/24 is subnetted, 1 subnets
D        150.3.3.0 [90/3072] via 10.0.0.10, 00:14:59, GigabitEthernet0/1
```

```
R3#show ip ro e
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

      10.0.0.0/8 is variably subnetted, 5 subnets, 2 masks
D        10.0.0.0/30 [90/3072] via 10.0.0.9, 00:15:13, GigabitEthernet0/1
                     [90/3072] via 10.0.0.5, 00:15:13, GigabitEthernet0/2
      150.1.0.0/24 is subnetted, 1 subnets
D EX     150.1.1.0 [170/3072] via 10.0.0.5, 00:02:23, GigabitEthernet0/2
      150.2.0.0/24 is subnetted, 1 subnets
D        150.2.2.0 [90/3072] via 10.0.0.9, 00:15:13, GigabitEthernet0/1
```

Last part of this task asks us to reach 150.2.2.2 and 150.3.3.3 from R4 (and vice-versa). We can use ping on R4 to verify connectivity:

```
R4#ping 150.3.3.3
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 150.3.3.3, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/2/3 ms
R4#ping 150.2.2.2
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 150.2.2.2, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/2/3 ms
```

Everything is working as intended. We can also reach R4 from R2 and R3:
```
R2#ping 150.1.1.4
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 150.1.1.4, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/8 ms
```

```
R3#ping 150.1.1.4
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 150.1.1.4, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/4 ms
```

# Task 5: Force R1 to prefer link via R2

This task sounds somewhat convoluted. I'd strongly advise to break it down into smaller pieces as well:

* Assume that the WAN link between R1 and R3 is unreliable and should only be used when the WAN link between R1 and R2 is down. 
* However, an EIGRP neighbor relationship should still be maintained across this link. 
* Configure EIGRP so that neither routers R1 nor R3 use this link unless the WAN link between R1 and R2 is down. 
* You are only allowed to configure R3. 
* Do NOT issue any configuration commands on R1 to complete this task.

What this essentially means is that path via R2 should be preferred to reach R3's connected subnet and we may not use some tricks with interface tracking to shut down the R1 interface facing R3 to prevent neighborship from forming. 

~~Since we need to force g0/2 interface in this case to be the less preferred interface, we can up the delay value by just a little bit and it should be enough. Let's verify EIGRP routing table of R3 BEFORE changing anything though~~

I thought initially that we can modify one of the K values on R3 (set higher delay on interface g0/2), but this didn't work out fully, because setting delay seemed to be locally significant only and R1 would continue to prefer path to R3's local network via g0/2 interface.

Instead, what we can do to fulfill the condition is to configure an `offset-list` option under EIGRP process on R3 using both in and out arguments on g0/2 interface. Before we proceed however, I would like to verify R1 and R3 EIGRP RIBs to have something to compare to later on.

R1:

```
R1#show ip ro e | b Gate
Gateway of last resort is 0.0.0.0 to network 0.0.0.0

D*    0.0.0.0/0 is a summary, 01:15:15, Null0
      10.0.0.0/8 is variably subnetted, 5 subnets, 2 masks
D        10.0.0.8/30 [90/3072] via 10.0.0.6, 00:02:13, GigabitEthernet0/2
                     [90/3072] via 10.0.0.2, 00:02:13, GigabitEthernet0/0
      150.2.0.0/24 is subnetted, 1 subnets
D        150.2.2.0 [90/3072] via 10.0.0.2, 00:02:13, GigabitEthernet0/0
      150.3.0.0/24 is subnetted, 1 subnets
D        150.3.3.0 [90/3072] via 10.0.0.6, 00:02:13, GigabitEthernet0/2
```

R3:

```
R3#show ip ro e | b Gate
Gateway of last resort is not set

      10.0.0.0/8 is variably subnetted, 5 subnets, 2 masks
D        10.0.0.0/30 [90/3072] via 10.0.0.9, 00:01:45, GigabitEthernet0/1
                     [90/3072] via 10.0.0.5, 00:01:45, GigabitEthernet0/2
      150.1.0.0/24 is subnetted, 1 subnets
D EX     150.1.1.0 [170/3072] via 10.0.0.5, 00:01:45, GigabitEthernet0/2
      150.2.0.0/24 is subnetted, 1 subnets
D        150.2.2.0 [90/3072] via 10.0.0.9, 00:01:45, GigabitEthernet0/1
```

Now that we have a full view of EIGRP routing tables on both routers, let's apply the config modifications using offset-list:

R3

```
conf t
router eigrp 2
offset-list 0 out 5000000 GigabitEthernet0/2
offset-list 0 in 5000000 GigabitEthernet0/2
end
```

Wait until EIGRP process performs a resync and verify routes again. Now we should see completely different routes from both sides.

R1:

```
R1#show ip ro e | b Gate
Gateway of last resort is 0.0.0.0 to network 0.0.0.0

D*    0.0.0.0/0 is a summary, 01:16:24, Null0
      10.0.0.0/8 is variably subnetted, 5 subnets, 2 masks
D        10.0.0.8/30 [90/3072] via 10.0.0.2, 00:00:33, GigabitEthernet0/0
      150.2.0.0/24 is subnetted, 1 subnets
D        150.2.2.0 [90/3072] via 10.0.0.2, 00:00:33, GigabitEthernet0/0
      150.3.0.0/24 is subnetted, 1 subnets
D        150.3.3.0 [90/3328] via 10.0.0.2, 00:00:33, GigabitEthernet0/0
```

R3:

```
R3#show ip ro e | b Gate
Gateway of last resort is not set

      10.0.0.0/8 is variably subnetted, 5 subnets, 2 masks
D        10.0.0.0/30 [90/3072] via 10.0.0.9, 00:00:46, GigabitEthernet0/1
      150.1.0.0/24 is subnetted, 1 subnets
D EX     150.1.1.0 [170/3328] via 10.0.0.9, 00:00:46, GigabitEthernet0/1
      150.2.0.0/24 is subnetted, 1 subnets
D        150.2.2.0 [90/3072] via 10.0.0.9, 00:00:46, GigabitEthernet0/1
```

To sum it up, we are now forcing R1 to talk to R3 via R2 and vice versa (R3 path towards R1 is via R2). As a last test we can launch a ping from R1 to R3's 150.3.3.3 and shutdown R3-facing interface on R2. We should experience small dropout, but due to EIGRP feasible successor condition, the secondary route takes over almost immediately.

```
R1#ping 150.3.3.3 rep 5000
Type escape sequence to abort.
Sending 5000, 100-byte ICMP Echos to 150.3.3.3, timeout is 2 seconds:
<SHUTDOWN R2's g0/1 INTERFACE HERE DURING PING PROCESS>
Success rate is 99 percent (4999/5000), round-trip min/avg/max = 1/2/18 ms
```

It seems that we lost only one packet which indicates a relatively rapid path switchover. Lastly, we can verify routes one last time on R1 and R3, because now we do not have primary path online.

R1:

```
R1#show ip ro e | b Gate
Gateway of last resort is 0.0.0.0 to network 0.0.0.0

D*    0.0.0.0/0 is a summary, 01:21:19, Null0
      10.0.0.0/8 is variably subnetted, 5 subnets, 2 masks
D        10.0.0.8/30 [90/5003072] via 10.0.0.6, 00:02:12, GigabitEthernet0/2
      150.2.0.0/24 is subnetted, 1 subnets
D        150.2.2.0 [90/3072] via 10.0.0.2, 00:05:28, GigabitEthernet0/0
      150.3.0.0/24 is subnetted, 1 subnets
D        150.3.3.0 [90/5003072] via 10.0.0.6, 00:02:12, GigabitEthernet0/2
```

R3:

```
R3#show ip ro e | b Gate
Gateway of last resort is not set

      10.0.0.0/8 is variably subnetted, 5 subnets, 2 masks
D        10.0.0.0/30 [90/5003072] via 10.0.0.5, 00:01:41, GigabitEthernet0/2
      150.1.0.0/24 is subnetted, 1 subnets
D EX     150.1.1.0 [170/5003072] via 10.0.0.5, 00:01:41, GigabitEthernet0/2
      150.2.0.0/24 is subnetted, 1 subnets
D        150.2.2.0 [90/5003328] via 10.0.0.5, 00:01:41, GigabitEthernet0/2
```

And indeed, the offset-list worked just fine - we can clearly see EIGRP metric has been modified by a massive amount and everything is working as intended.

This concludes the lab exercise.
