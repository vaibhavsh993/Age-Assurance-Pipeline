# Application design (component responsibilities)
### 1. Client (browser / mobile)
**Responsibilities**:
- Collect minimal session signals (client-only): scroll depth, dwell time, click latencies, categories viewed. No PII.
- Optionally run lightweight TF-Lite / WASM model for immediate decisions in anonymous sessions (privacy-first).
- Call Edge API to fetch server-scored age_band and confidence.
- Render UI restrictions: blur thumbnails, replace with neutral placeholder, inline banner or family clusters.
**Security/Privacy**:
- All signals are hashed/aggregated. Client never sends raw identifiers.
- Provide user toggle for “I am an adult (verified)” when appropriate.

### 2. Edge API / Session Gateway
**Responsibilities**
- Accept client events; assemble ephemeral session features.
- Provide low-latency endpoint GET /session/{session_id}/age-assurance returning {band, p_adult, flags, ttl}.
- Cache results in Redis for TTL (e.g., 15–60 minutes).
- Forward events to Kafka for offline processing.
**Endpoints**
- POST /events — batched session events (click, view, query) (no PII).
- GET /session/{id}/score — fast score from Redis or call model service.
- POST /session/{id}/override — only for verified actions (logged-in override or appeal links).
**Auth**
- Edge API uses short-lived API keys between client and edge, mutual TLS for mobile apps.

### 3. Product Tagging Service
**Responsibilities**
- Classify products as adult_tag: true|false|uncertain.
- Hybrid pipeline: rule-based heuristics (title/description), text-model (DistilBERT), image classifier (explicitness).
- Flag high-risk products for manual review.
**Outputs**
- Product document update: {product_id, adult_tag, tag_confidence, flagged_timestamp, reviewer_id}
**Integration**
- On new listing / edit → Tagging service → update product catalog and searchable index (Elasticsearch).

### 4. Age-Assurance Model Service
**Responsibilities**
- Accept ephemeral session features and produce probability p_adult and explainability tokens (top features).
- Provide two modes: fast (lightweight model for server; low features) and full (ensemble).
**Model components**
- Text encoder (DistilBERT small) → text-prob
- Behavior model (XGBoost / LightGBM) → behavior-prob
- Fusion layer to combine results → final p_adult
- Calibration & monotonic constraints to bias toward safety (reduce false negatives).
**APIs**
- POST /predict → {session_id, features} returns {p_adult, band, reasons[], model_version}
- POST /batch_predict for bulk scoring (used in offline re-score).

### 5. Feature Pipeline & Storage
**Realtime**
- Kafka topic events for raw session events.
- Stream processors (Flink / Kafka Streams) aggregate session features into session_features topic and materialize into Redis (hot store) and S3 (cold store).
**Offline**
- Periodic job to build training datasets: join labeled users, session features, product tags → produce training tables in Parquet on S3/HDFS.

### 6. Training Platform
**Setup**
- Jobs run in k8s notebooks / batch. Provide DP-SGD or Federated pipelines where applicable.
- Artifacts stored in Model Registry (MLflow or internal).
- CI: validate fairness metrics, privacy constraints, and regression checks before promoting model.

### 7. Moderation Dashboard
**Responsibilities**
- Display flagged products, edge false-negatives, and disputed cases.
- Allow human annotators to change product tags and mark examples for retraining.

### 8. Monitoring & Experimentation
**Metrics**
- Safety recall (minor prevented / actual minor exposures), false positives, revenue metrics, conversion rate deltas, CS tickets related to "blocked" items.
- Per-model latency, throughput, and drift metrics.
**A/B**
- Integrated experiment framework: split sessions into control/variant. Store evaluation metrics and user feedback.
