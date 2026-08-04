---
title: NTDLL unhooking rants
tags: []
categories: []
date: 2026-08-02 03:12:02
---

# NTDLL Unhooking rants 

_Disclaimer: This blog is NOT a tutorial about unhooking NTDLL - it's more of a rant about the different things I have encountered while unhooking NTDLL - memory permissions, VADs, mapping stuff, which I think other people might find interesting._

So a while back, I was writing a little payload which was unhooking NTDLL from a process by mapping it from `KnownDlls`. Pretty simple, right? Here is a pseudo-snippet of the code which was responsible for mapping ntdll from `KnownDlls` into memory:

```c
OBJECT_ATTRIBUTES objattr = {0};
HANDLE hSection = NULL; 
uniNtdll.Buffer = (PWSTR) KNOWN_NTDLL;
uniNtdll.Length = wcslen(KNOWN_NTDLL) * sizeof(WCHAR);
uniNtdll.MaximumLength = uniNtdll.Length + sizeof(WCHAR);

InitializeObjectAttributes(&objattr, &uniNtdll, OBJ_CASE_INSENSITIVE, NULL, NULL);
HMODULE hmodNtdll = GetModuleHandleA("ntdll");
    
fNtOpenSection pNtOpenSection = (fNtOpenSection) GetProcAddress(hmodNtdll, "NtOpenSection");
    
NTSTATUS status = pNtOpenSection(&hSection, SECTION_MAP_READ, &objattr);
LPVOID pNtdllMapped = MapViewOfFile(hSection, FILE_MAP_READ, 0, 0, 0);
CloseHandle(hSection);
getchar();
```

<!-- more -->

Now, since we are mapping things to memory, we should have two copies of NTDLL in the current process memory:

<img src="/images/ntdll_unhooking/two_ntdlls.png" style="display:block; margin:0;" />

<br />

The first question on my dumb mind is _"Why is this mapped as a module?"_. First things first, we inspect the `\KnownDlls\ntdll.dll` object. 

![](./images/ntdll_unhooking/ntdll_obj.png)

<br />

So it's a `SEC_IMAGE` object. I am assuming `smss.exe` is the one responsible for creating it at boot time, but just to be sure, set Procmon to log boot time events and with some filtering, we get the following:


| Process Name | PID | Operation | Path | Result | Detail |
|--------------|-----|-----------|------|--------|--------|
|smss.exe|512|Load Image|C:\Windows\System32\ntdll.dll|SUCCESS|Image Base: 0x7ff901ea0000, Image Size: 0x266000|
|smss.exe|512|CreateFile|C:\Windows\System32\ntdll.dll|SUCCESS|Desired Access: Execute/Traverse, Read Control, Synchronize, Disposition: Open, Options: Synchronous IO Non-Alert, Non-Directory File, Attributes: n/a, ShareMode: Read, Delete, AllocationSize: n/a, OpenResult: Opened|
|smss.exe|512|CreateFileMapping|C:\Windows\System32\ntdll.dll|FILE LOCKED WITH ONLY READERS|SyncType: SyncTypeCreateSection, PageProtection: PAGE_EXECUTE|
|smss.exe|512|CreateFileMapping|C:\Windows\System32\ntdll.dll|SUCCESS|SyncType: SyncTypeOther|
|smss.exe|512|Load Image|C:\Windows\System32\ntdll.dll|SUCCESS|Image Base: 0x1fd86800000, Image Size: 0x266000|
|smss.exe|512|CloseFile|C:\Windows\System32\ntdll.dll|SUCCESS||

<br />

The key eveny here is the `CreateFileMapping` opetation with `SyncType: SyncTypeCreateSection` and `PageProtection: PAGE_EXECUTE`. This is Process Monitor's representation of `NtCreateSection` being called with `SEC_IMAGE` semantics. 

