# 04 — Backend Services and API Specification

## Objective

Implement the runtime services that turn asset requests into ComfyUI generations, reviewable candidates, approved assets, published assets, and stale dependency updates.

This document defines:

- Asset service responsibilities
- Queue worker behavior
- ComfyUI dispatcher behavior
- REST API routes
- Request and response shapes
- Approval, rejection, revision, and publication flow
- Failure handling
- End-to-end orchestration

## Service Placement

Recommended main repository location:

```text
server/asset-service/
```

Recommended structure:

```text
server/asset-service/
  src/
    app.ts
    config.ts
    db.ts
    routes/
      assetRoutes.ts
      jobRoutes.ts
      workflowRoutes.ts
      reviewRoutes.ts
    services/
      AssetRequestService.ts
      AssetReviewService.ts
      AssetPublishService.ts
      AssetDependencyService.ts
      PromptBuilderService.ts
      StorageService.ts
      WorkflowRegistryService.ts
      ComfyDispatcher.ts
    workers/
      AssetGenerationWorker.ts
    types/
      asset.ts
      job.ts
      workflow.ts
      comfy.ts
    utils/
      imageValidation.ts
      pathBuilder.ts
      json.ts
  workflows/
  workflow-manifests/
  package.json
  tsconfig.json
```

Use TypeScript/Fastify if matching the existing API Gateway prototype stack.

## Configuration

Required environment variables:

```text
ASSET_SERVICE_PORT=3300
DATABASE_URL=postgres://...
COMFYUI_BASE_URL=http://127.0.0.1:8188
ASSET_STORAGE_ROOT=C:/dev/wishes/repos/wishes-game
ASSET_RELATIVE_ROOT=generated-assets
ASSET_WORKER_ID=local-asset-worker-1
ASSET_WORKER_POLL_MS=2000
ASSET_MAX_CONCURRENT_JOBS=1
```

Do not hard-code local paths.

## Service Responsibilities

### AssetRequestService

Creates generation requests.

Responsibilities:

1. Validate object reference.
2. Validate asset role.
3. Resolve source asset requirements.
4. Build prompt package.
5. Select workflow.
6. Create generation job.
7. Mark job queued.
8. Return job id.

### PromptBuilderService

Builds structured prompt JSON.

Responsibilities:

1. Combine global Wishes style.
2. Add asset role prompt instructions.
3. Add card/character identity details.
4. Add source identity constraints for downstream assets.
5. Add negative prompts.
6. Preserve user notes.
7. Return structured prompt object.

### WorkflowRegistryService

Loads workflow definitions.

Responsibilities:

1. Read `asset_workflow` registry.
2. Load workflow JSON.
3. Load manifest/injection schema.
4. Validate required nodes.
5. Produce injectable workflow graph.

### ComfyDispatcher

Integrates with ComfyUI.

Responsibilities:

1. Submit workflow graph to `/prompt`.
2. Store ComfyUI prompt id.
3. Poll `/history/{prompt_id}`.
4. Resolve output files.
5. Copy outputs into normalized storage.
6. Return output metadata.

### AssetGenerationWorker

Processes queued jobs.

Responsibilities:

1. Poll for queued jobs.
2. Claim job with `FOR UPDATE SKIP LOCKED`.
3. Load workflow.
4. Inject runtime values.
5. Submit to ComfyUI.
6. Await completion.
7. Validate output.
8. Create asset record.
9. Create dependency rows.
10. Mark asset review pending.
11. Mark job review pending.

### AssetReviewService

Handles approval and rejection.

Responsibilities:

1. Approve asset.
2. Reject asset.
3. Move/copy files to approved/rejected storage.
4. Mark current asset.
5. Create review event.
6. Invalidate descendants if needed.
7. Queue regeneration if requested.

### AssetPublishService

Publishes approved assets.

Responsibilities:

1. Validate asset approved.
2. Copy to published storage.
3. Set published status/metadata.
4. Expose stable published URI.
5. Create review event.

### AssetDependencyService

Maintains the asset graph.

Responsibilities:

1. Create dependency records.
2. Resolve descendants.
3. Mark stale descendants.
4. Queue regeneration of stale required roles.
5. Provide lineage response for UI.

### StorageService

Normalizes file system operations.

Responsibilities:

1. Build storage paths.
2. Ensure directories exist.
3. Copy generated outputs.
4. Validate image dimensions.
5. Move/copy approved, rejected, published, archived files.
6. Derive MIME types and file sizes.

## Worker Claim SQL

Use a transaction-safe queue claim.

```sql
WITH next_job AS (
  SELECT uuid
  FROM asset_generation_job
  WHERE status = 'queued'
    AND attempt_count < max_attempts
  ORDER BY priority ASC, created_at ASC
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
UPDATE asset_generation_job j
SET
  status = 'in_progress',
  worker_id = $1,
  claimed_at = now(),
  started_at = now(),
  attempt_count = attempt_count + 1,
  updated_at = now()
FROM next_job
WHERE j.uuid = next_job.uuid
RETURNING j.*;
```

