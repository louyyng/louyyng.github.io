---
title: "Concept Of DLL"
date: 2026-01-03
categories:
  - blog
tags:
  - windows
author_profile: true
read_time: true
comments: false
share: true
related: true
header:
  teaser: /images/profile.png
---

# Concept of Dynamic-Link Libraries (DLL)

## What is the differen between DLL and EXE?

- EXE is a standalone program. Has its own memory space. Executed by double-clicking.

- DLL is a shared library of functions. Must be loaded into an exe (host process) to run

- Common DLLs: `kernel32.dll`, `ntdll.dll`, `user32.dll` 

These common dlls are loaded into almost every windows process automatically.

## Anatomy of a DLL

The Entry Point: `DllMain`

Just as C++ exe start at `main()`, DLL start at `DllMain`

### Exporting Functions

To let other programs use a function inside your DLL, you must export it using a specific keyword:

- `extern __declspec(dllexport)`
- Example: `extern "C" __declspec(dllexport) void MaliciousFunction() {...}`

## Dynamic Linking 

Normal programs link functions at compile time.

- Malware links functions at `Runtime` (Dynamic Linking) to hide what it is doing from antivirus static analysis.

The Trinity of APIs

- To use a function from a DLL manually, we need these three steps:

| Step  | API Function     | Purpose                                                      |
| ----- | ---------------- | ------------------------------------------------------------ |
| **1** | LoadLibraryA     | Loads a DLL from **Disk** into Memory. (Returns HMODULE).    |
| **2** | GetModuleHandleA | Finds a DLL already in **Memory**. (Returns HMODULE).        |
| **3** | GetProcAddress   | Finds the memory address of a specific **Function** inside that DLL. |

## Execution and Tools

`Rundll32.exe`

Since we cannot double click to execute dll, we can use Windows built-in tool to run

```
rundll32.exe <dllname>, <function name>
```

## Code Example

Create two projects in Visual Studio (C++ Console App and C++ DLL)

```dll
// dllmain.cpp : Defines the entry point for the DLL application.
#include <Windows.h>

// Export Function
extern "C" _declspec(dllexport) void HelloWorld() {
	MessageBoxA(NULL, "Hello World", "DLL Message", MB_ICONINFORMATION); 
}

// Entry Point
BOOL APIENTRY DllMain(HMODULE hModule, DWORD ul_reason_for_call, LPVOID lpReserved) {
	switch (ul_reason_for_call) {
		case DLL_PROCESS_ATTACH:
		case DLL_THREAD_ATTACH:
		case DLL_THREAD_DETACH:
		case DLL_PROCESS_DETACH:
			break;
	}
	return TRUE;
}
```



Console App for loading dll

```C++
#include <Windows.h>
#include <stdio.h>

// Constructing a new data type which represent HelloWorld's function pointer.
typedef void (* HelloWorldFunctionPointer)();

void call() {

	// Get Handle of the dll
	LPCSTR dllPath = "C:\\Users\\username\\source\\repos\\module6\\x64\\Debug\\simpledll.dll";

		HMODULE hModule = GetModuleHandleA(dllPath);
		// If the DLL is not loaded in memory, use LoadLibrary
		if (hModule == NULL) {
			// Loading DLL
			hModule = LoadLibraryA(dllPath);
		}
	
	if (hModule == NULL) {
		printf("[-] Failed to Load Dll. Error: %d\n", GetLastError());
		return;
	}

		// pHelloWord stores the HelloWorld's Function address --> dllmain.cpp
		PVOID pHelloWorld = GetProcAddress(hModule, "HelloWorld");

	if(pHelloWorld == NULL){
		printf("[-] Failed to find function. Error: %d\n", GetLastError());
		return;
	}

		HelloWorldFunctionPointer HelloWorld = (HelloWorldFunctionPointer)pHelloWorld;
		HelloWorld();
}

int main() {
	call();
	return 0;
}
```



![dll_hello_world](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/dll_hello_world.jpg)


