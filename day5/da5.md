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

Undocumented !vm 21 extension will display location,name, and size of all regions
```
lkd> !vm 21
Page File: \??\C:\pagefile.sys
  Current:   4063232 Kb  Free Space:   4063224 Kb
  Minimum:   4063232 Kb  Maximum:     75497472 Kb
Page File: \??\C:\swapfile.sys
  Current:    262144 Kb  Free Space:    262136 Kb
  Minimum:    262144 Kb  Maximum:     37621984 Kb
No Name for Paging File
  Current: 100578796 Kb  Free Space: 100553176 Kb
  Minimum: 100578796 Kb  Maximum:    100578796 Kb

Physical Memory:          6270331 (   25081324 Kb)
Available Pages:          4625989 (   18503956 Kb)
ResAvail Pages:           5900295 (   23601180 Kb)
Locked IO Pages:                0 (          0 Kb)
Free System PTEs:      4295077047 (17180308188 Kb)

******* 67968 kernel stack PTE allocations have failed ******


******* 247785920 kernel stack growth attempts have failed ******

Modified Pages:             57837 (     231348 Kb)
Modified PF Pages:          57841 (     231364 Kb)
Modified No Write Pages:        0 (          0 Kb)
NonPagedPool Usage:           230 (        920 Kb)
NonPagedPoolNx Usage:       45874 (     183496 Kb)
NonPagedPool Max:      4294967296 (17179869184 Kb)
PagedPool Usage:            89527 (     358108 Kb)
PagedPool Maximum:     4294967296 (17179869184 Kb)
Processor Commit:            1638 (       6552 Kb)
Session Commit:              3041 (      12164 Kb)
Shared Commit:             145872 (     583488 Kb)
Special Pool:                   0 (          0 Kb)
Kernel Stacks:              18642 (      74568 Kb)
Pages For MDLs:             55682 (     222728 Kb)
ContigMem Pages:                0 (          0 Kb)
Pages For AWE:                  0 (          0 Kb)
NonPagedPool Commit:        66048 (     264192 Kb)
PagedPool Commit:           89527 (     358108 Kb)
Driver Commit:              20066 (      80264 Kb)
Boot Commit:                 4872 (      19488 Kb)
PFN Array Commit:           74033 (     296132 Kb)
SmallNonPagedPtesCommit:      879 (       3516 Kb)
SlabAllocatorPages:         23040 (      92160 Kb)
SkPagesInUnchargedSlabs:        0 (          0 Kb)
System PageTables:           6476 (      25904 Kb)
ProcessLockedFilePages:       300 (       1200 Kb)
Pagefile Hash Pages:            0 (          0 Kb)
Sum System Commit:         510116 (    2040464 Kb)
Total Private:            1308427 (    5233708 Kb)
Misc/Transient Commit:       1508 (       6032 Kb)
Committed pages:          1820051 (    7280204 Kb)
Commit limit:             7286139 (   29144556 Kb)

System Region               Base Address    NumberOfBytes

PfnDatabase           : ffff830000000000      38000000000
Cfg                   : ffff873a7f879fe0      28000000000
Session               : ffff8b0000000000       8000000000
PagedPool             : ffff8b8000000000     100000000000
PageTables            : ffff9f8000000000       8000000000
NonPagedPool          : ffffa08000000000     100000000000
KernelStacks          : ffffb10000000000      10000000000
SystemPtes            : ffffb80000000000     100000000000
SecureNonPagedPool    : ffffd00000000000       8000000000
UltraZero             : ffffd08000000000     100000000000
SystemCache           : ffffe40000000000     100000000000
SystemImages          : fffff80000000000       8000000000
HyperSpace            : fffffa8000000000      10000000000
```