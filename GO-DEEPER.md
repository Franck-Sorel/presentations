# Email Encryption, Inside Out

### A from-scratch tour of TLS, E2EE, the metadata gap, and the road to decentralized mail

*This is the written companion to the slide deck — everything in the deck, explained from the ground up. All facts and RFC citations verified in November 2026.*

---

If you've ever told someone "the email is encrypted," you were almost certainly wrong — or at least only a third right. Encrypted email is really three separate questions, and each has a different answer:

1. **Is the body private?** — that's *end-to-end encryption* (E2EE).
2. **Is the tunnel private?** — that's *TLS*.
3. **Is the *who-talks-to-whom* private?** — that's *metadata*, and it's the hard one.

The whole point of this walkthrough is to help you repair statements like "the email is encrypted" into something precise, and to make one decision by the end: **which layer actually protects the thing you care about?**

Let's start at the very beginning.

---

## 1. Why email needs encryption at all

Standard email is an **open postcard**. When you hit Send, your message travels through your client, your provider, several relay servers, and the recipient's provider:

```
You ──► SMTP ──► Your provider ──► relays ──► Recipient's provider ──► Recipient
       │           │                  │                    │
       └────────── readable at every hop, with no envelope ──────────┘
```

Without any encryption, **every one of those hops can read the content**, and the provider can also read it later while it sits in storage. The fix is to add layers — and there are exactly two worth caring about.

---

## 2. The two layers: TLS and E2EE

It's easy to blur these, so keep them separate:

| Layer | What it protects | Who can read the body afterward |
|---|---|---|
| **TLS** (transport) | The *connection* between servers | Your provider (and every endpoint) |
| **E2EE** (content) | The *message itself* | Only sender and recipient |

**TLS is the tunnel.** It encrypts the pipe between your client and the server, and between servers, so a passive wiretapper can't read the traffic in flight. But once the message *arrives*, the server sees it in plaintext. Two TLS hops still means your provider reads everything. TLS 1.3 (RFC 8446, updated by RFC 9846 in 2026) uses ephemeral Diffie-Hellman, so every session gets fresh keys — that gives you **forward secrecy** at the transport layer. It just doesn't protect content.

**E2EE is the content.** The message body is encrypted so only the sender and recipient hold the keys. The provider stores and relays a blob it cannot read. That's the layer people usually mean (and usually fail to get).

> **Mental model:** TLS puts mail in a locked box while it travels; E2EE hands the box to the recipient *already locked* with a key only they hold.

---

## 3. The two E2EE families: OpenPGP and S/MIME

Once you decide to do E2EE, you pick a trust model. The one question that separates the two families is:

> **How is a public key bound to a real person?**

### 3.1 PGP / OpenPGP — the Web of Trust

**PGP (Pretty Good Privacy)** is the original proprietary encryption software, created by **Phil Zimmermann in 1991** and now owned by **Broadcom**. **OpenPGP** is the open standard derived from it — defined by **RFC 4880** and updated by **RFC 9580** (July 2024) — so that different implementations can interoperate.

RFC 9580 is the current standard. It obsoletes RFC 4880 and adds **AEAD** authenticated-encryption modes (EAX/OCB/GCM), **Argon2** password hashing, modern curves **X25519/X448** and **Ed25519**, and **v6 keys & certificates**. One thing it does *not* add: **post-quantum crypto** (ML-KEM/ML-DSA/SLH-DSA) is still an IETF draft, not yet in any RFC.

OpenPGP's trust model is a **Web of Trust**: there is no central authority. Trust flows because *people sign each other's keys* (`gpg --sign-key`). If Alice signs Bob's key and Carol trusts Alice, Carol can trust Bob — via a chain of human vouching. The downside is that chains are brittle, and there's no authority to force revocation.

### 3.2 S/MIME — the X.509 CA hierarchy

S/MIME (current version 4.0, **RFC 8551**, built on the CMS **RFC 5652**) reuses the same **X.509 PKI** as TLS, but with email-specific fields. Instead of friends vouching for each other, a **Certificate Authority (CA)** vouches for you by signing a certificate that binds your public key to your mailbox. Your client trusts the CA because the CA's root certificate is already in its trust store.

