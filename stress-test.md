PhonePe — Fraud Detection: 6-Layer Technical Teardown
# Layer 1 — Data Foundation

What is happening
Streaming and batch collection of every touchpoint: payment/merchant events, card/UPI metadata, device & SDK signals (device fingerprint, OS, app version), network (IP, geo), KYC/customer profiles, historical chargebacks and investigator labels. Labels and human-investigation outcomes are joined back to transactions so supervised models can be trained and evaluated. Data lineage (who changed which label and when) and immutable audit logs are enforced for compliance and post-hoc investigations.

Key technologies (likely)
Apache Kafka / Amazon Kinesis (ingest), Debezium (CDC), Flink or Spark Structured Streaming (real-time ETL), S3 / Hudi / Delta Lake (raw + curated lakes), PostgreSQL / Cassandra (hot indexes), Hive/BigQuery / Snowflake (analytics), Parquet for storage.

Engineering challenge
Ensuring deterministic, low-latency joinability between offline labels and real-time events without label leakage — i.e., guaranteeing that features used online could only have been computed using data available at inference time (causal feature engineering across streaming windows).

Skill required
Senior Data Engineer — streaming ETL, CDC, schema evolution, data lineage, experience with Parquet/Hudi/Delta and Kafka/Flink at production scale.

Honesty check
Not minimal — this is a critical, heavy layer; thin implementation would break model validity.

# Layer 2 — Statistics & Analysis

What is happening
Exploratory and statistical pipelines compute risk aggregates (per-user/per-card/per-merchant rates), feature distributions, cohort analyses, uplift/backtesting, and drift detection. They quantify class imbalance, false positive/negative tradeoffs, and provide KPIs (precision@low-recall operating points, time-to-detect). Statistical tooling also supports manual investigator workflows (risk scoring thresholds, whitelisting).

Key technologies (likely)
Apache Spark / Presto / ClickHouse for large-scale aggregation; dbt and SQL for transformation; Pandas/R for ad-hoc analysis; Great Expectations / Deequ for tests; Jupyter + Airflow for analysis orchestration.

Engineering challenge
Producing stable, comparable metrics across offline/online environments — e.g., matching streaming aggregates to batch recomputations and ensuring metric continuity across schema changes and event-time vs processing-time semantics.

Skill required
Applied Statistician / Analytics Engineer — time-series statistics, hypothesis testing under selection bias, cohort analysis, and metrics instrumentation.

Honesty check
Not minimal — essential for monitoring, threshold tuning, and investigating model regressions.

# Layer 3 — Machine Learning Models

What is happening
Multi-strategy modeling: (a) high-precision tabular classifiers (gradient-boosted trees like XGBoost/LightGBM/CatBoost) for immediate scoring; (b) graph-based link analysis (graph embeddings + GNNs) to detect mule networks and ring fraud; (c) sequential behavior models (temporal transformers or RNNs) for session/behavior anomalies; (d) unsupervised models (autoencoders, isolation forests, density estimation) for zero-day anomaly detection. Models are ensembled with calibrated risk scores and a downstream rules engine for business logic.

Key technologies (likely)
XGBoost / LightGBM / CatBoost; PyTorch / TensorFlow for deep models; DGL or PyG for graph ML; scikit-learn for baseline models; MLflow or Sagemaker for experiment tracking.

Engineering challenge
Combining heterogeneous model outputs into a single, well-calibrated risk score under concept drift and adversarial input while preserving interpretability for investigators — particularly blending graph signals (sparse, relational) with dense tabular features in low-latency paths.

Skill required
ML Engineer / ML Researcher — expertise in GBDTs, GNNs, sequence models, calibration (Platt/Isotonic), and adversarial robustness for tabular/graph data.

Honesty check
Substantial — this is the core modeling layer; cannot be minimal.

# Layer 4 — LLM / Generative AI

What is happening
LLMs are used as adjuncts: parsing free-text (merchant notes, customer complaints), creating investigator summaries, generating human-readable explanations for flagged cases, and powering retrieval-augmented investigative assistants (querying embeddings of prior cases). They can also be used for synthetic data generation for rare fraud types (with heavy controls).

Key technologies (likely)
Embedding store + vector DB (FAISS / Milvus / Pinecone); a smaller private LLM or instruction-tuned transformer deployed via Hugging Face or an internal model; LangChain or custom orchestration for RAG; strict PII scrubbers and prompt-sanitization layers.