## Worker Loop

Pseudo-code:

```ts
while (serviceRunning) {
  const job = await claimNextJob(workerId);

  if (!job) {
    await sleep(pollMs);
    continue;
  }

  try {
    await processJob(job);
  } catch (error) {
    await markJobFailed(job.uuid, error);
  }
}
```

## Job Processing Flow

```text
claim job
  -> load role
  -> load workflow registry row
  -> validate source asset if required
  -> build workflow input
  -> submit to ComfyUI
  -> update job with comfy_prompt_id
  -> poll for outputs
  -> copy output to pending storage
  -> validate image
  -> create asset row
  -> create dependency rows
  -> create review event
  -> mark asset review_pending
  -> mark job review_pending
```

## ComfyUI Dispatcher API

### submitWorkflow

Input:

```ts
type SubmitWorkflowInput = {
  workflow: unknown;
  clientId: string;
};
```

Output:

```ts
type SubmitWorkflowOutput = {
  promptId: string;
};
```

### waitForCompletion

Input:

```ts
type WaitForCompletionInput = {
  promptId: string;
  timeoutMs: number;
  pollMs: number;
};
```

Output:

```ts
type ComfyOutput = {
  promptId: string;
  outputs: Array<{
    filename: string;
    subfolder: string;
    type: string;
    url: string;
  }>;
  rawHistory: unknown;
};
```

## REST API Routes

Base path:

```text
/api/assets
```

### GET /health

Returns service health.

Response:

```json
{
  "status": "ok",
  "postgres": "connected",
  "comfyui": "connected",
  "storage": "available"
}
```

### GET /roles

Returns asset roles.

Response:

```json
{
  "roles": [
    {
      "code": "portrait",
      "asset_type_code": "image",
      "name": "Portrait",
      "boundaries": {}
    }
  ]
}
```

### GET /workflows

Returns active workflow registry entries.

### POST /requests

Create a generation request.

Request:

```json
{
  "object_type": "card",
  "object_uuid": "00000000-0000-0000-0000-000000000000",
  "asset_role": "portrait",
  "source_asset_uuid": null,
  "user_notes": "Generate an elf mage portrait.",
  "priority": 100,
  "settings": {
    "seed": null,
    "width": 1024,
    "height": 1024
  }
}
```

Response:

```json
{
  "job_uuid": "...",
  "status": "queued"
}
```

Validation:

- `asset_role` must exist.
- If role requires source, `source_asset_uuid` must exist and be approved/published.
- Object reference should be valid where possible.
- Workflow must exist and be active.

### GET /jobs/:jobUuid

Returns job state and result asset if available.

### GET /objects/:objectType/:objectUuid

Returns all assets for an object grouped by role.

Response:

```json
{
  "object_type": "card",
  "object_uuid": "...",
  "roles": {
    "portrait": [
      {
        "asset_uuid": "...",
        "status": "approved",
        "version": 1,
        "is_current": true,
        "storage_uri": "generated-assets/approved/..."
      }
    ]
  }
}
```

### GET /:assetUuid

Returns one asset with metadata.

### GET /:assetUuid/lineage

Returns asset ancestors and descendants.

Response:

```json
{
  "asset_uuid": "...",
  "root_asset_uuid": "...",
  "ancestors": [],
  "descendants": []
}
```

### POST /:assetUuid/approve

Approve an asset candidate.

Request:

```json
{
  "actor_uuid": null,
  "comment": "Approved as canonical portrait.",
  "publish": false,
  "queue_required_descendants": true
}
```

Behavior:

1. Validate asset is reviewable.
2. Copy file to approved storage.
3. Clear current asset for same object/role.
4. Mark this asset approved and current.
5. Create review event.
6. If approving a new portrait, mark old descendants stale.
7. Optionally queue downstream required assets.

### POST /:assetUuid/reject

Reject an asset candidate.

Request:

```json
{
  "actor_uuid": null,
  "comment": "Face does not match approved portrait."
}
```

Behavior:

1. Copy file to rejected storage.
2. Mark asset rejected.
3. Create review event.
4. Do not delete prompt history.

### POST /:assetUuid/publish

Publish an approved asset.

Request:

```json
{
  "actor_uuid": null,
  "comment": "Ready for portal use."
}
```

Behavior:

1. Validate asset approved.
2. Copy to published storage.
3. Mark published.
4. Store publish metadata.
5. Create review event.

### POST /:assetUuid/regenerate

Queue a regenerated candidate from an existing source.

Request:

```json
{
  "actor_uuid": null,
  "asset_role": "full_body",
  "revision_notes": "Make the boots more visible and preserve the original cloak colors.",
  "settings": {
    "seed": null
  }
}
```

