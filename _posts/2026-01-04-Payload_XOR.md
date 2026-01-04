---
title: "Payload - XOR"
date: 2026-01-04
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

# Payload Encryption - XOR

XOR is a simple way to encrypt shellcode, it does not require additional libraries or the usage of Windows APIs. It is faster than AES and RC4. 

it is Symmetric and Reversible. Flip it once to hide, flip it again (with the same key) to reveal. You do not need complex math libraries like OpenSSL; just a simple ^ operator.

```
Encryption: Shellcode + Key = C
Decryption: C + Key = Shellcode
```

## The purpose of XOR

1. **Break Signatures:** An AV looks for the byte sequence FC 48 83 (Common Shellcode). XOR changes this to A1 22 9F. The AV signature fails.
2. **Hide Strings:** Hides URLs (http://c2.com) and APIs (kernel32.dll) from tools like strings.exe and dumpbin.
3. **Polymorphism:** By changing the XOR key every time you compile, the **File Hash** (MD5/SHA256) changes completely, evading static hash blacklists.

## Implementation Variants 

1. Single Byte Key

The simplest form. Every byte of shellcode is XORed against one byte (e.g., 0xAA).
Weakness: If the shellcode has many Null bytes (0x00), the encrypted version will reveal the key (because 0x00 ^ Key = Key).

2. Multi-Byte Key (Roaming Key)
   Using a longer key (e.g., supersecret) and cycling through it using the modulo operator (%).
   Strength: Breaks repeating patterns in the shellcode, making it harder for analysts to guess the key visually.

## Simple Code - Python for Encryption

```python
XOR_KEY = 0xfa
buf = b"This is a text message"

encoded_buf = bytearray(len(buf))
for i in range(len(buf)):
    encoded_buf[i] = buf[i] ^ XOR_KEY

print("\n[+] C++ Output:")
print("unsigned char encrypted_buf[] = {")
for i, byte in enumerate(encoded_buf):
    print(f"0x{byte:02x}", end=", ")
    if (i + 1) % 10 == 0:
        print("")
print("\n};")

decoded_buf = bytearray(len(encoded_buf))
for i in range(len(encoded_buf)):
    decoded_buf[i] = encoded_buf[i] ^ XOR_KEY

print(decoded_buf)
```


## Simple Code - C++ for decryption

```
#include <Windows.h>
#include <stdio.h>


// Simple xor encryption
VOID XorByIOneKey(IN PBYTE pShellcode, IN SIZE_T sShellcodeSize, IN BYTE bKey) {
	for (size_t i = 0; i < sShellcodeSize; i++) {
		pShellcode[i] = pShellcode[i] ^ bKey;
	}
}

unsigned char encrypted_buf[] = {
	0xae, 0x92, 0x93, 0x89, 0xda, 0x93, 0x89, 0xda, 0x9b, 0xda,
	0x8e, 0x9f, 0x82, 0x8e, 0xda, 0x97, 0x9f, 0x89, 0x89, 0x9b,
	0x9d, 0x9f,
};

unsigned char key = 0xfa;

int main() {
	printf("[+] Shellcode: 0x%p \n", encrypted_buf);

	// Decryption
	XorByIOneKey(encrypted_buf, sizeof(encrypted_buf), key);
	printf("[i] shellcode: \"%s\" \n", (char*)encrypted_buf);

	printf("[i] Press <Enter> to Quit ...");
	getchar();
	return 0;
}
```

![payload_xor](https://raw.githubusercontent.com/louyyng/louyyng.github.io/refs/heads/master/files/screenshots/payload_xor.jpg)
