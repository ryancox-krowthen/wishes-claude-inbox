# Wishes ComfyUI Asset Pipeline Deployment

## Purpose

Implement a complete ComfyUI-driven asset generation and approval system for Wishes.

This task creates a canonical portrait-first art pipeline:

1. Generate a portrait from a prompt.
2. Review and approve exactly one portrait as the canonical visual source.
3. Use the approved portrait to generate all downstream assets.
4. Keep all downstream assets visually similar to the approved portrait.
5. Store asset lineage, prompts, versions, approvals, and dependency status.
6. Integrate with the existing Wishes asset tables, generated asset folders, and web portal.

## Required Output Assets

For each character/card visual identity, support generation and approval of:

- Approved portrait
- Card front
- Full body illustration
- Icon
- Thumbnail
- Tactical sprite sheet

Optional later extensions:

- Emoji/expression set
- Turnaround sheet
- Marketing art
- Character-specific LoRA
- 3D model reference sheet

## Core Rule

The approved portrait is the canonical visual source.

No downstream asset should be generated only from text unless it is explicitly a revision candidate. Every approved downstream asset must retain a source reference to the approved portrait or to another approved asset derived from that portrait.

## Folder Contents

- `01_system_architecture.md` — architecture, dependency graph, states, and asset lifecycle.
- `02_database_and_storage.md` — database schema, indexes, storage layout, lineage, and invalidation rules.
- `03_comfyui_workflows.md` — workflow structure, ComfyUI standards, prompt injection, output rules.
- `04_backend_and_api.md` — services, queue worker, API routes, status transitions, validation.
- `05_web_portal_review.md` — review UI, approval flow, revision flow, sprite review.
- `06_claude_execution_checklist.md` — consolidated Claude Code implementation checklist.

## Execution Policy

Claude must implement this in the main Wishes repository, not in this inbox repository.

This inbox folder is a task payload. After consuming it, Claude should copy or reference these instructions in the Wishes repo documentation only if useful.

Claude must not commit or push changes to any repository unless explicitly authorized by Ryan.
