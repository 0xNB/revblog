---
title: DLL Injection Primer
published: true
---

The so called `Dynamic-link libraries` (DLLs) are Microsoft's implementation of a shared library concept in the Microsoft Windows Operating System. The file format of a `.dll` is the same as the one of an `.exe` file, and they can contain code, data and resources. 

The dynamic libriares help promote modularization of code, code reuse, efficient memory usage and reduced disk space. [[1](https://docs.microsoft.com/en-us/troubleshoot/windows-client/deployment/dynamic-link-library)]. For example in Windows operating systems the Comdlg32 DLL performs dialog box related functions. Each program can use the functionality contained in this DLL to implement an Open dialog box. Thus it helps promote code reuse and efficient memory usage. 
Additionally, updates are easier to apply to each module without affecting other parts of the program. For example in a payroll system, the tax rates change each year. When the changes are isolated inside a DLL, you can apply an update without needing to build or install the whole program again. 

DLLs are usually shared among all the processes that use the DLL. DLLs do not use position independent code but undergo relocation as they are loaded, fixing addresses for all the entry points at locations where there is free memory space of the first process that loads the `.dll`. 

In older versions Window all processes occupied a single common address space, thus a `.dll` would only needed to be loaded once. However, in modern versions of Windows a `.dll` might be loaded multiple times if the virtual address range is occupied inside a process that wants to load an already loaded `.dll`.


# Caveats of ASLR on Windows 

Unlike in Linux where ELF images can be position indpeendent executables and position independent code in shared libraries to supply freshly randomized address space for the main program and all its libraries on each launch - sharing the same machine code even when loaded at different addresses. 

Windows ASLR does not work this way. Each DLL or EXE image gets assigned a random load address by the kernel the first time its used, and as additionaly instanes of the DLL or EXE are loaded, they receive the same load address. Only rebooting can guarantee fresh base addresses for all images systemwide. [[2](https://www.mandiant.com/resources/six-facts-about-address-space-layout-randomization-on-windows)]. This can be used for example in attacks that exploit programs that automatically restart after crashing to find out the windows base address. 

This leads to the following observation for Windows: 

**If an attacker can discover where a DLL is loaded in any process, the attacker knows where it is loaded in all processes**

# Run-time dynamic linking

In run-time dynamic linking, an application calls either the `LoadLibrary` function or the `LoadLibraryEx` function to load the DLL at runtime. After the DLL is successfully loaded you can use the `GetProcAddress` function to obtain the address of the exported DLL function that you want to call. 

`LoadLibary` maps a DLL library into the address space of the calling process and returns a handle to the loaded DLL. To remotely execute the `LoadLibrary` function for a victim the `CreateRemoteThread` Windows API function can be used. This is the most straightforward way how a process can force another process to inject a foreign DLL file. 


```cpp
//Open the target process with read , write and execute priviledges
Process = OpenProcess(PROCESS_CREATE_THREAD|PROCESS_QUERY_INFORMATION|PROCESS_VM_READ|PROCESS_VM_WRITE|PROCESS_VM_OPERATION, FALSE, ID); 
//Get the address of LoadLibraryA
LoadLibrary = (LPVOID)GetProcAddress(GetModuleHandle("kernel32.dll"), "LoadLibraryA"); 
// Allocate space in the process for our DLL 
Memory = (LPVOID)VirtualAllocEx(Process, NULL, strlen(dll)+1, MEM_RESERVE | MEM_COMMIT, PAGE_READWRITE); 
// Write the string name of our DLL in the memory allocated 
WriteProcessMemory(Process, (LPVOID)Memory, dll, strlen(dll)+1, NULL); 
// Load our DLL 
CreateRemoteThread(Process, NULL, NULL, (LPTHREAD_START_ROUTINE)LoadLibrary, (LPVOID)Memory, NULL, NULL); 
//Let the program regain control of itself
CloseHandle(Process); 
```


# Sources 
https://en.wikipedia.org/wiki/Dynamic-link_library
[1] https://docs.microsoft.com/en-us/troubleshoot/windows-client/deployment/dynamic-link-library
[2] https://www.mandiant.com/resources/six-facts-about-address-space-layout-randomization-on-windows
[3] https://is.muni.cz/th/qe1y3/bk.pdf