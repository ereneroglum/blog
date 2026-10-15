---
title: 'To Text or not to Text'
date: 2026-10-15T13:00:59+03:00
tags: [ "protocol", "text", "binary" ]
---

I LOVE TEXT AND I HATE STRING PARSING

Yes this is hypocratical. On one side I love that I can netcat to an SMTP
server, complete the authentication and write my message. This requires the
protocol to be constructed such that a human may understand it solely by
looking at the protocol messages. This is why I think the older protocols will
never die. They are simple and can be used with simple programs. I love SMTP,
I love IRC, I love FTP, I love IMAP, I love POP, I love the old HTTP protocols.

On the other side these human readable protocols are a nightmare to code. They
need to handle strings and string parsing is really hard. It is better to have a
defined message structure for developers. Otherwise it becomes a security
nightmare. This message structure ideally should not be variable length. IPv4
made the mistake of including useless options field which makes it a variable
length.

Looking back protocols have evolved from defined message structure, binary
protocols like Ethernet, IP, TCP, UDP, OSPF, BGP, DNS etc. to human readable
protocols (like the ones I have mentioned) and then back into binary protocols
like HTTP/2. I don't see these human readable protocols coming back though. But
I also don't see them dying either.