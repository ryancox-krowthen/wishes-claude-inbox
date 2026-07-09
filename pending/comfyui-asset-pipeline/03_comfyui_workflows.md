# 03 — ComfyUI Workflow Implementation Specification

## Objective

Define the ComfyUI workflow strategy for the Wishes asset pipeline.

This document tells Claude Code how to organize workflow JSON files, inject runtime values, preserve visual consistency, and generate all required assets from an approved portrait.

Required assets:

- Portrait
- Card front
- Full body
- Icon
- Thumbnail
- Tactical sprite sheet

Primary rule:

```text
Only the portrait is generated from text alone. All downstream assets must use the approved portrait or an approved derivative as reference input.
```

## Repository Placement

Place workflow files in the main Wishes repository:

```text
server/asset-service/workflows/
```

Recommended structure:

```text
server/asset-service/
  workflows/
    portrait_generate.json
    portrait_refine.json
    full_body_from_portrait.json
    card_front_from_portrait.json
    icon_from_portrait.json
    thumbnail_from_portrait.json
    sprite_base_from_full_body.json
    sprite_sheet_from_sprite_base.json
    emoji_from_portrait.json
    README.md
  workflow-manifests/
    portrait_generate.manifest.json
    full_body_from_portrait.manifest.json
    card_front_from_portrait.manifest.json
    icon_from_portrait.manifest.json
    thumbnail_from_portrait.manifest.json
    sprite_sheet_from_sprite_base.manifest.json
```

Do not hard-code node IDs only in source code. Store node mappings in manifests or in `asset_workflow.injection_schema`.

## Workflow Manifest Standard

Each workflow must have a manifest file.

Example:

```json
{
  "code": "full_body_from_portrait",
  "name": "Full Body From Portrait",
  "asset_role": "full_body",
  "workflow_path": "server/asset-service/workflows/full_body_from_portrait.json",
  "workflow_version": 1,
  "requires_source_asset": true,
  "preferred_source_role": "portrait",
  "output_role": "full_body",
  "injection_schema": {
    "positive_prompt_node": "6",
    "negative_prompt_node": "7",
    "seed_node": "25",
    "width_node": "12",
    "height_node": "13",
    "source_image_node": "31",
    "output_prefix_node": "42"
  },
  "default_settings": {
    "width": 1024,
    "height": 1536,
    "steps": 12,
    "cfg": 1.0,
    "sampler": "euler",
    "scheduler": "simple"
  }
}
```

## Workflow Loading Rules

The asset worker should:

1. Load the workflow registry row from `asset_workflow`.
2. Load the workflow JSON file.
3. Load the matching manifest or injection schema.
4. Validate all required nodes exist.
5. Inject prompt, source image, seed, dimensions, output prefix, and optional model settings.
6. Submit the modified workflow to ComfyUI.
7. Capture the ComfyUI `prompt_id`.
8. Poll or subscribe until generation finishes.
9. Copy outputs into normalized Wishes storage.
10. Store workflow input/output metadata on the asset/job.

## Runtime Injection Requirements

The asset worker must support injecting:

- Positive prompt
- Negative prompt
- Seed
- Width
- Height
- Source image path
- Output filename prefix
- LoRA stack
- Model/checkpoint selection
- Optional ControlNet image
- Optional mask image
- Optional pose image
- Optional reference strength
- Optional denoise value

The service must fail fast if a workflow requires a source image but no approved source asset is available.

## Prompt Composition

The prompt should be assembled from structured parts, not a single hard-coded string.

Recommended positive prompt structure:

```text
{global_style}
{asset_role_instruction}
{character_identity}
{equipment_and_clothing}
{color_palette}
{composition_instruction}
{consistency_instruction}
{quality_instruction}
```

Recommended negative prompt structure:

```text
{global_negative}
{asset_role_negative}
{identity_negative}
{technical_negative}
```

## Global Wishes Style Prompt

Use this as the baseline style until a better style guide exists:

```text
high quality fantasy RPG character art, painterly digital illustration, clean readable silhouette, detailed but not noisy, expressive face, elegant fantasy design, cohesive color palette, game-ready concept art, consistent lighting, polished card-game illustration
```

## Global Negative Prompt

```text
low quality, blurry, distorted anatomy, duplicate face, extra limbs, missing limbs, malformed hands, inconsistent eyes, crossed eyes, bad proportions, unreadable silhouette, text artifacts, watermark, logo, signature, cropped head, cropped feet unless requested, inconsistent clothing, modern objects, photorealistic style unless requested
```

## Identity Preservation Prompt Block

For downstream assets, always include:

```text
same character as the approved portrait, same face, same hairstyle, same hair color, same eye color, same age presentation, same species traits, same outfit theme, same dominant colors, same art style, preserve identity from the reference portrait
```

