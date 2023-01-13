Objects
====
Grabbing random object 
```
lkd> !process -1 0
PROCESS ffffaf8f1d4bb080
    SessionId: 1  Cid: 23cc    Peb: 64e5675000  ParentCid: 0f0c
    DirBase: 3db957002  ObjectTable: ffff998ea1ac6940  HandleCount: 475.
    Image: EngHost.exe

lkd> !object 0xffffaf8f1d4bb080
Object: ffffaf8f1d4bb080  Type: (ffffaf8f180967a0) Process
    ObjectHeader: ffffaf8f1d4bb050 (new version)
    HandleCount: 10  PointerCount: 317676
```
Grab object by name
- cast to OBJECT_HEADER type
- cast offset to OBJECT_HEADER_NAME_INFO* 
```
lkd> !object \PowerPort
Object: ffffaf8f18085df0  Type: (ffffaf8f180ea560) ALPC Port
    ObjectHeader: ffffaf8f18085dc0 (new version)
    HandleCount: 1  PointerCount: 31092
    Directory Object: ffff998e7e288e70  Name: PowerPort

lkd> dx @$hdr = (nt!_OBJECT_HEADER*)0xffffaf8f18085dc0
@$hdr = (nt!_OBJECT_HEADER*)0xffffaf8f18085dc0                 : 0xffffaf8f18085dc0 [Type: _OBJECT_HEADER *]
    [+0x000] PointerCount     : 31076 [Type: __int64]
    [+0x008] HandleCount      : 1 [Type: __int64]
    [+0x008] NextToFree       : 0x1 [Type: void *]
    [+0x010] Lock             [Type: _EX_PUSH_LOCK]
    [+0x018] TypeIndex        : 0x75 [Type: unsigned char]
    [+0x019] TraceFlags       : 0x0 [Type: unsigned char]
    [+0x019 ( 0: 0)] DbgRefTrace      : 0x0 [Type: unsigned char]
    [+0x019 ( 1: 1)] DbgTracePermanent : 0x0 [Type: unsigned char]
    [+0x01a] InfoMask         : 0x6 [Type: unsigned char]
    [+0x01b] Flags            : 0x42 [Type: unsigned char]
    [+0x01b ( 0: 0)] NewObject        : 0x0 [Type: unsigned char]
    [+0x01b ( 1: 1)] KernelObject     : 0x1 [Type: unsigned char]
    [+0x01b ( 2: 2)] KernelOnlyAccess : 0x0 [Type: unsigned char]
    [+0x01b ( 3: 3)] ExclusiveObject  : 0x0 [Type: unsigned char]
    [+0x01b ( 4: 4)] PermanentObject  : 0x0 [Type: unsigned char]
    [+0x01b ( 5: 5)] DefaultSecurityQuota : 0x0 [Type: unsigned char]
    [+0x01b ( 6: 6)] SingleHandleEntry : 0x1 [Type: unsigned char]
    [+0x01b ( 7: 7)] DeletedInline    : 0x0 [Type: unsigned char]
    [+0x01c] Reserved         : 0x0 [Type: unsigned long]
    [+0x020] ObjectCreateInfo : 0x1 [Type: _OBJECT_CREATE_INFORMATION *]
    [+0x020] QuotaBlockCharged : 0x1 [Type: void *]
    [+0x028] SecurityDescriptor : 0xffff998e7e34af2d [Type: void *]
    [+0x030] Body             [Type: _QUAD]
    ObjectName       : "PowerPort"
    ObjectType       : ALPC Port

lkd> dx ((char*)&nt!ObpInfoMaskToOffset)[@$hdr->InfoMask & (4 - 1)]
((char*)&nt!ObpInfoMaskToOffset)[@$hdr->InfoMask & (4 - 1)] : 32 ' ' [Type: char]

lkd> dx (nt!_OBJECT_HEADER_NAME_INFO*)((__int64)@$hdr - 0x20)
(nt!_OBJECT_HEADER_NAME_INFO*)((__int64)@$hdr - 0x20)                 : 0xffffaf8f18085da0 [Type: _OBJECT_HEADER_NAME_INFO *]
    [+0x000] Directory        : 0xffff998e7e288e70 [Type: _OBJECT_DIRECTORY *]
    [+0x008] Name             : "PowerPort" [Type: _UNICODE_STRING]
    [+0x018] ReferenceCount   : 0 [Type: long]
    [+0x01c] Reserved         : 0x50000000 [Type: unsigned long]  
```
extended footer information 
```
lkd> dx (nt!_OBJECT_HEADER_EXTENDED_INFO*)((__int64)@$hdr - 0x40)
(nt!_OBJECT_HEADER_EXTENDED_INFO*)((__int64)@$hdr - 0x40)                 : 0xffffaf8f18085d80 [Type: _OBJECT_HEADER_EXTENDED_INFO *]
    [+0x000] Footer           : 0x43504c4102250000 [Type: _OBJECT_FOOTER *]
    [+0x008] Reserved         : 0xffffaf8f18070d89 [Type: unsigned __int64]
lkd> dx -r1 ((ntkrnlmp!_OBJECT_FOOTER *)0x43504c4102250000)
((ntkrnlmp!_OBJECT_FOOTER *)0x43504c4102250000)                 : 0x43504c4102250000 [Type: _OBJECT_FOOTER *]
    [+0x000] HandleRevocationInfo [Type: _HANDLE_REVOCATION_INFO]
    [+0x020] ExtendedUserInfo [Type: _OB_EXTENDED_USER_INFO]
lkd> dx -r1 (*((ntkrnlmp!_HANDLE_REVOCATION_INFO *)0x43504c4102250000))
(*((ntkrnlmp!_HANDLE_REVOCATION_INFO *)0x43504c4102250000))                 [Type: _HANDLE_REVOCATION_INFO]
    [+0x000] ListEntry        [Type: _LIST_ENTRY]
    [+0x010] RevocationBlock  : Unable to read memory at Address 0x43504c4102250010
    [+0x018] AllowHandleRevocation : Unable to read memory at Address 0x43504c4102250018
    [+0x019] Padding1         [Type: unsigned char [3]]
    [+0x01c] Padding2         [Type: unsigned char [4]]
```
Core and Executive Mechanisms
===

