# Vanguard Technical Documentation
> Research documentation of Riot Games' Vanguard anti-cheat system (vgc.exe / vgk.sys)

---

## Architecture Overview

Vanguard consists of two components:

**vgk.sys** — Kernel Driver. Runs in Ring-0, prevents other kernel drivers from accessing the Valorant process. Primary function: memory protection, callback registration, image load verification. Starts as a Windows Service before vgc or the game are loaded.

**vgc.exe** — Usermode Client. Complete network layer of Vanguard. Responsible for hardware fingerprinting, auth handshake, telemetry and heartbeat. Communicates with vgk via IOCTL over a kernel device.

---

## Service & Driver Control

vgc manages vgk as a Windows Service via SCM:

| Function | Purpose |
|---|---|
| `OpenSCManagerW` / `CreateServiceW` / `StartServiceW` / `DeleteService` | Full service lifecycle management |
| `SetServiceObjectSecurity` / `QueryServiceObjectSecurity` | Service object hardened against tampering |
| `RegisterServiceCtrlHandlerExW` / `SetServiceStatus` | vgc runs as its own service dispatcher |

---

## Crypto Layer

From the IAT of `vgc.exe`:

| Import | Purpose |
|---|---|
| `BCryptGenRandom` / `CryptGenRandom` / `SystemFunction036` | RNG for session keys and token generation |
| `WinVerifyTrust` + full CRYPT32 stack | PE signature validation, cert chain verification |
| `CertVerifyTimeValidity` | Cert expiry check |

**Session encryption:**
- AES-256-GCM session keys
- Wrapped with RSA-OAEP SHA-512

---

## Hardware Fingerprinting

vgc collects the following data to construct `machine_id` and HWID:

| API | Data Collected |
|---|---|
| `GetSystemFirmwareTable` | SMBIOS tables (board serial, UUID) |
| `GetIpNetTable` | ARP table, MAC addresses |
| `GetBestRoute` | Network adapter identification |
| `gethostname` / `gethostbyname` | Hostname resolution |
| — | CPU, GPU, OS info (F11) |
| — | Security flags: HVCI, IOMMU, SecureBoot, TPM2, VBS (F14) |

---

## Protobuf Schema

### AuthenticationRequest

Gateway payload is an encrypted protobuf with the following fields:

```proto
F1  = machine_id
F2  = PlatformInfo / Sub2 { A=1, B=2, version="10.0.19045" }
F4  = game_token (JWT)
F5  = VGW_CLIENT_PUBKEY (RSA-2048 SPKI base64)
F6  = vg_version { A=major, B=minor, C=patch, D=build }
F7  = vgk_version { A=major, B=minor, C=patch, D=build }
F8  = game_id (string)
F9  = boot_state (int, value: 3)
F10 = ephemeral_id (optional, only send if already cached)
F11 = core_info { CPU, GPU, OS }
F13 = external_sid (PUUID)
F14 = { HVCI=1, IOMMU=1, SB=1, TPM2=1, VBS=1 }
F15 = { ht: value } base64_sha1 derived from machine_id
```

> **F10** is a temporary crypto identity per session — P-256 ECDH-derived keys + nonce. Only send if already cached.
> **F15 (ht)** is a hash token — base64-encoded SHA1 derived from machine_id. Server recomputes and verifies the pair on every request.

### vg_version Fields

```
A = major
B = minor
C = patch
D = build
```

Current known version: `1.18.3.77`

### Sub2 / PlatformInfo Fields

```
A = 1        (platform)
B = 2        (arch)
version      (OS version string e.g. "10.0.19045")
```

### AuthenticationResponse Fields

```
getServerRsaPublicKey()   ← live server pubkey for subsequent seals
getToken()                ← session token used in AccessRequest
```

### AccessRequest Fields

```
token   ← token from AuthenticationResponse
```

### Game IDs

```
com.riotgames.valorant
com.riotgames.league
```

---

## Envelope Format

### Outer Wrapper

```
08 <type_byte> 12 <varint(payload_len)> <rito_payload>
```

### Rito Payload Layout

```
52 47 01 00       ← RG\x01\x00 magic (4 bytes)
<256 bytes>       ← RSA-OAEP SHA-512 wrapped AES-256 key
<12 bytes>        ← AES-GCM IV
<N bytes>         ← AES-256-GCM encrypted protobuf
<16 bytes>        ← GCM authentication tag
```

