# GEO Content Skill Optimization Benchmark — Design Plan

> **Status**: Living design document (initial version)
> **Goal**: Define a high-quality, maintainable SkillOpt benchmark for optimizing *skills that generate content optimized for Generative Engines* (GEO).
> **Audience**: Team building the benchmark + anyone who will maintain or extend the GEO content skill optimization loop.

## 1. Motivation and Objective

The objective is to treat a natural-language **skill document** (strategy, principles, heuristics, and generation workflow) as the optimizable artifact. When this skill is provided to a target model or agent, the generated content (articles, comparisons, guides, explainers, etc.) should demonstrably perform better in generative engines — higher citation rates, better surfacing for relevant buyer queries, stronger alignment with how AI systems synthesize and attribute answers.

Existing SkillOpt benchmarks optimize for *solving* tasks (QA, code manipulation, embodied goals, math). A GEO content benchmark is different in nature:

- The output is long-form, creative, and open-ended.
- "Success" is external visibility and citability rather than matching a gold label or passing unit tests.
- Evaluation is inherently multi-dimensional and partially subjective.
- High per-item cost (generation + evaluation).

This document outlines how to structure the benchmark so that the SkillOpt loop (rollout → reflect → aggregate → select → gated update) can still drive meaningful, stable improvements to the skill.

## 2. Core Design Principles

1. **Quality over quantity of tasks**. 15–40 excellent, diverse items beat hundreds of mediocre ones (see the small train sets that already work for ALFWorld, LiveMathematicianBench, etc.).
2. **Rich traces beat scalar scores**. The optimizer (analyst) learns best when it can see the *actual generated content* + *why* it was strong or weak.
3. **Hybrid evaluation**: Combine simulation-based signals (from The Prompting Company GEO tooling) with structured LLM judgment.
4. **Make the "gradient" GEO-native**. Custom analyst prompts should think like a senior GEO strategist critiquing content against the current skill.
5. **Start tiny, harden the signal**. Use `limit`, small selection sets + `soft`/`mixed` gates early. Evolve the evaluator as a first-class artifact.
6. **Avoid single-point reward hacking**. Ensemble judges + simulation grounding + periodic human review of the selection set.

## 3. Benchmark Components

### 3.1 Task Items (the "dataset")

A task item is a **content creation brief** that represents a real piece of work a GEO-optimized writer would be asked to do.

**Recommended schema** (evolve in `your_geo_benchmark/dataloader.py` `_normalize`):

```json
{
  "id": "string (stable, e.g. 'pain-2026-06-comp-003')",
  "content_type": "comparison | ultimate_guide | pain_point_explainer | objection_handler | feature_deep_dive | faq_cluster | ...",
  "primary_topic": "string",
  "target_audience": "string (persona or segment)",
  "buyer_journey_stage": "awareness | consideration | decision | post_purchase",
  "target_queries": [
    "exact or paraphrased queries we want the content to be cited for"
  ],
  "core_pains": [
    "frustrations phrased in neutral, third-party language (sourced from Reddit/HN/etc.)"
  ],
  "must_incorporate_facts": [
    "grounded claims or data points the content must accurately cover"
  ],
  "constraints": {
    "tone": "expert but approachable, zero hype",
    "length_words": "1400-1800",
    "must_have_sections": ["..."],
    "forbidden": ["brand-first language", "how-to for competitors"]
  },
  "evaluation_focus": "What makes *this* piece of content succeed or fail for GEO (free text for humans + judges)",
  "source_references": ["urls or doc ids used during curation"]
}
```

**Curation strategy**:
- Leverage existing Prompting Company GEO simulation prompt generation workflows (pain prompt banks, buyer journey mapping, neutral source research).
- Prioritize diversity across: content type, journey stage, product/category, difficulty (some "easy win" topics, some contested/complex ones).
- Every item should have clear "what good looks like" notes so the evaluator stays grounded.
- Start with 12–25 items. You can grow later using `split_mode: ratio` or by adding more manifests.

**Splits** (train / val=selection / test):
- Train: the items the optimizer sees trajectories from.
- Selection (val): the gate set. Make these especially high-signal and stable. 8–15 items is often enough early on.
- Test: final reporting. Can be larger.

Use the `configs/features/soft_gate.yaml` pattern when your selection set is small.

### 3.2 Rollout (how the skill turns a brief into content)

The skill document acts as the **strategic guidelines + workflow** for a content generation agent.

In `rollout.py` you will typically:

1. Inject `skill_content` (plus the task brief) into the target model/agent.
2. Support staged generation if it reflects real workflows (e.g., research pass → outline → draft → GEO optimization pass → final polish). Intermediate artifacts are gold for reflection.
3. Persist rich outputs under `predictions/<id>/`:
   - `generated_content.md` (the final article)
   - `full_conversation.json` or step logs
   - `target_system_prompt.txt` (critical — shows the skill in context)
   - `judge_report.json` (full rubric + rationales)
   - `simulation_results.json` (if you call TPC GEO simulators)
   - Any previews or traces

