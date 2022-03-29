# NtGlobalFlag

## NtGlobalFlag

在WindowsNT中，有一组标志存储在全局变量NtGlobalFlag中，这在整个系统中是常见的。启动时，将使用系统注册表项中的值初始化NtGlobalFlag全局系统变量：
```
[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Control\Session Manager\GlobalFlag]
```

此变量值用于系统跟踪、调试和控制。变量标志未被文档化，但是SDK包含gflags实用程序，它允许您编辑全局标志值。PEB结构还包括NtGlobalFlag字段，它的位结构不对应于NtGlobalFlag全局系统变量。调试过程中，在NtGlobalFlag字段中设置这些标志:

```
FLG_HEAP_ENABLE_TAIL_CHECK (0x10)
FLG_HEAP_ENABLE_FREE_CHECK (0x20)
FLG_HEAP_VALIDATE_PARAMETERS (0x40)
```

若要检查是否已使用调试器启动进程，请检查PEB结构中NtGlobalFlag字段的值。在x32和x64系统中,该字段位于PEB结构的开始处的0x068和0x0bc偏移处。

x32
```
0:000> dt _PEB NtGlobalFlag @$peb  
ntdll!_PEB 
   +0x068 NtGlobalFlag : 0x70
```

x64
```
0:000> dt _PEB NtGlobalFlag @$peb
ntdll!_PEB
   +0x0bc NtGlobalFlag : 0x70
```

以下代码片段是基于NtGlobalFlag标志检查的反调试保护示例：

```
#define FLG_HEAP_ENABLE_TAIL_CHECK   0x10
#define FLG_HEAP_ENABLE_FREE_CHECK   0x20
#define FLG_HEAP_VALIDATE_PARAMETERS 0x40
#define NT_GLOBAL_FLAG_DEBUGGED (FLG_HEAP_ENABLE_TAIL_CHECK | FLG_HEAP_ENABLE_FREE_CHECK | FLG_HEAP_VALIDATE_PARAMETERS)
void CheckNtGlobalFlag()
{
    PVOID pPeb = GetPEB();
    PVOID pPeb64 = GetPEB64();
    DWORD offsetNtGlobalFlag = 0;
#ifdef _WIN64
    offsetNtGlobalFlag = 0xBC;
#else
    offsetNtGlobalFlag = 0x68;
#endif
    DWORD NtGlobalFlag = *(PDWORD)((PBYTE)pPeb + offsetNtGlobalFlag);
    if (NtGlobalFlag & NT_GLOBAL_FLAG_DEBUGGED)
    {
        std::cout << "Stop debugging program!" << std::endl;
        exit(-1);
    }
    if (pPeb64)
    {
        DWORD NtGlobalFlagWow64 = *(PDWORD)((PBYTE)pPeb64 + 0xBC);
        if (NtGlobalFlagWow64 & NT_GLOBAL_FLAG_DEBUGGED)
        {
            std::cout << "Stop debugging program!" << std::endl;
            exit(-1);
        }
    }
}
```