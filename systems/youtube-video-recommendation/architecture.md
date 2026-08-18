# Detailed Architecture — YouTube Video Recommendation System

## 1. Architecture Goal

Design a large-scale personalized recommendation platform inspired by public YouTube/Google research.

The key architectural idea is to avoid scoring the entire catalog with one expensive model. Instead:

```text
User Request
    ↓
Feature Retrieval
    ↓
Candidate Generation
    ↓
Candidate Filtering
    ↓
Multi-task Ranking
    ↓
Re-ranking / Constraints
    ↓
Top-N Feed
```

This design is intentionally interview-oriented. YouTube's actual production system is proprietary and evolves continuously.

---

## 2. Requirements and Assumptions

Assume an interview asks us to design recommendations for a global video platform.

### Functional

- Personalized Home and Watch Next recommendations.
- Logged-in and anonymous users.
- Millions/billions of videos.
- Fresh uploads should become discoverable quickly.
- Recommendations should adapt to recent user behavior.
- Avoid repetitive recommendations.
- Support language, region, device, and contextual personalization.
- Balance engagement with quality, diversity, freshness, and safety.
- Support controlled experimentation.

### Non-functional

For interview estimation, assume:

```text
500M daily active users
10 recommendation requests/user/day
5B recommendation requests/day
```

Average request rate:

```text
5B / 86,400 ≈ 57,870 requests/sec
```

At a 5x peak:

```text
≈ 290K requests/sec
```

The exact numbers are assumptions. The important part is deriving the scale and showing which components must scale independently.

---

## 3. High-Level Architecture

```text
                                     USER
                                       |
                                       v
                              ┌──────────────────┐
                              │ Recommendation   │
                              │ API / Gateway    │
                              └────────┬─────────┘
                                       |
                                       v
                              ┌──────────────────┐
                              │ Recommendation   │
                              │ Service          │
                              └────────┬─────────┘
                                       |
                        ┌──────────────┼──────────────┐
                        |              |              |
                        v              v              v
                 User Features   Context Features  Session State
                        |              |              |
                        └──────────────┼──────────────┘
                                       v
                              ┌──────────────────┐
                              │ Candidate        │
                              │ Generation      │
                              └────────┬─────────┘
                                       |
             ┌─────────────────────────┼──────────────────────────┐
             |                         |                          |
             v                         v                          v
      ┌─────────────┐          ┌─────────────┐           ┌─────────────┐
      │ Collaborative│          │ ANN / Vector│           │ Fresh /     │
      │ Retrieval    │          │ Retrieval   │           │ Trending    │
      └──────┬──────┘          └──────┬──────┘           └──────┬──────┘
             |                         |                         |
             └─────────────────────────┼─────────────────────────┘
                                       v
                              ┌──────────────────┐
                              │ Candidate Union  │
                              │ + Dedup          │
                              └────────┬─────────┘
                                       |
                                       v
                              ┌──────────────────┐
                              │ Safety / Policy  │
                              │ Filtering        │
                              └────────┬─────────┘
                                       |
                                       v
                              ┌──────────────────┐
                              │ Multi-task       │
                              │ Ranking Model    │
                              └────────┬─────────┘
                                       |
                                       v
                              ┌──────────────────┐
                              │ Re-ranker        │
                              │ Diversity        │
                              │ Freshness        │
                              │ Exploration      │
                              └────────┬─────────┘
                                       |
                                       v
                                  Top-N Feed
                                       |
                                       v
                                  USER ACTION
                                       |
                                       v
                              ┌──────────────────┐
                              │ Event Streaming  │
                              └────────┬─────────┘
                                       |
                    ┌──────────────────┼────────────────────┐
                    v                  v                    v
              Feature Store       Data Lake             Monitoring
                    |                  |
                    v                  v
              Online Serving     Training Pipeline
                                       |
                                       v
                                  Model Registry
                                       |
                                       v
                                 A/B Platform
                                       |
                                       +----> Production
```

---

## 4. Why Two-Stage Retrieval + Ranking?

Suppose the catalog contains:

```text
1 billion videos
```

Running a large deep ranking model against every video is infeasible.

Instead:

