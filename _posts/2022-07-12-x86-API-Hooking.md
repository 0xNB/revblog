---
title: x86 API Hooking
published: true
---

Hooking Windows native functions is pretty straight forward. Plenty of native Windows `.dll`s have been compiled with the hotpatching flag. 
If a function accepts a so called hotpatch then it has been prepared in a certain way to allow patching and detouring of code to other regions [http://jbremer.org/x86-api-hooking-demystified/](Hotpatching). For example with the Microsoft Visual C++ Compiler the hotpatching feature adds a `mov edi, edi` instruction and five `nop` instructions at the preamble of a function. This allows one to place a short jump (8 bit relative offset) at the function address and normal jump with 32bit relative offset at the place of the nop instructions. 





