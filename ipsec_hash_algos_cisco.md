
# IPSec VPN Hashing (Integrity) Algorithms on Cisco: Uses, Differences & Commands

> **Scope**: Cisco IOS/IOS-XE/ASA syntax for IKEv1/IKEv2 and IPsec (ESP/AH). Focus on integrity/hash (HMAC) algorithms used for *authentication/integrity*, not encryption. Includes AEAD notes (e.g., GCM/GMAC) where separate hash is not used.

---

## 1) Where Hashing Is Used in IPSec

- **IKE (Phase 1 / IKEv2 SA)**: Integrity protection for IKE messages during key exchange.
  - IKEv1: `hash` in `crypto isakmp policy` (MD5/SHA1)
  - IKEv2: `integrity` in `crypto ikev2 proposal` (SHA-2 family supported)
- **IPsec SA (Phase 2 / Child SA)**: Integrity (and optional authentication) for user data.
  - **ESP** with HMAC (e.g., `esp-sha-hmac`, `esp-sha256-hmac`)
  - **AH** provides integrity/auth only (rare in practice)
  - **AEAD ciphers** (e.g., `esp-gcm`) include integrity; **no separate hash configured**

---

## 2) Common Hash/Integrity Algorithms

> All below are used in HMAC mode unless explicitly noted.

| Algorithm | Cisco Keywords | Bit Length | Security/Status | Typical Use |
|---|---|---:|---|---|
| **MD5** | `md5`, `esp-md5-hmac` | 128 | **Deprecated** (collision attacks) | Legacy IKEv1/IPsec. Avoid in new designs.
| **SHA-1** | `sha`, `sha1`, `esp-sha-hmac` | 160 | **Weak/Legacy** (collision attacks) | Backward compatibility; avoid when possible.
| **SHA-256** | `sha256`, `esp-sha256-hmac` | 256 | **Recommended** | Modern baseline for IKEv2/IPsec integrity.
| **SHA-384** | `sha384`, `esp-sha384-hmac` | 384 | Strong | Higher-security environments; performance hit.
| **SHA-512** | `sha512`, `esp-sha512-hmac` | 512 | Strong | Highest assurance; highest CPU cost.
| **GMAC** (AES-GMAC) | `esp-gmac` | 128/192/256 | AEAD (auth-only) | Use when encryption separate; niche.
| **GCM** (AES-GCM) | `esp-gcm` | 128/192/256 | AEAD (enc+auth) | Preferred where supported; no separate HMAC hash.

**Notes**
- HMAC-SHA2 variants (256/384/512) are widely supported on recent Cisco IOS/IOS-XE/ASA.
- For **Suite-B**/BCP-grade profiles, use **SHA-256+** with AES-GCM where possible.
- AES-GCM/GMAC provide integrity internally (via GHASH); configuring an extra HMAC is unnecessary and not allowed.

---

## 3) Differences That Matter

1. **Security strength**: MD5 < SHA-1 < SHA-256 < SHA-384 < SHA-512. Use SHA-256 or stronger today.
2. **Performance**: Higher-bit hashes consume more CPU. SHA-256 is usually the best balance.
3. **Compatibility**: Some older peers only support MD5/SHA-1 (IKEv1). For new tunnels, prefer IKEv2 with SHA-256+.
4. **Operational behavior**: With **AEAD (GCM/GMAC)** you **do not** configure a separate integrity algorithm for ESP.

---

## 4) Cisco Configuration Examples

### 4.1 IKEv1 (Legacy) — Prefer avoiding in new builds
```cisco
! IKEv1 Phase 1
crypto isakmp policy 10
 encryption aes 256
 hash sha
 authentication pre-share
 group 14
 lifetime 86400
!
! Phase 2 / IPsec (ESP with HMAC-SHA256 if platform supports)
crypto ipsec transform-set TS1 esp-aes 256 esp-sha256-hmac
 mode tunnel
!
crypto map CMAP 10 ipsec-isakmp
 set peer 203.0.113.10
 set transform-set TS1
 match address ACL_VPN
```

> If peer cannot do SHA-256 on ESP, fall back to `esp-sha-hmac` (HMAC-SHA1). MD5 is discouraged.

