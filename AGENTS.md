## docs
Read `CONTEXT.md` before exploring.
Read the ADRs in `docs/adr/` that touch the work.
Name domain concepts with that glossary.
Name an ADR conflict before you override it.

## code
Name every project-defined function with `<verb>-<noun>`.
Use only `new`, `del`, `set`, or `get` as operation verbs.

Choose the verb by the function's effect:

- `new-*` creates a resource or value.
- `set-*` changes state, applies input, runs a workflow, or sends output.
- `get-*` reads or derives a value.
- `del-*` stops, closes, removes, or cleans up a resource.

Use these three exceptions only:

- Predicate functions end with `-p`.
- Common Lisp, library, FFI, macro-generated, and struct-generated names keep required names.
- Test declarations and test helpers may use descriptive names.

Write comments for a junior reviewer.
Code should be as easy to understand and review.
Use Conventional Commits and Branch.

## language style

Apply these rules to every chat, reply, document, and commit message:

- Use ASD-STE100 Simplified Technical English.
- Use active voice and present tense.
- Write short sentences.
- Express one idea per sentence.
- Put each sentence on its own line.
- Limit each sentence to 12 words or fewer.

## Tests

Run tests with `--non-interactive` to keep the debugger off.

```sh
sbcl --noinform --non-interactive --load path/to/test.lisp
```