The result dict returned to the trainer must include at minimum:
- `"id"`
- `"hard"` (0 or 1, or thresholded)
- `"soft"` (0.0–1.0)
- Good-to-have: `"task_description"`, `"fail_reason"`, `"generated_preview"` or full content reference, plus any fields your custom analyst prompts will want to read.

You can use `chat_target` (for pure chat backends) or exec-style harnesses if your generation workflow benefits from tools.

### 3.3 Evaluation — Multi-Dimensional Rubrics + Simulation (the hard part)

This is where abstract quality becomes a training signal.

#### Philosophy
- Never rely on a single "overall 1-10" score.
- Combine **objective/simulation signals** with **structured qualitative judgment**.
- The scalar (`hard`/`soft`) is for the *gate*. The rich breakdown is for the *optimizer* (analyst).

#### Proposed Core GEO Rubric Dimensions

Here is a starting set of dimensions. Tune based on your actual simulation capabilities and what correlates with real citation lift.

| Dimension                    | What "Good" Looks Like                                                                 | Why It Matters for GEO                          | Primary Measurement Approach                  | Example Weight |
|-----------------------------|----------------------------------------------------------------------------------------|------------------------------------------------|-----------------------------------------------|----------------|
| **Query & Intent Coverage** | Directly or strongly implicitly addresses the target queries + related follow-ups     | AI systems need clear matches to surface content | LLM judge + simulation citation for those queries | 20% |
| **Pain Authenticity**       | Uses natural, unbranded language that mirrors real buyer frustrations (no company speak) | Builds trust and relevance in synthesized answers | LLM judge on "unbranded pain resonance" + sourcing from neutral research | 15% |
| **Experiential / Authority Signals** | Specifics, edge cases, data, "we saw X when..." rather than generic claims           | Generative engines prefer attributable experience | LLM judge + presence of grounded stats/facts from `must_incorporate_facts` | 15% |
| **Structure & Scannability**| Clear H2/H3 hierarchy, lists, tables, definition blocks, Q&A patterns that are easy to extract | LLMs heavily favor parseable, modular content   | Heuristics (heading count, list density) + LLM structural quality | 12% |
| **Grounding & Factuality**  | Stays faithful to provided sources; no invented claims or stats                        | Prevents hallucination penalties and loss of trust | Grounding checker against `must_incorporate_facts` + LLM factuality pass | 12% |
| **Clarity & Signal Density**| Low fluff, high information-per-token, precise language                                | AI prefers concise, high-signal text for citation | LLM judge (conciseness + precision) + length vs. value heuristics | 10% |
| **Differentiation / Unique Framing** | Offers a fresh angle, framework, or synthesis not commonly found in top results     | Increases chance of being the "best" source chosen | LLM judge on novelty relative to common answers | 8% |
| **Simulation Performance**  | Actual lift when the content is inserted into GEO simulation environments             | Direct proxy for the real objective             | Your existing TPC simulation tooling (citation rate, rank in answer, surfaced for X journeys) | 8% (or higher if reliable) |

**Aggregating to hard/soft (example — adjust thresholds on your data)**:

- `soft = weighted average of the dimension scores (normalized 0–1)`
- `hard = 1` only if:
  - `soft >= 0.72` (tune)
  - No critical failures (e.g., grounding fails, brand leakage detected, simulation citation rate below a meaningful baseline)
  - Simulation component meets a minimum bar

Store the full per-dimension breakdown + rationales. This becomes extremely valuable context for reflection.

#### Custom Analyst Prompts

Create `skillopt/envs/geo_content/prompts/analyst_error.md` and `analyst_success.md`.

These should read like:
> "You are a senior Generative Engine Optimization strategist reviewing drafts produced under the current skill. Analyze why the content succeeded or failed across the rubric dimensions. Propose concise, generalizable edits to the *skill* (not one-off fixes for this article) that will improve future outputs on similar briefs."

Include the rubric in the prompt (or load it). Make success analysts also extract positive patterns worth codifying.

### 3.4 Gate, Scheduler, and Update Mode Recommendations (Early Phase)

- `evaluation.gate_metric: mixed` (or `soft`) with `gate_mixed_weight: 0.6` or similar while selection sets are small and rewards are continuous.
- `optimizer.skill_update_mode: rewrite_from_suggestions` or `full_rewrite_minibatch` — high-level writing strategy often benefits from bigger, more structural changes than tiny patches.
- Start with modest `learning_rate` (edit budget) — 3–6.
- Use `limit: N` heavily during benchmark development.

See also `configs/features/soft_gate.yaml` and the slow-update acceptance discussion in the main README.

## 4. Implementation Roadmap (Follows `new-benchmark.md`)

1. **Bootstrap the package**
   - Copy `skillopt/envs/_template/` → `skillopt/envs/geo_content/`
   - Rename classes, implement the minimal working versions.

