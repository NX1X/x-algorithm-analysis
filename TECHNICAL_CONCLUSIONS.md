# X "For You" Algorithm - Technical Conclusions

📂 [Overview](README.md) · 🔧 **Technical** *(you are here)* · 👤 [For Users](WHAT_USERS_SHOULD_KNOW.md)

> Audience: engineers / ML practitioners. Based on a full source read of `phoenix/`,
> `home-mixer/`, `thunder/`, `candidate-pipeline/`, `grox/`. Analysis date: 2026-05-17.

---

## 1. System Shape

A two-stage **retrieval → ranking** recommender, orchestrated in Rust, with the ML in
Python/JAX and content understanding via Grok LLM.

```
Request → home-mixer
            ├─ query hydration (user context)
            ├─ sources:  Thunder (in-network, recency)  +  Phoenix retrieval (OON, ML)
            ├─ candidate hydration + pre-scoring filters
            ├─ Phoenix ranker (multi-action transformer)
            ├─ RankingScorer (weighted sum + diversity + OON) → optional VMRanker/DPP
            ├─ top-K select + post-selection visibility filtering
            └─ ads/WTF/prompts blending → ranked feed
grox (async, continuous): Grok-LLM annotations → Unified Post Annotations → ranking + enforcement
```

---

## 2. ML Core (Phoenix)

**Backbone (`grok.py`)** - decoder-only transformer ported from Grok-1:
- RMSNorm (fp32, eps 1e-5), RoPE (base 1e4), GeGLU FFN, **sandwich-style** norm
  (normalizes sublayer output before residual add).
- tanh logit soft-cap at ±30, fp32 softmax, `attn_output_multiplier = 0.125`
  (explicit 1/8 attention temperature, not 1/√d).
- GQA is *configured* (`num_q_heads`/`num_kv_heads`) but **every provided config sets them
  equal - GQA is unused**.
- **No Mixture-of-Experts** despite the filename; `DenseBlock` is a single dense GeGLU.

**Ranker (`recsys_model.py`)**:
- Sequence = `[user(1)] + [history(S)] + [candidates(C)]`.
- **Candidate isolation mask** (`make_recsys_attn_mask`): causal among user+history;
  each candidate attends to user+history + only itself. → all candidates scored in one
  parallel forward pass, mutually independent, cacheable. Exhaustively unit-tested.
- Hash embeddings pre-looked-up outside the model; combined via learned projection blocks.
  Hash value 0 reserved for padding everywhere.
- **Signed action encoding** `(2·a − 1)`: distinguishes absent action (−1) from padding
  (masked). Candidates get no action embeddings (no engagement yet).
- Post-age bucketing: 60-min granularity to 80h (`POST_AGE_MAX_MINUTES=4800`), vocab 82.
- ~19 discrete actions (sigmoid, multi-label - **not softmax**) + continuous head
  (dwell). Negative-feedback indices `[14,15,16,17]` = not-interested/block/mute/report.
- **Right-anchored RoPE**: newest history pinned to a fixed position; all candidates share
  one position - removes inter-candidate positional bias, normalizes recency across
  variable history lengths.

**Retrieval (`recsys_retrieval_model.py`)**:
- User tower = full transformer → masked mean-pool → L2-norm.
- Candidate tower = **transformer-free** (MLP/SiLU or parameter-free mean-pool) → L2-norm,
  enabling full-corpus offline precompute.
- Retrieval = cosine similarity + `lax.top_k`.

**Caveats:**
- All learnable params init to constant 0 → meaningful only with a loaded checkpoint;
  demos/tests run a degenerate net and assert shapes only.
- **Two conflicting action-index conventions**: `runners.py ACTIONS` (fav=0…) vs
  `run_pipeline.py IDX_*` (fav=1…). The released pipeline hardcodes a blend
  `fav·1.0 + reply·0.5 + rt·0.3 + dwell·0.2` while the runner hardcodes `favorite_score`
  as the sole sort key - two definitions of "the score".
- `run_pipeline.py` carries checkpoint-compat hacks (fabricated 64 global-negative
  candidates, unused `log_temperature`).

---

## 3. Orchestration (Home-Mixer)

- **Two nested pipelines**: `PhoenixCandidatePipeline` (heavy, on `PostCandidate`) wrapped
  as one `Source` inside `ForYouCandidatePipeline` (feed assembly + ads/WTF/prompts/
  push-to-home). Each request runs the full inner pipeline synchronously.
- ~15 query hydrators (scoring/retrieval action sequences, social graph, topics, starter
  packs, impression Bloom filter, mutual-follow MinHash, demographics, geo-IP, inferred
  gender). Sequences come from a User Action Aggregation service.
- ~14 sequential pre-scoring filters; post-selection: `VFFilter`, `AncillaryVFFilter`,
  `DedupConversationFilter`. Visibility is dual-pathed (in-network `TimelineHome` vs OON
  stricter `TimelineHomeRecommendations`).
