# Update Wishes Canon Workflow

Created: 2026-07-12
Priority: High
Mode: implementation

Allow Edit: true
Allow Commit: true
Allow Push: true
Allow Delete: false
Allow Asset Import: false
Allow File Copy: false

## Objective

Update the repository workflow and `CLAUDE.md` in `Krowthen/wishes-canon` to establish the official canonical development process for the Wishes Canon repository.

## Scope

- Repository: `Krowthen/wishes-canon`
- Target branch: `main`
- Feature branch: `canon/repository-workflow` (or another clear `canon/<topic>` name if this branch already exists)
- Files to review and update:
  - `CLAUDE.md`
  - `README.md`
- This is a documentation-only change. Do not edit canon content or generated publication artifacts.

## Steps

1. Pull the latest `main`.
2. Create a feature branch named `canon/repository-workflow`, or another clear `canon/<topic>` name if necessary.
3. Review the existing `CLAUDE.md` and `README.md`.
4. Update `CLAUDE.md` to establish the official canon workflow.
5. State explicitly that Markdown files under `source/` are the authoritative source of truth for canon.
6. Require every canon edit to be made on a feature branch.
7. Require every change to be merged into `main` through a Pull Request.
8. Document the preferred workflow in this order:
   - Pull the latest `main`.
   - Create a feature branch named `canon/<topic>`.
   - Edit Markdown files only.
   - Commit the changes.
   - Push the feature branch.
   - Open a Pull Request targeting `main`.
   - Resolve all review conversations.
   - Squash Merge the Pull Request.
   - Automatically delete the feature branch after merge.
9. Clarify that generated DOCX and PDF files are publication artifacts. They must be regenerated from authoritative Markdown and must never be manually edited.
10. Add a clearly labeled optional future CI validation section covering:
    - Markdown linting
    - Link checking
    - Pandoc publication builds
11. Update any workflow or contribution guidance in `README.md` so it matches `CLAUDE.md` and does not contradict the canonical process.
12. Review the final diff for internal consistency and documentation-only scope.
13. Commit the changes using this exact commit message:

    `docs(canon): update repository workflow and contribution process`

14. Push the feature branch and open a Pull Request targeting `main`.
15. Do not merge the Pull Request unless separately authorized.

## Expected Output

- Updated `CLAUDE.md` defining the official canon workflow.
- Updated `README.md` with matching workflow and contribution guidance.
- No manual changes to generated DOCX or PDF files.
- A pushed `canon/<topic>` feature branch.
- A Pull Request targeting `main`.
- A completion report containing:
  - Branch name
  - Commit SHA
  - Pull Request URL
  - Files changed
  - Validation performed
  - Any unresolved review conversations or blockers

## Safety Rules

- Preserve unrelated changes.
- Do not modify files outside the documented scope unless a directly related workflow reference must be corrected; report any such additional file.
- Do not edit canon content under `source/` as part of this task.
- Do not edit DOCX or PDF publication artifacts.
- Do not push directly to `main`.
- Do not force-push.
- Do not merge the Pull Request without separate authorization.
- Do not delete files or branches manually.

## Notes

The repository documentation should distinguish authoritative source material from generated publication output. The desired long-term process is Markdown-first, review-driven, and reproducible through future automated validation and publication builds.
