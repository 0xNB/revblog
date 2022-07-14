---
title: Rootkit Detection with GMER
published: true
---

GMER is an application that detects modification and rootkits in software it scans for: 

```
hidden processes
hidden threads
hidden modules
hidden services
hidden files
hidden disk sectors (MBR)
hidden Alternate Data Streams
hidden registry keys
drivers hooking SSDT
drivers hooking IDT
drivers hooking IRP calls
inline hooks
```

GMER checks virtual memory aginst physical memory on disk to detect tampering with files. Ransomware actors also frequently use GMER to find and shut down hidden processes such as antivirus software protecting a server. 