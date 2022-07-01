---
title: Writing A (covert) Dynamic Loader in x86-64 MASM [0x02]
categories: [PROGRAMMING, ASSEMBLY]
tags: [windows, masm, x86-64]
---

<H3 style="text-align:center">
    [0x02] - The Problem
</H3>

---

### Imports:

As the analysts we'd like to quickly get an idea of [what this binary is doing](https://grx78fl.github.io/posts/Dynamic-Loader-x86-64-MASM-0x01/) statically.

So we sit down, open our favorite tool and check what kind of file it is, what libraries it depends on and what functions it calls:

![image](https://user-images.githubusercontent.com/20095224/176815891-486fbbae-cbb1-428f-9e9d-098bdeb7c5d2.png)
_Imports tab in IDA_

Looks like this executable will perform [dynamic linking at runtime](https://docs.microsoft.com/en-us/windows/win32/dlls/run-time-dynamic-linking), which is generally a red flag when dealing with unknown programs - even more so when it's the only functionality that's exposed to us.

### Strings:

Another major giveaway are strings because they shine a light on the intentions of a programmer.
Here we can see that the author didn't bother concealing their intentions, giving us valuable insight into what is going on:

![image](https://user-images.githubusercontent.com/20095224/176840814-f4126500-a2fc-4425-8f18-3b2bf8e55ed5.png)
_Strings tab in IDA (Shift + F12)_

by simply looking at this we can be confident that the binary will:
- get a handle to kerne32.dll;
- load user32.dll since it wasn't linked at compile time;
- resolve LoadLibraryA, ExitProcess, Beep and MessageBoxA.

and we didn't need to look at a single byte of code.

### The code:

Not satisfied with just strings and imports we decide to look at the code to make sure our assumptions are correct.
The entry point looks very minimal as there's only 3 function calls.

Once decompiled, the first function looks like so:

![image](https://user-images.githubusercontent.com/20095224/176848045-1b237084-8044-4332-adef-658654cad87c.png)
_(Hit "\\" to get rid of type casts)_

Nothing that we didn't know already, so what about the second function?

![image](https://user-images.githubusercontent.com/20095224/176848745-bdac09ab-0a86-4c04-9fd6-0bc719089855.png)

Not much going on here either and that little that's going on is clear as day.

The last function call is a little different as it appears to be a dynamically resolved API, which we've seen in function 1 as ExitProcess.

### Conclusion

The author of this program has been careless enough to let us know all their secrets and we have a good understanding of what their code does.
Maybe nex time we won't be as lucky, but for now we publish our research and call it a win.

Score: sloppy at best.