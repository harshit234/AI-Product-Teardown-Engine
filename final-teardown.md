Zomato — Recommendation System (6-Layer Technical Teardown
Layer 1 — Data Foundation

What is happening: Zomato ingests multi-modal telemetry (searches, clicks, impressions, orders, cart events), restaurant metadata (menu, cuisine tags, opening hours), logistics signals (delivery ETA, availability), and third-party context (weather, local events). Events stream into real-time and batch pipelines where they’re validated, deduplicated, sessionized, and materialized as offline training tables and online feature slices.
Key technologies: Kafka / Kinesis (events), Spark / Flink (ETL & enrichment), S3 / HDFS (data lake), Parquet + Delta Lake, Feast or in-house Feature Store, Airflow for orchestration.
Engineering challenge: Keeping train/serve feature parity while ingesting geo-distributed low-latency events and handling data skew across cities and peak meal windows.
Skill required: Data engineering (stream & batch), schema evolution, feature engineering for recommendation.
Honesty check: Core and heavy — cannot be minimal; foundational to any personalization stack.

Layer 2 — Statistics & Analysis

What is happening: Product metrics (CTR, conversion, order value, delivery success) are instrumented per cohort; experiments (A/B and multi-arm bandit tests) evaluate ranking tweaks, business rules, and incentives. Offline analysis detects biases (popularity, locality), computes uplift, and segments users by recency/frequency/monetary (RFM) and latent taste clusters.
Key technologies: Presto / BigQuery / Redshift (analytics), Python (pandas, StatsModels), in-house experimentation platform or Optimizely, OLAP cube tooling.
Engineering challenge: Running causal A/B tests in a two-sided marketplace where treatment affects supply (restaurant load) and violates SUTVA — requiring guardrails and user/merchant-level randomization.
Skill required: Experiment design, causal inference, cohort analysis, SQL at scale.
Honesty check: Essential — thin layer would limit safe model iteration.

Layer 3 — Machine Learning Models

What is happening: A multi-stage pipeline: (A) Candidate generation via embedding-based recall (two-tower neural models or approximate collaborative filtering) producing ~100–1000 candidates; (B) Ranking using a supervised Learning-to-Rank model (gradient boosting like XGBoost/LightGBM or LambdaMART, or DNN rankers) combining user/item/context features; (C) Re-ranking with business constraints (promotions, delivery SLA, restaurant capacity) and exploration controllers (contextual bandits). Models incorporate real-time features (current ETA, kitchen load) and offline latent features.
Key technologies / models: Two-tower (Siamese) embeddings, matrix factorization (ALS) for cold start, Faiss / ScaNN for ANN recall, XGBoost / LightGBM or TensorFlow/PyTorch DNN rankers, contextual bandits (e.g., Vowpal Wabbit or custom).
Engineering challenge: Managing cold start + data sparsity across long-tail restaurants while preventing popularity bias; additionally, computing fresh features for ranking under strict latency.
Skill required: Recommender systems (embedding design, LTR), offline/online evaluation, production ML engineering.
Honesty check: Core layer — not applicable to omit.

Layer 4 — LLM / Generative AI

What is happening: LLMs are likely used at the periphery: natural-language query parsing (convert “best paneer near me” → structured filters or embedding query), review summarization, and conversational discovery assistants. They are not the primary ranking model but improve recall and UX.
Key technologies: SentenceTransformers / BERT for embeddings, OpenAI / Anthropic (likely) or self-hosted Llama2/FlanT5 for summarization; RAG patterns for long review context.
Engineering challenge: Integrating LLM outputs deterministically into ranking (ensuring legal/factual accuracy and low latency) and avoiding hallucinations when mapping free text to filters.
Skill required: NLP engineering, embedding search, prompt design, RAG architecture.
Honesty check: Auxiliary — important for search/UX but thin relative to core ranking.

Layer 5 — Deployment & Infrastructure

What is happening: Models and feature services are packaged as containerized microservices (Kubernetes). Real-time inference stack serves candidate recall and ranking via low-latency gRPC/HTTP APIs; batch jobs regenerate embeddings and retrain models offline. Feature-serving tiers (Redis / DynamoDB) supply hot features; model versions and feature schemas are tracked in CI/CD pipelines.
Key technologies: Kubernetes, Docker, TensorFlow Serving / TorchServe / BentoML, Redis / DynamoDB (online features), gRPC, Prometheus + Grafana, CI/CD (Jenkins/GitLab).
Engineering challenge: Achieving sub-100ms tail latency for ranking across millions of requests while enabling safe model rollouts and rollback with canary experiments.
Skill required: MLOps, Kubernetes operations, model serving optimization, observability and SLO management.
Honesty check: Heavily used — cannot be minimal.

Layer 6 — System Design & Scale

What is happening: The system scales via geo-partitioning (regionally sharded services), caching (edge & application), and autoscaling to absorb meal-time spikes. Reliability uses retries, graceful degradation (fallback to popularity/distance), and feature/timeouts to avoid cascading failures. Data governance enforces privacy (PII masking), and throughput is tuned with horizontal sharding of candidate search indices.
Key technologies: CDN / edge caching, Envoy / Istio for traffic management, Cassandra / DynamoDB for distributed storage, Kafka for durability, autoscaling on AWS/GCP.
Engineering challenge: Maintaining freshness and consistency of ephemeral features (kitchen load, ETA) across distributed caches without violating latency SLOs; and preventing feedback loops that amplify popularity.
Skill required: Distributed systems design, capacity planning, reliability engineering (SRE), observability.
Honesty check: Central for production — not optional.

OVERALL ANALYSIS

Most critical layer: Layer 3 — Machine Learning Models.
Why: The ranking model directly determines user experience and revenue; it must synthesize personalization, context (ETA, availability), and business objectives (promos, margin) under latency and fairness constraints. Candidate quality drives downstream compute and UX.

Hardest engineering problem (product-specific):
Real-time personalized ranking with fresh, ephemeral features (delivery ETA, kitchen load, surge pricing) under strict tail-latency SLOs while avoiding feedback loops that prioritize already-popular restaurants.
This requires sub-100ms feature assembly, fast ANN search over embeddings, constrained optimization for business rules, and mechanisms (exploration policies + debiasing) to prevent popularity reinforcement.

Complexity rating: Advanced → Bleeding Edge (in parts).
Reasoning: Multi-stage recommender + contextual bandits + real-time freshness under marketplace dynamics pushes into advanced design; integrating LLMs for robust NL search and RAG elevates parts to bleeding edge when combined with low-latency constraints.

If rebuilding from scratch — first priority:
Implement robust data foundation (event logging + feature store) and a simple two-stage recommender (geographic + popularity recall → lightweight LTR ranker). Without reliable, consistent data and feature parity, iterative model improvements and safe experiments are impossible.
