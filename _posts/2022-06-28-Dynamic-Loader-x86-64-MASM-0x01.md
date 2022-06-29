---
title: Writing A (covert) Dynamic Loader in x86-64 MASM [0x01]
categories: [PROGRAMMING, ASSEMBLY]
tags: [windows, masm, x86-64]
---

<link rel="stylesheet"
      href="//cdnjs.cloudflare.com/ajax/libs/highlight.js/11.5.1/styles/vs2015.min.css">
<script src="//cdnjs.cloudflare.com/ajax/libs/highlight.js/11.5.1/highlight.min.js"></script>

<script src="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.5.1/languages/x86asm.min.js"></script>

<script type="module">
    import hljs from 'https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.5.1/highlight.min.js';
    import x86asm from 'https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.5.1/languages/x86asm.min.js';
    hljs.registerLanguage('x86asm', x86asm);
    hljs.registerLanguage('dos', dos);
</script>

<script>hljs.highlightAll();</script>

<H3 style="text-align:center">
    0x01 - General Code Structure
</H3>

---

### What we're doing: 
<p style="text-align:center">
    <video width=500 src="https://user-images.githubusercontent.com/20095224/176233849-564be7b5-3e66-4581-878e-9e22459a488a.mp4" controls="controls"></video>
</p>

### How we're doing it:
#### We'll start by declaring the only two imports we require to bootstrap our loader.

<pre><code class="language-x86asm">
; HMODULE GetModuleHandleA(LPCSTR modulename)
EXTRN __imp_GetModuleHandleA:PROC

; FARPROC GetProcAddress (HMODULE modulehandle, LPCSTR procname)
EXTRN __imp_GetProcAddress:PROC
</code></pre>

#### The next step is to prepare some static global variables in the ```.data section```.

<pre><code class="language-x86asm">
; Static data 
_DATA SEGMENT
    
    User32dll       db      "User32.dll", 0h
    Kernel32dll     db      "kernel32.dll", 0h
    Beep            db      "Beep", 0h
    LoadLibraryA    db      "LoadLibraryA", 0h
    MessageBoxA     db      "MessageBoxA", 0h
    ExitProcess     db      "ExitProcess", 0h
    Caption         db      "Hello!", 0h
    Message         db      "Click on ", '"', "Ok", '"', '.', 0h

_DATA ENDS
</code></pre>

#### We'll also need global variables for the resolved APIs (function pointers), zero initialized.

<pre><code class="language-x86asm">
; Uninitialized data
_BSS SEGMENT align(8) READ WRITE
    
    User32Ptr       dq      0000000000000000h
    Kernel32Ptr     dq      0000000000000000h
    BeepPtr         dq      0000000000000000h
    MessageBoxAPtr  dq      0000000000000000h
    ExitProcessPtr  dq      0000000000000000h
    LoadLibraryAPtr dq      0000000000000000h

_BSS ENDS
</code></pre>

#### Our code will live in the ```.text``` section and contains 3 functions: ***start***, ***setup*** and ***main***.
 ***start*** is the entry point and is responsible for the loader's general code flow:
 it calls setup, then main and terminates the process.

<pre><code class="language-x86asm">
; Code
_TEXT SEGMENT
    
    ; Entry point
    start PROC
        sub     rsp, 8
        call    setup
        call    main
        mov     rcx, rax
        call    ExitProcessPtr
        add     rsp, 8
        ret
    start ENDP
</code></pre>

#### ***setup*** is responsible for resolving APIs and saving the resulting function pointers in the ```.bss``` section using [GetModuleHandleA](https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getmodulehandlea), [LoadLibraryA](https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-loadlibrarya) and [GetProcAddress](https://docs.microsoft.com/en-us/windows/win32/api/libloaderapi/nf-libloaderapi-getprocaddress).

<pre><code class="language-x86asm">
    ; Resolves Win32 APIs and saves function ptrs as 
    ; global variables for later use
    setup PROC
        sub     rsp, 28h
        lea     rcx, Kernel32dll
        call    QWORD PTR __imp_GetModuleHandleA            ; Kernel32Ptr = GetModuleHandleA ("kernel32.dll")
        mov     Kernel32Ptr, rax
        mov     rcx, Kernel32Ptr
        lea     rdx, LoadLibraryA
        call    QWORD PTR __imp_GetProcAddress              ; LoadLibraryAPtr = GetProcAddress (Kernel32Ptr, "LoadLibraryA")
        mov     LoadLibraryAPtr, rax
        lea     rcx, User32dll
        call    LoadLibraryAPtr                             ; User32Ptr = LoadLibraryA ("user32.dll")
        mov     User32Ptr, rax
        mov     rcx, Kernel32Ptr
        lea     rdx, Beep
        call    QWORD PTR __imp_GetProcAddress              ; BeepPtr = GetProcAddress (Kernel32Ptr, "Beep")
        mov     BeepPtr, rax
        mov     rcx, User32Ptr
        lea     rdx, MessageBoxA
        call    QWORD PTR __imp_GetProcAddress              ; MessageBoxAPtr = GetProcAddress (User32Ptr, "MessageBoxA")
        mov     MessageBoxAPtr, rax
        mov     rcx, Kernel32Ptr
        lea     rdx, ExitProcess
        call    QWORD PTR __imp_GetProcAddress              ; ExitProcessPtr = GetProcAddress (Kernel32Ptr, "ExitProcess")
        mov     ExitProcessPtr, rax
        add     rsp, 28h
        ret
    setup ENDP
</code></pre>

#### Finally, we call ***main*** which is where the program starts using the collected artifacts: 

<pre><code class="language-x86asm">
    ; Just like nothing happened!
    main PROC
        sub     rsp, 28h
        xor     ecx, ecx
        lea     rdx, Message
        lea     r8, Caption
        xor     r9d, r9d
        call    MessageBoxAPtr                              ; MessageBoxA (NULL, Message, Caption, NULL)
        mov     ecx, 100h
        mov     edx, 100h
        call    BeepPtr                                     ; Beep (256, 256)
        add     rsp, 28h
        ret
    main ENDP

_TEXT ENDS
END
</code></pre>

#### Assemble and Link:

Paste the following code in a file called `make.bat`:

```batch
@echo off
if ["%~1"]==[""] (
	
    ml64 main.asm /link /entry:start /subsystem:windows /OUT:poc.exe kernel32.lib /nologo

) else if ["%~1"]==["clean"] (
	
    del *.obj,*.exe,*.lnk

) else (

	echo usage:
	echo make           - assembles and links main.asm into poc.exe.
	echo make clean     - deletes all .exe, .obj and .lnk files in the dir.
)
@echo on
```

This will serve as a rudimentary build system, and that's all we really need in this case.

Issue the `make` command.

If your copy pasta game is on point you should see our `poc.exe` binary.

And that's it!

We now have a functional piece of code that we can use as reference.