(Advanced) Local Prodecure Call - (A)LPC
- Entirely undocumented
LPC basics
- Client/server model
- object manager support
- full security (ACLs) and naming/lookup
- two data passing mechanisms
  - Message buffer (256 bytes of data)
    - message data is directly on the stack/part of allocated LPC message
  - shared section (more than 256)
    - region of physical memory mapped both in client and server
- ALPC systemcalls
```
119  0xfffff80709cc7c1c                               nt!NtAlpcAcceptConnectPort(0xfffff8070a2bb3c0) 0n99850245  
120  0xfffff80709cc7c20                                   nt!NtAlpcCancelMessage(0xfffff8070a319360) 0n106009088 
121  0xfffff80709cc7c24                                     nt!NtAlpcConnectPort(0xfffff8070a2b9770) 0n99734279  
122  0xfffff80709cc7c28                                   nt!NtAlpcConnectPortEx(0xfffff8070a2b97f0) 0n99736327  
123  0xfffff80709cc7c2c                                      nt!NtAlpcCreatePort(0xfffff8070a305a00) 0n104725504 
124  0xfffff80709cc7c30                               nt!NtAlpcCreatePortSection(0xfffff8070a2c0cc0) 0n100214786 
125  0xfffff80709cc7c34                           nt!NtAlpcCreateResourceReserve(0xfffff8070a2f1960) 0n103412224 
126  0xfffff80709cc7c38                               nt!NtAlpcCreateSectionView(0xfffff8070a2c1170) 0n100233984 
127  0xfffff80709cc7c3c                           nt!NtAlpcCreateSecurityContext(0xfffff8070a2ee0d0) 0n103180544 
128  0xfffff80709cc7c40                               nt!NtAlpcDeletePortSection(0xfffff8070a2c0bd0) 0n100210944 
129  0xfffff80709cc7c44                           nt!NtAlpcDeleteResourceReserve(0xfffff8070a4c3500) 0n133934080 
130  0xfffff80709cc7c48                               nt!NtAlpcDeleteSectionView(0xfffff8070a2c0aa0) 0n100206080 
131  0xfffff80709cc7c4c                           nt!NtAlpcDeleteSecurityContext(0xfffff8070a210500) 0n88648704  
132  0xfffff80709cc7c50                                  nt!NtAlpcDisconnectPort(0xfffff8070a306eb0) 0n104810240 
133  0xfffff80709cc7c54                nt!NtAlpcImpersonateClientContainerOfPort(0xfffff8070a4c25c0) 0n133871616 
134  0xfffff80709cc7c58                         nt!NtAlpcImpersonateClientOfPort(0xfffff8070a20e420) 0n88514048  
135  0xfffff80709cc7c5c                               nt!NtAlpcOpenSenderProcess(0xfffff8070a2be6f0) 0n100059906 
136  0xfffff80709cc7c60                                nt!NtAlpcOpenSenderThread(0xfffff8070a307440) 0n104833026 
137  0xfffff80709cc7c64                                nt!NtAlpcQueryInformation(0xfffff8070a2e3350) 0n102469889 
138  0xfffff80709cc7c68                         nt!NtAlpcQueryInformationMessage(0xfffff8070a2b6620) 0n99532290  
139  0xfffff80709cc7c6c                           nt!NtAlpcRevokeSecurityContext(0xfffff8070a4c2800) 0n133880832 
140  0xfffff80709cc7c70                             nt!NtAlpcSendWaitReceivePort(0xfffff8070a20c400) 0n88382468  
141  0xfffff80709cc7c74                                  nt!NtAlpcSetInformation(0xfffff8070a2f0980) 0n103347200 
```
ALPC root objects 
```
ffffaf8f1ffa0aa0 ALPC Port                 ThemeApiPort
ffffaf8f18f0e8b0 ALPC Port                 PdcPort
ffffaf8f1cb65720 ALPC Port                 SeRmCommandPort
ffffaf8f18085df0 ALPC Port                 PowerPort
ffffaf8f1fe30ae0 ALPC Port                 SmSsWinStationApiPort
ffffaf8f18070e00 ALPC Port                 PowerMonitorPort
ffffaf8f1df36af0 ALPC Port                 SeLsaCommandPort
ffffaf8f1d44adf0 ALPC Port                 SmApiPort
```
alpc graph show what processes are using what alpc port
```
lkd> !alpc /graph
ffffaf8f1f816690 0000000000000004 ffffaf8f18070e00 0000000000000004 PowerMonitorPort
ffffaf8f1dd38df0 000000000000026c ffffaf8f1d44adf0 00000000000001dc SmApiPort
ffffaf8f1dd82df0 00000000000001dc ffffaf8f1dd74df0 000000000000026c SbApiPort
ffffaf8f1ddabdf0 000000000000035c ffffaf8f1db1a310 000000000000026c ApiPort
ffffaf8f1df21b80 0000000000000004 ffffaf8f1df14c40 000000000000035c WMsgKRpc0B1470
ffffaf8f1df5ab80 00000000000003b0 ffffaf8f1db1a310 000000000000026c ApiPort
ffffaf8f1de8d310 00000000000003d0 ffffaf8f1db1a310 000000000000026c ApiPort
ffffaf8f1df36d50 00000000000003d0 ffffaf8f1cb65720 0000000000000004 SeRmCommandPort
ffffaf8f1de99df0 0000000000000004 ffffaf8f1df36af0 00000000000003d0 SeLsaCommandPort
ffffaf8f1df7dd40 0000000000000370 ffffaf8f1d44adf0 00000000000001dc SmApiPort
ffffaf8f1dfa8d20 00000000000001dc ffffaf8f1df7dae0 0000000000000370 SbApiPort
ffffaf8f1df8bd30 0000000000000228 ffffaf8f1dd96df0 0000000000000370 ApiPort
ffffaf8f1dfb1b80 0000000000000004 ffffaf8f1dfadd40 0000000000000228 WMsgKRpc0C3231
ffffaf8f1dfb7700 0000000000000004 ffffaf8f1dfc19a0 00000000000003d0 lsasspirpc
ffffaf8f1dfe6570 00000000000003d0 ffffaf8f1dfc19a0 00000000000003d0 lsasspirpc
ffffaf8f1dfe8ce0 00000000000003b0 ffffaf8f1dfc19a0 00000000000003d0 lsasspirpc
ffffaf8f1df34a80 00000000000003d0 ffffaf8f1dfe6a50 00000000000003b0 ntsvcs
ffffaf8f1dff0ce0 000000000000035c ffffaf8f1dfc19a0 00000000000003d0 lsasspirpc
```

Reg hack
- null terminated keys hide info after
- 