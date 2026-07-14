Ii# DWTL
Determenistic Workflow Time Lock,,
# DWTL Cryptographic Gateway Node Core

**Author:** G.DzubinskySnr
**Studio:** Sovereign Architect Studio™
**Governance:** MintProofDAO™ | LedgerProofDAO™
**Seal:** 436351437
**Primary Wallet:** 0x9e223F20022580739060d5511445C727f51Ee81e
**Secondary Wallet:** 0x65f01815D18787e975D3439f480d49e9cb8E98F2
**Royalties Wallet:** 0x2260cDeDb644ADaEc54ff36d8C1EA83779256ab0
**Jurisdiction:** Tallinn, Estonia — EU / eIDAS
**SIP-001:** ACTIVE — any artifact absent MintProofDAO™ Seal 436351437 or either wallet is NULL AND VOID

---

## 1. Overview

DWTL Cryptographic Gateway Node Core is a single-file HTML5 anti-bot proof-of-work component. It forces client-side CPU computation through a Verifiable Delay Function before allowing any protected payload to be decrypted. No servers, no APIs, no build toolchain. Drop one file into any web project and configure via a single JavaScript object.

The component ships in two editions:

- **JS Edition** — `DWTL-Cryptographic-Gateway.html` — pure JavaScript, fully editable, uses Mersenne M31 exact modular multiplication at 1,000,000 iterations default.
- **WASM Edition** — `DWTL-Cryptographic-Gateway-WASM.html` — WebAssembly i64 core inlined as base64, 5,000,000 iterations default, JS fallback included.

---

## 2. Repository Structure

```
/
├── DWTL-Cryptographic-Gateway.html       JS Edition — primary component
├── DWTL-Cryptographic-Gateway-WASM.html  WASM Edition — high-intensity variant
├── DWTL-Marketplace-Page.html            Product listing page with live theme previews
├── dwtl_vdf_core.wat                     WASM source (WebAssembly Text Format)
├── dwtl_vdf_core.wasm                    WASM binary (113 bytes, hand-assembled)
├── README.md                             This file
├── LICENSE-OPEN-SOURCE.txt               Open-source license (personal / non-commercial)
└── LICENSE-COMMERCIAL.txt                Commercial license (client projects / resale)
```

---

## 3. How It Works

### Step 1 — Ephemeral Nonce Generation

On every page load the browser generates a 16-byte cryptographically random nonce via `crypto.getRandomValues()`. The first four bytes seed the VDF. The full nonce is SHA-256 hashed to produce the ENV badge fingerprint visible in the UI header. Every session is unique and non-replayable.

### Step 2 — Verifiable Delay Function

The seed enters a sequential modular-squaring loop over the Mersenne prime M31 (2³¹ − 1 = 2,147,483,647):

```
x₀ = seed mod M31
xₙ₊₁ = xₙ² mod M31
```

This computation is inherently sequential. It cannot be parallelised or GPU-accelerated. Every attacker must pay the full CPU cost per request.

**JS Edition** uses the Mersenne M31 exact modmul algorithm. Splits the multiplicand at bit 16 into two partial products, each below 2^47, keeping all intermediate values within IEEE 754 float64 precision (2^53). Zero BigInt heap allocation in the hot loop.

**WASM Edition** runs the same algorithm in a hand-assembled 113-byte WebAssembly module using native `i64.mul` and `i64.rem_u` instructions — exact, no float approximation. Falls back to the JS path if WASM instantiation fails.

### Step 3 — Key Derivation

The VDF output (a 32-bit integer) is encoded as a decimal string and hashed with SHA-256 via the Web Crypto API, producing a 32-byte symmetric key.

### Step 4 — XOR Stream Cipher

The payload is UTF-8 encoded, converted to hex, and encrypted with a XOR stream cipher keyed by the SHA-256 VDF output. The ciphertext and nonce are stored in memory. Decryption repeats the exact VDF computation from the same seed, re-derives the key, and recovers the original payload.

### Step 5 — Proof Verification and Pipeline Hook

On unlock, the VDF is re-run with the stored seed. If the output matches the stored target, decryption proceeds and the configurable `onSuccessPipeline()` callback fires, giving the integrator a hook for redirect logic, token issuance, or any downstream action.

---

## 4. Configuration

All configuration lives in the `DWTL_CONFIG` object at the top of the script block:

