# Ember — AI Maintainer's Guide

Standing instructions for any AI working on Ember, now or decades from now. A human pointing an
AI at this folder should be enough for that AI to create, fix, and extend Ember programs and the
environment itself, correctly.

## Orientation (read in this order)

1. `00_FOUNDATION/CLAUDE.md` — what Ember is, the five principles, architecture.
2. `00_FOUNDATION/CHARTER.md` — the locked direction and durable decisions.
3. `04_LANGUAGE/EMBER_SPEC.md` — the complete language definition. **This is your source of
   truth for what is valid Ember.** Do not infer syntax from memory of other languages.
4. `01_OPERATING/BUILD_LOG.md` — what has happened so far.

## Dogfooding rule (2026-06-10, Nero's directive)

**When the thing being written is a program, write it in Ember** — demos, examples, lesson
solutions, corpus entries, test fixtures, and any small computation that Ember can express.
Run it via the CLI (`04_LANGUAGE/cli/ember-cli.mjs`, engine loaded from ember.html) or the
headless harness. Reach for Python/Node only for *plumbing Ember cannot express* — patching
HTML, freezing JSON corpora, driving test runners — and when you do, say so. Every time
plumbing repeats a pattern Ember almost-but-can't express, note it: that's a worlds-test
feature earning its way in.

## Writing Ember programs

- Follow `EMBER_SPEC.md` exactly. The whole language fits there; if a construct isn't in the
  spec, it does not exist — do not invent syntax.
- Every program you produce gets the self-describing header (`# title:`, `# about:`,
  `# language:`, `# created:`). The header is how future readers — human or AI — know what a
  file is without running it.
- Prefer the canonical forms (`say`, `otherwise`, `box`, `color`) over the aliases.
- Programs are for ordinary people: choose friendly variable names, add a `#` comment where a
  choice isn't obvious, keep examples under ~25 lines.

## Maintaining the environment (ember.html)

- It must remain a **single self-contained HTML file**: no external libraries, no build step, no
  network calls, no accounts. This is Principle 4 (durable by construction) and is locked.
- All side effects flow through the `host` object. New features that touch the world (sound,
  timing, storage) get a host method, never a direct DOM/Web API call from the interpreter.
- Errors are part of the language: line number + plain-language problem + concrete suggested fix.
  Match the existing voice ("I expected…", "→ Like: repeat 5 times").
- When you add a feature, all five of these move together:
  1. the implementation in `ember.html`,
  2. a row on the in-app cheat sheet,
  3. an example in the in-app dropdown (and `05_EXAMPLES/` if it's gallery-worthy),
  4. the section in `EMBER_SPEC.md` (+ version bump if behavior changed),
  5. headless tests still green (see below), then a Build Log entry + Status Tracker update.

## Verifying your work (required before claiming success)

**The E1 world (2026-06-10, Decision D14): the engine's source of truth is
`04_LANGUAGE/src/ember-core.js`.** Edit the engine THERE, never inside ember.html. Rebuild the
page with `node src/build.mjs` (deterministic; the artifact stays one dependency-free file).
The bytecode VM lives in `src/ember-vm.js` and must stay byte-identical to the walker
(differential suite). Embedders use `src/ember-api.mjs`; hosts obey `src/HOST_INTERFACE.md`.
**One command tells the whole truth: `node verify.mjs` — every gate, all or nothing (19 gates
as of the worlds release; the count only grows).**
(HTML extraction survives only as the CLI's compatibility path for gifts and capsules.)

Three layers, all required — do not claim success with fewer:

1. **Language suite.** The core is testable without a browser because of the host seam: load
   `src/ember-core.js`, drive `tokenize → parse → interpret` (and `EmberVM.runVM`) with a
   recording host. Run the full regression plus checks for anything you changed, plus the
   conformance catalog (`02_QUALITY/conformance/`) — a wrong or unhelpful error message is a
   bug in Ember, by specification.
2. **Boot test.** Extract the page's ENTIRE script and run it in a fake browser (stub
   `document`, canvas context, `localStorage`, `window.prompt`). Assert: the examples dropdown
   fills, all buttons attach, the default program loads, and a simulated Run click completes
   without error text. Testing the core alone once shipped a dead page (Build Log, fifth
   entry). **Extract like a browser does:** non-greedy — stop at the FIRST closing-script
   sequence — and separately assert the file contains EXACTLY ONE such sequence. The
   closing-script byte-string is reserved everywhere inside the file: code, strings, and
   *comments* (Build Log, twelfth entry — a comment killed the page once).
3. **Install verification (per Bradley OS corrected finding 2026-05-14 + Decision D6).**
   File tools (Read/Write/Edit) are authoritative. The bash mount can serve a stale, size-pinned
   view — never verify a file-tool write by bash readback in the same session, and never install
   by copying a bash view of a file-tool-written file. Practice: build and test bash-native,
   install in one copy, then verify the installed file via Read/Grep: `<script>…</script>`
   complete, `</html>` last line, expected markers present.

These three layers are Steps 2–3 of `02_QUALITY/VERIFICATION_PROTOCOL.md`, which governs every
release. Step 4 (fresh-instance review) and Step 5 (apparatus compliance) complete the gate —
read the protocol before claiming a release done.

## Decision discipline

- Don't violate the five principles silently. If a trade-off is genuinely needed, log it in
  `BUILD_LOG.md` and, if durable, in `CHARTER.md` — and say so to the human.
- Backward compatibility: old `.ember` files keep running. The `# language:` header exists so
  tools can tell vintages apart if a break is ever unavoidable.
- The human decides direction; you decide implementation. When a request conflicts with the
  Charter, surface the conflict instead of quietly complying.
