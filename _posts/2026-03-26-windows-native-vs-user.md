---
layout: post
title: "Native vs. User Mode in Windows: An in-depth look"
date: 2026-03-26
categories: blog
---

In Windows, the way which programs and features are supported depend on two boot modes: Native and User modes. If you're using Windows
right now and you're reading this, you're in User mode, unless you hijacked low-level Windows.

# The relationship between the two modes
Right, so every program you're trying to start in Windows is likely an exe file. An exe file is a file format that contains executable code.
Now, it's not just linear machine code of course - it also contains metadata, resources, and other information.

It is typically composed of two things: a PE header which has info and settings about the exe file, followed by machine code, as well as
miscellaneous resources like the icon. Each is grouped into so-called "sections", with offset and length of data. For example, .rsrc contains
resources like the icon, and .text is where machine code is contained.

There's also a DOS header (identified with two bytes, M and Z), which is a legacy from the old days of MS-DOS. It contains a small stub that prints "This program cannot be run in DOS mode" if you try to run it in DOS.
It is only there for backwards compatibility, so that, whenever you try to run an exe file in DOS, it doesn't just crash, but instead gives you a message.

Okay, but we don't care about any of that. We really only care about the PE header. The PE header contains a field called "Subsystem", which indicates the intended environment for the executable. It can be set to values like "Windows GUI", "Windows CUI", "Native", and so on.
There are many other obscure or legacy values you can set to it, including EFI ROM, EFI Application, EFI Boot Service Driver, EFI Runtime Driver, Xbox, POSIX, and so on. But the most common ones are "Windows GUI", "Windows CUI", and "Native".

If the subsystem of the PE file is not recognized, Windows defaults to "Native".

Now, every time you run an EXE file, Windows checks this field in the PE header. For example, if it's set to "Windows CUI", that's Console mode,
and Windows will create a console window for it. That's for example, cmd.exe. If it's set to "Windows GUI", that's GUI mode, and Windows will create a window for it. That's for example, notepad.exe - because,
let's be honest, notepad.exe doesn't have a console window, and that's by design.

Now, even under CUI programs, you can still create your custom window using Windows API (for example, Blender does that). GUI just prevents the
console window from showing by default.

# Native Mode
So let's start off with Native Mode.

Look into the Windows Task Manager and scroll down to the Windows Processes field. The key processes here are:

- System (NTOSKrnl.exe)
- Registry (NTOSKrnl.exe)
- Windows Session Manager (SMSS.exe)
- Client Server Runtime Process (CSRSS.exe)
- autochk.exe (runs at startup, but it's not a long-running process, so it won't be in Task Manager)

