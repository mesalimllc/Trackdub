# Design Spec — L1: CPU Text Refinement — Benchmark First, llama.cpp/GGUF Second

**Pitch status:** proposed — seeking product sign-off before any code.
**One-liner:** first measure the shipped Qwen2.5-1.5B refinement path on real CPU-only hardware; add a llama.cpp/GGUF tier only if that evidence shows a quality or speed gap that a larger quantized model would close.

---

## 1. Problem — restated against the shipped product

The original draft of this pitch claimed CPU users had no local text-refinement option. That claim is wrong, and this section corrects it:

- The bundled refinement model is **Qwen2.5-1.5B-Instruct** (`bundled-models.manifest.json`, `task: text-refinement`, `engine_family: qwen-instruct`, tier `balanced`, Apache-2.0, commercial-verified) — a 1.5B model, not the 7–9B assumed earlier.
- **CPU is an allowed provider** for the stage: `StageRuntimeRequirements` lists `DefaultOnnxStageAllowedProviders` for TextRefinement (includes `ExecutionProviderKind.Cpu`), with int8/quantized variants allowed, and the TRT-RTX family is deliberately *excluded* for `qwen-instruct` (GenAI + TRT-RTX terminates the process per the inline note).
- `RoutedTextRefinementEngine` selects the Qwen local engine by default when registered, regardless of hardware — there is no silent cloud fallback for CPU users.

So CPU users today get **local refinement with a 1.5B model**. The real, unproven questions are:

1. **Speed:** is the 1.5B ONNX int8 path fast enough on target low-end CPUs (no measurements exist in the repo for this hardware class)?
2. **Quality:** does 1.5B output hold up for transcript polishing/punctuation/hallucination-removal on hard audio, or does it need a bigger model?

This spec proposes answering those questions with data, then — only if the answers show a gap — adding llama.cpp/GGUF as the delivery vehicle for a larger model at CPU-usable speed.

## 2. Goals / Non-goals

**Goals**
- **Phase 0 (the ask):** measured CPU evidence for the shipped path — Qwen2.5-1.5B ONNX (int8/CPU provider) on reference low-end hardware: tokens/sec, time-to-first-token, and a semantic quality evaluation on a fixed refinement prompt suite.
- **Phase 1 (conditional on Phase 0 evidence):** if quality is the bottleneck and speed has headroom, a llama.cpp/GGUF tier for a larger refinement model, integrated through the **runtime planner**, not an ad-hoc router bypass.
- Real readiness semantics per the never-fake-readiness invariant.

**Non-goals**
- Translation and ASR. The earlier draft bundled them; per review they are separate scopes and ASR would add another runtime. If Phase 0 evidence motivates them, they get their own specs.
- Replacing ONNX on hardware where the shipped path measures fine.
- New quant-selection UI; planner defaults plus alias override.

## 3. Phase 0 — measure what ships (no new runtime)

**Benchmarks (DubBench):**
- Reference hardware: at least one true CPU-only low-end machine profile (no NPU/GPU), plus one mid CPU for context.
- Metric: real-time-factor-equivalent for refinement (tokens/sec against segment throughput needs), TTFT, and peak RSS — the latter matters because low-end machines are often also RAM-constrained.

**Semantic quality evaluation (the missing piece):**
- The existing `QwenRefinementOutputGuard` is a **safety filter** (structure/charset/guardrail checks) — it validates output is safe to consume, not that refinement is semantically good. It cannot measure quality regression or improvement.
- Phase 0 therefore needs a small human-reviewed evaluation suite: fixed prompt set (hallucinated ASR output, punctuation-degraded, speaker-turn mess), golden expected outputs, reviewed by a native speaker. The suite itself is a deliverable and is reused as the gate for any Phase 1 model.

**Decision matrix:**

| Phase 0 result | Action |
|---|---|
| 1.5B speed OK, quality OK | Close the spec — no llama.cpp; document CPU expectations |
| Speed OK, quality insufficient | Phase 1: larger model needed; evaluate GGUF tier (a larger ONNX int8 export is the alternative — compare both) |
| Speed insufficient even at 1.5B | Phase 1 is *not* justified by a bigger model; investigate smaller/faster options instead; possibly close |

## 4. Phase 1 (conditional) — llama.cpp/GGUF tier, planner-integrated

Only entered on the "quality gap, speed headroom" outcome.

**Architecture:** managed sidecar (`llama-server`, loopback-only, health-checked startup, deterministic shutdown), one binary per platform, GGUF entries in the existing bundled-models manifest (URL + size + SHA-256, license, `commercial_use_verified` per the manifest schema — GGUF quantizations are community-produced, so license verification must be done per artifact, not inherited from the base model).

**Routing — through the runtime planner, not around it.** The stage's `StageRuntimeRequirements` gains the llama.cpp engine family as an additional allowed family with its own provider set, and `RuntimePlanner` selects it per its existing rules (provider availability, tier preferences, alias requirements). No second hardware-probe path in `RoutedTextRefinementEngine`; the router keeps its current role (alias resolution over registered engines) and the planner owns tier/provider choice — same division of labor as TTS (`RoutedTtsEngine` → `StageRuntimePlanningRequest`).

**Readiness ladder:** sidecar binary present → GGUF downloaded/hash-verified → process healthy (`/health`) → stage ran → stage succeeded.

**Quality gate for the candidate GGUF model:** the Phase 0 human-reviewed suite, plus `QwenRefinementOutputGuard` as the safety filter it already is. A larger quantized model is adopted only on suite parity-or-better against 1.5B golden outputs.

## 5. Risks

| Risk | Mitigation |
|---|---|
| Phase 0 evidence shows no gap | Spec closes with documented CPU expectations — cheap outcome, no new runtime shipped |
| Community GGUF artifacts carry unclear licenses | Per-artifact license verification at manifest-entry time; unverified = excluded (repo policy: unknown license = unsafe) |
| Sidecar lifecycle bugs (orphans, port conflicts) | Loopback-only binding, health-checked startup, shutdown tied to stage lifetime |
| Planner-integrated routing delays the spike | Spike may hard-register the engine for benchmarking, but ship-gating requires planner wiring; no router-level shortcuts in the product |
| Larger model regresses safety-filter behavior in ways the guard catches | Guard remains in the path unchanged; suite adds the semantic layer the guard doesn't cover |

## 6. Open questions for review

1. Reference low-end hardware profile(s) — which machine class defines "target"?
2. If Phase 1 triggers: same Qwen family at a larger size (suite comparability) vs a stronger open model with better GGUF support?
3. Bundle `llama-server` vs on-demand download — packaging call, same as any runtime dependency.
4. Is a larger-model ONNX int8 export preferable to a GGUF sidecar if Phase 1 triggers? Decided by the Phase 0/1 numbers and the "no second runtime without evidence" bar.
