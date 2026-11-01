# Deploy the optional guideline reranker to Cloud Run

The app still retrieves candidates from Supabase. When configured, it sends up to
50 candidates to this service, filters publication notices, and uses the model's
highest-scoring passages. This is a cross-encoder reranking step, not an answer
generation service. It does not load PDFs, a local index, or an embedding model.
Its scores do not reproduce all the Python full-search service's query enrichment
and composite scoring adjustments.

If the service fails, times out, returns invalid/empty results, or has no usable
candidates, the app returns the original Supabase results in database order.
The short answer is then built from whichever passages were returned. Fallback
does not guarantee that those passages will provide a suitable short answer.
Supabase itself must be available; this is not a fallback for a database outage.

## 1. Prepare Google Cloud

Use a project attached to your trial billing account. Install the Google Cloud
CLI and run the following in PowerShell from the repository root:

```powershell
gcloud auth login
$ProjectId = "YOUR_GOOGLE_CLOUD_PROJECT_ID"
gcloud config set project $ProjectId
gcloud services enable run.googleapis.com cloudbuild.googleapis.com artifactregistry.googleapis.com secretmanager.googleapis.com
gcloud iam service-accounts create cliniq-reranker --display-name="ClinIQ reranker"
```

In Google Cloud Console, open **Secret Manager → Create secret**. Name it
`cliniq-rerank-key`, enter a new random secret value (at least 32 random bytes),
and keep that value for the Heroku configuration below. This is a new shared
API key, separate from your Supabase keys. Do not commit it to this repository.

Give the runtime service account access to this secret:

```powershell
gcloud secrets add-iam-policy-binding cliniq-rerank-key --member="serviceAccount:cliniq-reranker@$ProjectId.iam.gserviceaccount.com" --role="roles/secretmanager.secretAccessor"
```

These instructions assume your account has permission to enable APIs, build
containers, create service accounts and deploy Cloud Run services. Source builds
may require the build service account's Cloud Run Builder role; follow Google's
source-deployment permission instructions if the build reports missing access.

## 2. Deploy only the reranker directory

```powershell
gcloud run deploy cliniq-reranker --source ./ml-service/reranker --region us-central1 --memory 2Gi --cpu 1 --concurrency 1 --min-instances 0 --max-instances 1 --timeout 20s --cpu-throttling --allow-unauthenticated --service-account="cliniq-reranker@$ProjectId.iam.gserviceaccount.com" --set-secrets="RERANK_SERVICE_API_KEY=cliniq-rerank-key:1" --startup-probe="httpGet.path=/health,httpGet.port=8080,periodSeconds=10,timeoutSeconds=2,failureThreshold=24"
```

Accept the source-build setup prompts if asked. The supplied Dockerfile downloads
the model during the build and serves from its baked cache. It runs one Uvicorn
worker, one inference at a time, with CPU threads limited to one and batch size
four. No models are added to Heroku or your PDF extraction service.

Cloud Run's endpoint is publicly reachable so Heroku can call it; `/rerank`
requires the shared API key. `/health` exposes readiness only. Do not send patient
records to this endpoint; the app sends only the guideline query and excerpts.

The 2 GiB allocation is an initial resource limit, **not a measured memory
guarantee**. Check Cloud Run memory metrics under load before relying on this in
production. Keeping minimum instances at zero reduces idle cost but means cold
starts; an initial request may fall back to Supabase while the model loads.

Get the deployed URL and check readiness:

```powershell
$RerankUrl = gcloud run services describe cliniq-reranker --region us-central1 --format="value(status.url)"
Invoke-RestMethod "$RerankUrl/health"
```

## 3. Configure and deploy the Next.js app

Keep existing Supabase credentials and `LAB_EXTRACT_SERVICE_URL` unchanged.
Deploy this repository's updated app code to Heroku using your normal deployment
workflow. Set the following variables in Heroku **Settings → Config Vars**:

```text
RERANK_SERVICE_URL       = the Cloud Run service URL (without /rerank)
RERANK_SERVICE_API_KEY   = the same value stored in Secret Manager
RERANK_TIMEOUT_MS        = 8000
```

For local testing, put the same values in `.env.local` and restart Next.js.
The timeout is clamped to 500–15000 ms. Keep it below your frontend/proxy timeout.
You do not need to re-embed or re-seed Supabase for this change.

## 4. Verify success and fallback

```powershell
$AppUrl = "https://YOUR_HEROKU_APP_URL"
$SearchBody = @{query="Find guidance about diabetes and kidney disease."; limit=5} | ConvertTo-Json
Invoke-RestMethod "$AppUrl/api/knowledge-assistant/search" -Method Post -ContentType "application/json" -Body $SearchBody | ConvertTo-Json -Depth 8
```

After the model is warm, expect `engine: supabase_pgvector_rerank` and
`reranking_executed: true` when it finds usable passages. To test failure locally,
temporarily use `RERANK_SERVICE_URL=http://127.0.0.1:1`, restart Next.js, and make
the same query: it should return HTTP 200 with `engine: supabase_pgvector`,
`reranking_executed: false`, and `reranker_fallback_used: true`. Restore the real
URL afterward. Removing `RERANK_SERVICE_URL` disables reranking entirely.

Monitor Cloud Run request latency and container memory at concurrency one, then
test simultaneous searches. Review credit expiry and set billing alerts; alerts
are notifications, not a hard spending cap. Cloud Build, Artifact Registry and
network usage can incur costs in addition to Cloud Run.

## Local contract tests

```powershell
node tests/knowledge-rerank.cjs
python -m pytest ml-service/reranker/test_server.py -q
npm.cmd run typecheck
```

Tests use fake model scores and mock service failures. They do not measure model
quality, real inference latency or peak container memory.

References: [Cloud Run source deployment](https://cloud.google.com/run/docs/deploying-source-code),
[deployment flags](https://docs.cloud.google.com/sdk/gcloud/reference/run/deploy),
[Secret Manager integration](https://docs.cloud.google.com/run/docs/configuring/services/secrets).
