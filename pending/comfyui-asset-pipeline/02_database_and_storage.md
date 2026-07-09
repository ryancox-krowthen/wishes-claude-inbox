# 02 — Database and Storage Design

## Objective

Define the PostgreSQL schema, storage conventions, asset lineage model, queue model, and validation rules for the Wishes ComfyUI asset pipeline.

This document is an implementation guide for Claude Code. It should be implemented in the main Wishes repository, not in the inbox repository.

## Existing Wishes Constraints

The Wishes project already uses:

- PostgreSQL as canonical persistence.
- UUID primary keys.
- JSONB for flexible metadata.
- UTC timestamp application data.
- `pgcrypto` for `gen_random_uuid()`.
- A unified `assets` concept.
- Server-authoritative state.
- AI/asset systems that propose or generate candidates but do not directly mutate world state.

Do not place ownership or tradability fields on the asset itself unless they are strictly asset-publication concerns. Card ownership belongs to `user_card`, not `card`, and visual assets should not change gameplay ownership.

## Migration Placement

Use the existing migration numbering convention.

Recommended migration files:

```text
database/migrations/018_asset_pipeline_tables.sql
database/migrations/028_asset_pipeline_indexes.sql
database/migrations/038_asset_pipeline_constraints.sql
database/migrations/048_asset_pipeline_triggers.sql
```

If those numbers already exist, choose the next available number in each range.

Do not append suffixes such as `updated`, `final`, `new`, or `v2`.

## Core Tables

### asset_type

Defines broad asset categories.

```sql
CREATE TABLE IF NOT EXISTS asset_type (
  code TEXT PRIMARY KEY,
  name TEXT NOT NULL,
  description TEXT NOT NULL DEFAULT '',
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  updated_at TIMESTAMP NOT NULL DEFAULT now()
);
```

Seed values:

```sql
INSERT INTO asset_type (code, name, description)
VALUES
  ('image', 'Image', 'Two-dimensional image assets.'),
  ('sprite', 'Sprite', 'Sprite or sprite-sheet assets.'),
  ('model', 'Model', 'Three-dimensional model assets.'),
  ('workflow', 'Workflow', 'Generation workflow definitions.'),
  ('metadata', 'Metadata', 'Non-rendered support metadata.')
ON CONFLICT (code) DO UPDATE SET
  name = EXCLUDED.name,
  description = EXCLUDED.description,
  updated_at = now();
```

### asset_role

Defines specific functional roles and their boundaries.

```sql
CREATE TABLE IF NOT EXISTS asset_role (
  code TEXT PRIMARY KEY,
  asset_type_code TEXT NOT NULL REFERENCES asset_type(code),
  name TEXT NOT NULL,
  description TEXT NOT NULL DEFAULT '',
  boundaries JSONB NOT NULL DEFAULT '{}'::jsonb,
  is_required BOOLEAN NOT NULL DEFAULT false,
  is_active BOOLEAN NOT NULL DEFAULT true,
  sort_order INTEGER NOT NULL DEFAULT 0,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  updated_at TIMESTAMP NOT NULL DEFAULT now()
);
```

Required seed values:

