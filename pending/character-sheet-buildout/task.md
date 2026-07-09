# Character Sheet Build-Out

Created: 2026-07-08
Priority: high
Mode: implement

Allow Edit: true
Allow Commit: false
Allow Push: false
Allow Delete: false
Allow Asset Import: false
Allow File Copy: false

## Objective

Build out the Character Creator's Character Sheet to present the full sheet
structure defined in `references/character sheet.txt`: Status, Core Stats,
Elements, and Stats sections.

## Scope

- `prototypes/character-creator/` (client `CharacterSheet` + supporting API).
- Sections and fields per the reference file:
  - **Status**: Health (Pool).
  - **Core Stats**: Physical (Might, Dexterity, Constitution), Mental
    (Intelligence, Wit, Judgement), Innate (Aptitude, Deception), Charisma.
  - **Elements**: grouped by Master Element (Universal, Divine, Astral,
    Spiritual, Ethereal, Physical) with their sub-elements.
  - **Stats**: Speed, Interaction (Pool), Reaction (Pool), Accuracy, Dodge.

## Steps

1. Review `references/character sheet.txt` and current `CharacterSheet` UI.
2. Verify element grouping against Core Concept 1 (canon wins over the note's
   spelling/ordering).
3. Extend the API/`wishes_character` output as needed to supply the fields.
4. Implement the sheet sections in the client.
5. Validate with representative characters (Human, Elf, Dwarf, Chosen One).

## Expected Output

Character Sheet renders all four sections with live data for an existing
character.

## Safety Rules

- No commits or pushes without explicit user approval.

## Notes

- Source note was submitted as root-level `character sheet.txt` in the external
  inbox repo before the pending/ bundle format existed; wrapped into this bundle
  2026-07-08. Original text preserved verbatim in `references/`.
- The note contains spelling variants (Pysical, Consitution, Charaisma) — treat
  canon and existing stat naming as authoritative.
