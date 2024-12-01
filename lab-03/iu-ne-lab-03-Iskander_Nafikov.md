* **Name**: Iskander Nafikov
* **E-mail**: i.nafikov@innopolis.university
* **Username**: [iskanred](https://github.com/iskanred)
* **Hostname**: `lenovo`
---
# 1. Preparation

The preparations were done. I used Mikrotik router.
# 2. VLANs

> **a.** `Change the topology of your network to as follows, and make the necessary configurations.`

* I made a topology as [follows](https://imgur.com/SXJqeaD) also added changes to the topology from the previous labs (i.e. changed name of "Worker" node to "ITManager"). The results are the following:
	![[Pasted image 20241116192225.png]]

> **b.** `Configure the switches and make sure you have connectivity between the hosts.`

* I checked the connectivity between **ITManager** and Management & HR
	![[Pasted image 20241116193646.png]]
* Between **Management** and ITManager and HR
	![[Pasted image 20241116193837.png]]
* Between **HR** and ITManager and Management
	![[Pasted image 20241116193853.png]]

> **c.** `How do VLANs work at a packet level? What are the two main protocols used for this?`

**VLAN** is a technology that exists only in 2nd OSI-layer (Link layer). Therefore, it is not correct to mention working at a "packet" level, since packets are entities of the 3rd OSI-layer (Network layer). The more convenient would be to use word "frames". **Note**: Actually it can exist above the 2nd layer, but it is not related to `IEEE 802.1Q`, so I will not consider it

The frames contains special IEEE 802.1Q header
![[Pasted image 20241116194908.png]]
This header has the following structure:
![[Pasted image 20241116195141.png]]

Where:
* **TPID** = Tag protocol identifier: A 16-bit field set to a value of `0x8100` in order to identify the frame as an IEEE 802.1Q-tagged frame.
* **TCI** = Tag control information: A 16-bit field containing several sub-fields. The most interesting sub-field for us is:
	* **VID** = VLAN identifier: A 12-bit field specifying the VLAN to which the frame belongs.

Marking a frame with different **VID** is the same as making them belonging to different VLANs.

The main protocols used:
* **IEEE 802.1Q (or Dot1Q)**
* **Q-in-Q** (or **IEEE 802.1ad**) which is an extension to 802.1Q and it allows two or more VLAN headers in the same frame

> **d.** `What is the Native VLAN?`

**Native VLAN** is a concept in the [802.1Q](https://en.wikipedia.org/wiki/IEEE_802.1Q) standard that designates a VLAN on a switch where all frames go without a tag, i.e. traffic is transmitted untagged. By default, this is `VLAN 1`. In some switch models, such as Cisco, this can be changed by specifying another VLAN as native. The network switch automatically adds the labels of this VLAN to all received frames that do not have any labels. VLANs based on ports have some limitations. Only one VLAN can receive all the untagged frames!

> **e.** `Configure the VLANs on the switches to isolate the two virtual networks as follow.`

* I configured the topolgy as follows (some changes in port's names because of switches reconfiguration):
	![[Pasted image 20241124212421.png]]
* Here is an example of configuration of the switch `Administration`
	![[Pasted image 20241124212655.png]]

> **f.** `Ping between ITManager and HR, do you have replies? Ping between ITManager and Management, do you have replies? Can you see the VLAN ID in Wireshark?`

* Ping between `ITManager` and `HR`:
	* The address of `HR` remained the same
		![[Pasted image 20241124213641.png]]
	* The `HR` did not reply
		![[Pasted image 20241124213711.png]]
	* I decided to capture the traffic of the link between switch `Internal` and `Administration`. There we can see that ARP frames had `IEEE 802.1Q` ID $= 3$ of VLAN. However, there were no reply for these requests.
		![[Pasted image 20241124213848.png]]
* Ping between `ITManager` and `Management`. The same is applicable here the result is the opposite though:
	* The address of `Management` remained the same
		![[Pasted image 20241124214223.png]]
	* The `Management` did reply!
		![[Pasted image 20241124214232.png]]
	* ICMP requests were replied successfully. Inside their frames we can see the same `IEEE 802.1Q` ID $=3$
		![[Pasted image 20241124214244.png]]

> **g.** `Configure Inter-VLAN Routing between Management VLAN and HR VLAN and Show that you can now ping between them.`

I faced a **problem** with my configuration after configuring Inter-VLAN communication. The problems was that the router always tried to make `ARP` broadcast requests to the **wrong VLAN**:
-  For example, I tried `10.1.1.2` which was on VLAN $=3$. However, the router continued searching for this IP with ARP broadcast requests tagging its frame with ID$=2$ and vice a versa!
- I realized that the problem had been in that router had the same address for three interfaces: simple `bridge2`, `VLAN2`, `VLAN3`
	![[Pasted image 20241125001625.png]]
- Of course, there always been a confusion since the router using its internal algorithm chose some of the available interfaces and the choice had always been wrong 😢
	![[Pasted image 20241125002046.png]]
- Therefore, I decided to **change the subnets** for each of the VLANs to: `10.1.12.0/24` for `VLAN2` and `10.1.13.0/24` for `VLAN3`
	- Below is my **old** configuration
		![[Pasted image 20241125002655.png]]
	- So now, the configuration had been changed to:
		![[Pasted image 20241125003106.png]]
* Moreover, after that I had to **change the IP** addresses on my nodes `Management`, `HR` and `ITManager`
	* HR
		![[Pasted image 20241125003647.png]]
	* Management
		![[Pasted image 20241125003657.png]]
	* ITManager
		![[Pasted image 20241125004008.png]]
* Finally, I was able to perform ping from Management to HR and vice a versa:
	* Ping from `HR`
		![[Pasted image 20241125004450.png]]
	* Ping from `Management`
		![[Pasted image 20241125004528.png]]
	* Ping from `ITManager`
		![[Pasted image 20241125004539.png]]
* And we can see that ARP table in the router was updated correctly
	![[Pasted image 20241125010028.png]]
* What is interesting is that now in Wireshark we can see the ICMP request from the `HR` to `Management` twice
	* Below is capturing of the link `HR - Administration`. Here we have no VLAN ID in the header since it is added and stripped by `Administration` switch.
		![[Pasted image 20241125011659.png]]
	* However, if we look at the link `Administration - Internal` we will notice that there are two ICMP requests and replies within the different VLANs
		![[Pasted image 20241125011830.png]]
	* And that is an evidence that both the request and response were transferred across the `Administration` switch twice: before and after being routed into different VLAN
		* The request's path
			```
			HR -> Administration (VID = 2) -> Internal (VID = 2) -> Router (VID = 2) -> Router (VID = 3) -> Internal (VID = 3) -> Administration (VID = 2) -> Management
		    ```
		* The reply's path
			```
			Management -> Administration (VID = 2) -> Internal (VID = 3) -> Router (VID = 3) -> Router (VID = 2) -> Internal (VID = 2) -> Administration (VID = 2) -> HR
		    ```
# 3. Fault Tolerance
> **a.** `What is Link Aggregation? How does it work (briefly)? What are the possible configuration modes?`

**Definition**
**Link aggregation** is a concept from **IEEE 802.1AX** (IEEE 802.3ad in the past, but no longer related **only** to Ethernet) that provides a way of using a bunch of links to increase performance of transmission and/or to ensure fault tolerance. In this context such links may exist as real links between the same two devices using different ports. On the other hand, these links could be two different devices acting as one (to achieve it clients must use Cisco VSS or Nexus VCC proprietary protocols). A group of ports combined together is called a **link aggregation group**, or LAG (bond, team, or port-channel in different implementations) with maximum of 8 links in a group. The rule that defines which packets are sent along which link is called the **scheduling algorithm**. The active monitoring protocol that allows devices to include or remove individual links from the LAG is called **Link Aggregation Control Protocol** (LACP).

**How it works?**
LACP works by sending frames (LACPDUs) down all links that have the protocol enabled. If it finds a device on the other end of a link that also has LACP enabled, that device will independently send frames along the same links in the opposite direction enabling the two units to detect multiple links between themselves and then combine them into a single logical link. LACP can be configured in one of two modes: active or passive. In active mode, LACPDUs are sent 1 per second along the configured links. In passive mode, LACPDUs are not sent until one is received from the other side, a speak-when-spoken-to protocol.

**Configuration modes**
The configuration modes may include:
* **Scheduling** (load-balancing) algorithm. ***Example***: MAC-address hashing (L2), IP-address hashing (L3), socket hashing (L4), round-robin, other. It must be selected according utilisation. 
* **Active** or **passive** (or merely on), equivalently, to use LACP or not to use (e.g. to use PAgP in Cisco). The **active** option means the device will actively monitor the state of the link and automatically remove any failed links from the bundle. However, the other device may not support LACP, so it can lead to the situation when one device will keep dumping packets down a link that the other device isn’t watching.

> **b.** `Use link aggregation between the Web and the Gateway to have Load Balancing and Fault Tolerance as follows.`

![[Pasted image 20241201203716.png]]

- If we put just one more link for each of the parts of the path from the Web to the Gateway we will get such a picture:
	- All the traffic goes through one of the links constantly for both `Web - External (switch)` and `External (switch) - Mikrotik (router)`
		![[Pasted image 20241201171057.png]]
		![[Pasted image 20241201171231.png]]
	- While other links do not see any traffic
		![[Pasted image 20241201171313.png]]
		![[Pasted image 20241201171320.png]]
- So, let's firstly put a new `bonding` interface to our router which will combine `ether2` and `ether4` as slaves.
	* Let's have a look at the configuration of the router (`10.1.2.1/24`) with added `bonding` interface:
		![[Pasted image 20241201182735.png]]
	* The default algorithm of load balancing for a `bonding` interface in Mikrotik is `balance-rr` which is a simple Round Robin algorithm.
		![[Pasted image 20241201182825.png]]
	* Also, let's take a look at the configuration of the `Web` node (`10.1.2.2/24`). For now it's important just to check it, the reasons will be considered after.
		![[Pasted image 20241201182834.png]]
	- Let's check if it works
		- Let's perform ping from the `Router` to the `Web`
			![[Pasted image 20241201184151.png]]
		- We can notice that all ICMP traffic was equally divided between two links that are located between the nodes `Router` - `Switch`. 
			![[Pasted image 20241201184206.png]]
			![[Pasted image 20241201184216.png]]
		- However, the distribution between the links `Switch` - `Web` remained the same (only one link transmitted all the frames).
			![[Pasted image 20241201184231.png]]
			![[Pasted image 20241201184238.png]]
		- This happened because I didn't configured anything on the `Switch` side or `Web` side, so we had an impact only on links that are connect to the router directly. Since my switch is really limited in its configuration (because of version of GNS3 used), let's configure only `Web` side.
- So, to configure the `Web` side for link aggregation let's use `netplan`
	- First let's configure bonding interface on the `Web` using `netplan`
		![[Pasted image 20241201195608.png]]
	- So, let's again perform ping from the `Router` to the `Web`
		![[Pasted image 20241201195954.png]]
	- Now it's noticeable that both links transmitted ICMP traffic
		![[Pasted image 20241201200147.png]]
		![[Pasted image 20241201200154.png]]
	- What is interesting here, is that all ICMP requests always go through the same link. Not surprisingly, all replies also always go through the other link.

> **c.** `Test the Fault Tolerance by stopping one of the cables and see if you have any downtime`

- I decided to remove the link from the `Web` to `External`. But before it I started pinging from the `Web` to the `Router`
- Then, I removed the link
	![[Pasted image 20241201204322.png]]
- Looking at ping further we can see there was no downtime at all! 0% packet loss is the proof 
	![[Pasted image 20241201204035.png]]
- Let's check Wireshark capturing frames
	- Here we can see the link that was removed
		![[Pasted image 20241201204238.png]]
	- Here is the link that was NOT removed
		![[Pasted image 20241201204231.png]]
* Nothing was duplicated and there was no downtime. Probably, such a fast forwarding was caused of using failure detection algorithm that is called **media-independent interface** (MII) which is fast and it depends on the device driver.
---
# References
* https://en.wikipedia.org/wiki/IEEE_802.1Q
* https://en.wikipedia.org/wiki/VLAN
* http://xgu.ru/wiki/Native_VLAN
* https://www.smart-soft.ru/blog/tehnologija_vlan/
- https://www.auvik.com/franklyit/blog/network-basics-link-aggregation/