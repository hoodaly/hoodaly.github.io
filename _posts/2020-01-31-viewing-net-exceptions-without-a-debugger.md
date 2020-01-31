---
layout: post
title:  Viewing .NET Exceptions without a Debugger
categories: [RE,.NET,IDA]
---
## e0434352 anyone?
I was debugging a process that was injected with a DLL written in C# in IDA, when IDA reported an exception:

```
771835D2: Managed .NET exception (V4+) (exc.code e0434352, tid 11400)
```

434352 is the ASCII code for CCR and it signifies a generic .NET exception. If we would be debugging the .NET code with Visual Studio, the IDE would take care of showing more details about the exception, but I couldn't get working breakpoints in disassembly while at the same time debugging the injected C# DLL.

## WinDBG - can do, but no
The WinDBG debugger allows debugging assembly, while .NET exceptions can be dissected using the SOS plugin:
After attaching the debugger, run
```
!load C:\Windows\Microsoft.NET\Framework\v4.0.30319\SOS.dll
sxe clr
```
to load the SOS plugin and break on clr exceptions. Then, when an exception is triggered, run the

```
!pe
```

command to print exception information.

Frankly, this feature is the only positive aspect I can find about WinDBG. It's clunky and I can't get it to do simple stuff. Therefore I was looking into different ways to get the exception details other than a full-fledged debugger, as there can be only one debugger (invasively) attached to any given process, and non-invasively attaching either didn't work or was cumbersome.

## Reversing is hard...
Unfortunately, there were many ways to get stuck. I tried understanding the SOS.dll multiple times, but it was too complicated and lead nowhere. I tried to understand how [dnSpy](https://github.com/0xd4d/dnSpy) does it, but it uses huge `ComImport` statements and an abstraction-heavy design, so it didn't provide an easy way to get to the details of an exception.

## Microsoft to the rescue
That's when I learned from dnSpy's readme page about [ClrMD](https://github.com/microsoft/clrmd). It contains the `Microsoft.Diagnostics.Runtime` module that can be used to do various managed debugging tasks. With this library it was easy to write my own tool that attaches passively to the process that is currently opened in IDA and lists the latest exception for each thread:

```c#
using System;
using System.Diagnostics;
using Microsoft.Diagnostics.Runtime;

namespace ReadNetException
{
    public class Program
    {
        public static void Main()
        {
            while (true)
            {
                string processName = "notepad";
                Process process = Process.GetProcessesByName(processName)[0];
                // Get process runtime
                DataTarget dt = DataTarget.AttachToProcess(process.Id, 500, AttachFlag.Passive);
                ClrInfo version = dt.ClrVersions[0];
                var dac = dt.SymbolLocator.FindBinary(version.DacInfo);
                ClrRuntime clr = version.CreateRuntime(dac);

                foreach (var thread in clr.Threads)
                {
                    Console.WriteLine(thread.OSThreadId);
                    ClrException ex = thread.CurrentException;
                    if (ex != null)
                    {
                        Console.WriteLine(ex.Type.ToString());
                        Console.WriteLine(ex.Message.ToString());
                        foreach (var frame in ex.StackTrace)
                        {
                            Console.WriteLine(frame.DisplayString);
                        }
                    }
                    Console.WriteLine();
                }

                // Wait for key press, then start over
                Console.ReadLine();
                Console.Clear();
            }
        }
    }
}
```

One caveat is that if the architecture of the process is x86, then the architecture of the ReadNetException program has to be x86, too.

The output looks as follows:

```
28304
System.Exception
Cannot find corresponding section.

4360

6976
```
One exception has been found in thread 28304. It was a generic System.Exception with the message "Cannot find corresponding section.". In this case, a strack trace was not available.

## Fazit
I am very happy that in the end the solution was as concise and easy to implement as I imagined it would be in the beginning. Even though I had to go down a deep rabbit hole to get to the important clue.