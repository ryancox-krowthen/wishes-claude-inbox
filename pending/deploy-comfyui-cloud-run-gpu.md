# Task: Deploy ComfyUI to Google Cloud Run GPU

## Objective

Implement and document a production-shaped deployment of ComfyUI as a private, GPU-backed Google Cloud Run service for the Wishes asset-generation pipeline.

The service must:

- Use Cloud Run GPU with one NVIDIA L4.
- Scale to zero with `min-instances=0`.
- Start with `max-instances=1` and `concurrency=1`.
- Store its image in Google Artifact Registry.
- Store models, workflows, inputs, and outputs in Google Cloud Storage.
- Be callable only by the Wishes asset-service or authorized operators.
- Never directly approve assets or mutate authoritative Wishes game state.
- Return generation status and output metadata to the Wishes asset-service.

Do not commit or push changes to the Wishes game repository unless the user explicitly authorizes it. Prepare all implementation changes and validation results for review.

---

## Target Architecture

```text
Wishes database / asset queue
        |
        v
server/asset-service
        |
        v
Authenticated private Cloud Run request
        |
        v
wishes-comfyui API wrapper :8080
        |
        v
ComfyUI process :8188
        |
        +--> Cloud Storage models/workflows/inputs
        |
        +--> Cloud Storage generated outputs
        |
        v
asset-service records pending_review output
```

Initial Cloud Run configuration:

```text
Service: wishes-comfyui
Region: us-central1
GPU: NVIDIA L4
GPU count: 1
CPU: 4
Memory: 16Gi
Concurrency: 1
Timeout: 3600 seconds
Minimum instances: 0
Maximum instances: 1
Authentication: required
```

---

## 1. Inspect Existing Wishes Implementation

Before editing:

1. Read the repository root `CLAUDE.md`.
2. Inspect `server/asset-service` and any current asset-generation implementation.
3. Inspect existing asset tables, migrations, roles, boundaries, workflow records, queues, and status enums.
4. Inspect any existing ComfyUI client, workflow JSON, model configuration, or deployment infrastructure.
5. Reuse current project conventions and avoid creating parallel duplicate systems.
6. Preserve the Wishes rule that the server is authoritative and generated assets require approval.

Report conflicts or missing prerequisites before attempting deployment.

---

## 2. Required Google Cloud Variables

Document these variables in a deployment README or script without committing real secrets:

```bash
PROJECT_ID=your-google-cloud-project-id
REGION=us-central1
REPOSITORY=wishes-containers
IMAGE_NAME=wishes-comfyui
SERVICE_NAME=wishes-comfyui
RUNTIME_SERVICE_ACCOUNT=wishes-comfyui-runtime
CALLER_SERVICE_ACCOUNT=wishes-asset-service
MODEL_BUCKET=${PROJECT_ID}-wishes-ai-models
WORKFLOW_BUCKET=${PROJECT_ID}-wishes-ai-workflows
INPUT_BUCKET=${PROJECT_ID}-wishes-ai-inputs
OUTPUT_BUCKET=${PROJECT_ID}-wishes-ai-outputs
IMAGE_URI=${REGION}-docker.pkg.dev/${PROJECT_ID}/${REPOSITORY}/${IMAGE_NAME}:0.1.0
```

Bucket names must include the project ID or another unique prefix because Cloud Storage bucket names are globally unique.

---

## 3. Enable Google Cloud APIs

Document and validate:

```bash
gcloud config set project "$PROJECT_ID"

gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  storage.googleapis.com \
  secretmanager.googleapis.com \
  iam.googleapis.com
```

Add other APIs only when the implementation actually requires them.

---

## 4. Create Artifact Registry

```bash
gcloud artifacts repositories create "$REPOSITORY" \
  --repository-format=docker \
  --location="$REGION" \
  --description="Wishes container images" \
  --project="$PROJECT_ID"

gcloud auth configure-docker "${REGION}-docker.pkg.dev"
```