Here is how the flow happens: `smss.exe` reads `C:\Windows\System32\ntdll.dll` from disk, Creates a section object (the `FILE LOCKED WITH ONLY READERS` result is normal; it means the file system acquired a shared lock for the section creation), verifies the image (`LoadImage`) and then closes the handle to file on disk. Cool, now we can go back to answering our question - _"Why does this show up as a mapped module?"_

For this we need to first ask: _"Hey how does System Informer know which modules are loaded into a process?"_

So System Informer does not just walk the PEB/LDR (which I initially thought), it also uses Virtual Address Descriptors. From Windows Internals Part 1:
 
> When a process reserves address space or maps a view of a section, the memory manager creates a VAD to store any information supplied by the allocation request, such as the range of addresses being reserved, whether the range will be shared or private, whether a child process can inherit the contents of the range, and the page protection applied to pages in the range. 

When know `smss.exe` creates the section object with `SEC_IMAGE`, which tells the kernel memory manager: _"this section backs a PE image"_(See the [documentation for `CreateFileMappingW`](https://learn.microsoft.com/en-us/windows/win32/api/memoryapi/nf-memoryapi-createfilemappingw)). When we map it using `MapViewOfFile()`, the kernel DOES NOT perform a flat byte-for-byte file mapping. It then proceeds to map each PE section at its virtual address offset (not raw file offset) - `.text` at one RVA, `.data` at another, etc. Again from Windows Internals:

>  The section object pointers structure points to one or two control areas. One control area is used to map the file when it is accessed as a data file and the other is used to map the file when it is run as an executable image. A control area in turn points to subsection structures that describe the mapping information for each section of the file (read-only, read/write, copy-on write, and so on). The control area also points to a segment structure allocated in paged pool, which in turn points to the prototype PTEs used to map to the actual pages mapped by the section object. 

This tells us:
- Image files get a distinct image control area (separate from the data control area)
- The control area points to subsection structures, one per PE section: each describing a different range of the file with its own protection attributes
- The segment's prototype PTEs are what ultimately map virtual addresses to physical page

The memory manager tags every page in the resulting region with memory type `MEM_IMAGE` (`0x1000000`) - so ideally, we should be able to use `NtQueryVirtualMemory()` with `MemoryMappedFilenameInformation` to get some of this information:

```c
void EnumImageMappings() {
    MEMORY_BASIC_INFORMATION mbi = {0};
    MEMORY_MAPPED_FILE_NAME_INFORMATION nameInfo = {0};
    UCHAR *addr = NULL;
    SIZE_T retLen = 0;
    LPVOID lastBase = NULL;

    printf("\n[+] === MEM_IMAGE mappings (what System Informer sees) ===\n");
    printf("    %-18s  %-10s  %s\n", "Base Address", "Size", "Mapped File");
    printf("    %-18s  %-10s  %s\n", "------------", "----", "-----------");

    while (VirtualQuery(addr, &mbi, sizeof(mbi))) {
        if (mbi.Type == MEM_IMAGE && mbi.AllocationBase != lastBase) {
            lastBase = mbi.AllocationBase;

            NTSTATUS status = NtQueryVirtualMemory(
                GetCurrentProcess(),
                mbi.AllocationBase,
                MemoryMappedFilenameInformation,
                &nameInfo,
                sizeof(nameInfo),
                &retLen
            );

            if (status == STATUS_SUCCESS) {
                printf("    0x%-16p  0x%-8lx  %.*ls\n",
                    mbi.AllocationBase,
                    (unsigned long)mbi.RegionSize,
                    (int)(nameInfo.Name.Length / sizeof(WCHAR)),
                    nameInfo.Name.Buffer);
            } else {
                printf("    0x%-16p  0x%-8lx  <query failed: 0x%lx>\n",
                    mbi.AllocationBase,
                    (unsigned long)mbi.RegionSize,
                    status);
            }
        }

        addr += mbi.RegionSize;
        if (addr < (UCHAR*)mbi.BaseAddress)
            break;
    }

    printf("\n");
}
```

Calling this function before and after mapping ntdll verifies our claim:

![](./images/ntdll_unhooking/section_map.png)

<br />

So, we know now how System Informer gets its information. Cool. Time to move on (we still have opsec considerations but we would ignore that for now). 

Time to copy the clean `.text` section. For testing, we would be using Frida to hook `NtAllocateVirtualMemory()`<sup>1</sup>

```js
var pNtAllocateVirtualMemory = Module.findExportByName("ntdll.dll", "NtAllocateVirtualMemory");

Interceptor.attach(pNtAllocateVirtualMemory, {
    onEnter: function (args) {
        send("[+] Called NtAllocateVirtualMemory [+]");
    }
});``
```

Pre-hooking the address looks like this:

![](./images/ntdll_unhooking/pre_hooking.png)

After-hooking the address looks like:

![](./images/ntdll_unhooking/post_hooking.png)

The biggest change we see is that the memory protection for the changes from `PAGE_EXECUTE_READ` to `PAGE_EXECUTE_READWRITE`. So this gives me the idea: Instead of mapping the entirity of `.text` section, how about we just map the pages with `RWX` permissions? Time to write some code! 

Here is the function we write:

```c
BOOL FindAndRestoreHookedPages(LPVOID pOrigTextSection, LPVOID pCleanTextSection, DWORD textSize) {
    MEMORY_BASIC_INFORMATION mbi = {0};
    UCHAR *addr = (UCHAR *)pOrigTextSection;
    UCHAR *end  = addr + textSize;
    BOOL found = FALSE;

    printf("\n[+] === Scanning original NTDLL .text for RWX pages ===\n");

    while (addr < end && VirtualQuery(addr, &mbi, sizeof(mbi))) {
        DWORD prot = mbi.Protect & 0xFF;
        if (prot == PAGE_EXECUTE_READWRITE) {
            ULONG_PTR offset = (ULONG_PTR)mbi.BaseAddress - (ULONG_PTR)pOrigTextSection;
            SIZE_T regionSize = mbi.RegionSize;

            if ((UCHAR *)mbi.BaseAddress + regionSize > end)
                regionSize = end - (UCHAR *)mbi.BaseAddress;

            printf("[!] RWX page found at 0x%p (offset 0x%llx, size 0x%llx)\n",
                mbi.BaseAddress,
                (unsigned long long)offset,
                (unsigned long long)regionSize);

            memcpy(mbi.BaseAddress, (UCHAR *)pCleanTextSection + offset, regionSize);
            printf("[+] Restored %llu bytes from clean NTDLL copy\n", (unsigned long long)regionSize);

           found = TRUE;
        }

        addr += mbi.RegionSize;
        if (addr < (UCHAR *)mbi.BaseAddress)
            break;
    }

    if (!found)
        printf("[+] No RWX pages found in .text section (no hooks detected)\n");

    return found;
}
``` 

Pretty simple stuff (not the best code - but we make do). Now let's do the Frida hooking again:

![](./images/ntdll_unhooking/frida_hooking.png)

Now, hitting Enter should overwrite this hook:

![](./images/ntdll_unhooking/overwrite_hook.png)

And on windbg side of things:

![](./images/ntdll_unhooking/overwritten_hook.png)

So, we are able to overwrite the hook. Unmapping the section also removes it from the list of loaded modules. Now while I do want to say that this RWX memory region can be an indicator of an EDR hook but from testing, it turns out that they usually revert back to RX permissions. I guess we can use `NtQueryVirtualMemory(MemoryWorkingSetExInformation)` to find private pages but that is left as an exercise to the reader ;)

Another good exercise for the reader would be to get rid of the _Second NTDLL loaded_ IoC (even I am experimenting with some stuff - will publish some stuff when I am a bit more confident). Till next time - _Ciao_.

## References

- [(Un)Hooking, COWs and Meow Meow](https://dazzyddos.github.io/posts/(Un)Hooking,-COWs-and-Meow-Meow/)
