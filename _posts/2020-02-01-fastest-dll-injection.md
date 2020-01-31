---
layout: post
title:  Fastest DLL Injection
categories: [RE,.NET]
comment_issue_id: 2
---
## Basics
The classical workflow for injecting and running code in a process, or at least the one that someone else used when writing the launcher I am working on, is as follows:
1. Start the process
2. Inject the DLL
3. Suspend the process
4. Call the DLL's function
5. Resume process

There is a faster way, but first let me explain this one, as they only differ in one detail. In code, it works as follows:
#### Start the process
```c#
using Process = System.Diagnostics.Process;
var process = Process.Start(executable, arguments);
```
#### Inject the DLL
Using functions from `kernel32.dll`, we `CreateRemoteThread` in the target process that loads the DLL using `LoadLibraryA`. The DLL name is passed as an argument to `LoadLibraryA`, therefore memory has to be allocated in the process, written and freed using `VirtualAllocEx`, `WriteProcessMemory` and `VirtualFreeEx`. When `LoadLibraryA` returns, we are done. 
```c#
[DllImport("kernel32.dll")]
private static extern IntPtr GetProcAddress(IntPtr hModule, string lpProcName);
[DllImport("kernel32.dll")]
private static extern IntPtr GetModuleHandle(string lpModuleName);
[DllImport("kernel32.dll")]
private static extern IntPtr VirtualAllocEx(IntPtr hProcess, IntPtr lpAddress, int dwSize, uint flAllocationType, uint flProtect);
[DllImport("kernel32.dll")]
private static extern int WriteProcessMemory(IntPtr hProcess, IntPtr lpBaseAddress, byte[] buffer, int size, int lpNumberOfBytesWritten);
[DllImport("kernel32.dll")]
private static extern IntPtr CreateRemoteThread(IntPtr hProcess, IntPtr lpThreadAttribute, IntPtr dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);
[DllImport("kernel32", EntryPoint = "WaitForSingleObject")]
private static extern uint WaitForSingleObject(IntPtr hObject, uint dwMilliseconds);
[DllImport("kernel32.dll")]
private static extern bool VirtualFreeEx(IntPtr hProcess, IntPtr lpAddress, int dwSize, uint dwFreeType);
[DllImport("kernel32.dll")]
private static extern int CloseHandle(IntPtr hObject);

public static bool Inject(Process process, string dll)
{
    IntPtr loadLibraryAddress = GetProcAddress(GetModuleHandle("kernel32.dll"), "LoadLibraryA");
    if (loadLibraryAddress == IntPtr.Zero) return false;

    IntPtr argument = VirtualAllocEx(process.Handle, IntPtr.Zero, dll.Length, (0x1000 | 0x2000), 0x40);
    if (argument == IntPtr.Zero) return false;

    byte[] argumentValue = Encoding.ASCII.GetBytes(dll);
    if (WriteProcessMemory(process.Handle, argument, argumentValue, argumentValue.Length, 0) == 0) return false;

    IntPtr thread = CreateRemoteThread(process.Handle, IntPtr.Zero, IntPtr.Zero, loadLibraryAddress, argument, 0, IntPtr.Zero);
    if (thread == IntPtr.Zero) return false;

    WaitForSingleObject(thread, 0xFFFFFFFF);
    VirtualFreeEx(thread, argument, argumentValue.Length, 0x10000);
    CloseHandle(thread);

    return true;
}
```
#### Suspend process
We don't want the target to move while our code is running, so suspend all threads:

```c#
[DllImport("kernel32.dll")]
private static extern IntPtr OpenThread(int dwDesiredAccess, bool bInheritHandle, int dwThreadId);
[DllImport("kernel32.dll")]
private static extern uint SuspendThread(IntPtr hThread);

private static void SuspendProcess(Process process)
{
    foreach (IntPtr thread in process.Threads.Cast<ProcessThread>().Select(t => OpenThread(2, false, t.Id)).Where(t => t != IntPtr.Zero))
    {
        SuspendThread(thread);
        CloseHandle(thread);
    }
}
```
#### Call the DLL's function
Now that the thread is all ours, load the DLL and get the offset of the function in the DLL using `GetProcAddress`. Get the base address of the DLL in the target process and add it to the offset. Then again create a remote thread with the function address (here without arguments) and wait for it to finish.
```c#
[DllImport("kernel32.dll", EntryPoint = "LoadLibraryExW", SetLastError = true)]
private static extern IntPtr LoadLibraryEx([MarshalAs(UnmanagedType.LPWStr)] string lpFileName, IntPtr hFile, int dwFlags);

private static ProcessModule GetModule(Process process, string dll)
{
    return process.Modules.Cast<ProcessModule>().FirstOrDefault(module => module.FileName.Equals(dll));
}

public static bool CallExport(Process process, string dll, string function)
{
    IntPtr module = LoadLibraryEx(dll, IntPtr.Zero, 1);
    if (module == IntPtr.Zero) return false;

    IntPtr functionAddress = GetProcAddress(module, function);
    if (functionAddress == IntPtr.Zero) return false;
    functionAddress = GetModule(process, dll).BaseAddress + (int)functionAddress - (int)module;

    IntPtr thread = CreateRemoteThread(process.Handle, IntPtr.Zero, IntPtr.Zero, functionAddress, IntPtr.Zero, 0, IntPtr.Zero);
    if (thread == IntPtr.Zero) return false;

    WaitForSingleObject(thread, 0xFFFFFFFF);
    CloseHandle(thread);

    return true;
}
```
#### Resume threads
The opposite of `SuspendProcess`.
```c#
[DllImport("kernel32.dll")]
private static extern uint ResumeThread(IntPtr hThread);

private static void ResumeProcess(Process process)
{
    foreach (IntPtr thread in process.Threads.Cast<ProcessThread>().Select(t => OpenThread(2, false, t.Id)).Where(t => t != IntPtr.Zero))
    {
        uint suspendCount;
        do
        {
            suspendCount = ResumeThread(thread);
        } while (suspendCount > 0);

        CloseHandle(thread);
    }
}
```
## So slow
The problem is that there is no way to know at which point in the execution of the process we suspend it. Possibly, code we are interested in was already run and cannot be altered anymore. I would like to have a way to consistently run my code in the process context as early as possible.
## Fast and Furious
The trick for the fastest DLL injection is to launch the process already in suspended state. Thanks to hofingerandi, the author of the article found at [https://www.codeproject.com/Articles/230005/Launch-a-process-suspended](https://www.codeproject.com/Articles/230005/Launch-a-process-suspended). Instead of .NET's `Process.Create` we use the `CreateProcess` function, again from `kernel32.dll`, and pass it the `ProcessCreationFlags.CREATE_SUSPENDED` flag. The resulting main thread of the process is suspended and we can directly proceed with injecting the DLL and running our code. This way, it is our turn already before the first instruction of the main process is run.

