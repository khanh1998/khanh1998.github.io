---
date: '2024-11-09T19:57:13+07:00'
draft: true
title: 'What happens when you use a browser to access google.com'
tags: ['networking']
---
So this is just a quick recap of what I have learned about networking

First, you have to connect your computer to the local network. Your computer then will be assigned a local IP by a **DHCP** server (Dynamic Host Configuration Protocol), you cannot use that local IP to talk to the outside network. All the devices in the same local network will use the IP of the gateway to talk to the internet, sometimes it’s called public IP. The protocol to map from local address to public address is called **NAT** (Network Address Translation).

Google.com is a **URL**, the computer doesn’t understand those names, they made those up so the human can remember them easier.

Therefore, the first thing is to convert google.com to an IP address. The computer can do that using **DNS** protocol, each ISP will have a default **DNS** server, or you also can choose the **DNS** server that you want. So now you will have the IP of the server and be ready to make a connection.

Now, the client will make a **TCP** connection to the server using a **“three ways handshake“**. After that, if we use **HTTPS**, then right after TCP connection is established, **TLS/SSL handshake** will happen, the server will send a **TLS/SSL Certification** and it’s the **public key** to the client, the client then verify to see if the certificate is valid or not.

When the **TCP** connection is ready, the client can use **HTTP/HTTPS** protocol over **TCP/IP** protocol to talk to the server to request resources.

HTTP -> TCP -> IP -> PPP

TCP is a protocol to exchange messages between two processes, could be in the same or different machine. We usually see TCP come with IP, IP is another protocol to exchange messages between two hosts.

Bellow IP protocol could be **PPP** (Point to Point Protocol). So the **IP** package is transferred by **PPP**, and PPP doesn’t use an **IP** address, it uses **MAC** Address instead. So there is an **ARP** (Address Resolution Protocol) to get **MAC** addresses of surrounding hosts.