### 4.2 IKEv2 (Recommended)
```cisco
! IKEv2 proposal with SHA-256 integrity
crypto ikev2 proposal IKEV2-PROP
 encryption aes-cbc-256 aes-cbc-128
 integrity sha256
 group 14 19
!
crypto ikev2 policy IKEV2-POLICY
 proposal IKEV2-PROP
!
! IPsec proposal: ESP with HMAC-SHA256
crypto ipsec transform-set TS-SHA256 esp-aes 256 esp-sha256-hmac
 mode tunnel
!
crypto map CMAP 10 ipsec-isakmp
 set peer 203.0.113.10
 set ikev2 ipsec-proposal TS-SHA256
 match address ACL_VPN
```

### 4.3 IKEv2 + AEAD (ESP-GCM) — No separate hash
```cisco
! IKEv2 SA uses SHA-256 integrity; ESP uses AES-GCM (enc+auth)
crypto ikev2 proposal IKEV2-GCM
 encryption aes-gcm-256 aes-gcm-128
 prf sha256
 group 14
!
! For ESP-GCM on IOS/IOS-XE (no extra esp-*-hmac)
crypto ipsec transform-set TS-GCM esp-gcm 256
 mode tunnel
!
crypto map CMAP 10 ipsec-isakmp
 set peer 203.0.113.10
 set ikev2 ipsec-proposal TS-GCM
 match address ACL_VPN
```

### 4.4 ASA Examples
```cisco
! IKEv2 proposal on ASA
crypto ikev2 policy 10
 encryption aes-256
 integrity sha256
 group 14
 prf sha256
 lifetime seconds 86400
!
! IPsec transform-set
crypto ipsec ikev2 ipsec-proposal IPSEC-PROP
 protocol esp encryption aes-256
 protocol esp integrity sha-256
```

> On ASA, for AEAD: `protocol esp encryption aes-gcm-256` (no `protocol esp integrity` line).

---

## 5) Verification & Troubleshooting Commands

```cisco
show crypto isakmp policy              ! IKEv1 hash listing
show crypto ikev2 proposal             ! IKEv2 integrity per proposal
show crypto ikev2 sa detail            ! IKEv2 SA negotiated integrity
show crypto ipsec transform-set        ! ESP/AH integrity settings
show crypto ipsec sa                   ! Per-SA algorithms in use
show run | sec crypto                  ! Config review
debug crypto ikev2 protocol            ! Live IKEv2 negotiation (use with care)
```

**Symptoms of mismatch**
- IKE SA fails to form: check IKE integrity set (`hash`/`integrity`) and PRF.
- Child SA fails: check ESP integrity (HMAC vs GCM). If one side uses `esp-gcm`, the other must also.

---

## 6) Best Practices

- Prefer **IKEv2** with **SHA-256** (or stronger) and **AES-GCM** for ESP where both peers support it.
- Avoid **MD5** and **SHA-1** except for compatibility with legacy peers.
- Keep proposals **ordered from strongest to weakest** to negotiate the best common suite.
- Monitor device CPU; high HMAC bit-lengths (SHA-512) increase overhead. Consider crypto offload/ASA/ISR with hardware acceleration.
- Audit tunnels periodically: `show crypto ikev2 sa detail`, `show crypto ipsec sa` to confirm negotiated suites.

---

## 7) Quick Reference: Minimal Working Sets

**Strong baseline (recommended):**
- IKEv2: AES-256, **integrity SHA-256**, DH Group 14/19, PRF SHA-256
- ESP: **AES-GCM-256** (no extra HMAC)

**If peer lacks GCM:**
- ESP: AES-256 + **HMAC-SHA256** (`esp-aes 256 esp-sha256-hmac`)

**Legacy holdout (not recommended):**
- IKEv1: `hash sha` (SHA-1)
- ESP: `esp-aes 256 esp-sha-hmac` (HMAC-SHA1)

---

## 8) Glossary
- **HMAC**: Keyed-hash message authentication code (integrity + authentication)
- **PRF**: Pseudorandom function (used in IKE; often same family as integrity hash)
- **AEAD**: Authenticated Encryption with Associated Data (e.g., GCM)
- **SA**: Security Association (IKE SA and IPsec/Child SA)

---

## 9) Compatibility Notes
- Ensure both peers advertise the same integrity families. Mixing AEAD (GCM) on one side and HMAC on the other will fail Child SA negotiation.
- Some older platforms require separate knobs for SHA-2 (e.g., `esp-sha256-hmac` not available on very old images).

---

*Author: Generated for operational reference.*