These processes, except Registry (which is relatively new, but it's just a group of threads under System), have existed for a **long long time**. They've been around ever
since Windows NT came in 1993, but at that time, it was mostly for Server editions. It was only in Windows XP that it became mainstream for consumer editions.

Now, although there are many Windows processes, these four stand out, because they're in Native mode.

So, open up `C:\Windows\System32`, and try running one of these programs. For example, try running `CSRSS.exe`. You'll get an error message that says "The program csrss.exe cannot be run in Win32 mode.". That's because these programs are in Native mode, and they can only be run by the Windows kernel.

They effectively depend on low level system features, and if you could theoretically run them in User mode, that would be a security flaw, so by design, these programs cannot be run in User mode.
If you were to take a look into their Subsystem field, indeed, it's "Native".

Okay, now, curiosity gets better of you, and you want to know what happens if you try to run them in User mode. And let me warn you: **don't try this on your main computer, because it can cause instability and crashes.**

So I spun up a virtual machine, and I copied these programs into Desktop. Then I installed `CFF Explorer` and changed the Subsystem value to CUI.

Right off the bat, csrss.exe and autochk.exe did absolutely nothing. They just exited immediately, without any error message. Even with Admin/SYSTEM rights, they really did nothing at all.

Now, ntoskrnl is a bit more interesting. When I tried running it, it actually did something. Console window pops up, but I get a generic Windows error message box that reads:

> ntoskrnl.exe - Application Error
>
> The application was unable to start correctly (0xc0000481). Click OK to close the application.
>
> [OK]

The error code is highly interesting - 0xc0000481. That's not even a Windows error. That's a CPU exception. Specifically, part of Intel Trusted Execution Technology (TXT). It means "A TXT error occurred". So, it seems that when you try to run ntoskrnl.exe in User mode, it tries to execute some privileged instructions that are only allowed in Native mode, and that causes a CPU exception.

It's probably trying to run some Intel SMX x86 instruction, perhaps one of those GETSEC instructions. That makes sense, as that's part of setting up a secure environment.

But again, SYSTEM rights don't help. These instructions require privileges beyond SYSTEM.

And what about smss.exe? Well, I saved this one for last, because it's catastrophic. When I tried running smss.exe in User mode... I literally got a Blue Screen of Death (BSOD), stop code CRITICAL_PROCESS_DIED. Okay, WHAT??? This is in USER mode, WITHOUT ADMIN RIGHTS! How is that even possible? I thought that in User mode, you can't cause a BSOD, because you're not supposed to have access to low-level system features.

Okay, beside that, let's keep Native mode going further.
In Native Mode, you have full control over everything. Every single byte on your storage, every single byte in RAM, every process state, and nearly every single CPU instruction. This is also why
drivers run in Native Mode, because they need to interact with hardware directly, and they need to have access to privileged instructions.

The Blue Screen of Death (BSOD) is also a feature of Native mode. NTOSKrnl.exe is responsible for displaying and handling BSODs, and it needs to:
- Stop all non-Microsoft drivers (because they might be the cause of the crash)
- Reset the screen to a basic mode, so that it can display the error message
- Collect every single byte of memory and CPU state, so that it can include that in the error report
- Possibly connect to a debugger, if one is attached, so that it can provide more information to the developer, or if you're debugging with WinDbg.

Drivers can directly invoke a BSOD by calling the `KeBugCheckEx` function located inside ntoskrnl.exe, which is part of the Windows Driver Kit (WDK). This is typically used when a driver detects a critical error that it cannot recover from, and it needs to stop the system to prevent further damage.
BSODs can also be triggered without a driver's action to do so, for example, if the driver tries to access invalid memory, or if it tries to execute a privileged instruction that it's not allowed to. In those cases, the CPU will raise an exception, and if it's not handled properly, the kernel will intervene and bugcheck on its own, with the name of the driver provided right on the screen.

# User Mode
Right, so now let's talk about User mode.

If you've experimented with Windows for a while, you might've once tried to replace, for example, csrss.exe with Taskmgr.exe. That's not a thing. The Task Manager needs to create a window, and
window management is a feature of User mode. So, if you try to run Taskmgr.exe in Native mode, it will just crash immediately, because it tries to use features that are not available in Native mode.

That, by design, means, it is impossible to create a window or start a console program in Native mode, because those features haven't been loaded yet.

You might ask, "what handles user mode?" Well, it's not a process per se. It's actually handled by a driver, called, `win32k.sys`, which is located in `C:\Windows\System32`. This driver is responsible for handling all the graphical user interface (GUI) features in Windows, including window management, graphics rendering, input handling, mouse pointers, hardware acceleration, and who knows what. It runs in kernel mode, but it provides services to user mode applications.

Yes, internally, User mode is called Win32 Mode.

Yes, technically, you're always in Native mode, because the driver that handles User mode is a kernel driver, but the features of User mode are only available to processes that have the appropriate subsystem set in their PE header. So, while you might be running a process in User mode, the underlying system is still in Native mode, and the User mode features are provided by the win32k.sys driver. Win32k also enforces strict CPU state, such as, stricter memory protection, and it also prevents the execution of privileged instructions, which is why you can't cause a BSOD from User mode.
It's only unless you go into Administrator mode and try killing critical processes, that you can cause a BSOD from User mode, but that's a different story.

Once User mode settles, there's no going back to Native mode. You can't just switch back and forth between the two modes.

By design, this driver is loaded a bit late during boot phase. This ensures that low level system features have actually fully initialized.

So yes. You have a mouse pointer, you have a desktop, you can move and resize windows - that's hard work of win32k.sys.

Now, Console mode is actually part of User Mode too. When you run a console application:
- Win32k.sys runs conhost.exe
- Conhost.exe uses `AttachConsole` kernel32 function for your console application to attach to it
- Conhost.exe then creates a console window for your application, and it handles all the input and output for that console window.
- Unless we're talking about older versions of Windows, nowadays, your console program is technically invisible. The console window you're seeing is actually a separate process, conhost.exe, that is running in User mode, and your console application is just attached to it. This is a security feature, because it prevents console applications from directly interacting with the console window, which could be a potential attack vector.

The other Windows processes , such as `winlogon.exe`, `wininit.exe`, and `svchost.exe`, are also part of User mode. They provide various services and features to user mode applications, such as handling user logins, managing system services, and so on. They, therefore,
cannot access low-level system features, and they cannot cause a BSOD on their own. As soon as Win32k is loaded, it starts `wininit.exe`, which loads all these User components like winlogon or services.

Of course, you're probably wondering, "well, why if I end some of these processes, Windows crashes with a BSOD?" Here's the thing:
- Some of these processes are critical processes, which means that if they terminate unexpectedly, the system will crash to prevent further damage. For example, if `wininit.exe` terminates, the system will crash with a BSOD, because those processes are essential for the functioning of the operating system. They're not just regular processes; they're critical components of the Windows architecture.
- Normally, killing those processes would not lead to a BSOD.
- However, some of these processes invoke an ntdll.dll function called `RtlSetProcessIsCritical`, which marks the process as critical. This means that if the process terminates, whether due to an error or because you killed it, the system will crash with a BSOD. This is a security feature, because it prevents attackers from easily crashing the system by killing critical processes.
- This is by design. If some of these processes are killed, the system is in both in an unstable and unusable state. Windows is designed mostly for consumers, and it's likely safer to outright crash than leave the user use a computer behind a very quirky state.
- You can technically try this with your own application, and it will work. You can call `RtlSetProcessIsCritical` in your own application, and if you terminate it, it will cause a BSOD. However, you need to have the appropriate privileges to do this, and it's not something that should be done lightly, because it can lead to data loss and other issues. In real world apps, that's a really bad practice. Just accept that this function is something that should be used by Microsoft Windows components for the integrity of the OS.

# Windows 9x
If we're talking Windows 95, 98, and ME, there was no Native or User mode. There was only one mode, and it was a hybrid of the two. It had some features of Native mode, such as direct hardware access, but it also had some features of User mode, such as window management and graphics rendering. This is why Windows 9x was so unstable and prone to crashes, because there was no separation between the two modes, and any program could potentially cause a crash by accessing hardware directly or by using buggy drivers.

Instead of a driver securely handling User mode, it was a mix of all.

- `VMM32.VXD` was like today's `ntoskrnl.exe`. That was a low-level system component that provided core operating system services, such as memory management, process management, and hardware abstraction. It was responsible for managing the virtual memory system, handling interrupts, and providing a layer of abstraction between the hardware and the software.
- `GDI.EXE` handled all the graphics rendering.
- `USER.EXE` handled all the window management and input handling.
- `KRNL386.EXE` was responsible for handling the 386-specific features, such as virtual memory and multitasking.

These Windows versions did not have processes like `csrss.exe`, `ntoskrnl.exe`, `smss.exe`, or even user-mode components like `winlogon.exe`, `wininit.exe` or `svchost.exe`. Those are Windows NT traits.

None of the functions dedicated to listing all running processes displayed GDI.EXE, USER.EXE or KRNL386.EXE. They were hidden from the user, because they were considered part of the core operating system, and they were not meant to be interacted with directly by user applications. They were essentially running in a privileged mode, and they had access to all the hardware and system resources, but they were not exposed to the user in the same way that other processes were.
Instead, these functions would give `KERNEL32.DLL` as a process name, which abstracted all of these components into one. This is why, if you look at Sysinternals Process Explorer in Windows 9x, you'll see a process called "KERNEL32.DLL", which is actually a combination of all these components.