Use Artifact Registry rather than Docker Hub.

---

## 5. Create Cloud Storage Buckets

```bash
gcloud storage buckets create "gs://${MODEL_BUCKET}" \
  --project="$PROJECT_ID" \
  --location="$REGION" \
  --uniform-bucket-level-access

gcloud storage buckets create "gs://${WORKFLOW_BUCKET}" \
  --project="$PROJECT_ID" \
  --location="$REGION" \
  --uniform-bucket-level-access

gcloud storage buckets create "gs://${INPUT_BUCKET}" \
  --project="$PROJECT_ID" \
  --location="$REGION" \
  --uniform-bucket-level-access

gcloud storage buckets create "gs://${OUTPUT_BUCKET}" \
  --project="$PROJECT_ID" \
  --location="$REGION" \
  --uniform-bucket-level-access
```

Use these logical layouts:

```text
models/
  checkpoints/
  diffusion_models/
  text_encoders/
  clip/
  loras/
  vae/
  controlnet/
  upscale_models/

workflows/
  manifests/
  api/

inputs/
  cards/{cardUuid}/{assetRole}/{jobUuid}/

outputs/
  cards/{cardUuid}/{assetRole}/{jobUuid}/
```

Do not delete rejected or temporary outputs automatically until retention requirements are confirmed.

---

## 6. Create Service Accounts and IAM

```bash
gcloud iam service-accounts create "$RUNTIME_SERVICE_ACCOUNT" \
  --display-name="Wishes ComfyUI Cloud Run Runtime" \
  --project="$PROJECT_ID"

gcloud iam service-accounts create "$CALLER_SERVICE_ACCOUNT" \
  --display-name="Wishes Asset Service Caller" \
  --project="$PROJECT_ID"

RUNTIME_EMAIL="${RUNTIME_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com"
CALLER_EMAIL="${CALLER_SERVICE_ACCOUNT}@${PROJECT_ID}.iam.gserviceaccount.com"
```

Grant least privilege:

```bash
gcloud storage buckets add-iam-policy-binding "gs://${MODEL_BUCKET}" \
  --member="serviceAccount:${RUNTIME_EMAIL}" \
  --role="roles/storage.objectViewer"

gcloud storage buckets add-iam-policy-binding "gs://${WORKFLOW_BUCKET}" \
  --member="serviceAccount:${RUNTIME_EMAIL}" \
  --role="roles/storage.objectViewer"

gcloud storage buckets add-iam-policy-binding "gs://${INPUT_BUCKET}" \
  --member="serviceAccount:${RUNTIME_EMAIL}" \
  --role="roles/storage.objectViewer"

gcloud storage buckets add-iam-policy-binding "gs://${OUTPUT_BUCKET}" \
  --member="serviceAccount:${RUNTIME_EMAIL}" \
  --role="roles/storage.objectAdmin"
```

Do not use Owner or Editor roles and do not create downloadable service-account keys unless unavoidable.

---

## 7. Container Implementation

Use the existing asset-service conventions. If no equivalent exists, create:

```text
server/asset-service/comfyui-cloudrun/
  Dockerfile
  entrypoint.sh
  requirements.txt
  app/
    main.py
    schemas.py
    comfyui_process.py
    comfyui_client.py
    storage_client.py
  workflows/
    README.md
  README.md
```

Requirements:

- API wrapper listens on Cloud Run `PORT`, normally `8080`.
- ComfyUI listens only on `127.0.0.1:8188` inside the container.
- Do not expose the raw ComfyUI UI publicly.
- Pin ComfyUI to a tested commit SHA.
- Pin custom nodes and Python dependencies.
- Do not clone unpinned moving branches during production builds.
- Do not include credentials in the image.

Example Dockerfile baseline:

