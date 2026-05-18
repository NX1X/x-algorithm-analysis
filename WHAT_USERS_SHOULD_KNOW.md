# What X Users Should Know About the "For You" Feed

📂 [Overview](README.md) · 🔧 [Technical](TECHNICAL_CONCLUSIONS.md) · 👤 **For Users** *(you are here)*

> Plain-language summary of how X's "For You" feed actually decides what you see,
> based on a full read of the open-sourced algorithm code. No code knowledge required.

---

## 1. The Feed Is Almost Entirely AI-Decided

X removed nearly all hand-written rules. What you see is decided by a large AI model
(a Grok-based transformer) that **learns from your behavior** - what you like, reply to,
repost, quote, click, watch, and how long you linger. There is no simple "chronological"
or "topical" rule behind it; the model infers what you want from your engagement history.

**Two pools of posts compete for your feed:**
- **In-network:** recent posts from accounts you follow (ranked, then re-scored by the AI).
- **Out-of-network:** posts from people you *don't* follow, surfaced by AI similarity
  search across a huge pool of content.

Both are scored by the same model and mixed together.

---

## 2. What the Algorithm Predicts About You

For every candidate post, the model predicts the probability that **you specifically**
will take ~19 different actions, including:

**Positive signals (push content up):**
like, reply, repost, quote, click, profile click, video view, photo expand,
share, share via DM, share via copy link, dwell (lingering), follow the author.

**Negative signals (push content down):**
mark "not interested", block author, mute author, **report**, scroll past without
dwelling.

> **Important:** The system actively predicts whether you'll *dislike* something. Content
> the model thinks you'd block, mute, or report is **demoted on purpose**. Using the
> "Not interested", mute, and block tools genuinely shapes your feed - they are real
> training signals, not cosmetic.

Your final feed order is a **weighted sum** of all these predictions. Likes, replies,
reposts, and dwell time carry the most visible weight.

---

## 3. What Data About You Is Used

The system pulls together a substantial amount of context per request:

- Your **engagement history** (the core personalization signal - a sequence of your past
  actions on posts and authors).
- Your **social graph**: who you follow, block, mute, subscribe to, and mutual-follow
  overlap with authors.
- **Topics and starter packs** you follow, and AI-**inferred topic interests**.
- Posts you've **already seen or were recently served** (to avoid repeats - tracked via an
  impression history and a probabilistic "Bloom filter").
- **Demographics**, **inferred gender** (for brand-new accounts a model predicts it),
  and **approximate location from your IP address**.
- Account age and follower count.

Much of this is gated behind internal feature flags, but the capability to use all of it
is present in the code.

---

## 4. What Gets Content Demoted or Hidden From You

Beyond the AI score, posts can be filtered or downranked because they are:

- Duplicates, too old, or your own posts.
- From accounts you blocked/muted, or authors who block you.
- Behind a paid subscription you don't have.
- Containing your **muted keywords**.
- Previously seen or already served to you.
- Flagged by **visibility/safety filtering** - content judged deleted, spam, violent,
  gore, or otherwise rule-violating is dropped, especially for out-of-network content
  (which is held to a *stricter* safety bar than posts from people you follow).

A separate Grok-LLM service (**"Grox"**) continuously reviews posts and labels them for
**quality, spam, and safety**. These AI judgments feed back into both ranking and
automated enforcement:
- Each post gets an AI **quality / "is this good content" score** and topic tags.
- Replies under large accounts get an AI **reply-quality score**; spammy replies from
  low-follower accounts under big posts are specifically targeted for demotion.
- Adult, violent, and unsafe content gets escalating AI scrutiny (multiple models). Safety
  labels are **"sticky"** - once flagged, hard to un-flag in that path. The system is
  deliberately biased toward over-suppressing unsafe content rather than under-suppressing.

> Note: these moderation judgments are made by an AI language model. They are
> near-deterministic but can shift between model versions, and they can directly affect
> a post's visibility and trigger automated enforcement labels.

---

## 5. Author Diversity - Why You Don't See One Person 10× in a Row

Even if one account would otherwise dominate your feed, the algorithm **deliberately
attenuates repeated posts from the same author** within a single feed load. The first post
from an author keeps full score; each additional one is progressively discounted. This is
an intentional anti-flooding mechanism for feed variety.

---

## 6. New Users Are Treated Differently

If your account is new or has little engagement history:
- You're routed to a **different AI model** tuned for users with short histories.
- Out-of-network (discovered) content gets a **different boost factor**, because you have
  few followed accounts to draw from.
- New users may get a topic-interest-based feed until enough behavior is collected.

So your early experience is **not** representative of the steady-state algorithm - the
feed changes meaningfully as you engage.

---

## 7. Ads

- Ads are injected into the feed by a separate blending step.
- In the default configuration, **no more than ~half of "safe" posts can be ads**, and
  **the feed never ends on an ad**.
- Ads have **brand-safety protections**: an ad won't be placed next to content judged
  risky, and advertisers can block their ads from appearing near specific accounts or
  keywords. Content that hasn't been safety-reviewed is treated as risky-for-ads by
  default (it can still appear organically - this rule is about ad adjacency).

---

## 8. Caching - Why Your Feed Sometimes Feels Slightly Stale

For speed, the system can cache a batch of pre-scored candidate posts for you for **about
3 minutes**. On a refresh within that window, it may re-rank cached posts rather than
recompute everything from scratch. This means very recent posts can occasionally be
missing for a short window in exchange for a faster feed.

---

## 9. Things to Be Aware Of (Honest Caveats)

- **The system is opaque.** Because relevance is learned by one big model rather than
  explicit rules, even operators cannot easily explain *why* a specific post ranked where
  it did. There is no human-readable "reason" attached to ranking.
- **AI moderates AI-ranked content.** An LLM both helps rank and helps suppress posts.
  Its judgments are powerful inputs to visibility.
- **Negative actions are weighted as well as positive ones.** Blocking/muting/reporting
  and even *not* dwelling on a post all teach the model.
- **This is a partial public snapshot.** Exact ranking weights and safety thresholds are
  **not disclosed** in the released code, model checkpoints are a frozen mini-version, and
  some logic is mid-migration. The public code shows *how* the system works, not the exact
  tuning values production uses.

---

## 10. Practical Takeaways for Users

| If you want to… | Do this |
|---|---|
| See more of a topic/person | Engage genuinely: like, reply, repost, dwell, follow |
| See less of something | Use **Not interested**, **Mute**, **Block**, mute keywords |
| Reduce repeats from one account | The algorithm already limits this per refresh |
| Get a better feed faster | Engage consistently - the model adapts to your behavior over time |
| Understand new-account feeds | Expect it to improve and change as you interact |
| Reduce tracked context | Following fewer accounts/topics and limiting engagement reduces signal - but also reduces personalization |

---

*This is an informational summary derived from source-code analysis only. It does not
disclose internal tuning values (which are not in the public code) and should not be read
as official X documentation.*
