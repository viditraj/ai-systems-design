# Mock Interview Walkthrough — YouTube Video Recommendation System

> A realistic Senior AI Engineer system-design interview simulation focused on recommender systems, retrieval, embeddings, ranking, multi-task learning, feedback loops, cold start, exploration, experimentation, and production ML.

## Interview Setup

**Role:** Senior AI Engineer — Machine Learning / Recommendation Systems

**Duration:** 60–75 minutes

**Focus:** Large-scale ML serving, candidate generation, ranking, embeddings, feature stores, implicit feedback, selection bias, multi-objective optimization, online experimentation, and model lifecycle.

> **Public-research note:** The scenario is inspired by public YouTube/Google research. The candidate architecture is an educational design, not a claim about YouTube's current proprietary system.

---

## 1. Problem Statement

### Interviewer

Design a personalized recommendation system for a YouTube-scale video platform.

The system should:

- Recommend relevant videos for each user.
- Handle a huge video catalog.
- React to recent user behavior.
- Surface new videos quickly.
- Work for new users and new videos.
- Balance click-through, watch time, satisfaction, freshness, diversity, and safety.
- Support online experimentation.

Walk me through the architecture.

### Candidate

Before I choose a model, I want to understand the scale, latency requirements, and the business objective because recommendation architecture depends heavily on those constraints.

---

## 2. Requirement Clarification

### Candidate

**1. How many users and videos are we designing for?**

### Interviewer

Assume 500 million daily active users and a catalog of roughly 1 billion videos.

**2. How many recommendation requests?**

### Interviewer

Around 10 recommendation requests per active user per day, with a 5x peak over average traffic.

### Candidate

That gives:

```text
500M × 10 = 5B requests/day

5B / 86,400 ≈ 57.9K requests/sec average

5x peak ≈ 290K requests/sec
```

So this is a very high-throughput online ML serving system.

**3. What's the latency target?**

### Interviewer

Let's target roughly 100 ms for recommendation generation.

### Candidate

That strongly suggests a multi-stage architecture rather than running an expensive model over the entire catalog.

**4. What is the primary objective?**

### Interviewer

We want engagement, but we don't want clickbait or repetitive content. User satisfaction and long-term retention matter too.

### Candidate

Then I would design this as a multi-objective ranking problem rather than simply optimizing CTR.

---

## 3. First Architecture

### Candidate

My initial architecture is:

```text
USER
  |
  v
Recommendation API
  |
  v
Feature Retrieval
  |
  v
Candidate Generation
  |
  v
Candidate Filtering
  |
  v
Ranking Model
  |
  v
Re-ranker
  |
  v
Top-N Recommendations
  |
  v
USER ACTIONS
  |
  v
Event Stream
  |
  +----> Feature Store
  |
  +----> Training Data
  |
  v
Model Training
  |
  v
Model Registry
  |
  v
A/B Testing
  |
  v
Production
```

The most important decision is the separation between candidate generation and ranking.

---

## 4. Interviewer Challenge — Why Two Stages?

### Interviewer

Why can't we just build one neural network that scores all 1 billion videos?

### Candidate

The compute would be enormous.

Suppose we had 290K requests/sec at peak.

Scoring 1 billion videos per request would imply an impossible number of model evaluations.

Instead:

```text
1B videos
   |
   v
Candidate Retrieval
   |
   v
~1,000 candidates
   |
   v
Expensive Ranker
   |
   v
~100 candidates
   |
   v
Re-ranker
   |
   v
20–50 items
```

Candidate generation optimizes recall and latency.

Ranking optimizes precision and user utility.

This is a fundamental information-retrieval pattern and is also reflected in public YouTube recommendation research.

---

## 5. Candidate Generation

### Interviewer

How do you generate those 1,000 candidates?

### Candidate

I wouldn't depend on one retrieval strategy.

I'd combine multiple candidate generators:

```text
                         User
                          |
        +-----------------+-----------------+
        |                 |                 |
        v                 v                 v
   Collaborative      ANN / Embedding   Fresh / Trending
      Retrieval          Retrieval          Retrieval
        |                 |                 |
        +-----------------+-----------------+
                          |
                          v
                     Candidate Union
```

Additional sources could include:

- subscriptions
- similar videos
- regional content
- language-specific content
- creator affinity
- exploration candidates

The union improves recall and makes the system more robust to cold-start and popularity bias.

---

## 6. Interviewer Challenge — Why Vector Search?

### Interviewer

Why do you need embeddings?

