---
title: Cheat Classification
published: true
---

There are generally two types of cheats that are developed. The so called `external` and `internal` cheats. The difference is as follows in short:

> External Cheats use operating system calls like `WriteProcessMemory` (WPM) and `ReadProcessMemory` (RPM) to interact with the state of program to be hacked. Kernel mode anti-cheat software can easily block external hacks by using the kernel function `ObRegisterCallbacks` in `NtosKrnl.lib` to block handle creation (https://douggemhax.wordpress.com/2015/05/27/obregistercallbacks-and-countermeasures/).
    Real life examples include the use in BattleEye and EasyAntiCheat as well as Vanguard. 
> 


## Circumventing ObRegisterCallbacks

The callbacks are stored in kernel object and there 