## Portrait Generation Workflow

### File

```text
portrait_generate.json
```

### Purpose

Generate portrait candidates from text.

### Inputs

- Character/card prompt package
- Style prompt
- Negative prompt
- Seed
- Width/height
- Optional quality/element prompt

### Output

- One or more portrait candidates

### Recommended dimensions

```text
1024 x 1024
```

### Prompt role instruction

```text
front-facing or three-quarter fantasy RPG character portrait, upper body and face visible, centered composition, clear facial features, character identity suitable for future full body and sprite generation
```

### Approval behavior

No downstream workflow may execute until a portrait is approved.

## Portrait Refinement Workflow

### File

```text
portrait_refine.json
```

### Purpose

Create revised portrait candidates using an existing portrait as reference.

### Inputs

- Existing portrait candidate or approved portrait
- Revision notes
- Denoise setting
- Seed

### Use cases

- Fix eyes
- Improve hair consistency
- Adjust expression
- Improve costume detail
- Preserve composition but improve quality

### Important

A refined portrait is not automatically approved. It must return to review.

## Full Body From Portrait Workflow

### File

```text
full_body_from_portrait.json
```

### Purpose

Generate a full body character image based on the approved portrait.

### Required source

```text
asset_role = portrait
status = approved or published
```

### Output role

```text
full_body
```

### Recommended dimensions

```text
1024 x 1536
```

### Prompt role instruction

```text
full body standing character illustration, show head to toe, same character as reference portrait, complete outfit visible, boots visible, hands visible, weapon or accessories visible if applicable, neutral readable pose for later sprite generation
```

### Negative additions

```text
cropped feet, cropped hands, different face, different outfit, different hair, helmet hiding face unless source portrait has helmet, back turned, extreme action pose
```

### Reference strategy

Use whichever ComfyUI reference strategy is available and stable locally:

- IPAdapter
- Redux
- Flux reference workflow
- Flux Kontext when available
- ControlNet pose plus portrait reference

The workflow should prioritize identity preservation over dramatic composition.

## Icon From Portrait Workflow

### File

```text
icon_from_portrait.json
```

### Purpose

Create square icon assets from the approved portrait.

### Recommended approach

Prefer deterministic crop/resize/sharpen over regeneration.

Only use AI regeneration if:

- The portrait crop is unsuitable.
- The user requests a stylized icon.
- The icon requires a special frame/background.

### Dimensions

```text
512 x 512
```

### Output role

```text
icon
```

### Prompt role instruction if AI is used

```text
square RPG character icon, same character as approved portrait, clear face, readable at small size, simple background, strong silhouette
```

## Thumbnail From Portrait Workflow

### File

```text
thumbnail_from_portrait.json
```

### Purpose

Create small preview thumbnails.

### Recommended approach

This should be implemented as image processing, not AI generation.

Operations:

1. Load approved portrait.
2. Center crop if needed.
3. Resize to target dimensions.
4. Save optimized PNG or WebP.

### Dimensions

```text
256 x 256
```

### Output role

```text
thumbnail
```

## Card Front From Portrait Workflow

### File

```text
card_front_from_portrait.json
```

### Purpose

Generate or compose a full card front using approved portrait art.

### Preferred architecture

Separate character art generation from card layout composition.

Recommended card front pipeline:

```text
approved portrait/full body
  -> card art crop or extension
  -> deterministic card frame composition
  -> text/stat/element overlay rendering
  -> exported card_front image
```

### Important

Do not rely on AI to render readable card text. AI-generated text is not reliable.

Card text, stats, names, quality icons, and element symbols should be rendered by the application layer using deterministic canvas/SVG/image composition.

### Recommended dimensions

```text
768 x 1075
```

### Card composition inputs

- Portrait or full body art
- Card name
- Quality
- Element/domain tags
- Frame style
- Stats
- Card type
- Optional rarity effects

### Output role

```text
card_front
```

## Sprite Base From Full Body Workflow

### File

```text
sprite_base_from_full_body.json
```

### Purpose

Convert approved full body art into a simplified tactical character base suitable for sprite-sheet generation.

### Required source

Preferred:

```text
asset_role = full_body
status = approved or published
```

Fallback:

```text
asset_role = portrait
status = approved or published
```

### Output

A clean sprite base or character reference sheet.

### Prompt role instruction

```text
small tactical RPG character sprite base, same character as reference, simplified readable outfit, chibi tactical proportions only if configured, clean silhouette, front-facing neutral stance, transparent or simple background
```

### Dimensions

Recommended working canvas:

```text
512 x 512
```

Final sprite extraction may use:

```text
128 x 128 per frame
```

## Sprite Sheet From Sprite Base Workflow

### File

```text
sprite_sheet_from_sprite_base.json
```

### Purpose