### Candidate

Embeddings allow us to represent users and videos in a shared latent space.

For example:

```text
User Encoder
     |
     v
User Embedding
     |
     v
ANN Search
     |
     v
Relevant Video Embeddings
```

This can capture semantic or behavioral relationships that exact metadata matching cannot.

For example, a user who watches several distributed-systems videos might retrieve other technically related videos even if the titles don't share exact keywords.

At large scale, approximate nearest-neighbor indexing makes this practical.

---

## 7. Interviewer Challenge — New Video

### Interviewer

A creator uploads a video 30 seconds ago. It has zero views. How can we recommend it?

### Candidate

That's a cold-start problem.

Collaborative signals don't exist yet, so I'd create a content-based candidate path.

```text
New Video
   |
   +--> Title / Description Embedding
   |
   +--> Audio Features
   |
   +--> Visual / Frame Features
   |
   +--> Creator Features
   |
   v
Content Representation
   |
   v
Fresh Candidate Index
```

We can retrieve the video for users whose interests match its content even before it accumulates behavioral data.

This is also useful for long-tail content.

---

## 8. Interviewer Challenge — New User

### Interviewer

What about a brand-new user with no history?

### Candidate

I'd fall back progressively:

```text
No history
   |
   +--> Locale
   +--> Language
   +--> Device
   +--> Time/context
   +--> Regional trends
   +--> Safe popular content
   +--> Exploration
```

After the first few interactions, the session itself becomes a powerful personalization signal.

So the system should rapidly transition from contextual recommendations to personalized recommendations.

---

## 9. Ranking Model

### Interviewer

Now we have 1,000 candidates. What does the ranking model do?

### Candidate

It predicts multiple user outcomes.

For example:

```text
P(click)
P(long_watch)
P(like)
P(skip)
P(session_continue)
P(satisfaction)
```

I'd use a multitask architecture:

```text
                Shared Representation
                        |
          +-------------+-------------+
          |             |             |
          v             v             v
       Click Head   Watch Head     Like Head
          |             |             |
          +-------------+-------------+
                        |
                        v
                 Utility Function
                        |
                        v
                      Score
```

This is better than optimizing only CTR because clicks don't necessarily mean users had a good experience.

---

## 10. Interviewer Challenge — Clickbait

### Interviewer

Suppose a clickbait video has a 30% click probability but users leave after 5 seconds. Another video has 15% CTR but users watch for 15 minutes. Which should rank higher?

### Candidate

The second video may be more valuable depending on the product objective.

I'd explicitly model multiple outcomes.

For example:

```text
score =
    w1 * P(click)
  + w2 * P(long_watch)
  + w3 * P(like)
  + w4 * P(session_continue)
  - w5 * P(skip)
```

The exact weights should be validated experimentally.

The important point is that the ranking model predicts outcomes and a downstream utility layer defines the product objective.

---

## 11. Interviewer Challenge — Is the Score Calibrated?

### Interviewer

Can you directly add all those probabilities together?

### Candidate

Not blindly.

Different prediction heads can have different calibration characteristics.

I'd monitor:

- reliability curves
- expected calibration error
- Brier score

If the outputs are combined into a utility function, calibration becomes important because a poorly calibrated task can dominate the final score.

---

## 12. Re-ranking

### Interviewer

The top 20 individually highest-scoring videos are all about the same topic and from the same creator. Is that okay?

### Candidate

No.

The ranker scores items independently, but the feed is a list.

I'd introduce a re-ranking layer:

```text
Ranked Candidates
      |
      v
Diversity
      |
      v
Creator Diversity
      |
      v
Freshness
      |
      v
Repetition Penalty
      |
      v
Exploration
      |
      v
Top-N
```

This allows us to optimize list-level properties.

---

## 13. Exploration vs Exploitation

### Interviewer

Why would you ever show a video the model isn't confident about?

### Candidate

Because recommendation systems have a feedback loop.

If we only show what we already know works:

```text
Known Content
   -> More Exposure
   -> More Data
   -> More Confidence
   -> More Exposure
```

New content may never get enough exposure to prove itself.

I'd allocate a controlled exploration budget:

```text
Feed
 |
 +--> Exploitation
 |
 +--> Exploration
```

Exploration candidates still pass safety and quality constraints.

This is where bandit-style thinking becomes useful.

---

## 14. Selection Bias

### Interviewer

Explain the statistical problem with training on clicks.

### Candidate

The system only observes feedback for items it chose to show.

