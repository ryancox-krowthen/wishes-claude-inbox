# Inbox (local, gitignored)

Drop work requests here for Claude Code to pick up. Scanned at session start
(Workflow 2 in `docs/claude/workflows.md`).

## Convention

- **Each immediate child folder = one action item.**
- Inside a folder, put anything that describes the request: a `README`/notes,
  text files, images, links, sample data, etc.
- At session start (or on demand), Claude scans these folders, adds each as a
  task to `docs/claude/todo.md`, and asks what to do with each.
- Handled items are tracked in the to-do list; inbox folders are not deleted
  unless you ask.

Example:

```
tools/claude/inbox/
  add-deck-cover-art/
    README.md        # what you want
    reference.png    # optional supporting files
  fix-portrait-prompt/
    notes.txt
```

This folder is under gitignored `tools/claude/`, so nothing here is committed.
