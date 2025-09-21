# High-level design (HLD & data flows)

### 1 HLD Sequence 
- Sequence for a search page load (server-side scoring):
- Client loads search page → sends GET /search?q=... with session_id cookie.
- Frontend requests GET /session/{session_id}/score from Edge API.
- Edge checks Redis for cached {band, p_adult, model_version}. If present, return.
- If not present, Edge collects recent session aggregates from Redis/feature store, then calls POST /predict on Model Service.
- Model Service computes p_adult (fast mode), returns result.
- Edge caches the result and returns to frontend.
- Frontend queries Product Catalog / Search index (ES) with additional filtering: exclude items with product.adult_tag == true if band == minor or p_adult <= low_threshold and apply soft-blur if band == uncertain.
- Page rendered accordingly. Client continues to send events to Edge API which get published to Kafka for offline features.

### 2 HLD Notes
- Product-level classification is authoritative for content tagging; user model only affects visibility decisions.
- Decision logic (mapping p_adult → UI action) is configuration-driven and lives in a centralized Feature Flag / Config store for quick A/B changes.
- Use consistent TTLs and cache-busting semantics for sessions to allow re-evaluation on user action (e.g., login or verification).
