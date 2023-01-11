### OS Design
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