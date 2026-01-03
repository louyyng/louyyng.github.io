---
title: "Concept Of Windows API"
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

# Concept Of Windows API

It is an interface that allows user-mode applications to interact with Windows Kernel.

## Windows Data Types

| Windows Type     | Standard C Equivalent             | Description                                              |                                                              |
| ---------------- | --------------------------------- | -------------------------------------------------------- | ------------------------------------------------------------ |
| **DWORD**        | unsigned int (32-bit)             | A 32-bit integer (0 to `232−1232−1`).                    | Used for flags, PIDs, and sizes.                             |
| **size_t**       | unsigned int / unsigned long long | Size of an object. **32-bit** on x86, **64-bit** on x64. | Critical for buffer allocation (malloc).                     |
| **VOID / PVOID** | void / void*                      | Generic pointer (address).                               | Used when data type is unknown (e.g., shellcode address).    |
| **HANDLE**       | void* (Internal)                  | An "ID card" for an OS object (File, Process, Thread).   | You need a HANDLE to inject into a process.                  |
| **HMODULE**      | void*                             | Base address of a generic module (DLL/EXE).              | Used in GetModuleHandle to find where DLLs are loaded.       |
| **ULONG_PTR**    | unsigned long                     | An integer the same size as a pointer.                   | **Critical:** Used for Pointer Arithmetic to avoid compiler errors. |

## String ("A" VS "W")

| Type        | Char Type      | Encoding         | Example   | Size     |
| ----------- | -------------- | ---------------- | --------- | -------- |
| **LPCSTR**  | const char*    | ANSI (8-bit)     | "String"  | 7 bytes  |
| **LPCWSTR** | const wchar_t* | Unicode (16-bit) | L"String" | 14 bytes |

### Naming Logic:

- LP = Long Pointer
- C = Constant (Read-Only)
- STR = String (ANSI)
- WSTR = Wide String (Unicode)

## Function Name Conventions

What is the different between "A" and "W"

e.g `CreateFileA` and `CreateFileW`

`CreateFileA`: Excepts ANSI strings (`LPCSTR`)

`CreateFileW`: Except Unicode strings (`LPCWSTR`)

Tips: Modern Windows uses --> Unicode internally.`CreateFileA` converts the strings and call `CreateFileW`

### Pointers in Naming

If a type start with P, it is a pointer to that type.

`DWORD` = The integer itself

`PWORD` = `DWORD*` (Pointer to integer)

## Parameters (IN vs OUT)

- [IN]: Data you give to the function. e.g FileName
- [OUT]: Data the function gives back to you. Usually passed by reference (Pointer).

```
// Example of OUT parameter
BOOL GetProcessId(OUT int* pid) {
	*pid = 1234;
	return TRUE;
}
```

## Error Handling and Debugging

Malware crashes often. We should know why.

### WinAPI Errors (User Mode)

If a function fails (returns NULL or INVALID_HANDLE_VALUE):

1. Call `GetLastError()`
2. Check the code against System Error Code (e.g Error `5` = Access Denied).

### Native API Errors (NTAPI)

Function from `ntdll.dll` do not use `GetLastError`. They usually return a status code directly (`NTSTATUS`).

- Success: `STATUS_SUCCESS` (Value `0`).
- Failure: Non-Zero 
- Macro: Use `NT_SUCCESS (STATUS)` to check boolean success.

```
NTSTATUS status = NtCreateThreadEx(...);
if (!NT_SUCCESS(status)) {
	printf("Failed: 0x%X", status);
}
```

## Code Example

### Objective

Write a C++ program that:

1. Uses CreateFileW (Unicode) to create payload.txt.
2. Uses WriteFile to write data into it.
3. Implements strict **Error Handling** (checking INVALID_HANDLE_VALUE and GetLastError).

```
// Objective: 
// 1. Use CreateFileW (Unicode) to create payload.txt
// 2. Use WriteFile to write data into it.
// 3. Implements strict Error Handling 

#include <Windows.h>
#include <stdio.h>

int main() {
	LPCWSTR filename	= L"C:\\Users\\Public\\payload.txt";
	LPCSTR payloadData	= "Testing, Windows API";

	DWORD payloadSize = (DWORD)strlen(payloadData);
	DWORD bytesWritten = 0;

	printf("[*] Target File: %ls \n", filename);

	// Create File with WinAPI - https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-createfilew
	HANDLE hFile = CreateFileW(
		filename,					  //lpFileName
		GENERIC_WRITE,                //dwDesiredAccess
		0,                            //dwShareMode
		NULL,						  //lpSecurityAttributes
		CREATE_ALWAYS,                //dwCreationDisposition
		FILE_ATTRIBUTE_NORMAL,        //dwFlagsAndAttributes
		NULL                          //hTemplateFile
	);

	// Error Handling
	if (hFile == INVALID_HANDLE_VALUE) {
		printf("[-] Failed to create file. Error Code: %d \n", GetLastError());
		return 1;
	}
	printf("[+] Handle obtained successfully \n");

	// Write Data to File with WinAPI - https://learn.microsoft.com/en-us/windows/win32/api/fileapi/nf-fileapi-writefile
	BOOL success = WriteFile(
		hFile,				//hFile,
		payloadData,		//lpBuffer,
		payloadSize,		//nNumberOfBytesToWrite,
		&bytesWritten,		//lpNumberOfBytesWritten,
		NULL				//lpOverlapped
	);

	// Error Handling
	if (success && bytesWritten == payloadSize) {
		printf("[+] Successfully wrote %d bytes to disk. \n", bytesWritten);
	}
	else {
		printf("[-] Writting Failed. Error: %d \n", GetLastError());
	}
	
	// Clean Up
	CloseHandle(hFile);
	return 0;
}

```

After compiled, we can find the payload.txt in Public folder

![con_winapi_1.png](https://github.com/louyyng/louyyng.github.io/blob/master/files/screenshots/con_winapi_1.png)
