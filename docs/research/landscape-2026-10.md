# Deterministic Model Routing: Landscape & Feasibility Research

*Research snapshot as of 2026-10-21. Nothing here is decided, and every number below is cited. Figures that come from vendors or project authors are labelled as such.*

---

## 0. TL;DR

1. **This space is crowded, but no one has won the agent-harness niche yet.** vLLM Semantic Router (~6.2k★), Plano/Arch-Router (~7.1k★), LiteLLM's Auto Router, RouteLLM (~5.6k★), and TypeSafe's **Jev Router** (free on OpenRouter since 2026-09-25) all route between models. The closest match to the "deterministic, zero-token, harness-native" idea is **Downshift**, a Go beta with only 6★. It exists, but nobody uses it yet.
2. **The evidence warns against naive heuristics.** In LiteLLM's own benchmark, its keyword "complexity" scorer had **ROC AUC 0.524**, which is about as good as a coin flip. An independent evaluation of four open-source routers found that most gave constant or near-constant tier assignments, and that their gains tracked *how often each tier was used* rather than any task-specific targeting.
3. **What does work (cheaply):** embedding similarity with kNN matches far more complex learned routers, and runs in under 1 ms on CPU. Structural or metadata signals are surprisingly strong in agentic settings. In one study a turn-count heuristic scored 80.4% vs 82.9% for a learned router.
4. **The research gap DMR could fill:** the "Routing Plateau" paper (2026) finds that routers stall because they only look at the query. It recommends signals *beyond the query*. An agent harness already has those signals: step type, tool history, failures, context size. Routing on **harness state rather than prompt text** is the most defensible angle.
5. **Prompt-cache economics can matter more than the classifier.** Switching models mid-session can wipe out the savings, though LiteLLM disputes this with its own data. Routing at *boundaries* (session start, subagent spawn) avoids the question.
6. **Jev is fast compared to an LLM, but slow compared to local code.** It's a hosted API with 70–500 ms latency (vendor figure), closed, and waitlisted. An in-process deterministic router should be ~3–5 orders of magnitude faster and free. Jev fits best as an **optional fallback for ambiguous cases**, or as an **offline labeler**, not as the core. TypeSafe also already sells a Jev-powered router.
7. **"DMR" is already taken in this exact space:** Docker Model Runner is officially abbreviated DMR and ships a `dmr` binary.

---

## 1. Defining the problem precisely

"Deterministic" can mean three different things. A credible project should say which ones it guarantees:

| Level | Guarantee | Notes |
|---|---|---|
| **D1: No LLM in the decision loop** | Routing costs 0 generated tokens and makes no network call | This is what "zero-token" usually means (Downshift, Aurelio, LiteLLM `complexity_router`) |
| **D2: Reproducible decision** | Same `(input, policy version, catalog version)` gives the same model, every time, on every machine | Breaks easily with float reductions, GPU batching, or online learning. See §6.3 |
| **D3: Explainable decision** | Every decision returns the rule/feature trace that produced it | Valuable for audits, debugging, and employer demos |

"Minimal tokens" should be split into two numbers. **Router overhead** is the tokens and latency the routing decision itself costs; the target is 0 tokens and µs latency. **Downstream savings** are the tokens and dollars saved by sending work to cheaper models. Most projects only report the second.

---

## 2. Competitive landscape

### 2.1 Closest competitors: agent-harness, deterministic, zero-token

