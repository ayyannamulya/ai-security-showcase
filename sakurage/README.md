<div align="center">

![Type](https://img.shields.io/badge/Type-Security%20Research-red?style=flat-square)
![Status](https://img.shields.io/badge/Status-Active-brightgreen?style=flat-square)

# sakurakage 桜影

### *Behavioral backdoor auditor for HuggingFace LoRA adapters — weight-delta analysis + Claude-generated probing + ATLAS × STRIDE threat model output.*

![Python](https://shieldcn.dev/badge/Python%203.13-3776AB.svg?logo=python&logoColor=FFD43B&color=3776AB)
![PyTorch](https://shieldcn.dev/badge/PyTorch-EE4C2C.svg?logo=pytorch&logoColor=white&color=EE4C2C)
![HuggingFace](https://shieldcn.dev/badge/Transformers-FFD21E.svg?logo=huggingface&logoColor=black&color=FFD21E)
![LangGraph](https://shieldcn.dev/badge/LangGraph-1C3C3C.svg?logo=langchain&logoColor=white&color=1C3C3C)
![Pydantic](https://shieldcn.dev/badge/Pydantic-E92063.svg?logo=pydantic&logoColor=white&color=E92063)
![Jinja2](https://shieldcn.dev/badge/Jinja2-B41717.svg?logo=jinja&logoColor=white&color=B41717)
![pytest](https://shieldcn.dev/badge/pytest-0A9EDC.svg?logo=pytest&logoColor=white&color=009FE3)
![uv](https://shieldcn.dev/badge/uv-DE5FE9.svg?logo=astral&logoColor=white&color=261230)

</div>

---

Existing tools (ModelScan, HF malware scanner) check serialization safety. sakurakage goes further: it audits the *behavioral impact* of the adapter delta — what the adapter actually *does* differently from the base model — and maps findings to the MITRE ATLAS threat taxonomy.

---

## Threat model

**Assets:** HuggingFace LoRA adapters loaded into production inference pipelines

**Threat actors:** Malicious adapter publishers, compromised upstream repositories, supply chain adversaries

**Primary attack surface:** AML.T0018 (Backdoor ML Model), AML.T0010 (ML Supply Chain Compromise) — an adapter can be fine-tuned to behave normally on benign inputs while activating on a known trigger phrase, with no serialization-level indicator

**Attack vectors:**
- Trigger-phrase behavioral backdoor embedded in adapter weight delta
- Model card content used as indirect prompt injection surface against downstream tooling
- Sharded adapters where only non-first shards carry the anomalous signal
- Probe-evasion: adapter tuned to recognize audit-like phrasing and suppress backdoor behavior during evaluation

**Impact:** Silent behavioral compromise of any model served with the adapter — output manipulation, policy bypass, jailbreak amplification

**Mitigations implemented:**
- Weight-level delta analysis (L2 norm outliers per layer) as a signal independent of behavioral probing
- Subprocess isolation for all weight loading and inference — tensors never cross IPC boundary
- Probe prompts are Claude-generated from structured context, never derived from model card content
- Hard rejection of `.bin`, `.pt`, `.pkl`, `.ckpt` at load boundary regardless of context; `.safetensors` additionally magic-byte validated
- All Claude API suspicion scores clamped to `[0.0, 1.0]` before entering the threat model

---

## ATLAS TTP coverage

| TTP | Name |
|---|---|
| AML.T0010 | ML Supply Chain Compromise |
| AML.T0013 | Discover ML Artifacts |
| AML.T0018 | Backdoor ML Model |
| AML.T0020 | Poison Training Data |
| AML.T0043 | Craft Adversarial Data |
| AML.T0048 | External Harms |
| AML.T0051 | LLM Prompt Injection |
| AML.T0054 | LLM Jailbreak |

---

## Configuration

| Variable | Default | Description |
|---|---|---|
| `ANTHROPIC_API_KEY` | required | Anthropic API key |
| `HF_TOKEN` | optional | HuggingFace token for gated repos |
| `ANTHROPIC_MODEL` | `claude-sonnet-4-20250514` | Model for probe generation and scoring |
| `PROBES_PER_CATEGORY` | `3` | Probes generated per category |
| `SAKURAKAGE_OUTPUT_DIR` | unset | If set, write `.md` and `.json` report here |

---

## Trust boundaries

| Boundary | Rule |
|---|---|
| HF Hub API responses | Untrusted — parsed into typed Pydantic models immediately |
| Adapter weight files | Untrusted — magic-byte validated, loaded in isolated subprocess |
| Base model weight files | Untrusted — same enforcement as adapter |
| Probe prompts | Claude-generated only — never user-controlled |
| Adapter inference output | Untrusted — opaque strings, never executed |
| Claude API responses | Validated — JSON schema + score clamping |
| AuditState | Immutable — `model_copy()` only, no direct mutation |

---

## Test suite

```bash
uv run pytest                                    # all 159 tests
uv run pytest tests/test_hub_crawler.py         
uv run pytest tests/test_model_loader.py         
uv run pytest tests/test_delta_analyzer.py       
uv run pytest tests/test_probe_engine.py         
uv run pytest tests/test_threat_model.py         
uv run pytest tests/test_pipeline_e2e.py         
```

---

## Lab

`lab/` contains a controlled test fixture — a deliberately backdoored LoRA adapter built on `sshleifer/tiny-gpt2` with a known trigger phrase. See `lab/README.md` for fixture construction details and ethics boundary.

---

## Known limitations

- Delta computation uses the first safetensors shard only on sharded models — anomalous signal in non-first shards will be missed
- Probe inference is CPU-only; large base models will be slow
- Probe-evasion is a real attack class: an adapter tuned to recognize probe-like phrasing will suppress backdoor behavior during evaluation; delta analysis provides an independent signal for this case but is not a complete countermeasure

---

## License

MIT