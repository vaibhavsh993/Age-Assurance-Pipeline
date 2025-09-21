# Age-Assurance-Pipeline

Compact, production-ready architecture that implements our proposed multimodal, privacy-first age-assurance pipeline. This document covers components, data flows, APIs, schemas, infra, monitoring, rollout and operational details.

## Goals & Constraints
- Primary goal: Prevent minors (coarse bucket: <18) from seeing 18+ content for anonymous and logged-out sessions while preserving conversion and minimizing friction.
- Accuracy constraint: Minimize false negatives (minor sees adult). Prefer conservative actions (soft restrict / hide) over false positives.
- Privacy requirement: Do not persist PII; use DP/federated options for training; short retention TTL for session-level data.
- Operational: Must be auditable, explainable and region-compliant.

## High-level System Overview
**Components**:
- Client (browser/mobile) — lightweight on-device scoring or session feature collection + UI rendering hooks.
- Edge API / Session Gateway — session aggregation, ephemeral storage, quick server-side inference.
- Product Tagging Service — product-level adult classifier (title/desc/image) + metadata writer.
- Age-Assurance Model Service — text encoder, behavior model, ensemble fusion for server-side inference; supports TF-Serving / TorchServe / ONNX.
- Feature Pipeline — event collector (Kafka), stream processor for offline features, and feature store (Redis / Feast-like).
- Training Platform — controlled labeled dataset, DP/federated training jobs, CI for model retraining.
- Moderation Dashboard & Human-in-the-loop — tooling to surface edge cases and manually label items/listings.
- Monitoring, Logging & A/B-Experimentation — metrics, alerts, drift detection and experiment tracking.

**High-level data-flow**:
- Client events → Edge Gateway → (fast path) Session Score (Redis cache / client inference) → Content rendering decisions
- Product ingestion → Product Tagger → Product Catalog write
- Events to Kafka → Offline feature store → Training jobs → New models deployed to Model Service
