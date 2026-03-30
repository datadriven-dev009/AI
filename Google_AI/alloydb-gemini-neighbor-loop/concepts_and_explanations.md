# Neighbor Loop — Concepts & Explanations
> **Codelab:** Real-Time Surplus Engine with Gemini 3 Flash & AlloyDB

---

## What It Builds
**Neighbor Loop** — a "swipe-to-match" surplus sharing web app where AI lives inside the database, not the application layer.

---

## Core Idea: In-Database Intelligence

Traditional approach → Pull data out → Process in app → Send back  
This approach → **AI runs inside SQL** → No round-trips → Faster + cheaper

| Layer | What It Does |
|-------|-------------|
| **AlloyDB AI** | Generates + stores vectors in real-time inside INSERT |
| **Gemini 3 Flash** | Sees item images + writes bios; runs "vibe checks" in SQL |
| **GCS** | Stores item images |
| **Cloud Run** | Hosts Flask backend |

---

## Key Features Explained

### 1. Multimodal Bio Generation (Gemini in App Layer)
User uploads photo + title → Gemini "sees" the image → auto-generates a witty dating-style bio + category. No manual description needed.

### 2. In-Database Embeddings
Vector is generated **inside the INSERT statement** using AlloyDB's `embedding()` function. Item is searchable by meaning the moment it's inserted — no Python roundtrip.

### 3. Semantic Search + AI Filter (`ai.if()`)
Two-layer search inside one SQL query:
- **Layer 1:** Vector cosine similarity (`<=>`) finds mathematically close items
- **Layer 2:** `ai.if()` calls Gemini inside SQL to do a "vibe check" — ensures results are contextually relevant, not just mathematically close

### 4. Swipe to Match
- Swipe Left → discard
- Swipe Right → records in `swipes` table, shows provider contact info

---

## Schema Overview

```
items table
  ├── item_id, owner_id, provider info
  ├── title, bio, category, image_url
  ├── item_vector VECTOR(768)   ← semantic search
  └── status (available/matched)

swipes table
  ├── swiper_id, item_id
  ├── direction (left/right)
  └── is_match BOOLEAN
```

---

## Connectivity — Local vs Cloud Run

| Scenario | Use |
|----------|-----|
| Local testing | **Public IP** + `0.0.0.0/0` in Authorized Networks |
| Cloud Run | **Private IP** + `--vpc-egress=all-traffic` |

⚠️ Always remove `0.0.0.0/0` after local testing.

---

## Key Gotchas

| Problem | Fix |
|---------|-----|
| SQL can't call Gemini | Grant `Vertex AI User` role to AlloyDB service agent |
| Extension fails | Wait for instance to finish provisioning |
| Connection refused | Check Public IP is enabled + IP whitelisted |
| Private IP from local | Use Public IP locally, Private IP on Cloud Run |
| Cloud Run 500 error | Check env vars + Cloud Run SA has `alloydb.client` role |
| Bucket name taken | Add random suffix to bucket name |
| IAM not propagated | Wait 60 seconds after IAM changes |