If video B was never displayed, we cannot interpret:

```text
no click on B
```

as:

```text
user dislikes B
```

The recommendation system creates its own training distribution:

```text
Model
  |
  v
Exposure
  |
  v
Observed Behavior
  |
  v
Training Data
  |
  v
Model
```

This is selection bias and creates feedback loops.

We can mitigate it with controlled exploration, randomized exposure buckets, careful negative sampling, and where appropriate propensity-aware or counterfactual methods.

---

## 15. Interviewer Challenge — Negative Sampling

### Interviewer

A user didn't watch a video. Is that a negative label?

### Candidate

Not necessarily.

The critical distinction is:

```text
Not watched
    !=
Disliked
```

If the user was never exposed to the video, there is no direct preference signal.

For training, I would distinguish:

- explicit negatives
- impressions with skips
- impressions with no action
- positive interactions
- never-exposed items

The data pipeline must preserve exposure information.

---

## 16. Interviewer Challenge — Feature Store

### Interviewer

What features does your model use?

### Candidate

I would separate features by time horizon.

```text
Long-term:
  preferred categories
  creator affinity
  language

Medium-term:
  recent watch statistics
  topic distribution

Short-term:
  last N watched videos
  recent skips
  current session

Context:
  device
  locale
  time
  network
```

These features must exist consistently offline and online.

Otherwise we get training-serving skew.

---

## 17. Interviewer Challenge — Training-Serving Skew

### Interviewer

What exactly do you mean by training-serving skew?

### Candidate

Suppose training computes:

```text
watch_count_last_7_days
```

using one SQL definition, while online serving calculates it using another implementation.

The model sees a different feature distribution at inference time.

I'd therefore use shared feature definitions, feature validation, and parity tests between offline and online computation.

---

## 18. Event Pipeline

### Interviewer

What events do you collect?

### Candidate

Not only clicks.

I'd capture:

```text
impression
click
watch duration
skip
like
dislike
share
comment
session continuation
session end
```

Each impression should record enough information to reconstruct the recommendation decision:

```json
{
  "request_id": "R123",
  "user_id": "U456",
  "video_id": "V789",
  "position": 3,
  "model_version": "ranker-42",
  "candidate_source": "embedding",
  "timestamp": "..."
}
```

That enables offline replay, debugging, and experiment analysis.

---

## 19. Interviewer Challenge — 290K RPS

### Interviewer

We now have roughly 290K recommendation requests/sec at peak. How do you scale?

### Candidate

I'd make the serving path horizontally scalable and separate expensive stages.

```text
Load Balancer
      |
      +---- Recommendation Workers
      |
      +---- Candidate Retrieval Fleet
      |
      +---- Ranking Fleet
      |
      +---- Re-ranking Fleet
```

The ANN index should be distributed/sharded.

Feature serving should be horizontally scalable.

Ranking inference should support batching where latency permits.

The system should have multiple candidate fallbacks so one retrieval dependency doesn't make the feed empty.

---

## 20. Latency Budget

### Interviewer

You have a 100 ms budget. How do you divide it?

### Candidate

A rough target could be:

```text
Feature lookup       5–15 ms
Candidate retrieval 10–30 ms
Filtering             5–10 ms
Ranking              10–30 ms
Re-ranking            2–10 ms
Other/network        10–20 ms
-----------------------------
Total                <100 ms
```

These are design targets, not fixed production numbers.

The biggest optimization is architectural: don't run the expensive ranker against the entire catalog.

---

## 21. Interviewer Challenge — ANN Goes Down

### Interviewer

Your vector retrieval service is unavailable. Do you return no recommendations?

### Candidate

No.

Recommendation systems should degrade gracefully.

I'd use fallback retrieval:

```text
ANN unavailable
     |
     +--> Collaborative retrieval
     |
     +--> Trending
     |
     +--> Fresh regional content
     |
     +--> Safe popular baseline
```

If the safety service is unavailable, however, I would fail closed for restricted content rather than bypass safety checks.

---

## 22. Model Rollout

### Interviewer

You trained a new ranker that improves offline NDCG by 8%. Do you deploy it globally?

### Candidate

No.

I'd use:

```text
Offline validation
       |
       v
Shadow traffic
       |
       v
Small canary
       |
       v
A/B experiment
       |
       v
Progressive rollout
```

I'd track both primary metrics and guardrails.

For example:

```text
Primary:
  satisfied watch time

Guardrails:
  skip rate
  session abandonment
  safety exposure
  diversity
  latency
```