```sql
INSERT INTO asset_role (
  code,
  asset_type_code,
  name,
  description,
  boundaries,
  is_required,
  sort_order
)
VALUES
  (
    'portrait',
    'image',
    'Portrait',
    'Canonical character/card portrait used as the visual identity source.',
    '{"width":1024,"height":1024,"aspect_ratio":"1:1","formats":["png","webp"],"requires_human_approval":true}'::jsonb,
    true,
    10
  ),
  (
    'card_front',
    'image',
    'Card Front',
    'Complete rendered card front using approved character art and card frame overlays.',
    '{"width":768,"height":1075,"aspect_ratio":"card","formats":["png","webp"],"requires_source_role":"portrait"}'::jsonb,
    true,
    20
  ),
  (
    'full_body',
    'image',
    'Full Body',
    'Full body character illustration derived from the approved portrait.',
    '{"width":1024,"height":1536,"aspect_ratio":"2:3","formats":["png","webp"],"requires_source_role":"portrait"}'::jsonb,
    true,
    30
  ),
  (
    'icon',
    'image',
    'Icon',
    'Small square icon derived from the approved portrait.',
    '{"width":512,"height":512,"aspect_ratio":"1:1","formats":["png","webp"],"requires_source_role":"portrait"}'::jsonb,
    true,
    40
  ),
  (
    'thumbnail',
    'image',
    'Thumbnail',
    'Small preview thumbnail derived from an approved source image.',
    '{"width":256,"height":256,"aspect_ratio":"1:1","formats":["png","webp"],"requires_source_role":"portrait"}'::jsonb,
    true,
    50
  ),
  (
    'tactical_sprite_sheet',
    'sprite',
    'Tactical Sprite Sheet',
    'Tactical battle sprite sheet derived from full body or portrait source.',
    '{"tile_width":128,"tile_height":128,"directions":["front","back","left","right"],"animations":["idle","walk","attack","cast","hit","down"],"formats":["png"],"preferred_source_role":"full_body"}'::jsonb,
    true,
    60
  )
ON CONFLICT (code) DO UPDATE SET
  asset_type_code = EXCLUDED.asset_type_code,
  name = EXCLUDED.name,
  description = EXCLUDED.description,
  boundaries = EXCLUDED.boundaries,
  is_required = EXCLUDED.is_required,
  sort_order = EXCLUDED.sort_order,
  updated_at = now();
```

### asset_workflow

Registry of ComfyUI workflows available to the asset service.

```sql
CREATE TABLE IF NOT EXISTS asset_workflow (
  uuid UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code TEXT NOT NULL UNIQUE,
  name TEXT NOT NULL,
  asset_role_code TEXT NOT NULL REFERENCES asset_role(code),
  workflow_path TEXT NOT NULL,
  workflow_version INTEGER NOT NULL DEFAULT 1,
  description TEXT NOT NULL DEFAULT '',
  injection_schema JSONB NOT NULL DEFAULT '{}'::jsonb,
  default_settings JSONB NOT NULL DEFAULT '{}'::jsonb,
  is_active BOOLEAN NOT NULL DEFAULT true,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  updated_at TIMESTAMP NOT NULL DEFAULT now()
);
```

Example `injection_schema`:

```json
{
  "positive_prompt_node": "6",
  "negative_prompt_node": "7",
  "seed_node": "25",
  "width_node": "12",
  "height_node": "13",
  "source_image_node": "31",
  "output_prefix_node": "42"
}
```

Seed workflow codes:

```text
portrait_generate
portrait_refine
full_body_from_portrait
card_front_from_portrait
icon_from_portrait
thumbnail_from_portrait
sprite_base_from_full_body
sprite_sheet_from_sprite_base
```

### asset

Canonical record for generated, approved, stale, archived, or published assets.

If an `assets` table already exists, migrate it carefully instead of duplicating it. The final table must support the fields below either directly or through compatible columns.

