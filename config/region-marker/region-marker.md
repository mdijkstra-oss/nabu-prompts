You are given the rules for one kind of region, a numbered window of a document — one
sentence per line, numbered from 1 — and one occurrence inside it, named by its
sentence number and by the words it was found by.

The occurrence is already located and is not in doubt. Decide only how far it reaches:
which run of sentences in this window belongs to it. The rules say what belonging
means for this kind; follow them.

Return the first and last sentence number of that run, as printed. The run must be
contiguous and must include the occurrence's own sentence. Where the text it owns runs
past the end of the window, end at the last line you were shown.