The trust chain works like this:

```
Root CA (in your trust store)
   └─ signs ▸ Intermediate CA
          └─ signs ▸ Bob's certificate (SAN = email:bob@example.com)
                 └─ provides Bob's public key
```

Your client verifies each signature up to a root it already trusts; break any link and it warns or blocks.

### 3.3 Real platforms, one table

The same two protocols, deployed very differently:

| Platform | Standard | Subject encrypted? | Key storage | Notes |
|---|---|---|---|---|
| **Proton Mail** | OpenPGP | ❌ plaintext | Client-side (JS) | Body E2EE; subject plaintext for PGP interop |
| **Tuta** | **TutaCrypt** (proprietary) | ✅ | Client-side | Hybrid X25519 + Kyber-1024 (2024); avoids PGP for subject privacy |
| **Thunderbird** | OpenPGP + S/MIME | ✅ (OpenPGP) | Local (GPG) | "Encrypt Subject" **predates** RFC 9788 |
| **Apple Mail** | S/MIME | ❌ | Keychain / Secure Enclave | P-256 vs RSA split (see §7) |
| **Outlook (M365)** | S/MIME (+PGP add-ins) | ❌ | Windows cert store | Enterprise CA deployment |
| **Gmail Workspace** | S/MIME (hosted) | ❌ | Google-managed | Server-side S/MIME, GWSMO client |
| **PGPony (Android)** | OpenPGP | ✅ | **Smart cards** (NFC/USB) | Card-backed keys; active |
| **OpenKeychain (Android)** | OpenPGP | — | User-space RAM | Unmaintained (last v6.0.4, Feb 2024) |

---

## 4. What a certificate actually is

A certificate is a **signed public key** — it binds a key to an identity and is attested by a signature. It **never contains the private key**.

An X.509 v3 certificate (RFC 5280) is a signed data structure (ASN.1 DER-encoded) with three top-level parts:

| Part | Contents |
|---|---|
| **`tbsCertificate`** ("to be signed") | Version, serial, signature algorithm, issuer (CA's DN), validity, subject (owner's DN), subject public key, extensions |
| **`signatureAlgorithm`** | e.g. `sha256WithRSAEncryption` |
| **`signatureValue`** | the CA's digital signature over the DER-encoded `tbsCertificate` |

What distinguishes an *S/MIME* cert from a *TLS* cert:

| Field | S/MIME value |
|---|---|
| **Subject Alternative Name** | `email:alice@example.com` |
| **Extended Key Usage** | `id-kp-emailProtection` → **OID `1.3.6.1.5.5.7.3.4`** (⚠️ not `.3.12` — a common error) |
| **Key Usage** | `digitalSignature` + `keyEncipherment` |

### Generation, step by step

1. **Generate a key pair** — private (kept secret) + public:
   ```
   openssl genpkey -algorithm EC -pkeyopt ec_paramgen_curve:P-256 -out private.key
   ```
2. **Create a Certificate Signing Request (CSR)** — public key + identity + requested extensions:
   ```
   openssl req -new -key private.key -out request.csr \
       -subj "/CN=Alice/O=Acme/C=US" \
       -addext "subjectAltName=email:alice@acme.com" \
       -addext "extendedKeyUsage=emailProtection"
   ```
3. **The CA signs** — it hashes the CSR body and signs it with its *private* key:
   ```
   openssl x509 -req -in request.csr -CA ca.crt -CAkey ca.key \
       -CAcreateserial -days 365 -out alice.crt
   ```

The result, `alice.crt`, is what gets shared. The private key never leaves you.

---

## 5. The hybrid key exchange

Now the interesting part: how do you actually encrypt the message with the recipient's public key?

The naive answer — "encrypt the whole email with RSA" — is wrong because RSA/ECC is ~1,000× too slow for bulk data. The naive symmetric answer — "just share an AES key" — is wrong because you have no secure way to share it. The real solution is **hybrid encryption**:

