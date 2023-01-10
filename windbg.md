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
        - `.scriptload` executes root script then executes `initializeScript`
      - winddgb preview
        - distributed on the windows store
        - support for time travel debugging TTD 
        - conditional bp
          - `bp /w @ @$evaluateDllName() kernelbase!LoadLibraryExW`
          - support for javascript bps
      - User shared data
        - limits syscalls
        - shared address across all processes
        - ```
            dx (nt!_KUSER_SHARED_DATA*)0xfffff78000000000
          (nt!_KUSER_SHARED_DATA*)0xfffff78000000000                 : 0xfffff78000000000 [Type: _KUSER_SHARED_DATA *]
          [+0x000] TickCountLowDeprecated : 0x0 [Type: unsigned long]
          [+0x004] TickCountMultiplier : 0xfa00000 [Type: unsigned long]
          [+0x008] InterruptTime    [Type: _KSYSTEM_TIME]
          [+0x014] SystemTime       [Type: _KSYSTEM_TIME]
          [+0x020] TimeZoneBias     [Type: _KSYSTEM_TIME]
          [+0x02c] ImageNumberLow   : 0x8664 [Type: unsigned short]
          [+0x02e] ImageNumberHigh  : 0x8664 [Type: unsigned short]
          [+0x030] NtSystemRoot     : "C:\WINDOWS" [Type: wchar_t [260]]
          [+0x238] MaxStackTraceDepth : 0x0 [Type: unsigned long]
          [+0x23c] CryptoExponent   : 0x0 [Type: unsigned long]
          [+0x240] TimeZoneId       : 0x1 [Type: unsigned long]
          [+0x244] LargePageMinimum : 0x200000 [Type: unsigned long]
          [+0x248] AitSamplingValue : 0x0 [Type: unsigned long]
          [+0x24c] AppCompatFlag    : 0x0 [Type: unsigned long]
          [+0x250] RNGSeedVersion   : 0x29 [Type: unsigned __int64]
          [+0x258] GlobalValidationRunlevel : 0x0 [Type: unsigned long]
          [+0x25c] TimeZoneBiasStamp : 12 [Type: long]
          [+0x260] NtBuildNumber    : 0x62b8 [Type: unsigned long]
          [+0x264] NtProductType    : NtProductWinNt (1) [Type: _NT_PRODUCT_TYPE]
          [+0x268] ProductTypeIsValid : 0x1 [Type: unsigned char]
          [+0x269] Reserved0        [Type: unsigned char [1]]
          [+0x26a] NativeProcessorArchitecture : 0x9 [Type: unsigned short]
          [+0x26c] NtMajorVersion   : 0xa [Type: unsigned long]
          [+0x270] NtMinorVersion   : 0x0 [Type: unsigned long]
          [+0x274] ProcessorFeatures [Type: unsigned char [64]]
          [+0x2b4] Reserved1        : 0x7ffeffff [Type: unsigned long]
          [+0x2b8] Reserved3        : 0x80000000 [Type: unsigned long]
          [+0x2bc] TimeSlip         : 0x0 [Type: unsigned long]
          [+0x2c0] AlternativeArchitecture : StandardDesign (0) [Type: _ALTERNATIVE_ARCHITECTURE_TYPE]
          [+0x2c4] BootId           : 0x2 [Type: unsigned long]
          [+0x2c8] SystemExpirationDate : {133392743800000000} [Type: _LARGE_INTEGER]
          [+0x2d0] SuiteMask        : 0x110 [Type: unsigned long]
          [+0x2d4] KdDebuggerEnabled : 0x1 [Type: unsigned char]
          [+0x2d5] MitigationPolicies : 0xa [Type: unsigned char]
          [+0x2d5 ( 1: 0)] NXSupportPolicy  : 0x2 [Type: unsigned char]
          [+0x2d5 ( 3: 2)] SEHValidationPolicy : 0x2 [Type: unsigned char]
          [+0x2d5 ( 5: 4)] CurDirDevicesSkippedForDlls : 0x0 [Type: unsigned char]
          [+0x2d5 ( 7: 6)] Reserved         : 0x0 [Type: unsigned char]
          [+0x2d6] CyclesPerYield   : 0x9a [Type: unsigned short]
          [+0x2d8] ActiveConsoleId  : 0x1 [Type: unsigned long]
          [+0x2dc] DismountCount    : 0x6 [Type: unsigned long]
          [+0x2e0] ComPlusPackage   : 0x1 [Type: unsigned long]
          [+0x2e4] LastSystemRITEventTickCount : 0x16f889d8 [Type: unsigned long]
          [+0x2e8] NumberOfPhysicalPages : 0x3e622d [Type: unsigned long]
          [+0x2ec] SafeBootMode     : 0x0 [Type: unsigned char]
          [+0x2ed] VirtualizationFlags : 0x0 [Type: unsigned char]
          [+0x2ee] Reserved12       [Type: unsigned char [2]]
          [+0x2f0] SharedDataFlags  : 0x10e [Type: unsigned long]
          [+0x2f0 ( 0: 0)] DbgErrorPortPresent : 0x0 [Type: unsigned long]
          [+0x2f0 ( 1: 1)] DbgElevationEnabled : 0x1 [Type: unsigned long]
          [+0x2f0 ( 2: 2)] DbgVirtEnabled   : 0x1 [Type: unsigned long]
          [+0x2f0 ( 3: 3)] DbgInstallerDetectEnabled : 0x1 [Type: unsigned long]
          [+0x2f0 ( 4: 4)] DbgLkgEnabled    : 0x0 [Type: unsigned long]
          [+0x2f0 ( 5: 5)] DbgDynProcessorEnabled : 0x0 [Type: unsigned long]
          [+0x2f0 ( 6: 6)] DbgConsoleBrokerEnabled : 0x0 [Type: unsigned long]
          [+0x2f0 ( 7: 7)] DbgSecureBootEnabled : 0x0 [Type: unsigned long]
          [+0x2f0 ( 8: 8)] DbgMultiSessionSku : 0x1 [Type: unsigned long]
          [+0x2f0 ( 9: 9)] DbgMultiUsersInSessionSku : 0x0 [Type: unsigned long]
          [+0x2f0 (10:10)] DbgStateSeparationEnabled : 0x0 [Type: unsigned long]
          [+0x2f0 (31:11)] SpareBits        : 0x0 [Type: unsigned long]
          [+0x2f4] DataFlagsPad     [Type: unsigned long [1]]
          [+0x2f8] TestRetInstruction : 0xc3 [Type: unsigned __int64]
          [+0x300] QpcFrequency     : 10000000 [Type: __int64]
          [+0x308] SystemCall       : 0x0 [Type: unsigned long]
          [+0x30c] Reserved2        : 0x0 [Type: unsigned long]
          [+0x310] FullNumberOfPhysicalPages : 0x3e622d [Type: unsigned __int64]
          [+0x318] SystemCallPad    [Type: unsigned __int64 [1]]
          [+0x320] TickCount        [Type: _KSYSTEM_TIME]
          [+0x320] TickCountQuad    : 0x1785ac4 [Type: unsigned __int64]
          [+0x320] ReservedTickCountOverlay [Type: unsigned long [3]]
          [+0x32c] TickCountPad     [Type: unsigned long [1]]
          [+0x330] Cookie           : 0x21f21e9d [Type: unsigned long]
          [+0x334] CookiePad        [Type: unsigned long [1]]
          [+0x338] ConsoleSessionForegroundProcessId : 16840 [Type: __int64]
          [+0x340] TimeUpdateLock   : 0x5530f34 [Type: unsigned __int64]
          [+0x348] BaselineSystemTimeQpc : 0x381541c582e [Type: unsigned __int64]
          [+0x350] BaselineInterruptTimeQpc : 0x381541c582e [Type: unsigned __int64]
          [+0x358] QpcSystemTimeIncrement : 0x8000000000000000 [Type: unsigned __int64]
          [+0x360] QpcInterruptTimeIncrement : 0x8000000000000000 [Type: unsigned __int64]
          [+0x368] QpcSystemTimeIncrementShift : 0x1 [Type: unsigned char]
          [+0x369] QpcInterruptTimeIncrementShift : 0x1 [Type: unsigned char]
          [+0x36a] UnparkedProcessorCount : 0x8 [Type: unsigned short]
          [+0x36c] EnclaveFeatureMask [Type: unsigned long [4]]
          [+0x37c] TelemetryCoverageRound : 0x2 [Type: unsigned long]
          [+0x380] UserModeGlobalLogger [Type: unsigned short [16]]
          [+0x3a0] ImageFileExecutionOptions : 0x0 [Type: unsigned long]
          [+0x3a4] LangGenerationCount : 0x3 [Type: unsigned long]
          [+0x3a8] Reserved4        : 0x0 [Type: unsigned __int64]
          [+0x3b0] InterruptTimeBias : 0x28f8876d25e [Type: unsigned __int64]
          [+0x3b8] QpcBias          : 0x88903bc342 [Type: unsigned __int64]
          [+0x3c0] ActiveProcessorCount : 0x8 [Type: unsigned long]
          [+0x3c4] ActiveGroupCount : 0x1 [Type: unsigned char]
          [+0x3c5] Reserved9        : 0x0 [Type: unsigned char]
          [+0x3c6] QpcData          : 0x83 [Type: unsigned short]
          [+0x3c6] QpcBypassEnabled : 0x83 [Type: unsigned char]
          [+0x3c7] QpcShift         : 0x0 [Type: unsigned char]
          [+0x3c8] TimeZoneBiasEffectiveStart : {133177482425183198} [Type: _LARGE_INTEGER]
          [+0x3d0] TimeZoneBiasEffectiveEnd : {133230780000000000} [Type: _LARGE_INTEGER]
          [+0x3d8] XState           [Type: _XSTATE_CONFIGURATION]
          [+0x720] FeatureConfigurationChangeStamp [Type: _KSYSTEM_TIME]
          [+0x72c] Spare            : 0x0 [Type: unsigned long]
          [+0x730] UserPointerAuthMask : 0x0 [Type: unsigned __int64]
          ```
