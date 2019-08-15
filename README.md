# WinDbg Notes

## Capture dump

From here: https://dougrathbone.com/blog/2014/03/20/investigating-aspnet-memory-dumps-for-idiots-like-me

The first thing you need to do is download the Windows Debugging tools to the server and install them.: http://www.microsoft.com/whdc/devtools/debugging/default.mspx

Navigate to the folder `C:\Program Files (x86)\Windows Kits\10\Debuggers\x64`.

`.\adplus.exe -p <process_id> -o c:\temp\crash-analysis -hang`

If you hare hunting first chance exceptions in `-crash` mode, adding the `-FullOnFirst` param will result in a full dump being generated (instead of a mini dump)

AdPlus supports two modes –hang for hung processes, and –crash.

When running in crash mode, AdPlus attaches a debugger to your process for the lifetime of that process. To manually detach the debugger from the process you must press CTRL+C to break the debugger. This is great if you know what to do to reproduce the problem as you can replicate it to cause the crash while AdPlus is connected.

In my experience usually these issues can be picked up from event viewer or logs for .net managed crashed - for me the only thing that draws me towards memory dumps at all are the problems where you don’t know.

As the scenario I’m walking you through involves investigating a crashed worker process after it has become unstable, we’ll be using the –hang option. This simply takes a memory dump of the entire worker process for investigation.

### Alternative to adplus: procdump

Documentation: https://docs.microsoft.com/en-us/sysinternals/downloads/procdump

```bash
.\procdump64.exe -e 1 -ma -f "*UriFormatException" 18856 D:\temp\dumps\
```

## Sequence

Copy the following dlls to the location of the .dump file:

```
C:\Windows\Microsoft.NET\[Framework or Framework64]\[Framework Version #]\SOS.dll
C:\Windows\Microsoft.NET\[Framework or Framework64]\[Framework Version #]\clr.dll
C:\Windows\Microsoft.NET\[Framework or Framework64]\[Framework Version #]\mscordacwks.dll
```

- `.sympath+ SRV*C:\symbols-microsoft*http://msdl.microsoft.com/download/symbols;C:\symbols`
- `.reload`
- `.loadby sos clr`
- `.load C:\debug\Psscor4\amd64\amd64\psscor4.dll`
- `!analyze`
- `!ClrStack`
- `!ClrStack -p` - Stack of current thread with parameters
- `!ClrStack -a` - Stack of current thread with parameters and locals
- `!threads` - shows all threads, including exceptions
- `!pe <number_of_exception>` - show information about exception (<number_of_exception> is right to the exception name)
- `~*e !clrstack` - show current stack of all threads
- `!eestack` - all managed stack traces
- `!eestack -ee` - just managed frames
- `!runaway` - This will return a list of all the threads and the amount of CPU time that they have consumed.
- `!runaway 1` - (user time)
- `!runaway 2` - (kernel time)
- `!runaway 4` - (elapsed time)
- `!dumpheap` - Show heap and prints statistics about number of occurrences
- `!dumpheap -stat` - Shows statistics about number of occurrences
- `!dumpheap -type System.IO.MemoryStream` - This will then list all of the objects on the heap that are of type MemoryStream.
- `!dumpheap -stat -live` - show alive objects
- `!dumpheap -stat -dead` - show dead objects
- `!do [address]` - inspect object
- `!gcroot [address to object]` - This will show you the reference stack of an object allowing you to then see where your memory leak is coming from.
- `!VerifyHeap`
- `!GCHandles` - get an overview of handle usage in the process
- `!heapstat` - statistics about heap
- `!objsize <address to object>` - this command takes an object as (a parameter and counts the number of objects it keeps alive as well as reports (the total size of all objects the given object keeps alive.)

- `dt nt!_GUID <address_of_object>` - dump guid. You might need to use `dt nt!_GUID <address_of_object>+8` if you are testing on a 64bit machine.

- `!refs <address>` (from SOSex) - like `gcroot`, but better
- `!SyncBlk` - dump information about anyone using lock()

## Misc

- `.cls` - clear screen
- `.symfix`
- `.chain` - show loaded modules (like sos.dll)
- `.cmdtree cmdtree.txt` - Lists common commands
- `.prefer_dml 1` - debugger markup language -> allows to click lots of the output
- `windbg -z <path to dump> -c "list of commands here"` - automate windbg commands
-`.symfix`
- `lm` - list loaded modules
- `bl` - list break points
- `bp kernel32!CreateFileW` - sets breakpoint at `CreateFileW`
- `.sympath` - shows path
- `.sympath+` - adds new sources
- `.symfix` defaults to MS symbols server
- `.reload`
- `.srcpath` - enables source debugging
- `.prefer_dml 1`
- `.loadby sos clr`
- `lm` list loaded modules
- `k` native stack trace
- `d` dump
- `sxe clr` enable clr
- `!threads` - show managed threads
- `!eestack` - all managed stack traces
- `!eestack -ee` - just managed frames
- `!address –summary` - for an overview of the memory usage, you can also run `!address –?` to get some additional details about the command
- `!eeheap -gc` - gc statistics

`dumpmt`
`!DumpMT -md <method_table_address>`

```bash
0:102> !DumpModule -md 00007ff853383e70
Unknown option: -md
0:102> !DumpMT -md 00007ff853383e70
EEClass:         00007ff853375478
Module:          00007ff84bad78f8
Name:            Sitecore.Pipelines.RenderField.ExpandLinks
mdToken:         0000000002000a06
File:            C:\Windows\Microsoft.NET\Framework64\v4.0.30319\Temporary ASP.NET Files\root\307a1276\9863d5e4\assembly\dl3\289a8c0a\00e40cde_7090d301\Sitecore.Kernel.dll
BaseSize:        0x18
ComponentSize:   0x0
Slots in VTable: 6
Number of IFaces in IFaceMap: 0
--------------------------------------
MethodDesc Table
           Entry       MethodDesc    JIT Name
00007ff8a334c080 00007ff8a2ea7fb8 PreJIT System.Object.ToString()
00007ff8a33b49e0 00007ff8a2ea7fc0 PreJIT System.Object.Equals(System.Object)
00007ff8a3425030 00007ff8a2ea7fe8 PreJIT System.Object.GetHashCode()
00007ff8a337fbc0 00007ff8a2ea8000 PreJIT System.Object.Finalize()
00007ff85d1c06d0 00007ff853383e60    JIT Sitecore.Pipelines.RenderField.ExpandLinks.Process(Sitecore.Pipelines.RenderField.RenderFieldArgs)
00007ff85d1c06b0 00007ff853383e68    JIT Sitecore.Pipelines.RenderField.ExpandLinks..ctor()
```

```bash
0:102> !DumpMD 00007ff853383e60
Method Name:  Sitecore.Pipelines.RenderField.ExpandLinks.Process(Sitecore.Pipelines.RenderField.RenderFieldArgs)
Class:        00007ff853375478
MethodTable:  00007ff853383e70
mdToken:      0000000006004dbc
Module:       00007ff84bad78f8
IsJitted:     yes
CodeAddr:     00007ff85d1c06d0
Transparency: Critical
```

```bash
!Name2EE * Sitecore.Pipelines.RenderField.ExpandLinks.Process
[...]
--------------------------------------
Module:      00007ff84bad78f8
Assembly:    Sitecore.Kernel.dll
Token:       0000000006004dbc
MethodDesc:  00007ff853383e60
Name:        Sitecore.Pipelines.RenderField.ExpandLinks.Process(Sitecore.Pipelines.RenderField.RenderFieldArgs)
JITTED Code Address: 00007ff85d1c06d0
--------------------------------------
[...]
```
