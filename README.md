# meme-canonical-fixer

Memelang lexical layout canonicalizer CLI — self-hosting step 3.

Built 100% in Memelang, consuming the step-1 lexer (`lexer.meme` and `keywords.meme` vendored from `meme-tokenizer`) and complementing the step-2 checker (`meme-canonical-checker`). Compiles to a standalone native binary via `meme build` with zero runtime dependencies.

## Honest Scope

> [!IMPORTANT]
> **HONEST SCOPE:** This tool canonicalizes **LEXICAL LAYOUT only** — spacing, indentation, trailing whitespace, and final newline. It does **NOT** restate casual English into canonical phrasing (that is IR-level compiler work). After `fix`, `meme-canonical-checker` reports `CANONICAL TOKENS` or `FRESH`.

## CLI Usage

```bash
# Build native standalone binary:
uv run --project /mnt/ssd/projects/meme meme build main.meme -o fixer_bin

# Fix one or more files in place (rewrites only if layout changes):
./fixer_bin fix <file.meme> [file2.meme ...]

# Preview canonical layout to stdout WITHOUT modifying the file:
./fixer_bin preview <file.meme>
```

### Exit Codes & Diagnostics

- `0`: All requested operations succeeded.
- `1`: Any error occurred (missing file, unparseable lexical syntax, unknown command, or missing arguments).
- Errors print a single clear line to standard error (e.g. `Error: cannot read file: <path>`).

### Output Messages

- When a file's layout is modified: `<file>: fixed (<N> lines)`
- When a file's layout is already canonical: `<file>: already canonical`

## Canonicalization Rules

The emitter module (`emit(tokens) -> String`) applies the following rules:

1. **Line Structure Preservation:** Tokens are grouped by their original line numbers. The line structure is preserved exactly without reflowing, joining, or splitting lines. Blank lines in source code are preserved.
2. **Indentation:** The column of the first token of each line is emitted as spaces. (Tabs in source code already became column offsets during tokenization; lines whose first token column is 0 receive no indentation).
3. **Inter-Token Spacing:**
   - **NO space before:** `,` `:` `;` `)` `]` `}` `.`
   - **NO space after:** `(` `[` `.`
   - **Single space otherwise.**
4. **Trailing Whitespace:** All trailing whitespace characters (` `, `\t`, `\r`) are stripped from every line.
5. **Final Newline:** Files end with exactly one `\n`.
6. **Empty Input:** Empty input produces empty output (`""`).
7. **Comments:** Comment tokens are preserved as-is at their line positions, with trailing whitespace stripped.

## Correctness Invariant and Idempotence

- **Correctness Invariant:** For any tokenizable input, re-tokenizing `emit(tokens)` yields the **SAME** kind+value token stream as the original input (with comments filtered from both sides before comparing, matching `meme-canonical-checker` semantics). Spacing choices never split or merge tokens.
- **Idempotence:** `emit(emit(x)) == emit(x)` byte-for-byte. Fixing an already-canonical file leaves it unchanged and reports `already canonical`.

## Test Instructions

Tests are located in `tests/meme/` and can be run with `meme test`:

```bash
# Run the test suite on the Python target:
uv run --project /mnt/ssd/projects/meme meme test tests/meme --target python

# Run against individual targets:
uv run --project /mnt/ssd/projects/meme meme test tests/meme/01_emitter_rules_test.meme --target c
uv run --project /mnt/ssd/projects/meme meme test tests/meme/01_emitter_rules_test.meme --target rust
```

Per plan instructions, development runs against individual targets; the coordinator verifies the full six-target matrix (`python`, `rust`, `cpp`, `go`, `c`, `ts`).