Offline improvement doesn't guarantee online improvement.

---

## 23. Interviewer Challenge — Offline vs Online

### Interviewer

Why could NDCG improve while the product gets worse?

### Candidate

Because offline evaluation uses a fixed historical dataset.

Changing the ranking changes user behavior and therefore changes the data distribution.

The model might improve relevance to historical labels but:

- reduce exploration
- increase repetition
- concentrate traffic on a small number of creators
- increase clickbait
- hurt long-term satisfaction

That's why online controlled experiments are necessary.

---

## 24. Model Training

### Interviewer

Walk me through your training pipeline.

### Candidate

```text
Event Stream
     |
     v
Data Lake
     |
     v
Sessionization
     |
     v
Label Generation
     |
     v
Feature Join
     |
     v
Training Dataset
     |
     +--> Retrieval Model
     +--> Ranking Model
     +--> Content Model
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
```

I'd version the dataset, features, code, model artifact, and evaluation results together so a production model can be reproduced.

---

## 25. Interviewer Challenge — Training Instability

### Interviewer

Your multitask ranking model's loss diverges during training. What do you investigate?

### Candidate

I'd inspect:

```text
Per-task losses
Gradient norms
Learning rate
Task weighting
NaN / Inf values
Feature distributions
Label distributions
Prediction distributions
```

One task may dominate the shared representation or the optimization may become unstable as model capacity increases.

Possible interventions include task weighting, gradient clipping, learning-rate tuning, normalization, and changes to shared/task-specific architecture.

This is an important production ML concern: a sophisticated ranking model is useless if the training pipeline isn't stable.

---

## 26. Interviewer Challenge — Privacy

### Interviewer

This system processes detailed user behavior. How do you handle privacy?

### Candidate

I'd apply data minimization and strict access controls.

Separate identity data from model features where possible.

Apply:

- encryption
- retention limits
- access control
- audit logging
- privacy-aware analytics
- controlled training-data access

We should not expose raw behavioral data to every engineering or modeling component unnecessarily.

---

## 27. Final Architecture Summary

### Candidate

My final architecture is:

```text
                              USER
                                |
                                v
                       Recommendation API
                                |
                                v
                       Online Feature Store
                                |
                                v
                     Candidate Generation
                                |
             +------------------+------------------+
             |                  |                  |
             v                  v                  v
       Collaborative        Embedding          Fresh/Trend
        Retrieval             ANN              Retrieval
             |                  |                  |
             +------------------+------------------+
                                |
                                v
                         Candidate Union
                                |
                                v
                       Safety / Filtering
                                |
                                v
                       Multi-task Ranker
                                |
                                v
                         Re-ranker
                    /      |       |      \
             Diversity  Freshness  Explore  Repeat
                                |
                                v
                              TOP-N
                                |
                                v
                             USER
                                |
                                v
                          Event Stream
                                |
                 +--------------+--------------+
                 |                             |
                 v                             v
            Feature Store                 Data Lake
                                                 |
                                                 v
                                           ML Training
                                                 |
                                                 v
                                           Model Registry
                                                 |
                                                 v
                                           A/B Platform
                                                 |
                                                 +----> Production
```

The key principle is:

> **At large scale, recommendation is a continuously learning retrieval-and-ranking system, not a single neural network.**

---

## 28. Interviewer Final Assessment

### Interviewer

This is a strong Senior AI Engineer answer because the candidate demonstrated understanding beyond basic LLM applications.

Strengths:

- correctly separated retrieval and ranking
- understood ANN and embeddings
- handled cold-start users and items
- designed multitask ranking
- recognized selection bias
- distinguished exposure from negative feedback
- addressed exploration vs exploitation
- included re-ranking and list-level constraints
- considered feature-store architecture
- addressed training-serving skew
- included online experimentation
- considered training stability
- reasoned about extreme serving scale

### Concepts an interviewer can probe next

1. Design the user/video embedding training objective.
2. How would you construct positive and negative samples?
3. How would you prevent popularity bias?
4. How would you estimate propensity for an impression?
5. Design an ANN index for 1 billion videos.
6. How would you update embeddings without rebuilding the entire index?
7. How would you handle model versioning during an A/B test?
8. How would you perform counterfactual evaluation?
9. How would you detect feedback-loop collapse?
10. How would you optimize GPU inference for the ranking model?
11. How would you design a multimodal video embedding pipeline?
12. How would you measure long-term user satisfaction rather than immediate clicks?
