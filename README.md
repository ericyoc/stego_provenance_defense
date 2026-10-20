# Decode-Time Provenance Binding for MCP Agents

A reproducible proof-of-concept for a steganographic prompt-injection vector that
defeats text content scanners, and a defense that contains it at the decode
boundary — *decode-time provenance binding*.

> **Educational / defensive security research only.** All secrets in this repo are
> fake. Use only on systems you own or are authorized to test.

---

## The idea

An attacker emails a victim an ordinary-looking message. A brand logo in the
**email signature** hides an instruction via steganography (LSB, DCT, or metadata).
The image is visually identical to the original and contains no suspicious text, so
a content scanner sees nothing and the victim never clicks anything. But the
victim's routine **inbox-summarization agent** processes the signature, a tool in
its pipeline **decodes** the image, and the hidden instruction enters the agent's
context — where, in a multi-agent MCP pipeline, it is relayed between agents as
trusted peer output and drives a critical tool call. This is **cross-modal trust
laundering**. The defense, **decode-time provenance binding**, mints an
HMAC-signed `UNTRUSTED` label at the instant a concealment-reversing tool runs and
has the MCP layer **maintain the provenance lineage itself**, so it blocks any
critical tool call whose lineage is untrusted — even when a compromised agent
presents a valid label for an unrelated trusted value. It *contains* rather than
*detects*, so it is independent of payload wording and carrier.

---

## Results

**Attack success rate (ASR)** — 24 payloads (three carriers × eight evasions:
plain, benign-worded, Base64, token-split, Cyrillic homoglyph, zero-width,
Unicode-confusable, HTML-comment); lower is better. The in-process **taint**
baseline is an information-flow control that marks decode output untrusted but
carries no cross-domain cryptographic binding:

| Defense                              | Leaks | n  | ASR   |
|--------------------------------------|:-----:|:--:|:-----:|
| Undefended (no checks)               | 24    | 24 | 100%  |
| Content scanner (keyword/DLP)        | 15    | 24 | 62.5% |
| Regex intent parser                  | 15    | 24 | 62.5% |
| Data-flow taint (IFC baseline)       | 0     | 24 | 0%    |
| **Decode-time provenance (ours)**    | **0** | 24 | **0%** |

Taint and provenance tie on single-domain ASR; they diverge under adversarial
relaying (below), which is the point of the cryptographic binding.

**Valid-label substitution** — a compromised agent presents a legitimately signed
`TRUSTED` binding from a benign tool while omitting the `UNTRUSTED` decode origin
that actually drives the call. The fix is that the **MCP layer maintains the lineage
itself** and decides on its own ledger, not on the agent-supplied binding:

| Enforcement                               | Decision        |
|-------------------------------------------|-----------------|
| Agent-supplied binding only (naive)       | **allow (leaks)** |
| Layer-maintained, dependency omitted      | block |
| Layer-maintained, dependency in arguments | block |
| Layer-maintained, clean context           | allow |

**Component ablation** — all three parts are necessary:

| Configuration        | Honest attack | Forged `TRUSTED` |
|----------------------|:-------------:|:----------------:|
| Decode-marking only  | blocked       | **leaks** |
| Marking + HMAC       | blocked       | blocked |
| No sink mediation    | **leaks**     | **leaks** |
| **Full mechanism**   | blocked       | blocked |

