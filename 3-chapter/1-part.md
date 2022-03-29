# IsDebuggerPresent

## IsDebuggerPresent

这是最简单的反调试方法，直接调用IsDebuggerPreSent函数，判断一个进程是否在用户模式下被调试器调试。这里是MSDN给出的解释
> [IsDebuggerPresent](https://docs.microsoft.com/zh-cn/windows/win32/api/debugapi/nf-debugapi-isdebuggerpresent?redirectedfrom=MSDN)

大体伪代码是

```cpp
if (IsDebuggerPresent()) {
    endGame();
}
```

```asm
call IsDebuggerPresent
test al, al
jne  being_debugged
```

IsDebuggerPresent函数内部
```
0:000< u kernelbase!IsDebuggerPresent L3
KERNELBASE!IsDebuggerPresent:
751ca8d0 64a130000000    mov     eax,dword ptr fs:[00000030h]
751ca8d6 0fb64002        movzx   eax,byte ptr [eax+2]
751ca8da c3              ret
```

我们通过相对于fs段的30h偏移量来查看PEB(进程环境块)结构。查看PEB中的2个偏移量，就会发现BeingDebugged字段：
```
0:000< dt _PEB
ntdll!_PEB
   +0x000 InheritedAddressSpace : UChar
   +0x001 ReadImageFileExecOptions : UChar
   +0x002 BeingDebugged    : UChar
```

换句话说，IsDebuggerPresent函数读取BeingDebugged字段的值。如果正在调试进程，则值为1，否则为0。
在函数内部，该函数只返回beingdebuged标志的值

## PEB（进程环境块）

1. PEB地址取得
    PEB是Windows操作系统中使用的一个封闭结构。根据环境的不同，您需要以不同的方式获得PEB结构指针。
    FS段寄存器指向当前的TEB结构，在TEB偏移0x30(64位下FS的偏移为0x60)处是PEB指针。通过这个指针就可以取得PEB的地址。
    ```
    mov eax,fs:[0x30]
    mov PEB,eax
    ```
    下面是如何为x32和x64系统获取PEB指针的例子:
    ```
    PVOID GetPEB()
    {
    #ifdef _WIN64
        return (PVOID)__readgsqword(0x0C * sizeof(PVOID));
    #else
        return (PVOID)__readfsdword(0x0C * sizeof(PVOID));
    #endif
    }
    ```
    WOW64机制用于在x64系统上启动x32进程，并创建另一个PEB结构。以下是如何在WOW64环境中获取PEB结构指针的示例：
    ```
    PVOID GetPEB64()
    {
        PVOID pPeb = 0;
    #ifndef _WIN64
        // 1. There are two copies of PEB - PEB64 and PEB32 in WOW64 process
        // 2. PEB64 follows after PEB32
        // 3. This is true for versions lower than Windows 8, else __readfsdword returns address of real PEB64
        if (IsWin8OrHigher())
        {
            BOOL isWow64 = FALSE;
            typedef BOOL(WINAPI *pfnIsWow64Process)(HANDLE hProcess, PBOOL isWow64);
            pfnIsWow64Process fnIsWow64Process = (pfnIsWow64Process)
                GetProcAddress(GetModuleHandleA("Kernel32.dll"), "IsWow64Process");
            if (fnIsWow64Process(GetCurrentProcess(), &isWow64))
            {
                if (isWow64)
                {
                    pPeb = (PVOID)__readfsdword(0x0C * sizeof(PVOID));
                    pPeb = (PVOID)((PBYTE)pPeb + 0x1000);
                }
            }
        }
    #endif
        return pPeb;
    }
    ```
    用于检查操作系统版本的函数的代码如下：
    ```
    WORD GetVersionWord()
    {
        OSVERSIONINFO verInfo = { sizeof(OSVERSIONINFO) };
        GetVersionEx(&verInfo);
        return MAKEWORD(verInfo.dwMinorVersion, verInfo.dwMajorVersion);
    }
    BOOL IsWin8OrHigher() { return GetVersionWord() >= _WIN32_WINNT_WIN8; }
    BOOL IsVistaOrHigher() { return GetVersionWord() >= _WIN32_WINNT_VISTA; }
    ```



## 如何绕过IsDebuggerPresent检查

1. 将JE变为JMP（74 -> EB）
    ```cpp
    void BypassAntiDebug()
    {
        DWORD OldProtection = NULL;
        VirtualProtect(reinterpret_cast<void*>(0x004F7306), 1, PAGE_EXECUTE_READWRITE, &OldProtection);
        *reinterpret_cast<PBYTE>(0x004F7306) = 0xEB;
        VirtualProtect(reinterpret_cast<void*>(0x004F7306), 1, OldProtection, &OldProtection);
    }
    ```
2. call IsDebuggerPresent -> mov eax,0 ret
3. 将BeingDebugged设置为0
   
    x32
    ```
    mov eax, dword ptr fs:[0x30]  
    mov byte ptr ds:[eax+2], 0 
    ```
    x64
    ```
    DWORD64 dwpeb = __readgsqword(0x60);
    *((PBYTE)(dwpeb + 2)) = 0;
    ```
