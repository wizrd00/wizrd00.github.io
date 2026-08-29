+++ 
draft = true
date = 2026-08-01
title = "Venux Kernel"
description = "My Minimal Unix-Like Kernel"
slug = "venux"
authors = ["Me"]
tags = ["kernel", "c", "c99"]
categories = ["Projects"]
+++

## UEFI Bootloader
---
#### What a UEFI Bootloader does?
**1**.It tries to find & open the kernel, How?  
It gets information about it's own `Image` with `ImageHandle` the UEFI passes to
`efi_main()` and `BootServices->HandleProtocol()` function.  
In that information, there is pointer called `DeviceHandle` which is the handle
to the device that includes the `Image`.
It uses the `BootServices->HandleProtocol()` function again to get filesystem of
that device and uses filesystem `OpenVolume()` function to open a volume.  
After that, the `Volume->Open()` used to open the kernel file.

**2**.It tries to parse the kernel ELF64  
After some validation of ELF header the bootloader need to know how much is the
size of all program headers inside the ELF to allocate enough memory for them,
so I decided to loop over all program header to find out how much memory they
need but in that loop I check two important things too.  
First I count program headers with `PT_LOAD` type to be sure that at least one
program header with type `PT_LOAD` exists.  
Second I check that the entry point be in one of `PT_LOAD` program headers.  
These two checks are very important because if one of them fails, and the
bootloader still jumps to entry point, there gonna be a huge confusion.

**3**.It allocates enough memory to load kernel  
In the previous step the bootloader figured out value of the `kernel_size` and
`kernel_start` so now it can use `BootServices->AllocatePages()` to allocate
enough pages that can fit the kernel but, It also knows stack size from
`STACK_SIZE` macro, so it'll allocate memory in address:
>`kernel_start - STACK_SIZE`

So the first `STACK_SIZE` of the allocated memory is gonna be used as a *boot
stack* for the kernel.