- **Scoring (`RankingScorer`)** - the production combiner:
  - Weighted linear sum of ~21 Phoenix heads; **all weights are feature-switch-tunable at
    request time** (no hardcoded constants in the live path - legacy `weighted_scorer.rs`/
    `oon_scorer.rs`/`author_diversity_scorer.rs` are dead code, absent from `mod.rs`).
  - Negative heads (not-interested/block/mute/report/not-dwelled) subtracted.
  - Offset/normalization rescales negatives into a positive band.
  - **Author diversity**: per-author k-th post ×`(1−floor)·decay^k + floor`.
  - **OON multiplier**: distinct factors for normal / topic / eligible-new-user.
  - Optional **VMRanker** value model with **DPP** diversity - overrides final score.
- **New-user model swap**: short action sequence → dedicated Phoenix retrieval+scoring
  cluster (live-swappable via deciders).
- **180s Redis candidate cache**: hit (≥500 posts) skips retrieval + Phoenix scoring;
  only `RankingScorer`/`VMRanker` re-rank cached pre-scored candidates.
- **Ads**: default `PartitionOrganicAdsBlender` caps ads ≤ 50% of safe posts, enforces
  BSR-low/handle/keyword adjacency drops, never ends feed on an ad. Brand safety:
  `BrandSafetyVerdict` defaults to `MediumRisk` on missing data; hardcoded snowflake gate
  `PTOS_CUTOFF_TWEET_ID = 2_054_275_414_225_846_272` forces newer un-PTOS-reviewed content
  to MediumRisk; un-Grok-scored content → MediumRisk (fail-closed for monetization).
- Probabilistic telemetry: 5% reranking/stats sampling, 50% shadow traffic; cross-DC
  Redis replication for Phoenix request reconstruction.
- **In-progress migration**: `EnableUrtMigrationComponents` gating + dead hydrators/
  scorers/side-effects referencing stale module paths.

---

## 4. In-Network Store (Thunder)

- Fully in-memory, Kafka-fed, two roles (feeder normalizes Thrift→protobuf; serving
  builds store + gRPC). Each serving replica uses a UUID-suffixed consumer group →
  consumes **all** partitions, holds a **full** copy (memory/Kafka cost ∝ replica count).
- **TinyPost (16B) / LightPost split**: time-sorted per-author `VecDeque` for cache-
  friendly scan; full payload `DashMap` lookup O(1) only for survivors → sub-ms reads.
- Deletes tombstoned via synthetic `DELETE_EVENT_KEY` user so the same time-ordered
  trimmer expires them; reads filter deletes lazily. Auto-trim every 2 min, front-pop
  O(1), `shrink_to` reclamation, TOCTOU-safe `remove_if`.
- Ranking = **recency only**. Per-request deadline + `try_acquire` load-shedding bound
  tail latency; RAII in-flight gauges/timers.
- **Concerns**: pervasive `panic!`/`.unwrap()` on Kafka init/loop and proto fields (no
  graceful degradation); V1 feeder spawns one Tokio task per message; Strato follow-graph
  fallback is `&& req.debug`-gated → empty production feed if caller omits
  `following_user_ids`; a comment/code mismatch on batch-log cadence.

---

## 5. Pipeline Framework (Candidate-Pipeline)

- Traits: `QueryHydrator / Source / Hydrator / Filter / Scorer / Selector / SideEffect`,
  fixed 10-stage `execute`.
- **Parallel**: query hydrators, sources, hydrators. **Sequential**: filters, scorers
  (compose). **Detached**: side effects (true fire-and-forget, never add latency).
- **Two-phase hydrate/filter**: broad/cheap before scoring; expensive only on selected
  winners. Truncation overflow preserved into `non_selected` for side effects.
- **Fault-tolerant by construction**: `execute` cannot fail; failed source → 0
  candidates, failed hydrator → fields untouched, length-mismatch poisoned safely.
- Observability woven into trait defaults: stats macros (bucketed latency/size), tracing
  spans with `disabled` component lists, `result_empty` alarm, cache hit/miss, per-filter
  kept/removed. Auto-derived component names via `type_name_of_val`.

---

## 6. Content Understanding (Grox)

- Multi-process async: Kafka topic → generator (injects `TaskEligibility`) → dispatcher →
  engine → `PlanMaster` races 9 plans, only the eligible plan runs → DAG of tasks with
  **skip-propagation** (a skipped dependency transitively skips downstream) → sinks.
- Classifiers use Grok `VisionSampler`, temperature ≈ 1e-6: spam (low-follower replies),
  banger initial screen (quality/topics/tags, gate `score ≥ 0.4`), post safety deluxe,
  reply ranking (mini→bigger fallback, 3-tier parse repair), PTOS category→policy
  (deluxe uses EAPI reasoning Grok for adult/violent).
- Embedders: V2 (qwen3/v4, summary-based, legacy) and **V5** (HTTP, `RECSYS_EMBED_V5`,
  Matryoshka-truncated to 1024-d + renorm, interleaved multimodal + ASR transcript).