```text
1B videos
   |
   v
Candidate Generation
   |
   v
~1,000 candidates
   |
   v
Expensive Ranking Model
   |
   v
~100 candidates
   |
   v
Re-ranking
   |
   v
20–50 recommendations
```

Candidate generation prioritizes recall and speed.

Ranking prioritizes precision and user utility.

This separation also lets us add multiple candidate sources without redesigning the ranking model.

Public YouTube research explicitly describes a two-stage candidate generation and ranking architecture. The public paper should be treated as conceptual inspiration rather than a current production specification.

---

## 5. User Representation

A user's representation should combine multiple time scales.

```text
User Representation
        |
        +--> Long-term interests
        |
        +--> Recent watch history
        |
        +--> Current session
        |
        +--> Explicit preferences
        |
        +--> Context
               |
               +--> locale
               +--> device
               +--> time
               +--> network
```

A simple representation could concatenate learned embeddings:

```text
user_embedding = f(
    long_term_embedding,
    recent_sequence_embedding,
    session_embedding,
    context_features
)
```

For sequence modeling, recent video IDs can be embedded and processed with an attention/transformer-style encoder or another sequential model.

The architecture should not assume that a single static user vector is sufficient.

---

## 6. Candidate Generation

Use multiple retrieval channels.

### 6.1 Collaborative / Co-watch Retrieval

Users who watched A also watched B.

```text
Recent history
     ↓
Co-watch / collaborative index
     ↓
Candidate videos
```

This is strong for established content but weak for new or rarely watched videos.

### 6.2 Learned Embedding Retrieval

Train a model so that relevant users/videos are close in embedding space.

```text
User Encoder                    Video Encoder
     |                               |
     v                               v
User Vector                    Video Vector
     |                               |
     +---------- ANN Search ---------+
                    |
                    v
               Candidates
```

An approximate nearest-neighbor index makes large-scale vector retrieval practical.

### 6.3 Fresh Content Retrieval

A separate path retrieves recently uploaded content:

```text
Fresh uploads
    ↓
Quality / safety filters
    ↓
Language / region filters
    ↓
Fresh candidate pool
```

This prevents popularity-based retrieval from starving new creators.

### 6.4 Trending / Regional Retrieval

Use region and time-window aware popularity signals.

### 6.5 Exploration Retrieval

Intentionally retrieve uncertain candidates to learn whether the user likes them.

---

## 7. Candidate Union

Candidate sources may overlap heavily.

```text
Collaborative       500
Embedding           500
Fresh               200
Trending            100
Subscriptions       100
Exploration         100
                    ---
Raw candidates     1500
```

Deduplicate by video ID and preserve source provenance:

```json
{
  "video_id": "V123",
  "sources": ["embedding", "fresh"],
  "retrieval_scores": {
    "embedding": 0.91,
    "fresh": 0.42
  }
}
```

Source provenance can become a ranking feature.

---

## 8. Filtering Before Ranking

Some constraints should be deterministic.

Examples:

- unavailable videos
- blocked content
- policy-restricted content
- region restrictions
- age restrictions
- already watched / recently shown content
- duplicate videos
- user-blocked creators

```text
Candidates
    ↓
Availability
    ↓
Safety
    ↓
Region / Age
    ↓
Repetition
    ↓
Ranking
```

This avoids wasting expensive ranking compute on candidates that can never be served.

---

## 9. Ranking Model

The ranking model predicts multiple outcomes rather than a single click score.

Example outputs:

```text
P(click)
P(long_watch)
P(like)
P(skip)
P(session_continue)
P(satisfaction)
```

A multi-task architecture can share representations while using task-specific heads.

```text
                   Shared Features
                         |
                         v
                 Shared Representation
                         |
          +--------------+--------------+
          |              |              |
          v              v              v
      Click Head     Watch Head     Like Head
          |              |              |
          +--------------+--------------+
                         |
                         v
                  Utility Combiner
                         |
                         v
                       Score
```

Public YouTube research has described multitask ranking approaches for large-scale video recommendation.

---

## 10. Why Optimize More Than Clicks?

If we optimize only:

```text
P(click)
```

we can select clickbait.

A user might click but immediately leave.

Instead we can model multiple outcomes:

