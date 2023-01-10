### windbg
  - [TimMisiak/WinDbgCookbook](https://github.com/TimMisiak/WinDbgCookbook)
  - [microsoft/WinDbg-Samples](https://github.com/microsoft/WinDbg-Samples)
  - [yardenshafir/WinDbg_Scripts](https://github.com/yardenshafir/WinDbg_Scripts)
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
        - Create your own object
          - `dx @$myObject = new { Type = 5, Name = "Process Object" }`
      - Built in variables
        - `@$currprocess` , `@$curthread` , `@$cursession`
      - Casting
        - `dx @$readQword = *(unsigned __int64*)0x7ffe0000`
        - array 
          - ```
            >>> dx (__int64 (*)[12])0xfffff78000000000
            (__int64 (*)[12])0xfffff78000000000                 : 0xfffff78000000000 [Type: __int64 (*)[12]]
            [0]              : 1125899906842624000 [Type: __int64]
            [1]              : 3159701338976 [Type: __int64]
            [2]              : 7883470045844603615 [Type: __int64]
            [3]              : 133177645865706589 [Type: __int64]
            [4]              : 180000000000 [Type: __int64]
            [5]              : -8762731210901290967 [Type: __int64]
            [6]              : 24488718114619459 [Type: __int64]
            [7]              : 22236815223029833 [Type: __int64]
            [8]              : 5439575 [Type: __int64]
            [9]              : 0 [Type: __int64]
            [10]             : 0 [Type: __int64]
            [11]             : 0 [Type: __int64]
            ```
      - function
        - `dx @$addFive = (x => x + 5)`
        - `dx @$addFive(10)`
        - anonymous types
          - `dx ((d, y)) => (x + y))("Hello", "World")`
      - LINQ (Language Integrated Query)
        - SQL Queries
        - ```
          >>> dx @$cursession.Processes.Where (p => p.KernelObject.BreakOnTermination == 0x1)
          @$cursession.Processes.Where (p => p.KernelObject.BreakOnTermination == 0x1)                
              [0x1e0]          : smss.exe [Switch To]
              [0x290]          : csrss.exe [Switch To]
              [0x378]          : wininit.exe [Switch To]
              [0x38c]          : csrss.exe [Switch To]
              [0x3cc]          : services.exe [Switch To]
              [0x31c]          : svchost.exe [Switch To]
              [0x40c]          : svchost.exe [Switch To]
              [0xcac]          : svchost.exe [Switch To]
              [0xdd0]          : svchost.exe [Switch To]
           ```
        - Can convert any linq query to a table
          ![Ref 1](.\pics\table_ex.png)
      - Creating Iterable Object
        - Useful when need to manually create objects
        - Ex using LIST_ENTRY (LIST_ENTRY is not an array)
        - ``` 
          >>> dx -r0 @$listEntry = *(nt!_LIST_ENTRY*)&(nt!PsActiveProcessHead)
          @$listEntry = *(nt!_LIST_ENTRY*)&(nt!PsActiveProcessHead)                 [Type: _LIST_ENTRY]

          >>> dx -r0 @$processList = Debugger.Utility.Collections.FromListEntry(@$listEntry, "nt!_EPROCESS", "ActiveProcessLinks")
          @$processList = Debugger.Utility.Collections.FromListEntry(@$listEntry, "nt!_EPROCESS", "ActiveProcessLinks") 

          >> dx @$processList
          @$processList                
              [0x0]            [Type: _EPROCESS]
              [0x1]            [Type: _EPROCESS]
              [0x2]            [Type: _EPROCESS]
              [0x3]            [Type: _EPROCESS]
              [0x4]            [Type: _EPROCESS]
              [0x5]            [Type: _EPROCESS]
              ...

          ```
          - Homework
            - NkProcessList parsing
            - User `dx Debugger.Utility.Control.ExecuteCommand` to find filetime and run time utility to output
      - Javascript
        - [hugsy/windbg_js_scripts](https://github.com/hugsy/windbg_js_scripts)
        - [Microsoft javascript-debugger-example-scripts](https://learn.microsoft.com/en-us/windows-hardware/drivers/debugger/javascript-debugger-example-scripts)
        - 