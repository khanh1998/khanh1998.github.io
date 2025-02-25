---
date: '2021-12-07T19:57:13+07:00'
draft: false
title: 'What happens when you use a browser to access google.com'
tags: ['networking']
---
So this is just a quick recap of what I have learned about networking

First, you have to connect your computer to the local network. Your computer then will be assigned a local IP by a **DHCP** server (Dynamic Host Configuration Protocol), you cannot use that local IP to talk to the outside network. All the devices in the same local network will use the IP of the gateway to talk to the internet, sometimes it’s called public IP. The protocol to map from local address to public address is called **NAT** (Network Address Translation).

Google.com is a **URL**, the computer doesn’t understand those names, they made those up so the human can remember them easier.

Therefore, the first thing is to convert google.com to an IP address. The computer can do that using **DNS** protocol, each ISP will have a default **DNS** server, or you also can choose the **DNS** server that you want. So now you will have the IP of the server and be ready to make a connection.

Now, the client will make a **TCP** connection to the server using a **“three ways handshake“**. After that, if we use **HTTPS**, then right after TCP connection is established, **TLS/SSL handshake** will happen, the server will send a **TLS/SSL Certification** and it’s the **public key** to the client, the client then verify to see if the certificate is valid or not.

When the **TCP** connection is ready, the client can use **HTTP/HTTPS** protocol over **TCP/IP** protocol to talk to the server to request resources.

HTTP -> TCP -> IP -> Ethernet (PPP) -> Physical layer

TCP is a protocol to exchange messages between two processes, could be in the same or different machine. We usually see TCP come with IP, IP is another protocol to exchange messages between two hosts.

Bellow IP protocol could be **Ethernet**. So the **IP** package is transferred by **Ethernet**, and Ethernet doesn’t use an **IP** address, it uses **MAC** Address instead. So there is an **ARP** (Address Resolution Protocol) to get **MAC** addresses of surrounding hosts.

There is another protocol which is on the same layer (layer 2: data link layer) with Ethernet, it is PPP (point-to-point protocol).
PPP is a dedicated connection between two nodes, unlike Ethernet which is in a multi-access network where multiple devices can share the same medium.
PPP doesn't need MAC address, address field in PPP frame is filled with 0xFF.

The lowest layer is physical layer, but I am not gonna talk about it, because it's out of my knowledge.

|Layer | Layer Name | Protocol |   Unit  | Job | What are in the header?|
|------|------------|----------|---------|-----|---------------------------|
|5     |Application | HTTP     | Request | Specify the resource like image, and files and action to do like GET, PUT,...| HTTP Method, Resource URL|
|4     |  Transport | TCP      | Segment | Split the big file into several packages, and send them to the target. Ensure all packages are arrived in correct order |Port Number, sequence number|
|3     |    Network | IP       | Package | Transmit data from one host to another host, in different network | IP Address |
|2     |  Data Link | Ethernet | Frame   | Transmit data from node to node, in same network | MAC Address |
|1     |   Physical | | | |