```dockerfile
FROM nvidia/cuda:12.4.1-runtime-ubuntu22.04

ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PIP_NO_CACHE_DIR=1 \
    COMFYUI_PATH=/opt/ComfyUI \
    PORT=8080

ARG COMFYUI_COMMIT

RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates curl git libgl1 libglib2.0-0 \
    python3 python3-pip python3-venv \
    && rm -rf /var/lib/apt/lists/*

RUN git clone https://github.com/comfyanonymous/ComfyUI.git ${COMFYUI_PATH} \
    && cd ${COMFYUI_PATH} \
    && git checkout ${COMFYUI_COMMIT}

WORKDIR ${COMFYUI_PATH}

RUN python3 -m pip install --upgrade pip setuptools wheel
RUN python3 -m pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu124
RUN python3 -m pip install -r requirements.txt

COPY requirements.txt /tmp/wishes-requirements.txt
RUN python3 -m pip install -r /tmp/wishes-requirements.txt

COPY app /opt/wishes-comfyui/app
COPY entrypoint.sh /opt/wishes-comfyui/entrypoint.sh
RUN chmod +x /opt/wishes-comfyui/entrypoint.sh

EXPOSE 8080
CMD ["/opt/wishes-comfyui/entrypoint.sh"]
```

Verify CUDA, PyTorch, ComfyUI, and custom-node compatibility before locking versions.

---

## 8. API Wrapper

Implement these authenticated routes:

```text
GET  /health
GET  /ready
POST /generate
GET  /jobs/{jobId}
POST /jobs/{jobId}/cancel
```

Wrapper responsibilities:

1. Start ComfyUI on `127.0.0.1:8188`.
2. Wait for ComfyUI readiness before `/ready` returns success.
3. Validate request schemas and asset-role boundaries.
4. Load only approved, versioned workflow definitions.
5. Resolve models and LoRAs from approved manifests.
6. Download required inputs from Cloud Storage.
7. Submit the API-format workflow to ComfyUI.
8. Monitor prompt completion.
9. Upload outputs to the requested approved output prefix.
10. Return job ID, prompt ID, output URIs, metadata, and errors.
11. Reject arbitrary filesystem paths and arbitrary workflow code.
12. Sanitize filenames and prevent bucket/path traversal.
13. Log job IDs without logging secrets or sensitive prompt content unnecessarily.

Raw ComfyUI internal endpoints may include `/prompt`, `/history/{prompt_id}`, `/view`, and `/ws`.

---

## 9. Generation Contract

Implement or adapt the current contract. Suggested request:

```json
{
  "assetJobId": "uuid",
  "objectType": "card",
  "objectUuid": "uuid",
  "assetRole": "portrait",
  "workflowName": "character_portrait_v1",
  "workflowVersion": "1.0.0",
  "prompt": {
    "positive": "Wishes character portrait...",
    "negative": "low quality, watermark, signature..."
  },
  "inputs": {
    "referenceImageUri": "gs://project-wishes-ai-inputs/cards/cardUuid/reference.png"
  },
  "output": {
    "bucket": "project-wishes-ai-outputs",
    "prefix": "cards/cardUuid/portrait/jobUuid/"
  },
  "settings": {
    "seed": -1,
    "width": 1024,
    "height": 1024,
    "steps": 25,
    "cfg": 7
  }
}
```

Suggested immediate response:

```json
{
  "assetJobId": "uuid",
  "providerJobId": "comfy-prompt-id",
  "status": "generating"
}
```

Suggested completed response/status payload:

```json
{
  "assetJobId": "uuid",
  "providerJobId": "comfy-prompt-id",
  "status": "generated",
  "outputs": [
    {
      "uri": "gs://project-wishes-ai-outputs/cards/cardUuid/portrait/jobUuid/output.png",
      "mimeType": "image/png",
      "width": 1024,
      "height": 1024,
      "seed": 12345
    }
  ]
}
```

The Cloud Run service generates files only. It must not mark outputs approved.

---

## 10. Model and Workflow Strategy

Do not assume that Cloud Storage can be used as a low-latency mounted model filesystem without testing.

Implement an explicit startup/cache strategy:

