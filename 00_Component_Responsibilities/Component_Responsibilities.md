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
