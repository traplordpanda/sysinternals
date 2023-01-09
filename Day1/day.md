## Day 1 

### local debugging
- NtSystemDebugControl 
  - undocumented full read write api 
  - used for pirating encrypted movies
  - moved to kernel 
  - replaced with KdSystemDebugControl api  
- KdSystemDebugControl 
  - originally a drm protection
  - must be run in debug mode
  - debug mode cannot stream high quality movies
- debug mode
  - SecureBoot disables /DEBUG
  - BitLocker checks for boot options prevents usage of /DEBUG without recovery key  
- Sysinternals LiveKD
  - uses kernel driver
  - creates 0-byte sparse dump
  - intercept windbg address requests and sent back physical RAM address
  - bypass KdSYstemDebugControl/NtSyStemDebugControl
  - bypass DRM 
  - Microsoft buys Sysinternals
- Return of NtSystemDebugControl
  - "live" kernel dump
  - only current point in time 
  - makes bypassing drm tedious
- Debugging symbols
  - "Public Symbols"
    - MS prevents full symbols by default
    - global variables
    - some structs
    - function names
  - "Private Symbols"
    - accidents led to leaks for private symbols
    - replaced by build script
    - some dlls come with private symbols
- setup symbols
  - elevated command line ` setx _NT_SYMBOL_PATH srv*c:\symbols*http://msdl.microsoft.com/download/symbols`
- examples of symbols
  - Win32k.sys - public symbols (type enhanced in Windows 7)
  - Ntoskrnl.exe - type enhanced symbols
  - Ole32.dll - private symbols
- setup debugging
  - download windows sdk `https://developer.microsoft.com/en-us/windows/downloads/windows-sdk/`
  - `bcdedit /debug on`
  - `bcdedit /dbgsettings`
  - check bitlocker
    - backup or suspend or turn off
    - suspend 1 free boot

### windbg
  - [WinDbgCookbook](https://github.com/TimMisiak/WinDbgCookbook)
  - `.reload` loads kernel symbols
  -  NatVis
     -  part of the new Debugger Data Model
     -  allows describing a structure into readable format 
     -  ex
        -  
        ```
        >>> dx *(nt!_EPROCESS**)nt!PsInitialSystemProcess
        Error: No type (or void) for object at Address 0xfffff8000dcfc420
        ```
        - does not work since globals do not have types. reference address to cast. 
        ```
        >>> dx *(nt!_EPROCESS**)&nt!PsInitialSystemProcess
          *(nt!_EPROCESS**)&nt!PsInitialSystemProcess                 : 0xffffe10676a8e040 [Type: _EPROCESS *]
              [+0x000] Pcb              [Type: _KPROCESS]
              [+0x438] ProcessLock      [Type: _EX_PUSH_LOCK]
              [+0x440] UniqueProcessId  : 0x4 [Type: void *]
              [+0x448] ActiveProcessLinks [Type: _LIST_ENTRY]
              [+0x458] RundownProtect   [Type: _EX_RUNDOWN_REF]
              [+0x460] Flags2           : 0xd000 [Type: unsigned long]
              [+0x460 ( 0: 0)] JobNotReallyActive : 0x0 [Type: unsigned long]
              [+0x460 ( 1: 1)] AccountingFolded : 0x0 [Type: unsigned long]
              [+0x460 ( 2: 2)] NewProcessReported : 0x0 [Type: unsigned long]
              [+0x460 ( 3: 3)] ExitProcessReported : 0x0 [Type: unsigned long]
              [+0x460 ( 4: 4)] ReportCommitChanges : 0x0 [Type: unsigned long]
          ```
      - User variables
        -  `>>> dx @$myVar = 1`
        -  `>>> dx @$myStr = "2"`
        -   ```
              >>> dx @$myStr
                @$myStr          : 2
                Length           : 0x1
            ```
      - Built in variables
        - `@$currprocess` , `@$curthread` , `@$cursession`
          
