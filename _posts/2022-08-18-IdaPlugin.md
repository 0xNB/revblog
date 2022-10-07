---
title: IDA Plugin For Deobfuscating Public Anti Cheats
published: true
---

Collection of IDA Scripts for various types of obfuscated or anti tampering software to make it more readble.
Those scripts are to be used with IDA + IDAPython, what they do is pretty self explanatory. More precisely the scripts Rebuild/Fix VACs imports, Decrypt encrypted VAC stringtables and strings.

## 1 VAC 

- Strings are encrypted with XOR key sometimes
- Import Table is obfuscated

![IDA installation from WSL](/assets/idaplugin.png)


### 1.1 Find Scangates

```python
# Name: FindScangates.py
# Desc: Finds all main scangates from the call array
# Author: Niklas Betten

from python import idautils
from python import idaapi

start = 0x10001000
end = 0x10060000
		
		
#68 ?? ?? ?? ?? B9 ?? ?? ?? ?? E8 ?? ?? ?? ?? C3   		
initScanGatePat1 = [0x68, 0x00, 0x00, 0x00, 0x00, 0xB9, 0x00, 0x00, 0x00, 0x00, 0xE8, 0x00, 0x00, 0x00, 0x00, 0xC3]
initScanGateMask1 = "x????x????x????x"	
	
def findPattern(current, pat, mask) :
	Index = 0
	for x in pat :
		if mask[Index] == "?" :
			Index += 1
			continue
		if x != Byte(current + Index) :
			return 0
		else :
			Index += 1
	return current
	
# find all scangate funcs
print("\nLooking for scan gates..")
n = start
found = 0
while n < end :
	if findPattern(n, initScanGatePat1, initScanGateMask1) != 0 :
		# 68 08 B0 00 10  push    offset scan_gate1
		# B9 31 C0 00 10  mov     ecx, offset unk_1000C031
		# E8 09 17 00 00  call    init_gate
		gate = Dword(n + 1)
		found += 1
		funcName = "scan_gate_" + str(found)
		print("\nFound gate at 0x%X" % gate)
		MakeName(gate, funcName)
	n += 1
print("\nFound: %i scan gates" % found)
```