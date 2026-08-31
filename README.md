Vanguard (vgc.exe / vgk.sys) — Full Technical Documentation
Architecture Overview

Vanguard consists of two components:

vgk.sys — Kernel Driver. Runs in Ring-0, prevents other kernel drivers from accessing the Valorant process. Primary function: memory protection, callback registration, image load verification. Starts as a Windows Service before vgc or the game are loaded.

vgc.exe — Usermode Client. Complete network layer of Vanguard. Responsible for hardware fingerprinting, auth handshake, telemetry and heartbeat. Communicates with vgk via IOCTL over a kernel device.

Service & Driver Control

vgc manages vgk as a Windows Service via SCM:

OpenSCManagerW / CreateServiceW / StartServiceW / DeleteService
SetServiceObjectSecurity / QueryServiceObjectSecurity — service object is hardened against tampering
RegisterServiceCtrlHandlerExW / SetServiceStatus — vgc itself runs as a service dispatcher
Crypto Layer (IAT-based)

From the IAT of vgc.exe:

BCryptGenRandom / CryptGenRandom / SystemFunction036 — RNG for session keys and token generation
WinVerifyTrust + full CRYPT32 stack — PE signature validation, cert chain verification
CertVerifyTimeValidity — cert expiry check
AES-256-GCM session keys wrapped with RSA-OAEP SHA-512
Hardware Fingerprinting

vgc collects the following data for machine_id and HWID:

GetSystemFirmwareTable — SMBIOS tables (board serial, UUID)
GetIpNetTable — ARP table, MAC addresses
GetBestRoute — network adapter identification
gethostname / gethostbyname — hostname resolution
CPU, GPU, OS info (F11 in protobuf)
Security flags: HVCI, IOMMU, SecureBoot, TPM2, VBS (F14)
Protobuf Schema (Auth Request)

Gateway payload is an encrypted protobuf with the following fields:

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

F10 is a temporary crypto identity per session — P-256 ECDH-derived keys + nonce. Only send if already cached.

Envelope Format
Magic:      RG\x01\x00
Encryption: AES-256-GCM
Key-Wrap:   RSA-OAEP SHA-512
AAD:        Message Type Byte
Types:      3 = AuthRequest, 4 = ServerResponse, 5 = ServerPubKey
Network Endpoints

All 6 known subdomains:

Endpoint	Port	Infra	Status
logs.vg.ac.pvp.net	443	AWS CloudFront	Active
na.vg.ac.pvp.net	8443	Cloudflare	Active
eu.vg.ac.pvp.net	8443	Cloudflare	Active
telemetry.vg.ac.pvp.net	443	AWS CloudFront	Active
mod.vg.ac.pvp.net	443	—	Denied without client cert
qa.vg.ac.pvp.net	—	—	Inactive / IP-restricted

Resolved IPs (as of August 2026):

na.vg.ac.pvp.net → 104.18.41.168, 172.64.146.88 (Cloudflare)
eu.vg.ac.pvp.net → 172.64.146.88, 104.18.41.168 (Cloudflare)
logs.vg.ac.pvp.net → 18.66.192.47, 18.66.192.16, 18.66.192.8, 18.66.192.125 (CloudFront)
telemetry.vg.ac.pvp.net → 108.138.36.9 (CloudFront)
TLS Behavior

Gateway (8443):

TLS 1.3
Cloudflare terminates TLS
Server requests mTLS renegotiation on every connection
Client cert validated at application layer (protobuf F5), not at TLS layer
Cloudflare accepts connections without client cert — backend rejects on invalid payload

logs / telemetry (443):

TLS 1.3
No mTLS
AWS CloudFront
SSLKEYLOGFILE should work here for decryption
Connection Flow & Timing
vgk.sys boots (as Windows Service)
        ↓
vgc.exe starts
        ↓
~3.6s  logs.vg.ac.pvp.net:443
       Hardware fingerprint upload (~25KB)
       Fields: F1, F6, F7, F9, F11, F14, F15
        ↓
~8.0s  na.vg.ac.pvp.net:8443          telemetry.vg.ac.pvp.net:443
       Sealed auth check               Parallel telemetry stream
       Fields: F1-F15
       Client → 7 bytes
       Server → 24 bytes
       Immediate FIN both sides
        ↓
[Session active]
        ↓
~57s   eu.vg.ac.pvp.net:8443
       Heartbeat re-auth
       Identical protobuf format
       Region switch NA → EU
Gateway Oracle

Validation rule directly at the endpoint:

HTTP 400 = wrong key / bad payload layout / bad decrypt
HTTP 401 = key decrypts correctly → auth failure (wrong account / ban)
HTTP 2xx = success
Anti-Debug / Process Protection

From the IAT:

IsDebuggerPresent / OutputDebugStringW / DebugBreak — debug detection
SetUnhandledExceptionFilter / RtlAddVectoredContinueHandler — exception handler manipulation
NtCreateThreadEx — remote thread detection
LdrLoadDll — hooked at kernel level to detect DLL injection
NtCreateSection / NtMapViewOfSection — section-based mapping is monitored
IOCTL / Kernel Communication
vgc communicates with vgk via NtDeviceIoControlFile
vgk IOCTL responses can be reversed via hypervisor hook on NtDeviceIoControlFile
IRP hooks via Drivermon as an alternative
Kernel callbacks are registered by vgk after startup — RWX pages must be allocated before this point
Session & Token Management
WTSEnumerateSessionsW / WTSQueryUserToken — session enumeration for cross-session launch
DuplicateTokenEx / CreateProcessAsUserW — vgk launches vgc under the correct user token from service context
ProcessIdToSessionId / WTSGetActiveConsoleSessionId — active user session identification
WLDAP32 imports (ldap_init, ldap_search_s etc.) — Active Directory / domain check, environment detection
