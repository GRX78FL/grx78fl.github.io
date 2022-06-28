---
title: Writing A (covert) Dynamic Loader in x86-64 MASM | 0x00
categories: [PROGRAMMING, ASSEMBLY]
tags: [windows, masm, x86-64]
---

<H1 style="text-align:center">
    Writing A (covert) Dynamic Loader in x86-64 MASM
</H1>

<H3 style="text-align:center">
    0x00 - Introduction 
</H3>

---

#### Assumptions about the reader's knowledge:

- Understanding of C code (pointers, structs, control flow ...);
- Some Python3 experience (basic operations);
- Some Assembly experience (g.p. registers, stack layout, control flow ...);
- Basic computer usage knowledge.

#### Goals of this series:

- Get a working POC that when inspected statically gives nothing away;
- Avoid AV flagging;
- Get more comfortable in writing x86-64 Assembly code;
- Call Kernel32!Beep without importing it directly;
- Having fun, otherwise what's the point?

#### Tools used in this series:

- Operating System: [Windows 11](https://www.microsoft.com/en-ca/software-download/windows11);
- Code Editor: [Visual Studio Code](https://code.visualstudio.com/);
- VS Code extension: [x86-64 ASM Syntax Highlighting](https://marketplace.visualstudio.com/items?itemName=13xforever.language-x86-64-assembly)
- Assembler: [MASM](https://docs.microsoft.com/en-us/cpp/assembler/masm/masm-for-x64-ml64-exe?view=msvc-170);
- Scripting/Automation: [Python3](https://www.python.org/downloads/);
- Debugger: [WinDbg](https://docs.microsoft.com/en-us/windows-hardware/drivers/debugger/debugger-download-tools);
- Disassembler/Decompiler: [IDA Freeware](https://hex-rays.com/ida-free/);
- and of course: Just a little [Patience](https://www.youtube.com/watch?v=ErvgV4P6Fzc).

#### Inspiration, techniques and external resources:

- [Architecture 1001: x86-64 Assembly](https://p.ost2.fyi/courses/course-v1:OpenSecurityTraining2+Arch1001_x86-64_Asm+2021_v1/course/) by ([@XenoKovah](https://twitter.com/XenoKovah));
- [Undocumented Windows Structs](https://www.vergiliusproject.com/) (by [RevJay](https://github.com/RevJay) and [SergiusTheBest](https://github.com/SergiusTheBest));
- [PEB Walking](https://blog.christophetd.fr/hiding-windows-api-imports-with-a-customer-loader/) (by [@christophetd](https://twitter.com/christophetd));
- [API Hashing](https://www.ired.team/offensive-security/defense-evasion/windows-api-hashing-in-malware) (by [@spotheplanet](https://twitter.com/spotheplanet));
