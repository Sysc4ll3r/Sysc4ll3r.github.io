---
title: Hextree Weather App - Defeating JNI Obfuscation
date: 2024-12-20 03:38:00
categories: [android, hextree-challenges]
tags: [reverse-engineering, hextree-challenges, android]
---

Today I explain my approach to solve the Defeating JNI Obfuscation challenge on  hextree. 
The challenge can be described as the developer will not add a raw API key in the app, but he will  add it encrypted and use JNI functions in a native library to decrypt the key to use it later.

The approach that `LiveOverflow` used is fine but requires writing the app from scratch and writing the same native library from scratch. My approach is analyze + runtime hooking by Frida. second appraoch by static analyzing. my approaches require less effort and less time  .

First, we can download the APK file from [This Link](https://storage.googleapis.com/hextree_prod_image_uploads/media/uploads/reverse-android-apps/io.hextree.weatherusa_update1.apk)

## Analyze

Open the downloaded APK in JADX to analyze it :

![Hex Tree Jadx](/assets/img/hextree/hextree_jni_challenge_1.png)

i found interest class called `InternetUtil` at app main package `io.hextree.weatherusa` so i started to check it
u will find 2 methods used here `public static String a(String,String)` and `private static native String getkey(String)`
as u know native methods in java is a bridge to call `JNI` native functions defined in c code.. if u don,t understand this don,t worry i will explain this in a  later step in this writeup .
u will find the `a` method load a native library `native-lib`
`System.loadLibrary` do the following as pseudo code under the hood
```code
System.loadLibrary(libraryName){
lib = path_to_appfiles/<cpu_arch>/"lib"+libraryName+".so"
dlopen(lib) 
}
```
that mean the real name for library is `libnative-lib.so` as we notice this in JADX

![Hex Tree Jadx](/assets/img/hextree/hextree_jni_challenge_2.png)

u will find api key here used in header `X-API-KEY` but there is a call for getKey native method declared at the end of `InternetUtil` class as we see above 
u will find this line 
```java 
httpURLConnection2.setRequestProperty("X-API-KEY", getKey("moiba1cybar8smart4sheriff4securi"));
```
which infer us that there is api key used but seems encrypted or not clear because there is a jni call `getKey(api_key)`
which return String so we need to check getKey after extracting `libnative-lib.so`

extracting `libnative-lib.so` will can be done by just unzip it only from the apk because apk actully is a zip file but aligned by specific format
![Hex Tree Unzip](/assets/img/hextree/hextree_jni_challenge_3.png)

now let,s fire ghidra and create ghidra project and then use decompiler (code explorer) and import library as file to anlayze

![Hex Tree ghidra](/assets/img/hextree/hextree_jni_challenge_4.png)


from functions we get i found  interested function call `Java_io_hextree_weatherusa_InternetUtil_getKey(_JNIEnv *param_1,undefined8 param_2,_jstring *param_3)`
native methods in java is used to call functions from c by 
this format : `methodName()` in c `Java_com_package_classname_methodName()`
so `native getKey()` in java call `Java_io_hextree_weatherusa_InternetUtil_getKey` and take the return value from it

now notice `Java_io_hextree_weatherusa_InternetUtil_getKey` call `xorDecrypt(param_1,param_3)`;
so we need to analyze the decryption function `xorDecrypt`
![Hex Tree ghidra](/assets/img/hextree/hextree_jni_challenge_5.png)

and now that,s the end of our analyze step

## Pwn moment - Part 1 dynamic instrumentation solution
1.so we can use `frida` tool the dynamic instrumentation toolkit to solve this challenge easy, by calling this method  getKey dynamically by `frida` 

firda_get_api_key.js :

```javascript
Java.perform(() => {
        console.log("[+] Access io.hextree.weatherusa.InternetUtil class ")
        var InternetUtil = Java.use("io.hextree.weatherusa.InternetUtil");
        try {
                InternetUtil.a.implmentation = function (arg1, arg2) {
                        console.log('[+] Calling Hooked InternetUtil.a method because it call System.loadLibrary("native-lib") which loadLibrary libnative-lib.so');
                        this.a(arg1,arg2);
                };
            InternetUtil.a("","")
                console.log("[+] Calling getKey native method with key")
                const decryptedKey = InternetUtil.getKey("moiba1cybar8smart4sheriff4securi");
                console.log(`[+] Decrypted easy : ${decryptedKey} ;)`);
        } catch (e) {
                console.log("[!] Error: " + e);
        }
});
```

