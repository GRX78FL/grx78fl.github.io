---
title: WinDbg Preview - a dirty fix for "Levels not implemented for this platform"
categories: [WINDBG, ASSEMBLY]
tags: [windows, windbg, x86-64]
---

## **The issue**

Imagine being deep into a kernel debugging session and suddenly your debugger stops working as expected. 

<div>
  <center>

![(Levels not implemented for this platform)](https://user-images.githubusercontent.com/20095224/219982466-9d03c9f1-4218-4b15-883b-43d70a3ff7a8.png)

  </center>
</div>

<u>_([This](https://learn.microsoft.com/en-us/answers/questions/1010386/windbg-pte-command-levels-not-implemented-for-this)</u> has been
<u>[going on](https://github.com/MicrosoftDocs/windows-driver-docs/issues/3058)</u> for
<u>[**years**](https://github.com/MicrosoftDocs/windows-driver-docs/issues/2587)</u> at this point and it doesn't look like there's active interest in fixing or investigating the bug, yet.)_

Still, it continues to be a crippling bug that affects us at the worst possible times.

Rebooting to roll the dice in hopes this will temporarily fix the issue might be out of the question since debugging without symbols is already quite involved, and scrapping progress in for convenience's sake feels like a massive waste of time, so let's try to figure out what causes this to happen by looking at the symptoms.

## **KdExts.dll**

Extensions! `!pte` is what's called an extension in WinDbg terms, that is, it's not *core* functionality since it only applies to kernel debugging sessions.
It is therefore implemented in the Kernel Debugger Extensions DLL (KdExts.dll) found in the install directory of WinDbg - which is likely somewhere in `C:\Program Files\WindowsApps`, for my particular version it is found at  
`C:\Program Files\WindowsApps\Microsoft.WinDbg_1.2210.3001.0_x64__8wekyb3d8bbwe\amd64\winxp\kdexts.dll`

I'll save you the trouble of finding where exactly the "Levels not implemented for this platform" message is referenced (you can do this an exercise if you'd like), and show you what the code responsible looks like once decompliled:

```cpp
  ...
  switch ( g_TargetMachine )                     // debuggee machine's arch
  {
    case IMAGE_FILE_MACHINE_I386:
      goto I386;
    case IMAGE_FILE_MACHINE_ARM:
    case IMAGE_FILE_MACHINE_THUMB:
    case IMAGE_FILE_MACHINE_ARMNT:
      v10 = 10;
  I386:
      v9 = 2;
      if ( !PaeEnabled )
        v10 = 10;
      break;
    case IMAGE_FILE_MACHINE_AMD64:             // we're obviously debugging an x64 machine
      v9 = DbgPagingLevels;
      if ( (unsigned int)DbgPagingLevels > 5 ) // Paging levels above 5 aren't implemented yet
      {
        msg = "Levels not implemented for this platform\n";
        goto FAIL;
      }
      break;
    ...
  FAIL:
      ExtensionApis.lpOutputRoutine(msg);
      return -1;
  }
  ...
 ```

 If you're not familiar with paging, this [odsev article](https://wiki.osdev.org/Paging) has some insight into why our bug is so absurd.

 ## **WinDbg-Inception**

 Here comes the dirty fix part.

 We have :
 - symbols for kdexts.dll;
 - a debugger (yes, we'll use WinDbg to patch WinDbg);
 - a good sense of humor

<div>
  <center>

 ![thanos](https://user-images.githubusercontent.com/20095224/219986156-6be94a4d-ee18-4122-928c-1311d1ded8b6.jpg)
 
  </center>
</div>

## **The fix**

#### **Steps:**

- open a second instance of WinDbg and attach to the faulty instance;
- issue the `.reload /f` command to force symbol loading;
- locate the g_TargetMachine global variable and confirm it's still IMAGE_FILE_MACHINE_AMD64 (0x8664) 

<div>
  <center>

![image](https://user-images.githubusercontent.com/20095224/219986361-74ff7230-a066-447b-89e9-fd0622fa085f.png)

  </center>
</div>

- great, let's look at DbgPagingLevels

<div>
  <center>

![image](https://user-images.githubusercontent.com/20095224/219986536-c0026428-22e5-4be0-9203-2f9c804dca8f.png)

  </center>
</div>

That makes no sense. 

Let's edit the memory to reflect the correct value as of Feb 2023 this value [should be](https://en.wikipedia.org/wiki/Intel_5-level_paging) either 4 or 5 depending on your sw/hw combo.
I'll edit it to 4, because of my setup. (This is due to be way more variable in the future, but by then, hopefully you won't need this guide anymore).

<div>
  <center>

![image](https://user-images.githubusercontent.com/20095224/219987026-cd2776ae-ad73-43f4-aaa7-f94a2a08f1d8.png)

  </center>
</div>

- hit `g` or `Go` button and make sure your Kd debugger instance is stable
- detach from from it and close the second WinDbg instance
- now try `!pte` again

<div>
  <center>

![image](https://user-images.githubusercontent.com/20095224/219987236-2d9593e0-bf81-4c6b-a58e-516b16ebf127.png)

  </center>
</div>

And that's it, you can keep on hunting bugs.

I hope this was useful.