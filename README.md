# X "For You" Algorithm - Analysis

📂 **Overview** · 🔧 [Technical](TECHNICAL_CONCLUSIONS.md) · 👤 [For Users](WHAT_USERS_SHOULD_KNOW.md)

An independent, code-level analysis of **X's open-sourced "For You" feed algorithm**.

X published the source for its recommendation system at
**[github.com/xai-org/x-algorithm](https://github.com/xai-org/x-algorithm)**. This repo
reads through **all of it** - the ML models, the Rust serving pipeline, the in-network
store, and the Grok-LLM content-understanding service - and turns it into clear
conclusions.

> Produced with [Claude Code](https://www.anthropic.com/claude-code) doing a full source
> read. **Unofficial** - not affiliated with or endorsed by X. Findings describe the
> *published* code; exact production tuning values are not in the open-source release.

---

## 📚 Where to go next

This page **is** the overview - read it top to bottom for the whole story in ~3 minutes.
Then branch by who you are:

| Document | For whom | What's inside |
|---|---|---|
| 👤 **[WHAT_USERS_SHOULD_KNOW.md](WHAT_USERS_SHOULD_KNOW.md)** | Non-technical X users | Plain English: what actually shapes your feed, what's tracked, ads, moderation, practical tips |
| 🔧 **[TECHNICAL_CONCLUSIONS.md](TECHNICAL_CONCLUSIONS.md)** | Engineers / ML | Code-level internals: the transformer, scoring, pipeline, per-component deep dive, defects table |

---

## 🧠 TL;DR

- The feed is almost **rule-free** - one Grok-based transformer learns relevance from your
  raw engagement history.
- It predicts **~19 actions** per post, including whether you'd **block, mute, or report**
  it. It optimizes *against being disliked*, not only for likes.
- The architectural keystone is **candidate isolation**: posts are scored independently of
  each other, making scores consistent and cacheable.
- A second Grok LLM (**Grox**) reviews posts for quality/spam/safety, and those AI
  judgments feed back into both **ranking and enforcement**.
- This is a **partial snapshot**: weights, safety thresholds, and the full model are not
  in the public release.

---

## 🏗 The System in One Look

It's a classic two-stage **retrieval → ranking** recommender. The distinctive choice:
**almost all hand-engineered features were removed** - relevance is learned end-to-end by
a Grok-derived transformer from your raw engagement sequence.

Five cooperating parts:

| Component | Language | Role |
|---|---|---|
| **`phoenix/`** | Python / JAX | The ML brain: two-tower retrieval + transformer ranker (ported from Grok-1) |
| **`home-mixer/`** | Rust | Orchestration: builds the feed - sources, hydration, filters, scoring, ads |
| **`thunder/`** | Rust | In-memory store of recent in-network posts (sub-ms lookup) |
| **`candidate-pipeline/`** | Rust | Reusable, observable pipeline framework under home-mixer |
| **`grox/`** | Python | Grok-LLM content understanding: safety, spam, quality, embeddings |

**Data flow:** a request enters `home-mixer` → it hydrates your context → pulls
in-network posts from `thunder` and out-of-network posts from `phoenix` retrieval → scores
everything with the `phoenix` ranker → applies weighted multi-action scoring + diversity +
safety filtering → blends in ads → returns a ranked feed. Separately, `grox` continuously
annotates posts with Grok-generated quality/safety/spam signals that **feed back** into
ranking and enforcement.

---

## ⚙️ How It Actually Works

**1. Retrieval (millions → hundreds).** A two-tower model: the *user tower* runs the full
Grok transformer over your history into one embedding; the *candidate tower* is
deliberately transformer-free so the whole corpus can be precomputed. Retrieval is cosine
similarity. In-network posts skip ML entirely - `thunder` serves recent followed-account
posts ranked **purely by recency**.

**2. Ranking (hundreds → feed).** A transformer with **candidate isolation**: each
candidate post can attend to your history but **never to other candidate posts**. So all
candidates are scored in one parallel pass, mutually independent - consistent and
cacheable. The model predicts **~19 engagement actions** at once (positive: like, reply,
repost, dwell…; **negative: not-interested, block, mute, report**). Final score = a
**weighted sum** of these probabilities; negative actions carry negative weights, actively
demoting content you'd likely dislike. All weights are runtime-tunable.

**3. Shaping.** Author diversity geometrically attenuates repeat authors; out-of-network
content gets a tunable multiplier (different for new users and topics); new users are
routed to a dedicated model; an optional value-model re-rank adds DPP diversity.

**Notable design choices:** right-anchored positional encoding (removes positional bias
between candidates), hash-based embeddings (fixed memory), and a signed action encoding
that distinguishes "action absent" from "padding."

---

## ⚖️ What Stands Out vs What's Concerning

**Strengths**
- Architecturally elegant: clean two-stage recsys, composable observable pipeline, and
  candidate isolation enabling both caching and parallelism.
- Strong latency engineering (Thunder's compact in-memory store, candidate cache,
  deadlines, load-shedding).
- Safety/monetization defaults are conservative and fail-closed.
- Negative-feedback modeling actively protects the user experience.

**Risks**
- Total dependence on an opaque transformer with no interpretable feature attribution.
- LLM-generated content verdicts (Grox) directly drive visibility and enforcement -
  non-deterministic across model versions; single hardcoded thresholds gate promotion.
- Silent degradation paths (timeouts → default data; missing inputs → empty feed) are
  visible only via telemetry - monitoring is load-bearing.
- Internal inconsistencies and dead code indicate a system mid-migration; the public
  artifacts won't reproduce production behavior without the withheld config.

→ Full per-component depth, scoring internals, and a defects table:
**[TECHNICAL_CONCLUSIONS.md](TECHNICAL_CONCLUSIONS.md)**.
→ What this means for you as a user: **[WHAT_USERS_SHOULD_KNOW.md](WHAT_USERS_SHOULD_KNOW.md)**.

---

## 🗂 Repository Layout

```
.
├── README.md                    # you are here - hub + overview
├── TECHNICAL_CONCLUSIONS.md     # engineer-facing deep dive
├── WHAT_USERS_SHOULD_KNOW.md    # user-facing explainer
├── LICENSE                      # MIT - covers the analysis docs above
└── upstream/                    # X's original code, UNMODIFIED
    ├── README.md                # X's original project README
    ├── LICENSE                  # Apache-2.0 - covers everything in upstream/
    ├── CODE_OF_CONDUCT.md       # X's original
    ├── phoenix/                 # ML: retrieval + ranking (Python/JAX)
    │   └── artifacts/           # NOT included - large binary (>2 GiB), see note below
    ├── home-mixer/              # orchestration / serving (Rust)
    ├── thunder/                 # in-network post store (Rust)
    ├── candidate-pipeline/      # reusable pipeline framework (Rust)
    └── grox/                    # Grok-LLM content understanding (Python)
```

Everything under [`upstream/`](upstream/) is X's original source, kept **unmodified** for
faithful reference and diffing. All original analysis lives at the repository root.

> **⚠️ Excluded artifact:** `upstream/phoenix/artifacts/oss-phoenix-artifacts.zip` (the
> pretrained Phoenix model artifacts, **>2 GiB**) is **not bundled** in this repository -
> it exceeds GitHub's hard 2 GiB per-file limit. Download it from X's official release:
> **[xai-org/x-algorithm → phoenix/artifacts/oss-phoenix-artifacts.zip](https://github.com/xai-org/x-algorithm/blob/main/phoenix/artifacts/oss-phoenix-artifacts.zip)**.
> Place it at `upstream/phoenix/artifacts/` locally if you need to run the model.

---

## 📌 Licensing & Attribution

This repository is **dual-licensed by directory** - read this before reusing anything:

| Path | Content | License |
|---|---|---|
| Repository root (`*.md`) | The independent analysis & docs | **MIT** - see [LICENSE](LICENSE) |
| [`upstream/`](upstream/) | X's open-sourced "For You" algorithm | **Apache-2.0** - see [`upstream/LICENSE`](upstream/LICENSE) |

- The analysis docs are independent commentary; reuse them freely under MIT.
- The `upstream/` code belongs to X under its original Apache-2.0 terms, mirrored
  unmodified from the official release at
  **[github.com/xai-org/x-algorithm](https://github.com/xai-org/x-algorithm)**. The
  Phoenix transformer is itself ported from xAI's
  [Grok-1 release](https://github.com/xai-org/grok-1). See
  [`upstream/README.md`](upstream/README.md) for X's original documentation.

---

<sub>Maintainer note: a local `docs-internal/` directory holds private working drafts
(social copy, notes, archives). It is git-ignored and intentionally **not** part of the
published analysis.</sub>