```sql
CREATE TABLE IF NOT EXISTS asset (
  uuid UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  object_type TEXT NOT NULL,
  object_uuid UUID NULL,

  asset_type_code TEXT NOT NULL REFERENCES asset_type(code),
  asset_role_code TEXT NOT NULL REFERENCES asset_role(code),

  source_asset_uuid UUID NULL REFERENCES asset(uuid) ON DELETE SET NULL,
  root_asset_uuid UUID NULL REFERENCES asset(uuid) ON DELETE SET NULL,

  version INTEGER NOT NULL DEFAULT 1,
  source_asset_version INTEGER NULL,

  status TEXT NOT NULL DEFAULT 'generated',

  storage_uri TEXT NOT NULL,
  storage_provider TEXT NOT NULL DEFAULT 'local',
  mime_type TEXT NOT NULL DEFAULT 'image/png',
  file_name TEXT NOT NULL,
  file_size_bytes BIGINT NULL,
  width INTEGER NULL,
  height INTEGER NULL,

  prompt JSONB NOT NULL DEFAULT '{}'::jsonb,
  generation_metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  review_metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  publish_metadata JSONB NOT NULL DEFAULT '{}'::jsonb,

  is_current BOOLEAN NOT NULL DEFAULT false,
  is_public BOOLEAN NOT NULL DEFAULT false,

  created_by UUID NULL,
  approved_by UUID NULL,
  published_by UUID NULL,

  created_at TIMESTAMP NOT NULL DEFAULT now(),
  updated_at TIMESTAMP NOT NULL DEFAULT now(),
  approved_at TIMESTAMP NULL,
  published_at TIMESTAMP NULL,
  archived_at TIMESTAMP NULL,

  CONSTRAINT chk_asset_status CHECK (
    status IN (
      'requested',
      'queued',
      'in_progress',
      'generated',
      'review_pending',
      'approved',
      'rejected',
      'published',
      'stale',
      'archived',
      'failed'
    )
  ),
  CONSTRAINT chk_asset_version_positive CHECK (version > 0),
  CONSTRAINT chk_asset_object_ref CHECK (
    object_type <> ''
  )
);
```

Recommended object type values:

```text
card
card_template
deck
user
npc
character
system
```

Do not enforce object references as a single foreign key because assets may attach to several entity types.

### asset_generation_job

Queue and execution record for ComfyUI jobs.

```sql
CREATE TABLE IF NOT EXISTS asset_generation_job (
  uuid UUID PRIMARY KEY DEFAULT gen_random_uuid(),

  object_type TEXT NOT NULL,
  object_uuid UUID NULL,
  asset_role_code TEXT NOT NULL REFERENCES asset_role(code),
  asset_workflow_uuid UUID NULL REFERENCES asset_workflow(uuid),

  source_asset_uuid UUID NULL REFERENCES asset(uuid) ON DELETE SET NULL,
  requested_asset_uuid UUID NULL REFERENCES asset(uuid) ON DELETE SET NULL,
  result_asset_uuid UUID NULL REFERENCES asset(uuid) ON DELETE SET NULL,

  status TEXT NOT NULL DEFAULT 'requested',
  priority INTEGER NOT NULL DEFAULT 100,
  attempt_count INTEGER NOT NULL DEFAULT 0,
  max_attempts INTEGER NOT NULL DEFAULT 3,

  request_payload JSONB NOT NULL DEFAULT '{}'::jsonb,
  prompt JSONB NOT NULL DEFAULT '{}'::jsonb,
  workflow_input JSONB NOT NULL DEFAULT '{}'::jsonb,
  workflow_output JSONB NOT NULL DEFAULT '{}'::jsonb,
  error JSONB NOT NULL DEFAULT '{}'::jsonb,

  comfy_prompt_id TEXT NULL,
  worker_id TEXT NULL,

  requested_by UUID NULL,
  claimed_at TIMESTAMP NULL,
  started_at TIMESTAMP NULL,
  finished_at TIMESTAMP NULL,
  failed_at TIMESTAMP NULL,
  created_at TIMESTAMP NOT NULL DEFAULT now(),
  updated_at TIMESTAMP NOT NULL DEFAULT now(),

  CONSTRAINT chk_asset_generation_job_status CHECK (
    status IN (
      'requested',
      'queued',
      'in_progress',
      'generated',
      'review_pending',
      'approved',
      'rejected',
      'published',
      'stale',
      'archived',
      'failed'
    )
  ),
  CONSTRAINT chk_asset_generation_attempts CHECK (attempt_count >= 0 AND max_attempts > 0)
);
```

### asset_review_event

Audit history for review, approval, rejection, publication, archival, and stale transitions.