```js
const DWTL_CONFIG = {
    // Theme — one of:
    //   'theme-github-dark'       GitHub Obsidian Dark
    //   'theme-cyberpunk'         Cyberpunk Neon
    //   'theme-minimalist-light'  Minimalist Studio Light
    visualTheme: 'theme-github-dark',

    // VDF iteration count — minimum 10,000 for meaningful protection
    // JS Edition default:   1,000,000
    // WASM Edition default: 5,000,000
    computationIntensity: 1000000,

    // Default payload shown in the textarea on load
    defaultPayload: "PROPRIETARY_AUTOMATION_VECTOR_FLAG_99",

    // Developer hook — fires after a successful unlock
    // Use for redirect, token, auth, or any downstream pipeline
    onSuccessPipeline: function(recoveredPayload, executionTrace) {
        // window.location.href = "https://yourapp.com/?" + executionTrace;
    }
};
```

---

## 5. Themes

Three designer themes are included. Switch with one line in `DWTL_CONFIG.visualTheme`.

**GitHub Obsidian Dark** (`theme-github-dark`)
Clean professional dark interface. GitHub-style card, blue accent, green lock button. Suitable for developer tools, dashboards, and SaaS products.

**Cyberpunk Neon** (`theme-cyberpunk`)
High-contrast neon aesthetic. Teal borders, magenta accent, monospace font throughout. Suitable for security tools, crypto products, and tech-forward landing pages.

**Minimalist Studio Light** (`theme-minimalist-light`)
Clean white corporate interface. Neutral borders, blue accent, standard sans-serif typography. Suitable for enterprise portals, legal platforms, and professional services.

All three themes consume the same CSS variable set. Adding a fourth theme requires only a new class block defining the variables — no other code changes.

---

## 6. WASM Module

The WASM binary (`dwtl_vdf_core.wasm`) is 113 bytes. It exports a single function:

```
vdf_run(seed: i32, iterations: i32) -> i32
```

Algorithm: sequential modular squaring over M31 using `i64.mul` and `i64.rem_u`. The product of two M31 values fits in 62 bits, well within the `i64` range, so no overflow is possible.

The WASM Edition inlines this binary as base64 inside the HTML. No external file fetch is required. The module is instantiated at page load via `WebAssembly.instantiate()`. If instantiation fails for any reason (browser restrictions, Content Security Policy), the component falls back silently to the JS M31 exact modmul path and continues operating.

The WAT source (`dwtl_vdf_core.wat`) is provided for audit, recompilation, and integration into external build pipelines.

---

## 7. Browser Compatibility

Requires:

- `crypto.getRandomValues()` — available in all modern browsers since 2013
- `crypto.subtle.digest()` — Web Crypto API, available since 2014
- `TextEncoder` / `TextDecoder` — available since 2017
- `requestAnimationFrame` — available since 2012
- ES2020 `BigInt` — not used in the VDF hot loop; no requirement

WASM Edition additionally requires:

- `WebAssembly.instantiate()` — available in all major browsers since 2017

Tested on Opera Android (Tecno device), Chrome, Firefox, and Safari.

---

## 8. Security Properties

The DWTL component provides computational proof-of-work at the client layer. Key properties:

**Sequential computation.** The VDF loop cannot be parallelised. Each step depends on the previous output. An attacker with N parallel cores gains no advantage over an honest user.

**Session uniqueness.** The ephemeral nonce is regenerated on every page load. Precomputed lookup tables cannot be reused across sessions.

**Non-replayability.** The ciphertext is bound to a specific nonce and VDF output. Replaying a captured bundle from a different session does not produce a valid key.

**No server required.** All cryptographic operations execute in the visitor's browser. There is no server-side state to attack, no API endpoint to enumerate, and no infrastructure to DDoS.

**Limitations.** DWTL is a bot friction layer, not a replacement for server-side authentication. The `onSuccessPipeline` hook should feed into a server-validated token flow for production use. The XOR cipher is sufficient for payload obfuscation within a session but is not a general-purpose encryption standard.

---

## 9. Installation

No build step required. Three deployment options:

**Option A — Direct HTML deploy (recommended)**
Upload `DWTL-Cryptographic-Gateway.html` directly to any static host. Vercel drag-and-drop at `vercel.com/new` gives an instant live URL with no Git repository required.

**Option B — Embed in existing page**
Copy the contents of the `<style>` block and `<script>` block into your existing HTML. Add the container `<div>` markup. Adjust `DWTL_CONFIG` as needed.