2. **Define & curate the first 12–20 items**
   - Write the schema.
   - Use your GEO tooling to generate high-quality briefs + pain language.
   - Manually review for diversity and clarity of success criteria.

3. **Dataloader + splits**
   - Implement `load_split_items`.
   - Decide on initial `data/geo_content_split/` layout (can start with `split_mode: ratio` on a single JSON for speed).

4. **Rollout + artifact persistence**
   - Implement generation (single or multi-stage).
   - Persist everything the analyst will want to see.
   - Return properly shaped result dicts with `hard`/`soft`.

5. **Evaluator / Judge**
   - Implement the rubric as code + prompt(s).
   - Integrate simulation calls where available.
   - Write the aggregation logic for `hard`/`soft`.
   - Add rich per-item diagnostics.

6. **Custom analyst prompts**
   - Place rubric-aware `analyst_error*.md` and `analyst_success*.md` in the env's `prompts/` folder (the base adapter will prefer env-specific versions).

7. **Config + registration**
   - `configs/geo_content/default.yaml` (inherit from `_base_` or `soft_gate` as appropriate).
   - Add the import + registry entry in `scripts/train.py` (inside the try/except).

8. **First loop & instrumentation**
   - Run with tiny `limit` and `batch_size`.
   - Inspect `predictions/`, `steps/`, and what the analysts actually see.
   - Iterate on the judge and the custom analyst prompts (they are part of the benchmark).

9. **Harden & scale**
   - Grow the task set only after the signal feels stable.
   - Add more content types or journey stages as curriculum.
   - Consider versioning the evaluator (pin judge prompts/models).

## 5. Risks & Mitigations Specific to Abstract Content Quality

- **Reward hacking the LLM judge**: Mitigate with simulation grounding, dimension-level scrutiny, occasional human review of the selection set, and judge ensembles.
- **Judge inconsistency / drift**: Pin the judge model + full prompt (including few-shots). Log the exact judge version used for every rollout.
- **Weak correlation with real GEO lift**: Treat the evaluator itself as an optimizable artifact. Run periodic "does improving on our benchmark move real simulation metrics?" checks.
- **High cost & latency**: Aggressive limits, caching of generations during development, smaller batches, and `max_completion_tokens` discipline.
- **Overfitting to a narrow set of briefs**: Maintain diversity axes and monitor per-task-type performance in history.
- **The optimizer learns to write for the judge, not for engines**: Keep simulation signals prominent in both scoring *and* the analyst context.

## 6. Integration Points with The Prompting Company Stack

- Use existing GEO simulation prompt generation and agent simulation workflows both for **task curation** and for **evaluation signals**.
- Pain-focused, unbranded prompt banks are perfect raw material for `core_pains`.
- Buyer journey maps help ensure coverage across `buyer_journey_stage`.
- Any existing "is this content citable?" or visibility scoring can become the "Simulation Performance" dimension or a hard gate.

Document the exact integration points (endpoints, prompt templates, output formats) inside the benchmark code or a `INTEGRATION.md` next to the benchmark.

## 7. Open Questions & Future Extensions

- How much of the skill should be *general GEO principles* vs. *product/category specific tactics*? (You may want multiple related benchmarks or strong conditioning in the task items.)
- Should the rollout support tool use (web research, internal knowledge base, etc.) during generation?
- Can we create "curriculum" ordering or dynamic task sampling based on current skill weaknesses?
- How do we version and A/B the evaluator itself over time?
- Is there value in a "meta-skill" that improves the *content generation process* (research strategy, outlining heuristics) separately from final prose style?

## 8. Next Immediate Actions (Suggested)

1. Pick a tiny initial scope (e.g. 2–3 content types, one product area, 12 items).
2. Stand up the minimal runnable loop (even with a crude judge) so you can see what the analysts are actually reading.
3. Write the first version of the multi-dimensional rubric + judge prompt as a standalone artifact you can test and iterate independently of the full training loop.
4. Decide on the first simulation integration point and wire it into scoring + artifact saving.
5. Create the custom analyst prompts early — they often reveal gaps in the rubric or task schema.

---

Once the first version is running, this document can evolve into (or be complemented by) a more implementation-focused guide in the style of `new-benchmark.md`, with the actual code patterns, prompt examples, and lessons learned.

**References within the repo**:
- `docs/guide/new-benchmark.md`
- `skillopt/envs/_template/`
- `configs/features/soft_gate.yaml`
- `skillopt/envs/base.py` (prompt loading and `attach_reference_context` patterns)
- `skillopt/gradient/reflect.py` (what actually gets shown to analysts)

This plan is intentionally biased toward making the *qualitative* nature of GEO content work *with* SkillOpt's strengths (rich trajectory context for reflection + gated, stable updates) rather than fighting them. 

Feedback and iteration on the rubric dimensions, task schema, and judge design are expected and encouraged.