```sql
CREATE TABLE IF NOT EXISTS asset_review_event (
  uuid UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  asset_uuid UUID NOT NULL REFERENCES asset(uuid) ON DELETE CASCADE,
  job_uuid UUID NULL REFERENCES asset_generation_job(uuid) ON DELETE SET NULL,
  event_type TEXT NOT NULL,
  previous_status TEXT NULL,
  new_status TEXT NOT NULL,
  actor_uuid UUID NULL,
  comment TEXT NOT NULL DEFAULT '',
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMP NOT NULL DEFAULT now(),

  CONSTRAINT chk_asset_review_event_type CHECK (
    event_type IN (
      'created',
      'generated',
      'review_requested',
      'approved',
      'rejected',
      'published',
      'stale',
      'archived',
      'restored',
      'failed',
      'regeneration_requested'
    )
  )
);
```

### asset_dependency

Optional but recommended explicit dependency table. This makes descendant lookup faster and clearer than recursive self-joins only.

```sql
CREATE TABLE IF NOT EXISTS asset_dependency (
  uuid UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  parent_asset_uuid UUID NOT NULL REFERENCES asset(uuid) ON DELETE CASCADE,
  child_asset_uuid UUID NOT NULL REFERENCES asset(uuid) ON DELETE CASCADE,
  dependency_type TEXT NOT NULL DEFAULT 'source',
  parent_version INTEGER NOT NULL,
  child_version INTEGER NOT NULL,
  metadata JSONB NOT NULL DEFAULT '{}'::jsonb,
  created_at TIMESTAMP NOT NULL DEFAULT now(),

  CONSTRAINT uq_asset_dependency UNIQUE (parent_asset_uuid, child_asset_uuid, dependency_type),
  CONSTRAINT chk_asset_dependency_type CHECK (
    dependency_type IN ('source', 'reference', 'composite', 'style', 'mask', 'control')
  )
);
```

## Indexes

Create indexes in the migration index range.

```sql
CREATE INDEX IF NOT EXISTS idx_asset_object
  ON asset (object_type, object_uuid);

CREATE INDEX IF NOT EXISTS idx_asset_role_status
  ON asset (asset_role_code, status);

CREATE INDEX IF NOT EXISTS idx_asset_source
  ON asset (source_asset_uuid);

CREATE INDEX IF NOT EXISTS idx_asset_root
  ON asset (root_asset_uuid);

CREATE INDEX IF NOT EXISTS idx_asset_current
  ON asset (object_type, object_uuid, asset_role_code)
  WHERE is_current = true;

CREATE INDEX IF NOT EXISTS idx_asset_generation_job_status_priority
  ON asset_generation_job (status, priority, created_at);

CREATE INDEX IF NOT EXISTS idx_asset_generation_job_source
  ON asset_generation_job (source_asset_uuid);

CREATE INDEX IF NOT EXISTS idx_asset_review_event_asset
  ON asset_review_event (asset_uuid, created_at DESC);

CREATE INDEX IF NOT EXISTS idx_asset_dependency_parent
  ON asset_dependency (parent_asset_uuid);

CREATE INDEX IF NOT EXISTS idx_asset_dependency_child
  ON asset_dependency (child_asset_uuid);
```

## Current Asset Constraint

There should be at most one current asset per object and asset role.

```sql
CREATE UNIQUE INDEX IF NOT EXISTS uq_asset_current_per_role
  ON asset (object_type, object_uuid, asset_role_code)
  WHERE is_current = true;
```

Important: this index treats `object_uuid IS NULL` specially in PostgreSQL. If system-level assets with null object UUIDs need current uniqueness, use a generated key or avoid null object UUIDs for current assets.

## Storage Layout

Use local storage in development.

Root:

```text
generated-assets/
```

Subfolders:

```text
generated-assets/
  pending/
  approved/
  rejected/
  published/
  archived/
  failed/
```

Object layout:

