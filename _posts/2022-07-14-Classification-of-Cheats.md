---
title: Cheat Classification
published: true
---

# 1 Classification of Cheat Source 

There are generally two types of cheats that are developed. The so called `external` and `internal` cheats. The difference is as follows in short:

- External Cheats use operating system calls like `WriteProcessMemory` (WPM) and `ReadProcessMemory` (RPM) to interact with the state of program to be hacked. Kernel mode anti-cheat software can easily block external hacks by using the kernel function `ObRegisterCallbacks` in `NtosKrnl.lib` to block handle creation [[1](https://douggemhax.wordpress.com/2015/05/27/obregistercallbacks-and-countermeasures/)].
    Calls to WPM and RPM are slow because you have kernel trap overhead. The frequency of these calls should be limited and as much information as possible should be stored locally in the client. 
    Real life examples include the use in BattleEye and EasyAntiCheat as well as Vanguard. 
    + Quick and easy way to patch some bytes
    - Easy to detect because of opnen process handles
    - Hard to use but easy to master because it has no skill cap
    - Slow
- Internal hacks are made by injecting `.dll` with various techniques. It gives direct access to the games memory and hacks can be written cleaner and more easy to read. When developing internal cheats you can typecast pointers to memory and interact with in game objects from code.  The dlls are typically injected by the use of so called 'dll injection'. There are several methods how code can be injected which will be covered in another page [[2](https://is.muni.cz/th/qe1y3/bk.pdf)]. 

So the main distinction between external and internal cheat is whether the cheat is running inside the game's address space or not. The external cheat needs to keep running while the injected `.dll` executes in the games address space usually. 

# 2 Classification of Cheat Privilege Level

Cheats can run under different privilege levels in Windows. There is a difference between a cheat running in user or kernel mode with the main differences being as follows

- Kernel-Mode-Cheats execute in Kernel Address Space and so code and data are not accessible from programs running in user mode. 
- Cheat has full access to computers memory. 
- The kernel mode cheat can change the behavior of Window API functions, preventing attempts of detecting the cheat from user mode. 

# 3 Difference between Cheat and Exploit

A cheat intentionally modifies game behavior and encompasses exploits that do the same but maybe lead to unintended game states. For example an exploit would bring the game in a state that is undefined and thus lead to interesting results. 

# 4 Availability of Cheats

Cheats can come from different sources. Depending on the way the user obtained the cheat a couple of categories exist: 

- Public Cheats: Available on public websites as compiled binaries or ready to compile code or code examples. http://unknowncheats.me, http://cheathappens.com, http://mpgh.net
- Private Cheats: Developed by an individual or a company and made readily available with some kind or DRM or over the counter as is. 

# 4 Mitigations

## Circumventing ObRegisterCallbacks

Callbacks used to block handle creation can be circumvented [1](https://douggemhax.wordpress.com/2015/05/27/obregistercallbacks-and-countermeasures/). Callbacks are stored in kernel objects and can be found by iterating over the linked list that `ObRegisterCallbacks` returns. 

A problem arises when the anti-virus or anti-cheat software tries to re-register its callback and check for return value. For example if the re-registration succeeds, it can be an indicator that the callback had been removed. Otherwise if the status is `STATUS_FLT_INSTANCE_ALTITUDE_COLLISION` it is a good indicator that the callback was still in place.

A solution to this is to register your own callback at the same so called `altitude` (indicates the order in which callbacks are called) as the system callback to hide check via re-registering the old callback. 

# 5 Sources 

[1] https://douggemhax.wordpress.com/2015/05/27/obregistercallbacks-and-countermeasures/
[2] https://is.muni.cz/th/qe1y3/bk.pdf