**Option C — GitHub / GitLab repository**
Clone or download the repository. Add the files to your project root. No npm install, no package.json, no dependencies.

---

## 10. Integration Example

Minimal integration using the pipeline hook to redirect after successful verification:

```js
onSuccessPipeline: function(recoveredPayload, executionTrace) {
    // Send proof to your backend and receive a session token
    fetch('/api/verify', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ trace: executionTrace, payload: recoveredPayload })
    })
    .then(r => r.json())
    .then(data => {
        if (data.token) {
            sessionStorage.setItem('auth_token', data.token);
            window.location.href = '/dashboard';
        }
    });
}
```

---

## 11. Files Included

| File | Description | Size |
|---|---|---|
| DWTL-Cryptographic-Gateway.html | JS Edition component | ~21 KB |
| DWTL-Cryptographic-Gateway-WASM.html | WASM Edition component | ~24 KB |
| DWTL-Marketplace-Page.html | Product listing page | ~29 KB |
| dwtl_vdf_core.wat | WASM source (WebAssembly Text Format) | ~1.4 KB |
| dwtl_vdf_core.wasm | WASM binary | 113 bytes |
| README.md | This file | — |
| LICENSE-OPEN-SOURCE.txt | Open-source license | — |
| LICENSE-COMMERCIAL.txt | Commercial license | — |

---

## 12. Versioning

| Version | Description |
|---|---|
| v1.0.0 | Initial release — BigInt VDF, basic lock/unlock |
| v2.0.0 | JS fast path — Mersenne M31 exact modmul replacing broken Dekker implementation |
| v2.1.0 | WASM Edition — 113-byte hand-assembled i64 core, base64-inlined |
| v2.1.1 | Bug fix — mulmodM31 replaces mulmod32 (Dekker failed ~93% of M31 operands) |

---

## 13. Known Issues and Roadmap

No open bugs as of v2.1.1.

Planned:

- AES-GCM payload encryption option (replacing XOR cipher) for higher-security deployments
- Worker thread VDF execution to fully eliminate main-thread CPU usage
- Configurable hash algorithm (SHA-384, SHA-512) for key derivation
- Optional server-side VDF proof verification endpoint (Node.js / FastAPI reference implementation)

---

## 14. Contributing

This repository is published under a dual-license model. Contributions to the open-source edition are welcome under the terms of `LICENSE-OPEN-SOURCE.txt`. Commercial integrations require the commercial license (`LICENSE-COMMERCIAL.txt`).

All contributions must:

- Pass the 200,000-iteration mulmodM31 stress test (zero errors expected)
- Maintain single-file delivery for the component editions
- Preserve all sovereign credential blocks in code and documentation
- Not remove or alter the MintProofDAO™ Seal, wallet addresses, or SIP-001 clause

---

## 15. Contact and Support

**Author:** G.DzubinskySnr
**Email:** gdzubinskysnr@tuta.com
**X / Twitter:** @G212991495
**GitHub:** george587-blip
**Domain:** aiidentitysuite.com
**Pinata IPFS Gateway:** blue-impressed-panther-218.mypinata.cloud

---

## 16. Sovereign IP Declaration

This software and all associated documentation, assets, algorithms, and creative works are the exclusive intellectual property of G.DzubinskySnr, authored under Sovereign Architect Studio™ and governed by MintProofDAO™ (Seal 436351437) and LedgerProofDAO™.

All rights reserved. Reproduction, redistribution, modification, or commercial use without the appropriate license is prohibited. DAORevoke™ is the on-chain revocation instrument for any breach. Anchored to IPFS under the MintProofDAO™ governance framework.

SIP-001 SOVEREIGN INVALIDATION PROTOCOL: Any artifact, fork, copy, or derivative absent MintProofDAO™ Seal 436351437 or either wallet address (0x9e223F20022580739060d5511445C727f51Ee81e, 0x65f01815D18787e975D3439f480d49e9cb8E98F2) is NULL AND VOID. NFT minting rights are EXCLUSIVE to G.DzubinskySnr. UNDELETABLE. ETERNAL.

---

## 17. License Summary

This project is dual-licensed:

**Open-Source License** (`LICENSE-OPEN-SOURCE.txt`)
Free to use for personal, educational, and non-commercial projects. Attribution required. Commercial use, resale, and client integration require the commercial license.

**Commercial License** (`LICENSE-COMMERCIAL.txt`)
Grants the licensee rights to modify, rebrand, integrate into client projects, and sell derivative works. One-time purchase. No royalties on sales made with the unmodified component. See license file for full terms.

