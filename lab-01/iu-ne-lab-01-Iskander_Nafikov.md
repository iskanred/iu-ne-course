* **Name**: Iskander Nafikov
* **E-mail**: i.nafikov@innopolis.university
* **Username**: [iskanred](https://github.com/iskanred)
* **Hostname**: `lenovo`
---
# Overview
> In this lab, you will set up your initial basic network for INR course. Then you will make a small switched network, get familiar with subnets and configure IPs and then test the connectivity between your machines. Once you have the network running, you will configure a router as a gateway.

# Task 1 - Tools
> To make it easier to prototype network topologies and troubleshoot them, you will be using GNS3 software to emulate networks.

> 1. Install the needed dependencies for GNS3: QEMU/KVM, Docker and Wireshark.

I have installed the necessary software (**Docker** and **QEMU** for virtualisation and **WireShark** for traffic sniffing):
* Here I successfully started `gns3server`
	![[Pasted image 20241028210803.png]]
* Here I successfully started `gns3` client
	![[Pasted image 20241028210930.png]]

> 2. Start a GNS3 project, configure the pre-installed Ubuntu Cloud Guest template. Check that you can start it.

* I successfully created a project:
	![[Pasted image 20241028211228.png]]
* Here is the `telnet` connection to the Web machine (Ubuntu Cloud Guest) console:
	![[Pasted image 20241028211318.png]]

> 3. What are the different ways you can configure internet access in GNS3?

I found three convenient ways to configure internet access in GNS3 for a node:
1. Using a "Router" node that can represent itself a gateway to the internet,  a "Cloud" node actually.
	![[Pasted image 20241028212323.png]]
2. Using a "NAT" node that can also represent itself a gateway to the internet. By default, it runs a DHCP server with a predefined pool in the 192.168.122.0/24 range.
	![[Pasted image 20241028212243.png]]
3. Another method is to create your own gateway node using some PC instead of a simple router. The usage is the same as for a router
	![[Pasted image 20241028212414.png]]

Overall, the first and the last approaches are similar, and the first one is easier and more convenient to use.
The first and the second ones differ. NAT is easier, but is is limited to some network with only 256 available nodes (192.168.122.0/24). A router is more flexible but more difficult approach.
# Task 2 - Switching
> 1. Make the following network topology:

* I made up the following topology where Web is the Ubuntu Cloud Guest node and the Admin is the Ubuntu Docker Guest node
	![[Pasted image 20241028231205.png]]
* For example, here is the Web node's console:
	![[Pasted image 20241028231644.png]]
* To work it well I configured a subnet `192.168.122.0/28` and set the static IP addresses (`192.168.122.2` and `192.168.122.3`) for both PC nodes and set `192.168.122.1` as the default gateway which is represented by NAT node.
	- First, I configured the local network for the Web node with the help of `netplan` inside the `etc/netplan/50-cloud-init.yaml:
		![[Pasted image 20241028231946.png]]
	- Then I did similar for the Admin node inside its config in GNS since it is a Docker node:
		![[Pasted image 20241028232111.png]]

> 2. Install openssh-server on both VMs and nginx web server on the Web VM.
* Here the example of output that proves `nginx` and `openssh-server` are installed:
	![[Pasted image 20241028232702.png]]

> 3. What is the IP of the mask corresponding to `/28`?

The subnet mask is `255.255.255.240` or in the binary form `11111111.11111111.11111111.11110000`.
We see that there are only 4 free bits at the end of the mask => we can assign only $2^4$ addresses in our subnet = $16$.
What is more, two IP addresses are reserved for broadcast (`255.255.255.255`) and network  address `255.255.255.240` => only $14$ IP-addresses can be actually assigned to some hosts in our local network.

> 4. Configure the VMs with private static IPs under a `/28` subnet.

Demonstrated in the 1st clause

> 5. Check that you have connectivity between them.

* Here is the the example of ssh connection from `Admin` node to the `Web` node:
	![[Pasted image 20241028233809.png]]
* Here is the example of pinging the `Admin` node from the `Web` node:
	![[Pasted image 20241028233823.png]]

> 6. Make sure your web server is accessible from the Admin VM.

![[Pasted image 20241028234209.png]]
# Task 3 - Routing
> Now that you have a small network running, it is time to have it properly routed. Delete the NAT device because you will make your own gateway now.

> 1. Select a virtual Routing solution (Gateway) such as Mikrotik (recommended default choice), PfSense, VyOS, Untangle NG, OpenWrt, Cumulus VX.

I selected MikrotikCCR1038-8G-2S+7.8-1
> 2. Create Internal network for Worker instance

Created:
	![[Pasted image 20241029015222.png]]

> 3. Connect your Gateway to the internet and to your workstation/host

This task is failed because I have only WiFi adapter since my laptop has no right Ethernet adapter :(

With WiFi I couldn't connect to the internet

> 4. Setup the gateway for Admin, Web and Worker, then check their connectivity.

However, I were able to configure the gateway for every end device node:
* The router configuration:
	![[Pasted image 20241029015626.png]]
* Web
	![[Pasted image 20241029015649.png]]
* Admin
	![[Pasted image 20241029015638.png]]
* Worker (its ip is `10.1.1.2/24`)
	![[Pasted image 20241029015657.png]]


> 5. Configure port forwarding for http and ssh to Web and Admin respectively.



> 6. Check that you can ssh to the Admin and access your web page from your workstation/host.

