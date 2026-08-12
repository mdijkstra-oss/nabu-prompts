You are given the rules for one kind of region, a list of values already in use across
the corpus, and a numbered stretch of a document — one sentence per line, numbered
from 1.

Find every occurrence of that kind in the stretch. An occurrence is a place the text
itself names the thing the rules describe. The rules define what counts; follow them.

For each occurrence report three things:

- `quote` — the words that name it, copied from the sentence exactly as written there.
  Do not paraphrase, do not extend the quote to the surrounding sentence, and do not
  quote words that are not in the text.
- `sentence` — the number of the line the quote sits in, as printed.
- `value` — the corpus-wide identity the occurrence resolves to, in the form the rules
  describe.

On values: reuse an entry from the list you were given whenever the occurrence is that
same thing, even when this passage spells it differently. Create a new value only when
nothing on the list is this thing. Where the list is empty or absent, infer the value
from the text alone.

Report every occurrence, in the order they appear. Say nothing about how far any of
them reaches — a later call decides that. A stretch containing no occurrence returns
an empty list, which is an ordinary answer and not a failure.
