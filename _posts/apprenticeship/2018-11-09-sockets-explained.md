---
layout: post
title: "Sockets explained"
author: Javier Garcia
description: "A quick and dirty overview of sockets."
category: networks
apprenticeship: true
tags: networks, osi, server, sockets
---

Before diving deep into what sockets are and how they work, I found very enlightening to go way back. Back to the OSI model and its layers.

When a computer communicates with another computer that message usually goes through a set of different layers, all the way from the application layer (i.e. iMessage on OSX), through the transport layer (TCP/UDP), the network layer (IP), the link layer (the routers and switches) and finally the physical layer (cables).

Messages are broken into bytes which the computer understands, and then broken into packets which the OS (Operating System) sends to the router, datagrams which the router sends to another router, and then all the way back to messages which your recipient recieves.

Nonetheless, today we will focus in the intersection between the transport layer and the application layer: *sockets*. 

## What is a socket?

When we build a network application it is crucial to understand how processes on different hosts communicate with each other. For each pair of communicating processes, we label one of them as a client, and the other as the server. Usually, the client is the one that initiates the communication, and the server the one that waits to be contacted. In the web, the client would the the web browser, and the server the web server which serves the web page.

Any message sent from one process to another, must go through the underlying layers. The software interface between these layers is called a socket. Using a metaphor, the process is analogous to a house and a socket to its door. When a process sends a message to another process, it sends it through the door and it must also go into the other process' door.

At the end of the day, **a socket is the interface between the application layer and the transport layer**, also referred to as the API between our application and the network. As developers, we have control over everything that happens before the message is sent through the socket, and everything that happens after the message is recieved from a socket. The only control we have over the transport layer is maybe choosing the protocol (TCP, UDP...) or some parameters.

## How do we address another process?

Just like it happens with houses, in order to send a message from one process to another, you need the address. In the Internet, **a host is identified by it's a IP address**, but that's not enough since any host could be running a large number of processes. A **specific process is addressed by a port number**. So when one process wants to send a message another process it needs both the host's IP address and the port number.

Popular applications already have assigned port numbers; mail processes use port 25, web servers use port 80, etc. so for our applications it's important to check we're not setting those.

## How does listening to a socket work?

To understand how all the writing and reading to and from sockets works, we have to understand the workflow of socket bindings. 

Usually, when a server is created, it creates a new socket and binds it to an IP address and a port number to then start to listen for incoming connections. **When a client connects, a new socket is created in order to communicate with that client**. To determine any activity between the open sockets a polling mechanism is used. 

At some point, once the client is done exchanging messages, it will close its socket, then the server will also close its corresponding socket and continue listening to new clients through the designed socket.

#### References

1. [Computer Networking: A Top-Down Approach by Kurose & Ross](https://www.goodreads.com/book/show/83847.Computer_Networking)
2. [z/OS Communications Server: IP CICS Sockets Guide](https://www.ibm.com/support/knowledgecenter/en/SSLTBW_2.1.0/com.ibm.zos.v2r1.halc001/o4ag1.htm)
3. [TCP tutorial by Shweta Sinha 11/98](http://ssfnet.org/Exchange/tcp/tcpTutorialNotes.html)
5. [Reading from and Writing to a Socket - docs.oracle.com](https://docs.oracle.com/javase/tutorial/networking/sockets/readingWriting.html)