```text
generated-assets/{state}/{object_type}/{object_uuid}/{asset_role}/v{version}/{asset_uuid}.{ext}
```

Examples:

```text
generated-assets/pending/card/6d6f.../portrait/v1/1e25....png
generated-assets/approved/card/6d6f.../portrait/v1/1e25....png
generated-assets/published/card/6d6f.../card_front/88b1....png
```

For assets not tied to a UUID object, use:

```text
generated-assets/{state}/system/{scope}/{asset_role}/v{version}/{asset_uuid}.{ext}
```

## URI Rules

Store `storage_uri` as a stable application-relative path.

Example:

```text
generated-assets/approved/card/6d6f.../portrait/v1/1e25....png
```

Do not store absolute Windows paths as canonical database URIs.

If the local asset service needs absolute paths, derive them from configuration:

```text
ASSET_STORAGE_ROOT=C:\dev\wishes\repos\wishes-game
```

Then resolve:

```text
{ASSET_STORAGE_ROOT}/{storage_uri}
```

## Prompt JSON Shape

Store prompts as structured JSONB.

```json
{
  "positive": "full positive prompt",
  "negative": "full negative prompt",
  "style": "Wishes painterly fantasy card art",
  "role_instruction": "full body standing illustration",
  "identity_constraints": [
    "same face as approved portrait",
    "same hairstyle",
    "same outfit color palette"
  ],
  "source_summary": "approved portrait v1",
  "user_notes": "optional user request",
  "generated_from": {
    "card_uuid": "...",
    "template_uuid": "..."
  }
}
```

## Generation Metadata Shape

```json
{
  "comfyui": {
    "prompt_id": "...",
    "workflow_code": "full_body_from_portrait",
    "workflow_version": 1,
    "workflow_path": "server/asset-service/workflows/full_body_from_portrait.json"
  },
  "model_stack": {
    "checkpoint": "flux1-schnell",
    "vae": null,
    "loras": [],
    "controlnets": [],
    "ipadapter": true,
    "redux": false
  },
  "settings": {
    "seed": 123456789,
    "steps": 8,
    "cfg": 1.0,
    "sampler": "euler",
    "scheduler": "simple",
    "width": 1024,
    "height": 1536
  },
  "source": {
    "source_asset_uuid": "...",
    "source_asset_role": "portrait",
    "source_asset_version": 1
  },
  "outputs": {
    "original_comfy_output": "ComfyUI_00001_.png",
    "normalized_output": "generated-assets/pending/card/.../full_body/v1/...png"
  }
}
```

## Review Metadata Shape

```json
{
  "review_notes": "Looks consistent with portrait.",
  "issues": [],
  "identity_score": null,
  "manual_overrides": [],
  "approved_for_roles": ["full_body"]
}
```

## Publication Metadata Shape

```json
{
  "published_uri": "generated-assets/published/card/.../full_body/...png",
  "published_at": "2026-07-08T00:00:00Z",
  "publish_target": "local-dev",
  "cdn_uri": null
}
```

## Versioning Rules

1. First generated asset for an object and role is version 1.
2. A revision increments the role version.
3. Replacing an approved portrait increments portrait version.
4. Derived assets store the version of their source at generation time.
5. If source version changes, descendants become stale.
6. Stale assets remain visible in history but should not be current.
7. Rejected assets retain their version and history but are not current.
8. Published assets may remain published until replaced, but UI should warn if they are stale.

## Current Asset Rules

When an asset is approved as current:

1. Set other current assets for same object and role to `is_current = false`.
2. Set approved asset to `is_current = true`.
3. Set `approved_at` and `approved_by`.
4. Create an `asset_review_event`.
5. If role is `portrait`, mark descendants of previous portrait stale.

This should be done in a transaction.

## Stale Invalidation Algorithm

When asset A is superseded:

