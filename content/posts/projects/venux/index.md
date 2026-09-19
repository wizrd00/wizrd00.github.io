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
enough pages that can fit the kernel.

**4**.It copies kernel to the allocated memory  
After parsing the kernel's ELF64, copying data from file at `offset = p_offset`
for each program header to the memory is the easy part.

**5**.It creates new *Page Table* and maps the bootloader itself and allocated
memory to the address that the kernel really needs (in my case it's
**0xffffffff80000000 + 1M**)

### What Does My UEFI Bootloader Do (step by step)?
1.It initialize three important variables `efi_app_start`, `efi_app_end` and
`efi_app_size` at the begining of the code by calling `HandleProtocol()`
function and passing the first argument of `efi_main()` to it like this :
``` C
SysTab->BootServices->HandleProtocol(ImgHdl, &LoadedImageGuid,
    (VOID **) &LoadedImage);
```

2.It set the `bios` member of `kargs` to `UEFI_BIOS`.

3.It passes the `RuntimeServices` address to the kernel by calling the
`efi_kargs_add_rt()` function.This function include this code :
``` C
args.uefi_rt = (void *)SysTab->RuntimeServices;
```

4.It opens the **venux.elf** file and parse it's ELF64 headers, validates them
and allocate memory based on them then copies the kernel code into that memory.
The `efi_load_kernel()` does all that operations, How? like this :
- 1.First it opens the kernel file with the code below :
``` C
EFI_LOADED_IMAGE *LoadedImage = NULL;
EFI_GUID LoadedImageGuid = EFI_LOADED_IMAGE_PROTOCOL_GUID;
SysTab->BootServices->HandleProtocol(ImgHdl, &LoadedImageGuid,
    (VOID **) &LoadedImage);

EFI_SIMPLE_FILE_SYSTEM_PROTOCOL *FileSystem = NULL;
EFI_GUID FileSystemGuid = EFI_SIMPLE_FILE_SYSTEM_PROTOCOL_GUID;
SysTab->BootServices->HandleProtocol(LoadedImage->DeviceHandle, &FileSystemGuid,
    (VOID **) &FileSystem);

EFI_FILE_PROTOCOL *Volume = NULL;
FileSystem->OpenVolume(FileSystem, &Volume);

EFI_FILE_PROTOCOL *KernelFile = NULL;
Volume->Open(Volume, &KernelFile, "venux.elf", EFI_FILE_MODE_READ, (UINT64)0);
```
- 2.Then it read exactly `sizeof(struct boot_elf64_ehdr)` from the file.
- 3.Validating the Entry Header of the ELF file is the next move
``` C
static int
efi_validate_elf(const struct boot_elf64_ehdr *ehdr)
{
	int ret = 0;
	unsigned char exp_ident[] = {0x7f, 'E', 'L', 'F', ELFCLASS64,
	    ELFDATA2LSB, EV_CURRENT, ELFOSABI_SYSV, 0, 0, 0, 0, 0, 0, 0, 0};
	/* the efi_memcmp() is a custom memcmp function */
	if (efi_memcmp((void *)ehdr->e_ident, (void *)exp_ident,
	    (size_t)ELFNIDENT) != 0)
		return ret = LOAD_ERROR_INVALID_ELF_IDENT;
	if (ehdr->e_type != (UINT16)ET_EXEC)
		return ret = LOAD_ERROR_INVALID_ELF_TYPE;
	if (ehdr->e_machine != (UINT16)EM_X86_64)
		return ret = LOAD_ERROR_INVALID_ELF_MACHINE;
	return ret;
}
```
- 4.Then it sets the kernel file position to `ehdr.e_phoff` and reads each
  Program Header to set critical variables `virt_kernel_start` and
  `virt_kernel_end`, then verifies the *entry point* of the kernel to be valid.
- 5.With two variable `virt_kernel_start` and `virt_kernel_end`, the bootloader
  allocate `(kernel_size / 4096) + 1` pages with `AllocatePages()` and stores
  the start address of the first page into `real_kernel_start` and calculates
  `real_kernel_end` with help of `kernel_size`
- 6.Sets the kernel file position to `ehdr.e_phoff` again.
- 7.This time reads each kernel code and copies them into relative address like
  this :
``` C
size_t filesz = (size_t)phdr.p_filesz;
size_t gapsz = (size_t)phdr.p_memsz - filesz;
/* calculating the relative address like this */
size_t addr = real_kernel_start + ((size_t)phdr.p_vaddr - virt_kernel_start);
while (filesz > 0) {
	UINTN readsz = (UINTN)((filesz < BUFFER_SIZE) ? filesz : BUFFER_SIZE);
	status = KernelFile->Read(KernelFile, &readsz, (VOID *)buffer);
	if (EFI_ERROR(status)) {
		/* do stuff */
	}
	/* efi_memcpy() and efi_memset() are custom functions */
	efi_memcpy((void *)addr, (void *)buffer, readsz);
	filesz -= readsz;
	addr += readsz;
}
efi_memset((void *)addr, 0, gapsz);
```
- 8.Now the `efi_load_kernel()` completes it's job by closing both `KernelFile`
  and `Volume`.

4.It stores the `real_kernel_start` into `kargs.kern_start`, because the kernel
only knows about it's virtual address and has no idea about it's real address.
So it is bootloader's responsibility to passes this value to the kernel.
