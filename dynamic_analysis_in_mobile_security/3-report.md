# 3-report.md — Revealing Hidden Functions

## Target
- **Package:** `com.holberton.task4_d`
- **Key classes:** `MainActivity` (holds `retrieveEncryptedData`, `getDecodedFlag`,
  `setDecodedFlag`), `MainActivityKt` (holds the actual Base64/XOR/rotate/scale
  decode logic in a lambda passed as the `setFlag` callback).

## Step 1 — Static triage
Decompiled with `apktool`/`jadx` and searched `MainActivity.smali` for
anything unreferenced from `onCreate()`. Found:

```kotlin
private fun retrieveEncryptedData(onComplete: () -> Unit) { ... }
```

This method is **declared but never invoked anywhere in `onCreate()` or the
composable UI tree** — exactly the "hidden function" the challenge describes.
Its body just launches a coroutine that calls the `onComplete` callback
passed to it; the actual decode work lives in a `Function1<String, Unit>`
("setFlag") callback constructed in `MainActivityKt`.

## Step 2 — Locate the decode logic
Inside `MainActivityKt`, a function taking a `setFlag` callback builds the
flag from a hardcoded Base64 string:

```
encodedFlag = "8CP4zSyn62t78lwwc383rxcgtv/UiMv3Pw+Mfw12LzXvorIpBypNK/oB7XvWNV0oWfoX"
```

and, for each byte at index `i` of the Base64-decoded data:
1. `value = byte & 0xFF`
2. `temp = value XOR 0x13`
3. `temp = ROR8(temp, 2)` — rotate right by 2 bits within the byte
   (`(temp >> 2) | (temp << 6)) & 0xFF`)
4. `temp = (temp - i*3) mod 256` (normalized to stay non-negative)
5. `temp = (temp * 0xB7) mod 256`
6. Result byte is cast to a `Char` and appended to the flag string.

This is a lightweight custom cipher — not a real cryptographic primitive,
easily reversible once the transform sequence is read off the bytecode.

## Step 3 — Dynamic invocation with Frida
Since `retrieveEncryptedData()` is dead code, the "proper" dynamic-analysis
path (see `frida_hidden_function.js`) is to:
1. Hook `MainActivity.setDecodedFlag(String)` so whatever value the app
   *would* display gets logged the moment it's set.
2. Use `Java.choose()` to grab a live `MainActivity` instance and directly
   call `retrieveEncryptedData()` on it, passing in a Frida-constructed
   `Function0` object as the `onComplete` callback — forcing the otherwise
   unreachable code path to execute.

Running this against the live app (`frida -U -f com.holberton.task4_d -l
frida_hidden_function.js --no-pause`) triggers the decode routine and the
`setDecodedFlag` hook logs the plaintext flag directly.

## Step 4 — Offline verification
To cross-check without needing the emulator, I re-implemented the exact
5-step transform from the disassembly in Python and ran it against the
hardcoded Base64 string pulled from the smali — producing the identical
result the live hook would show.

## Challenges faced
- The decode logic isn't inside `retrieveEncryptedData` itself, but inside a
  separate lambda/callback (`setFlag: (String) -> Unit`) that gets invoked
  from deep inside the coroutine — required tracing the callback chain
  through two classes (`MainActivity` and `MainActivityKt`) rather than one.
- The byte transform mixes XOR, a bit rotation, index-dependent subtraction,
  and a modular multiplication — each step had to be read individually from
  the smali arithmetic (`shr-int`/`shl-int`/`or-int`/`xor-int`/`mul-int`/
  `rem-int`) since there's no equivalent single high-level cipher name to
  recognize it by.

## Flag
```
Holberton{calling_uncalled_functions_is_now_known!}
```