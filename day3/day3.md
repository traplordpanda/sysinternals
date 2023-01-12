OS Design
=====
- SMEP and SMAP (supervisor mode execution prevention)
  - requires ivy bridge processor
  - cpu will trigger exception if RIP is ring 3 PTE but CS is ring 0
  - trivial to bypass through kernel heap spraying or return oriented programming (rop)
  - hard to implement in windows
- Hypervisor based CPU protection
  - Hyper-V cpu protection features
    - non priviliged instruction execution prevention (NPIEP)
      - restricts the use of SGDT, SIDT, SLDT, and SMSW  
        these instructions are only useful in ring 3 if you are trying to leak a kernel address
      - generates #UD exception
      - made cpu vendors look bad. on Tiger Lake implemented User Mode Instruction Prevention (UMIP)
    - restricted user mode (RUM)
      - Um pages are always RWX in EPTE (page table entry) tricks cpu to run arbitrary code as CPL0 by modifying PTE
      - made intel look bad. on Kaby Lake cpus OS uses MBEC (mode-based execution control)
        
        adds separate KNX vs UNX bits in EPTE to avoid EPT switch
        ```
        lkd> dx @$myUserdat->SystemCall
        @$myUserdat->SystemCall : 0x0 [Type: unsigned long]
        ```
        0x0 does not support MBEC, 0x1 MBEC support
- Software Guard Extensions (SGX)
  - supported since Skylake 
  - creates Enclaves to run in an isolated execution and memory context
  - enclaves have in memory encryption
  - must register with intel and all code in enclave must be signed
  - mostly only used for DRM. consumes a chunk of contigious memory
  - deprecated in 11th and 12th Generation Alder Lake. Still supported by xeon mostly server cpus.
- Hypervisor Enclaves
  - Virtual Secure Mode (VSM)
    added in fall creator's update in Windows 10. can launch dedicated trustlet.
    - special PE file. IAT has IMAGE_ENCLAVE_CONFIG directory and IMAGE_ENCLAVE_IMPORT descriptor
    ```
    lkd> dx @$cursession.Processes.Where(p => p.KernelObject.Pcb.SecureState.SecureHandle != 0)
    @$cursession.Processes.Where(p => p.KernelObject.Pcb.SecureState.SecureHandle != 0)                
        [0x48]           : Secure System [Switch To]
        [0x3c8]          : LsaIso.exe [Switch To]
        [0xcec]          : SgrmBroker.exe [Switch To]
    ```
    ```
    lkd> dx -g @$cursession.Processes.Where(p => p.KernelObject.Pcb.SecureState.SecureHandle != 0).Select(p => new {Name = p.Name, SecureProcess = p.KernelObject.Pcb.SecureState.Flags.SecureProcess})
    ==================================================
    =            = Name              = SecureProcess =
    ==================================================
    = [0x48]     - Secure System     - 0x1           =
    = [0x3c8]    - LsaIso.exe        - 0x1           =
    = [0xcec]    - SgrmBroker.exe    - 0x0           =
    ==================================================
    ```

    
- Intel CET
  - attempt to solve return oriented programming (ROP)
  - combination of shadow stacks & indirect branch tracing
  - shadow stack
    - checks return address in shadow stack. return address shouldn't change inside in func. raises exception int 21.
    - officially supported for usermode in windows 20h1. kernel support added in windows 11 21h2
    - creates concept of shadow IST for interrupt stacks
    - optionally adds instructions for reading/writing to shadow stack
  - indirect branch tracing
    - ENDBRANCH state machine to track branches and return to a branch
    - currently not enabled in any OS
  - only works for processes that have enabled CET. if not enabled by process kernel will edit shadow stack to reflect different return address.
  - sets KTHREAD CetShadowStack field if CET shadow inabled