Behavior:

1. Use existing asset as source or root depending on role.
2. Build revised prompt.
3. Queue new generation job.

### POST /:assetUuid/regenerate-descendants

Queue regeneration jobs for stale descendants.

Request:

```json
{
  "actor_uuid": null,
  "roles": ["full_body", "icon", "thumbnail", "card_front", "tactical_sprite_sheet"]
}
```

## Approval Transaction

Approval should be atomic.

Pseudo-code:

```ts
await db.transaction(async tx => {
  const asset = await lockAsset(tx, assetUuid);
  await storage.copyToApproved(asset);
  await clearCurrentForRole(tx, asset.object_type, asset.object_uuid, asset.asset_role_code);
  await markApprovedCurrent(tx, assetUuid, actorUuid);
  await createReviewEvent(tx, assetUuid, 'approved', previousStatus, 'approved', actorUuid, comment);

  if (asset.asset_role_code === 'portrait') {
    await markOldDescendantsStale(tx, asset);
  }
});
```

If file copy fails, do not update database.

If database update fails after copy succeeds, leave file in approved storage but record reconciliation need in logs.

## Authorization Notes

For first local prototype, user authentication may be stubbed.

However, route handlers should accept `actor_uuid` or derive it from authenticated user context later.

Future rules:

- Only authorized users may approve/publish.
- Rejection may require comment.
- Publication may require approved status.

## Validation Rules by Role

### portrait

- Does not require source.
- Must be square.
- Must be human-approved before downstream generation.

### full_body

- Requires approved portrait source.
- Must preserve identity.
- Should be vertical 2:3.

### icon

- Requires approved portrait source.
- Prefer deterministic crop/resize.

### thumbnail

- Requires approved portrait source.
- Prefer deterministic resize.

### card_front

- Requires approved portrait or full_body source.
- Text must be deterministic overlay, not AI-generated text.

### tactical_sprite_sheet

- Prefer approved full_body source.
- May fall back to approved portrait.
- Must include sprite metadata.

## Error JSON Shape

```json
{
  "code": "COMFYUI_UNAVAILABLE",
  "message": "Could not connect to ComfyUI at http://127.0.0.1:8188",
  "details": {},
  "occurred_at": "2026-07-08T00:00:00Z"
}
```

Recommended error codes:

```text
INVALID_ASSET_ROLE
SOURCE_ASSET_REQUIRED
SOURCE_ASSET_NOT_APPROVED
WORKFLOW_NOT_FOUND
WORKFLOW_NODE_MISSING
COMFYUI_UNAVAILABLE
COMFYUI_REJECTED_WORKFLOW
COMFYUI_TIMEOUT
OUTPUT_NOT_FOUND
OUTPUT_COPY_FAILED
IMAGE_VALIDATION_FAILED
DATABASE_ERROR
UNKNOWN_ERROR
```

## Logging

Log every state transition:

- Job requested
- Job queued
- Job claimed
- Workflow submitted
- ComfyUI prompt id received
- Output received
- Asset created
- Review pending
- Approved/rejected/published
- Descendants marked stale

## First Milestone Implementation

Implement endpoints in this order:

1. `GET /health`
2. `GET /roles`
3. `GET /workflows`
4. `POST /requests`
5. Worker job claim loop
6. Portrait generation
7. `GET /jobs/:jobUuid`
8. `GET /objects/:objectType/:objectUuid`
9. `POST /:assetUuid/approve`
10. Full body generation from approved portrait
11. Icon/thumbnail generation
12. Card front composition
13. Sprite sheet candidate generation

## Testing Strategy

### Unit tests

- Prompt builder
- Path builder
- Workflow injection
- Storage URI generation
- Role validation
- Source asset validation

### Integration tests

- Database migration reset
- Seed asset roles
- Create portrait job
- Mock ComfyUI output
- Create review pending asset
- Approve portrait
- Queue downstream full body
- Mark descendants stale

### Manual test

1. Start local database.
2. Start ComfyUI.
3. Start asset service.
4. Request portrait.
5. Confirm output appears in pending storage.
6. Approve portrait.
7. Request full body from portrait.
8. Confirm lineage.
9. Replace portrait.
10. Confirm full body becomes stale.

## Acceptance Criteria

Backend/API implementation is complete when:

1. Asset service starts with health endpoint.
2. Roles and workflows are queryable.
3. A portrait generation job can be queued.
4. Worker can claim the job.
5. Worker can submit a workflow to ComfyUI.
6. Output is copied to normalized storage.
7. Asset row is created as review pending.
8. Asset can be approved.
9. Approved portrait becomes current.
10. Derived asset jobs can use approved portrait as source.
11. Rejection and publication routes work.
12. Lineage endpoint returns ancestors and descendants.
13. Portrait replacement marks descendants stale.
14. Failed jobs are retryable and preserve error metadata.