let,s break it down 
- `Java.use` can be used to access some class u want
- `Java.perform` used to perform or execute code 
- so we access `io.hextree.weatherusa.InternetUtil` which is our interested in class and use frida to hook a method and overide its implementation to print this message and then call original a method 
- then we call `getKey(encrypted_key)` method to decrypt it 

install apk
```shell
adb install io.hextree.weatherusa_update1.apk
```
now let,s run it by :

```shell
frida -U -f io.hextree.weatherusa -l frida_get_api_key.js
```

![Hex Tree ghidra](/assets/img/hextree/hextree_jni_challenge_6.png)


We solve it ;) but wait we didn,t finished yet , in fact we have another solution that require better reverse engineering skills

## Pwn Moment - Part 2 the static analyze solution
analyze xorDecrypt i will explain only hard parts

```c
undefined8 xorDecrypt(_JNIEnv *param_1,_jstring *param_2)

{
  int iVar1;
  long lVar2;
  void *__ptr;
  undefined8 uVar3;
  ulong uVar4;
  
  lVar2 = (**(code **)(*(long *)param_1 + 0x548))(param_1,param_2,0);
  if (lVar2 != 0) {
    iVar1 = (**(code **)(*(long *)param_1 + 0x540))(param_1,param_2);
    __ptr = malloc(0x21);
    if (__ptr != (void *)0x0) {
      uVar4 = 0;
      do {
        *(byte *)((long)__ptr + uVar4) =
             *(byte *)(lVar2 + ((long)((ulong)(uint)((int)uVar4 >> 0x1f) << 0x20 |
                                      uVar4 & 0xffffffff) % (long)iVar1 & 0xffffffffU)) ^
             (&DAT_001005a0)[uVar4];
        *(byte *)((long)__ptr + uVar4 + 1) =
             *(byte *)(lVar2 + ((long)((int)uVar4 + 1) % (long)iVar1 & 0xffffffffU)) ^
             (&DAT_001005a1)[uVar4];
        uVar4 = uVar4 + 2;
      } while (uVar4 != 0x20);
      *(undefined *)((long)__ptr + 0x20) = 0;
      (**(code **)(*(long *)param_1 + 0x550))(param_1,param_2,lVar2);
      uVar3 = (**(code **)(*(long *)param_1 + 0x538))(param_1,__ptr);
      free(__ptr);
      return uVar3;
    }
    (**(code **)(*(long *)param_1 + 0x550))(param_1,param_2,lVar2);
  }
  return 0;
}
```


