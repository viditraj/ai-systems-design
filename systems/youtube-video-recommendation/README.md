# YouTube Video Recommendation System

> Real-world AI system design inspired by publicly documented YouTube recommendation research. Design a large-scale personalized video recommendation platform that retrieves candidate videos, ranks them with multi-objective ML, handles cold-start and freshness, learns from implicit feedback, and continuously improves through experimentation.

> **Scope note:** This design is an interview-oriented architecture inspired by public Google/YouTube research. It is not a claim about YouTube's current proprietary production implementation.

## Problem Statement

Design a recommendation system for a YouTube-scale video platform that can generate a personalized Home / Watch Next feed for each user.

The system should:

- Recommend relevant videos for each user.
- Handle billions of videos and a long-tail catalog.
- Support both logged-in and anonymous users.
- React quickly to recent user behavior.
- Surface fresh and newly uploaded content.
- Balance multiple objectives such as click probability, watch time, satisfaction, and quality/safety constraints.
- Avoid repeatedly recommending the same content.
- Handle new users and new videos with little or no interaction history.
- Support online experimentation and rapid model iteration.
- Keep serving latency low despite a very large candidate corpus.

## Public Research Inspiration

The design follows concepts publicly described in Google/YouTube research, including the two-stage **candidate generation + ranking** architecture, deep neural recommendation models, multitask ranking, selection-bias challenges, and large-scale neural retrieval. See the official research references linked at the end of this document.

## Core Design Principle

**Recommendation at internet scale is not one model. It is a multi-stage retrieval, ranking, filtering, and feedback system.**

The architecture separates:

```text
Candidate Generation
        ↓
Candidate Filtering
        ↓
Ranking
        ↓
Re-ranking / Constraints
        ↓
Serving
        ↓
User Feedback
        ↓
Training Data
        ↓
Model Updates
```

## Main Components

| Component | Responsibility |
|---|---|
| API Gateway | Authentication, rate limiting, request routing |
| Recommendation Service | Coordinates online recommendation request |
| Feature Store | Serves online/offline user, video, and contextual features |
| User Representation Service | Maintains recent behavior and learned user embeddings |
| Candidate Generators | Produce diverse candidate pools from different sources |
| ANN / Embedding Index | Fast semantic and collaborative candidate retrieval |
| Ranking Service | Predicts multiple user/item outcomes and produces ranking scores |
| Re-ranker | Applies diversity, freshness, policy, repetition, and business constraints |
| Video Metadata Store | Stores video metadata, creator information, categories, quality signals |
| Event Pipeline | Captures impressions, clicks, watches, skips, likes, dislikes, and other feedback |
| Training Pipeline | Builds datasets, trains models, validates models |
| Model Registry | Stores approved model versions and metadata |
| Experiment Platform | A/B tests models and ranking strategies |
| Safety / Policy Service | Removes or down-ranks content violating serving constraints |

## Request Flow

```text
User opens YouTube
        ↓
Recommendation API
        ↓
Load user/context features
        ↓
Candidate Generation
        ↓
Candidate Filtering
        ↓
Multi-task Ranking
        ↓
Re-ranking
        ↓
Diversity / Freshness / Safety
        ↓
Top-N recommendations
        ↓
User interaction
        ↓
Event stream
        ↓
Feature + Training pipelines
```

## Candidate Generation

A single retrieval strategy is insufficient. Use multiple candidate sources:

```text
                         User + Context
                              |
          +-------------------+-------------------+
          |                   |                   |
          v                   v                   v
   Collaborative       Embedding / ANN       Trending / Fresh
    Retrieval              Retrieval            Retrieval
          |                   |                   |
          +-------------------+-------------------+
                              |
                              v
                    Candidate Union / Dedup
                              |
                              v
                     Candidate Filtering
```

Candidate sources can include:

- user-history / co-watch retrieval
- learned user-to-video embedding retrieval
- similar-video retrieval
- subscriptions / creator affinity
- fresh uploads
- trending content
- regional or language candidates
- exploration candidates

