---
title: x86 API Hooking on Windows
published: true
---

Function or API hooking allows reverse engineers or hackers to tamper with flow of execution during runtime. For example if we want to examine what a function does during execution we can hook it, print the arguments and then keep executing the original function. Likewise hackers might be interested in inserting malicious code in the hooked function calls and then execute them during runtime.

This blog entry will cover most basic function hooking methods and deep dive into basic implemenation examples.

https://guidedhacking.com/threads/how-to-hook-functions-code-detouring-guide.14185/
http://jbremer.org/x86-api-hooking-demystified/
https://en.wikipedia.org/wiki/Hooking#Source_modification

# Theoretical Background

Hooking Windows native functions is pretty straight forward. Plenty of native Windows `.dll`s have been compiled with the hotpatching flag. 
If a function accepts a so called hotpatch then it has been prepared in a certain way to allow patching and detouring of code to other regions [http://jbremer.org/x86-api-hooking-demystified/](Hotpatching). For example with the Microsoft Visual C++ Compiler the hotpatching feature adds a `mov edi, edi` instruction and five `nop` instructions at the preamble of a function. This allows one to place a short jump (8 bit relative offset) at the function address and normal jump with 32bit relative offset at the place of the nop instructions. https://en.wikipedia.org/wiki/Hooking 

## Source Modification 

Function hooks can be inserted statically or dynamically. For example in static analysis the entry point of a function within a module can be found. This entry point can then be altered to instead dynamically load some other library or module and then have it execute desired methods within that loaded library. 
Another approach is altering the import table of an executable. The table can be modified to load any additional library as well as change what external code is invoked when a function is called by the application. 

## Runtime Modification

During runtime hooks can be placed at the beginning of a function to detour exection flow. Function hooking is implemented by changing the very first bytes of code instructions of the target funtion to jump to an injected code section. 

These tactics employ the same ideas as those of source code modification but instead alter structures located in memory of already running processes. 

## Averting Detection of Function Hooks

Placing a detour at the beginning of the functions makes finding functions hooks pretty easy. Even if it is preceeded by `nops` a detectino routine could look like this for example.

```c
unsigned char *addr = function_A;
while (*addr == 0x90) addr++;
if(*addr == 0xe9) {
    printf("Hook detected in function A.\n");
}
```

To circumvent this problem a hook can also be inserted in the middle of a function. If the anti-cheat one is tryingg to circumvent reads the first bytes of a function, then one could write the detour lower in the assembly, thus creating a so called 'mid function hook'.

## External Detouring / Hooking

Another way of hooking is to inject own code into memory of another process by writing to process memory with `WriteProcessMemory`. However this is slightly more complicated than injecting your `dll` into the process. The written memory needs to be shellcode instead of compiling your code to a native `.dll`. 

## On Stolen Bytes

When hooking into functions its important to execute the overwritten instructions after the hook in order to keep the state of the program. 

## Code Caves

In order to be less noisy when using a detour injected code can be placed in regions of executable memory that have already been allocated but are not used by the process. Protection can be modified with `VirtualProtect` or `VirtualProtectEx`

## Background Information on Linux function hooking

Hooking API-Calls on Linux is significantly easier than hooking API-Calls on Windows. For Linux you can overwrite loaded shared objects by specifying the `LD_PRELOAD` with your own `.so` files. 

# Technical Implementation

After injecting a DLL into the process that should receive the hook we can overwrite the first few bytes of the hooked function with our own function. The code below basically overwrites `MessageBoxW` from the `user32.dll` with a short `jmp` to our function by calculating the offset during runtime.

## Splicing

```cpp
/*
 This idea is based on chrom-lib approach, Distributed under GNU LGPL License.
 Source chrom-lib: https://github.com/linuxexp/chrom-lib
 Copyright (C) 2011  Raja Jamwal
*/
#include <windows.h>  
#define SIZE 6

 typedef int (WINAPI *pMessageBoxW)(HWND, LPCWSTR, LPCWSTR, UINT);  // Messagebox prototype
 int WINAPI MyMessageBoxW(HWND, LPCWSTR, LPCWSTR, UINT);            // Our detour

 void BeginRedirect(LPVOID);                                        
 pMessageBoxW pOrigMBAddress = NULL;                                // address of original
 BYTE oldBytes[SIZE] = {0};                                         // backup
 BYTE JMP[SIZE] = {0};                                              // 6 byte JMP instruction
 DWORD oldProtect, myProtect = PAGE_EXECUTE_READWRITE;

 INT APIENTRY DllMain(HMODULE hDLL, DWORD Reason, LPVOID Reserved)  
 {  
   switch (Reason)  
   {  
   case DLL_PROCESS_ATTACH:                                        // if attached
     pOrigMBAddress = (pMessageBoxW)                      
       GetProcAddress(GetModuleHandleA("user32.dll"),              // get address of original 
               "MessageBoxW");  
     if (pOrigMBAddress != NULL)  
       BeginRedirect(MyMessageBoxW);                               // start detouring
     break;

   case DLL_PROCESS_DETACH:  
     VirtualProtect((LPVOID)pOrigMBAddress, SIZE, myProtect, &oldProtect);   // assign read write protection
     memcpy(pOrigMBAddress, oldBytes, SIZE);                                 // restore backup
     VirtualProtect((LPVOID)pOrigMBAddress, SIZE, oldProtect, &myProtect);   // reset protection

   case DLL_THREAD_ATTACH:  
   case DLL_THREAD_DETACH:  
     break;  
   }  
   return TRUE;  
 }

 void BeginRedirect(LPVOID newFunction)  
 {  
   BYTE tempJMP[SIZE] = {0xE9, 0x90, 0x90, 0x90, 0x90, 0xC3};              // 0xE9 = JMP 0x90 = NOP 0xC3 = RET
   memcpy(JMP, tempJMP, SIZE);                                             // store jmp instruction to JMP
   DWORD JMPSize = ((DWORD)newFunction - (DWORD)pOrigMBAddress - 5);       // calculate jump distance
   VirtualProtect((LPVOID)pOrigMBAddress, SIZE,                            // assign read write protection
           PAGE_EXECUTE_READWRITE, &oldProtect);  
   memcpy(oldBytes, pOrigMBAddress, SIZE);                                 // make backup
   memcpy(&JMP[1], &JMPSize, 4);                                           // fill the nop's with the jump distance (JMP,distance(4bytes),RET)
   memcpy(pOrigMBAddress, JMP, SIZE);                                      // set jump instruction at the beginning of the original function
   VirtualProtect((LPVOID)pOrigMBAddress, SIZE, oldProtect, &myProtect);   // reset protection
 }

 int WINAPI MyMessageBoxW(HWND hWnd, LPCWSTR lpText, LPCWSTR lpCaption, UINT uiType)  
 {  
   VirtualProtect((LPVOID)pOrigMBAddress, SIZE, myProtect, &oldProtect);   // assign read write protection
   memcpy(pOrigMBAddress, oldBytes, SIZE);                                 // restore backup
   int retValue = MessageBoxW(hWnd, lpText, lpCaption, uiType);            // get return value of original function
   memcpy(pOrigMBAddress, JMP, SIZE);                                      // set the jump instruction again
   VirtualProtect((LPVOID)pOrigMBAddress, SIZE, oldProtect, &myProtect);   // reset protection
   return retValue;                                                        // return original return value
 }
```

Splicing with a 32bit detour function

```cpp
bool Detour32(void * src, void * dst, int len)
{
    if (len < 5) return false;

    DWORD curProtection;
    VirtualProtect(src, len, PAGE_EXECUTE_READWRITE, &curProtection);

    memset(src, 0x90, len);

    uintptr_t relativeAddress = ((uintptr_t)dst - (uintptr_t)src) - 5;

    *(BYTE*)src = 0xE9;
    *(uintptr_t*)((uintptr_t)src + 1) = relativeAddress;

    DWORD temp;
    VirtualProtect(src, len, curProtection, &temp);

    return true;
}
```

## Trampoline Hooking 

The idea of trampoline hooks is to execute the original function while also inserting own instructions into the assembly. This is usally done by a trampoline function that executes the first few bytes which were overwritten when installing the hook and then execution `function + couple of bytes` to skip the jump.

```cpp
char* TrampHook32(char* src, char* dst, const intptr_t len)
{
    // Make sure the length is greater than 5
    if (len < 5) return 0;

    // Create the gateway (len + 5 for the overwritten bytes + the jmp)
    void* gateway = VirtualAlloc(0, len + 5, MEM_COMMIT | MEM_RESERVE, PAGE_EXECUTE_READWRITE);

    //Write the stolen bytes into the gateway
    memcpy(gateway, src, len);

    // Get the gateway to destination addy
    intptr_t  gatewayRelativeAddr = ((intptr_t)src - (intptr_t)gateway) - 5;

    // Add the jmp opcode to the end of the gateway
    *(char*)((intptr_t)gateway + len) = 0xE9;

    // Add the address to the jmp
    *(intptr_t*)((intptr_t)gateway + len + 1) = gatewayRelativeAddr;

    // Perform the detour
    Detour32(src, dst, len);

    return (char*)gateway;
}
```

## Import Address Table Hooking

A stealthier way of hooking is to overwrite entries in the `import address table`. Splicing can be easily detected by checking the preamble of functions for maliciously inserted code, so anti-virus or anti-cheat software can react by marking the process as maliciously tampered with. 

For example to overwrite the import address table we move from the DOS header to the NT header of a PE executable. After finding the import table we write the address of the imported function with the address of our injeted function. Thus the process executes our function instead of the mapped original function. 

```cpp
#include <windows.h>

typedef int(__stdcall *pMessageBoxA) (HWND hWnd, LPCSTR lpText, LPCSTR lpCaption, UINT uType); //This is the 'type' of the MessageBoxA call.
pMessageBoxA RealMessageBoxA; //This will store a pointer to the original function.

void DetourIATptr(const char* function, void* newfunction, HMODULE module);

int __stdcall NewMessageBoxA(HWND hWnd, LPCSTR lpText, LPCSTR lpCaption, UINT uType) { //Our fake function
    printf("The String Sent to MessageBoxA Was : %s\n", lpText);
    return RealMessageBoxA(hWnd, lpText, lpCaption, uType); //Call the real function
}

int main(int argc, CHAR *argv[]) {
   DetourIATptr("MessageBoxA",(void*)NewMessageBoxA,0); //Hook the function
   MessageBoxA(NULL, "Just A MessageBox", "Just A MessageBox", 0); //Call the function -- this will invoke our fake hook.
   return 0;
}

void **IATfind(const char *function, HMODULE module) { //Find the IAT (Import Address Table) entry specific to the given function.
	int ip = 0;
	if (module == 0)
		module = GetModuleHandle(0);
	PIMAGE_DOS_HEADER pImgDosHeaders = (PIMAGE_DOS_HEADER)module;
	PIMAGE_NT_HEADERS pImgNTHeaders = (PIMAGE_NT_HEADERS)((LPBYTE)pImgDosHeaders + pImgDosHeaders->e_lfanew);
	PIMAGE_IMPORT_DESCRIPTOR pImgImportDesc = (PIMAGE_IMPORT_DESCRIPTOR)((LPBYTE)pImgDosHeaders + pImgNTHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_IMPORT].VirtualAddress);

	if (pImgDosHeaders->e_magic != IMAGE_DOS_SIGNATURE)
		printf("libPE Error : e_magic is no valid DOS signature\n");

	for (IMAGE_IMPORT_DESCRIPTOR *iid = pImgImportDesc; iid->Name != NULL; iid++) {
		for (int funcIdx = 0; *(funcIdx + (LPVOID*)(iid->FirstThunk + (SIZE_T)module)) != NULL; funcIdx++) {
			char *modFuncName = (char*)(*(funcIdx + (SIZE_T*)(iid->OriginalFirstThunk + (SIZE_T)module)) + (SIZE_T)module + 2);
			const uintptr_t nModFuncName = (uintptr_t)modFuncName;
			bool isString = !(nModFuncName & (sizeof(nModFuncName) == 4 ? 0x80000000 : 0x8000000000000000));
			if (isString) {
				if (!_stricmp(function, modFuncName))
					return funcIdx + (LPVOID*)(iid->FirstThunk + (SIZE_T)module);
			}
		}
	}
	return 0;
}

void DetourIATptr(const char *function, void *newfunction, HMODULE module) {
	void **funcptr = IATfind(function, module);
	if (*funcptr == newfunction)
		 return;

	DWORD oldrights, newrights = PAGE_READWRITE;
	//Update the protection to READWRITE
	VirtualProtect(funcptr, sizeof(LPVOID), newrights, &oldrights);

	RealMessageBoxA = (pMessageBoxA)*funcptr; //Some compilers require the cast like "MinGW" not sure about MSVC
	*funcptr = newfunction;

	//Restore the old memory protection flags.
	VirtualProtect(funcptr, sizeof(LPVOID), oldrights, &newrights);
}
```


# TODO:

- Write own hooking library to patch publicly known locations of fucntions or patterns 
- Maybe write other tool that dumps modules