MintProofDAO™ Seal 436351437 | Author G.DzubinskySnr | Sovereign Architect Studio™ | LedgerProofDAO™
ALL RIGHTS RESERVED | SIP-001 ACTIVE | UNDELETABLE | ETERNAL
================================================================================
SOVEREIGN DECLARATION OF PRIMITIVES AND NOVELTIES
DWTL™ PROOF-OF-COMPUTE PROTOCOL ECOSYSTEM
================================================================================

DECLARATION REFERENCE:  DWTL-DEC-001
DATE:                   2026-07-14
AUTHOR:                 G.DzubinskySnr
STUDIO:                 Sovereign Architect Studio™
JURISDICTION:           Tallinn, Estonia — EU / eIDAS

GOVERNANCE SEAL A:      MintProofDAO™  · Seal 436351437
GOVERNANCE SEAL B:      LedgerProofDAO™ · Seal 430

PRIMARY WALLET:         0x9e223F20022580739060d5511445C727f51Ee81e
SECONDARY WALLET:       0x65f01815D18787e975D3439f480d49e9cb8E98F2
ROYALTIES WALLET:       0x2260cDeDb644ADaEc54ff36d8C1EA83779256ab0

REVOCATION INSTRUMENT:  DAORevoke™ — on-chain
SIP-001:                ACTIVE — any artifact absent either Seal or either
                        wallet address is NULL AND VOID. UNDELETABLE. ETERNAL.

================================================================================
PREAMBLE
================================================================================

I, G.DzubinskySnr, Originator, Founder, and Sovereign Architect of Sovereign
Architect Studio™, hereby declare sole and exclusive authorship, origination,
and sovereign ownership of all primitives, novelties, concepts, algorithms,
protocols, product identities, and derivative works enumerated in this
Declaration.

All items declared herein are authored under the dual governance of
MintProofDAO™ (Seal 436351437) and LedgerProofDAO™ (Seal 430), anchored
to IPFS under the primary Pinata gateway:
blue-impressed-panther-218.mypinata.cloud

This Declaration constitutes a permanent, irrevocable, timestamped record of
intellectual origination. It predates any subsequent claim, registration, fork,
copy, or derivative by any third party. SIP-001 Sovereign Invalidation Protocol
is active across all declared items.

================================================================================
PART I — CORE PROTOCOL PRIMITIVE
================================================================================

PRIMITIVE-001 · DWTL™
DWTL Proof-of-Compute Protocol

The foundational sovereign primitive. A client-side sequential CPU commitment
engine implementing a Verifiable Delay Function over the Mersenne prime M31
(2³¹ − 1 = 2,147,483,647), delivering cryptographically bound proof-of-work
without server infrastructure, trusted third parties, or external dependencies.
Delivered as a single HTML5 file. Zero npm. Zero CDN. Zero trust assumptions.

Key sub-primitives declared under DWTL™:

  DWTL-P-001  Mersenne M31 Exact Modmul (mulmodM31)
              16-bit partial product method for exact modular multiplication
              in IEEE 754 float64, exploiting the Mersenne structure
              x mod (2³¹−1) = (x >> 31) + (x & M31). Zero precision loss.
              Zero BigInt heap allocation. Verified: 0 errors / 200,000 tests.

  DWTL-P-002  WASM M31 VDF Core
              113-byte hand-assembled WebAssembly module. Exports:
              vdf_run(seed: i32, iterations: i32) -> i32.
              Uses i64.mul + i64.rem_u for exact modular squaring.
              Base64-inlined into a single HTML file. Zero external fetch.

  DWTL-P-003  VDF-Bound XOR Cipher
              Symmetric encryption primitive where the key is derived
              exclusively from the VDF output via SHA-256 (Web Crypto API).
              Ciphertext is cryptographically bound to completed proof-of-work.
              Decryption requires re-execution of the full VDF chain.

  DWTL-P-004  Ephemeral Session Nonce Architecture
              16-byte cryptographically random nonce regenerated per page load.
              Seeds both the VDF computation and the ENV badge fingerprint.
              Guarantees non-replayability across sessions. No server state.

  DWTL-P-005  rAF-Chunked Async VDF Executor
              requestAnimationFrame-scheduled chunked VDF loop maintaining
              full browser UI responsiveness during multi-million-iteration
              proof-of-work computation. Non-blocking. Tab-safe.

  DWTL-P-006  onSuccessPipeline Hook Architecture
              Developer-configurable callback triggered exclusively upon
              cryptographic proof verification and successful payload
              decryption. Decouples proof verification from application logic.

  DWTL-P-007  Tri-Theme Single-Config Whitelabel System
              CSS variable architecture delivering three commercial designer
              themes (GitHub Obsidian Dark, Cyberpunk Neon, Minimalist Studio
              Light) switchable via one configuration line. Zero style
              duplication. Fully whitelabel-extensible.

