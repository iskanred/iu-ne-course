* **Name**: Iskander Nafikov
* **E-mail**: i.nafikov@innopolis.university
* **Username**: [iskanred](https://github.com/iskanred)
* **Hostname**: `lenovo`
---
#  Task 1 - Prepare your network topology

> **1.** In the GNS3 project, select and install a virtual routing solution that you would like to use: for example, Mikrotik (recommended), Pfsense, vyos.

I prefer Mikrotik

> **2.** Prepare a simple network consisting of at least three router and two hosts. Each one of them has a different subnet, and the routers should be able to reach each other (for example, a bus topology with dynamic routing). Your network must have routing protocols configured.

- I decided to create a new network that will be more interesting for configuring MPLS and to practise OSPF configuring.
	![[Pasted image 20241217042252.png]]
- This is routers' IP addresses that lay in different subnets. Pay attention to the address assigned to loopback `lo` interface. It was given to be a router-id in OSPF as a LSR-ID in LDP
	![[Pasted image 20241217152237.png]]
# Task 2 - MPLS learning & configuring

>**1.** Briefly answer the questions or give one-line description what is it: LSP, VPLS, PHP, LDP, MPLS L2VPN, CE-router, PE-router?

- **MPLS (Multiprotocol Label Switching)**: High-performance networking technology that directs data from one node to the next based on short, fixed-length labels rather than long network addresses, which speeds up the flow of traffic across the network. It enables the creation of end-to-end circuits (LPS) which can support various types of traffic, including IP packets, by efficiently managing Traffic Engineering such as managing bandwidth and ensuring quality of service (QoS). MPLS is commonly used in enterprise and service provider networks to create Virtual Private Networks (VPNs), improve traffic management, and optimise network performance.
	![[Pasted image 20241216194056.png]]
- **LSP (Label Switched Path)**: A predetermined path in an MPLS network through which labeled packets travel from a source to a destination. This path is established using special protocols such as LDP or RSVP.
- **LDP (Label Distribution Protocol)**: A protocol used in MPLS networks to establish the mapping between network layer (IP) addresses and MPLS labels.
- **VPLS (Virtual Private LAN Service)**: A Layer 2 VPN service that allows multiple customer sites to communicate as if they are on the same local area network (LAN), regardless of geographic locations. An alternative is VPWS (Virtual Private WAN Service) which implements the same idea but for a point-to-point interaction.
- **PHP (Penultimate Hop Popping)**: A technique used in MPLS networks where the penultimate router removes the MPLS label, passing the packet to the egress router in MPLS network with only the IP header instead of replacing this label to `Explicit Null` (label $=0$) which is used by default by to signal the egress router to remove the label. For this purpose an egress routers sends an `Implicit Null` (label $=3$) message to its penultimate router. This might be used where QoS (on the MPLS level) is not important because after stripping the MPLS label, everything `Traffic Class` field disappears.
- **CE-router (Customer Edge Router)**: A router located at the customer’s premises that connects to the service provider’s network and typically performs routing between the customer’s internal network and the service provider's network. This router does not belong to MPLS network and does not know anything about it. It is just connected to a PE-router.
- **PE-router (Provider Edge Router)**: A router located at the edge of the service provider’s network that connects to customer edge devices. It acts as a gateway for the customer to the MPLS network. Another name is **LER (Label Edge Router)**.
- **L2VPN**: A service that provides Layer 2 connectivity over an MPLS network, enabling Ethernet, Frame Relay, or ATM services across geographically dispersed sites.

>**2.** Configure MPLS domain on your OSPF network, first without authentication.

- I configured MPLS "dynamically" using LDP
	```shell
	# R1
	/mpls ldp add afi=ip lsr-id=11.11.11.11 transport-addresses=11.11.11.11
	/mpls ldp interface add interface=ether1
	/mpls ldp interface add interface=ether2
	# ether3 is a customer network

	# R2
	/mpls ldp add afi=ip lsr-id=22.22.22.22 transport-addresses=22.22.22.22
	/mpls ldp interface add interface=ether1
	/mpls ldp interface add interface=ether2
	# ether3 is a customer network

	# R3
	/mpls ldp add afi=ip lsr-id=33.33.33.33 transport-addresses=33.33.33.33
	/mpls ldp interface add interface=ether1
	/mpls ldp interface add interface=ether2
	# ether3 is a customer network

	# R4
	/mpls ldp add afi=ip lsr-id=44.44.44.44 transport-addresses=44.44.44.44
	/mpls ldp interface add interface=ether1
	/mpls ldp interface add interface=ether2
	# ether3 is a customer network
	```
- And we see that it worked
	![[Pasted image 20241217153106.png]]

>**3.** Enable authentication (what kind of authentication did you use)? Make sure that you can ping and trace all your network.

- I found no information on how to configure authentication for LDP's TCP packets (since they can be sniffed and corrupted) as in [Cisco](https://www.cisco.com/c/dam/en/us/td/docs/ios/12_0/12_0sy/feature/guide/md5orig.pdf) for example. [It seems](https://forum.mikrotik.com/viewtopic.php?t=149304) there is no such a possibility and nobody talks about it. However, we know that LDP uses OSPF for routing based on labels. Thus, if we OSPF hosts have other `auth-key`, these paths won't be distributed over all LDP peers.
- Therefore I configured `MD5` authentication on all the routers' interfaces
	```shell
	/routing ospf interface-template set numbers=0,1 auth=md5 auth-id=1 auth-key=password
	```
- And we can see that it worked, the password was encrypted or better to say hashed
	![[Pasted image 20241217160207.png]]

# Task 2 - Verification

> **1.** Show your LDP neighbors.

- The command
	```shell
	mpls ldp neighbor print
	```
- `R1`
	![[Pasted image 20241217163336.png]]
- `R2`
	![[Pasted image 20241217163344.png]]
- `R3`
	![[Pasted image 20241217163354.png]]
- `R4`
	![[Pasted image 20241217163401.png]]
- We can conclude that the configuration was done correctly

>**2.** Show your local LDP bindings and remote LDP peer labels.

- The command for LDP local bindings
	```shell
	mpls ldp local-mapping print
	```
- The command for LDP remote peer labels
	```shell
	mpls ldp remote-mapping print
	```
- `R1`
	![[Pasted image 20241217164156.png]]
- `R2`
	![[Pasted image 20241217164351.png]]
- `R3`
	![[Pasted image 20241217164357.png]]
- `R4`
	![[Pasted image 20241217164408.png]]
- We can see that we have `impl-null` which implies PHP from `R2` & `R3` to `R4` going from `R1` ingress and vice a versa: from `R2` & `R3` to `R1` going over from `R4` ingress.

>**3.** Show your MPLS labels.

W
The labels are:
- `impl-null`
- 16 
- 17
- 18
- 19
- 20
- 21
- 22
- 23
We see that for each FEC (Forwarding Equivalence Class) or simply subnet there is a unique label

>**4.** Show your forwarding table.

- The command
	```shell
	/mpls forwarding-table print
	```
- `R1`
	![[Pasted image 20241217181954.png]]
- `R2`
	![[Pasted image 20241217182001.png]]
- `R3`
	![[Pasted image 20241217182008.png]]
- `R4`
	![[Pasted image 20241217182016.png]]

>**5.** Show your network path from one customer edge to the other customer edge.

- The command
	```shell
	/tool traceroute {dst-address} src={src-address}
	```
- `10.0.10.1` (`R1 - ether3`) to `10.0.40.2` (`PC4`)
	![[Pasted image 20241217182738.png]]
- We see MPLS works and the label used to transmit packets from `R1` to `R3` to reach `10.0.40.2` $=20$. Then `R3` strips MPLS header and sends it right to `R4`. Then `R4` transmits traffic to `PC4` using IP.
- `10.0.10.1` (`R1 - ether3`) to `44.44.44.44` (`R4 - lo`)
	![[Pasted image 20241217182843.png]]
- We see MPLS works and the label used to transmit packets from `R1` to `R2` to reach `44.44.44.44` $=23$. Then `R2` strips MPLS header and send it right to `R4`.
# Task 3 - MPLS packets analysis

> **1.** Can you use Wireshark to see the MPLS packets?

- Yes, e.g. link `R1` - `R2` (`10.0.13.0/24`). I did ping and we can see ICMP request on the left and ICMP reply on the right
	![[Pasted image 20241217183522.png]]
- The key moment here is that for ICMP request `R1` gives label $=20$ and we can see it in MPLS header. However, in the reply PHP was occurred and `R2` stripped MPLS header before transmitting it from `R4` to `R1`.
- We see that
	- Label is **20**
	- This label is the **bottom** of MPLS labels stack
	- Time To Live is **255** hops
	- And strange experimental bits which is actually a QoS field 
		> The use of "experimental" bits is not specified by MPLS standards, but the most common use is to carry QoS information, similar to 802.1q priority in the VLAN tag. Note that the EXP field is 3 bits only therefore it can carry values from 0 to 7 only, which allows having 8 traffic classes.

>**2.** Look deeper into the MPLS packets: can you identify MAC address, ICMP, Ethernet header or something else useful?

In Wireshark I can identify anything actually and we can see it on the pictures above
# Task 4 - VPLS

> **1.** Configure VPLS between the 2 hosts edges.

- I decided to configure VPLS between `R1` and `R4`
- The command
	```shell
	# R1
	/interface vpls add name="1-to-4" peer=44.44.44.44 vpls-id=1:1
	
	#R2
	/interface vpls add name="4-to-1" peer=11.11.11.11 vpls-id=1:1
	```

>**2.** Show your LDP neighbors again, what has been changed?

- Let's check neighbours table on our VPLS-connected routers
	- `R1`
		![[Pasted image 20241217224355.png]]
	- `R4`
		![[Pasted image 20241217224410.png]]
- Now we can notice that `R1` is a neighbour of `R4` and vice a versa, but earlier this was not true. 
- What is also noticeable is that we see here `t - SENDING-TARGETED-HELLO` and `v - VPLS` flags.
- `t - SENDING-TARGETED-HELLO` means that LDP `Hello` message will be sent exactly from `44.44.44.44` to `11.11.11.11` and vice a versa. However, other routers prefer IP mutlicast using `224.0.0.2` address
	![[Pasted image 20241217225217.png]]

>**3.** Find a way to prove that the two customers can communicate at OSI layer 2.

- We can use "Layer 2 ping" or LLC protocol
	- `R1` to `R4` through the VPLS interface
		![[Pasted image 20241217234314.png]]
	- `R4` to `R1` through the VPLS interface
		![[Pasted image 20241217234323.png]]
- And we see exactly the stack of labels now (16 and then 23)
	![[Pasted image 20241217234426.png]]

>**4.** Is it required to disable PHP? Explain your answer.

I see 2 cases where it can be needed:
1. In MPLS networks where QoS is used. In these cases it may be needed that QoS won't be removed from penultimate hop to the next because QoS type is stored in MPLS header (also, in IP, but penultimate routers will not check IP header transmitting it to the destination).
2. In MPLS networks where label stacking is used (e.g., when using MPLS in Layer 2 VPNs or L3 VPNs), it might be necessary to disable PHP to ensure that the entire label stack is maintained until the packet reaches the appropriate egress point. This is important for proper label processing.
---
# References
- https://en.wikipedia.org/wiki/Multiprotocol_Label_Switching
- https://help.mikrotik.com/docs/spaces/ROS/pages/40992794/MPLS+Overview
- https://www.cisco.com/c/dam/en/us/td/docs/ios/12_0/12_0sy/feature/guide/md5orig.pdf