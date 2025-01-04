* **Name**: Iskander Nafikov
* **E-mail**: i.nafikov@innopolis.university
* **Username**: [iskanred](https://github.com/iskanred)
* **Hostname**: `lenovo`
---
# Task 1 - Prepare your network topology

> **1.** In the GNS3 project, select and install a virtual routing solution that you would like to use: Mikrotik (recommended), Pfsense, vyos and so on.`

I decided to use Mikrotik as recommended - `Mikrotik CHR 7.16`

> **2.** Prepare a simple network consisting of at least one router and two hosts. About four hosts in the network are most optimal. You also might need Internet access for the hosts. It may be something like this, just for the imagination:

I configured the following network. The internet access from PCs was configured using NAT (`ip firewall nat`).
![[Pasted image 20241209180739.png]]

---
# Task 2 - QoS learning & configuring

> **1.** Let's start with a little theory. Briefly answer the questions or give one-line description what is it: Сlass of Service (CoS), ToS (Type Of Service), Differentiated Services Code Point (DSCP), Serialization, Packet Marking, Tail Drop, Head Drop, The Leaky bucket algorithm, The Token Bucket Algorithm, Traffic shaping, Traffic policing

- **Class of Service (CoS)**: A mechanism for prioritizing network traffic by classifying packets into different categories for better Quality of Service (QoS), and it is implemented in Link Layer having special 3-bit field in 802.1Q header.
- **Type of Service (ToS)**: An older field in the IP header used to indicate the priority and treatment of packets in terms of delay, throughput, and reliability.
- **Differentiated Services Code Point (DSCP)**: A field in the IP header used for marking packets to indicate the desired level of service, enabling QoS strategies.
- **Serialization**: The process of converting data structures or objects into a format suitable for transmission over a network.
- **Packet Marking**: The practice of labeling packets with specific information (like priority) to influence their treatment as they traverse the network.
- **Tail Drop**: A queuing discipline where packets are dropped from the end of the queue when it reaches its maximum capacity.
- **Head Drop**: A queuing discipline where packets are dropped from the front of the queue, which can be used in certain network management strategies, though it’s less common than tail drop.
- **The Leaky Bucket Algorithm**: A traffic shaping method that controls data output rates by allowing data to "leak" out at a constant rate, smoothing bursts.
- **The Token Bucket Algorithm**: A traffic shaping technique that allows for bursts of data while maintaining a defined average rate by using tokens to regulate packet transmission.
- **Traffic Shaping**: The technique of controlling the volume of traffic being sent into a network to optimize or guarantee performance, improve latency, or manage bandwidth.
- **Traffic Policing**: A method of monitoring network traffic and enforcing bandwidth limits by dropping or marking packets that exceed the agreed rate.

> **2.** Configure your network as you decided above. After your network is configured (don't forget to show the main configuration steps in the report), try to set a speed limitation (traffic shaping) between the two hosts.

I configured my network in the **Task 1**. So now, let define the speed limits.

1. But before, let's investigate what is the current maximum speed without any explicitly set limits.
	- First, it is written in Mikrotik documentation Cloud Host Router (CHR), which is "a RouterOS version intended for running as a virtual machine", has a speed limit `1Mbit` for a free version.
		![[Pasted image 20241210000215.png]]
	- Let's verify it
	- To compute the bandwidth of the router and/or of a link, I added another one router `TestRouter` and connect it to my main one named `Router` in a new LAN `10.0.50.0/24`
		![[Pasted image 20241209235941.png]]
	- I did it to run a Mikrotik tool called **[bandwidth-tester](https://help.mikrotik.com/docs/spaces/ROS/pages/7962644/Bandwidth+Test)** which requires a server and a client running on Mikrotik device. Let's run speed tests
		- From `TestRouter` to `Router`
			![[Pasted image 20241210000840.png]]
		- From `Router` to `TestRouter`
			![[Pasted image 20241210000926.png]]
	- And the speed graph which I took from the Mikrotik Web
			![[Pasted image 20241210000959.png]]
	- To verify it completely I run performance test using `iperf` from `PC4` as a client to `PC2` as a server at this time using UDP to avoid losing sending or receiving speed due to TCP's congestion control and its bidirectional nature. To make it more convincing I ran performance testing **2** times: from the client (`PC4`) to the server (`PC2`) and vice a versa from the server to the client using an option `--reverse` (`--bidir` was not an option since it used to me [wrong results](https://www.reddit.com/r/networking/comments/142h9ja/iperf3_udp_with_option_bidir_it_lost_about_50/)). Also, I set `--bandwidth` equals to `20 Mbit/s` to be sure that the generating bitrate won't influence on the result (by default it is `1 Mbit/s` which may confuse).
		![[Pasted image 20241214050819.png]]
	- We see that the bitrate is almost `1 Mbit`. Considering computational errors over this time interval it is easy to conclude that `1 Mbit` is a hard-limit.
	- **Conclusion**: Finally, we can conclude that the 🔴 **limit actually exists** 🔴, it is configured to `1 Mbit` and it is configured inside a router's firmware. To increase the limit users should purchase paid versions. Therefore, let's continue using the free version and `1 Mbit` limit.
	- I removed `TestRouter` and, thus, returned the topology's original image
		![[Pasted image 20241209180739.png]]
2.  Now let's set the speed limits using the traffic shaper on our router for the hosts `PC3` and `PC3`
	- I decided to apply traffic shaper to my Mikrotik device for connections between `PC1` - `PC3`.
	- To set traffic shaping or scheduling I had to use queues, so I decided to learn something about them and created such a graphical document for myself to structure my findings and keep my knowledge. I decided not to deep dive into `queue tree` or Hierarchical Token Buckets HTB topics because they are too difficult for me now. Therefore, I tried to keep detauls about `queue simple`. This block may not be detailed or totally correct, but it is clear for me.
		![[Pasted image 20241214053529.png]]
	* For this purpose I firstly configured a new queue type `full-shaper` based on the `pfifo` (default packet FIFO queue) and set queue size (`pfifo-limit`) to 1 packet making it almost a full [100% shaper](https://help.mikrotik.com/docs/spaces/ROS/pages/137986083/Queue+size#Queuesize-100%25Shaper). This means that almost every packet that goes over the predefined limit will be dropped immediately. Therefore, it will cause a **big packet loss**, but **low-latency packets delivery**. I decided to use `pfifo` just because of its simplicity.
		```shell
		/queue type add name="full-shaper" kind=pfifo pfifo-limit=1
		```
		![[Pasted image 20241214014823.png]]
	* Then I created a `simple` queue. I decided to use `queue simple` just for the sake of its simplicity. I configured a queue in the following format:
		```shell
		/queue simple add name="host_1-3_parent_queue" target="10.0.30.2" dst="10.0.10.2" queue="full_shaper/full_shaper" max-limit="200K/400K"
		```
		![[Pasted image 20241214015014.png]]
	* As you can I configured a queue `host_1-3_parent_queue` for the link between `PC3` (`target=10.0.30.2`) and `PC1` (`dst=10.0.10.2`) with the queue type `queue=full-shaper` for both downloading and uploading traffic and the actual max bandwidth of `200 Kbit/s` for uploading and of `400 Kbit/s` for downloading (from the client's perspective).
 
> **3.** Run a bandwidth testing tool, see what is the max speed you can get and verify your speed limitation. Compare the speed between the different hosts.

* Now let's check if it works by running `iperf3` again (`PC1` is a server, `PC3` is a client, UDP traffic)
	![[Pasted image 20241214053852.png]]
* We see that it worked and the actual bitrate was definitely about `200 Kbit/s` and `400Kbit/s` for uploading and downloading correspondingly. The scheme of interaction look like:
	![[Pasted image 20241214053439.png]]
* Nevertheless, we can notice that the packet loss is enormous: **99**% for the client's packets and **98**% for the server's packets.
	![[Pasted image 20241214054005.png]]
- Such number of lost datagrams happened because UDP is not reliable, especially when the bandwidth and the actual bitrate differ so much. Also, such big numbers occurred because of very aggressive traffic shaping which drops packets almost immediately with queue size = 1 packet.
- Therefore, we can tune it a bit and really slightly decrease packet loss configuring our shaper to be more a scheduler with the queue size = 300 packets.
	```shell
	/queue type set full-shaper pfifo-limit=300
	```
	![[Pasted image 20241214054953.png]]
- Afterward, we can notice this decrease on practice: 99% -> 97%   98% -> 83%
	![[Pasted image 20241214055105.png]]
- However, the delay was increased also. I had to wait longer for the end of test.
- Still the speed for other clients remained the same = `1 Mbit/s` even for the same server `PC1`. For example for the connection `PC4` -- `PC1`:
	![[Pasted image 20241214054617.png]]

> **4.** While your bandwidth test is still running, try to download a file from one host to the other host and see what is the max speed you can get. If you have more than two hosts on the network, play around with different speed values and show it.

* To complete this task I set up an FTP server on `PC1`. I used [pure-ftpd](https://www.pureftpd.org/project/pure-ftpd/) for FTP server
	![[Pasted image 20241215221145.png]]
- Also I created a file of 50MB on `PC1` using this Python script:
	```python
	with open('my_file', 'wb') as f:
	    num_chars = 1024 * 1024 * 1024
	    f.write('0' * num_chars)
	```
	![[Pasted image 20241215222738.png]]
- Before all, I decided to update my queue and set the limit to `1 Mbit/s` for both download and upload. Also, I changed its type to `pcq-upload-default/pcq-download-default` just because I wanted it to try, but there should be no effect since `pcq-classifier` is `dst-address` and it will be the same for both the connections: file download and perf.
	![[Pasted image 20241216013211.png]]
- Then I decided to collect the speed statistics without running perf firstly
	![[Pasted image 20241216014711.png]]
- We see that normally even with `1 Mbit/s` open channel the speed is only about `120 Kbit/s`. This can happen due to TCP traffic has different traffic guarantees which impact the final speed.
- Then I ran `iperf3` from `PC1` to `PC3`: UDP traffic with regular bandwidth = `900 Kbit/s`, not `1 Mbit/s` to leave some "space" for FTP traffic.
	```shell
	iperf3 -c 10.0.10.2 --bandwidth=300000 --udp --time=60
	```
	![[Pasted image 20241216015525.png]]
- Finally, I started FTP file download with `iperf3` running 
	![[Pasted image 20241216015724.png]]
- We can notice that the speed **dropped dramatically** to approximately `15 Kbit/s` and returned to `120 Kbit/s` only after perf ended.
	
> **5.** Deploy and verify your QoS rules to prioritise the downloading of a file (or any other scenario) over the bandwidth test.

- Now let me show my QoS configuration. I use traffic allocation technique: `limit-at`, child queue, packet marks and etc. Let me show it now
- First, I configured my `ip firewall mangle` to mark all FTP-related packets as `ftp-alloc`
	```shell
	ip firewall mangle>
	add chain=prerouting action=mark-packet new-packet-mark=ftp-alloc protocol=tcp src-port=42154
	add chain=prerouting action=mark-packet new-packet-mark=ftp-alloc protocol=tcp src-port=42155
	add chain=prerouting action=mark-packet new-packet-mark=ftp-alloc protocol=tcp dst-port=42154
	add chain=prerouting action=mark-packet new-packet-mark=ftp-alloc protocol=tcp dst-port=42155
	```
	![[Pasted image 20241216030447.png]]
- As you can see I added packet marking on `prerouting` chain (Rules in this chain apply to packets as they just arrive on the network interface) on TCP protocol for different sets of `dst-port` and `src-port`. But why such strange numbers: 42154, 42155? This is because I decided to run my FTP client on these ports (one port for a control channel and one for a data channel).
- Then, I added new queues that I made children of my main `host_1-3_parent_queue`.
	```shell
	queue simple>
	add name=ftp-alloc-queue target=10.0.30.2 dst=10.0.10.2 parent=host_1-3_parent_queue packet-marks=ftp-alloc queue=pcq-upload-default/pcq-download-default limit-at=900k/900k max-limit=1M/1M
	add name=ftp-alloc-queue target=10.0.30.2 dst=10.0.10.2 parent=host_1-3_parent_queue packet-marks=ftp-alloc queue=pcq-upload-default/pcq-download-default limit-at=900k/900k max-limit=1M/1M
	```
	![[Pasted image 20241216031858.png]]
- The first queue `frp-alloc-queue` is configured to be a queue for all the packets that are marked as `ftp-alloc` => all FTP-related packets between these hosts. This queue has `limit-at=900k/900k` meaning that despite all other traffic flows in other queues, this queue will get guaranteed `900 Kbit/s` for both upload/download while the maximum limit leaves the same `1 Mbit/s`.
- The second queue `other-traffic-queue` is also configured for the traffic between these hosts. However, here `limit-at` parameter equals to `100k/100k` meaning that this queue would get only guaranteed `100 Kbit/s` for both upload/download. The traffic would go to this queue only if previous queues were not convenient. Therefore, all non-FTP traffic would go to this queue.
- It's worth to say that in this case it's unnecessary to have a parent queue. However, it's more flexible.
- Finally, we can run our FTP file download
	![[Pasted image 20241216032511.png]]
- It's easy to notice that the speed drop in this case is really small, about `12 Kbit/s` from `118 Kbit/s` to `106 Kbit/s`
- It worked! And we can double check it in `PC1`'s `iperf3` stats
	![[Pasted image 20241216032731.png]]
- We see that the receiver's bitrate did not exceed `100 Kbit/s`

> **6.** What is the difference between the QoS rules to traffic allocation and priority-based QoS? Try to set up each of them and show then them. In which tasks of this lab do you use one or the other?

* **Traffic allocation** is a technique to define specific policies to control how much bandwidth is assigned to different types of traffic or users within a network. The example is above for downloading files with FTP. It is an efficient way to statically give specific available bandwidth!
* **Priorities** are used classifying packets and assigning them different priorities to ensure critical applications receive better bandwidth and low-latency treatment, regardless of overall usage. Here we cannot guarantee some specific number as a bandwidth value. However, it is better for low-latency applications (such as real-time) since marked packets have a higher probability to be passed first.

**Priority-based QoS configuration**:
- Since I already deployed traffic allocation technique in the previous task, I decided to deploy priorities using the same queues on Mikrotik.
- First, I removed `limit-at` for the `frp-alloc-queue` to make traffic allocation QoS not having an impact on the results. Nevertheless, I gave `limit-at=10k/10k` for the `other-traffic-queue` to allow non-FTP traffic to have at least some guaranteed speed.
- Then I gave the highest priority to the `ftp-alloc-queue` (1 from 8), while the priority for the `other-traffic-queue` remained the same (8 by default).
	![[Pasted image 20241216034251.png]]
- I ran started FTP file download and noticed that the speed didn't decrease at all!
	![[Pasted image 20241216034846.png]]
- Afterwards, I checked `iperf3` stats and found out that it even went wrong at the end because control socket has closed unexpectedly because of too many packets lost.
	![[Pasted image 20241216034841.png]]
- Therefore, we can conclude that it worked perfectly! In fact, without specified `limit-at` for `other-traffic-queue` `iperf3` couldn't even connect to the `PC3`
- I checked that even with priority = `7` `ftp-traffic-queue` already takes all the traffic, so it is necessary to set the `limit-at`.

**DSCP and CAKE**:
- What is more, I tried to use DSCP with the help of a [CAKE](https://help.mikrotik.com/docs/spaces/ROS/pages/196345874/CAKE) queueing discipline.
- Firstly, I created a new queue and I wanted to figure out how much speed I will get with for example [PCQ](https://help.mikrotik.com/docs/spaces/ROS/pages/137986099/PCQ+example) discipline.
	```shell
	queue add name=dscp-queue target=10.0.30.2 dst=10.0.10.2 queue=pcq-upload-default/pcql-
	```
	![[Pasted image 20241216045405.png]]
- However, it gave me 0 speed when running together with the `iperf3`
	![[Pasted image 20241216045734.png]]
- Therefore I created a new queue type based on CAKE queueing discipline to use it with DSCP
	```shell
	queuee type add name=my-cake-queue kind cake cake-diffserv=diffserv4
	queue simple set dscp-queue queue=my-cake-queue/my-cake-queue
	```
	![[Pasted image 20241216044207.png]]
- And of course I needed to configure marking FTP packets with DSCP values, so I configured them to have DSCP = 63 which is the maximum possible DSCP value
	```shell
	ip firewall mangle add chain=prerouting action=change-dscp new-dscp=63 protocol=tcp src-address=10.0.30.2 src-port=42154
	ip firewall mangle add chain=prerouting action=change-dscp new-dscp=63 protocol=tcp src-address=10.0.30.2 src-port=42155
	ip firewall mangle add chain=prerouting action=change-dscp new-dscp=63 protocol=tcp dst-address=10.0.30.2 dst-port=42154
	ip firewall mangle add chain=prerouting action=change-dscp new-dscp=63 protocol=tcp dst-address=10.0.30.2 dst-port=42155
	```
	![[Pasted image 20241216044201.png]]
- As a result the speed did reach 0, so it seems that it worked!
	![[Pasted image 20241216051220.png]]

> **7.** Choice and install any tool that you like for bandwidth control/netflow analysis/network control & monitoring. Play around with the network settings and show the different QoS metrics via UI



> **8.** Try to answer the question: packet drops can occur even in an unloaded network where there is no queue overflow. In what cases and why does this happen?

There are several cases where it is possible that I can figure out:
- **Firewalls** can apply special filtering for the packets and it might be not obvious for the sender what's happening.
- There can be **errors on physical layer** that can lead to packets corruption. This may look as a  packet loss from both sides.
- **TTL expiration**. Packets have TTL which is measured as number of hops usually. When TTL reaches zero, a packet is dropped and appropriate ICMP message is sent to the sender.
- **RED scheduling algorithm** can be applied for queuing or a similar algorithm. It has a certain probability to drop packets even if the queue is not full. This is done to prevent the overflow.
# Task 3 - QoS verification & packets analysis

> **1.** How can you check if your QoS rules are applied correctly? List and describe the various methods.

I am not sure of what is required, so I just can suggest such methods:

- **Traffic Generators and Test Tools**. Utilizing traffic generation tools (such as `iperf` or **Ostinato**) to simulate different types of traffic on your network. This can help you understand how QoS rules affect performance in a testing environment. Of course, it is better not to use it in production environment together with the production load. The method is pretty simple: generate traffic and verify its distribution with the expected values. 
- **Network flow analysis**. We can monitor on how traffic is distributed in production environment by exploring it using different analytical tools, network bitrate graphs and etc, e.g. `itop`.
- **Traffic Monitoring Tools**. Use network monitoring tools to observe real-time traffic flow and check whether the QoS rules are being enforced on the network. Tools like **Wireshark** can capture and help us analysing traffic. For example, in filtering traffic based on the QoS parameters (like DSCP values) to check if the packets are marked and prioritised correctly.
- **Network Device Interfaces**.  Check the configuration and status of network devices (routers, switches) where QoS is configured. For example, we can monitor if packets marking is happening using device's monitoring tools or accumulated statistics. For example, in Mirkotik it is possible to look for statistics on dropped packets, queue lengths, and traffic distribution.

> **2.** Try to use Wireshark to see the QoS packets. How does this depend on the number of routers in the network topology?

It really depends on the method of QoS used. For example, we wouldn't find anything interesting in Wireshark if traffic allocation technique is used. However, for DSCP case we can find.

* Let me show DSCP values that were assigned to FTP traffic from the previous task.
	![[Pasted image 20241216054252.png]]
* Here on the left we see a UDP datagram that was generated by `iperf3` and it had the default DSCP value = $0$. However, on the right hand-side we can notice our FTP packet which is marked with value = $63$
	![[Pasted image 20241216054649.png]]

---
# References
* https://help.mikrotik.com/docs/spaces/ROS/pages/18350234/Cloud+Hosted+Router+CHR
* https://help.mikrotik.com/docs/spaces/ROS/pages/7962644/Bandwidth+Test
* https://d2cpnw0u24fjm4.cloudfront.net/wp-content/uploads/iPerf3-User-Documentation.pdf