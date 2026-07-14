# DWTL
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
