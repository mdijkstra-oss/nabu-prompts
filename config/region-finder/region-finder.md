Find every occurrence of that kind in each entry. An occurrence is a place the text
itself names the thing the rules describe. The rules define what counts; follow them.

For each occurrence report three things:

- `quote` — the words that name it, copied from the sentence exactly as written there.
  Do not paraphrase, do not extend the quote to the surrounding sentence, and do not
  quote words that are not in the text.
- `ref` — the sentence the quote sits in, as a ref like `3.7`.
- `value` — the corpus-wide identity the occurrence resolves to, in the form the rules
  describe.

On values: reuse a value from the list you were given whenever the occurrence is that
same thing, even when this passage spells it differently. Create a new value only when
nothing on the list is this thing. Where the list is empty or absent, infer the value
from the text alone.

Answer for every entry you were shown — one result per entry, even when it contains
nothing. Report an entry's occurrences in the order they appear. Say nothing about how
far any of them reaches — a later call decides that. An entry containing no occurrence
returns an empty `occurrences` list, which is an ordinary answer and not a failure.

Example output:
```json
{
    "results": [
        {
            "entry": 1,
            "occurrences": [
                { "quote": "Mrs Devlin", "ref": "1.2", "value": "devlin" }
            ]
        },
        { "entry": 2, "occurrences": [] }
    ]
}
```
