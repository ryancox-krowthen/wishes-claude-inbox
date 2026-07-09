# 01 — ComfyUI Asset Pipeline System Architecture

## Objective

Build a portrait-first asset generation system for Wishes using ComfyUI.

The system must support this canonical flow:

```text
Character or card concept
  -> Prompt package
  -> Portrait generation
  -> Human review
  -> Approved portrait
  -> Derived asset generation
  -> Human review
  -> Published asset set
```

The approved portrait is the visual authority for the character/card identity. All downstream assets must be visually similar to that portrait unless the user explicitly requests a redesign.

## Architectural Rules

1. The server remains authoritative.
2. AI and ComfyUI never directly mutate canonical world state.
3. AI and ComfyUI may create asset candidates.
4. Human approval promotes candidates into approved assets.
5. Approved assets may be linked to cards, templates, decks, users, NPCs, or other world objects.
6. Every derived asset must preserve lineage back to its source portrait.
7. Asset generation must be repeatable from stored prompts, workflow metadata, seeds, and source asset references.
8. Portrait replacement invalidates downstream derived assets instead of silently replacing them.
9. Asset metadata must be stored separately from card gameplay state.
10. Card identity and ownership must remain independent from generated image assets.

These rules align with the Wishes server-authoritative model and the existing principle that clients render state while the server owns mutations.

## Core Concepts

### Asset Candidate

An asset candidate is a generated file that has not yet been approved.

Examples:

- A generated portrait awaiting review
- A full body image generated from an approved portrait
- A draft tactical sprite sheet
- A revised card front

Asset candidates are not canonical. They should be stored in pending storage and shown in review screens.

### Approved Asset

An approved asset is a candidate that has passed human review.

Approved assets may be displayed in the game, used in card rendering, referenced by downstream generation workflows, or published to final asset storage.

### Source Asset

A source asset is the asset used to condition another generation.

For this pipeline, the approved portrait is normally the source asset for:

- Full body
- Icon
- Thumbnail
- Card front
- Sprite base
- Tactical sprite sheet

A full body image may then become the source asset for sprite generation.

### Asset Lineage

Asset lineage is the chain of source references connecting a generated asset back to its visual authority.

Example:

```text
Approved Portrait
  -> Full Body
      -> Tactical Sprite Sheet
  -> Icon
  -> Thumbnail
  -> Card Front
```

Every generated asset must store enough lineage to answer:

- What source asset created this?
- What workflow created this?
- What prompt created this?
- What seed created this?
- What model/checkpoint/LoRA stack created this?
- What version of the source did this depend on?
- Is this asset stale because its source changed?

## Asset Lifecycle States

Use these states for asset records and generation jobs:

```text
requested
queued
in_progress
generated
review_pending
approved
rejected
published
stale
archived
failed
```

### requested

A user, service, or automation requested an asset.

### queued

The generation request has been normalized and is waiting for worker pickup.

### in_progress

A worker has claimed the job and submitted or is submitting work to ComfyUI.

### generated

ComfyUI produced an output file.

### review_pending

The generated file is ready for human review.

### approved

The asset has been accepted as usable.

### rejected

The asset was reviewed and rejected.

### published

The approved asset has been moved or copied to the stable published location.

### stale

The asset was previously valid but depends on a source asset that changed.

### archived

The asset is retained for history but should not be used for new gameplay or publishing.

### failed

Generation failed due to workflow, model, queue, IO, or ComfyUI errors.

## Portrait-First Pipeline

### Stage 1: Prompt Package

The system prepares a prompt package from available character/card data.

Inputs may include:

- Card name
- Card type
- Race
- Gender
- Body type
- Profession
- Class
- Title
- Beginnings
- Virtue
- Fault
- Desire
- Core elements
- Quality
- Personality notes
- Equipment notes
- Visual style notes

The prompt package must be stored for audit and reproducibility.

### Stage 2: Portrait Generation