```text
utility =
    w_click * P(click)
  + w_watch * P(long_watch)
  + w_like * P(like)
  + w_session * P(session_continue)
  - w_skip * P(skip)
```

Weights can be tuned through experimentation and product objectives.

The important system-design point is that **ranking becomes a multi-objective optimization problem**.

---

## 11. Calibration

Model scores are not automatically comparable across tasks or candidate sources.

For example:

```text
P(click) = 0.20
P(long_watch) = 0.10
```

Those probabilities may have different calibration properties.

We should monitor calibration and, where appropriate, calibrate predictions before combining them.

Useful metrics include:

- reliability diagrams
- expected calibration error
- Brier score

Calibration becomes particularly important when scores feed a downstream utility function.

---

## 12. Re-ranking

The ranking model optimizes individual item utility, but the feed is a sequence.

A top-20 list containing 20 nearly identical videos is poor even if every individual item scores highly.

Re-ranking can optimize list-level properties:

```text
                    Ranked Candidates
                           |
                           v
                   Diversity Constraint
                           |
                           v
                   Freshness Constraint
                           |
                           v
                   Creator Diversity
                           |
                           v
                  Exploration Budget
                           |
                           v
                         Top-N
```

A simple greedy re-ranker can use marginal utility:

```text
final_score(video) =
    relevance_score
  + diversity_bonus
  + freshness_bonus
  + exploration_bonus
  - repetition_penalty
```

More advanced systems can formulate list construction as constrained optimization.

---

## 13. Exploration vs Exploitation

Pure exploitation repeatedly chooses the safest known recommendations.

Pure exploration harms immediate relevance.

Use a controlled exploration budget.

```text
Candidate Pool
      |
      +---- 90% exploitation
      |
      +---- 10% exploration
                    |
                    v
              uncertain/new
              candidates
```

The exact ratio should be experimentally determined.

Exploration should still obey:

- safety
- quality
- user preferences
- region
- age restrictions

The system can use bandit-style ideas, uncertainty estimates, or simpler randomized candidate quotas depending on complexity and maturity.

---

## 14. Cold Start

### 14.1 New User

No history means collaborative retrieval is weak.

Use:

```text
Locale
Language
Device
Time
Context
Trending
Editorial / safe popular pool
Exploration
```

After a few interactions, session behavior can quickly personalize recommendations.

### 14.2 New Video

No interaction history means collaborative signals are weak.

Use content understanding:

```text
Title / Description
       |
       v
Text Embedding

Video Frames / Audio
       |
       v
Multimodal Embedding
       |
       v
ANN / Content Retrieval
```

This enables discovery of fresh content before sufficient behavioral data accumulates.

---

## 15. Feature Store

We need both offline and online features.

### Offline

Used for training:

```text
Data Lake
   ↓
Feature Generation
   ↓
Training Dataset
```

### Online

Used during recommendation serving:

```text
User request
   ↓
Online Feature Store
   ↓
Recent user features
```

A key production concern is **training-serving skew**.

If training computes:

```text
watch_count_last_7_days
```

but serving computes it differently, model behavior can degrade.

Use shared feature definitions and validation between offline and online pipelines.

---

## 16. Event Pipeline

Every impression should be captured, not only clicks.

```text
Recommendation Response
       |
       v
Impression Event
       |
       +--> click
       +--> watch duration
       +--> skip
       +--> like
       +--> dislike
       +--> share
       +--> session end
```

An event should include enough context to reconstruct the recommendation decision:

```json
{
  "request_id": "R123",
  "user_id": "U456",
  "video_id": "V789",
  "model_version": "ranker-42",
  "position": 3,
  "candidate_source": "embedding",
  "timestamp": "2026-08-18T10:00:00Z"
}
```

This is critical for debugging and offline replay.

---

## 17. Selection Bias

This is one of the most important concepts in recommendation systems.

Suppose the model only shows:

```text
Popular Video A
```

Users click it often.

We cannot conclude that A is intrinsically better than Video B if B was rarely shown.

The model controls what gets observed.

```text
Model
  ↓
Exposure
  ↓
User Behavior
  ↓
Training Data
  ↓
Model
```