Create a tactical sprite sheet for turn-based tactical battles.

### Required animations

```text
idle
walk
attack
cast
hit
down
```

### Recommended later animations

```text
guard
interact
run
celebrate
use_item
```

### Required directions

Start with four directions:

```text
front
back
left
right
```

Optional future expansion:

```text
front_left
front_right
back_left
back_right
```

### Recommended layout

Rows = animations.
Columns = frames.
Separate sheets may be generated per direction if easier.

Example:

```text
row 1: idle frames
row 2: walk frames
row 3: attack frames
row 4: cast frames
row 5: hit frames
row 6: down frames
```

### Frame target

```text
128 x 128
```

### Sprite metadata

Store metadata describing frame positions:

```json
{
  "tile_width": 128,
  "tile_height": 128,
  "animations": {
    "idle": { "row": 0, "frames": 4 },
    "walk": { "row": 1, "frames": 8 },
    "attack": { "row": 2, "frames": 6 },
    "cast": { "row": 3, "frames": 6 },
    "hit": { "row": 4, "frames": 3 },
    "down": { "row": 5, "frames": 1 }
  }
}
```

## Emoji From Portrait Workflow

### File

```text
emoji_from_portrait.json
```

### Purpose

Future extension for small expression portraits.

### Expressions

```text
happy
angry
sad
surprised
confused
determined
injured
laughing
```

### Required source

```text
approved portrait
```

## LoRA Strategy

Do not require LoRA training for the first milestone.

However, prepare the architecture for character-specific LoRAs.

Recommended future flow:

```text
approved portrait
  -> generate 5-20 approved reference images
  -> train lightweight character LoRA
  -> attach LoRA to character visual identity
  -> use LoRA for all future downstream assets
```

Store LoRA metadata as an asset or workflow metadata item:

```json
{
  "lora_code": "character_6d6f_visual_v1",
  "source_assets": ["..."],
  "training_config": {},
  "model_path": "server/asset-service/models/loras/character_6d6f_visual_v1.safetensors"
}
```

## Model Stack Strategy

Initial local target:

```text
Flux1-Schnell or currently installed local Flux workflow
```

The implementation must not assume one permanent model. Store model stack in workflow metadata.

Supported model stack fields:

- checkpoint
- vae
- clip
- loras
- controlnets
- ipadapter
- redux
- sampler
- scheduler

## ComfyUI API Integration

Use ComfyUI HTTP API for submission and history retrieval.

Recommended endpoints:

```text
POST /prompt
GET /history/{prompt_id}
GET /view?filename=...&subfolder=...&type=output
```

Optional WebSocket:

```text
/ws
```

The dispatcher should support polling first. WebSocket can be added later.

## Workflow Output Normalization

ComfyUI output filenames are not canonical.

After generation:

1. Locate output image(s) from history.
2. Copy output to Wishes `generated-assets/pending/...` path.
3. Generate or assign asset UUID.
4. Store normalized URI.
5. Store original ComfyUI output metadata.

## Error Handling

Workflow execution fails if:

- Workflow JSON cannot be loaded.
- Manifest is missing.
- Required node ID is missing.
- Source image is missing.
- ComfyUI is unavailable.
- ComfyUI returns an error.
- Output cannot be found.
- Output cannot be copied.
- Output dimensions do not match boundary constraints.

On failure:

- Mark job failed.
- Store error JSON.
- Do not create approved asset.
- Preserve prompt and workflow input.
- Allow retry.

## Validation Checklist Per Workflow

For each workflow file:

- JSON loads successfully.
- Required node IDs exist.
- Positive prompt can be injected.
- Negative prompt can be injected.
- Seed can be injected.
- Width/height can be injected where relevant.
- Source image can be injected where required.
- Output prefix can be injected.
- ComfyUI accepts the graph.
- Output image is produced.
- Output image is normalized into Wishes storage.
- Asset record is created.
- Job record is updated.

## First Milestone Workflows

Implement these first:

1. `portrait_generate.json`
2. `full_body_from_portrait.json`
3. `icon_from_portrait.json`
4. `thumbnail_from_portrait.json`
5. `card_front_from_portrait.json`
6. `sprite_sheet_from_sprite_base.json`

## Acceptance Criteria

This workflow layer is complete when:

1. Workflows are stored under `server/asset-service/workflows`.
2. Each workflow has a manifest or registry injection schema.
3. A portrait can be generated from text.
4. A full body can be generated from approved portrait.
5. Icon and thumbnail can be generated from approved portrait.
6. Card front can be composed from approved art and deterministic overlays.
7. Sprite sheet generation can produce at least an initial tactical sheet candidate.
8. Every ComfyUI output is copied to normalized Wishes storage.
9. Every output stores workflow metadata.
10. Failed workflows leave retryable job records.