### Message Type Bytes

| Byte | Message | Direction |
|---|---|---|
| `\x03` | AuthRequest | client → server |
| `\x04` | AccessRequest | client → server |
| `\x05` | ServerPubKey delivery | server → client |
| `\x07` | Heartbeat | client → server |

### Response Payload Layout (server → client)

```
<9 byte header>
<256 bytes>    ← RSA-OAEP encrypted AES key (decrypt with client RSA private key)
<12 bytes>     ← AES-GCM IV
<N bytes>      ← AES-256-GCM ciphertext
<16 bytes>     ← GCM tag
```

---

## Network Endpoints

All 6 known subdomains:

| Endpoint | Port | Infra | Status |
|---|---|---|---|
| `logs.vg.ac.pvp.net` | 443 | AWS CloudFront | Active |
| `na.vg.ac.pvp.net` | 8443 | Cloudflare | Active |
| `eu.vg.ac.pvp.net` | 8443 | Cloudflare | Active |
| `telemetry.vg.ac.pvp.net` | 443 | AWS CloudFront | Active |
| `mod.vg.ac.pvp.net` | 443 | — | Denied without client cert |
| `qa.vg.ac.pvp.net` | — | — | Inactive / IP-restricted |

### Resolved IPs (August 2026)

| Endpoint | IPs |
|---|---|
| `na.vg.ac.pvp.net` | `104.18.41.168`, `172.64.146.88` |
| `eu.vg.ac.pvp.net` | `172.64.146.88`, `104.18.41.168` |
| `logs.vg.ac.pvp.net` | `18.66.192.47`, `18.66.192.16`, `18.66.192.8`, `18.66.192.125` |
| `telemetry.vg.ac.pvp.net` | `108.138.36.9` |

### Request Format

```
POST https://{region}.vg.ac.pvp.net:8443/vanguard/v1/gateway
Content-Type: application/x-protobuf
Body: <raw binary envelope>
```

---

## TLS Behavior

### Gateway (port 8443)
- TLS 1.3
- Cloudflare terminates TLS
- Server requests mTLS renegotiation on every connection
- Client cert validated at **application layer** (protobuf F5), not at TLS layer
- Cloudflare accepts connections without client cert — backend rejects on invalid payload

### logs / telemetry (port 443)
- TLS 1.3
- No mTLS
- AWS CloudFront
- `SSLKEYLOGFILE` works for traffic decryption

---

## Connection Flow & Timing

```
vgk.sys boots (as Windows Service)
        │
        ▼
vgc.exe starts
        │
        ▼
~3.6s  logs.vg.ac.pvp.net:443
       Hardware fingerprint upload (~25KB)
       Fields: F1, F6, F7, F9, F11, F14, F15
        │
        ▼
~8.0s  na.vg.ac.pvp.net:8443 ──────────── telemetry.vg.ac.pvp.net:443
       Sealed AuthRequest (\x03)           Parallel telemetry stream
       Fields: F1-F15
       Client → 7 bytes
       Server → 24 bytes
       Immediate FIN both sides
        │
        ▼
       AccessRequest (\x04)
       Token from AuthenticationResponse
        │
        ▼
   [Session active]
   Modules + Tasks processing via vgk IOCTL
        │
        ▼
~57s   eu.vg.ac.pvp.net:8443
       Heartbeat (\x07)
       Identical envelope format
       Region switch NA → EU
```

---

## Gateway Oracle

```
HTTP 400  →  wrong key / bad payload layout / bad decrypt / wrong Content-Type
HTTP 401  →  key decrypts correctly → auth failure (wrong account / ban)
HTTP 2xx  →  success
```

---

## Session Lifecycle

A valid gateway session alone is **not sufficient** to maintain a live session. The full session lifecycle requires:

1. **AuthRequest** (`\x03`) — initial sealed protobuf with machine fingerprint
2. **AuthenticationResponse** — server delivers session token + server RSA pubkey
3. **AccessRequest** (`\x04`) — sealed with server pubkey, contains session token
4. **Modules + Tasks** — randomized module data delivered via vgk IOCTL responses, must be processed correctly to keep session alive
5. **Heartbeat** (`\x07`) — periodic sealed keepalive, same envelope format as AccessRequest

