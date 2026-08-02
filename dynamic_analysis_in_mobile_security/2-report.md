# 2-report.md — Android Cryptography Challenge: Intercepting and Decrypting Data

> **Status note:** The APK uploaded for this task (`app-release-task2.apk`) decompiled to
> the same `com.holberton.task3` Fibonacci-decryption app from the earlier static
> analysis task — no `INTERNET` permission, no networking code, no AES/RSA anywhere
> in it. This report is a methodology template based on the task's actual
> requirements; the **findings sections are placeholders** to fill in once the
> correct target APK is confirmed and analyzed.

## Objective
Capture and manipulate HTTP requests between the Android app and its server,
analyze the app's cryptographic implementation (AES/RSA), and decrypt the
hidden flag.

## Step 1 — Environment setup
- Android emulator (or rooted device) with a user-installed CA certificate
  trusted at the system level, so TLS traffic can be intercepted without the
  app rejecting the connection outright.
- `mitmproxy` (or Burp Suite) configured as the device's HTTP/HTTPS proxy
  (`adb shell settings put global http_proxy <host>:<port>`, or via Wi-Fi
  proxy settings on the emulator).
- Confirm interception works against a known-good site before touching the
  target app, to rule out proxy/cert issues later.

## Step 2 — Static triage first
Before touching the network layer, decompile the APK (`apktool` + `jadx`) to
scope the work:
- Confirm the `INTERNET` permission and any `usesCleartextTraffic` /
  Network Security Config settings in the manifest.
- Search app code for `Cipher.getInstance(...)`, `SecretKeySpec`,
  `KeyGenerator`, `javax.crypto.*`, and any `OkHttp`/`Retrofit`/
  `HttpURLConnection` usage — this tells you which classes actually build
  requests and which classes do the encryption/decryption, so you know
  exactly what to hook or read.
- Note the server hostname/endpoint(s) the app talks to, and whether
  certificate/public-key pinning is present (`CertificatePinner`,
  `TrustManager` overrides, or a bundled cert in `res/raw` or `assets`) —
  this determines whether you need to bypass pinning before traffic becomes
  visible at all.

## Step 3 — Intercept traffic
- Launch the app with the proxy active and trigger whatever action causes it
  to talk to the server.
- In mitmproxy/Burp, log the full request (headers, body) and response
  (status, headers, body) for each call.
- If TLS pinning blocks interception: use Frida (or Objection's
  `android sslpinning disable`) to hook the app's `TrustManager`/
  `CertificatePinner` check and force it to accept the intercepting proxy's
  certificate.

## Step 4 — Identify the crypto scheme
From the decompiled code, determine:
- **Algorithm** (AES-CBC / AES-GCM / RSA / hybrid) and mode/padding used.
- **Where the key/IV come from** — hardcoded in code, derived from a
  password with a KDF (PBKDF2, etc.), fetched from the server at runtime, or
  stored in `SharedPreferences`/the Android Keystore. Hardcoded or
  client-derivable keys are the usual vulnerability this style of challenge
  is testing for.
- Whether the same key encrypts everything (static) or it's session-specific
  (would need to be captured per-run via Frida hook rather than reused).

## Step 5 — Modify/replay and decrypt
- Use Burp's Repeater (or mitmproxy's scripting) to replay a captured
  request with modified parameters, observing how the server or client
  reacts — useful for spotting logic flaws (e.g., a debug/plaintext fallback
  path) in addition to pure key recovery.
- With the key/IV identified, decrypt the captured response body offline
  (Python `pycryptodome`, or CyberChef for a fast manual pass) to recover the
  plaintext flag.

## Step 6 — Findings
| Item | Value |
|---|---|
| Algorithm | *(fill in)* |
| Key source | *(fill in — hardcoded / derived / server-issued)* |
| Pinning present? | *(fill in)* |
| Encrypted response sample | *(fill in)* |
| Decrypted flag | *(fill in)* |

## Challenges faced
*(fill in — e.g., pinning bypass required, obfuscated key derivation,
emulator detection, etc.)*

## Flag
```
(pending correct APK)
```