1.
```c
lVar2 = (**(code **)(*(long *)param_1 + 0x548))(param_1,param_2,0);
```
calls a JNI function (offset `0x548` in `param_1`) which is `_JNIENV` as we see before in `Java_io_hextree_weatherusa_InternetUtil_getKey` to retrieve a pointer to the UTF-8 encoded representation of the string param2  , output will be `lVar2` holding String as `utf8`
i hear u wonder how this  calcuted :) , no problem i am here to help
first as we know `param`_1 is _JNIENV structure right 
from offical oracle jvm documentation [https://docs.oracle.com/en/java/javase/21/docs/specs/jni/functions.html](https://docs.oracle.com/en/java/javase/21/docs/specs/jni/functions.html)
we will do the following equation `function_offset/machine_pointer_size = function index in _JNIENV`
u can search by this index it in oracle jni docs by above link 
`machine_pointer_size` here will be `8 for x86_64` cause i use this `cpu arch` so
`param_1+0x548`=`_JNIENV+0x548` this converted to function pointer as we see by `**(code **)(*(long *)`
so `0x548/8` = `169` if we searched by 169 in JNI Strcture 
![Hex Tree ghidra](/assets/img/hextree/hextree_jni_challenge_7.png)
we will find it mapped to `GetStringUTFChars` if we search by `169` - when this function pointer called it,s  convert string data at pointer to utf8 string and lVar2 will hold result
`if (lVar2 != 0) ` mean return if  converting  failed ,  lVar2 will be NULL in this case :  return if IVar2 = `0x00000000000` 
2. 
```c
iVar1 = (**(code **)(*(long *)param_1 + 0x540))(param_1,param_2);
```
`0x540/8 = 168` refer to `GetStringUTFLength` calc length for `utf8` string 
`iVar1` store string at `iVar2` length
3.` __ptr = malloc(0x21);` allocate 33 byte for pointer , `if (__ptr != (void *)0x0)` means continue  if pointer valid
4.
```c
uVar4 = 0;
do {
    *(byte *)((long)__ptr + uVar4) =
         *(byte *)(lVar2 + ((long)((ulong)(uint)((int)uVar4 >> 0x1f) << 0x20 |
                                  uVar4 & 0xffffffff) % (long)iVar1 & 0xffffffffU)) ^
         (&DAT_001005a0)[uVar4];
    *(byte *)((long)__ptr + uVar4 + 1) =
         *(byte *)(lVar2 + ((long)((int)uVar4 + 1) % (long)iVar1 & 0xffffffffU)) ^
         (&DAT_001005a1)[uVar4];
    uVar4 = uVar4 + 2;
} while (uVar4 != 0x20);
```
this is a loop in this loop uVar4=0 and it increased by 2 each cycle and this loop should stop when uVar2 = 32 
notice length of our encrypted api key = 32 `len("moiba1cybar8smart4sheriff4securi")` so  this loop 


`*(byte *)((long)__ptr + uVar4)` mean there is a  byte will stored at `_ptr[uVar4]`
the byte is result of `lVar2[uVar4 % uVar1] ^ DAT_001005a0[uVar4 %uVar1]` DAT_001005a0 if we look at .rodata segment we will find array of bytes represent key 

![Hex Tree ghidra](/assets/img/hextree/hextree_jni_challenge_8.png)


```py
[0x25,0x37, 0x3D, 0x19, 0x0E, 0x53, 0x05, 0x0C, 0x11, 0x02, 0x13, 0x4C, 0x16, 0x09, 
    0x4C, 0x13, 0x04, 0x5D, 0x5E, 0x03, 0x00, 0x0B, 0x44, 0x07, 0x15, 0x56, 0x42, 0x57, 
    0x55, 0x00, 0x01, 0x14]
```

and do the same with next byte mean another xor operation 
finally means `encryptedString[int(counter % length )] ^ key[int(counter % length )] `
any operation like `& 0xffffffff , >> 0x1f) << 0x20 , & 0xffffffffU ` to handle just sign of uVar4  ,iVar1 .do u see how it now more clear ? :D


5. `*(undefined *)((long)__ptr + 0x20) = 0;` is final step to add null terminator byte `00` to the end of stirng 

6. 
```c
      (**(code **)(*(long *)param_1 + 0x550))(param_1,param_2,lVar2);
      uVar3 = (**(code **)(*(long *)param_1 + 0x538))(param_1,__ptr);
      free(__ptr);
      return uVar3;
```


`_JNIENV+0x550` = function offset `170` (u now know how this calculated) at `_JNIENV` structure which is `ReleaseStringUTFChars`
free the encrypted utf8 String memory

`_JNIENV+0x538` = function offset `167`  at `_JNIENV` structure which is `NewStringUTF`
generate utf8 string from decrypted string at `__ptr` and store the new pointer for it at `uVar3`

then free decrypted ascii data at `__ptr` cause we have new utf8 verstion of decrypt text at `uVar3`
then return this utf8 pointer as `jstring`

let,s simulate the operation with `python` to decrypt key 
```python
key_bytes = [0x25,
    0x37, 0x3D, 0x19, 0x0E, 0x53, 0x05, 0x0C, 0x11, 0x02, 0x13, 0x4C, 0x16, 0x09, 
    0x4C, 0x13, 0x04, 0x5D, 0x5E, 0x03, 0x00, 0x0B, 0x44, 0x07, 0x15, 0x56, 0x42, 0x57, 
    0x55, 0x00, 0x01, 0x14
]
def xor_decrypt(input_bytes):
    string_length = len(input_bytes)
    decrypted_bytes = bytearray()

    for i in range(32): 
        decrypted_byte = input_bytes[i % string_length] ^ key_bytes[i]
        decrypted_bytes.append(decrypted_byte)

    return decrypted_bytes.decode('utf-8', errors='ignore') 
encrypted_data = bytearray("moiba1cybar8smart4sheriff4securi", 'utf-8')

decrypted = xor_decrypt(encrypted_data)
print("[+] Decrypted key:", decrypted)
```
and when running it we found the key again :)

![Hex Tree ghidra](/assets/img/hextree/hextree_jni_challenge_9.png)

thx for reading , see u later in next writeup <3


