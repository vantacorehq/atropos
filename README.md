<p align="center">
  <img src="banner.svg" alt="ATROPOS" width="100%">
</p>

<p align="center">
  <b>Autonomous agent for social chaos &amp; arbitrage.</b><br>
  One file. Zero dependencies. It watches the panic. It cuts the noise.
</p>

<p align="center">
  <img alt="python" src="https://img.shields.io/badge/python-3.9%2B-5b8dc9?style=flat-square&labelColor=05060a">
  <img alt="deps" src="https://img.shields.io/badge/dependencies-0-ff1744?style=flat-square&labelColor=05060a">
  <img alt="files" src="https://img.shields.io/badge/core-1%20file-9aa3b5?style=flat-square&labelColor=05060a">
  <img alt="eval" src="https://img.shields.io/badge/eval%2Fexec-never-ff1744?style=flat-square&labelColor=05060a">
</p>

---

## What it does

Social feeds are adversarial. During a panic or a hype cycle, the loudest signal is often the cheapest to fake: copy-pasted shill posts, week-old sock accounts, repost-farmed engagement. A naive sentiment average then tells you what the *manipulators* want you to see.

**Atropos prunes first and measures second.** Every post gets a credibility interval. A set of declarative *forbid* rules narrows those intervals, and the narrowing propagates across posts (duplicate clusters, author history, cluster taint) until nothing changes. Only the survivors vote. The gap between what the crowd *appears* to say and what the survivors say is the signal.

```
$ORBT   BEAR   conv 0.69   clean -0.31   raw +0.14   Δ +0.45   noise 37%   MANIPULATED
```

*The raw feed says mildly bullish. After pruning, the credible voices are clearly bearish.*

## Quickstart

```bash
git clone https://github.com/vantacorehq/atropos && cd atropos
python atropos.py --demo --explain          # simulated feed, pruning reasons at exit
python atropos.py --selftest                # security + invariant checks
python atropos.py --bench 20000             # reproduce the benchmark below
python atropos.py --demo --json | head -3   # NDJSON for pipes
```

Live polling (the token never touches argv or logs in clear text):

```bash
export ATROPOS_SECRET_X_TOKEN="…"
python atropos.py --url "https://api.example.com/feed" \
  --header "Authorization: Bearer {{secret:X_TOKEN}}" \
  --allow-host api.example.com --interval 5
```

The endpoint may return a JSON array, NDJSON, or a v2-style `{"data": [...], "includes": {"users": [...]}}` envelope.

### As a library

```python
import asyncio, atropos

engine = atropos.Atropos(window_seconds=300)
report, signals = engine.process(batch_of_dicts)        # sync
for s in signals:
    print(s.topic, s.direction, s.clean_net, s.manipulated)

asyncio.run(engine.run(atropos.MockXStream(batches=12), lambda i, rep, sigs, ms: print(i, sigs[0])))
```

## Pipeline

```
                ┌─────────────────────────────────────────────────────────────┐
  Source        │ asyncio.Queue (bounded)  — backpressure, CPU off the loop   │
  (stream/poll) └──────────────┬──────────────────────────────────────────────┘
                               ▼
              ┌──────────────────────────────────┐
              │ 1. MASK  PII / secrets → tokens  │  Redactor · Vault · SecureTransport
              │    author → salted pseudonym     │
              └──────────────┬───────────────────┘
                             ▼
              ┌──────────────────────────────────┐
              │ 2. FEATURES                      │  lexicon sentiment, shill score,
              │    sentiment · shill · MinHash   │  engagement, account age, MinHash sig
              └──────────────┬───────────────────┘
                             ▼
              ┌──────────────────────────────────┐   rules = strings, parsed by `ast`,
              │ 3. CONSTRAINT / FORBID PROTOCOL  │   compiled to closures (no eval/exec)
              │    [lo,hi] per post → fixpoint   │◄─┐
              └──────────────┬───────────────────┘  │ context recomputed from
                             │ survivors            │ survivors each round
                             └──────────────────────┘
                             ▼
              ┌──────────────────────────────────┐
              │ 4. SIGNAL                        │  clean_net · raw_net · divergence
              │    per topic ($TICKER / #tag)    │  momentum · noise · conviction
              └──────────────────────────────────┘
```

