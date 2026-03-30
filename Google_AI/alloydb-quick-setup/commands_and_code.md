# Commands & Code — AlloyDB Quick Setup Lab
> **Codelab:** One-click AlloyDB cluster + instance setup

---

## 1. Before You Begin — GCP Project Setup

```bash
# Verify authentication
gcloud auth list

# Verify active project
gcloud config list project

# Set project if needed
gcloud config set project <YOUR_PROJECT_ID>
```

Enable required APIs via the GCP Console or use:
```bash
gcloud services enable alloydb.googleapis.com \
                       compute.googleapis.com \
                       cloudresourcemanager.googleapis.com
```

---

## 2. AlloyDB Setup — Script-Based (Recommended)

```bash
# Step 1: Clone the repository
git clone https://github.com/GoogleCloudPlatform/devrel-demos.git

# Step 2: Navigate to the project directory
cd devrel-demos/infrastructure/easy-alloydb-setup

# Step 3: Run the setup script (must be inside project directory)
sh run.sh
```

After running, a web UI link appears in terminal:
- Click the link (or use **Web Preview** in Cloud Shell)
- Enter your **Project ID**, **Cluster name**, **Instance name**
- Wait for provisioning to complete ☕

---

## 3. Alternative — One-Click Setup

Use the direct one-click button from the codelab page which opens the Cloud Console with the setup pre-configured. Provides the same result as the script approach.

---

## 4. Enable Public IP (For Local/External Testing Only)

> ⚠️ **Only for development. Remove when done.**

**Via Console:**
1. Go to your AlloyDB instance in the Console
2. Click **EDIT**
3. Enable **Public IP** (not outbound)
4. Under **Authorized External Networks**, add: `0.0.0.0/0`
5. Save

**Via gcloud:**
```bash
# Enable public IP on an AlloyDB instance
gcloud alloydb instances update <INSTANCE_ID> \
  --cluster=<CLUSTER_ID> \
  --region=<REGION> \
  --assign-inbound-public-ip=ASSIGN_IPV4
```

**Remove public access when done:**
```bash
# Disable public IP
gcloud alloydb instances update <INSTANCE_ID> \
  --cluster=<CLUSTER_ID> \
  --region=<REGION> \
  --no-assign-inbound-public-ip
```

---

## 5. Verify AlloyDB Cluster and Instance

```bash
# List all AlloyDB clusters
gcloud alloydb clusters list --region=<REGION>

# Describe a specific cluster
gcloud alloydb clusters describe <CLUSTER_ID> --region=<REGION>

# List instances in a cluster
gcloud alloydb instances list \
  --cluster=<CLUSTER_ID> \
  --region=<REGION>
```

---

## 6. Connect to AlloyDB (psql)

```bash
# Connect via Cloud Shell using psql (private IP)
psql -h <PRIVATE_IP> -U postgres -d postgres

# Connect via public IP (if enabled for testing)
psql -h <PUBLIC_IP> -U postgres -d postgres
```

---

## 7. Sample SQL — Verify Connection

```sql
-- Check PostgreSQL version (AlloyDB is PostgreSQL-compatible)
SELECT version();

-- List all databases
\l

-- List all tables in current database
\dt

-- Check AlloyDB extensions available
SELECT * FROM pg_available_extensions WHERE name LIKE '%google%';
```

---

## 8. AlloyDB AI Extension (google_ml_integration)

```sql
-- Enable the AI integration extension
CREATE EXTENSION IF NOT EXISTS google_ml_integration;

-- Verify extension is active
SELECT * FROM pg_extension WHERE extname = 'google_ml_integration';

-- Example: Generate embeddings for a text column
SELECT google_ml.embedding(
  'textembedding-gecko@003',
  description
) AS embedding
FROM my_table
LIMIT 5;
```

---

## 9. Vector Search with ScaNN (pgvector + ScaNN)

```sql
-- Enable pgvector extension
CREATE EXTENSION IF NOT EXISTS vector;

-- Create a table with a vector column
CREATE TABLE my_embeddings (
  id SERIAL PRIMARY KEY,
  content TEXT,
  embedding vector(768)
);

-- Create a ScaNN index for fast similarity search
CREATE INDEX ON my_embeddings
USING scann (embedding cosine_distance);

-- Vector similarity search query
SELECT content, embedding <=> '[0.1, 0.2, ...]'::vector AS distance
FROM my_embeddings
ORDER BY distance
LIMIT 5;
```

---

## 10. Cleanup — Delete AlloyDB Cluster

**Via Console:**
1. Go to `https://console.cloud.google.com/alloydb/clusters`
2. Click the vertical ellipsis (⋮) next to your cluster
3. Click **DELETE**
4. Confirm — this deletes the cluster AND all its instances

**Via gcloud:**
```bash
# Delete the cluster (and all instances within it)
gcloud alloydb clusters delete <CLUSTER_ID> \
  --region=<REGION> \
  --force   # force-deletes instances inside the cluster
```

> ⚠️ Deletion is irreversible. All data in the cluster will be permanently lost.

---

## 11. Reference Links

| Resource | URL |
|----------|-----|
| AlloyDB Console | https://console.cloud.google.com/alloydb/clusters |
| AlloyDB Documentation | https://cloud.google.com/alloydb/docs |
| AlloyDB gcloud reference | https://cloud.google.com/sdk/gcloud/reference/alloydb |
| Google ML Integration docs | https://cloud.google.com/alloydb/docs/ai/work-with-embeddings |
| ScaNN Index docs | https://cloud.google.com/alloydb/docs/ai/create-index-vector-search |
| devrel-demos GitHub repo | https://github.com/GoogleCloudPlatform/devrel-demos |
| pgvector Extension | https://github.com/pgvector/pgvector |
