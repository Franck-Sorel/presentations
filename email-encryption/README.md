# Email Encryption, Inside Out

> Part of the **`presentations`** repo — this deck lives in its own directory, `email-encryption/`. Each presentation is a self-contained directory; run it from here.

A blog-style technical walkthrough presentation on email encryption — from the TLS vs E2EE split, through PGP/S-MIME internals and the metadata gap, to decentralized mail with DIDs + DIDComm.

Built with **Slidev** (markdown → interactive slides) with heavy **Mermaid** diagramming.

## Files

| File | What it is |
|---|---|
| `slides.md` | The presentation (39 slides, one concept per slide, sized Mermaid diagrams). Edit this to change the deck. |
| `GO-DEEPER.md` | The companion deep-dive reference — everything verified online, with RFC citations. Share this with anyone who wants the full internals. |
| `styles/index.css` | Global CSS (hides the on-screen slide-outline TOC panel, centers diagrams). |
| `dist/` | A pre-built static export you can deploy/host as-is. |
| `package.json` | Slidev + theme dependencies. |

## Run it

```bash
npm install          # first time only
npm run dev          # dev server with hot reload (open the printed URL)
npm run build        # static export to dist/
npm run export       # PDF export (needs playwright chromium)
```

## Structure (the narrative)

One concept per slide, per the "How we do presentations" playbook (transformations, worked examples, form-follows-example, comparisons as tables).

1. **Opening** — the decision you'll make; the journey.
2. **The problem** — email is a postcard; the threat model.
3. **TLS vs E2EE** — two layers that protect different things.
4. **The E2EE protocols** — PGP/OpenPGP (Web of Trust) vs S/MIME (X.509 CA), platform examples in one table, what a certificate is, generation.
5. **Hybrid key exchange** — why two algorithms, the hybrid trick, multi-recipient, receiver flow.
6. **Secrets & forward secrecy** — private key vs session key blast radius; the no-forward-secrecy gap.
7. **The metadata problem** — what's still exposed + RFC 9788, ECH, zero-access providers, sealed sender.
8. **Decentralized mail** — DIDs + DIDComm (anoncrypt/authcrypt) vs centralized email, and why DIDs aren't a complete fix.
9. **Where crypto runs** — Secure Enclave / TEE / TPM / smart card vs RAM, and the universal session-key-exit gap.
10. **Wrap** — the decision revisited; further reading.

## UI fixes applied

- **Slide-outline TOC bar** removed via `styles/index.css` (the `.autocomplete-list` top-right panel that used to stay visible and intercept clicks).
- **Mermaid sizing**: every diagram is scaled with `{scale: …}` per block so it fits the slide (verified in-browser, no overflow). The journey and hybrid were restructured to one-concept slides.

## Fact-checking notes

All dates and claims were verified against primary sources in **November 2026**. Key corrections baked in (a few common errors worth knowing):

- S/MIME EKU OID is **`1.3.6.1.5.5.7.3.4`**, not `.3.12`.
- Android Keystore **does** support RSA decryption — the OpenPGP gap is key types/schemes + no session-key boundary.
- PGPony is **hardware-key-first** (NFC/USB smart cards), not pure user-space RAM.
- OpenKeychain is unmaintained (Feb 2024) but **not** incompatible with modern Android.
- `did:key` ≠ Veres One (`did:v1`); DIDComm v2 is DIF-ratified, not an IETF RFC.
- RFC 9580 adds AEAD/X25519/v6 keys but **not** post-quantum (still a draft).
- Thunderbird's "Encrypt Subject" **predates** RFC 9788 and isn't an implementation of it.
- Mailchain (ENS-based email) shut down **May 2026**.

## Keyboard

`→` next · `←` back · `o` overview · `p` presenter mode · `f` fullscreen. (`?` for all shortcuts.)