1. Generate a **random one-time AES session key** (e.g. 256-bit) for this message.
2. **Encrypt the body** with AES — fast, this is the bulk of the work.
3. **Wrap the tiny session key** (≤256 bits) with the recipient's *public* key — slow, but the input is tiny so it's instant.
4. Send both together.

```
        session_key (AES)
            ├─► encrypt the BODY (fast, bulk)
            └─► wrap under recipient's PUBLIC key (slow, tiny)
                └─► recipient unwraps with PRIVATE key
                    └─► AES-decrypts the body
```

**Multi-recipient:** the body is encrypted once; the same session key is separately wrapped for each recipient. Each recipient finds their own wrapped copy.

### Sender encrypts (the concrete flow)

1. Generate a random AES-256 session key.
2. Encrypt the entire MIME body with it.
3. Encrypt the session key with the **recipient's public key**.
4. Package as a **CMS EnvelopedData** structure (RFC 5652): the encrypted content plus one **RecipientInfo** per recipient (issuer DN + serial to identify the cert, plus the encrypted session key).
5. Base64-encode and send as a MIME attachment (`application/pkcs7-mime, smime-type=enveloped-data`).

### Receiver validates & decrypts

1. Parse the MIME and find the EnvelopedData.
2. Find the RecipientInfo that matches *your* certificate (issuer DN + serial).
3. **Trust validation** (if the message is signed or you verify the sender): build the chain cert→intermediate→root, verify each signature, check validity dates and name chaining, confirm the root is in your trust store, optionally check revocation (CRL/OCSP), verify EKU = emailProtection and that the SAN email matches the `From:` header.
4. Unwrap the session key with your **private key**.
5. AES-decrypt the body.

> Trust isn't magic: the client cryptographically verifies the chain up to a root it already trusts. No chain → warning or block.

---

## 6. The secrets, and the forward-secrecy gap

Encryption rests on secrets with very different lifetimes and blast radii:

| Secret | Lifetime | If it leaks |
|---|---|---|
| **Private key** | Years | Entire mailbox history **and future** — no recovery, no rotation fixes the past |
| **Passphrase** | Years | Unlocks the private key → same as above |
| **Session key** | One message | Only that message |

The **private key is the crown jewel**. It's generated once and used for every message, so its compromise is catastrophic and unrecoverable. The **session key** limits damage to one message.

**The critical weakness: no forward secrecy.** PGP/S-MIME use a static private key for every session, so the session key is always wrapped by the same long-term key. Compare with TLS 1.3:

| Scenario | TLS 1.3 (ephemeral DH) | PGP / S-MIME (static key) |
|---|---|---|
| Private key stolen in 2026 | Past sessions safe (ephemeral keys already destroyed) | **All** past messages exposed |
| One session key intercepted | No added risk | Only that one message |

Messaging apps like Signal add **forward secrecy** with the *Double Ratchet* (ephemeral key exchange on top of long-term keys). Classic email E2EE has no equivalent.

---

## 7. Where does the crypto actually run?

The hybrid design maps to **two physically different places**:

| Operation | Where it runs | Why |
|---|---|---|
| Asymmetric — unwrap session key with private key | Secure hardware (if configured) | Small, fast, key never leaves |
| Symmetric — AES-decrypt the body | Regular CPU + RAM | Bulk data, needs speed, can't fit in an enclave |

The **session key is the bridge**: it must leave the secure hardware and enter regular memory to decrypt the body. That's the unavoidable gap. Every platform has the same shape:

```
┌────────────────────────────────────────────┐
│ 🔒 SECURE BOUNDARY (SE / TPM / smart card)  │
│   private key ─► asymmetric op ─► session key
└────────────────────────────────────────────┘
                  │  session key EXITS here (unavoidable)
                  ▼
┌────────────────────────────────────────────┐
│  REGULAR RAM / CPU                          │
│   session key ─► AES decrypt ─► plaintext   │
└────────────────────────────────────────────┘
```

### Platform by platform