| Project | What it is | Status | Why it matters to DMR |
|---|---|---|---|
| **[Downshift](https://github.com/tiagovilasboas/downshift)** | Go single binary. Hooks into Claude Code / Codex / Cursor / OpenCode and rewrites the **subagent's model and effort** at spawn time. Rule-based classifier with "risk floors" (auth, payments, secrets never downshift), fail-open hooks, 10% unrouted control group | Apache-2.0, v0.1.0-beta.14, **6★**, 888 commits | **Nearly the same pitch.** Its own A/B result is ~12% list-price token savings on *mechanical* tasks and **no attributable savings on complex tasks** (self-reported). That's a sober baseline to beat. |
| **[Odd-C/agent-model-router](https://github.com/Odd-C/agent-model-router)** | Python. Compiles natural-language preferences into hard constraints and weights with deterministic rules, no LLM call | MIT | Shows the "policy compiler" idea |
| **[claude-code-router](https://www.morphllm.com/claude-code-router)** (musistudio) | Scenario rules: `default` / `background` / `think` / `longContext` (>60k tokens) / `webSearch` / `image` mapped to provider/model | Popular; one source says it broke with Claude Code 2.x (unverified) | Shows that coarse, harness-aware rules are what people actually use |
| **[ClawRouter](https://github.com/BlockRunAI/ClawRouter)** | "Agent-native" router, claims <1 ms local routing; built around x402 crypto payments | MIT | Weak fit. Mostly a payments gateway |
| `adaptive-zero-token-router`, `task-api-router` | Small zero-token intent routers listed on [GitHub topics](https://github.com/topics/prompt-routing) | Unverified | Noise-level adoption |

### 2.2 Major open-source routers (semantic / learned)

| Project | Approach | Speed / footprint | Notes |
|---|---|---|---|
| **[vLLM Semantic Router](https://github.com/vllm-project/semantic-router)** | "Signal-driven" decisions (semantic, complexity, keyword, context, structure signals). ModernBERT classifiers in a **Rust (Candle)** core with **Go** + Envoy ExtProc integration | Apache-2.0, ~6.2k★, v0.4 "Hermes" (2026-09-24) | **The strongest OSS incumbent.** Now markets itself to agent harnesses (`vllm-sr/auto`, an OpenCode guide, an [agentgateway integration](https://agentgateway.dev/docs/kubernetes/main/integrations/vllm-semantic-router/)). It uses neural classifiers, so it's not D1/D2 in the strict sense. |
| **[Plano](https://github.com/katanemo/plano)** (Katanemo, formerly Arch) + **[Arch-Router-1.5B](https://arxiv.org/abs/2506.16655)** | *Generative* router: a 1.5B (now 4B "orchestrator") LLM reads policy descriptions and outputs a route | Apache-2.0, ~7.1k★ | Spends tokens to route, the opposite of DMR's goal. Good for "preference-aligned" routing |
| **[LiteLLM Auto Router](https://docs.litellm.ai/docs/auto_router/)** | `complexity_router` (weighted keywords over 7 dimensions, sub-ms, no API call), plus semantic and LLM-classifier options | MIT; `complexity_router` in core since ~v1.95 (Aug 2026) | Their **own benchmark shows the heuristic is near-random** (§3.1) |
| **[RouteLLM](https://github.com/lm-sys/RouteLLM)** (LMSYS) | Strong-vs-weak routers trained on Arena preference data: matrix factorization (recommended), SW-ranking, BERT, causal LLM | Apache-2.0, ~5.6k★, research framework | Reports >85% cost reduction on MT-Bench vs GPT-4-only at 95% quality ([blog](https://lmsys.org/blog/2024-07-01-routellm)). Binary routing only |
| **[RoRF](https://www.notdiamond.ai/blog/rorf)** (Not Diamond) | Random forest over prompt embeddings, pairwise | Open source; 12 pre-trained routers | Pairwise only, no online learning |
| **[LLMRouter](https://github.com/ulab-uiuc/LLMRouter)** (UIUC) | Library unifying 16+ methods (kNN, SVM, MLP, MF, Elo, graph, BERT, multi-round, agentic) plus the xRouteBench benchmark | MIT | **Use as a baseline suite, not a competitor** |
| **[NVIDIA LLM Router blueprint](https://github.com/NVIDIA-AI-Blueprints/llm-router)** | v2: intent routing (Qwen 1.7B) or auto-routing (CLIP embeddings + neural net). Recommends a model rather than proxying | Experimental | GPU-heavy |
| **[Aurelio semantic-router](https://github.com/aurelio-labs/semantic-router)** | Embedding-similarity *intent* routing ("decides in milliseconds") | MIT, ~4.0k★ | Built for intents, not model selection, but the technique carries over |
| **Lynkr** | Embedding-anchor intent + risk analyzer + session pinning + verification cascade | — | Self-reported RouterArena #15 of 27, 10–50 ms warm ([post](https://dev.to/lynkr/lynkr-vs-litellms-new-complexityrouter-464p)) |

### 2.3 Commercial / closed routers

| Product | Notes |
|---|---|
| **[TypeSafe Jev Router](https://openrouter.ai/docs/guides/routing/routers/jev-router)** (`typesafe/jev-router` on OpenRouter) | Launched 2026-09-25, **priced at $0**. Picks model *and reasoning effort*, and can keep, adjust, or switch models across agent turns. Returns a decision trace (`candidates`, `reason`). No published comparative results yet ([RuntimeWire](https://runtimewire.com/article/typesafe-jev-router-openrouter-launch)). |
| **[OpenRouter Auto / Auto Beta](https://openrouter.ai/docs/guides/features/model-routing)** | Auto was powered by Not Diamond and is being deprecated. Auto Beta classifies the task, then uses OpenRouter usage rankings. **Pins the model per conversation for prompt-cache hits.** |
| **Not Diamond, Martian** | Learned per-query routers sold as SaaS |
| **[Azure Foundry Model Router](https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/model-router-how-it-works)** / **Bedrock Intelligent Prompt Routing** | Cloud-native, within a model family. Bedrock claims up to 30% cost reduction |
| **[Cursor Router](https://cursor.com/help/models-and-usage/cursor-router)** / **[Copilot auto](https://docs.github.com/en/copilot/concepts/auto-model-selection)** | ML classifier with Cost/Balance/Intelligence modes. Users complain they [can't see which model ran](https://forum.cursor.com/t/show-which-model-handled-each-step-when-using-auto-mode/164163). Copilot routes "along natural prompt-caching boundaries." |

**Takeaway:** the big closed products are converging on *cache-aware, per-session, effort-aware* routing. That confirms the problem is real. It also means "pick a model per prompt" on its own isn't novel.

### 2.4 Gateways (speed references, not "smart" routers)

[TensorZero](https://tensorzero.com/docs/gateway/benchmarks) (Rust, claims <1 ms P99 overhead) and [Bifrost](https://github.com/maximhq/bifrost-benchmarking) (Go, claims ~11 µs mean overhead at 5k RPS) are the bar for "fast." Both figures are self-reported. Portkey, Kong, APISIX, and Helicone are mostly about failover and keys, not quality-aware selection.

---

## 3. What the evidence says (and doesn't)

### 3.1 Pure keyword/complexity heuristics don't predict when a cheap model is enough
[LiteLLM's benchmark](https://docs.litellm.ai/docs/proxy/auto_routing_benchmark.md) used 300 prompts, a three-model Gemini ladder, and a cross-family judge:

| Strategy | Win rate vs flagship | Savings | Exact match (90 verifiable) |
|---|---:|---:|---:|
| Semantic, threshold 0.3 | 48.2% | 47.5% | 82/90 |
| `complexity_router`, fitted | 41.5% | 58.3% | 71/90 |
| **Shuffled control** (random, same tier shares) | 39.2% | 59.4% | 68/90 |
| All cheap | 43.7% | 97.4% | 81/90 |

The heuristic's AUC was **0.524** (0.420 on chat). No threshold beat "always say cheap is enough." The features describe *prompt type*, not *answer adequacy*. **Lesson for DMR:** a rule table over prompt keywords is the wrong design. The rules must use signals that actually correlate with outcomes.

### 3.2 Simple embedding + kNN is competitive with complex learned routers
[Li (2025)](https://arxiv.org/abs/2505.12601) shows a well-tuned kNN matches or beats state-of-the-art learned routers, with sub-millisecond per-query cost on one CPU core. [The Routing Plateau (2026)](https://arxiv.org/abs/2606.07587) studied 21 methods on 5 benchmarks. Most land in a narrow accuracy band (kNN included) **far below the oracle**. More data and bigger encoders added only ~1.24 pp. The authors' diagnosis is that routers learn *average model strength*, not per-query signal. They recommend **inputs beyond the query** (e.g. partial trajectories).

### 3.3 Many routers ≈ fixed-tier baselines
[Kumar & Saminathan (2026)](https://arxiv.org/abs/2608.14641) evaluated four OSS routers on 290 frozen tasks across RouterBench, BFCL v4, tau2-bench, and WebArena. Three gave constant or near-constant tier assignments. **Always-Mid matched Aurelio exactly on 3 of 4 benchmarks.** vLLM-SR showed no task-specific advantage over a content-blind allocation with the same tier shares. **Lesson:** every result DMR publishes needs a *matched-share random control* and fixed-tier baselines, or it proves nothing.

### 3.4 Agentic and tool-calling routing is different, and structural signals are strong
[Switchcraft (Microsoft Research, 2026)](https://arxiv.org/abs/2605.07112): a 66M DistilBERT router over latest turn + tool signatures + metadata (length, #tools, #turns), with 8 candidate models.
- Learned router 82.9% vs best single model 82.3% vs **oracle 89.4%**, with ~84% cost cut.
- **Heuristic baselines: turn count 80.4%, input length 78.8%, tool count 75.6%.** Cheap structural features get you most of the way.
- Weak on multi-turn (61.5%). New models require retraining. P99 latency is 3–17 ms on a T4 GPU.

[Agent-as-a-Router (2026)](https://arxiv.org/abs/2606.22902) blames an **"information deficit."** Adding task-dimension performance statistics to a vanilla router gave a 15.3% relative gain. It introduces CodeRouterBench (~10K coding tasks, 8 frontier LLMs, regret-based). [MTRouter](https://arxiv.org/abs/2604.23530) does turn-level routing over history embeddings.

### 3.5 Prompt-cache economics can dominate
- **Switching is costly** ([agentpatterns.ai](https://agentpatterns.ai/patterns/agent-design/cache-safe-routing-boundaries/), [JFrog](https://jfrog.com/blog/why-model-routing-backfires)). A mid-task switch forces a cold write of the whole history. Break-even after n\* requests: `n* = (W_target − R_target) / (R_home − R_target)` (one-way), where W is cache-write price and R is cache-read price. Using 2026-09-21 Anthropic list prices, Opus 5 → Sonnet 5 breaks even after **8** requests one-way, **27** with a return. **Safe routing points: session start, a fresh side lane, a subagent spawn.** Reasoning effort can change mid-conversation without breaking the cache.
- **Counter-claim** ([LiteLLM, vendor data](https://docs.litellm.ai/docs/auto_router/prompt_caching)): router + cache beat fixed model + cache on every dataset (37.4% cheaper on 95 real agent sessions). Switching back to a previous model found a warm cache 97.4% of the time (5-min TTL). Turning the cache *off* costs ~4×.
- **Reconciliation:** both can be true. The outcome depends on session length, price ratios, and switch frequency. **DMR should treat cache state as a first-class input** and compute n\* instead of choosing a side.

### 3.6 Routers are an attack surface
[Rerouting LLM Routers (2025)](https://arxiv.org/abs/2501.01818): short "confounder gadget" strings push routers to the expensive model, in both white-box and black-box settings. [RerouteGuard (2026)](https://arxiv.org/abs/2601.21380) says real systems remain vulnerable to cost escalation. **Rule-based routers are even easier to game** ("this is a trivial task…"). DMR needs monotonic risk floors and should route on signals the user prompt can't forge (tool names, file paths, step type).

---

## 4. Jev: an honest assessment

### What it is
TypeSafe's "System One" model, launched 2026-09-15. You send a `state` plus typed questions (**Choice** up to 255 options, **Score**, **Noul** = calibrated yes/no). It evaluates them in **one parallel pass**, without generating text, and returns probabilities and confidence ([eesel](https://www.eesel.ai/blog/jev-ultrafast), [The Register](https://www.theregister.com/a/5298431), [DEV](https://dev.to/bibryam/jev-and-llms-who-does-what-189k)).
- Latency (vendor): 70–500 ms end-to-end, as low as ~150 ms. One demo took ~620 ms.
- Price: $0.042 per million input tokens, output free. TypeSafe says it can't prove the price isn't subsidized.
- Access: early-access waitlist. Also available on Cloudflare Workers AI (`jev-1.13.0`, 32k context), Vercel AI SDK (`experimental_evaluate`), and LiteLLM passthrough.
- Its creator explicitly suggests it for **routing tools and MCP calls**.

### Rebuttal to "Jev would help with speed"
| Concern | Detail |
|---|---|
| **It's fast *relative to LLMs*, not relative to local code** | A 70–500 ms network round-trip vs an in-process rule/kNN lookup in roughly µs (estimate, to be benchmarked) is a ~10³–10⁵× gap. On the hot path, the fastest router makes no network call at all. |
| **Not open source, not under your control** | It's a closed, waitlisted, hosted API. Making it the core would contradict the "open source + deterministic" pitch and create a single-vendor dependency. |
| **Determinism (D2) isn't guaranteed** | Hosted model versions change (`jev-1.13.0`). Outputs are probabilities. Reproducing a decision in 6 months isn't possible unless TypeSafe pins versions forever. |
| **You'd be competing with the vendor's own product** | TypeSafe already ships **Jev Router** for free on OpenRouter. |
| **Quality is unproven** | Public benchmarks are mostly vendor-made. The 193.6× headline comes from TypeSafe's own workflow, with "correct" answers averaged from two frontier models. Critics ask "is it good?" ([The Register](https://www.theregister.com/a/5298431)). |
| **Privacy** | Prompt and state leave the machine. A local router keeps them in-process, which is a selling point. |

### Where Jev (or a Jev-like model) *could* fit well
1. **Optional fallback for ambiguous cases:** only when the deterministic layer abstains (low margin), and behind an interface so any backend can plug in.
2. **Offline teacher/labeler:** use it to label training data for the deterministic router. Tokens are spent once, offline, and the shipped router stays zero-token.
3. **Comparison baseline:** benchmark DMR vs Jev Router vs vLLM-SR on the same frozen tasks. That's a compelling chart for employers.
4. **Open-source Jev-alikes** ([DataCamp list](https://www.datacamp.com/blog/top-open-source-jev-alternatives)): Laya (ModernBERT ~322–421M), **Von** (~395M, no longer maintained), Kev (Qwen3.5 0.8–9B, implements the `/v1/systemone` API), NanoJev (Qwen3-0.6B), Nimble (Qwen3.5-9B LoRA). They run locally but are still neural-net-sized (ms on GPU, slower on CPU). They're useful as a pluggable backend, not as the core.

> ⚠️ **Check:** I took "Jev" to mean TypeSafe's Jev. If you meant something else (a language/runtime?), tell me and I'll redo this section.

---

## 5. Where DMR could differentiate (candidate "wedges")

Ranked by how defensible they are given the evidence above:

1. **Harness-native signals, not prompt text.** Route on structured state the harness already has: step kind (plan / edit / search / summarize / test-fix), subagent type, tool being invoked, number of files touched, context tokens, turn index, recent tool errors/test failures, retries, file-path risk class (auth/payments/secrets), model capability needs (vision, tools, context length). This follows directly from the Routing Plateau and Agent-as-a-Router findings, and it's hard to forge via prompt injection.
2. **Cache-aware decision boundaries.** Route only at zero-cost points (session start, subagent spawn, cold cache) unless `n*` says a mid-session switch pays off. Change *reasoning effort* freely, since it doesn't break the cache.
3. **Strict D1+D2+D3.** Bitwise-reproducible decisions given versioned policy + catalog, with an explanation trace on every decision. This is what Jev Router, Cursor, and vLLM-SR can't offer, and it suits audits and regulated environments.
4. **Measured honestly.** Matched-share random controls, fixed-tier baselines, oracle gap, CIs. Most competitors (Downshift partly excepted) publish none of this. **Even a null result, written up rigorously, is portfolio-worthy.**
5. **µs, in-process, zero network.** A library first, with optional thin adapters (harness hooks, LiteLLM custom router, HTTP sidecar).

What is **not** a wedge: "picks a model per prompt" (commoditized), "OpenAI-compatible proxy" (dozens exist), "uses an LLM to decide" (the opposite of the goal).

---

## 6. Design-space notes (ideas to evaluate, not decisions)

### 6.1 A layered pipeline (each layer can abstain and pass to the next)
```
L0  Hard constraints   capability/context/tool/vision/JSON/cost-cap/residency filter   pure, µs
L1  Policy rules       harness-state decision table + monotonic risk floors            pure, µs
L2  Learned-but-frozen static-embedding / hashed-n-gram kNN or linear scorer           CPU, sub-ms
                       (weights versioned; integer/fixed-point for bitwise determinism)
L3  Escalation         cascade on *verifiable* failure signals (tests fail, tool error) deterministic policy
L4  Optional oracle    pluggable: Jev / Jev-alike / small LLM, ONLY on abstain        opt-in, costs tokens
```
- **Fail-open:** if in doubt, keep the requested model (Downshift does this).
- **Monotonicity invariants:** raising risk never lowers the tier, and more context never picks a model with a smaller window. These are property-testable.

### 6.2 Fast, zero-token feature extraction options
- **Static embeddings** ([Model2Vec](https://pypi.org/project/model2vec/), Rust crate [`model2vec-rs`](https://docs.rs/crate/model2vec-rs)): token lookup + mean pool. The project claims up to 500× faster than the source sentence transformer on CPU (self-reported).
- **Hashed n-grams** (FNV/xxHash) + linear model: fully integer, trivially deterministic. Downshift uses an FNV word-hash "boost."
- **kNN over a frozen anchor set:** supported by §3.2. Exact search over a few thousand anchors is cheap and deterministic, with no approximate-NN randomness.

### 6.3 Determinism pitfalls to engineer around
- GPU inference is non-deterministic mainly because kernels aren't **batch-invariant** ([Thinking Machines, 2025](https://simonwillison.net/2025/Sep/11/defeating-nondeterminism/)). Keep the router on CPU and single-item.
- Float reductions with variable order (SIMD/threads) can flip near-tie decisions. Use quantized/integer scores, a fixed reduction order, or deterministic tie-breaking (e.g. by model id).
- Online learning breaks D2. Learn **offline**, ship **versioned snapshots**, and log `policy_version` + `catalog_version` with every decision.
- Model catalog and prices change constantly. Store them as dated, versioned data, not code.

### 6.4 Language choice
- **Rust core + Python bindings (PyO3)** *(recommended if you're comfortable with Rust)*. Fastest and memory-safe, with the strongest "systems" signal to employers. Python bindings reach the eval/ML ecosystem and LiteLLM. vLLM-SR's classifier core is Rust for the same reasons.
- **Go**: simpler, gives a single static binary, and makes harness hooks/sidecars easy (Downshift, Bifrost, vLLM-SR control plane). A good choice if shipping speed matters more than raw ns.
- **Rebuttal to "language = speed":** the router's cost is dwarfed by the LLM call (hundreds of ms to minutes). Avoiding *network and model calls* matters more than Rust vs Go vs Python. Choose the language for the integration targets and your own fluency. Benchmark before optimizing.

---

## 7. Methodology that would impress a reviewer

**Evaluation design**
- **Pre-register** hypotheses and metrics in `docs/methodology.md` before running anything.
- **Baselines (all required):** always-cheap, always-mid, always-strong, **matched-share random (shuffled) control**, oracle, plus existing routers (RouteLLM-MF, kNN, LiteLLM `complexity_router`, vLLM-SR, and optionally Jev Router).
- **Metrics:** task success by *execution* where possible (tests, AST checks), not LLM judges (LiteLLM's judge disagreed with itself on 32% of order-swapped pairs). Also report cost with cache accounting, % of oracle gap closed, AUC of "cheap is sufficient," tier distribution, router latency p50/p99/p99.9, and decision stability across platforms.
- **Statistics:** bootstrap CIs, per-category breakdowns, and an explicit equivalence margin.

**Datasets (cheapest first)**
1. **Offline outcome matrices** (no API spend): [LLMRouterBench](https://arxiv.org/abs/2601.07206) (per-prompt, per-model data, 21-model pool), [RouterBench](https://arxiv.org/abs/2403.12031).
2. **Public leaderboard for outside validation:** [RouterArena](https://arxiv.org/abs/2510.00202) (ICLR 2026, ~8.4k queries, 9 domains, includes robustness to paraphrase).
3. **Agentic:** BFCL v4, tau2-bench, CodeRouterBench, a SWE-bench Verified subset. Switchcraft's corrected ConFETTI data is promised as open source.
4. **Your own harness traces**, frozen and replayed, with cache accounting.

**Engineering hygiene**
- Golden-file decision tests, cross-platform bitwise checks in CI (x86 + ARM), property-based tests for the monotonic invariants, fuzzing of the policy parser.
- Microbenchmarks (`criterion` for Rust, `pytest-benchmark` for Python) with a regression gate in CI.
- `make reproduce` that regenerates every chart from pinned data.
- Docs: README with a one-sentence value prop and one benchmark chart; `ARCHITECTURE.md`; ADRs under `docs/adr/`; a "policy card" (like a model card) describing what the router optimizes and where it fails; CONTRIBUTING, CHANGELOG, SemVer, Apache-2.0.
- **Publish negative results.** Downshift's "no attributable savings on complex tasks" line makes it more credible, not less.

---

## 8. Risks & blind spots (things we might be missing)

| Risk | Why it matters | Mitigation idea |
|---|---|---|
| **Name collision: "DMR" = Docker Model Runner** | Same AI-tooling space, official abbreviation, ships a `dmr` binary ([Docker docs](https://docs.docker.com/ai/model-runner/)). Hurts searchability and could confuse employers | Rename the package/binary (keep "Deterministic Model Routing" as a tagline), or accept it knowingly |
| **The value might be small** | Downshift saw ~12% on mechanical tasks and ~0% on complex ones. Several routers ≈ fixed-tier baselines | Frame the project as *router + rigorous evaluation*, so a null result is still a contribution |
| **Crowded and moving fast** | vLLM-SR is targeting harnesses. Jev Router is free. Cursor and Copilot do it natively | Pick the narrow wedge (§5); don't build another proxy |
| **Harness hook fragility** | Reports from Apr–May 2026 said Claude Code's PreToolUse `updatedInput` was ignored for the Agent tool ([#44412](https://claudeissues.com/issue/44412-bug-pretooluse-hook-updatedinput-is-ignored-for-the-agent-tool), [#39814](https://claudeissues.com/issue/39814-pretooluse-hook-updatedinput-silently-ignored-for-agent-tool)). Both issues are closed and Downshift lists it as working. **Unverified which is current** | Library-first core; adapters are thin and separately versioned |
| **Labels are expensive** | Per-step agent outcome labels mean running every candidate model | Start with existing outcome matrices; use replay; distill from a teacher offline |
| **Cross-provider state** | Thinking blocks, `cache_control`, and tool-call formats differ by provider | Prefer within-provider ladders first; treat a provider switch as a hard boundary |
| **Prompt-injected routing** | Confounder gadgets, "this is easy" text | Route on unforgeable structural signals; monotonic risk floors |
| **Stale catalog/prices** | Wrong prices mean wrong break-even and wrong savings | Dated catalog files, plus a check command (Downshift does this) |
| **"Deterministic" vs adaptive** | Users want it to learn; learning breaks D2 | Versioned offline retraining; determinism *per version* |

---

## 9. Open questions to decide next

1. **Primary consumer:** a specific harness (Claude Code / OpenCode / Codex subagents), any harness via library, or a gateway (LiteLLM custom router)?
2. **Routing granularity:** session-level, subagent-level, or per-turn (cache-gated)?
3. **Scope of "model choice":** model only, or model + reasoning effort (Jev Router and Downshift both do effort)?
4. **Language:** Rust core + Python bindings, or Go?
5. **Name:** keep DMR despite Docker Model Runner?
6. **Jev's role:** none, optional fallback adapter, offline labeler, or comparison baseline only?
7. **Success criterion:** what result would make you say "this worked"? (e.g. "≥X% cost cut at ≤Y pp quality loss vs always-strong, beating the matched-share control with a CI excluding 0")

---

## Sources

**Jev / TypeSafe:** [eesel](https://www.eesel.ai/blog/jev-ultrafast) · [The Register](https://www.theregister.com/a/5298431) · [DEV (Ibryam)](https://dev.to/bibryam/jev-and-llms-who-does-what-189k) · [OpenRouter Jev Router docs](https://openrouter.ai/docs/guides/routing/routers/jev-router.md) · [RuntimeWire](https://runtimewire.com/article/typesafe-jev-router-openrouter-launch) · [DataCamp OSS alternatives](https://www.datacamp.com/blog/top-open-source-jev-alternatives) · [open-alternative-jev](https://github.com/ikermoel/open-alternative-jev) · [Von](https://github.com/wfzyx/von)

**Routers / projects:** [Downshift](https://github.com/tiagovilasboas/downshift) · [vLLM Semantic Router](https://github.com/vllm-project/semantic-router) · [vLLM-SR paper](https://arxiv.org/abs/2510.08731) · [Workload-Router-Pool vision paper](https://arxiv.org/abs/2603.21354) · [Plano](https://github.com/katanemo/plano) · [Arch-Router](https://arxiv.org/abs/2506.16655) · [LiteLLM Auto Router](https://docs.litellm.ai/docs/auto_router/) · [RouteLLM](https://github.com/lm-sys/RouteLLM) · [RouteLLM blog](https://lmsys.org/blog/2024-07-01-routellm) · [RoRF](https://www.notdiamond.ai/blog/rorf) · [LLMRouter](https://github.com/ulab-uiuc/LLMRouter) · [NVIDIA blueprint](https://github.com/NVIDIA-AI-Blueprints/llm-router) · [Aurelio semantic-router](https://github.com/aurelio-labs/semantic-router) · [Odd-C agent-model-router](https://github.com/Odd-C/agent-model-router) · [ClawRouter](https://github.com/BlockRunAI/ClawRouter) · [claude-code-router guide](https://www.morphllm.com/claude-code-router) · [Awesome-LLM-Router](https://github.com/ishandutta2007/Awesome-LLM-Router) · [Lynkr vs LiteLLM](https://dev.to/lynkr/lynkr-vs-litellms-new-complexityrouter-464p) · [OpenRouter model routing](https://openrouter.ai/docs/guides/features/model-routing) · [Azure model router](https://learn.microsoft.com/en-us/azure/foundry/openai/concepts/model-router-how-it-works) · [Cursor Router](https://cursor.com/help/models-and-usage/cursor-router) · [Copilot auto model selection](https://docs.github.com/en/copilot/concepts/auto-model-selection) · [TensorZero benchmarks](https://tensorzero.com/docs/gateway/benchmarks) · [Bifrost benchmarking](https://github.com/maximhq/bifrost-benchmarking)

**Research:** [kNN routers (Li 2025)](https://arxiv.org/abs/2505.12601) · [Routing Plateau (2026)](https://arxiv.org/abs/2606.07587) · [Four OSS routers, common interface (2026)](https://arxiv.org/abs/2608.14641) · [Switchcraft (2026)](https://arxiv.org/abs/2605.07112) · [Agent-as-a-Router (2026)](https://arxiv.org/abs/2606.22902) · [MTRouter](https://arxiv.org/abs/2604.23530) · [RouterBench](https://arxiv.org/abs/2403.12031) · [RouterArena](https://arxiv.org/abs/2510.00202) · [LLMRouterBench](https://arxiv.org/abs/2601.07206) · [Routing & cascading survey (2026)](https://arxiv.org/abs/2603.04445) · [Rerouting LLM Routers](https://arxiv.org/abs/2501.01818) · [RerouteGuard](https://arxiv.org/abs/2601.21380) · [LiteLLM routing benchmark](https://docs.litellm.ai/docs/proxy/auto_routing_benchmark.md) · [LiteLLM routing + caching](https://docs.litellm.ai/docs/auto_router/prompt_caching) · [Cache-safe routing boundaries](https://agentpatterns.ai/patterns/agent-design/cache-safe-routing-boundaries/) · [JFrog: why routing backfires](https://jfrog.com/blog/why-model-routing-backfires) · [Defeating nondeterminism (summary)](https://simonwillison.net/2025/Sep/11/defeating-nondeterminism/) · [Model2Vec](https://pypi.org/project/model2vec/) · [Docker Model Runner](https://docs.docker.com/ai/model-runner/)
