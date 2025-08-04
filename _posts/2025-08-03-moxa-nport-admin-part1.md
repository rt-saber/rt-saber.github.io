---
title: "Reversing Moxa NPort Administrator | Part 1: Packets structure"
date: 2025-08-04
categories: [re, networking, moxa]
tags: [re, networking, moxa, nport]
author: saber
description: Deep dive into the protocol used by Moxa NPort Administrator
---

# What even is NPort Administrator?

NPort Administrator is a software developed by Moxa to remotely manage NPort serial device servers.
It lets you either specify an IP of a NPort server or search for such devices in your network (broadcast search).

Once a device is identified it retrieves some very basic information about it (model, MAC address, IP address, Server name and Status).

As explained in the documentation, status can be one of the following:
- Lock: The device is password protected and was found with a broadcast search, we did not specify the password to unlock the device yet.
- Unlock: The device is password protected and was found with a broadcast search, we already specified the password to unlock device.
- Blank: The device is not password protected and was found with a broadcast search.
- Fixed: The device is not password protected and was found with its IP address.
- Lock fixed: The device is password protected, was found with its IP address and we did not unlock it using its password  yet.
- Unlock fixed: The device is password protected, was found with its IP address and we already specified the password to unlock the device.

Once a device is unlocked we can do few things such as uploading a new firmware or modifying and downloading its configuration. 

# How is NPort Administrator communicating with NPort devices?

The NPort Administrator suite communicates with NPort devices on port 4800 using UDP. It uses what seems to be a custom protocol made by Moxa. I wanted to dig more into this protocol to understand it and potentially find security flaws. This is what led to this series of posts.

# How does NPort packets looks like?

Please be aware that I am digging more in this proprietary code and protocol I may modify this entry to add precisions or even rectify myself. This is undocumented protocol, so while I'm trying to get things as right as possible I may be totally wrong on some aspect as I can only guess what something specific is doing.

The first byte of the packet represents a message type. For example you may be familiar with the different IRC commands (PRIVMSG, QUIT, JOIN, PASS. PART etc.). this work the exact same way. except commands doesn't have a nice name, only a number.

Take the following packet data in hexadecimal representation:

`0100000800000000`

The first byte, `01`/`0x1`, is equal to 1, this mean this is command n°1.

Second to fourth bytes represents the packet length. In the previous packet these bytes are `000008`, so 8 basically, it correctly represents the packet length.

```python
>>> packet = "0100000800000000"
>>> int(len(packet) / 2)
8
>>>
```

This packet is always sent by the NPort Administrator client when trying trying to connect to a server. For this reason I call it the `INIT` packet.

However, this is a very simple packet and doesn't show much so I'm going to take a few more examples:

`81000018000000000054008004540090e858b597c0a8bec8`

Here, the first byte is `81`/`0x81`, this is worth noting: in this protocol the message type sent back from the server is always directly related to the message type sent by the client.

What I mean is that in the first packet the message type was `0x1`, the message type from the response packet is `0x81`, a difference of `0x80` (`0x1` + `0x80` = `0x81`), this behavior remains the same for all packet. Say the client send a packet with message type `0x23`, the server will respond with a packet containing a message type `0xa3`/`163` since `0x23` + `0x80` is equal to `0xa3`.

Second to fourth bytes here is `000018`, or `0x18` this mean the packet length is 24 bytes.

The purpose of the next 4 bytes (`00000000`) is unknown at this stage, this could be some padding or token.

The next 6 bytes (`0054008004540090`) is most likely indicating device type and version, this packet was sent from NPort 5450 device.

The next 6 bytes (`0090e858b597`) represents the device's MAC address.

The last 4 bytes (`c0a8bec8`) represents an IP address.

```python
>>> f"{0xc0}.{0xa8}.{0xbe}.{0xc8}"
'192.168.190.200'
>>>
```

Last example:
`9000003c000000000054008004540090e858b5974e50353435305f393333350000000000000000000000000000000000000000000000000000000000`

This is a packet sent by the server to the client. We can deduce thanks to the message type `0x90` that this is a response to a packet sent from the client that had message type `0x10` or `16` (since `0x90` - `0x80` is equal to 16/`0x10`).

The packet length is `0x00003c` or 60.

Bytes 16 to 21 is the MAC address.

Bytes 22 to 31 is interesting, this is ASCII:
```python
>>> hex = "4e50353435305f39333335"
>>> bytes.fromhex(hex).decode()
'NP5450_9335'
>>>
```

This is the "Server name" that gets displayed in the NPort client.
![First](/assets/images/nport/1.png)


Of course, the packets layout will change A LOT depending on the initial message type, what I explained above is only true for these specific message type, I explained it anyway because it will follow a similar pattern in many case (message type and length will ALWAYS be there).

--- 

#### References
[1#] https://www.moxa.com/Moxa/media/PDIM/S100000213/moxa-nport-ia5000-series-manual-v4.0.pdf