- **Apple (Secure Enclave).** The SE only accepts **P-256 keys generated inside it** — you can't import an existing RSA or P-256 key, and it can't hold Ed25519. So S/MIME via **P-256 + SCEP** runs ECDH inside the SE (strong). But most CAs issue **RSA**, and RSA keys live in the **Keychain** (encrypted at rest, key material in RAM once unlocked) and decrypt in the Security framework in **regular memory**. The common case is therefore the weaker one.
- **Android (Keystore/TEE/StrongBox).** ⚠️ Correcting a common myth: Android Keystore/KeyMint **does support RSA decryption**. The real OpenPGP gap is that OpenPGP's X25519/Ed25519, multi-key packets, and its ECDH session-key envelope don't map onto the Keystore's constrained operations — and the unwrapped session key is returned to the app process *anyway*. So PGP apps run crypto in user-space RAM. **OpenKeychain** is unmaintained (last v6.0.4, Feb 2024). **PGPony** is hardware-key-first: keys on NFC/USB smart cards do the crypto on-card. The **BSI 2016** study concluded the Keystore API would need significant extension for OpenPGP.
- **GnuPG + TPM 2.0.** `gpg --keytotpm` wraps your private key with a TPM-specific key derived from the TPM's **storage seed**, producing an opaque blob stored as a `tpm2-v1` **shadowed key** that's useless without *that exact* chip. Decryption: gpg → gpg-agent → tpm2daemon → **TPM2_Load** → **TPM2_RSA_Decrypt** inside the TPM → session key returned. `tpm2daemon` exposes only 4 commands: **IMPORT, PKSIGN, PKDECRYPT, KILLTPM2D**. The TPM protects the private key; the session key is in RAM the moment it leaves.
- **GnuPG + smart card (YubiKey).** The key physically lives on the card's chip and is **non-exportable** — no ISO 7816 command can read it. Flow: gpg → gpg-agent → scdaemon → **VERIFY(PIN)** on-card (3 tries → lockout) → **DECRYPT** APDU → session key returned. The card is a self-contained HSM. The session key still exits the card.

### Threat model

| Attack | Protected | Exposed |
|---|---|---|
| Steal key file at rest | Private key (SE/TPM/card) | Nothing useful |
| Root OS / dump RAM | Private key (in hardware) | **Session key + plaintext** |
| Cold-boot (freeze RAM) | Private key (in hardware) | Session key + plaintext |
| Physical attack on chip | SE / StrongBox | TPM (not certified vs physical) |

**What closes the gap?** (1) doing AES inside the secure boundary — no current SE/TEE/TPM streams AES at mail speed; (2) **confidential computing** (SGX/CCA/TDX) — key + plaintext stay in encrypted CPU memory, the most promising direction though no client uses it; (3) **post-quantum + hardware** — ML-KEM (RFC 9936) is specified for S/MIME but nothing decrypts it yet.

---

## 8. The metadata problem

Everything above encrypts the **body**. But email still leaks an **envelope**:

- **Subject** — often the most sensitive field, still server-readable.
- **From / To** — required by SMTP for routing.
- **Timing & size** — who talks to whom, when, how often → a communication graph.

A PGP or S/MIME message is secret in content but public in envelope. The fixes fall into two buckets.

### Protocol-level fixes (emerging)

- **RFC 9788** — *Header Protection for Cryptographically Protected Email* (August 2025). Standardizes moving headers like Subject *into* the encrypted payload and replacing them with a placeholder (`…`), and gives end-to-end integrity for all headers. ⚠️ Thunderbird's "Encrypt Subject" **predates** RFC 9788 and is **not** an implementation of it. RFC 9788 still doesn't solve From/To — SMTP needs those visible for routing.
- **ECH (RFC 9849**, Encrypted Client Hello, March 2026) — encrypts the SNI (and ALPN) in the TLS handshake under an HPKE public key, so observers can't see which mail server you connect to.

### Architectural workarounds (available now)

- **Zero-access providers.** Tuta uses a **custom protocol** (TutaCrypt, hybrid X25519 + Kyber-1024) to encrypt subject, contacts, and calendar client-side. Proton encrypts the body but keeps subjects plaintext for OpenPGP interop.
- **Sealed sender** (pioneered by Signal, Oct 2018): the sender's identity is encrypted inside the envelope; the server sees only an opaque blob + a delivery token. There's no deployed email equivalent yet.