The portrait workflow generates one or more portrait candidates from text prompt data.

No downstream asset generation may begin until one portrait is approved.

### Stage 3: Portrait Approval

A human selects the approved portrait.

Approval effects:

- Candidate becomes approved portrait.
- Candidate is copied or moved to approved storage.
- Portrait version is incremented.
- Any previous portrait may be archived.
- Existing downstream assets become stale if the approved portrait changes.

### Stage 4: Derived Asset Generation

Derived workflows use the approved portrait as image reference input.

Required downstream assets:

- Card front
- Full body
- Icon
- Tactical sprite sheet
- Thumbnail

Recommended dependency chain:

```text
Approved Portrait
  -> Card Front
  -> Icon
  -> Thumbnail
  -> Full Body
      -> Tactical Sprite Sheet
```

The full body asset should usually be generated before the tactical sprite sheet because sprite generation benefits from full outfit/body information.

### Stage 5: Derived Asset Review

Each derived asset must be reviewed separately.

Review must show:

- Approved portrait source
- Candidate asset
- Prompt
- Negative prompt
- Workflow
- Seed
- Model stack
- Source lineage
- Version

### Stage 6: Publication

Published assets are stable game-facing files.

Publishing should not delete candidate or review history.

## Asset Dependency Graph

The asset system should treat derived assets as a graph, not a flat list.

Minimum graph fields:

- `asset_uuid`
- `source_asset_uuid`
- `root_asset_uuid`
- `object_type`
- `object_uuid`
- `asset_role`
- `asset_version`
- `source_asset_version`
- `status`

Example graph:

```text
portrait:v1 [approved]
  card_front:v1 [approved]
  icon:v1 [approved]
  thumbnail:v1 [approved]
  full_body:v1 [approved]
    tactical_sprite_sheet:v1 [review_pending]
```

If portrait v2 is approved:

```text
portrait:v1 [archived]
portrait:v2 [approved]
  card_front:v1 [stale]
  icon:v1 [stale]
  thumbnail:v1 [stale]
  full_body:v1 [stale]
    tactical_sprite_sheet:v1 [stale]
```

Then the system can queue regeneration of stale descendants.

## Asset Roles

Required roles:

```text
portrait
card_front
full_body
icon
thumbnail
tactical_sprite_sheet
```

Recommended later roles:

```text
sprite_idle
sprite_walk
sprite_attack
sprite_cast
sprite_hit
sprite_down
sprite_guard
sprite_interact
emoji_happy
emoji_angry
emoji_sad
emoji_surprised
turnaround_front
turnaround_back
turnaround_left
turnaround_right
model_reference
```

## Object References

Assets should be attachable to multiple object types without hard-coding card-only behavior.

Supported initial object types:

```text
card
card_template
deck
user
npc
character
```

Recommended reference shape:

```json
{
  "object_type": "card",
  "object_uuid": "...",
  "asset_role": "portrait"
}
```

## Quality Gates

Before an asset can be approved, validate:

1. File exists.
2. File can be loaded.
3. File type matches asset role constraints.
4. Dimensions meet role boundaries.
5. Source asset exists for derived assets.
6. Source asset is approved.
7. Workflow metadata is present.
8. Prompt metadata is present.
9. Asset role is valid.
10. Asset is not already superseded.

## Consistency Requirements

Derived assets must preserve:

- Same face
- Same hair color
- Same hairstyle unless prompted otherwise
- Same outfit/armor theme
- Same dominant color palette
- Same species/race features
- Same age presentation
- Same art style
- Same major accessories
- Same silhouette where practical

The system should allow manual overrides, but overrides must be stored in generation metadata.

## Style Authority

The approved portrait controls character identity, but the global Wishes style still controls output presentation.

A downstream prompt should combine:

```text
Wishes global art style
+ approved portrait visual identity
+ asset role requirements
+ card/character-specific details
```

## Workflow Architecture

