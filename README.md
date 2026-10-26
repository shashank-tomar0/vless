# VLESS: Lightweight On-Device Privacy Browser Agent

[![Manifest V3](https://img.shields.io/badge/Manifest-V3-success.svg)](#)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.6-blue.svg)](#)
[![Test Suites](https://img.shields.io/badge/Test_Suites-25_Passing-brightgreen.svg)](#)
[![License](https://img.shields.io/badge/License-MIT-lightgrey.svg)](#)
[![MLH Hacktoberfest](https://img.shields.io/badge/MLH-Hacktoberfest_2026-orange.svg)](#)

**VLESS** is a next-generation, high-speed autonomous browser agent for Chromium Manifest V3. It enables multi-step autonomous browsing, form completion, and workflow execution while deterministically preventing sensitive personal data (passwords, card numbers, national IDs, and credentials) from ever leaking to external LLM providers.

Built during **MLH Hacktoberfest 2026** as the streamlined, lightweight evolution of our on-device privacy agent research.

---

## The Architectural Evolution

| Dimension | Legacy Vision Agent | VLESS (This Repository) |
| :--- | :--- | :--- |
| **Architecture** | Heavy Multimodal Vision Mesh | Lightweight DOM and ARIA Perception Stream |
| **On-Device Weights** | Heavyweight neural models (100MB+) | Zero heavyweight model downloads (3MB Tesseract Wasm only) |
| **Hardware Requirement** | High (Requires WebGPU acceleration) | Universal (Runs smoothly on any standard machine or laptop) |
| **Execution Latency** | 400ms to 1200ms per frame analysis | **Sub-50ms** instant perception loop |
| **Element Addressing** | Coordinate-based pixel bounding boxes | Deterministic numbered element IDs (`[12] button "Submit"`) |
| **DOM Interaction** | Synthetic click simulation | **Native Prototype Event Replay** (bypasses React state traps) |
| **Safety Invariant** | Post-capture redaction and tripwire | **Pre-Action Hard Safety Gates** (refuses sensitive controls outright) |
| **OCR Auditing** | Offscreen Canvas re-audit | Local Tesseract.js WASM verification over rendered captures |
| **Provider Support** | Single provider locked | **Anthropic Claude, OpenAI, OpenRouter, and Local Ollama** |

---

## Key Features

### 1. Element IDs, Never Selectors
The model receives numbered interactive elements with contextual accessible names:
```text
[12] link "Sign in" (href="github.com/login")
[14] input "Username" (type="text")
[15] button "Submit"
```
The planner invokes actions strictly through integer IDs (`click(12)`). The model never writes or hallucinates CSS selectors or fragile XPaths. After every page mutation, element IDs immediately expire and the DOM is re-perceived.

### 2. Native Prototype Event Replay
Frameworks like React, Vue, and Angular override HTML input value setters with synthetic internal change trackers. Standard `el.value = x` or `el.click()` calls fail silently on modern SPAs. VLESS dispatches the full pointer sequence (`pointerdown` -> `mousedown` -> `focus` -> `pointerup` -> `mouseup` -> `click`) and calls `HTMLInputElement.prototype` setters directly, followed by simulated `InputEvent` dispatch.

### 3. Three-Tier On-Device PII Protection and Surrogates
- **Tier 1 (DOM and ARIA Context):** Scans element attributes, ARIA tags, autocomplete hints, and name attributes (`cc-number`, `password`, `tax-id`).
- **Tier 2 (Mathematical Checksums):** High-precision regex engines paired with **Luhn Mod-10** (Credit Cards) and **Verhoeff Dihedral Group D5** (Aadhaar and National IDs) validation.
- **Tier 3 (Local WASM OCR):** When visual verification is required, Tesseract.js runs inside an isolated offscreen document to audit rendered captures for residual text leaks.

### 4. Zero-Trust Ephemeral Vault
Sensitive values typed by the user or read from the screen are replaced with ephemeral tokens (`<CRED_1>`, `<CARD_1>`). Real plaintext values reside exclusively in the Service Worker's volatile RAM and are swapped back only at the exact instant a native typing event is dispatched to the page.

### 5. Deterministic Pre-Action Safety Gates
Unlike prompt-based instructions that models frequently ignore, VLESS executes code-level invariant gates before every action:
- Any action attempting to target password, CVV, OTP, or identity fields is hard-rejected.
- Any typing payload that matches credential or private key patterns is blocked.
- Irreversible actions (purchases, account deletions, external transfers) pause and require user confirmation in the sidepanel.

---

## Installation and Development

```bash
# Clone the repository
git clone https://github.com/shashank-tomar0/vless.git
cd vless

# Install dependencies
npm install

# Build the extension into dist/
npm run build

# Run comprehensive test suite (25 test suites)
npm test

# Run TypeScript typecheck
npm run typecheck
```

### Loading in Chromium or Chrome:
1. Navigate to `chrome://extensions`.
2. Toggle on **Developer mode** in the top right.
3. Click **Load unpacked** and select the `dist/` directory.
4. Click the VLESS icon to open the Chrome Side Panel.
5. In Options (`chrome-extension://<id>/options.html`), select your preferred model provider (Anthropic, OpenAI, OpenRouter, or Custom Ollama).

---

## Test Suite and Verification

VLESS is verified by **25 standalone test suites** covering:
- `accuracy.test.ts` and `consistency.test.ts`: Deterministic DOM perception and element stability
- `checksums.test.ts`: Luhn and Verhoeff validation correctness
- `gates.test.ts` and `safety.test.ts`: Pre-action security gate enforcement
- `ocr.test.ts` and `ocr-correct.test.ts`: Local WASM OCR leak detection
- `planner.test.ts` and `loop.test.ts`: Multi-turn state-machine transitions
- `vault.test.ts`: Tokenization and de-tokenization lifecycle

```text
25 suites passed.
0 typecheck errors.
```

---

## License
MIT License. Built for MLH Hacktoberfest 2026.
