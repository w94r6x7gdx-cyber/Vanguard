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

Gateway payload is an encrypted protobuf with the following fields:

```proto
F1  = machine_id
F2  = PlatformInfo { platform=1, arch=2, os_version }
F4  = game_token (JWT)
F5  = VGW_CLIENT_PUBKEY
F6  = vg version
F7  = vgk version
F8  = game_id
F9  = boot_state
F10 = ephemeral_id (optional, only send if already cached)
F11 = core_info { CPU, GPU, OS }
F13 = external_sid (PUUID)
F14 = { HVCI=1, IOMMU=1, SB=1, TPM2=1, VBS=1 }
F15 = { ht: value } base64_sha1 derived from machine_id
```

> **F10** is a temporary crypto identity per session — P-256 ECDH-derived keys + nonce. Only send if already cached.
> **F15 (ht)** is a hash token — base64-encoded SHA1 derived from machine_id. Server recomputes and verifies the pair on every request.

---

## Envelope Format

```
Magic:      RG\x01\x00
Encryption: AES-256-GCM
Key-Wrap:   RSA-OAEP SHA-512
AAD:        Message Type Byte
```

| Type | Description |
|---|---|
| `3` | AuthRequest (client → server) |
| `4` | ServerResponse (server → client) |
| `5` | ServerPubKey delivery (server → client) |

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
       Sealed auth check                   Parallel telemetry stream
       Fields: F1-F15
       Client → 7 bytes
       Server → 24 bytes
       Immediate FIN both sides
        │
        ▼
   [Session active]
        │
        ▼
~57s   eu.vg.ac.pvp.net:8443
       Heartbeat re-auth
       Identical protobuf format
       Region switch NA → EU
```

---

## Gateway Oracle

```
HTTP 400  →  wrong key / bad payload layout / bad decrypt
HTTP 401  →  key decrypts correctly → auth failure (wrong account / ban)
HTTP 2xx  →  success
```

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

---

## Session & Token Management

| Import | Purpose |
|---|---|
| `WTSEnumerateSessionsW` / `WTSQueryUserToken` | Session enumeration for cross-session launch |
| `DuplicateTokenEx` / `CreateProcessAsUserW` | vgk launches vgc under correct user token from service context |
| `ProcessIdToSessionId` / `WTSGetActiveConsoleSessionId` | Active user session identification |
| `ldap_init` / `ldap_search_s` (WLDAP32) | Active Directory / domain check, environment detection |

---

## Notes

- `mod.vg.ac.pvp.net` — likely ban/enforcement lookup endpoint, hardened beyond the standard gateway
- `qa.vg.ac.pvp.net` — staging endpoint, inactive in normal client builds or restricted to Riot internal IPs
- The live server RSA public key (Pack A) is **not static** — delivered in type5 after session handshake, all static/embedded keys return HTTP 400
- vgc uses a **statically linked HTTP implementation** over raw Winsock — no `curl.dll` or `winhttp.dll` in PEB
```
