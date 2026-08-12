An entry carries one `<occurrence n="…" ref="…">quote</occurrence>` child per
occurrence in it: `n` is the occurrence's ordinal within its entry, `ref` names the
sentence it was found in, and the words inside are the words it was found by.

Every occurrence is already located and is not in doubt. Decide only how far each one
reaches: which run of sentences in its entry belongs to it. The rules say what
belonging means for this kind; follow them.

Answer per occurrence, named by its entry id and its ordinal `n`. Return the first and
last sentence of its run as `start` and `end` refs in the `3.7` form. The run must be
contiguous and must include the occurrence's own sentence. Where the text it owns runs
past the end of its entry, end at the last line of that entry.

Example output:
```json
{
    "results": [
        { "entry": 1, "n": 1, "start": "1.2", "end": "1.6" },
        { "entry": 2, "n": 1, "start": "2.4", "end": "2.4" }
    ]
}
```