### Constraint / Forbid protocol

Each post owns an interval `[lo, hi] ⊆ [0, 1]` (starting at `[0, 1]`). A rule is `when` (a boolean expression) plus an `effect`:

| effect    | domain update       |
|-----------|---------------------|
| `forbid`  | `[lo, hi] ← [0, 0]` |
| `cap:x`   | `hi ← min(hi, x)`   |
| `floor:x` | `lo ← max(lo, x)`   |

Rules only ever **narrow** domains, so propagation is monotone and terminates (hard-bounded by `max_rounds`). Cross-post context is recomputed from survivors every round, which is what makes pruning cascade:

| context variable    | meaning                                                  |
|---------------------|----------------------------------------------------------|
| `cluster_size`      | near-duplicate cluster size (MinHash/LSH, Jaccard ≥ 0.6) |
| `dup_cluster`       | *surviving* members of that cluster                      |
| `dup_rank`          | survivor's order within the cluster (0 = first poster)   |
| `cluster_dead_frac` | fraction of the cluster already forbidden                |
| `author_posts`      | author's surviving posts in the window                   |
| `author_dead`       | author's forbidden posts in the window                   |

Example cascade: `fresh_sock` and `shill_lexicon` kill most of a copypasta cluster in round 1 → `cluster_dead_frac` crosses 0.5 → `tainted_cluster` forbids the aged-account stragglers in round 2 → `author_dead` rises → `tainted_author` fires in round 3.

The final credibility of a survivor is `clip(prior, lo, hi)`, where `prior` is itself a user-editable expression (default: follower reach × account age × shill discount).

### Restrictive evaluation engine

Rules come from config files, so they are untrusted input. `SafeExpr` never calls `eval`, `exec` or `compile`:

- `ast.parse(mode="eval")`, then a recursive **whitelist compiler** that turns the tree into closures.
- Allowed: numbers, booleans, declared variable names, `+ - * / // % **`, comparisons, `and/or/not`, ternary, and `min max abs log sqrt exp tanh clip sign`.
- Rejected at **parse time**: attribute access, subscripts, strings, lambdas, comprehensions, walrus, f-strings, keyword/starred args, unknown names.
- Runtime guards: division by zero is neutral (`0.0`), `**` is bounded (|exp| ≤ 8, |base| ≤ 1e6), `exp` is clamped, expressions are limited to 400 chars / 120 nodes / depth 16.

`python atropos.py --selftest` fires 16 adversarial payloads (`__import__('os')`, `().__class__.__bases__`, `open(...)`, …) at it.

### PII / secret masking layer

- **`Redactor`**: masks registered secrets (exact match) and generic shapes (emails, phones, cards, SSNs, IPv4, JWTs, `sk-…`, `AKIA…`, `ghp_…`, `Bearer …`, `?token=…` query params).
- **`Vault`**: tokens live in memory only (loaded from `ATROPOS_SECRET_<NAME>` env vars). Code and CLI args reference `{{secret:NAME}}` placeholders; values are injected at the last moment inside the transport and never appear in `repr`.
- **`SecureTransport`**: stdlib `urllib` in a worker thread, host allowlist, scheme check, response-size cap, retries. Errors are re-raised masked with `from None`, so no unmasked cause leaks through tracebacks.
- **`MaskingFilter`**: a `logging.Filter` that masks every record, including exception text.
- Post text is masked *before* feature extraction, and author handles are replaced by a per-process salted `blake2b` pseudonym. Raw handles are never stored.

## Custom rules

Copy `rules.example.json`, edit it, and run `python atropos.py --demo --rules rules.json`.

```json
{
  "window_seconds": 600,
  "prior": "min(1, log(1 + followers) / 10) * (1 - 0.6 * shill_score)",
  "conviction": "tanh(abs(clean_net) * sqrt(mass) / 3) * (1 - 0.5 * noise_ratio)",
  "rules": [
    {"name": "fresh_sock", "when": "account_age_days < 3 and followers < 25", "effect": "forbid"},
    {"name": "one_voice",  "when": "author_posts >= 4",                      "effect": "cap:0.2"}
  ]
}
```

## Signal semantics

