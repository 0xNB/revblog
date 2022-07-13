---
title: Valve Anti Cheat Reverse Engineered 
published: true
---

Valve Anti Cheat is an userland Anti Cheat made to scan and detect external cheats (other processes) or internal cheats (in game processes). 

As it is a userland level anticheat tool that means that it usually runs within the process that its meant to protect. It also can only monitor other user-land software whilst kernel level cheats can not be detected from user land. 

VAC-Anti-Cheat thus can not detect hardware cheats (unless they are visible from userland) or kernel-land cheats (only enumerate list of drivers, check names and signatures and lower the trust factor sent to Valve if you allow unsigned drivers).

There are two different parts of VAC to consider:

- VAC the Anti-Cheat made by Valve for a lot of different games
- Custom VAC modules that can be applied for specific games like Call of Duty or CSGO. 

Both are delivered as modules (executables), each time a VAC protected game is launched. They are downloaded and executed to check for specific "cheat" patterns.

NOTE: Some games themselves include Anti-Cheat behaviors like CSGO, however those Anti-Cheat behaviours are developed by the same people as VAC so considered also as a part of Valve-Anti-Cheat. 


# [](#header-1)Architecture

After reading up on work done by good game hackers like [https://whereisr0da.github.io/blog/posts/2021-03-10-quick-vac/](r0da) we can narrow our hunt for Valve Anti Cheat down to `steamservice.dll` which could be used in `SteamService.exe` or in `steam.exe`, if Steam is executed with administrator rights. 

Valve splits the functionality of VAC3 into several modules. Those get streamed onto your computer while playing a VAC protected game. For this procedure they load steamclient.dll either into their steamservice.exe or if you start Steam with Admin privileges, into steam.exe.

## [](#header-2)steamservice.dll

The `steamservice.dll` found in `bin/` is a 32bit executable that holds the ValveAntiCheat core module. 

Interestingly metadata also hasn't been removed from the file, which leaks some of Valves internal build architecture. Following was leaked by `exiftool`: Full build address of the internal builder `buildbot_steam-relclient-win32-builder_steam_rel_client_win32@steam-relclient-win32-builder`.
From the pdb information we also get that Steam was build with a Windows PC from pdb path: 
`c:\buildslave\steam_rel_client_win32\build\bin\ClientRelease\bin\steamservice.pdb`. 

### [](#hooking_into_vac)How VAC is Loading Libraries

With the information provided by game hackers one could guess that Valve is somehow loading VAC modules during runtime with LoadLibrary or just manually map them to their process. Targets of interest would be calls like `LoadLibrary`, `LoadLibraryEx`, `CreateFileW`, `WriteFile` and `NtAllocateVirtualMemory`. 

Hooking functions basically means that the game or program "detours" the flow of execution to a region of code that is controlled by the attacker. 

## [](#encryption)Encryption and Hashing

VAC uses several encryption / hashing methods:

- MD5 - hashing data read from process memory
- ICE - decryption of imported functions names and encryption of scan results
- CRC32 - hashing table of WinAPI functions addresses
- Xor - encryption of function names on stack, e.g NtQuerySystemInformation. Strings are xor-ed with ^ or > or & char.

However Strings are not encrypted in the `steamservice.dll` For example we get Strings that look like `.rdata:1024D2E0	000000CC	C	\nHey, you just set %s's number values to be 0.  Is that what you wanted?\n\nIf you do `%s = 123`, you will set the convar to `= 123` which becomes `0` for numeric uses.\nDid you mean to do `%s %s` instead?\n` which is some kind of internal debug conversation maybe? 
