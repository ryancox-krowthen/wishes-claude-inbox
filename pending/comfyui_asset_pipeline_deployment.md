# ComfyUI Asset Pipeline Deployment

## Objective
Implement a multi-stage ComfyUI asset generation pipeline for the Wishes project.

## Requirements
1. Generate a portrait from a text prompt.
2. Present the portrait for human approval.
3. Treat the approved portrait as the canonical visual source.
4. Generate all downstream assets from the approved portrait using image-reference workflows.

## Assets
- Card front
- Full body illustration
- Icon
- Thumbnail
- Tactical sprite sheet (idle, walk, attack, cast, hit, death)

## Workflow
1. Portrait Generation
2. Human Approval
3. Full Body Generation
4. Card Front Composition
5. Icon Generation
6. Thumbnail Creation
7. Tactical Sprite Generation
8. Asset Review
9. Publish Approved Assets

## Technical Notes
- Use reference-image conditioning (IPAdapter/Redux or equivalent Flux-compatible workflow).
- Store source_asset_uuid on every generated asset.
- Mark downstream assets stale if the portrait changes.
- Keep workflows modular (one JSON workflow per asset type).
- Integrate with the existing Wishes asset queue and approval workflow.