1. Keep authoritative models and workflow files in Cloud Storage.
2. Maintain a versioned model manifest containing URI, expected filename, checksum, size, and destination.
3. At startup, download only the model set required by the deployed workflow set into writable ephemeral storage.
4. Verify checksums before startup readiness succeeds.
5. Fail readiness when required models are missing or invalid.
6. Log total model download and initialization time.
7. Evaluate baking stable models into the image only if startup download time makes scale-to-zero impractical.

Remember that scale-to-zero means local cache is lost whenever an instance terminates. Cold-start model transfer time must therefore be measured as part of acceptance testing.

---

## 11. Build and Push

Document both local Docker and Cloud Build options. Cloud Build is preferred when local Docker/GPU tooling is not required.

Local build:

```bash
docker build \
  --build-arg COMFYUI_COMMIT=<tested-commit-sha> \
  -t "$IMAGE_NAME:0.1.0" \
  server/asset-service/comfyui-cloudrun

docker tag "$IMAGE_NAME:0.1.0" "$IMAGE_URI"
docker push "$IMAGE_URI"
```

Cloud Build alternative:

```bash
gcloud builds submit \
  server/asset-service/comfyui-cloudrun \
  --tag "$IMAGE_URI" \
  --project "$PROJECT_ID"
```

---

## 12. Deploy Cloud Run GPU Service

Validate the current Google Cloud CLI syntax and regional L4 availability before execution.

Target command:

```bash
gcloud run deploy "$SERVICE_NAME" \
  --image="$IMAGE_URI" \
  --project="$PROJECT_ID" \
  --region="$REGION" \
  --platform=managed \
  --no-allow-unauthenticated \
  --service-account="$RUNTIME_EMAIL" \
  --gpu=1 \
  --gpu-type=nvidia-l4 \
  --cpu=4 \
  --memory=16Gi \
  --concurrency=1 \
  --timeout=3600 \
  --min-instances=0 \
  --max-instances=1 \
  --port=8080 \
  --set-env-vars="MODEL_BUCKET=${MODEL_BUCKET},WORKFLOW_BUCKET=${WORKFLOW_BUCKET},INPUT_BUCKET=${INPUT_BUCKET},OUTPUT_BUCKET=${OUTPUT_BUCKET},COMFYUI_PORT=8188"
```

Do not enable unauthenticated access.

---

## 13. Grant Cloud Run Invocation

```bash
gcloud run services add-iam-policy-binding "$SERVICE_NAME" \
  --project="$PROJECT_ID" \
  --region="$REGION" \
  --member="serviceAccount:${CALLER_EMAIL}" \
  --role="roles/run.invoker"
```

The Wishes asset-service must acquire a Google-signed ID token with the Cloud Run service URL as audience and send it as a bearer token.

Do not use a static shared API key as the primary service-to-service authentication mechanism.

---

## 14. Wishes Asset-Service Integration

Add environment variables using existing configuration conventions:

```text
COMFYUI_SERVICE_URL
COMFYUI_AUTH_MODE=google_oidc
COMFYUI_REQUEST_TIMEOUT_SECONDS=3600
ASSET_INPUT_BUCKET
ASSET_OUTPUT_BUCKET
```

Asset-service responsibilities:

1. Read pending asset-generation jobs.
2. Validate object, asset type, asset role, boundaries, prompt policy, and workflow.
3. Upload/reference approved input files.
4. Acquire a Cloud Run ID token.
5. Invoke `/generate`.
6. Track provider job ID and generation state.
7. Poll status or process the selected completion mechanism.
8. Record generated file metadata.
9. Transition output to `pending_review`.
10. Never approve generated assets automatically unless an explicit future rule authorizes it.

Recommended lifecycle:

```text
requested
queued
generating
generated
pending_review
approved
rejected
failed
cancelled
```

Reuse current lifecycle values when they already exist.

---

## 15. Database Changes

Inspect current tables before adding anything.

