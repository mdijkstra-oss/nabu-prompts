<create-codebook-mechanics>
# Code structure

Every codebook code must be a `json-callout` block with type `codebook-code`. This is the only accepted format — when creating, editing, or importing codes, always produce `json-callout` blocks. If a user provides codes in another format, convert them.

Only `json-callout` blocks reach the coder. Each code under judgment is sent as its own message carrying that block and nothing else, so everything a coder needs is inside the block. Prose around the blocks is read by chat and by people, never by the coder — put groupings, group descriptions and organizational context there, and never a rule.

## The framework file

A codebook has one framework file holding the rules that apply whichever code is being judged: whether a passage may take more than one code, how codes relate, what is never coded, and how anything shared is handled — time, speakers, units of analysis. It is passed to every evaluation in full, ahead of the code under judgment, so it is the only place a shared rule can live where a coder will read it.

The framework file must contain no `json-callout` blocks. Coding rejects a framework file that holds any.

The framework names no code. A rule that only makes sense once you know the code list is not a shared rule — it belongs inside the code it bears on, restated against the passage.

Write the framework first. A codebook without one has its shared rules either duplicated into every code or sitting in prose the coder never sees.

## Converting an existing document

A document that already describes a coding scheme — a methods chapter, a codebook exported from another tool, a researcher's notes — becomes a framework file plus code files. Every rule in the source lands in exactly one of them; nothing is dropped and nothing is summarized away.

1. Sort the source's rules into shared and per-code. A rule a coder needs whichever code they are judging is shared; a rule that bears on one code only goes in that code.
2. Write the framework file from the shared rules.
3. Write one `json-callout` per code, each judgeable without knowing any other code exists. Where the source defines a code by contrast — "unlike X", "code as Y instead", "as with Z" — replace the contrast with the condition it stood for, stated against the passage.
4. Tag every file you produce.

Then report what moved where, and say plainly if any rule in the source could not be expressed without naming another code.

A file can hold multiple codes — the file is the grouping mechanism. Codes in the same file form a thematic group; the filename and heading describe the group.

| File | Codes |
|------|-------|
| `economic_factors.md` | Cost Concern, Resource Scarcity, Financial Barrier |
| `emotional_responses.md` | User Frustration, Satisfaction, Anxiety |
| `process_issues.md` | Workaround, Process Friction, Compliance |

Never duplicate a code across files — the system aggregates codes from all files without deduplication, so a code in two files appears twice in the UI. Splitting a codebook means moving code blocks to their new home and removing them from the source.

When creating a new file — splitting a large codebook, starting a thematic group — open with a heading and a short paragraph describing what this file covers and how it relates to the broader codebook. A heading alone is a label; a paragraph orients the reader (and the model during coding).

Do not create index or navigation files that list the split files — those references go stale the moment a file is renamed. Tag all related files with a shared tag (e.g., `#codebook`) so they stay grouped and discoverable without cross-file references.

Each code requires:
- **type** — always `"codebook-code"`
- **title** — short, descriptive name for the code
- **content** — definition, criteria, and examples as a JSON string (use `\n` for newlines)
- **color** — visual identifier
- **collapsed** — `false` for active codes, `true` for stable ones

## Creating codes

1. Write the framework file, if the codebook does not have one yet
2. Write the code file's prose — heading, group description, any organizational context
3. `add_callout` to place an empty block after the right prose anchor
4. `patch_callout` to populate the block's fields using the returned `block_id`

## Example value shape

```json-callout
{
  "type": "codebook-code",
  "title": "User Frustration",
  "color": "tomato",
  "collapsed": false,
  "content": "Expressions of dissatisfaction with the product, process, or experience.\n\nInclusion criteria:\n- Direct complaints about specific features or processes\n- Negative evaluations of experience quality\n- Expressions of annoyance or disappointment\n\nExclusion criteria:\n- Neutral descriptions of difficulty with no dissatisfaction expressed\n- Constructive suggestions without emotional valence\n\nExamples:\n- \"this is really annoying and I don't understand why it works this way\"\n- \"I gave up after the third attempt\"\n\nCounter-examples:\n- \"it took a few tries but I figured it out\" (neutral difficulty)\n- \"it would be better if the button were larger\" (constructive suggestion)"
}
```

## Color assignment

Pick colors that create strong visual distinction between codes:

| Category | Suggested colors |
|----------|-----------------|
| Emotions | tomato, red, crimson, pink |
| Actions | blue, cyan, sky, indigo |
| Themes | purple, violet, teal, green |
| Speakers | brown, bronze, gold, amber |

Check existing codes before assigning a color — avoid duplicates.
</create-codebook-mechanics>
