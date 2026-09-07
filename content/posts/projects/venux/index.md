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
The UEFI has already provided us a flat 64-bit memory and several tools that
helps us writing a beautiful *bootloader*

---

### What Does a UEFI Bootloader Do?
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
`STACK_SIZE` macro, so it'll allocate memory at address:
>`kernel_start - STACK_SIZE`

So the first `STACK_SIZE` of the allocated memory is gonna be used as a *boot
stack* for the kernel.

**4**.It copies kernel to the allocated memory  
After parsing the kernel's ELF64, copying data from file at `offset = p_offset`
for each program header to the memory is the easy part.

**5**.It creates new *Page Table* and maps allocated memory to the address that
the kernel really needs (in my case it's **0xffffffff80000000 + 1M -
`STACK_SIZE`**)

### But How Do I Write a UEFI Bootloader?
I implemented the first four steps into the `efi_load_kernel()` function. And
the fifth step into `efi_map_kernel()` function.

Very Important part of writing a bootloader for me, is it's error handling. In
my bootloader I have several types of error :
- FATAL_ERROR : In this case I will clean up every `Open()`, `OpenVolume()` or
  any other open that interacts with file systems, **but** I won't free any
  allocated memory because after a FATAL_ERROR the system will shutdown or halt
  so memory isn't gonna use after it.
- LOAD_ERROR : this error occurs when `efi_load_kernel()` or some functions that
   `efi_load_kernel()` uses itself fail.
