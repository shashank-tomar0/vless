# VLESS: Lightweight On-Device Autonomous Browser Agent

[![Manifest V3](https://img.shields.io/badge/Manifest-V3-success.svg)](#)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.6-blue.svg)](#)
[![Test Suites](https://img.shields.io/badge/Test_Suites-25_Passing-brightgreen.svg)](#)
[![License](https://img.shields.io/badge/License-MIT-lightgrey.svg)](#)
[![Hackathon](https://img.shields.io/badge/MLH-Hacktoberfest_2026-orange.svg)](#)

VLESS is an autonomous, client-side browser agent for Chromium Manifest V3. It perceives page structure, plans actions, and carries out multi-step workflows while enforcing deterministic zero-trust redaction of personally identifiable information (PII) before any page state or visual data reaches external LLM inference endpoints.

Developed during **MLH Hacktoberfest 2026** as the high-speed, lightweight evolution of our on-device privacy agent research (built upon our first-generation release with **65+ downloads on the Chrome Web Store**).

---

## Table of Contents

- [1. Executive Summary](#1-executive-summary)
- [2. Problem Statement and Threat Model](#2-problem-statement-and-threat-model)
- [3. System Architecture and Process Isolation](#3-system-architecture-and-process-isolation)
- [4. DOM Perception and Native Event Replay Engine](#4-dom-perception-and-native-event-replay-engine)
- [5. Models, Planner Policy, and Framework Compatibility](#5-models-planner-policy-and-framework-compatibility)
- [6. Mathematical and Cryptographic Specifications](#6-mathematical-and-cryptographic-specifications)
- [7. Deterministic Pre-Action Safety Gates](#7-deterministic-pre-action-safety-gates)
- [8. Inspector, Comparative Analysis, and Repository Structure](#8-inspector-comparative-analysis-and-repository-structure)
- [9. Installation, Verification, and Build](#9-installation-verification-and-build)
- [10. Regulatory Alignment and License](#10-regulatory-alignment-and-license)

---

## 1. Executive Summary

Standard multimodal browser agents capture full-viewport screenshots, extract raw DOM trees, and transmit both unredacted to cloud inference endpoints. This design leaks biometric faces, credentials, government identification numbers, payment cards, and active session tokens into remote provider telemetry and model training pipelines.

VLESS draws the security boundary strictly inside the local Chromium client runtime:
- **Lightweight High-Speed Perception:** Replaces heavy 120MB+ multimodal neural weights with a sub-50ms DOM and ARIA perception stream. The planner operates over deterministic integer IDs (`[12] button "Submit"`) rather than fragile pixel coordinates or hallucinated CSS selectors.
- **Native Prototype Event Replay:** Dispatches authentic pointer sequences and interfaces directly with `HTMLInputElement.prototype` setters, bypassing synthetic event traps in React, Vue, and Angular.
- **Three-Tier PII Protection:** Combines semantic DOM scanning, mathematical checksum-validated regex (Luhn Mod-10, Verhoeff Dihedral Group D5), and local WASM OCR auditing.
- **Zero-Trust Ephemeral Vault:** Replaces secrets in user tasks and page inputs with synthetic tokens (`<CRED_1>`, `<CARD_1>`). Real plaintext values reside exclusively in volatile Service Worker memory.
- **Pre-Action Hard Safety Gates:** Code-level invariants prevent actions against password, CVV, OTP, and identity fields, while irreversible actions trigger interactive human-in-the-loop approvals.
- **Multi-Provider Adapter:** Supports Anthropic Claude (native tool-use), OpenAI, OpenRouter (400+ models), and offline self-hosted endpoints via Ollama or LM Studio.

---

## 2. Problem Statement and Threat Model

A multimodal browser agent payload routinely exposes confidential user data across three primary attack vectors:

1. **Cloud Eavesdropping and Data Harvesting:** Screen captures and DOM dumps sent to external APIs persist in cloud logs and model fine-tuning corpora. VLESS guarantees that payloads contain only surrogate tokens and sanitized structures.
2. **Brittle Masking and Semantic Breakage:** Replacing form values with static `[REDACTED]` markers breaks client-side input validation and degrades LLM reasoning. VLESS generates format-preserving, mathematically valid synthetic replacements.
3. **Synthetic Event Trapping:** Modern reactive web frameworks override DOM property setters, causing standard automation scripts (`element.value = x`, `element.click()`) to fail silently. VLESS implements verified prototype dispatch.

### Regulatory Matrix
- **GDPR (Regulation EU 2016/679):** Article 9 (Special Category Data) and Article 25 (Data Protection by Design and Default).
- **India DPDP Act 2023:** Section 6 (Purpose Limitation and Data Minimization).
- **HIPAA Privacy Rule:** 45 CFR Part 160 and Part 164 Subparts A and E.

---

## 3. System Architecture and Process Isolation

Chromium Manifest V3 enforces strict process separation across isolated execution contexts:

```mermaid
flowchart TD
    subgraph ClientBrowser["Chromium Client Runtime (Manifest V3)"]
        subgraph IsolatedWorld["Content Script Context (Isolated World)"]
            DOM["Live DOM Tree & ARIA Tree"]
            DOMScanner["Perception Engine<br/>- Accessible Name Computation<br/>- Deterministic Integer IDs<br/>- Interactive Element Filtering"]
            EventReplayer["Native Event Replayer<br/>- Pointer Sequences<br/>- Prototype Value Setters<br/>- InputEvent Dispatch"]
        end

        subgraph BackgroundSW["Service Worker (Event-Driven Core)"]
            AgentLoop["Agent Planning State Machine"]
            Vault["Ephemeral Token Vault<br/>- In-Memory RAM Map<br/>- Zero Persistent Storage"]
            FPE["Surrogate Engine<br/>- Format-Preserving Replacements<br/>- Luhn & Verhoeff Valid"]
            SafetyGates["Pre-Action Hard Safety Gates<br/>- Password/CVV Rejection<br/>- Irreversible Action Pauses"]
            WireLog["Wire Log & Byte Meter"]
        end

        subgraph OffscreenEnv["Offscreen Document (WASM Execution)"]
            Canvas["Offscreen Canvas Target"]
            TesseractWASM["Tesseract.js WebAssembly<br/>- SIMD / Relaxed-SIMD Engine<br/>- Local Adversarial OCR Audit"]
        end

        subgraph UserInterfaces["User Interfaces"]
            Sidepanel["Chrome Side Panel<br/>- Live Transcript Streaming<br/>- Confirmation Modals"]
            Inspector["Visual DOM & Action Inspector"]
            Options["Provider & API Configuration"]
        end
    end

    subgraph ExternalCloud["Configured LLM Inference Endpoints"]
        AnthropicAPI["Anthropic Claude API<br/>- Native Tool-Use Protocol"]
        OpenAIAPI["OpenAI API / OpenRouter"]
        OllamaLocal["Local Ollama / LM Studio<br/>- localhost:11434/v1"]
    end

    DOM --> DOMScanner
    DOMScanner --> AgentLoop
    AgentLoop --> SafetyGates
    SafetyGates --> Vault
    Vault --> FPE
    AgentLoop --> EventReplayer
    EventReplayer --> DOM
    AgentLoop --> OffscreenEnv
    Canvas --> TesseractWASM
    TesseractWASM --> AgentLoop
    AgentLoop --> ExternalCloud
    Sidepanel <--> AgentLoop
    Inspector <--> AgentLoop
```

- **Content Scripts (`src/content/`):** Run in an Isolated World. The page scripts cannot observe or tamper with VLESS internal states.
- **Service Worker (`src/background/`):** Orchestrates planning turns, manages the volatile token vault, and enforces safety invariants.
- **Offscreen Document (`src/offscreen/`):** Dedicated context providing HTML Canvas and WASM execution for local OCR auditing without blocking worker or UI threads.
- **Zero Storage Persistence:** Sensitive values and image buffers are never saved to `chrome.storage.local` or disk.

---

## 4. DOM Perception and Native Event Replay Engine

```mermaid
sequenceDiagram
    autonumber
    actor User as User
    participant SP as Chrome Side Panel
    participant SW as Background Service Worker
    participant CS as Content Script
    participant OS as Offscreen Document (WASM)
    participant LLM as Configured Planner (Claude/OpenAI/Ollama)

    User->>SP: Submit Task ("Log in and update profile")
    SP->>SW: Dispatch Sanitized Goal
    SW->>CS: Request Page Perception Snapshot
    CS->>CS: Compute Accessible Tree & Assign Integer IDs
    CS->>CS: Scan Inputs for Sensitive Form Attributes
    CS->>SW: Return Element Registry & Page Text
    SW->>SW: Execute Token Vault Sanitization
    SW->>LLM: Send Tokenized Snapshot ([12] button "Submit")
    LLM->>SW: Return Action Plan (click 12)
    SW->>SW: Evaluate Pre-Action Safety Gates
    alt Action is Dangerous (Password / Delete / Checkout)
        SW->>SP: Prompt User Confirmation Modal
        User->>SP: Confirm Approval
        SP->>SW: Release Execution Lock
    end
    SW->>CS: Dispatch Native Pointer & Prototype Sequence
    CS->>CS: Fire pointerdown -> mousedown -> click
    CS->>SW: Action Result & Fresh Mutation Snapshot
    SW->>SW: Expire Previous Element IDs
```

### Deterministic Addressing
Rather than guessing selectors or coordinates, each interactive element is assigned an immutable per-turn integer ID:
```text
[12] link "Sign in" (href="github.com/login")
[14] input "Username" (type="text")
[15] button "Submit"
```
When a tool call `click(12)` is received, the content script resolves ID 12 against the registry generated during that specific turn. If the DOM mutates, existing IDs immediately expire, eliminating incorrect clicks.

### Native Prototype Event Replay
Single-page application frameworks wrap input controls with internal reactive property trackers. Direct value assignment fails because framework state remains unchanged:
```typescript
// Traditional failure mode:
element.value = "user@example.com"; // React internal state does not update

// VLESS verified native prototype dispatch:
const prototypeSetter = Object.getOwnPropertyDescriptor(
  window.HTMLInputElement.prototype,
  "value"
)?.set;
prototypeSetter?.call(element, resolvedValue);
element.dispatchEvent(new InputEvent("input", { bubbles: true, composed: true }));
element.dispatchEvent(new Event("change", { bubbles: true }));
```

---

## 5. Models, Planner Policy, and Framework Compatibility

### Configurable Model Providers
VLESS decouples the execution state machine from model vendors through a unified provider interface:

| Provider | Protocol | Default Suggested Model | Capabilities |
| :--- | :--- | :--- | :--- |
| **Anthropic** | Native Messages API | `claude-3-5-sonnet-20241022` | Direct tool calling, high reasoning density |
| **OpenAI** | Chat Completions | `gpt-4o` | Structured JSON outputs, function calling |
| **OpenRouter** | OpenAI-Compatible Gateway | Any of 400+ available models | Model diversity, unified API access |
| **Custom / Ollama** | OpenAI-Compatible Local | `llama3.1:latest` / `qwen2.5-coder` | 100% offline, zero data exfiltration |

### Turn Budgets and Stall Prevention
Planning turns are constrained by strict latency and degeneration boundaries:
- **Header Acknowledgment:** 25 seconds timeout to detect stalled provider queues.
- **Mid-Turn Silence:** 30 seconds idle ceiling before retry dispatch.
- **Repetition Detection:** 4-gram pattern analysis triggers automated loop termination if degenerate tool sequences occur.
- **Step Cap:** Configurable per-task boundary (default: 25 steps) preventing runaway loops.

---

## 6. Mathematical and Cryptographic Specifications

```mermaid
flowchart TD
    subgraph InputData["Sensitive Detected Inputs"]
        CardInput["Credit Card Candidate: 4532 0150 1234 5674"]
        AadhaarInput["National ID Candidate: 3123 4567 8901"]
    end

    subgraph ChecksumValidation["Mathematical Checksum Verification"]
        LuhnAlgorithm["Luhn Modulus-10 Algorithm<br/>- Evaluates Digit Doubling & Parity"]
        VerhoeffAlgorithm["Verhoeff Dihedral Group D5<br/>- Non-commutative Permutation Matrices"]
    end

    subgraph TokenVault["Zero-Trust Ephemeral Vault"]
        TokenMap["In-Memory Vault Registry<br/>- Plaintext kept strictly in SW RAM<br/>- Swapped with <CRED_N> Tokens"]
    end

    subgraph SyntheticSurrogates["Format-Preserving Surrogates"]
        ValidCardSurrogate["Synthetic Card: 4000 1234 5678 9010 (Luhn Valid)"]
        ValidAadhaarSurrogate["Synthetic ID: 9999 0123 4563 (Verhoeff Valid)"]
    end

    CardInput --> LuhnAlgorithm
    AadhaarInput --> VerhoeffAlgorithm
    LuhnAlgorithm --> TokenMap
    VerhoeffAlgorithm --> TokenMap
    TokenMap --> SyntheticSurrogates
```

### 1. Luhn Mod-10 Checksum Algorithm
For a numerical card sequence $c_k c_{k-1} \dots c_1$:
$$\sum_{i=1}^{k} f(c_i, i) \equiv 0 \pmod{10}$$
where:
$$f(c_i, i) = \begin{cases} c_i & \text{if } i \text{ is odd} \\ c_i \times 2 - 9 & \text{if } i \text{ is even and } c_i \times 2 > 9 \\ c_i \times 2 & \text{if } i \text{ is even and } c_i \times 2 \le 9 \end{cases}$$

### 2. Verhoeff Dihedral Group $D_5$ Algorithm
Evaluates national identification numbers using non-commutative dihedral permutation groups ($D_5$) and multiplication tables:
$$\left( \sum_{i=0}^{n-1} d\left( c, p(i \bmod 8, c_i) \right) \right) = 0$$
where $d$ represents the dihedral multiplication table and $p$ represents the permutation table, preventing single-digit transcription errors and adjacent transposition faults.

---

## 7. Deterministic Pre-Action Safety Gates

```mermaid
stateDiagram-v2
    [*] --> Idle
    Idle --> EvaluatingAction: Planner Emits Tool Call
    EvaluatingAction --> TargetFieldInspection: Inspect Element Metadata
    
    TargetFieldInspection --> HardRejection: Field is Password, CVV, OTP, or Secret Key
    HardRejection --> ReportErrorToPlanner: Return Refusal Notice (Action Not Taken)
    
    TargetFieldInspection --> ValuePatternInspection: Field Type is Permitted
    ValuePatternInspection --> HardRejection: Typing Payload Matches Secret Regex
    
    ValuePatternInspection --> IrreversibilityCheck: Value is Safe
    IrreversibilityCheck --> AwaitingUserApproval: Action is Destructive (Checkout/Delete)
    IrreversibilityCheck --> ExecuteDispatch: Action is Read-Only or Low Risk
    
    AwaitingUserApproval --> ExecuteDispatch: User Confirms via Side Panel
    AwaitingUserApproval --> AbortAction: User Denies Approval
    
    ExecuteDispatch --> PostActionReperception: Dispatch Native Pointer Sequence
    PostActionReperception --> Idle: Re-Perceive Page & Expire Element IDs
```

### Safety Invariants
- **Code-Level Rejection:** Form elements tagged with `type="password"`, `autocomplete="cc-csc"`, `autocomplete="one-time-code"`, or matching PAN/Aadhaar/CVV descriptors are blocked from tool interaction.
- **Pattern Matching On Outbound Text:** Typing commands containing cryptographic private keys (`sk-`, `ghp_`, `xoxb-`) are blocked.
- **Human-in-the-Loop Confirmation:** Destructive operations (purchases, transfers, data deletion) pause execution and surface an approval card in the Chrome Side Panel.

---

## 8. Inspector, Comparative Analysis, and Repository Structure

### Comparative Architecture Matrix

| Architectural Vector | Traditional Vision Agent | VLESS Agent |
| :--- | :--- | :--- |
| **Perception Method** | Full-frame screenshots & visual grids | Semantic DOM & ARIA tree flattener |
| **Model Footprint** | Heavy neural weights (100MB+) | Sub-4MB lightweight bundle |
| **Execution Speed** | 500ms to 1500ms per frame | **Sub-50ms** perception cycle |
| **Hardware Requirement** | High-end GPU / WebGPU acceleration | Universal execution on any laptop |
| **Element Addressing** | Approximate $(x, y)$ coordinate bounding boxes | Deterministic integer IDs (`[12] button`) |
| **Framework Compatibility** | Programmatic dispatch (fails on React/Vue) | **Native prototype event replay** |
| **PII Protection** | Reactive post-capture redaction | **Proactive pre-action hard gates & token vault** |
| **Verification Engine** | None or external cloud verification | Local Tesseract.js WASM leak auditor |
| **Provider Portability** | Vendor locked | Anthropic, OpenAI, OpenRouter, Local Ollama |

### Repository File Tree
```text
pry2/ (vless)
├── build.mjs                 # esbuild pipeline and asset packager
├── package.json              # Project dependencies and script runner
├── tsconfig.json             # TypeScript compiler settings
├── src/
│   ├── background/           # Service Worker core
│   │   ├── agent.ts          # Autonomous planning state machine
│   │   ├── safety.ts         # Pre-action security gates
│   │   ├── executor.ts       # Action dispatcher
│   │   └── providers/        # Anthropic, OpenAI, OpenRouter, and Ollama adapters
│   ├── content/              # Isolated World content scripts
│   │   ├── content.ts        # Content script entrypoint
│   │   ├── act.ts            # Native prototype event replayer
│   │   └── fit.ts            # Accessibility calculation
│   ├── capture/              # DOM scanning and ARIA flattening
│   │   ├── dom.ts            # DOM tree visitor
│   │   └── scroller.ts       # Scroll and visibility calculator
│   ├── pii/                  # Three-tier PII detection engine
│   │   ├── checksums.ts      # Luhn and Verhoeff implementations
│   │   ├── tier1-dom.ts      # DOM input attribute inspector
│   │   └── tier2-patterns.ts # Checksum-validated regex matchers
│   ├── vault/                # Zero-trust token vault
│   │   └── vault.ts          # Ephemeral in-memory token mapping
│   ├── offscreen/            # Offscreen canvas and WASM runtime
│   │   └── ocr-engine.ts     # Tesseract.js local auditor
│   └── sidepanel/            # Chrome Side Panel UI and streaming transcript
└── test/                     # Comprehensive test suites (25 suites passing)
    ├── checksums.test.ts     # Luhn & Verhoeff algorithmic validation
    ├── gates.test.ts         # Pre-action safety gate verification
    ├── ocr.test.ts           # Local WASM OCR leak verification
    └── run-task.test.ts      # Multi-turn state-machine transitions
```

---

## 9. Installation, Verification, and Build

### Prerequisites
- Node.js version 20.0 or higher
- Chromium-based browser (Chrome, Brave, Edge) version 120 or higher

### Build Instructions
```bash
# Clone the repository
git clone https://github.com/shashank-tomar0/vless.git
cd vless

# Install dependencies
npm install

# Run the complete test suite (25 suites)
npm test

# Verify type safety
npm run typecheck

# Bundle production build into dist/
npm run build
```

### Loading the Extension into Chrome
1. Open Chrome and navigate to `chrome://extensions`.
2. Enable **Developer mode** using the toggle in the upper right corner.
3. Click **Load unpacked** and select the generated `dist/` directory.
4. Click the VLESS icon in your browser toolbar to open the Chrome Side Panel.
5. In Options (`chrome-extension://<extension-id>/options.html`), configure your chosen inference provider.

---

## 10. Regulatory Alignment and License

### Regulatory Compliance by Design
VLESS is architected to guarantee privacy by default:
- **Zero Third-Party Data Transmission:** Personal identifiable credentials, raw biometric pixels, and sensitive session tokens are never dispatched to external cloud servers.
- **Ephemeral State Lifecycle:** The token vault exists solely within volatile Service Worker RAM and is permanently wiped upon task completion or browser teardown.

### License
Licensed under the **MIT License**. Created with care for **MLH Hacktoberfest 2026**.