- Context Validation
  - kernel can change user context so checks added for 
    - raising exception and calling exception handler
    - resuming after exception
    - queuing and APC (Asynchronous Procedure Calls)
    - setting context threads
  - stack pointer checks using KeVerifyContextXStateCetU
  - `dx @$curprocess.KernelObject.MitigationFlags2Values`
    ```
    dx -id 0,0,ffffaf8f1df5c340 -r1 (*((ntkrnlmp!_EPROCESS *)0xffffaf8f1df5c340)).MitigationFlags2Values
    (*((ntkrnlmp!_EPROCESS *)0xffffaf8f1df5c340)).MitigationFlags2Values                 [Type: <anonymous-tag>]
        [+0x000 ( 0: 0)] EnableExportAddressFilter : 0x0 [Type: unsigned long]
        [+0x000 ( 1: 1)] AuditExportAddressFilter : 0x0 [Type: unsigned long]
        [+0x000 ( 2: 2)] EnableExportAddressFilterPlus : 0x0 [Type: unsigned long]
        [+0x000 ( 3: 3)] AuditExportAddressFilterPlus : 0x0 [Type: unsigned long]
        [+0x000 ( 4: 4)] EnableRopStackPivot : 0x0 [Type: unsigned long]
        [+0x000 ( 5: 5)] AuditRopStackPivot : 0x0 [Type: unsigned long]
        [+0x000 ( 6: 6)] EnableRopCallerCheck : 0x0 [Type: unsigned long]
        [+0x000 ( 7: 7)] AuditRopCallerCheck : 0x0 [Type: unsigned long]
        [+0x000 ( 8: 8)] EnableRopSimExec : 0x0 [Type: unsigned long]
        [+0x000 ( 9: 9)] AuditRopSimExec  : 0x0 [Type: unsigned long]
        [+0x000 (10:10)] EnableImportAddressFilter : 0x0 [Type: unsigned long]
        [+0x000 (11:11)] AuditImportAddressFilter : 0x0 [Type: unsigned long]
        [+0x000 (12:12)] DisablePageCombine : 0x0 [Type: unsigned long]
        [+0x000 (13:13)] SpeculativeStoreBypassDisable : 0x0 [Type: unsigned long]
        [+0x000 (14:14)] CetUserShadowStacks : 0x0 [Type: unsigned long]
        [+0x000 (15:15)] AuditCetUserShadowStacks : 0x0 [Type: unsigned long]
        [+0x000 (16:16)] AuditCetUserShadowStacksLogged : 0x0 [Type: unsigned long]
        [+0x000 (17:17)] UserCetSetContextIpValidation : 0x0 [Type: unsigned long]
        [+0x000 (18:18)] AuditUserCetSetContextIpValidation : 0x0 [Type: unsigned long]
        [+0x000 (19:19)] AuditUserCetSetContextIpValidationLogged : 0x0 [Type: unsigned long]
        [+0x000 (20:20)] CetUserShadowStacksStrictMode : 0x0 [Type: unsigned long]
        [+0x000 (21:21)] BlockNonCetBinaries : 0x0 [Type: unsigned long]
        [+0x000 (22:22)] BlockNonCetBinariesNonEhcont : 0x0 [Type: unsigned long]
        [+0x000 (23:23)] AuditBlockNonCetBinaries : 0x0 [Type: unsigned long]
        [+0x000 (24:24)] AuditBlockNonCetBinariesLogged : 0x0 [Type: unsigned long]
        [+0x000 (25:25)] Reserved1        : 0x0 [Type: unsigned long]
        [+0x000 (26:26)] Reserved2        : 0x0 [Type: unsigned long]
        [+0x000 (27:27)] Reserved3        : 0x0 [Type: unsigned long]
        [+0x000 (28:28)] Reserved4        : 0x0 [Type: unsigned long]
        [+0x000 (29:29)] Reserved5        : 0x0 [Type: unsigned long]
        [+0x000 (30:30)] CetDynamicApisOutOfProcOnly : 0x1 [Type: unsigned long]
        [+0x000 (31:31)] UserCetSetContextIpValidationRelaxedMode : 0x0 [Type: unsigned long]
    ```
    [R.I.P ROP: CET Internals in Windows 20H1](https://windows-internals.com/cet-on-windows/)

    [CET Updates – Dynamic Address Ranges](https://windows-internals.com/cet-updates-dynamic-address-ranges/)

    [CET Updates – CET on Xanax](https://windows-internals.com/cet-updates-cet-on-xanax/)

OS Hardware Architecture
========
-   Device drivers have two mechanisms to communicate with hardware
-   polling - software constantly queries for hardware change
-   push notifications - hardware sends interrupt (mostly used now)

### Interrupts
- hardware interrupts are electrical signals. hardware raises or lowers voltage to notify cpu
- interrupt controller (PIC or APIC) handles prioritization and masking and assigns an interrupt number to each interrupt line (IV interrupt vector)
- recieved directly by CPU
- Once cpu recieves signal calls iterrupt service routine (ISR)
  - x86/x64 uses Interrupt Dispatch Table (IDT)
  - ARM uses single entry in Exception Vector Table
```
dx (nt!_KINTERRUPT*)0xfffff8070a8f37e0
(nt!_KINTERRUPT*)0xfffff8070a8f37e0                 : 0xfffff8070a8f37e0 [Type: _KINTERRUPT *]
    [+0x000] Type             : 22 [Type: short]
    [+0x002] Size             : 288 [Type: short]
    [+0x008] InterruptListEntry [Type: _LIST_ENTRY]
    [+0x018] ServiceRoutine   : 0xfffff8070a0bc760 : ntkrnlmp!HalpPerfInterrupt+0x0 [Type: unsigned char (__cdecl*)(_KINTERRUPT *,void *)]
    [+0x020] MessageServiceRoutine : 0x0 : 0x0 [Type: unsigned char (__cdecl*)(_KINTERRUPT *,void *,unsigned long)]
    [+0x028] MessageIndex     : 0x0 [Type: unsigned long]
    [+0x030] ServiceContext   : 0x0 [Type: void *]
    [+0x038] SpinLock         : 0x0 [Type: unsigned __int64]
    [+0x040] TickCount        : 0x0 [Type: unsigned long]
    [+0x048] ActualLock       : 0xfffffffffffffffd : Unable to read memory at Address 0xfffffffffffffffd [Type: unsigned __int64 *]
    [+0x050] DispatchAddress  : 0xfffff80709ffb9d0 : ntkrnlmp!KiInterruptDispatchNoLockNoEtw+0x0 [Type: void (__cdecl*)()]
    [+0x058] Vector           : 0xfe [Type: unsigned long]
    [+0x05c] Irql             : 0xf [Type: unsigned char]
    [+0x05d] SynchronizeIrql  : 0xf [Type: unsigned char]
    [+0x05e] FloatingSave     : 0x0 [Type: unsigned char]
    [+0x05f] Connected        : 0x1 [Type: unsigned char]
    [+0x060] Number           : 0x0 [Type: unsigned long]
    [+0x064] ShareVector      : 0x0 [Type: unsigned char]
    [+0x065] EmulateActiveBoth : 0x0 [Type: unsigned char]
    [+0x066] ActiveCount      : 0x0 [Type: unsigned short]
    [+0x068] InternalState    : 0 [Type: long]
    [+0x06c] Mode             : Latched (1) [Type: _KINTERRUPT_MODE]
    [+0x070] Polarity         : InterruptPolarityUnknown (0) [Type: _KINTERRUPT_POLARITY]
    [+0x074] ServiceCount     : 0x0 [Type: unsigned long]
    [+0x078] DispatchCount    : 0x0 [Type: unsigned long]
    [+0x080] PassiveEvent     : 0x0 [Type: _KEVENT *]
    [+0x088] TrapFrame        : 0xffffae888bb391e0 [Type: _KTRAP_FRAME *]
    [+0x090] DisconnectData   : 0x0 [Type: void *]
    [+0x098] ServiceThread    : 0x0 [Type: _KTHREAD *]
    [+0x0a0] ConnectionData   : 0x0 [Type: _INTERRUPT_CONNECTION_DATA *]
    [+0x0a8] IntTrackEntry    : 0x0 [Type: void *]
    [+0x0b0] IsrDpcStats      [Type: _ISRDPCSTATS]
    [+0x110] RedirectObject   : 0x0 [Type: void *]
    [+0x118] PhysicalDeviceObject : 0x0 [Type: void *]
```
```
dx @$prcb -> InterruptObject.Where(p => p).Select(p => (nt!_KINTERRUPT*)p)
@$prcb -> InterruptObject.Where(p => p).Select(p => (nt!_KINTERRUPT*)p)                
    [53]             : 0xfffff8070a8f3000 [Type: _KINTERRUPT *]
    [54]             : 0xfffff8070a8f3240 [Type: _KINTERRUPT *]
    [80]             : 0xffffc50155951c80 [Type: _KINTERRUPT *]
    [81]             : 0xffffc501559518c0 [Type: _KINTERRUPT *]
    [96]             : 0xffffc50155951000 [Type: _KINTERRUPT *]
    [97]             : 0xffffc50155951780 [Type: _KINTERRUPT *]
    [112]            : 0xffffc50155951140 [Type: _KINTERRUPT *]
    [113]            : 0xffffc50155951a00 [Type: _KINTERRUPT *]
    [129]            : 0xffffc50155951b40 [Type: _KINTERRUPT *]
    [145]            : 0xffffc50155951640 [Type: _KINTERRUPT *]
    [161]            : 0xffffc501559513c0 [Type: _KINTERRUPT *]
    [176]            : 0xffffc50155951dc0 [Type: _KINTERRUPT *]
    [177]            : 0xffffc50155951280 [Type: _KINTERRUPT *]
    [178]            : 0xffffc50155951500 [Type: _KINTERRUPT *]
    [209]            : 0xfffff8070a8f3a20 [Type: _KINTERRUPT *]
    [210]            : 0xfffff8070a8f3900 [Type: _KINTERRUPT *]
    [215]            : 0xfffff8070a8f36c0 [Type: _KINTERRUPT *]
    [216]            : 0xfffff8070a8f3480 [Type: _KINTERRUPT *]
    [223]            : 0xfffff8070a8f3360 [Type: _KINTERRUPT *]
    [226]            : 0xfffff8070a8f35a0 [Type: _KINTERRUPT *]
    [227]            : 0xfffff8070a8f3120 [Type: _KINTERRUPT *]
    [254]            : 0xfffff8070a8f37e0 [Type: _KINTERRUPT *]
```
random interrupt selected
```
x -r1 ((ntkrnlmp!_KINTERRUPT *)0xffffc50155951c80)
((ntkrnlmp!_KINTERRUPT *)0xffffc50155951c80)                 : 0xffffc50155951c80 [Type: _KINTERRUPT *]
    [+0x000] Type             : 22 [Type: short]
    [+0x002] Size             : 288 [Type: short]
    [+0x008] InterruptListEntry [Type: _LIST_ENTRY]
    [+0x018] ServiceRoutine   : 0xfffff80709f3cb50 : ntkrnlmp!KiInterruptMessageDispatch+0x0 [Type: unsigned char (__cdecl*)(_KINTERRUPT *,void *)]
    [+0x020] MessageServiceRoutine : 0xfffff8070ede1580 : storport!RaidpAdapterMSIInterruptRoutine+0x0 [Type: unsigned char (__cdecl*)(_KINTERRUPT *,void *,unsigned long)]
    [+0x028] MessageIndex     : 0x0 [Type: unsigned long]
    [+0x030] ServiceContext   : 0xffffaf8f18fa71a0 [Type: void *]
    [+0x038] SpinLock         : 0x0 [Type: unsigned __int64]
    [+0x040] TickCount        : 0x0 [Type: unsigned long]
    [+0x048] ActualLock       : 0xffffaf8f18f09340 : 0x0 [Type: unsigned __int64 *]
    [+0x050] DispatchAddress  : 0xfffff80709ffb250 : ntkrnlmp!KiInterruptDispatch+0x0 [Type: void (__cdecl*)()]
    [+0x058] Vector           : 0x50 [Type: unsigned long]
    [+0x05c] Irql             : 0x5 [Type: unsigned char]
    [+0x05d] SynchronizeIrql  : 0x5 [Type: unsigned char]
    [+0x05e] FloatingSave     : 0x0 [Type: unsigned char]
    [+0x05f] Connected        : 0x1 [Type: unsigned char]
    [+0x060] Number           : 0x0 [Type: unsigned long]
    [+0x064] ShareVector      : 0x1 [Type: unsigned char]
    [+0x065] EmulateActiveBoth : 0x0 [Type: unsigned char]
    [+0x066] ActiveCount      : 0x0 [Type: unsigned short]
    [+0x068] InternalState    : 4 [Type: long]
    [+0x06c] Mode             : Latched (1) [Type: _KINTERRUPT_MODE]
    [+0x070] Polarity         : InterruptPolarityUnknown (0) [Type: _KINTERRUPT_POLARITY]
    [+0x074] ServiceCount     : 0x0 [Type: unsigned long]
    [+0x078] DispatchCount    : 0x0 [Type: unsigned long]
    [+0x080] PassiveEvent     : 0x0 [Type: _KEVENT *]
    [+0x088] TrapFrame        : 0xfffff80708c2ead0 [Type: _KTRAP_FRAME *]
    [+0x090] DisconnectData   : 0x0 [Type: void *]
    [+0x098] ServiceThread    : 0x0 [Type: _KTHREAD *]
    [+0x0a0] ConnectionData   : 0xffffaf8f18f09350 [Type: _INTERRUPT_CONNECTION_DATA *]
    [+0x0a8] IntTrackEntry    : 0xffffaf8f18f265e0 [Type: void *]
    [+0x0b0] IsrDpcStats      [Type: _ISRDPCSTATS]
    [+0x110] RedirectObject   : 0x0 [Type: void *]
    [+0x118] PhysicalDeviceObject : 0xffffaf8f18f1b360 [Type: void *]
```
```
lkd> dx -g @$myInterrupts.Select(p=> new {Vec = p->Vector, IRQL = p->Irql, Service = p->ServiceRoutine})
==============================================================================================
=          = Vec     = IRQL   = (+) Service                                                  =
==============================================================================================
= [53]     - 0x35    - 0x5    - 0xfffff8070a0cfe90 : ntkrnlmp!HalpInterruptCmciService+0x0   =
= [54]     - 0x36    - 0x5    - 0xfffff8070a0cfe90 : ntkrnlmp!HalpInterruptCmciService+0x0   =
= [80]     - 0x50    - 0x5    - 0xfffff80709f3cb50 : ntkrnlmp!KiInterruptMessageDispatch+0x0 =
= [81]     - 0x51    - 0x5    - 0xfffff8070e5d3c30 : Wdf01000!FxInterrupt::_InterruptThun... =
= [96]     - 0x60    - 0x6    - 0xfffff80724cb6790 : i8042prt!I8042KeyboardInterruptServi... =
= [97]     - 0x61    - 0x6    - 0xfffff8070e5d3c30 : Wdf01000!FxInterrupt::_InterruptThun... =
= [112]    - 0x70    - 0x7    - 0xfffff80724cb8890 : i8042prt!I8042MouseInterruptService+0x0 =
= [113]    - 0x71    - 0x7    - 0xfffff8071f3e7360 : USBPORT!USBPORT_InterruptService+0x0    =
= [129]    - 0x81    - 0x8    - 0xfffff80720b02760 : HDAudBus!HdaController::Isr+0x0         =
= [145]    - 0x91    - 0x9    - 0xfffff8070f2f6ed0 : ndis!ndisMiniportIsr+0x0                =
= [161]    - 0xa1    - 0xa    - 0xfffff8070f2f6ed0 : ndis!ndisMiniportIsr+0x0                =
= [176]    - 0xb0    - 0xb    - 0xfffff8070e905d30 : ACPI!ACPIInterruptServiceRoutine+0x0    =
= [177]    - 0xb1    - 0xb    - 0xfffff8071f3e7360 : USBPORT!USBPORT_InterruptService+0x0    =
= [178]    - 0xb2    - 0xb    - 0xfffff80720b02760 : HDAudBus!HdaController::Isr+0x0         =
= [209]    - 0xd1    - 0xd    - 0xfffff80709edb140 : ntkrnlmp!HalpTimerClockInterrupt+0x0    =
= [210]    - 0xd2    - 0xd    - 0xfffff80709ed24a0 : ntkrnlmp!HalpTimerClockIpiRoutine+0x0   =
= [215]    - 0xd7    - 0xf    - 0xfffff8070a0cfed0 : ntkrnlmp!HalpInterruptRebootService+0x0 =
= [216]    - 0xd8    - 0xf    - 0xfffff8070a0cff50 : ntkrnlmp!HalpInterruptStubService+0x0   =
= [223]    - 0xdf    - 0xf    - 0xfffff8070a0cff20 : ntkrnlmp!HalpInterruptSpuriousServic... =
= [226]    - 0xe2    - 0xf    - 0xfffff80709fa4990 : ntkrnlmp!HalpInterruptLocalErrorServ... =
= [227]    - 0xe3    - 0xe    - 0xfffff8070a0cfeb0 : ntkrnlmp!HalpInterruptDeferredRecove... =
= [254]    - 0xfe    - 0xf    - 0xfffff8070a0bc760 : ntkrnlmp!HalpPerfInterrupt+0x0          =
==============================================================================================
```
- before interrupt is finished writing trap frame CLI eflag is set to disable interrupts to prevent stack corruption
- seperate stacks
  - NMI (non maskable interrupts)
  - MCA (Machine check abort)
  - double fault interrupt
  - debug interrupt
`01:	fffff8070a615180 nt!KiDebugTrapOrFaultShadow	Stack = 0xFFFFF80708C269D0` uses seperate stack due to non avoidable bug 
- all interrupts that use seperate stack 
```
lkd> dx @$pcr->TssBase->Ist
@$pcr->TssBase->Ist                 [Type: unsigned __int64 [8]]
    [0]              : 0x0 [Type: unsigned __int64]
    [1]              : 0xfffff80708c263d0 [Type: unsigned __int64]
    [2]              : 0xfffff80708c265d0 [Type: unsigned __int64]
    [3]              : 0xfffff80708c267d0 [Type: unsigned __int64]
    [4]              : 0xfffff80708c269d0 [Type: unsigned __int64]
    [5]              : 0x0 [Type: unsigned __int64]
    [6]              : 0x0 [Type: unsigned __int64]
    [7]              : 0x0 [Type: unsigned __int64]
```
- unused slots for seperate stacks maybe possible to insert your own
- Deffered Procedure Calls KDPC
  - used for handling device work that entails more of 25us of time. 
  - `!dpcs` dmps per cpu queue usually won't see anything. must be remote debug
- Clock Interrupt & Timers
  -   Current procs timer resolution requested
    ```
    lkd> dx -g @$cursession.Processes.Where(p => p.KernelObject.SmallestTimerResolution > 0).Select(p => new {Name=p->Name, Requesteds=p.KernelObject.RequestedTimerResolution, smallest=p.KernelObject.SmallestTimerResolution}),d
    =============================================================================
    =            = Name                                 = Requesteds = smallest =
    =============================================================================
    = [7568]     - SearchApp.exe                        - 0          - 40000    =
    = [7928]     - Code.exe                             - 0          - 10000    =
    = [9040]     - chrome.exe                           - 0          - 10000    =
    = [10204]    - chrome.exe                           - 0          - 10000    =
    = [9400]     - chrome.exe                           - 0          - 10000    =
    = [8064]     - chrome.exe                           - 0          - 10000    =
    = [8024]     - chrome.exe                           - 0          - 10000    =
    = [8052]     - chrome.exe                           - 0          - 10000    =
    = [6968]     - Code.exe                             - 0          - 10000    =
    = [9972]     - Code.exe                             - 0          - 10000    =
    = [7956]     - Code.exe                             - 0          - 10000    =
    = [2336]     - Discord.exe                          - 0          - 10000    =
    = [6460]     - Code.exe                             - 0          - 10000    =
    = [8316]     - Discord.exe                          - 0          - 10000    =
    = [560]      - Discord.exe                          - 0          - 10000    =
    = [1564]     - Discord.exe                          - 10000      - 10000    =
    = [10116]    - msedge.exe                           - 0          - 10000    =
    = [1120]     - msedge.exe                           - 0          - 10000    =
    = [9896]     - msedge.exe                           - 0          - 10000    =
    = [584]      - msedge.exe                           - 0          - 10000    =
    = [3852]     - DbgX.Shell.exe                       - 10000      - 10000    =
    = [5480]     - Code.exe                             - 0          - 10000    =
    = [11692]    - msedge.exe                           - 0          - 10000    =
    = [220]      - msedge.exe                           - 0          - 10000    =
    = [13176]    - msedge.exe                           - 0          - 10000    =
    = [3024]     - msedge.exe                           - 0          - 10000    =
    = [14104]    - msedge.exe                           - 0          - 10000    =
    = [12372]    - Code.exe                             - 0          - 10000    =
    = [14220]    - devenv.exe                           - 0          - 10000    =
    = [13832]    - ServiceHub.ThreadedWaitDialog.exe    - 10000      - 10000    =
    = [7484]     - chrome.exe                           - 0          - 10000    =
    = [15340]    - chrome.exe                           - 10000      - 10000    =
    = [7384]     - chrome.exe                           - 0          - 10000    =
    = [20188]    - msedge.exe                           - 0          - 10000    =
    = [21020]    - chrome.exe                           - 0          - 10000    =
    = [19672]    - chrome.exe                           - 0          - 10000    =
    = [17064]    - chrome.exe                           - 0          - 10000    =
    = [20160]    - chrome.exe                           - 0          - 10000    =
    = [4772]     - chrome.exe                           - 0          - 10000    =
    = [11560]    - chrome.exe                           - 0          - 10000    =
    = [19576]    - chrome.exe                           - 0          - 10000    =
    = [18716]    - chrome.exe                           - 0          - 10000    =
    = [15720]    - SearchApp.exe                        - 0          - 40000    =
    = [11332]    - chrome.exe                           - 0          - 10000    =
    = [20076]    - chrome.exe                           - 0          - 10000    =
    = [1384]     - chrome.exe                           - 0          - 10000    =
    = [10380]    - chrome.exe                           - 0          - 10000    =
    = [12188]    - chrome.exe                           - 0          - 10000    =
    = [12896]    - chrome.exe                           - 0          - 10000    =
    = [21148]    - msedge.exe                           - 0          - 10000    =
    = [11208]    - chrome.exe                           - 0          - 10000    =
    = [14172]    - chrome.exe                           - 0          - 10000    =
    = [21168]    - chrome.exe                           - 0          - 10000    =
    = [17092]    - chrome.exe                           - 0          - 10000    =
    = [12880]    - chrome.exe                           - 0          - 10000    =
    = [15004]    - chrome.exe                           - 0          - 10000    =
    = [20556]    - chrome.exe                           - 0          - 10000    =
    = [21992]    - chrome.exe                           - 0          - 10000    =
    = [19436]    - chrome.exe                           - 0          - 10000    =
    =============================================================================
    ```
    ```
    !timer
    Dump system timers

    Interrupt time: 7d9c029d 0000045d [ 1/11/2023 11:39:15.492]

    PROCESSOR 0 (nt!_KTIMER_TABLE fffff8022bef1d80 - Type 0 - High precision)
    List Timer             Interrupt Low/High Fire Time                  DPC/thread
    69 ffff9b0f57abe180    85159b8a 0000045d [ 1/11/2023 11:39:28.033]  thread ffff9b0f57abe080 
        ffff9b0f602af180    85159b8a 0000045d [ 1/11/2023 11:39:28.033]  thread ffff9b0f602af080 
        ffff9b0f571c6180    85159b8a 0000045d [ 1/11/2023 11:39:28.033]  thread ffff9b0f571c6080 
        ffff9b0f4add4180    85159b8a 0000045d [ 1/11/2023 11:39:28.033]  thread ffff9b0f4add4080 
    121 ffff9b0f671e7180    89e4c454 0000045d [ 1/11/2023 11:39:36.102]  thread ffff9b0f671e7080 
    ...
    ```
    - Windows 10 KTIMER2    
      - Kernel APIs for high resolution 
      - uses red/black trees instead of 256 hash buckets
      - technically can use in user mode but not directly documented
      - callback is no longer a DPC routine
- Asynchronous Procedure Calls
  - KAPC (Kernel APC)
  -  apc example
    ```
    dx (nt!_KAPC*)0xffffaf8f21685308
    (nt!_KAPC*)0xffffaf8f21685308                 : 0xffffaf8f21685308 [Type: _KAPC *]
        [+0x000] Type             : 0x12 [Type: unsigned char]
        [+0x001] SpareByte0       : 0x0 [Type: unsigned char]
        [+0x002] Size             : 0x58 [Type: unsigned char]
        [+0x003] SpareByte1       : 0x6 [Type: unsigned char]
        [+0x004] SpareLong0       : 0x0 [Type: unsigned long]
        [+0x008] Thread           : 0xffffaf8f21685080 [Type: _KTHREAD *]
        [+0x010] ApcListEntry     [Type: _LIST_ENTRY]
        [+0x020] KernelRoutine    : 0xfffff80709f99090 : ntkrnlmp!EmpCheckErrataList+0x0 [Type: void (__cdecl*)(_KAPC *,void (__cdecl**)(void *,void *,void *),void * *,void * *,void * *)]
        [+0x028] RundownRoutine   : 0xfffff80709f99090 : ntkrnlmp!EmpCheckErrataList+0x0 [Type: void (__cdecl*)(_KAPC *)]
        [+0x030] NormalRoutine    : 0xfffff80709e8ca70 : ntkrnlmp!KiSchedulerApc+0x0 [Type: void (__cdecl*)(void *,void *,void *)]
        [+0x020] Reserved         [Type: void * [3]]
        [+0x038] NormalContext    : 0xffffaf8f21685080 [Type: void *]
        [+0x040] SystemArgument1  : 0x0 [Type: void *]
        [+0x048] SystemArgument2  : 0x0 [Type: void *]
        [+0x050] ApcStateIndex    : 0 [Type: char]
        [+0x051] ApcMode          : 0 [Type: char]
        [+0x052] Inserted         : 0x1 [Type: unsigned char]
    ```
Syscalls
====
- runs through ntdll
- microsoft changes them sometimes
- int 2eh in windows
- check if hyper-v uses syscall or int 2eh
-   ```
    dx @$myUserdat->SystemCall
    @$myUserdat->SystemCall : 0x0 [Type: unsigned long]
    ```
- win32k uses it's own syscall 
    ```
    lkd> dps nt!KeServiceDescriptorTableShadow
    fffff807`0a8fca40  fffff807`09cc7a40 nt!KiServiceTable
    fffff807`0a8fca48  00000000`00000000
    fffff807`0a8fca50  00000000`000001d7
    fffff807`0a8fca58  fffff807`09cc81a0 nt!KiArgumentTable
    fffff807`0a8fca60  fffff1dd`068f8000 win32k!W32pServiceTable
    fffff807`0a8fca68  00000000`00000000
    fffff807`0a8fca70  00000000`00000524
    fffff807`0a8fca78  fffff1dd`068f99bc win32k!W32pArgumentTable
    ```
- KiServiceTable is an array of syscalls. first 7 bits are offset from KiServiceTable.
- last bit is number of arguments

```
dd nt!KiServiceTable
fffff802`2e0d3e40  01a78e04 02844a00 07234d02 09ccf500
fffff802`2e0d3e50  05feb100 03613700 06bf4005 06054d06
fffff802`2e0d3e60  06c18505 064cb501 0683e700 06548800
fffff802`2e0d3e70  066c6c00 063aeb00 0683fa00 05fece00
fffff802`2e0d3e80  0753f401 060ac201 06304f00 069fb302
fffff802`2e0d3e90  07115100 07084900 05fed301 062cad02
fffff802`2e0d3ea0  063e4402 06594001 07074d01 0723ed05
fffff802`2e0d3eb0  06b70100 06a20f03 063d9f00 09025a00
```
```
dx @$table = (int (*)[0x1e8])&nt!KiServiceTable
dx @$table->Select(o => (void(*)())(((__int64)&nt!KiServiceTable) + (o >> 4)))
lkd> dx -g @$table->Select(o => new {Syscall = (void(*)())(((__int64)&nt!KiServiceTable) + (o >> 4)), Arguments = (o & 0xf) + 4 })
=======================================================================================
=          = (+) Syscall                                                  = Arguments =
=======================================================================================
= [0]      - 0xfffff80709f3b740 : ntkrnlmp!NtAccessCheck+0x0              - 8         =
= [1]      - 0xfffff80709f44110 : ntkrnlmp!NtWorkerFactoryWorkerReady+0x0 - 4         =
= [2]      - 0xfffff8070a311f80 : ntkrnlmp!NtAcceptConnectPort+0x0        - 6         =
= [3]      - 0xfffff8070a4d7030 : ntkrnlmp!NtMapUserPhysicalPagesScatt... - 4         =
= [4]      - 0xfffff8070a2083b0 : ntkrnlmp!NtWaitForSingleObject+0x0      - 4         =
= [5]      - 0xfffff80709ffd970 : ntkrnlmp!NtCallbackReturn+0x0           - 4         =
= [6]      - 0xfffff8070a1f0ff0 : ntkrnlmp!NtReadFile+0x0                 - 9         =
= [7]      - 0xfffff8070a2186f0 : ntkrnlmp!NtDeviceIoControlFile+0x0      - 10        =
= [8]      - 0xfffff8070a1f03c0 : ntkrnlmp!NtWriteFile+0x0                - 9         =
= [9]      - 0xfffff8070a2e4320 : ntkrnlmp!NtRemoveIoCompletion+0x0       - 5         =
= [10]     - 0xfffff8070a2e3940 : ntkrnlmp!NtReleaseSemaphore+0x0         - 4         =
= [11]     - 0xfffff8070a20fee0 : ntkrnlmp!NtReplyWaitReceivePort+0x0     - 4         =
= [12]     - 0xfffff8070a2bf050 : ntkrnlmp!NtReplyPort+0x0                - 4         =
= [13]     - 0xfffff8070a2175d0 : ntkrnlmp!NtSetInformationThread+0x0     - 4         =
= [14]     - 0xfffff8070a216cc0 : ntkrnlmp!NtSetEvent+0x0                 - 4         =
= [15]     - 0xfffff8070a2080e0 : ntkrnlmp!NtClose+0x0                    - 4         =
= [16]     - 0xfffff8070a1e1310 : ntkrnlmp!NtQueryObject+0x0              - 5         =
= [17]     - 0xfffff8070a21bca0 : ntkrnlmp!NtQueryInformationFile+0x0     - 5         =
= [18]     - 0xfffff8070a2f1e60 : ntkrnlmp!NtOpenKey+0x0                  - 4         =
= [19]     - 0xfffff8070a1f9620 : ntkrnlmp!NtEnumerateValueKey+0x0        - 6         =
= [20]     - 0xfffff8070a246e00 : ntkrnlmp!NtFindAtom+0x0                 - 4         =
= [21]     - 0xfffff8070a2fe0d0 : ntkrnlmp!NtQueryDefaultLocale+0x0       - 4         =
= [22]     - 0xfffff8070a22e590 : ntkrnlmp!NtQueryKey+0x0                 - 5         =
= [23]     - 0xfffff8070a22ec30 : ntkrnlmp!NtQueryValueKey+0x0            - 6         =
= [24]     - 0xfffff8070a24c960 : ntkrnlmp!NtAllocateVirtualMemory+0x0    - 6         =
= [25]     - 0xfffff8070a1e3830 : ntkrnlmp!NtQueryInformationProcess+0x0  - 5         =
= [26]     - 0xfffff8070a2f27f0 : ntkrnlmp!NtWaitForMultipleObjects32+0x0 - 5         =
= [27]     - 0xfffff8070a2f9e80 : ntkrnlmp!NtWriteFileGather+0x0          - 9         =
= [28]     - 0xfffff8070a24da60 : ntkrnlmp!NtSetInformationProcess+0x0    - 4         =
= [29]     - 0xfffff8070a29dc10 : ntkrnlmp!NtCreateKey+0x0                - 7         =
= [30]     - 0xfffff8070a28d5f0 : ntkrnlmp!NtFreeVirtualMemory+0x0        - 4         =
= [31]     - 0xfffff8070a4c2070 : ntkrnlmp!NtImpersonateClientOfPort+0x0  - 4         =
= [32]     - 0xfffff8070a216bb0 : ntkrnlmp!NtReleaseMutant+0x0            - 4         =
```
- misc
  - with arbitrary write could find kthread and edit previous mode to kernel mode 

Core & Executive Machanisms
====
Object Manager
  - OBJECT_TYPE STRUCT
    ```
    lkd> dt nt!_OBJECT_TYPE
    +0x000 TypeList         : _LIST_ENTRY
    +0x010 Name             : _UNICODE_STRING
    +0x020 DefaultObject    : Ptr64 Void
    +0x028 Index            : UChar
    +0x02c TotalNumberOfObjects : Uint4B
    +0x030 TotalNumberOfHandles : Uint4B
    +0x034 HighWaterNumberOfObjects : Uint4B
    +0x038 HighWaterNumberOfHandles : Uint4B
    +0x040 TypeInfo         : _OBJECT_TYPE_INITIALIZER
    +0x0b8 TypeLock         : _EX_PUSH_LOCK
    +0x0c0 Key              : Uint4B
    +0x0c8 CallbackList     : _LIST_ENTRY
    ```
  - OBJECT_TYPE_INITIALIZER STRUCT
    ```
    lkd> dt nt!_OBJECT_TYPE_INITIALIZER
    +0x000 Length           : Uint2B
    +0x002 ObjectTypeFlags  : Uint2B
    +0x002 CaseInsensitive  : Pos 0, 1 Bit
    +0x002 UnnamedObjectsOnly : Pos 1, 1 Bit
    +0x002 UseDefaultObject : Pos 2, 1 Bit
    +0x002 SecurityRequired : Pos 3, 1 Bit
    +0x002 MaintainHandleCount : Pos 4, 1 Bit
    +0x002 MaintainTypeList : Pos 5, 1 Bit
    +0x002 SupportsObjectCallbacks : Pos 6, 1 Bit
    +0x002 CacheAligned     : Pos 7, 1 Bit
    +0x003 UseExtendedParameters : Pos 0, 1 Bit
    +0x003 Reserved         : Pos 1, 7 Bits
    +0x004 ObjectTypeCode   : Uint4B
    +0x008 InvalidAttributes : Uint4B
    +0x00c GenericMapping   : _GENERIC_MAPPING
    +0x01c ValidAccessMask  : Uint4B
    +0x020 RetainAccess     : Uint4B
    +0x024 PoolType         : _POOL_TYPE
    +0x028 DefaultPagedPoolCharge : Uint4B
    +0x02c DefaultNonPagedPoolCharge : Uint4B
    +0x030 DumpProcedure    : Ptr64     void 
    +0x038 OpenProcedure    : Ptr64     long 
    +0x040 CloseProcedure   : Ptr64     void 
    +0x048 DeleteProcedure  : Ptr64     void 
    +0x050 ParseProcedure   : Ptr64     long 
    +0x050 ParseProcedureEx : Ptr64     long 
    +0x058 SecurityProcedure : Ptr64     long 
    +0x060 QueryNameProcedure : Ptr64     long 
    +0x068 OkayToCloseProcedure : Ptr64     unsigned char 
    +0x070 WaitObjectFlagMask : Uint4B
    +0x074 WaitObjectFlagOffset : Uint2B
    +0x076 WaitObjectPointerOffset : Uint2B
    ```

    `dx @$objTypes = (nt!_OBJECT_TYPE*(*)[67])&nt!ObpObjectTypes`