This creates a feedback loop and selection bias.

Mitigations include:

- exploration traffic
- randomized exposure buckets
- propensity-aware methods
- counterfactual evaluation where appropriate
- careful negative sampling

The interview takeaway is:

> **Observed engagement is not the same as unbiased preference.**

---

## 18. Negative Sampling

Most videos were not watched by a user, but that does not mean the user disliked them.

They may never have seen them.

Therefore:

```text
Not watched
   !=
Disliked
```

Training data construction must distinguish:

- positive interactions
- explicit negative signals
- impressions with no action
- never-exposed items

This is a major difference between recommendation ML and ordinary supervised classification.

---

## 19. Training Pipeline

```text
Event Stream
     |
     v
Data Lake
     |
     v
Sessionization / Label Generation
     |
     v
Feature Join
     |
     v
Training Dataset
     |
     +----> Candidate Retrieval Model
     |
     +----> Ranking Model
     |
     +----> Content Embedding Model
     |
     v
Offline Evaluation
     |
     v
Model Registry
     |
     v
Shadow / Canary
     |
     v
A/B Test
     |
     v
Production
```

Training should be versioned by:

- code
- dataset snapshot
- feature definitions
- model hyperparameters
- model artifact
- evaluation results

---

## 20. Model Training Stability

Large multitask ranking models can become unstable.

Monitor:

```text
loss curves
per-task loss
gradient norms
NaN / Inf
prediction distributions
label distribution
feature distribution
```

If one task dominates the shared representation, another task can degrade.

Possible controls include:

- task weighting
- gradient clipping
- learning-rate schedules
- normalization
- careful initialization
- task-specific towers
- monitoring training dynamics

Public YouTube/Google research has specifically discussed training stability challenges in large multitask ranking systems.

---

## 21. Online Serving Latency

A possible latency budget:

```text
Feature lookup          5–15 ms
Candidate retrieval    10–30 ms
Filtering               5–10 ms
Ranking                10–30 ms
Re-ranking              2–10 ms
Network / overhead     10–20 ms
-------------------------------
Target                 <100 ms
```

The exact budget depends on product requirements.

The key design decision is to keep expensive model inference limited to a small candidate set.

---

## 22. Caching

Caching can exist at multiple layers:

```text
User feature cache
Candidate cache
Video feature cache
Embedding cache
Model artifact cache
```

Be careful with user-specific recommendation caches because recent behavior can invalidate them.

A useful pattern is short TTL caching combined with event-driven invalidation for high-value changes.

---

## 23. Reliability

| Failure | Handling |
|---|---|
| Feature store unavailable | Use recent cached/default features |
| ANN retrieval unavailable | Fall back to collaborative/trending retrieval |
| Ranking model unavailable | Use previous model / simpler ranker |
| Safety service unavailable | Fail closed for restricted content |
| Event pipeline delayed | Continue serving; reconcile later |
| Model deployment failure | Roll back to last healthy model |
| Feature drift | Alert and route traffic to safe baseline |

Recommendation systems should degrade gracefully rather than returning an empty feed.

---

## 24. Model Rollout

Never replace a production recommender globally without controlled validation.

Use:

```text
Candidate Model
      |
      v
Offline Validation
      |
      v
Shadow Traffic
      |
      v
Small Canary
      |
      v
A/B Test
      |
      v
Progressive Rollout
```

Keep a known-good baseline available for rollback.

---

## 25. A/B Testing

Offline improvement does not guarantee product improvement.

A model can improve:

```text
NDCG +5%
```

while harming:

```text
Long-term retention
Creator diversity
Satisfied sessions
```

Online experiments should track multiple guardrail metrics.

Example:

```text
Primary:
  satisfied watch time

Guardrails:
  skip rate
  session abandonment
  safety violations
  diversity
  creator concentration
  latency
```

Use statistically valid experiment assignment and avoid repeatedly changing experiment populations mid-test.

---

## 26. Multi-Objective Ranking

A production recommender often optimizes multiple stakeholders:

```text
User satisfaction
       +
Creator ecosystem
       +
Content diversity
       +
Freshness
       +
Safety
       +
Business objectives
```

The architecture should make these objectives explicit rather than hiding them inside an opaque scalar.

