# [classifier] Pluggable request classifier with Jev hosted backend and regex fallback

**Track:** P2 — Request classifier for cost-aware routing.

## What this adds

- `RequestClassifier` trait (`routing/classifier.rs`), `RegexClassifier` wrapping the
  existing `classify_request_type` unchanged, and a shared `ClassifierService` that owns
  validation, timing, the overall deadline, counted regex fallback and a configurable
  low-confidence abstention. The router holds it once (`LlmRouterCtx.classifier`); the eval
  example and routing use the same path.
- A Jev (typesafe.ai) hosted backend (`routing/jev.rs`) using the documented HTTP API:
  one POST with the query and bounded context as `state`, a Choice over the seven request
  types and a Score over the five complexity levels. Full response validation, bounded
  input/response/concurrency, no redirects, credential redaction, opt-in bounded retry.
- Bounded, role-labelled classifier context from both wire formats (`routing/context.rs`,
  Responses adapter), excluding system prompts, tool results and non-text parts.
- Opt-in deterministic tier sampling (`CLASSIFIER_ROUTING_SEED`); legacy entropy RNG
  otherwise.
- `examples/classifier_eval.rs` extended to the trait path with the exact harness contract,
  plus a diagnostics sidecar; new `examples/classifier_report.rs` scorer.
- Labelled dev/calibration/held-out data with a split manifest and a leakage test;
  docs in `llm-router/docs/classifier/`.

Default behaviour is unchanged: `CLASSIFIER_BACKEND=regex` is the default, nothing is
contacted, downloaded or required, and existing routing tests pass unmodified.

## How to run

```sh
curl -fsSL https://registry.nasiko.dev/r/nasiko/classifier-eval -o /tmp/classifier-eval.json
EVAL_SET=/tmp/classifier-eval.json OUT=/tmp/classifier-out.jsonl \
cargo run --release -p nasiko-llm-router --example classifier_eval
```

Env vars (all optional): `CLASSIFIER_BACKEND=regex|jev` (default `regex`),
`CLASSIFIER_ENDPOINT` (default `https://api.typesafe.ai/v1/systemone`),
`CLASSIFIER_MODEL` (default `jev-1.13.0`), `TYPESAFE_API_KEY` (required for `jev`),
`CLASSIFIER_TIMEOUT_MS` (3000), `CLASSIFIER_MIN_CONFIDENCE` (0.0), `CLASSIFIER_ROUTING_SEED`,
`CLASSIFIER_MAX_CONCURRENCY` (8), `CLASSIFIER_RETRIES` (0). Diagnostics go to
`<OUT>.diagnostics.jsonl` (`DIAG_OUT`); `OUT` carries only the harness fields.

## Model IDs and hosted endpoint

- Model: `jev-1.13.0` (versioned; the response's `model` field is recorded per call).
- Endpoint to allow on the egress proxy: `https://api.typesafe.ai/v1/systemone` (POST).
  The key is sent only there; redirects are not followed.

## Measured results

Regex (measured here): public sample 30.0% accuracy, ECE 0.200; own held-out (48 cases)
27.1% accuracy, macro F1 0.258, ECE 0.138, p50 4 µs, p95 65 µs, zero fallbacks, $0 per
decision. Two runs produce identical semantic fields.

Jev: **not measured** — no API key was available in the development environment. The
adapter is tested against a local mock of the documented contract. The exact commands to
produce accuracy, ECE, latency, cost per decision, repeatability and option-order
sensitivity are in `docs/classifier/RESULTS.md`. No hosted number in this PR is estimated.

## Known limits and unsupported cases

- Live Jev accuracy, latency, cost and repeatability are unverified; Jev documents no
  seed/temperature, so repeatability is measured, not assumed.
- Only text is classified; multimodal parts contribute nothing.
- Regex complexity (3) and confidence (0.5/0.3) are uncalibrated placeholders.
- Complexity is predicted and reported but does not change tier selection; the bandit key
  is unchanged. No downstream routing-quality or cost-saving claim is made.
- Abstention is off by default (`CLASSIFIER_MIN_CONFIDENCE=0`) until a floor is chosen on
  the calibration split with live data.
- Labels in the own data are reviewed synthetic labels, not independently adjudicated.
- Harness note: `latency_us` is real timing and will differ between runs; the scorer diffs
  the semantic fields (`PRED2=`).

## Checks run

`cargo fmt --check`, `cargo clippy -p nasiko-llm-router --all-targets` (zero warnings),
`cargo test -p nasiko-llm-router` (all passing, live tests `#[ignore]`d),
`cargo check -p nasiko-server` (host call sites compile), the release evaluator twice on
the public sample and once per split, and `classifier_report` on each.

🤖 Generated with [Claude Code](https://claude.com/claude-code)
