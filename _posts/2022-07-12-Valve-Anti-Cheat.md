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

# 0 List of Tools

Majority of games do not have source code available. In order to understand games internals and stored gameplay relevant data, a cheat developer needs to reverse engineer location and structures of such data. Most common tools are IDA, CheatEngine, x64dbg, ReClass and Ghidra. 


# [](#header-1) 1 Architecture

After reading up on work done by good game hackers like [r0da](https://whereisr0da.github.io/blog/posts/2021-03-10-quick-vac/) and now Valve Employee Tomas Curda [[1](https://is.muni.cz/th/qe1y3/bk.pdf)] we can narrow our hunt down to a couple of files.

## Steam Client

The Steam Client occassionally tries to load the `winhttp.dll` into memory for ?? reason, when downloading a new game.

## VAC 2 

VAC is used in two versions VAC2 and VAC3. The older VAC2 version can be found inside steams resource folder and is called `sourceinit.dat`. It likely hasn't been updated in a while since the compile stamp of the latest version is `Wed Nov 03 16:01:46 2021`.   
The DLL is loaded once a game is ran by a function inside `steamclient.dll` that copies the DLL into system temp directory and loads it with `LoadLibary`.

There are mutliple disadvantages with this approach:

- Library is part of steam, a steam client update is required to update anti-cheat
- Whole Anti-cheat code is stored in users's machine and is therefore easy to analyze circumvent
- Trivial to detect anti-cheat updates (by using checksums)

## VAC 3 

For Valve Anti Cheat 3 we can narrow it down to `steamservice.dll` which could be used in `SteamService.exe` or in `steam.exe`. If Steam is executed with administrator rights `steamservice.exe` gets executed. VAC3 solves the problems that VAC2 had namely that VAC isn't part of the Steam client anymore and can be updated anytime or even multiple libraries can be loaded at he same time. The cheat developer never has full access to the whole anti-cheat code at once. 

The structure of the library is similar between VAC2 an VAC3 with the main difference being that instead of holding the whole anti cheat VAC 3 usually only contains one or two scan functions at the same time. 

Valve splits the functionality of VAC3 into several modules. Those get streamed onto your computer while playing a VAC protected game. For this procedure they load `steamclient.dll` either into their `steamservice.exe` or if you start Steam with Admin privileges, into `steam.exe`. The `steamclient.dll` is not the UI but more like the backend for steam. It's the core engine of the Steam Client. 

### When VAC is active



### [](#header-2)steamservice.dll

The `steamservice.dll` found in `bin/` is a 32bit executable that holds the ValveAntiCheat core module. 

Interestingly metadata also hasn't been removed from the file, which leaks some of Valves internal build architecture. Following was leaked by `exiftool`: Full build address of the internal builder `buildbot_steam-relclient-win32-builder_steam_rel_client_win32@steam-relclient-win32-builder`.
From the pdb information we also get that Steam was build with a Windows PC from pdb path: 
`c:\buildslave\steam_rel_client_win32\build\bin\ClientRelease\bin\steamservice.pdb`. 

### [](#hooking_into_vac)How VAC is Loading Libraries

With the information provided by game hackers one could guess that Valve is somehow loading VAC modules during runtime with LoadLibrary or just manually map them to their process. Targets of interest would be calls like `LoadLibrary`, `LoadLibraryEx`, `CreateFileW`, `WriteFile` and `NtAllocateVirtualMemory`. 

Hooking functions basically means that the game or program "detours" the flow of execution to a region of code that is controlled by the attacker. 
We detour `steamservice.dll` to take us to a code region and examine how the software loads modules on the fly.

While VAC2 is loading libraries to a temp directory %temp%, VAC3 does not load libraries like that instead they get downloaded on the fly and then executed.

## [](#encryption)Encryption and Hashing

VAC uses several encryption / hashing methods:

- MD5 - hashing data read from process memory
- ICE - decryption of imported functions names and encryption of scan results
- CRC32 - hashing table of WinAPI functions addresses
- Xor - encryption of function names on stack, e.g NtQuerySystemInformation. Strings are xor-ed with ^ or > or & char.

However Strings are not encrypted in the `steamservice.dll` For example we get Strings that look like `.rdata:1024D2E0	000000CC	C	\nHey, you just set %s's number values to be 0.  Is that what you wanted?\n\nIf you do `%s = 123`, you will set the convar to `= 123` which becomes `0` for numeric uses.\nDid you mean to do `%s %s` instead?\n` which is some kind of internal debug conversation maybe? 

# Dumping Steam Modules

The `SteamService.exe` manually maps the VAC modules, but we can force Steam to use `LoadLibrary` by byte patching the following instruction:

```
.text:1002A84C jz short loc_1002A895 -> jmp short loc_1002A895
```

# [](#modules) 2 List of Modules August 2922

Following is a compiled list of modules that get loaded for VAC after being started. 

I recently (a few nights ago or something) finished reversing my 7th VAC module out of the total of 14 which appear to be active at the moment, so I figured it would be a good time to give an overview of what I’ve found so far. Please note that I will only be giving a very general/vague overview of what the modules do, and I won’t be posting a detailed analysis at this time because quite frankly there are so many incompetent cheat sellers who are either too stupid or too lazy to reverse VAC and I don’t want to do all the hard work for them; they deserve the ban-waves they get.

There are intentional omissions made in the descriptions given below (nothing fundamental, just some of the specifics). If you’re interested in more accurate/detailed information about the functionality of a specific module then you’re welcome to contact me via the usual channels (and convince me that you’re not a retarded cheat seller who wants their work done for them for free).

I have identified module familes via their timestamp (from the export directory of the PE header) because it’s more convenient than using a hash. I have confirmed that none of the timestamps of any modules in my (reasonably large – 30MB) collection are spoofed, so it should be okay to rely on them for the purpose of taxonomy (though you should have a backup safety check like me to detect otherwise, because the timestamps could easily be spoofed in order to confuse researchers).

20130330-000323
Uses SNMP to get the IP and MAC of the first hop (see ipRouteNextHop) and returns an MD5 hash of the two combined, plus the plaintext OUI (first 3 octets) of the MAC.

20140402-164615
Enumerates windows looking for those owned by a PID specified in the parameters and reports back characteristics such as style, status, borders, etc. Then does the same thing for child windows, and also checks the window text against a list of MD5 hashes.

20140422-222258: Enumerates the NT object manager looking for mutexes of the form “steam_singleton_mutext_%08x_%08x”. For each match it parses out the app ID and process ID, reads the environment block from the remote process, and looks for an environment variable of the form “STEAMID=%llu”. It reports back an array of AppId/ProcessId/SteamId.

20140814-165711
Enumerates all USB devices in two passes (first currently present devices, then disconnected devices) and reports back an array of VID/PID for each entry.

20141007-225725
Gathers system configuration info with a higher than normal interest in recording specific API error codes. Reports back things including (but not limited to) CI options, KD info, boot time, number of disks, boot identifier, and syscall ids of some NT APIs which are retrieved by manually parsing them out of NTDLL loaded off the disk. Interestingly, this module is always loaded at the startup of CSGO, even if “-insecure” is specified on the command line (“+sv_lan 1” also appears to have no effect, even in combination with “-insecure”).

20150427-175232
Gathers BCD store configuration info about a boot identifier provided in the parameters. Reports back things including (but not limited to) kernel path, HAL path, load options, NX policy, and test signing. Much like the 20141007-225725 module, this one also has a higher than normal interest in API error codes.

20150501-213622
Enumerates the USN journal looking for files matching certain criteria. In the general case it MD5 hashes both the name and the USN reason (masked with a constant) together and compares against hardcoded digests, and reports back the file reference number, record timestamp, and MD5 digest. If the file matches specific criteria (a subset of the aforementioned criteria used for the general case) then further processing is done in the form of recursively enumerating files/directories on the system drive looking for related files and reporting back the MD5 digest of a portion of one of the related files if such a file is found.

EDIT (20150805-0247):

I meant to mention this earlier but forgot (although I did blog about it, so if you follow my other posts you would already know). 20150501-213622 has been updated/replaced with 20150707-205524, but nothing fundamental changed (just checking for some extra hashes, some of which I’ve been able to reverse, others which I’m yet to identify).

VAC Module Overview (2/2) - August 2022

The time has finally arrived to round out my VAC3 module overview. Same conditions as before still apply.
I’ve been a bit lazier with this one than in previous installments because I’ve been so busy recently and I haven’t really had any good chunks of time on the weekend to work on this. I haven’t finished reversing every single corner case of the modules before posting like I did for the last two posts, however given that this is only an overview anyway it doesn’t matter. So, my apologies if this post reads like it was rushed, but that’s because it was.

20150326-201829
Three scans. The first simply checks whether a file specified in the scan parameters exists and is accessible. The second retrieves events from the Steam service process monitor (which uses ETW to track process starts/stops). The third retrieves information on a specific PID from the process monitor (it gets the path, which is then used to look up the VSN and file index), along with some PE header metadata of steamservice.dll, and the full data and vtable of the process monitor provider. I may go into further detail about the Steam service process monitor component in the future, but in the meantime curious parties can get started by looking for the string “Steam_{E9FD3C51-9B58-4DA0-962C-734882B19273}_Pid:%000008X” in steamservice.dll.

20150422-164633
Two scans. The first returns information on processes which have an open handle to a process (i.e. the game). This is similar in concept to the system profiler component which the 20140422-222301 module returns the data for, however it is different in implementation. Along with the actual data being returned being different, the key differences are the fact that it utilizes manual syscalls (including using native x64 code on x64 processors instead of using the OS thunking), presumably to bypass usermode API hooks, and also that it runs in Steam instead of the game which means that in most cases it is running at a higher privilege level. The second scan appears to be detecting modified cvars for the purposes of cheating. Interestingly, while this scan appears to be specific to Dota 2 (it’s the only game I found with an engine.dll which contained the necessary code for the scan to actually work) I was unable to trigger it even during a Dota 2 game, so it may be inactive at the moment.

20150609-214236
Four scans. The first returns information about the modules loaded in a process (path, name, flags indicating various properties of the file, vsn and file index, hash, base address, name and path hash, mapping size, etc.). The second returns information about a specific module or file (or memory region in the case of a manually mapped module), including path, various PE file characteristics (headers, timestamp, checksum, pdb path, icon hash, section hashes, etc), VSN and file index, etc. It also returns strings found in the file/region, and finally it also does some “sig scanning” via function and/or string hashing and flags any matches, as well as containing some additional even more targeted checks for specific cheats. This is definitely the scan to look into if you want to know how VAC operates, because my guess is that this is how they catch the majority of cheats. Please note that I’ve been intentionally vague about the details of this particular scan so as not to allow my work to be leveraged by cheaters without them doing their own research, contact me directly if you have a question or want more details and can prove to me you’re not just after copy-pasta for your payhack. Scan three is very similar to scan one, however instead of returning information about modules it returns information about all the currently running processes. It appears to be inactive at the moment though because I have never seen it called and can’t think of any reasonable way to trigger it. Scan four is one I am not yet sure about. I’ve never seen it called, and reversing it statically is not making a whole lot of sense as it is doing a bunch of PE file format manipulations that don’t appear to be generally applicable, so my current suspicion is that it is targeting a specific cheat, but I still need to look deeper to confirm this.

What’s next? Finishing reversing the final corner cases and minor details/nuances of the modules so I can round off my private notes and counterpart test implementations (I like to re-implement all the scans so I can ensure my understanding is correct), especially the ones that I haven’t been able to trigger because those are obviously harder to verify (though I think I can simply spoof input data for all of them). Following that will be another long break (likely a very long one this time because I am in the process of moving overseas), and then probably some further analysis of the ‘supplementary’ VAC components (specifically the process monitor in SteamService, and the stack tracing in GameOverlayRenderer), and then finally an analysis of VAC2. I’ve also become more interested in DRM recently so who knows, maybe I’ll decide to look at CEG instead.


# 3 Sources

[1] bachelor's thesis https://is.muni.cz/th/qe1y3/bk.pdf
[2] anti breakpoint tricks http://waleedassar.blogspot.com/2012/11/defeating-memory-breakpoints.html
https://www.unknowncheats.me/forum/anti-cheat-bypass/354594-analysis-valve-anti-cheat.html
https://whereisr0da.github.io/blog/posts/2021-03-10-quick-vac/
https://madelinemiller.dev/blog/anticheat-an-analysis/
https://www.unknowncheats.me/wiki/Valve_Anti-Cheat:VAC_external_tool_detection_(and_more)
https://www.unknowncheats.me/forum/anti-cheat-bypass/481127-anti-cheats-create-signatures.html
http://lausiv.tech/posts/1
https://github.com/ajkhoury/ReClassEx
https://partner.steamgames.com/doc/sdk/api?
https://github.com/zyhp/vac3_inhibitor
https://www.unknowncheats.me/forum/anti-cheat-bypass/281883-dumping-vac-modules.html
http://dev.cra0kalo.com/?s=dump