The important insight is that **retrieval optimizes recall, not final ranking quality**.

## Ranking

The ranking model receives a much smaller candidate set and predicts multiple outcomes.

Possible predictions:

```text
P(click)
P(long_watch)
P(like)
P(skip)
P(session_continuation)
P(satisfaction)
```

A final utility score can combine these predictions:

```text
score =
    w1 * P(click)
  + w2 * P(long_watch)
  + w3 * P(like)
  - w4 * P(skip)
  + w5 * P(session_continuation)
```

The weights are product and policy decisions, not something the LLM decides.

## Cold Start

### New User

There may be no behavioral history.

Use:

- locale
- language
- device
- time of day
- coarse context
- trending content
- contextual exploration

### New Video

There may be no interaction history.

Use content signals such as:

- title / description
- creator
- category
- language
- audio/visual embeddings
- thumbnail / visual features
- semantic content representations

This allows content-based retrieval to surface new videos before sufficient collaborative data exists.

## Freshness and Exploration

A purely engagement-optimized model can become overly conservative and repeatedly recommend already popular content.

Introduce explicit exploration:

```text
                Final Feed
                    |
          +---------+---------+
          |                   |
       Exploit             Explore
     known-good          uncertain/new
          |                   |
          +---------+---------+
                    |
                    v
              Re-ranker
```

Exploration can be constrained by safety, quality, user preferences, and a traffic budget.

## Feedback Loop

Recommendations create the data used to train future recommendations.

```text
Recommendation
      ↓
Impression
      ↓
Click / Skip / Watch / Like / Dislike
      ↓
Event Stream
      ↓
Feature Store + Data Lake
      ↓
Training Dataset
      ↓
Model Training
      ↓
Offline Evaluation
      ↓
A/B Test
      ↓
Production
      ↓
More Feedback
```

This creates a feedback loop and introduces selection bias: the system only observes behavior for items it chose to show.

## Evaluation

Evaluate offline and online.

### Retrieval

- Recall@K
- candidate coverage
- catalog coverage
- fresh-content recall

### Ranking

- NDCG
- MRR
- log loss
- calibration
- watch-time prediction quality

### Product

- watch time
- satisfied sessions
- long-term retention
- creator ecosystem health
- diversity
- freshness

### Safety

- policy violation exposure
- unsafe recommendation rate
- complaint rate

Offline metrics are not sufficient. A recommendation model must be evaluated through controlled online experiments because user behavior changes when ranking changes.

## Key Concepts This Design Teaches

1. Two-stage retrieval + ranking.
2. Approximate nearest-neighbor retrieval.
3. User and item embeddings.
4. Multi-task learning.
5. Learning-to-rank.
6. Implicit-feedback modeling.
7. Selection bias.
8. Exploration vs exploitation.
9. Cold-start handling.
10. Feature stores and online/offline feature parity.
11. Re-ranking and constrained optimization.
12. A/B testing and online evaluation.
13. Feedback loops and distribution shift.
14. Calibration and score combination.
15. Training-serving skew and freshness.

## Detailed Architecture

See [architecture.md](./architecture.md) for the production architecture, ML pipeline, feature system, candidate generation, ranking, re-ranking, serving, estimation, reliability, and interview trade-offs.

## Mock Interview

See [mock-interview.md](./mock-interview.md) for a complete Senior AI Engineer interview roleplay with interviewer challenges and candidate answers.

## Official Public References

- Google Research — **Deep Neural Networks for YouTube Recommendations** (2016): https://research.google/pubs/deep-neural-networks-for-youtube-recommendations/
- Google Research — **Recommending What Video to Watch Next: A Multitask Ranking System** (2019): https://research.google/pubs/recommending-what-video-to-watch-next-a-multitask-ranking-system/
- Google Research — **Improving Training Stability for Multitask Ranking Models in Recommender Systems** (2023): https://research.google/pubs/improving-training-stability-for-multitask-ranking-models-in-recommender-systems/
- Google Research — **The YouTube Video Recommendation System** (2010): https://research.google/pubs/the-youtube-video-recommendation-system/
