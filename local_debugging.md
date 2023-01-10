### local debugging
- NtSystemDebugControl 
  - undocumented full read write api 
  - used for pirating encrypted movies
  - moved to kernel and replaced with KdSystemDebugControl api 
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