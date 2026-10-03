# Measured results

Everything here was produced by the commands in README.md on this machine (Apple Silicon,
macOS, `cargo --release`), October 2026. Numbers are from `classifier_report`; the raw
files are reproducible with the commands shown.

## Status at a glance

| backend | public sample (10) | own held-out (48) | verified? |
|---|---|---|---|
| regex (default) | 30.0% accuracy, ECE 0.200 | 27.1% accuracy, macro F1 0.258, ECE 0.138 | yes, measured here |
| jev-1.13.0 | **not measured** | **not measured** | **no** — no `TYPESAFE_API_KEY` in this environment |

**The hosted path is implemented and tested against a local HTTP mock of the documented
contract, but no live Jev call was made.** Every hosted figure (accuracy, ECE, latency,
cost per decision, repeatability, option-order sensitivity, model version) remains to be
measured by running the commands below with a key. Nothing in this document estimates
them.

## Regex baseline (measured)

Public sample, two consecutive runs (`PRED`/`PRED2`):

| metric | value |
|---|---|
| accuracy | 30.0% (3/10) |
| macro F1 | 0.262 |
| ECE (10 bins) | 0.200 |
| complexity exact / ±1 / MAE | 20% / 60% / 1.20 (fixed level 3, so this is what a constant scores) |
| latency p50 / p95 | 11 µs / 189 µs |
| init (excluded) | ~16 ms (regex table compile) |
| wall clock, whole example | 0.5–1.1 s |
| repeatability | 0 of 10 rows differ on request_type/complexity/confidence; `latency_us` differs as expected |

Own held-out split (48 cases):

| metric | value |
|---|---|
| accuracy | 27.1% |
| macro F1 | 0.258 |
| ECE (10 bins) | 0.138 |
| complexity exact / ±1 / MAE | 12.5% / 43.8% / 1.44 |
| latency p50 / p95 | 4 µs / 65 µs |

Dev split: 35.4% accuracy. The regex classifier predicts `general` for most held-out
cases (37/48 land in the 0.3 bin, i.e. nothing matched), which is the keyword-trap and
paraphrase behaviour the private set is said to target. Its confidence pair is
uncalibrated by construction: the 0.3 bin is 16% accurate, the 0.5 bin higher.

Fallback rate: 0 (no primary backend). Cost per decision: $0 (no network). Hardware
cost: negligible CPU.

## Jev (to be measured — exact commands)

```sh
export CLASSIFIER_BACKEND=jev TYPESAFE_API_KEY=…   # never commit the key
D=llm-router/tests/data/classifier
for run in 1 2; do
  EVAL_SET=/tmp/classifier-eval.json OUT=/tmp/jev-public-$run.jsonl \
  cargo run --release -p nasiko-llm-router --example classifier_eval
done
EVAL_SET=$D/heldout.json OUT=/tmp/jev-heldout.jsonl cargo run --release -p nasiko-llm-router --example classifier_eval
EVAL_SET=$D/calibration.json OUT=/tmp/jev-cal.jsonl cargo run --release -p nasiko-llm-router --example classifier_eval

EVAL_SET=/tmp/classifier-eval.json PRED=/tmp/jev-public-1.jsonl PRED2=/tmp/jev-public-2.jsonl \
  PRICE_PER_M_INPUT=0.042 cargo run --release -p nasiko-llm-router --example classifier_report
EVAL_SET=$D/heldout.json PRED=/tmp/jev-heldout.jsonl PRICE_PER_M_INPUT=0.042 \
  cargo run --release -p nasiko-llm-router --example classifier_report
cargo test -p nasiko-llm-router --test jev_live -- --ignored --nocapture   # repeatability + option order
```

What the report will contain once run: accuracy, per-class P/R/F1, confusion, macro F1,
ECE, complexity exact/±1/MAE (mode rule) with `complexity_expected` in the sidecar for the
rounding rule, p50/p95 latency end to end, fallback counts by cause, dispositions,
selective accuracy vs coverage, billed input tokens per decision and cost at the stated
price, model versions that answered, and semantic repeatability across the two runs.

Price basis for the cost line: typesafe.ai `/models` page, read October 2026, lists
jev-1.13.0 at $0.042 per million input tokens with output tokens free; pass it as
`PRICE_PER_M_INPUT` and cite the date next to the figure. Billed input includes the
instructions and criteria (~600–700 tokens per decision by construction; the sidecar
reports the actual count).

Choose `CLASSIFIER_MIN_CONFIDENCE` from the calibration split's selective-accuracy table,
then report held-out with that floor; do not pick it on held-out.

Local-model figures (size, load time, hardware) are **N/A** for this hosted-only
alternative, not zero.

## Runtime feasibility (15-minute CPU runner)

- Regex: the whole example runs in about one second for 10 cases and under two for 48;
  a 200-case private set is well under a minute including compilation of the example.
- Jev: per-decision latency is network-bound and unmeasured here. With the default 3 s
  deadline and sequential evaluation, the worst case for 200 cases is 200 × 3.25 s ≈ 11
  minutes **if every call times out** — inside the limit, and every such row is a counted
  regex fallback rather than a failure. Typical hosted latency is expected to be far lower,
  but that is an expectation, not a measurement.

## Harness ambiguity to flag

The brief says runs are diffed for determinism, but `latency_us` is required to be a real
measurement and will differ between runs. This contribution keeps real timing and provides
`classifier_report … PRED2=…` to diff the semantic fields (`request_type`, `complexity`,
`confidence`). Organizers should confirm whether their diff excludes `latency_us`.

## Downstream routing quality

Not measured. No claim of reduced spend or better answers is made; see DECISIONS.md §4.
