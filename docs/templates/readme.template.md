# <package or app name>

One paragraph: what this is, and what it is responsible for. If someone can't tell from
this paragraph whether their change belongs here, rewrite it.

## Responsibilities

- What this owns.
- **Not** responsible for: X (that lives in `…`).

The second list prevents scope creep more effectively than the first.

## Quick start

```bash
<install>
<run>
```

## Commands

| Command | Description |
|---|---|
| `…` | … |

## Structure

```text
src/
├── …
```

One line per directory, explaining why it exists — not what it obviously contains.

## Key decisions

- [ADR-NNNN](<path-to-repo-root>/docs/decisions/NNNN-….md) — why <thing> is the way it is.

Replace `<path-to-repo-root>` with the relative path from this README to the repository
root — `../..` for `apps/<name>/README.md`, `../../..` one level deeper. Do not copy the
traversal from another README without checking its depth.

## Gotchas

Things that will surprise someone. Non-obvious ordering constraints, a rule that looks
arbitrary but is not, a place where the obvious change is wrong. If you explained it in
review once, it belongs here.

## Testing

```bash
<test command>
```

What the tests here actually cover, and what they deliberately do not.
