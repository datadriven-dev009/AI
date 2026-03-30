# Commands & Code — Neighbor Loop (Gemini 3 Flash + AlloyDB)

---

## 1. GCP Setup

```bash
gcloud auth list
gcloud config list project
gcloud config set project <YOUR_PROJECT_ID>
```

---

## 2. AlloyDB Setup (One-Click Script)

```bash
git clone https://github.com/GoogleCloudPlatform/devrel-demos.git
cd devrel-demos/infrastructure/easy-alloydb-setup
sh run.sh
# Enter Project ID, Cluster name, Instance name in the web UI
```

Enable Public IP for local testing:
- AlloyDB Console → Instance → EDIT → Enable Public IP → Add `0.0.0.0/0` to Authorized Networks

---

## 3. Schema — AlloyDB Studio SQL Editor

**Enable extensions:**
```sql
CREATE EXTENSION IF NOT EXISTS google_ml_integration CASCADE;
CREATE EXTENSION IF NOT EXISTS vector;
```

**Create tables:**
```sql
CREATE TABLE items (
   item_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
   owner_id UUID,
   provider_name TEXT,
   provider_phone TEXT,
   title TEXT,
   bio TEXT,
   category TEXT,
   image_url TEXT,
   item_vector VECTOR(768),
   status TEXT DEFAULT 'available',
   created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE swipes (
   swipe_id SERIAL PRIMARY KEY,
   swiper_id UUID,
   item_id UUID REFERENCES items(item_id),
   direction TEXT CHECK (direction IN ('left', 'right')),
   is_match BOOLEAN DEFAULT FALSE,
   created_at TIMESTAMP DEFAULT NOW()
);
```

**Grant permissions:**
```sql
GRANT EXECUTE ON FUNCTION embedding TO postgres;
```

**Grant Vertex AI User role to AlloyDB service account:**
```bash
PROJECT_ID=$(gcloud config get-value project)
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:service-$(gcloud projects describe $PROJECT_ID \
  --format='value(projectNumber)')@gcp-sa-alloydb.iam.gserviceaccount.com" \
  --role="roles/aiplatform.user"
```

**Register Gemini 3 Flash model in AlloyDB:**
```sql
CALL google_ml.create_model(
   model_id => 'gemini-3-flash-preview',
   model_request_url => 'https://aiplatform.googleapis.com/v1/projects/<<YOUR_PROJECT_ID>>/locations/global/publishers/google/models/gemini-3-flash-preview:generateContent',
   model_qualified_name => 'gemini-3-flash-preview',
   model_provider => 'google',
   model_type => 'llm',
   model_auth_type => 'alloydb_service_agent_iam'
);
```

---

## 4. GCS Bucket Setup

```bash
# Create bucket
gcloud storage buckets create gs://<YOUR_BUCKET_NAME> --location=us-central1

# Make publicly readable (demo only)
gcloud storage buckets add-iam-policy-binding gs://<YOUR_BUCKET_NAME> \
  --member="allUsers" --role="roles/storage.objectViewer"

gcloud storage buckets add-iam-policy-binding gs://<YOUR_BUCKET_NAME> \
  --member="allUsers" --role="roles/storage.objectCreator"
```

---

## 5. Clone & Configure App

```bash
git clone https://github.com/AbiramiSukumaran/neighbor-loop
cd neighbor-loop
```

**Edit `.env` file:**
```env
GEMINI_API_KEY=<<YOUR_GEMINI_API_KEY>>
DATABASE_URL=postgresql+pg8000://postgres:<<YOUR_PASSWORD>>@<<PUBLIC_IP_HOST>>:5432/postgres
GCS_BUCKET_NAME=<<YOUR_GCS_BUCKET>>
```
> For local testing use **Public IP**. For Cloud Run use **Private IP**.

Get Gemini API Key: [aistudio.google.com](https://aistudio.google.com) → Get API key → Create API key

---

## 6. Key Code Snippets

**Bio generation (Gemini sees the image):**
```python
prompt = """
You are a witty community manager for NeighborLoop.
Analyze this surplus item and return JSON:
{
   "bio": "First-person witty dating-style profile bio, max 2 lines",
   "category": "One-word category",
   "tags": ["tag1", "tag2"]
}
"""
response = genai_client.models.generate_content(
   model="gemini-3-flash-preview",
   contents=[types.Part.from_bytes(data=image_bytes, mime_type="image/jpeg"), prompt],
   config=types.GenerateContentConfig(response_mime_type="application/json")
)
```

**In-database embedding on INSERT:**
```sql
INSERT INTO items (owner_id, provider_name, provider_phone, title, bio, category, image_url, status, item_vector)
VALUES (
   :owner, :name, :phone, :title, :bio, :cat, :url, 'available',
   embedding('text-embedding-005', :title || ' ' || :bio)::vector
)
```

**Semantic search with Gemini vibe check inside SQL:**
```sql
SELECT item_id, title, bio, category, image_url,
       1 - (item_vector <=> embedding('text-embedding-005', :query)::vector) AS score
FROM items
WHERE status = 'available'
  AND item_vector IS NOT NULL
  AND ai.if(
        prompt => 'Does this text: "' || bio || '" match the user request: "' || :query || '", at least 60%?',
        model_id => 'gemini-3-flash-preview'
      )
ORDER BY score DESC
LIMIT 5;
```

---

## 7. Deploy to Cloud Run

```bash
gcloud beta run deploy neighbor-loop \
   --source . \
   --region=us-central1 \
   --network=<<YOUR_NETWORK_NAME>> \
   --subnet=<<YOUR_SUBNET_NAME>> \
   --allow-unauthenticated \
   --vpc-egress=all-traffic \
   --set-env-vars GEMINI_API_KEY=<<YOUR_GEMINI_API_KEY>>,DATABASE_URL=postgresql+pg8000://postgres:<<YOUR_PASSWORD>>@<<PRIVATE_IP_HOST>>:5432/postgres,GCS_BUCKET_NAME=<<YOUR_GCS_BUCKET>>
```

**Grant Cloud Run SA access to AlloyDB:**
```bash
PROJECT_ID=$(gcloud config get-value project)
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/alloydb.client"
```

---

## 8. Cleanup

```bash
# Via Console:
# https://console.cloud.google.com/alloydb/clusters
# Click ⋮ next to cluster → DELETE

# Via gcloud:
gcloud alloydb clusters delete <CLUSTER_ID> --region=us-central1 --force
```