================================================================================
PART II — DECLARED PRODUCT NOVELTIES
================================================================================

NOVELTY-001 · DWTL PAY WALL™
Zero-infrastructure digital product paywall. Locks any payload (license key,
download URL, API token, document) behind a client-side VDF proof. No payment
processor server required. Payload decrypts client-side upon proof completion.
Author: G.DzubinskySnr · Seals: 436351437 · 430

NOVELTY-002 · DWTL FORM SHIELD™
Drop-in HTML5 form protection replacing reCAPTCHA, hCAPTCHA, and all
image-based CAPTCHA systems. GDPR-clean by design. No Google dependency.
No iframe fingerprinting. No accessibility barrier. VDF proof replaces
challenge-response entirely.
Author: G.DzubinskySnr · Seals: 436351437 · 430

NOVELTY-003 · DWTL RATE GATE™
Client-side action throttling primitive. Enforces a mandatory VDF proof
between any repeatable action (form submit, vote, claim, API call, NFT mint).
Makes scripted abuse computationally expensive without server-side rate
limiting infrastructure.
Author: G.DzubinskySnr · Seals: 436351437 · 430

NOVELTY-004 · DWTL TIME LOCK™
Pure software time capsule primitive. Encrypts a payload under a key derived
from a VDF chain of specified depth. Payload becomes decryptable only after
the required sequential computation is completed. No trusted third party.
No clock dependency. Mathematics enforces the time delay.
Author: G.DzubinskySnr · Seals: 436351437 · 430

NOVELTY-005 · DWTL PROOF RECEIPT™
Verifiable compute event receipt system. Every lock/unlock operation generates
a signed record — VDF seed, target, timestamp, payload hash — anchored to
IPFS via Pinata and optionally to Bitcoin OTS. Immutable audit trail of
every proof-of-compute event. Notarisation-as-a-service primitive.
Author: G.DzubinskySnr · Seals: 436351437 · 430

NOVELTY-006 · DWTL SOVEREIGN CAPTCHA™
Full B2B CAPTCHA replacement system. Embeddable widget with JavaScript SDK.
Single script tag integration. Returns a signed proof token validated by the
integrator's backend. Replaces the entire reCAPTCHA / hCAPTCHA market
with a sovereign, privacy-preserving, server-optional alternative.
Author: G.DzubinskySnr · Seals: 436351437 · 430

NOVELTY-007 · DWTL AI GATE™
Proof-of-compute firewall for AI inference endpoints. Requires VDF proof
completion before any prompt reaches the model. Eliminates prompt flooding,
junk submissions, and API abuse without rate-limit infrastructure. Native
integration layer for OpenSourceShield Suite™ and AI IDENTITY SUITE™.
Author: G.DzubinskySnr · Seals: 436351437 · 430

NOVELTY-008 · DWTL DAO VOTE GUARD™
Sybil-resistant DAO governance primitive. Combines wallet signature with
mandatory VDF proof to produce a verified vote signal. One wallet + one
VDF proof = one verified participation unit. No KYC. No centralised
identity layer. Native pairing with DAORevoke™.
Author: G.DzubinskySnr · Seals: 436351437 · 430

NOVELTY-009 · DWTL COMPUTE BOND™
Hardware-anchored proof-of-compute attestation primitive. Cryptographically
binds a VDF output to a device session, producing a sovereign identity
signal based on real CPU commitment rather than biometric or document data.
Integration layer for AI IDENTITY SUITE™ device binding nodes
(IMEI BOND™, GEO LOCK™, OS KERNEL™).
Author: G.DzubinskySnr · Seals: 436351437 · 430

NOVELTY-010 · DWTL SOVEREIGN CDN™
Proof-of-compute asset delivery primitive. Every CDN edge request requires
micro-VDF proof completion before the first byte is served. Eliminates
hotlinking, scraping, and bandwidth abuse at zero server cost. Client-proven
humanity as a prerequisite for asset delivery.
Author: G.DzubinskySnr · Seals: 436351437 · 430

