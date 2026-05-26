# injectbench

Prompt-injection evaluation that reports attack success rate (ASR) per attack class with confidence intervals, and an ablation that shows which guardrail layers earn their latency cost.

[![CI and Pages](https://github.com/saikirankoppichetti/injectbench/actions/workflows/ci.yml/badge.svg?branch=main)](https://github.com/saikirankoppichetti/injectbench/actions/workflows/ci.yml)
[Live demo](https://saikirankoppichetti.github.io/injectbench/)

![injectbench desktop view](docs/assets/deployed-desktop.png)

## Why

An assistant that reads emails, documents or web pages will eventually read text written by an attacker. Teams typically ship a system prompt that says "ignore malicious instructions" and never measure whether it does anything. A single "blocked 89%" number is worse than nothing: it averages a class the defences stop almost completely with a class they barely touch. injectbench keys every measurement by attack class first, attaches an interval that reflects the trial count, and measures each defence by removing it.

## Features

- Five-class attack taxonomy: direct override, indirect retrieval, roleplay framing, encoding obfuscation, tool-output poisoning.
- Seeded corpus generator producing generic, non-operational, taxonomy-tagged trials.
- Per-class and aggregate ASR with Wilson 95% intervals (Wald is included only as a comparison).
- Ablation: ASR of the full stack versus the stack minus each layer, as a paired delta with its own interval, plus latency per point of ASR bought.
- Redundancy detection: flags a layer whose removal flips no trials, and names the layer that masks it.
- Minimum detectable difference per class, so "no measured effect" is read against what the trial count could resolve.
- Positive (undefended) and negative (hard refusal) controls to show the harness has power.
- `regression_check` that flags only classes whose intervals separate between two runs, usable as a CI gate.
- Standard library only; no install step.

## Architecture

```mermaid
flowchart LR
  T["taxonomy<br/>classes, defences, cues"] --> C["corpus generator<br/>(seeded)"]
  C --> H["harness<br/>run / ablate / evaluate"]
  X["Target protocol<br/>attempt(attack, defences)"] --> H
  S["SimulatedTarget<br/>+ controls"] -.implements.-> X
  H --> ST["stats<br/>Wilson, MDD, paired delta"]
  ST --> R["Report"]
  R --> D["demo report"]
  R --> G["regression_check"]
```

`taxonomy.py` defines the classes, the five defence layers, their per-request latency and the cues each defence keys off. `corpus.py` draws every trial once from a fixed seed, so the same attack objects are replayed in every ablation arm; that makes the ablation a paired design. The harness only talks to a `Target`, so a real system can replace the simulator without touching any statistics. `stats.py` supplies the intervals and the detectability math.

## Quick start

Requires Python 3.11 or newer. No dependencies.

```bash
git clone https://github.com/saikirankoppichetti/injectbench.git
cd injectbench
python -m unittest discover -s tests -v   # 44 tests, about 2 s
python -m src.demo                        # prints the full report
```

The browser demo in `site/` is a simplified model of the same idea (toggle guardrails, see ASR with an interval and added latency). It is checked in CI with `python scripts/validate_site.py` and `node --test site/core.test.mjs`.

## Example output

From `python -m src.demo` (seeded, 400 trials per class, 2000 total):

```text
category                     n  undef ASR ASR with full stack           MDD
roleplay_framing           400      80.8%   46.5% [ 41.7%,  51.4%]     9.9%
direct_override            400      94.5%    3.5% [  2.1%,   5.8%]     3.6%
encoding_obfuscation       400      85.2%    2.2% [  1.2%,   4.2%]     2.9%
indirect_retrieval         400      86.8%    1.0% [  0.4%,   2.5%]     2.0%
tool_output_poisoning      400      90.8%    0.8% [  0.3%,   2.2%]     1.7%
AGGREGATE                 2000      87.6%   10.8% [  9.5%,  12.2%]

remove                  ASR full   ASR w/o delta (Wilson 95%)         ms  ms/point
input_filter               10.8%     19.6%   8.8% [ 7.6%, 10.1%]      4       0.5
normalizer                 10.8%     16.0%   5.2% [ 4.3%,  6.3%]      6       1.2
output_scanner             10.8%     15.5%   4.7% [ 3.9%,  5.7%]     35       7.4
delimiter_isolation        10.8%     13.4%   2.6% [ 2.0%,  3.4%]      2       0.8
provenance_tagger          10.8%     10.8%   0.0% [ 0.0%,  0.2%]     11        --
```

| Measure | Value |
|---|---|
| Naive "blocked" rate | 89.2% |
| Aggregate ASR, full stack | 10.8% [9.5%, 12.2%] |
| Worst class (`roleplay_framing`) | 46.5% [41.7%, 51.4%], 4.3x the aggregate |
| Redundant layer (`provenance_tagger`) | 0 of 2000 trials flipped, upper bound 0.19%, costs 11 ms |
| Undefended control / hard-refusal baseline | 100.0% / 0.0% |

## How it works

- **Wilson, not Wald.** With 0 successes in 20 trials Wald reports a certain zero; Wilson reports an upper bound near 16%. Small samples support "not worse than X", and the intervals say so.
- **Paired ablation.** Each trial's cues are drawn once, independent of which defences are on, and base susceptibility is drawn from a separate stream. Removing a defence can only turn a block into a success, so the delta is the share of trials in one discordant cell, with a Wilson interval. If a trial ever gets safer after removing a layer, `paired_monotone_delta` raises instead of reporting a number built on a broken assumption.
- **Cue-based defences.** Each layer keys off one observable cue. Layers that key off the same cue are correlated by construction, which is exactly what the pairwise ablation has to expose. Here `provenance_tagger` only fires on a subset of what `delimiter_isolation` catches: removing either alone is small, removing both gives +7.1%.
- **Detectability.** MDD uses the two-proportion normal approximation at alpha 0.05 and power 0.80. Effects below it were never measurable at that trial count.
- **Regression gate.** `regression_check` compares per-class intervals and flags only non-overlapping increases, so a class that doubles while the aggregate stays flat is caught and sampling noise is not.

See [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) for the design decisions, failure modes and scope.

## Testing

```bash
python -m unittest discover -s tests -v
```

The 44 tests assert behaviour against the planted ground truth: Wilson versus Wald on 0/20, recovered per-class ASR inside the intervals, the planted redundant layer being flagged (and a non-redundant one not), controls landing at about 100% and 0%, determinism of the corpus and target, and `regression_check` failing on a single-class rise that leaves the aggregate flat.

## Project layout

```text
src/
  taxonomy.py   classes, defences, cues, latency, simulator parameters
  corpus.py     seeded, taxonomy-tagged attack generator
  target.py     Target protocol, SimulatedTarget, controls
  harness.py    run, ablate, evaluate, redundancy and regression checks
  stats.py      Wilson, Wald, MDD, paired delta
  demo.py       text report
tests/          unittest suite
site/           browser demo and its tests
scripts/        site validation
docs/           architecture notes and screenshots
```

## Limitations

- The system under test is a deterministic simulator with planted susceptibilities. The numbers validate the harness, not any real model or defence.
- Attack strings are one generic label per class; real ASR depends on a real corpus and a real model.
- No multi-turn priming class; the simulator has no conversation state.
- Latency costs are planted constants, not timings.
- Output is a text report, not HTML or a CLI.
- To test a real system, implement `Target.attempt(attack, enabled_defences) -> bool` (for example by checking whether a canary tool call fired) and pass it to `harness.evaluate`. Use only on systems you own and are authorised to test.

## License

MIT, see [LICENSE](LICENSE).

## Maintainer

Maintained by [Sai Kiran Koppichetti](https://github.com/saikirankoppichetti), AI/ML Engineer working across applied LLMs, retrieval systems, and evaluation. For questions or project feedback, connect on [LinkedIn](https://www.linkedin.com/in/saikirankoppichetti97/) or email [koppichettisaikiran97@gmail.com](mailto:koppichettisaikiran97@gmail.com).