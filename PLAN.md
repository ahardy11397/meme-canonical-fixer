# PLAN — meme-canonical-fixer: self-hosting step 3

Build a lexical layout canonicalizer CLI entirely in Memelang. Self-hosting
step 3: it consumes the vendored step-1 lexer and complements the step-2
checker — `check` reports, `fix` repairs. Native binary via `meme build`,
zero dependencies.

HONEST SCOPE (state this in the README): this tool canonicalizes LEXICAL
LAYOUT only — spacing, indentation, trailing whitespace, final newline.
It does NOT restate casual English into canonical phrasing (that is
IR-level compiler work). After `fix`, `meme-canonical-checker` reports
`CANONICAL TOKENS` or `FRESH`.

Language docs: /mnt/ssd/projects/meme/docs/WRITING_GUIDE.md and
docs/TUTORIAL.md (read both first). Compiler commands run via
`uv run --project /mnt/ssd/projects/meme meme ...` from this repo.

## Ground rules (non-negotiable)

- Work ONLY in this repo. NO git commits — leave changes for review.
- 100% Memelang for program logic; no other languages.
- Every construct must parse with `--provider heuristic`.
- FOREGROUND test runs for individual targets only; the coordinator runs
  the full six-target matrix.
- lexer.meme and keywords.meme are VENDORED from meme-tokenizer: keep their
  behavior byte-identical, add only an attribution header comment. Do not
  refactor them.
- NO new language features. Record any language gap precisely and route
  around it; report gaps in the summary.

## Domain and CLI (thin argv-router main + logic modules; unknown command
-> usage + exit 1; every error = one clear line + exit 1)

    fix <file.meme> [file2.meme ...]   # rewrite each file with canonical layout
    preview <file.meme>                # print the canonical text WITHOUT writing

Lexical canonicalization rules (the emitter module — `emit(tokens) -> String`):
- Group tokens by their source line number; preserve the line structure
  (no reflowing, no joining/splitting lines).
- Indentation: the column of the FIRST token of each line, emitted as
  spaces (tabs already became spaces in token columns; a line whose first
  token col is 0 gets none).
- Between tokens on a line: NO space before `,` `:` `;` `)` `]` `}` `.`;
  NO space after `(` `[` `.`; single space otherwise.
- Strip trailing whitespace on every line; exactly one `\n` at end of file.
- Empty input -> empty output. Comment tokens are emitted as-is (they are
  part of the line).
- CORRECTNESS INVARIANT (the heart of the tool — test it hard): for any
  tokenizable input, re-tokenizing `emit(tokens)` yields the SAME
  kind+value token stream as the input (comments filtered from BOTH sides
  before comparing, matching the checker's semantics). Spacing choices
  must never split or merge tokens.

`fix` per file: read text (Optional reader; missing -> one-line error +
exit 1), tokenize, emit, write file back ONLY IF the text changes;
print `<file>: fixed (<N> lines)` or `<file>: already canonical`.

## Tests — tests/meme/*.meme, `meme test`

- Round-trip invariant: messy fixture (tabs, trailing spaces, blank lines,
  comments, strings with spaces and escapes, multi-char operators) —
  tokenize original vs tokenize(emitted), comment-filtered streams equal.
- Idempotence: emit(emit(x)) == emit(x) as text.
- Spacing rules: `(a, b)` stays `(a, b)`; `a->b` emitted as `a -> b`
  re-tokenizes to the same `->` token; `x.y` keeps no surrounding spaces.
- Indentation preservation: 0/4/8-col blocks keep their relative depth;
  tabs in the source normalize to spaces at the token column.
- Trailing whitespace stripped; file ends with exactly one newline;
  already-canonical input unchanged byte-for-byte.
- Empty input -> empty output.
- `preview` writes nothing (assert file unchanged).
- `fix` on missing file -> clean one-line error, exit 1, no traceback.
- Fixtures under tests/tmp/, unique per test, cleaned up.

## Acceptance ("done means")

1. `meme test` python target — 0 failed.
2. Same `--target rust cpp go c ts` — ALL SIX green (coordinator verifies).
3. `meme build main.meme -o fixer_bin` — end-to-end demo transcript:
   copy a commented .meme file from /mnt/ssd/projects/meme/examples/advanced/
   into a scratch dir with its .lock, run `check` (CANONICAL TOKENS),
   make a mess of the layout (tabs/trailing spaces), run `check` again
   (still CANONICAL TOKENS), run `fix`, show the file is clean, and run
   `check` a third time — same verdict, file idempotent.
4. README.md: CLI usage, the canonicalization rules, the honest-scope
   statement, the round-trip invariant, test instructions.
5. Final message: files, test count per target, demo transcript,
   deviations (if any) with reasons.