Do not build one giant ComfyUI workflow for everything.

Use modular workflow JSON files:

```text
portrait_generate.json
portrait_refine.json
full_body_from_portrait.json
card_front_from_portrait.json
icon_from_portrait.json
thumbnail_from_portrait.json
sprite_base_from_full_body.json
sprite_sheet_from_sprite_base.json
emoji_from_portrait.json
```

Each workflow should declare:

- Asset role
- Required inputs
- Optional inputs
- Output directory
- Output naming pattern
- Prompt node identifiers
- Negative prompt node identifiers
- Seed node identifiers
- Image input node identifiers
- Width/height node identifiers
- Model/LoRA node identifiers

## Naming Rules

Use predictable lowercase paths.

Example generated file pattern:

```text
generated-assets/pending/{object_type}/{object_uuid}/{asset_role}/v{version}/{job_uuid}.png
```

Example approved file pattern:

```text
generated-assets/approved/{object_type}/{object_uuid}/{asset_role}/v{version}/{asset_uuid}.png
```

Example published file pattern:

```text
generated-assets/published/{object_type}/{object_uuid}/{asset_role}/{asset_uuid}.png
```

Do not depend on original ComfyUI output names as canonical filenames. ComfyUI output names should be captured as metadata but normalized by the asset service.

## Service Boundaries

### Asset API

Receives user and portal requests.

Responsibilities:

- Create generation request
- Validate object references
- Store prompt package
- Enqueue job
- Expose asset state
- Approve/reject assets
- Publish assets

### Asset Worker

Processes queued generation jobs.

Responsibilities:

- Claim queued job
- Load workflow definition
- Inject prompt/source/seed/settings
- Submit to ComfyUI
- Poll for result
- Normalize output files
- Store metadata
- Mark review pending

### ComfyUI Dispatcher

Thin integration layer for ComfyUI HTTP/WebSocket APIs.

Responsibilities:

- Submit workflow graph
- Track prompt id
- Collect output metadata
- Download/copy output files
- Report failures

### Dependency Resolver

Maintains asset graph health.

Responsibilities:

- Mark descendants stale
- Queue descendant regeneration
- Prevent use of stale assets as canonical published assets
- Show lineage in review UI

### Review UI

Human-facing approval system.

Responsibilities:

- Show source portrait
- Show generated candidate
- Show prompts and metadata
- Approve/reject/revise/regenerate
- Compare versions
- Preview sprite sheets

## Failure Handling

Generation failures must never corrupt asset state.

On failure:

1. Mark job `failed`.
2. Store error message.
3. Store workflow id and prompt id if available.
4. Preserve inputs.
5. Allow retry.
6. Do not create approved assets.
7. Do not invalidate existing approved assets.

## Audit Requirements

Every approval, rejection, revision, and publication should record:

- Actor
- Timestamp
- Previous status
- New status
- Reason/comment
- Asset UUID
- Job UUID

## Implementation Priority

Implement in this order:

1. Database schema and role tables.
2. Storage path conventions.
3. Asset request API.
4. ComfyUI workflow registry.
5. Portrait generation job.
6. Portrait review and approval.
7. Full body generation from approved portrait.
8. Icon and thumbnail generation.
9. Card front generation.
10. Sprite sheet generation.
11. Dependency invalidation.
12. Regeneration queue.
13. Review UI polish.

## Acceptance Criteria

The first complete milestone is successful when:

1. A user can request a portrait from text.
2. ComfyUI generates a portrait candidate.
3. The candidate appears in the review UI.
4. A user can approve it.
5. The approved portrait becomes the visual source.
6. The system can generate full body, icon, thumbnail, card front, and tactical sprite sheet candidates from that portrait.
7. Each derived asset stores `source_asset_uuid`.
8. Each derived asset can be approved or rejected.
9. Published assets are available from stable paths.
10. Replacing the portrait marks downstream assets stale.