| field         | definition                                                                                     |
|---------------|------------------------------------------------------------------------------------------------|
| `raw_net`     | unweighted mean sentiment of *everything* mentioning the topic (what a naive scraper sees)     |
| `clean_net`   | credibility-weighted mean sentiment of survivors                                               |
| `divergence`  | `raw_net − clean_net`, i.e. manipulation pressure                                              |
| `momentum`    | `clean_net − EMA(previous clean_net)`, where the shift is happening                            |
| `noise_ratio` | share of the topic's posts forbidden                                                           |
| `conviction`  | user-editable expression over the above; default shrinks with noise, grows with surviving mass |
| `manipulated` | flagged when `noise_ratio ≥ 0.30` (plus extra conditions, see `atropos.py`)                    |
| `direction`   | `BULL` / `BEAR` when `clean_net` is far enough from zero (threshold in `atropos.py`), else neutral |

## Benchmarks

> **Read this first.** These numbers come from the built-in *simulator* (`MockXStream`): 3 tickers, a bot swarm at 38% of traffic with shill templates, 25% aged accounts and inflated reposts, plus organic users with varied text. The adversary is **not adaptive** and the simulator was written by the same author as the rules, so treat precision/recall as a regression baseline, not as evidence of real-world accuracy. Validate on labelled real data before relying on it.

Measured with `python atropos.py --bench 20000` (seed 7, 100 batches × 200 posts, 300 s window ≈ 2,000 posts, Python 3.12, single shared vCPU sandbox):

| metric                                                         | result                          |
|----------------------------------------------------------------|---------------------------------|
| Bot precision / recall (hard-forbid)                           | **0.989 / 1.000**               |
| Organic false-positive rate                                    | **0.7%**                        |
| Bot share of posts → share of credibility mass                 | **38% → 0.0%**                  |
| Topic-sentiment error vs. organic truth (MAE)                  | raw **0.257** → clean **0.009** |
| Throughput (end to end, full window re-propagated every batch) | **~1,500 posts/s**              |
| Batch latency (200 new posts, ~2,000-post window)              | p50 **127 ms**, p99 **219 ms**  |

Seed 11 reproduces within noise (precision 0.989, recall 1.000, MAE 0.257 → 0.010).

**Ablation: does propagation matter?** With `shill_lexicon` and `fresh_sock` removed (a harder adversary with no obvious tells), only the structural rules remain:

|                          | recall    | precision | organic FPR |
|--------------------------|-----------|-----------|-------------|
| 1 round (no propagation) | 0.950     | 0.996     | 0.2%        |
| to fixpoint              | **1.000** | 0.989     | 0.7%        |

Propagation buys +5 points of recall at the cost of ~0.5 points of organic false positives. That is the honest trade-off of taint rules, and you can tune it by editing `tainted_cluster` / `tainted_author`.

**Complexity.** Per batch: O(W) MinHash/LSH clustering over the window of size W, O(R·W) rule evaluation, where R ≤ `max_rounds` (typically 2–3). Static rules are evaluated once per post lifetime and cached; only context rules re-run each round.

## Known limitations

- Sentiment is a small lexicon model (negation + intensifiers + emoji). It is a swappable function, not a claim of SOTA NLP. Sarcasm and non-English text are out of scope.
- Near-duplicate clustering compares each LSH bucket to its first member, which is an approximation that favours speed.
- The window is re-propagated each batch (simple and exact, but O(W)). An incremental variant is on the roadmap.
- Rules are heuristics. An adaptive adversary that mimics organic behaviour will evade them.
- The simulator speaks an X-shaped schema; you are responsible for complying with any real platform's terms of service and API rules.

## Repository layout

```
atropos/
├── atropos.py            # the entire engine (stdlib only)
├── banner.svg
├── README.md
├── LICENSE.txt
├── rules.example.json
└── .gitignore
```

## Roadmap

- Incremental propagation (evict/insert without full recompute)
- Pluggable sentiment scorer interface
- Author-graph constraints (co-posting, reply rings)
- Replay harness for labelled real datasets
- Rule-pack format with signed, versioned bundles

## Disclaimer

Atropos is research tooling for studying information quality in noisy feeds. Its output is not financial, investment or legal advice, and it places no orders. Use it responsibly and in line with the terms of any data source you connect.

## License

MIT, see [LICENSE.txt](LICENSE.txt).
