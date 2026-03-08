# Recommendation System -- Cheat Sheet

## Key Numbers
- 500M MAU, 200M DAU, 1B items in catalog
- 70K QPS peak, < 200 ms latency target
- 256-dim embeddings: 500 GB user embeddings, 1 TB item embeddings
- ~1000 candidates generated, narrowed to 200 then 50

## Core Components
- Two-Tower Model: separate user and item embedding networks trained on interaction data
- ANN Index (HNSW): sub-millisecond nearest neighbor search over 1B item embeddings
- Multi-Source Candidate Generation: ANN + collaborative filtering + content-based + trending
- Feature Store (Redis): serves real-time and batch features at sub-ms latency
- Two-Stage Ranker: lightweight scorer (1000->200) then deep model (200->50)
- Re-Ranker: diversity, freshness, content policy filtering

## Architecture in One Sentence
Multi-source candidate generation retrieves ~1000 items via ANN embedding search, collaborative filtering, and trending signals; a two-stage ranking pipeline scores candidates using features from a real-time feature store; and a re-ranker applies diversity and policy constraints -- all within 200 ms.

## Top 3 Trade-offs
1. Two-tower + ANN vs. full cross-attention retrieval: two-tower enables sub-ms retrieval at billion scale but sacrifices some interaction modeling.
2. Multi-source retrieval vs. single embedding: multiple sources improve coverage and handle cold start at the cost of merging complexity.
3. Daily batch retraining vs. online learning: batch is simpler and more stable; online learning adapts faster to trends but risks instability.

## Scaling Story
10x: shard ANN index, add feature store replicas, distill ranking model for Stage 1.
100x: multi-region deployment, pre-computed candidate sets for returning users, custom accelerators for ranking, hierarchical ANN retrieval.

## Closing Statement
A multi-source candidate generation system with two-tower ANN retrieval, a two-stage deep ranking pipeline backed by a real-time feature store, and diversity-aware re-ranking delivers personalized recommendations for 500M users across 1B items within 200 ms.
