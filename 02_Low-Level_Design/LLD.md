# Low-level design (LLD): interfaces, schemas, configs, and infra

### 1. Key APIs (spec)
**Edge API**
- POST /events
- Body: {session_id: string, events: [{type, timestamp, metadata}], client_meta: {device_class, hw_class}}
- Response: 200 OK
GET /session/{session_id}/score
- Response: {session_id, p_adult: float, band: "adult|uncertain|minor", model_version, ttl_seconds, reasons:[{feature, weight}]}

**Model Service**
- POST /predict
- Body: {session_id, features: {text_queries: [...], click_counts: {...}, time_of_day: int, product_context: [product_ids]}}
- Response: {p_adult, band, reasons, model_version}

**Product Tagging**
- POST /product/{product_id}/tag
- Body: {title, description, seller_tags, image_ids}
- Response: {adult_tag, tag_confidence, reasons, flagged_for_review: bool}

### 2. Data Schemas
- SessionFeatures (Redis hash) — TTL 30m
 Key: session:{session_id}
 Value fields:
 ``` 
{
  session_id,
  last_active_ts,
  query_ngrams: ["phone", "sex", ...] (hashed),
  categories_viewed: {fashion:3, electronics:1},
  products_viewed: [p1,p2,...] (most recent 20),
  avg_dwell_ms,
  click_latency_ms,
  scroll_depth_pct,
  device_class: "mobile"|"desktop",
  referrer_class: "social"|"organic",
  p_adult_cached,   // optional
  model_version_cached,
  cached_at_ts
}
```

**ProductDocument (search index)**
```
{
  product_id,
  title,
  description,
  seller_id_hash,
  adult_tag: true|false|uncertain,
  adult_tag_confidence: 0-1,
  image_score: {nudity:0.0, explicit:0.0},
  category_tags: [...],
  last_tagged_ts
}
```
### 3. Feature engineering (LLD)
**Text features**
- Query embeddings: DistilBERT outputs (vector 128) — compute on-device or server.
- Token features: presence of high-risk n-grams hashed with Bloom/hash.

**Behavior features**
- category_counts (last N minutes), product_view_ratio (adult-tagged / all), avg_dwell, click_latency, queries_per_minute.
- Temporal: session_start_hour (bucketed), session_duration.

**Product-context features**
- top_k viewed product adult_tag fraction (e.g., ratio of viewed items that are adult-tagged).

**Normalization**
- z-score scaled per-cohort; store scaling params in model metadata.

### 4. Model LLD
- Behavior model: LightGBM with monotonic constraints where beneficial (e.g., more adult-tagged product views → increases p_adult monotonically). Save as ONNX for serving.
- Text model: DistilBERT distilled & quantized, exported as TF or ONNX; produce embedding then small MLP to map to prob.
- Fusion: small MLP with inputs {behavior_score, text_score, image_flags} → final output. Calibrate with Platt scaling or Isotonic.
**Explainability**
- Return top-3 contributing features (feature importance by SHAP-like approximation). Keep explanation small to avoid PII disclosure.

### 5. Decision Logic (LLD)
Psuedo-code:
```
def decide_render(p_adult, product_adult_flag, product_confidence, config):
    if product_adult_flag == True and product_confidence >= config.product_hard_threshold:
        # product-level override -> always hide for minors
        if p_adult < config.adult_threshold:
            return "hide"
        else:
            return "show"
    if p_adult >= config.high_adult:
        return "show"
    if p_adult <= config.minor:
        return "hide_or_blur"   # full hide from feed
    if config.minimize_friction:
        return "soft_restrict"  # blur thumbnails + family alternatives
    return "soft_restrict"
```

### 6. Security & privacy design
- PII: never collect raw names/emails; hashed IDs only.
- Retention: session events TTL 30 days for offline training; ephemeral session cache TTL 15–60 minutes; configurable.
- Differential privacy: training pipelines support DP-SGD knobs; maintain epsilon logs per training job.
- Federated option: on-device updates aggregated via secure aggregation if enabled.
- Audit logs: store model_version + decision traces for each session for a fixed retention (e.g., 90 days) for compliance. Access to logs requires approval.