One approach:

```text
Prediction Heads
      ↓
Utility Model
      ↓
Constraint Layer
      ↓
Final List
```

This also makes product trade-offs easier to inspect and experiment with.

---

## 27. Monitoring and Evaluation

### Retrieval

- Recall@K
- candidate coverage
- source contribution
- fresh-content recall
- cold-start recall

### Ranking

- NDCG
- MRR
- log loss
- calibration
- per-task AUC where appropriate

### Online Product

- watch time
- satisfied sessions
- long-term retention
- skip rate
- session continuation
- diversity
- creator concentration
- freshness

### Infrastructure

- p50/p95/p99 latency
- CPU/GPU utilization
- ANN latency
- feature-store latency
- ranking throughput
- error rate

### Model Health

- feature drift
- label drift
- prediction drift
- embedding drift
- training/serving skew
- task loss instability

---

## 28. Security and Privacy

Recommendation systems process behavioral data that can be highly sensitive.

Apply:

- strict access control
- data minimization
- retention policies
- encryption
- audit logging
- privacy-aware analytics
- separation of identity data from model features where possible

The model-training pipeline should not expose raw personal data unnecessarily.

---

## 29. Back-of-the-Envelope Compute Thinking

Assume:

```text
290K requests/sec peak
1,000 candidates/request
```

If the ranking model scored every candidate:

```text
290K * 1,000
≈ 290M candidate scores/sec
```

This illustrates why ranking models must be highly optimized and why candidate generation must aggressively reduce the search space.

If candidate generation returns 200 candidates for an expensive ranker:

```text
290K * 200
≈ 58M scores/sec
```

Further batching, model optimization, caching, and horizontal scaling are required.

The interview goal is not to claim exact infrastructure capacity but to demonstrate why architecture choices are necessary.

---

## 30. Important Trade-offs

### Candidate recall vs latency

More candidates improve the chance of finding good items but increase ranking cost.

### Personalized vs fresh

Strong personalization can suppress new content. Fresh candidate channels are needed.

### Engagement vs satisfaction

Click optimization can produce clickbait. Multi-objective ranking is safer.

### Exploitation vs exploration

Pure exploitation creates feedback loops. Exploration provides information but can reduce immediate metrics.

### Model complexity vs serving cost

A larger model may improve quality but can violate latency and infrastructure budgets.

### Offline metrics vs online metrics

Offline metrics accelerate iteration but cannot replace controlled online experiments.

### Centralized ranker vs multiple rankers

A single ranker is simpler; specialized surfaces can justify separate models or objectives later.

---

## 31. Interview Takeaways

1. **Do not rank the entire catalog with one expensive model.**
2. **Separate candidate generation from ranking.**
3. **Use multiple candidate generators to improve recall and freshness.**
4. **Embeddings enable scalable semantic/collaborative retrieval.**
5. **Multi-task learning can model several user outcomes simultaneously.**
6. **Ranking is multi-objective, not just CTR optimization.**
7. **Re-ranking operates on the list, not just individual items.**
8. **Cold start requires content and contextual signals.**
9. **Recommendation feedback is biased because the model controls exposure.**
10. **Exploration helps break feedback loops and discover uncertain content.**
11. **Training-serving feature parity is a major production concern.**
12. **Offline metrics must be complemented by online A/B tests.**
13. **Calibration matters when combining multiple model outputs.**
14. **Model training stability matters as ranking systems become multitask and large.**
15. **A recommendation system is a continuously learning feedback loop, not a static model.**

## 32. Public Research References

- Paul Covington, Jay Adams, Emre Sargin — **Deep Neural Networks for YouTube Recommendations**, RecSys 2016.
- Aditee Ajit Kumthekar et al. — **Recommending What Video to Watch Next: A Multitask Ranking System**, RecSys 2019.
- Daryl Chang et al. — **Improving Training Stability for Multitask Ranking Models in Recommender Systems**, KDD 2023.
- James Davidson et al. — **The YouTube Video Recommendation System**, RecSys 2010.

These papers are useful background for the interview concepts, but this repository's architecture is an educational synthesis rather than a reproduction of YouTube's proprietary implementation.