Engineering challenge
Controlling hallucination and PII leakage — ensuring generated summaries are factual and auditable, and that any synthetic data or explanations do not invent events or exposure of sensitive fields. Also integrating LLM latency and trust into investigator SLAs.

Skill required
Applied NLP / MLOps for LLMs — prompt engineering, RAG systems, embedding pipelines, and compliance-aware LLM deployment.

Honesty check
Often adjunct / thin — many fraud pipelines use LLMs for augmentation rather than real-time decisioning; may be minimal or optional depending on maturity.

# Layer 5 — Deployment & Infrastructure

What is happening
Models and rules are served in two paths: (a) real-time low-latency scoring service (sub-100ms end-to-end) for transaction blocking/soft-decline using feature-store online joins and model inference; (b) nearline/batch pipelines for retrospective scoring and training. CI/CD for models, canary rollouts, live A/B experiments, and automatic rollback are enforced. Observability spans model metrics, data drift, latency, and business KPIs.

Key technologies (likely)
Kubernetes (EKS/GKE/AKS) + Istio or Linkerd, feature store (Feast or in-house), TorchServe / TF-Serving / BentoML for model serving, Redis / Aerospike for caches, Prometheus + Grafana + ELK for monitoring/logging, ArgoCD/CircleCI/GitHub Actions for CI/CD.

Engineering challenge
Guaranteeing feature parity between offline training and online inference (exact same transformations, handling late-arriving events) while keeping tail latency low and ensuring deterministic scoring across multi-region deployments. Also, warm/cold cache behavior for cold-start users and burst traffic (festival days) is hard.

Skill required
Senior MLOps / Site Reliability Engineer — Kubernetes, observability, feature-store design, chaos testing, and secure CI/CD for models (signing, governance).

Honesty check
Not minimal — must be robust and production-grade; a brittle infra makes even accurate models unusable.

# Layer 6 — System Design & Scale

What is happening
End-to-end system design enforces scale (millions of transactions/day), multi-region redundancy, strict audit & compliance trails (KYC, RBI/Indian data rules), and governance workflows (model approvals, human-in-loop overrides, appeals). It also manages cost/latency tradeoffs (edge vs central scoring) and integrates feedback loops for label capture from chargebacks or manual investigations.

Key technologies (likely)
Cassandra / ScyllaDB for high-write time-series, Redis for session caches, Kafka for durable event backbone, Vault/KMS for secrets, IAM + logging stacks for audit trails. Service mesh and multi-AZ deployment strategies.

Engineering challenge
Designing feedback loops that close the detection → investigate → label cycle without introducing survivorship bias, while maintaining legal auditability and reproducibility (replayability of decisions for a past timestamp). Also handling attacker adaptation (adversarial arms race) at system scale.

Skill required
Principal Systems Architect / Distributed Systems Engineer — distributed databases, audit & compliance architecture, attacker-aware system design, cost engineering at scale.

Honesty check
Essential — this layer determines whether the solution can operate at PhonePe scale and meet regulatory constraints.

Overall Analysis

Most critical layer
Layer 1 — Data Foundation. For fraud detection the model can only be as good as the signals and labels you feed it. Poor lineage, missing event-time guarantees, or label leakage will produce models that break in production. In practice, getting deterministic, timestamped joins between streaming telemetry and human labels is the single greatest determinant of real-world efficacy.

Hardest engineering problem (product-specific)
Real-time feature consistency and adversary-resistant scoring at scale. Concretely: ensuring that the online scoring path uses exactly those features that would have been available at transaction time (no lookahead), while combining sparse relational graph features (which require multi-hop graph aggregation) with low-latency constraints (<100ms) and robust defences against adversarial evasion (mule networks, synthetic accounts). This demands novel engineering: incremental graph updates, approximate neighborhood embeddings, and carefully engineered caches that remain consistent under partition and heavy load.

Complexity rating
Advanced (toward Bleeding Edge) — justified because the system combines high-throughput streaming, graph ML and sequence models, adversarial robustness, low-latency guarantees, and strict compliance/auditability. Parts (graph ML at real-time scale, adversary adaptation) push into bleeding-edge territory; core stack remains industry-standard but integration is complex.

If rebuilding from scratch: first priority
Build a deterministic event ingestion + feature store + labeling workflow: reliable event timestamps, durable Kafka topics, schema evolution governance, a feature store that supports both online (low-latency) and offline (batch) joins, and an investigator UI that captures canonical labels and their provenance. This foundation lets you safely iterate on models and detect regressions without risking operational or legal fallout.
