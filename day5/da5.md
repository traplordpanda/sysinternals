Process Sandboxing
- windows 10 feature UMDF host. new apis can be used for hardware login
```
lkd> dx -g @$cursession.Processes.Where(p => p.KernelObject.Pcb.SecureState.SecureHandle).Select(p => new {secureproc=p.Name, securestate=p.KernelObject.Pcb.SecureState.Flags.SecureProcess})
================================================
=            = secureproc        = securestate =
================================================
= [0x48]     - Secure System     - 0x1         =
= [0x3c8]    - LsaIso.exe        - 0x1         =
= [0xcec]    - SgrmBroker.exe    - 0x0         =
================================================
```
- silos
  - application silos and server silos
  - server silo custom object directory root
  - combined with virtual machine compute