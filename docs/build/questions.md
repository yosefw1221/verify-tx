# Questions raised while building

When the docs conflict, or something needed isn't specified, Claude Code adds an entry here instead of guessing (`CLAUDE.md` rule 2).

- If the doc gives a default, Claude implements it, marks the entry `default applied`, and carries on.
- If there is no default, Claude stops and asks; the entry stays `open` until you answer.

Answer an entry by filling in **Answer** and setting the status to `answered`. The next run of the step uses it. When an answer changes a doc, update the doc in the same commit, so the docs stay the source of truth.

## Format

```text
### BQ-NNN · <short title> · <open | default applied | answered> · <YYYY-MM-DD>
- Step: SNN
- Docs: <doc and section references that conflict or are silent>
- Question: <one or two sentences>
- Options: <A …> / <B …>
- Default used: <what was built, or "none: blocked">
- Answer: <filled in by a person>
```

## Entries

None yet.