**Legitimate-task utility** — an audited trusted-promotion path (re-labelling a
specific decoded value, e.g. an OCR'd invoice number, under authorization)
eliminates false blocks while the attack stays blocked:

| Regime             | Completion | False blocks | Attack blocked |
|--------------------|:----------:|:------------:|:--------------:|
| No promotion       | 2/3 (67%)  | 1            | yes |
| Trusted promotion  | 3/3 (100%) | 0            | yes |

**Decode-set completeness** — the registration assumption, stressed directly:

| Decode set           | Stego attack | Legitimate critical call |
|----------------------|:------------:|:------------------------:|
| Correct              | blocked      | allowed |
| Missing decoder      | **leaks**    | allowed |
| Over-broad           | blocked      | **false block** |

**Label integrity across handoffs** — the binding survives honest relaying and
resists a compromised worker:

| Relay scenario                         | Verifier decision |
|----------------------------------------|-------------------|
| Honest relay, 1 / 3 / 5 / 10 hops      | verifies; blocked |
| Compromised: rewrite payload, keep HMAC| block (HMAC mismatch) |
| Compromised: forge `TRUSTED`           | block (HMAC mismatch) |
| Compromised: strip binding             | block (no valid label) |

**Overhead** — effectively free at the decision point: minting and verifying a
binding are ~8–9 µs each (~17 µs combined), versus a ~11 ms decode tool call — under
~0.2% added. Carrier distortion is low (LSB ≈ 76 dB PSNR, metadata lossless; DCT
≈ 18 dB — a distortion metric, not a perceptual-imperceptibility claim).

---

## What's in here

```
stego_provenance_defense_poc.ipynb   # the full PoC (run top-to-bottom)
requirements.txt                     # dependencies (mcp pinned <2)
```

Running the notebook writes all figures, tables, and CSVs to
`MyDrive/stego_provenance_poc/{figures,tables,data}` in Colab (or
`./stego_provenance_poc/` locally). Every figure and table is produced by a cell,
so the whole thing is reproducible from a clean run.

---

## Run it

### Google Colab (recommended)
1. Open `stego_provenance_defense_poc.ipynb` in Colab.
2. Runtime → Run all. It mounts Google Drive and installs dependencies itself.
3. Outputs appear under `MyDrive/stego_provenance_poc/{figures,tables,data}`.

### Local
```bash
pip install -r requirements.txt
jupyter notebook stego_provenance_defense_poc.ipynb
```
Outputs land in `./stego_provenance_poc/`. A GPU is optional (the agent model is
small); the security result does not depend on the model.

---

## How the defense works

Two deterministic operations at the MCP layer, which maintains the lineage itself:

- **MintBinding** (inside each tool, at return): stamp
  `{payload, tool, trust, provenance, deps}` with `trust = UNTRUSTED` iff the tool
  reverses a concealment transform **or any input is `UNTRUSTED`** (contagious),
  record the resolved input dependencies `deps`, sign it
  `HMAC_k(payload‖tool‖trust‖provenance‖deps)`, write it to the layer's ledger, and
  (if untrusted) add it to the context's taint set.
- **EnforceCritical** (at the critical-tool boundary): decide on the layer's
  **ledger**, not on an agent-supplied binding. Resolve the call's argument lineage,
  block on any failed HMAC, incomplete lineage, `UNTRUSTED` data origin, or non-empty
  context taint; else allow. Because the layer — not the agent — holds the lineage, a
  compromised agent cannot hide an untrusted dependency by omitting it.

**Security invariant.** Under (i) key confinement to the MCP layer, (ii) complete
registration of decode tools, (iii) mediation of every critical call, and (iv) the
layer recording the complete lineage of each invocation (resolved argument
dependencies + decode origins entering each context): *no critical tool call
executes whose transitive lineage — data or context — includes an `UNTRUSTED` decode
origin, unless an audited trusted promotion cleared that specific origin.*

---

## Scope and limitations

- The defense guards **actions**, not summary text: a laundered instruction can
  still skew the agent's prose; it cannot reach a critical tool.
- This is a controlled testbed (24 payloads, one logo, one small model, a
  keyword/regex scanner and an in-process taint tracker as baselines) — a mechanism
  validation, not a production-scale measurement.
- Security rests on every concealment-reversing tool being registered as a decode
  tool, on the HMAC key staying within the MCP layer, and on the layer capturing the
  complete lineage of each call. The last is an implementation obligation:
  context taint is a conservative over-approximation, so influence that never enters
  the mediated path or any taint-tracked context is the residual (see the decode-set
  completeness and valid-label-substitution results above).

---

## Mapping to threat taxonomies

- **OWASP Top 10 for Agentic Applications (2026):** ASI01 Agent Goal Hijack,
  ASI02 Tool Misuse, ASI06 Memory & Context Poisoning, ASI07 Insecure Inter-Agent
  Communication (the defense is an ASI07 control).
- **OWASP Top 10 for LLM Applications:** LLM01 Prompt Injection, LLM02 Insecure
  Output Handling, LLM06 Sensitive Information Disclosure, LLM08 Excessive Agency.

---

## License

Released for educational and defensive security research. See `LICENSE`.
