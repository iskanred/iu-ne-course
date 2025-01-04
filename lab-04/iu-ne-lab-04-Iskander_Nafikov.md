* **Name**: Iskander Nafikov
* **E-mail**: i.nafikov@innopolis.university
* **Username**: [iskanred](https://github.com/iskanred)
* **Hostname**: `lenovo`
---
# Task 1 - Prepare your network topology

> **1.** `In the GNS3 project, select and install a virtual routing solution that you would like to use: Mikrotik (recommended), Pfsense, vyos and so on.`

I decided to use Mikrotik as recommended - `Mikrotik CHR 7.16`

> **2.** `Prepare a simple network consisting of at least 3 routers, each one of them has a different subnet, and they should be able to reach each other (for example by a switch/router in the middle or a bus topology). Do not write a static routes between different networks.`

* I prepared my topology as it was suggested:
	![[Pasted image 20241205162351.png]]
* However, to prepare it more accurately I was recommended to draw a network scheme before starting the lab. So, I decided to draw the scheme
	![[Pasted image 20241205162845.png]]
* Finally, I configured the whole topology as following:
	![[Pasted image 20241205172203.png]]
* So now routers can reach each other and the PC node inside their own LAN, but cannot reach others' routers LANs.
	![[Pasted image 20241205172253.png]]
* The similar was applied to PC nodes: we can reach the router, but no other routers or PCs
	![[Pasted image 20241206013330.png]]

So far, I configured my topology without static or dynamic routing with only having direct connections between the routers.

---
# Task 2 - OSPF Learning & Configuring
> **1.** `Deploy OSPF in your chosen network topology.`

Afterwards, I stared deploying OSPF in my topology. Actually, since I had started using IPv4 I used OSPFv2.
1. Below is the example of configuring OSPF for the router `R1` -- `ether1` interface
	* First, I added **OSPF-instance** (`ospf-instance-1`) with the **router-id** = `1.1.1.1`, **OSPF-area** (`ospf-area-1` - `0.0.0.0` ) and OSPF-interface (`ether1` - `10.0.0.1/24`).
		![[Pasted image 20241205233513.png]]
	* What is noticeable above is that router's state changed from `waiting` to `dr` after 40s (== `dead-interval`) which means that the router was selected as Designated Router inside the OSPF network where there is only one the same router 😄
	* Also, with the help of Wireshark we can see that after adding the interface (`ether1`) to the OSPF the router started sending OSPF `Hello` packets broadcasting it (Destination == `224.0.0.5`) over this interface.
		![[Pasted image 20241205234045.png]]
2. Okay, then my task was to deploy OSFP to the router `R2` -- `ether1` interface
	* I repeated the same actions for the `R2` setting its router-id = `2.2.2.2`
		![[Pasted image 20241206003741.png]]
	* And we immediately see that `R2` got updates from `R1` becoming `bdr` while `R1` kept being the Designated Router 
		![[Pasted image 20241206003900.png]]
	* Ant it's easy to notice from the perspective of Wireshark. Both routers sent information about their neighbours in `Hello` packets and started exchanging their LSA summary in `DB Description` packets. Then, they requested absent LSAs, acknowledged it, and moved to the `FULL` state finally.
		![[Pasted image 20241206004649.png]]
	* This process can be represented in such a [state machine](https://www.firewall.cx/networking/routing-protocols/ospf-adjacency-neighbor-states-forming-process.html)
		![[Pasted image 20241206005058.png]]
	* Now we can check `R2` node's neighbour table and LSDB
		![[Pasted image 20241206005503.png]]
	* We see that it was updated and we actually see that our neighbour is `R1` with the `Full` state. What is more, we can see an LSA with `type 2 Network` which means that in this particular subnet there are only 2 routers right now. Let's check neighbour table and LSDB of the router `R1`
		![[Pasted image 20241206005804.png]]
	* The same is applied here. However if we check the routing tables on both of the routers, we see the same direct connected networks.
		![[Pasted image 20241206010039.png]]
		![[Pasted image 20241206010047.png]]
	* To advertise other subnets, we need to deploy OSPF to all the routers' interfaces
3. Let's deploy OSPF to `R1` -- `ether2` and `R2` -- `ether2`
	* First, let's just add interfaces to the OSPF domain
		- `R1`
			![[Pasted image 20241206010839.png]]
		- `R2`
			![[Pasted image 20241206010849.png]]
	* Afterwards, we notice that routers sent updates to their common subnet and advertised their own subnets with the help of `LS Update/Ack`
		![[Pasted image 20241206010934.png]]
	* Finally, we can see that the routing tables were updated and we can ping `PC2` from `R1` and `PC1` from `R2`
		* `R1`
			![[Pasted image 20241206011547.png]]
		* `R2`
			![[Pasted image 20241206011626.png]]
		- We could notice that Distance $=110$ is exactly the OSPF's administrative distance
	* And of course `PC1` can reach `R2` and `PC2`
		![[Pasted image 20241206012029.png]]
	* Meanwhile `PC2` can reach `R1` and `PC1`
		![[Pasted image 20241206012036.png]]
	- `R1`'s and `R2`'s LSDBs were updated and synchronised
		- `R1` LSDB
			![[Pasted image 20241206012553.png]]
		- `R2` LSDB
			![[Pasted image 20241206012600.png]]
4. Finally, we can deploy OSPF in the same way for all the other routers and its interfaces
	* After doing it, we can see for example `R3` router's neighbour table. (Notice also that I added 3 interfaces of `R3` using 1 template `10.0.0.0/8`).
		![[Pasted image 20241206015142.png]]
	* LSDB
		![[Pasted image 20241206015324.png]]
	* Or `R4`'s routing table
		![[Pasted image 20241206015347.png]]

> **2.** `Which interface you will select as the OSPF router ID and why?`

- **Loopback interface**: following best practices, the best choice is to configure loopback Interface on a router. The highest loopback interface's IP address is selected as Router ID by default. his is because loopback interfaces are **always up** and do **not depend on physical interfaces**, providing constant **availability** and **stability**.
- **Highest non-loopback interface IP**: If a loopback interface is not configured, OSPF will automatically select the highest IP address on any of the **active** interfaces as the Router ID.
- **Explicit configuration**: In addition, it is always possible to configure Router ID manually. It may helps to avoid relying on the automatic selection and to prevent collisions.

In my case, I configured it manually and set it in the following way:
1. `R1` - `1.1.1.1`
2. `R2` - `2.2.2.2`
3. `R3` - `3.3.3.3`
4. `R4` - `4.4.4.4`

> **3.** `What is the difference between advertising all the networks VS manual advertising (per interface or per subnet)? Which one is better?`

**⚠️ Note**: In this exercise and further router `R5` exists in my topology because I started doing it after I completed Task 3 where I explained why and how I had added it

Generally, manual advertising is better. In addition, one can manually configure advertising all the networks (`0.0.0.0/0`). However, I believe that anyway administrators should be as specific as possible configuring the wildcard which will be applied to all the interfaces on which IP address matches the mask.

For example, other networks can be connected a router but these networks may use not OSPF protocol (RIP, BGP, etc.) or this network may lay in a different OSPF domain - in other words may lay in a different Autonomy. In such a case a redistribution of routes can be configured, while the router should become **ASBR**.

What is more, in Mikrotik example specifically, we can only configure a cost and other parameters with the help pf `interface-template` which can include many interfaces.

However, if an administrator is sure that there will no other autonomies, than it may be more convenient to configure advertising all the networks.

For instance, I configured `R3` and `R4` routers using the mask (`10.0.0.0/8`) while `R1` and `R2` was configured by interfaces `ether1` and `ether2` separately:
* `R1`
	![[Pasted image 20241206021830.png]]
* `R2`
	![[Pasted image 20241206021841.png]]
* `R3`
	![[Pasted image 20241206021848.png]]
* `R4`
	![[Pasted image 20241206021856.png]]

> **4.** `If you have a static route in a router, how can you let your OSPF neighbors know about it? Approve and show it on practice`

- Let's change the link between `R3` and `R4` to be not an OSPF
	- From the perspective of `R3`
		![[Pasted image 20241207223739.png]]
	- With not forgetting to return `10.0.30.0/24` network to keep connectivity to the `PC3`
		![[Pasted image 20241207223947.png]]
	- And the same from the perspective of `R4`
		![[Pasted image 20241207224221.png]]
- Then, let's configure the static route from the `R3` to `10.0.40.0/24` through the `R4`
	![[Pasted image 20241207225408.png]]
- However, we still cannot reach `10.0.40.0/24` from the other routers such as `R2` because they have no suitable route in their routing tables
	![[Pasted image 20241207225746.png]]
- So now, let's make `R3` an ASBR.
	-  We can just add `redistribute` flag with the value `static` to our OSPF instance on the router `R3` to make it an ASBR for this particular OSPF domain redistributing its static routes as `type 5: External` LSAs throughout the whole domain
		![[Pasted image 20241208192839.png]]
	- We can immediately see this LSDB update requests initiated by `R3`
		![[Pasted image 20241208193427.png]]
		![[Pasted image 20241208193435.png]]
	- As we see these requests contain exactly the information that `R3` became an ASBR for this OSPF domain and this particular area, and new external Autonomous System, which is is marked  as `LSA-type 5`. Therefore, now we can actually make sure that this LSA exists in the LSDB
		![[Pasted image 20241208193856.png]]
- Although now we can reach the `10.0.40.0/24` from our OSPF domain it was not enough and we got only timeouts
	![[Pasted image 20241208194217.png]]
- They occurred because packets now can be easily routed outside the OSPF domain, but `R4` knows nothing about how to route them back.
	- So, let's also add a static route from `R4` to our OSPF domain through `R3`
		![[Pasted image 20241208200232.png]]
	- Notice that I have added `10.0.0.0/8` which is enough to ensure the coverage of the current addresses in the whole topology. A key point here is route precedence rules (packets go through the link which is specified as the most specific over all the routing table entries) keep the connectivity with `10.0.40.0/24` - and, thus, with `PC4`
		![[Pasted image 20241208200746.png]]
- Finally, me made it possible to reach another Autonomous System that uses static routing from our OSPF domain
	![[Pasted image 20241208201102.png]]

> **5.** `Enable OSPF with authentication between the neighbors and verify it`

I decided to enable OSPF authentication based on the simple plain-text password transmission inside the OSPF `Hello` packets' header
- First, I enabled the authentication on `R1`
	- We can see there are 3 `R1`'s neighbours. Also, other routers' subnets are reachable
		![[Pasted image 20241208225210.png]]
	- I set up the password `password` for the interface `ether1` which goes to the subnet `10.0.0.0/24` where other routers are located also. The interesting part here is that it became `DR` inside its own authenticated multicast link.
		![[Pasted image 20241208225635.png]]
	- And we can see the updates in Wireshark. `R1` had sent its auth data inside the OSPF `Hello` packet's header
		![[Pasted image 20241208230015.png]]
	- Next, we can see that router `R1` "lost" all its neighbours, while other routers "lost" him as a neighbour.
		![[Pasted image 20241208230305.png]]
		![[Pasted image 20241208230313.png]]
	- What is more, since there is no router who can become a neighbour to `R1`, it lost the reachability to other routers and vice a versa
		![[Pasted image 20241208230458.png]]
		![[Pasted image 20241208230504.png]]
- Okay, now let's configure authentication with the same password on `R2`
	- We can see there are 2 `R1`'s neighbours. Also, these routers' subnets are reachable
		![[Pasted image 20241208230803.png]]
	- I set up the password `password` for the interface `ether1` which goes to the subnet `10.0.0.0/24` where other routers are located also. The interesting part here is that it became `BDR` inside its authenticated multicast link, while `DR` remained the same - `R1`.
		![[Pasted image 20241208231052.png]]
	- We can also see the updates in Wireshark. 
		![[Pasted image 20241208231202.png]]
	- Now, we can see that `R1` and `R2` became neighbours, while other routers "lost" them both.
		![[Pasted image 20241208231320.png]]
		![[Pasted image 20241208231329.png]]
		![[Pasted image 20241208231416.png]]
	- And of course now we can reach both the routers from each other
		![[Pasted image 20241208231628.png]]
		![[Pasted image 20241208231637.png]]
---
# Task 3 - OSPF Verification

> **1.** `How can you check if you have a full adjacency with your router neighbor?`

* I can check it looking at neighbour's state. Let's see at the `R3` example:
	![[Pasted image 20241206025620.png]]
* I mentioned the state machine and possible state values above in the **Task 2**
* To make it more demonstrative, I added another one router `R5` (router-id=`5.5.5.5`) and PC `PC5`
	![[Pasted image 20241206235009.png]] 
* Then, I deployed OSPF to both `R5`'s interfaces
- After all we can see that `R3` has the same neighbours in `FULL` state: `R1`, `R2` in `10.0.0.0/24`, and `R4` in `10.255.255.0/30`. But where is `R5`?
	![[Pasted image 20241207000000.png]]
- And `R5` has indeed `TwoOnly` state from the `R3`'s perspective. 
	![[Pasted image 20241207000116.png]]
- This happened because `R5` and `R3` do not need to synchronise LSDB between each other. For this purpose the both have DR and BDR routers, `R1` and `R2` correspondingly

> **2.** `How can you check in the routing table which networks did you receive from your neighbors?`

* Since we receive all the updates for LSDB from our neighbours (and to be precise from DR or BDR) we can only check that entries of the routing table were generated by OSPF: 
	![[Pasted image 20241207002003.png]]
- Another way which is actually more detailed is to use `routing/route/` instead
	![[Pasted image 20241207190601.png]]

> **3.** `Use traceroute to verify that you have a full OSPF network.`

- I didn't get what does it mean to "have a full OSPF network", so I just performed `traceroute` from `R1` to every subnet
	![[Pasted image 20241207002958.png]]

> **4.** `Which router is selected as DR and which one is BDR ?`

It really depends on the subnet we are considering, or to be more precise, **Multicast Link**.
- For example, in the `10.0.0.0/24` `R1` was selected as DR and `R2` as BDR
	- `R1`
		![[Pasted image 20241207003736.png]]
	- `R2`
		![[Pasted image 20241207003752.png]]
- However, inside the `10.255.255.0/30` `R4` was selected as DR and `R3` as BDR
	![[Pasted image 20241207004327.png]]
- Actually, there always an election process between routers in the same Multicast Link.
	1. Routers check if there are already a DR or BDR elected in their network. If yes, then they just accept it.
	2. If no, routers elect one of them with the **highest priority** as a DR.
	3. If all the priorities are identical, then to break the tie the **highest router-id** is used.
	4. The same election process is applied for BDR after DR is elected.
- So, that's why in the `10.0.0.0/24` `R1` and `R2` was DR and BDR correspondingly -- they were enabled first one by one, therefore, other routers just accepted that during exchanging `Hello` packets.
- However, if I restart my topology, I will get `R5` (router-id=`5.5.5.5`) as DR and `R3` as BDR (router-id=`3.3.3.3`) because all the routers have the same priority $=128$ by default, but the highest router-ids belong to `R5` and `R4`
	- `R1` now in state `dr-other` in the `10.0.0.0/24` and among other things it is proven that is has `TwoWay` adjacency state with `R2` since thee both are not DR or BDR
		![[Pasted image 20241207010250.png]]
	-  `R5` now is DR and it has `Full` adjacency with each router in the `10.0.0.0/24`
		![[Pasted image 20241207010512.png]]
	- And in Wireshark we can see the actual election process.
		![[Pasted image 20241207010551.png]]
	- What is interesting here is that `R1` and `R2` send `LS Ack` packets to `224.0.0.6` that is the Multicast IP address for OSPF group containing DR and BDR routers only.
	- At the same time, both `R3` and `R3` send `LS Ack` to `224.0.0.5` that is Multicast IP address for OSPF group containing all the routers in a current Multicast Link.
 
> **5.** `Check what is the cost for each network that has been received by OSPF in the routing table.`

* Let's check OSPF path cost which is called precisely `ospf.metric` on `R1`
	![[Pasted image 20241207191338.png]]
- By default, each link has a link with an OSPF cost $=1$ in Mirkotik devices. Therefore, the cost from `R1` to `R4`'s subnet is $3$ while others have only $4$. In such a case cost are identical to the hops count. Since `R4` is separated from other routers by `R3`, the cost of path to it is $3$ which is really `R1` -> `R3` -> `R4`
- Now, let's check this metric from `R5` to `R4`
	![[Pasted image 20241207192320.png]]
- We can see that the cost to `R4`'s subnet `10.0.40.0/24` is exactly the same, which equals to $3$
- Now let's add another change to my topology:
	![[Pasted image 20241207193855.png]]
- We can see LSDB updates in Wireshark
	![[Pasted image 20241207193935.png]]
- And let's check the `ospf.metric` for the path from `R5` to `R4`
	![[Pasted image 20241207194042.png]]
- Now we see that it equals to $2$ because hops count now is $2$: `R5` -> `R4`
- But what if change cost to $10$ instead of $1$ for the interface on `R5` for the link `R5` -- `R4`
	![[Pasted image 20241207195818.png]]
- We see that now `ospf.metric` equals to $3$ for the path between `R5` and `R5`. This is because OSPF helped `R5` to help more efficient path with the less cost
	![[Pasted image 20241207200210.png]]
- The path from `R5` to `R4` that lays through the `Switch 1` costs $11$ in total. However, the most efficient path lays through the `Swtich` and `R3` which costs $3$ in total. OSPF protocol uses Dijkstra's algorithm for the purpose of selecting the route.
	
---
# References
* https://help.mikrotik.com/docs/spaces/ROS/pages/9863229/OSPF#OSPF-PropertyReference
* https://www.practicalnetworking.net/stand-alone/ospf-training-course-free-m1/
* https://www.firewall.cx/networking/routing-protocols/ospf-adjacency-neighbor-states-forming-process.html