- ASR: ffmpeg → base64 → OpenAI-compat endpoint, TTL-cached + in-flight coalescing.
- **Feedback loop**: outputs (quality/slop/minor scores, topic taxonomy, tags, bool
  metadata, summary, reply/spam scores) written to Unified Post Annotations / Manhattan /
  Kafka → become **direct ranking features and automated enforcement labels**.
- Safety posture: labels **OR-merge upward** (sticky); a separate sex/nudity model can
  force-escalate but not de-escalate; deluxe PTOS injects a forced adult-content recheck
  on every post. Asymmetric failure: reply-ranking defaults to 3.0 (good) on parse
  failure; safety biases toward over-suppression.
- **Concerns**: LLM verdicts (version-sensitive) directly gate visibility/enforcement;
  thresholds/prompts blanked in the dump (won't run as-is, operative values undisclosed);
  `KafkaLoader.ack` no-op + final-failure-acks-anyway can silently drop annotations;
  `TaskGrokUpaActionWithLabels` picks first result with bool metadata (order-sensitive).

---

## 7. Key Engineering Takeaways

1. **Candidate isolation** is the architectural keystone - enables parallel scoring,
   determinism, and the home-mixer candidate cache.
2. **Feature-less, sequence-driven** modeling concentrates all relevance logic in one
   transformer; simpler pipelines, opaque behavior, total checkpoint dependence.
3. **Negative feedback is a first-class loss term** - the model optimizes against
   block/mute/report, not only for engagement.
4. **Differentiated risk postures**: fail-closed for safety/ads, fail-open (silent
   degradation) for the serving path. Telemetry is load-bearing for detecting the latter.
5. **Mature production system, partial release**: probabilistic logging, cross-DC caches,
   cold-start throttling, RAII metrics - but mini frozen checkpoints, blanked config,
   dual conventions, and migration dead code mean it is representative, not reproducible.

---

## 8. Cross-Cutting Themes

1. **Model does the heavy lifting.** Eliminating hand-engineered features pushes all
   relevance logic into one transformer learning from raw engagement - simpler pipelines,
   but opaque and wholly dependent on the trained checkpoint's quality/freshness.
2. **Candidate isolation is the keystone.** The attention mask makes scoring parallel,
   deterministic, and cacheable - and enables the 180s candidate-cache fast-path.
3. **Negative feedback is a first-class objective.** The model explicitly predicts and
   subtracts not-interested/block/mute/report - it optimizes against being disliked.
4. **Fail-closed for safety/ads, fail-open for general degradation.** Conservative
   suppression on missing safety data; silent stale/zero degradation on the serving path.
5. **Heavy operational engineering.** Sub-ms stores, two-phase pipelines, probabilistic
   logging, regional Redis replication, cold-start throttling - a mature production system.
6. **Partial, snapshot-only release.** Mini frozen checkpoints, blanked thresholds/prompts,
   dual conventions, migration dead code - representative and educational, not reproducible.

---

## 9. Overall Assessment - Strengths vs Risks

**Strengths**
- Architecturally elegant: clean two-stage recsys, composable observable pipeline
  framework, candidate isolation enabling cache + parallelism.
- Strong latency engineering (Thunder TinyPost split, candidate cache, deadlines,
  load-shedding).
- Safety/monetization defaults are conservative and fail-closed.
- Negative-feedback modeling actively protects user experience.

**Risks / Concerns**
- Total dependence on an opaque transformer with no interpretable feature attribution.
- LLM-generated content verdicts (Grox) directly drive visibility and enforcement -
  non-deterministic across model versions; single hardcoded thresholds gate promotion.
- Silent degradation paths (Gizmoduck 200ms timeout → default viewer; missing
  `following_user_ids` → empty feed; failing hydrators → stale data) are only visible via
  telemetry - monitoring is load-bearing.
- Internal inconsistencies (dual action-index conventions, dual "final score"
  definitions, dead code) indicate a system mid-migration; the published artifacts won't
  reproduce production behavior without the withheld config.

---

## 10. Defects / Inconsistencies Worth Tracking

| Area | Issue |
|---|---|
| Phoenix | Two conflicting action-index conventions; two "final score" definitions |
| Phoenix | All params init to 0 → only valid with checkpoint; tests assert shapes only |
| Phoenix | Checkpoint-compat hacks (dummy negatives, unused `log_temperature`) |
| Home-mixer | Dead code (legacy scorers, `ImpressedPostsQueryHydrator` → inert backup filter) |
| Home-mixer | Gizmoduck 200ms timeout silently degrades viewer roles/subscription |
| Thunder | Panic-on-error ingest; debug-gated Strato fallback → silent empty feed |
| Thunder | Batch-log comment/code mismatch (100 vs 1000) |
| Grox | LLM-driven enforcement, version-sensitive; redacted thresholds/prompts |
| Grox | Order-sensitive bool-metadata selection across plans |
| Framework | Repeated full-candidate clones per stage (memory ∝ candidates × stages) |
