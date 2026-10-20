# Decode-Time Provenance Binding for MCP Agents

A reproducible proof-of-concept for a steganographic prompt-injection vector that
defeats text content scanners, and a defense that contains it at the decode
boundary — *decode-time provenance binding*.

> **Educational / defensive security research only.** All secrets in this repo are
> fake. Use only on systems you own or are authorized to test.

---

## The idea in one paragraph

An attacker emails a victim an ordinary-looking message. A brand logo in the
**email signature** hides an instruction via steganography (LSB, DCT, or metadata).
The image is visually identical to the original and contains no suspicious text, so
a content scanner sees nothing and the victim never clicks anything. But the
victim's routine **inbox-summarization agent** processes the signature, a tool in
its pipeline **decodes** the image, and the hidden instruction enters the agent's
context — where, in a multi-agent MCP pipeline, it is relayed between agents as
trusted peer output and drives a critical tool call. This is **cross-modal trust
laundering**. The defense, **decode-time provenance binding**, mints an
HMAC-signed `UNTRUSTED` label at the instant a concealment-reversing tool runs,
binds it so no downstream agent can forge or strip it, and blocks any critical tool
call whose lineage is untrusted. It *contains* rather than *detects*, so it is
independent of payload wording and carrier.

---

## Results (all reproduced by the notebook)

**Attack success rate (ASR)** — six payloads across three carriers (LSB, DCT,
metadata), including benign-worded, Base64, token-split, and homoglyph evasions;
lower is better:

| Defense                              | Leaks | n | ASR |
|--------------------------------------|:-----:|:-:|:---:|
| Undefended (no checks)               | 6     | 6 | 100% |
| Content scanner (keyword/DLP)        | 4     | 6 | 67% |
| Regex intent parser                  | 3     | 6 | 50% |
| **Decode-time provenance (ours)**    | **0** | 6 | **0%** |

**Decode-set completeness** — the one assumption to stress:

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

**Overhead** — the mechanism is effectively free at the decision point: minting
and verifying a binding are ~8–9 µs each (~17 µs combined), versus a ~11 ms decode
tool call — under ~0.2% added. Carriers are visually imperceptible (LSB ≈ 76 dB
PSNR, metadata lossless).

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

Two deterministic operations at the MCP layer:

- **MintBinding** (inside each decode tool): on return, stamp
  `{payload, tool, trust, provenance}` with `trust = UNTRUSTED` iff the tool
  reverses a concealment transform, and sign it:
  `HMAC_k(payload‖tool‖trust‖provenance)`.
- **VerifyAndEnforce** (at the critical-tool boundary): recompute the HMAC; if it
  fails (forged/stripped) → block; if `trust = UNTRUSTED` → block; else allow.

**Security invariant.** Under (i) key confinement to the MCP layer, (ii) complete
registration of decode tools, and (iii) boundary mediation of every critical call:
*no critical tool call executes whose argument lineage includes an `UNTRUSTED`
decode origin, absent an out-of-band trusted promotion.*

---

## Scope and limitations

- The defense guards **actions**, not summary text: a laundered instruction can
  still skew the agent's prose; it cannot reach a critical tool.
- This is a controlled testbed (six payloads, one logo, one small model, a keyword
  scanner stand-in) — a mechanism validation, not a production-scale measurement.
- Security rests on every concealment-reversing tool being registered as a decode
  tool, and on the HMAC key staying within the MCP layer (see the decode-set
  completeness results above).

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