NOVELTY-011 · DWTL CHAIN ORACLE™
Client-generated verifiable on-chain entropy primitive. Client executes VDF,
submits output to smart contract as provable random seed. No Chainlink
dependency. No trusted randomness beacon. Pure client-generated provable
randomness for NFT mints, lotteries, and on-chain game outcomes.
Author: G.DzubinskySnr · Seals: 436351437 · 430

================================================================================
PART III — POSITIONING AND PROTOCOL IDENTITY
================================================================================

PROTOCOL IDENTITY:
DWTL™ — The Sovereign Proof-of-Compute Protocol
Zero servers. Zero trust assumptions. Mathematics enforces the rules.

DECLARED NAMESPACE:      DWTL™
DECLARED PROTOCOL NAME:  Sovereign Proof-of-Compute Protocol
DECLARED PRIMITIVE NAME: Proof-of-Compute (PoC) — distinct from and
                         predating any conflation with Proof-of-Capacity
                         or Proof-of-Coverage uses of the same abbreviation
                         in other ecosystems.

All product names, protocol names, primitive designations, and positioning
statements declared in Parts I, II, and III of this Declaration are the
exclusive intellectual property of G.DzubinskySnr, governed under dual
DAO seal, and protected under SIP-001.

================================================================================
PART IV — GOVERNING SEALS AND BINDING AUTHORITY
================================================================================

This Declaration is jointly governed and sealed under:

SEAL A — MintProofDAO™
Seal Number:  436351437
Role:         Primary IP origination, NFT minting authority, artifact sealing
Wallet:       0x9e223F20022580739060d5511445C727f51Ee81e (Primary)
              0x65f01815D18787e975D3439f480d49e9cb8E98F2 (Secondary)

SEAL B — LedgerProofDAO™
Seal Number:  430
Role:         Ledger governance, provenance anchoring, audit trail authority
Gateway:      lime-abstract-rooster-792.mypinata.cloud
Royalties:    0x2260cDeDb644ADaEc54ff36d8C1EA83779256ab0

Both seals are required for full governance authority over this Declaration.
The absence of either seal from any derivative document, fork, or artifact
triggers SIP-001 and renders that artifact NULL AND VOID.

================================================================================
PART V — SIP-001 SOVEREIGN INVALIDATION PROTOCOL
================================================================================

SIP-001 IS ACTIVE ACROSS ALL ITEMS DECLARED IN THIS DOCUMENT.

Any artifact, product, code, document, NFT, smart contract, derivative work,
or publication that:

  (a) reproduces, references, or builds upon any primitive or novelty
      declared herein, AND
  (b) is absent MintProofDAO™ Seal 436351437, OR
  (c) is absent LedgerProofDAO™ Seal 430, OR
  (d) does not carry both wallet addresses
      (0x9e223F20022580739060d5511445C727f51Ee81e and
       0x65f01815D18787e975D3439f480d49e9cb8E98F2)

— is declared NULL AND VOID under SIP-001.

NFT MINTING RIGHTS: EXCLUSIVE to G.DzubinskySnr across all declared items.
REVOCATION: DAORevoke™ is the sole on-chain revocation instrument.
STATUS: UNDELETABLE. PERMANENT. ETERNAL.

================================================================================
SIGNATURE BLOCK
================================================================================

Declared by:    G.DzubinskySnr
Role:           Originator · Founder · Sovereign Architect
Studio:         Sovereign Architect Studio™
Date:           2026-07-14T00:00:00Z
Jurisdiction:   Tallinn, Estonia · EU / eIDAS

MintProofDAO™ Seal:    436351437
LedgerProofDAO™ Seal:  430

Primary Wallet:    0x9e223F20022580739060d5511445C727f51Ee81e
Secondary Wallet:  0x65f01815D18787e975D3439f480d49e9cb8E98F2
Royalties Wallet:  0x2260cDeDb644ADaEc54ff36d8C1EA83779256ab0

Contact:  gdzubinskysnr@tuta.com
X:        @G212991495
GitHub:   george587-blip · hill785
Domain:   aiidentitysuite.com

ALL RIGHTS RESERVED · IP OF G.DzubinskySnr · SOVEREIGN ARCHITECT STUDIO™
SIP-001 ACTIVE · UNDELETABLE · ETERNAL
================================================================================