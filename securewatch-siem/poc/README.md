# Proof of Concept — SecureWatch Core Modules

This directory contains standalone, dependency-light implementations
of the three most interesting core modules in SecureWatch:

- **`normalizer.py`** — multi-format log detection and ECS normalization
- **`rules.py`** — stateless detection rule examples
- **`scorer.py`** — risk scoring algorithm

These are simplified for readability and portfolio demonstration.
They are not the production implementations and contain no real
credentials, infrastructure references, or proprietary business logic.

---

## Running the PoC

```bash
# No dependencies required — pure Python 3.11+
python normalizer.py
python rules.py
python scorer.py
```

---

## What each file demonstrates

### `normalizer.py`
Shows how SecureWatch accepts logs in any format and produces a
consistent output schema. Run it to see syslog, CEF, JSON, and
key-value formats all normalizing to the same structure.

### `rules.py`
Shows the logic behind three detection rules without Redis or any
external state. Each rule is a pure function — easy to read,
test, and reason about.

### `scorer.py`
Shows the risk scoring algorithm. Takes a base score and a context
dict, applies weighted modifiers, clamps to 0–100, and maps to
a severity label.