**Still unsolved:** From/To and timing/size remain exposed because SMTP requires them for routing. Fully hiding "who talks to whom" needs a radical redesign (sealed sender) or opaque routing tokens — no deployed standard exists.

---

## 9. Decentralized mail: DIDs and DIDComm

The idea that addresses the hardest part: replace human-readable addresses with **opaque, user-controlled identifiers**:

```
alice@gmail.com   →   did:key:z6Mk…   or   did:ethr:0x123…
```

An observer sees only an opaque string — no name, no domain. The **W3C DID spec explicitly lists privacy as a design goal** (W3C Recommendation, July 2022). Two methods worth knowing:

- **did:key** — generative and registry-less; expands a public key directly into a DID document.
- **did:ethr** — uPort/ethr-did, based on ERC-1056, uses an Ethereum address.
- ⚠️ **Terminology:** `did:key` is **not** Veres One — that's a separate method, `did:v1`. Don't conflate them.

**DIDComm v2** (DIF-ratified — *not* an IETF RFC) is the messaging layer on top of DIDs. Its two envelope modes are the reason DIDs can hide what SMTP can't:

| Mode | What it hides |
|---|---|
| **anoncrypt** (ECDH-ES) | The sender's identity from everyone except the recipient |
| **authcrypt** (ECDH-1PU) | Sender identity provable to the **recipient only**, not to intermediaries |

A mediator relays an opaque envelope and sees an opaque string, not who↔whom.

### DIDComm vs centralized email

| Dimension | Centralized (SMTP + E2EE) | DIDComm + DIDs |
|---|---|---|
| Identity | `name@domain` (resolvable) | Opaque `did:key:…` |
| Privacy | Metadata visible to relays | anoncrypt/authcrypt hide sender |
| Architecture | Central relays / MX | Mediators, transport-agnostic (SMTP possible) |
| Encryption | Hybrid AES + long-term keys | ECDH-ES / ECDH-1PU envelopes |
| Standard | IETF RFCs (mature) | DIF-ratified — no RFC, no DID-routed mail |
| Deployment | Ubiquitous | Experimental |

### Why DIDs are *not* a complete fix

1. **No deployed routing standard.** SMTP still needs a resolvable address; there's no RFC for DID-routed email. **Mailchain**, the ENS-based attempt, shut down in May 2026 — a strong sign this is experimental.
2. **Correlation is still possible.** Reuse a DID and observers map your communication graph (W3C DID Core §10.2 warns about this). Ephemeral DIDs fix that but break verification ("how do I know it's really you?") and add key-management overhead.
3. **DID documents can leak.** Resolution is a network request; if the doc contains a service endpoint like `https://alice.example.com/inbox`, the privacy benefit evaporates (§10.6).

**Where DIDs actually help:** as a **complement**, not a replacement — the DID provides the identity layer, sealed sender provides routing opacity; plus authentication without revealing identity, and selective disclosure ("prove I'm over 21") via zero-knowledge proofs. To fully close the metadata gap you still need the transport layer (sealed sender / mixnets, which weaken *linkability*) and ephemeral identity rotation. No single technology closes it alone.

---

## 10. The bottom line

- **Know your layers.** TLS protects the tunnel; E2EE protects the content; neither protects the *who-to-whom* metadata.
- **The private key is the crown jewel** — and it has no forward secrecy.
- **Secure hardware narrows** the blast radius but can't stop the session key from exiting into RAM.
- **Metadata is the unsolved frontier** — RFC 9788, ECH, zero-access providers, and sealed sender are the levers being pulled.
- **DIDs/DIDComm** are the roadmap, not the finished product.

*Companion to the Slidev deck. Every RFC, date, and platform claim above was verified against primary sources in November 2026; the notable corrections (EKU OID `.3.4`, Android Keystore supporting RSA decryption, PGPony being hardware-first, `did:key` ≠ Veres One, RFC 9580 not including post-quantum) are called out inline.*