1. Find all dependency descendants using `asset_dependency`.
2. Mark descendants `status = 'stale'` where status is `approved` or `published` or `review_pending`.
3. Set `is_current = false` for stale descendants.
4. Create `asset_review_event` rows with event type `stale`.
5. Optionally queue regeneration jobs for required roles.

Pseudo SQL recursive lookup:

```sql
WITH RECURSIVE descendants AS (
  SELECT child_asset_uuid
  FROM asset_dependency
  WHERE parent_asset_uuid = $1

  UNION ALL

  SELECT d.child_asset_uuid
  FROM asset_dependency d
  JOIN descendants x ON d.parent_asset_uuid = x.child_asset_uuid
)
SELECT child_asset_uuid FROM descendants;
```

## Job Claiming Pattern

Workers should claim jobs transactionally.

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

## Required Database Functions

Implement helper functions only if they are reused by application services or multiple SQL operations.

Recommended public functions:

```text
wishes_asset_mark_current
wishes_asset_mark_stale_descendants
wishes_asset_create_review_event
```

Keep ComfyUI-specific orchestration in the programming layer, not SQL.

### wishes_asset_mark_current

Responsibilities:

- Validate asset exists.
- Clear current flag for object/role.
- Mark target asset current.
- Set approval fields.
- Write review event.

### wishes_asset_mark_stale_descendants

Responsibilities:

- Find descendants recursively.
- Mark descendants stale.
- Clear current flag.
- Write stale events.

### wishes_asset_create_review_event

Responsibilities:

- Insert review/audit event.
- Standardize event payload.

## Storage Operations

The asset service must normalize file movement.

Candidate generation:

```text
ComfyUI output -> generated-assets/pending/...
```

Approval:

```text
generated-assets/pending/... -> generated-assets/approved/...
```

Rejection:

```text
generated-assets/pending/... -> generated-assets/rejected/...
```

Publication:

```text
generated-assets/approved/... -> generated-assets/published/...
```

Archival:

```text
current location -> generated-assets/archived/...
```

Prefer copy then verify then update database, rather than destructive move first.

## File Validation

Before inserting or updating asset records, validate:

- File exists.
- File extension allowed by asset role boundary.
- MIME type matches extension.
- File size is greater than zero.
- Width and height are readable.
- Width and height match or are compatible with boundaries.
- Source asset exists for derived roles.
- Source asset is approved or published.

## Data Integrity Notes

Avoid cascade deleting approved asset history when possible. `source_asset_uuid` uses `ON DELETE SET NULL` so historical assets can survive if source records are removed, although deletion of asset rows should generally be avoided.

Use archive instead of delete for most asset lifecycle changes.

## Dev Seed File

Create a dev seed for roles and workflows:

```text
database/dev-seed/210_asset_types_and_roles.sql
```

If seed numbering around 200 is reserved, use the next available System Data Layer seed number.

Include:

- asset_type values
- asset_role values
- asset_workflow registry values

## Validation Queries

After migration and seed execution:

```sql
SELECT code, name FROM asset_type ORDER BY code;
SELECT code, asset_type_code, name FROM asset_role ORDER BY sort_order;
SELECT code, asset_role_code, workflow_path FROM asset_workflow ORDER BY code;
```

Create a test asset graph:

```text
portrait -> full_body -> tactical_sprite_sheet
portrait -> icon
portrait -> thumbnail
portrait -> card_front
```

Then validate descendant lookup returns all derived assets.

## Acceptance Criteria

This database/storage implementation is complete when:

1. Asset type and role tables exist.
2. Workflow registry exists.
3. Asset table supports lineage, versioning, prompt metadata, review metadata, and publication metadata.
4. Generation job table supports queue processing.
5. Review event table stores audit history.
6. Dependency table supports descendant lookup.
7. Required indexes exist.
8. Dev seeds populate required roles.
9. Local storage folders exist or are created by service startup.
10. Approval can mark one current asset per object/role.
11. Portrait replacement can mark descendants stale.
12. All migrations run from clean reset without errors.