Without correct module and task handling from vgk IOCTL the session will drop regardless of a valid init sequence.

---

## Anti-Debug & Process Protection

From the IAT:

| Import | Purpose |
|---|---|
| `IsDebuggerPresent` / `OutputDebugStringW` / `DebugBreak` | Debug detection |
| `SetUnhandledExceptionFilter` / `RtlAddVectoredContinueHandler` | Exception handler manipulation |
| `NtCreateThreadEx` | Remote thread detection |
| `LdrLoadDll` | Hooked at kernel level to detect DLL injection |
| `NtCreateSection` / `NtMapViewOfSection` | Section-based mapping monitored |

---

## IOCTL / Kernel Communication

- vgc communicates with vgk via `NtDeviceIoControlFile`
- vgk IOCTL responses can be intercepted via hypervisor hook on `NtDeviceIoControlFile`
- IRP hooks via Drivermon as an alternative
- Kernel callbacks registered by vgk after startup — RWX pages must be allocated **before** this point
- A decryption key exists inside `vgk.sys` that decrypts IOCTL data passed through the driver

**Process suspend/resume IOCTL codes (from vgk):**
```
suspendio: CTL_CODE(FILE_DEVICE_UNKNOWN, 0xBA6B4A3, METHOD_BUFFERED, FILE_SPECIAL_ACCESS)
resumeio:  CTL_CODE(FILE_DEVICE_UNKNOWN, 0xBA6B4B5, METHOD_BUFFERED, FILE_SPECIAL_ACCESS)
```

---

## Session & Token Management

| Import | Purpose |
|---|---|
| `WTSEnumerateSessionsW` / `WTSQueryUserToken` | Session enumeration for cross-session launch |
| `DuplicateTokenEx` / `CreateProcessAsUserW` | vgk launches vgc under correct user token from service context |
| `ProcessIdToSessionId` / `WTSGetActiveConsoleSessionId` | Active user session identification |
| `ldap_init` / `ldap_search_s` (WLDAP32) | Active Directory / domain check, environment detection |

---

## Pipe Communication (Legacy — Patched)

> **Status: Patched.** Pipe emulation via `WriteFile` / `ReadFile` hooks is no longer effective as of early 2026. Documented here for historical reference and internal communication research only.

### Named Pipe Device

```
\\.\{7C3D5F2A-91B4-44A7-8F1E-13A697D42C8}
```

vgc communicates with the Valorant game process over a named pipe. The pipe is tracked via `WriteFile`, `ReadFile`, and `PeekNamedPipe` hooks injected into the Valorant process.

### Packet Signatures

Pipe identification signatures written by vgc on init:

```
Signature 1:  65 00 00 00 56 00 00 00 04
Signature 2:  03 00 00 00 28 00 00 00 01
```

### Packet Types

| Packet | Bytes | Size | Description |
|---|---|---|---|
| Heartbeat | `03 00 00 00 28 ...` | 40 bytes | Periodic liveness check |
| Check | `67 00 00 00 44 ...` | variable | Integrity check packet |
| Drop filter | `02 ?? ?? ?? 24 ...` | variable | Silently dropped on real read |

### Heartbeat Response

vgc sends a 40-byte heartbeat, expects a 40-byte response with the first byte flipped:

```
Request:   03 00 00 00 28 ...
Response:  04 00 00 00 28 ...
```

### PeekNamedPipe Behavior

| fakedResponse[0] | lpTotalBytesAvail | Description |
|---|---|---|
| `0x04` | `0` | Response already consumed |
| `0x03` | `40` | Heartbeat response pending |
| other | `0` | Nothing available |

---

## Notes

- `mod.vg.ac.pvp.net` — likely ban/enforcement lookup endpoint, hardened beyond the standard gateway
- `qa.vg.ac.pvp.net` — staging endpoint, inactive in normal client builds or restricted to Riot internal IPs
- The live server RSA public key (Pack A) is **not static** — delivered in `AuthenticationResponse`, all static/embedded keys return HTTP 400
- vgc uses a **statically linked HTTP implementation** over raw Winsock — no `curl.dll` or `winhttp.dll` in PEB
- Pipe emulation was viable in early iterations of Vanguard but has since been addressed — current research focus is on the gateway crypto layer and IOCTL module/task handling
- Init session alone does not keep a session alive — module and task processing from vgk IOCTL is required
````