Only add an `asset_job` migration when equivalent queue/job data is not already represented.

Suggested logical fields:

```text
asset_job
- uuid
- object_type
- object_uuid
- asset_type
- asset_role
- workflow_name
- workflow_version
- status
- prompt jsonb
- settings jsonb
- input_refs jsonb
- output_refs jsonb
- provider_job_id
- error jsonb
- created_at
- updated_at
- completed_at
```

Asset records should retain approved/rejected status and generation provenance, including workflow version, model versions, LoRAs, seed, dimensions, and source job UUID.

Follow Wishes migration numbering and naming standards.

---

## 16. Health, Startup, and Failure Handling

`/health` should indicate the wrapper process is alive.

`/ready` should succeed only when:

- ComfyUI is running.
- Required workflows are loaded.
- Required model files exist and pass checksum verification.
- Cloud Storage configuration is valid.

Handle and report:

- model download failure
- checksum mismatch
- invalid workflow
- invalid asset role or boundaries
- GPU out-of-memory
- ComfyUI process failure
- generation timeout
- Cloud Storage input/output failure
- cancellation
- malformed output metadata

Ensure failed requests move the asset job to a retryable or terminal state according to existing queue rules.

---

## 17. Cost and Safety Controls

Implement and document:

- `min-instances=0`
- `max-instances=1`
- `concurrency=1`
- authenticated invocation only
- one queued dispatch at a time initially
- billing budget alerts
- generation duration metrics
- output size limits
- input size limits
- allowed workflow list
- allowed model/LoRA manifest
- no arbitrary custom-node installation at runtime

Recommend billing alerts at appropriate thresholds such as $25, $50, and $100, but do not create budgets without project billing details and explicit confirmation.

---

## 18. Validation

Complete these tests:

1. Build the container successfully.
2. Run static/lint/unit tests for the wrapper.
3. Start the container in CPU-safe validation mode where possible.
4. Deploy the private Cloud Run GPU service.
5. Confirm unauthenticated requests receive an authorization failure.
6. Confirm an authorized identity can call `/health` and `/ready`.
7. Submit one known API-format workflow.
8. Confirm the GPU is detected.
9. Confirm one image is generated.
10. Confirm the output is uploaded to the expected Cloud Storage prefix.
11. Confirm the asset-service records it as generated or pending review, never approved.
12. Confirm malformed workflows and disallowed paths are rejected.
13. Confirm `max-instances=1` and `concurrency=1` are set.
14. Allow the service to become idle and verify it scales to zero.
15. Trigger a new request after scale-to-zero and record cold-start time, model download time, initialization time, generation time, and total request time.
16. Determine whether scale-to-zero cold starts are acceptable for the Wishes review workflow.

---

## 19. Required Deliverables

Prepare the following without committing or pushing to `wishes-game` until authorized:

1. ComfyUI Cloud Run Dockerfile.
2. Entrypoint and process supervision.
3. Authenticated API wrapper.
4. Cloud Storage model/workflow/input/output integration.
5. Versioned model and workflow manifest support.
6. Wishes asset-service Cloud Run client.
7. Necessary database migration only when no equivalent exists.
8. Unit and integration tests.
9. Deployment README with exact commands.
10. Rollback instructions.
11. Cost-control and cold-start report.
12. List of files changed.
13. Validation results.
14. Proposed commit messages.

---

## Completion Report

At completion, report:

```text
STATUS
- complete, partial, or blocked

IMPLEMENTED
- files and behavior added

GOOGLE CLOUD RESOURCES
- resources created or commands prepared

VALIDATION
- tests run and results

COLD START
- model transfer, startup, and total latency

SECURITY
- authentication and IAM validation

COST CONTROLS
- scaling and concurrency settings

BLOCKERS
- missing permissions, quotas, billing, GPU availability, models, or workflows

NEXT USER ACTION
- exact approval or manual action required

PROPOSED COMMITS
- commit messages only; do not commit or push without authorization
```
