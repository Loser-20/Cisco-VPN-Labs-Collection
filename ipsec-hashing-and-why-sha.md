
# Hashing in IPsec/IKE: Types and Why We Use SHA

This note explains, in clear practical terms, **what hashing is in VPNs**, **where it is used (Phase 1 and Phase 2)**, **common hash types (MD5, SHA-1, SHA-256/384/512)**, and **why SHA—especially SHA‑256—is the current enterprise default**. I’ve also added real‑world tips, sample configs, and quick verification pointers.

---

## What Is a Hash?
A **hash** is a one‑way function that converts data into a fixed‑length **digital fingerprint** (the *digest*). In IPsec/IKE, hashes are used for **integrity** and **authentication** (of messages and peers). If even a single bit changes in transit, the recalculated hash won’t match, so the packet is rejected.

> Bottom line: **Hashing proves data was not modified** (tamper detection) and helps authenticate peers during negotiation.

---

## Where Hashing Appears in a VPN

### 1) IKE / ISAKMP (Phase 1)
- Used to protect the *control plane* (negotiation messages) during tunnel setup.
- Ensures that identity payloads, nonces, and DH parameters aren’t altered in transit.
- Defined in your **ISAKMP/IKE policy**.

```bash
crypto isakmp policy 10
 encr aes 256
 hash sha256         # Hash algorithm for Phase 1 integrity
 group 14
 authentication pre-share
 lifetime 86400
```

### 2) IPsec ESP (Phase 2)
- Used to protect the *data plane* (actual encrypted packets between subnets).
- Implemented as **ESP authentication** (often written as `esp-sha256-hmac`, etc.).
- Defined in your **transform-set** (IKEv1) or **IPsec proposal** (IKEv2).

```bash
crypto ipsec transform-set TS1 esp-aes 256 esp-sha256-hmac  # Data integrity/auth
```

> Think of Phase 1 hashing as protecting the **tunnel setup conversation**, and Phase 2 hashing as protecting **every data packet** that flows after the tunnel is established.

---

## Common Hash Types (and What They Mean)

### MD5
- **128-bit** digest, very fast, but **cryptographically broken** (collisions are feasible).
- Avoid in modern deployments.

### SHA-1
- **160-bit** digest, was a long‑time default; now considered **weak** due to collision attacks.
- Kept only for legacy compatibility.

### SHA-2 Family (Recommended)
- **SHA‑256**, **SHA‑384**, **SHA‑512**: modern, collision‑resistant, and widely supported.
- **SHA‑256** hits the sweet spot of **security + performance** on most gear (often hardware‑accelerated).

> **Today’s default:** Use **SHA‑256** for both Phase 1 and Phase 2 integrity wherever possible.

---

## Why Use SHA (and Prefer SHA‑256)?

1. **Security**: SHA‑2 (e.g., SHA‑256) is resistant to known collision attacks; MD5 and SHA‑1 are not.
2. **Interoperability**: All major vendors (Cisco, Juniper, Palo Alto, Fortinet, Check Point, AWS/Azure/GCP) support SHA‑2 broadly.
3. **Performance**: Modern routers/firewalls often accelerate SHA‑256 in hardware, giving strong integrity checks without a big CPU hit.
4. **Compliance**: Many frameworks discourage or forbid MD5/SHA‑1. SHA‑256 is a safe default for audits.

---

## Phase 1 vs Phase 2: How Hashing Is Used

| Stage | What It Protects | How It’s Configured | Example |
|------|-------------------|---------------------|---------|
| **Phase 1 (IKE/ISAKMP)** | Negotiation/control messages (IDs, nonces, DH values, auth payloads) | **ISAKMP/IKE policy** | `hash sha256` |
| **Phase 2 (IPsec/ESP)** | Actual encrypted data packets | **Transform‑set** (IKEv1) / **IPsec proposal** (IKEv2) | `esp-sha256-hmac` |

> Integrity in Phase 2 is often provided via HMAC (e.g., **HMAC‑SHA‑256**), which combines the hash with a secret key for stronger authentication.

---

## Good, Better, Best (Practical Recommendations)

- **Minimum (legacy interop only)**: `SHA-1` (use only if the peer can’t do better).
- **Recommended (enterprise default)**: `SHA-256` in Phase 1 and `esp-sha256-hmac` in Phase 2.
- **High‑security**: `SHA‑384` or `SHA‑512` if both ends support and performance is acceptable.

**Example (strong, practical):**
```bash
# Phase 1 (IKEv1 sample)
crypto isakmp policy 10
 encr aes 256
 hash sha256
 group 14
 authentication pre-share
 lifetime 86400

# Phase 2 (transform-set)
crypto ipsec transform-set TS-STRONG esp-aes 256 esp-sha256-hmac
```

For IKEv2, the idea is the same; you’ll specify integrity under the IKEv2 proposal/profile and the IPsec proposal.

---

## What If You Don’t Specify a Hash?
On many Cisco IOS/IOS‑XE systems, if you omit `hash` in Phase 1, the device applies a **default** (historically, SHA for hash in IKEv1; defaults can vary by code). Relying on defaults risks **mismatch** or **weaker security**. Always **explicitly** set your hash to avoid surprises and to meet compliance.

> Tip: Explicit configs make troubleshooting and audits much easier.

---

## Quick Verification

```bash
# Check Phase 1 policy and negotiated SA
a) show crypto isakmp policy
b) show crypto isakmp sa detail

# Check Phase 2 SAs and integrity stats
c) show crypto ipsec sa | include local|remote|auth|packets
```

Look for the negotiated **hash/integrity** values (e.g., `sha256`) and ensure packet counters increment during testing.

---

## TL;DR
- **Hashing = integrity and authentication** for VPN control/data traffic.
- Use **SHA‑256** as the standard choice today; avoid MD5/SHA‑1.
- Define hashing **explicitly** in both Phase 1 (ISAKMP/IKE) and Phase 2 (ESP) to ensure security, interoperability, and clean troubleshooting.

---

*Prepared for: Pratik (Sr Analyst) — concise field guide you can paste into change notes.*
