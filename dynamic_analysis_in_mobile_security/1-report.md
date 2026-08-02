# 1-report.md — Hooking Native Functions in Android

## Target
- **Package:** `com.holberton.task2_d`
- **Native library:** `libnative-lib.so` (armeabi-v7a, arm64-v8a, x86, x86_64)
- **Target function:** `MainActivity.getSecretMessage()` — declared `native`, takes no
  arguments, backed by the JNI export
  `Java_com_holberton_task2_1d_MainActivity_getSecretMessage`.

## Step 1 — Static triage
Decompiled the APK with `apktool` to confirm `getSecretMessage()` is a native
method and to find the library name it loads (`native-lib`, via
`System.loadLibrary("native-lib")` in `MainActivity`'s static initializer).
This told us exactly which `.so` and which exported symbol to target with
Frida before touching the device.

## Step 2 — Enumerate native exports
```
frida -U -n com.holberton.task2_d -i
```
Confirms `Java_com_holberton_task2_1d_MainActivity_getSecretMessage` is
present in `libnative-lib.so`'s export table — this is the exact symbol name
Frida needs for `Module.findExportByName()`.

## Step 3 — Install and launch
```
adb install task1_d.apk
frida -U -f com.holberton.task2_d -l frida_hook_getSecretMessage.js --no-pause
```

## Step 4 — Hook the function
Two complementary hooks (see `frida_hook_getSecretMessage.js`):

1. **Java-side hook** on `MainActivity.getSecretMessage` (via `Java.use(...).implementation = ...`).
   This is the simplest and most reliable approach: it lets the native code
   run normally, then intercepts the *already-decoded* Java `String` the
   native side hands back — no manual JNI string extraction needed.
2. **Native-side `Interceptor.attach()`** on the JNI export itself, as a
   backup/demonstration of hooking at the native boundary directly (useful
   when there's no convenient Java wrapper, or to inspect the call before
   the return value is even converted to a Java string).

## Step 5 — Observe the result
The Java-side hook logs the return value directly to the Frida console the
moment the button/action in the app triggers `getSecretMessage()`:

```
[+] getSecretMessage() returned: Holberton{native_hooking_is_no_different_at_all}
```

The value is never rendered anywhere in the app's UI — it's computed
natively and (in the unmodified app) simply discarded or used elsewhere,
which is exactly why dynamic hooking is required to observe it, versus just
reading the screen.

## What the native code was doing (for context)
Disassembly of the native library showed:
- A 49-byte obfuscated string embedded in `.rodata`.
- A helper function computing the Fibonacci sequence.
- A loop subtracting `fib(i % 10)` from each byte of the obfuscated string to
  recover the plaintext flag.

This is a lightweight, non-cryptographic obfuscation — trivial to reverse
statically once you have the disassembly, but the point of this exercise is
that a real-world app might layer in anti-debugging, root detection, or
JNI-level string encryption specifically designed to make *static* recovery
much harder, at which point hooking the function at runtime and letting the
app do the decoding for you (as done here) becomes the practical path to the
data.

## Flag
```
Holberton{native_hooking_is_no_different_at_all}
```