---
title: Serving from WSL2 Network to the Windows Host
published: true
---

# [](#header-1)Connecting to the VM

Host the service on the VM to listen on all addresses like `--host 0.0.0.0`. Connect to the Linux Server by using the local loopback address on windows which is also part of the IP which gets listened on.


Otherwise if its hosted on the loopback address `127.0.0.1` you can connect to it with the address you get in `ip addr` with the tags `<BROADCAST,MULTICAST,UP>` 



