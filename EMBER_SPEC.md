# Ember Language Specification — ember 4.4

This is the complete, authoritative definition of the Ember language. It is written to be
ingested whole by an AI (or read by a person) so that correct Ember programs can be written,
checked, and maintained from this document alone. The reference implementation is `ember.html`
in this folder.

## 1. The .ember file format

A `.ember` file is plain UTF-8 text. Its first lines SHOULD be a self-describing header made of
ordinary comments, so every file is simultaneously documentation and a valid program:

```
# ember program
# title: row of circles
# about: Draws seven circles in a row.
# language: ember 2.0
# created: 2026-06-09
```

Recognized header fields: `title`, `about`, `language`, `created`. The header ends at the first
non-comment, non-blank line. Unknown `# key: value` lines are permitted and ignored. A file
without a header is still valid; tools add one on save.

## 2. Lexical rules

- One statement per line. Statements end at the newline.
- Blank lines are ignored. `#` starts a comment that runs to the end of the line.
- Text (strings) is wrapped in `"` or `'`. Strings may not span lines.
- Numbers: decimal, optional fraction (`7`, `3.5`). Negative via unary minus (`-5`).
- Names (variables): letters, digits, underscore; must start with a letter or `_`.
  Names are case-insensitive (`Score` and `score` are the same variable).
- Keywords (reserved, case-insensitive): `say print show set to change by repeat times while
  if then otherwise else end is not greater less than at least most equal and or clear color
  pen dot line circle box rect fill filled ask into wait seconds with random background when
  pressed clicked beep write play for web true false list add each item of give back in
  contains does file put read`.
- **`count` is deliberately NOT reserved** (the compatibility oath protects a frozen program
  that uses it as a variable): it acts special only in the exact phrase `count of …`. A
  variable named `count` keeps working everywhere else, forever. The same guarantee covers
  the 4.2 phrase families: `total`/`sum`, `highest`/`biggest`/`largest`, `lowest`/`smallest`,
  `average`/`mean`, `words`, and `lines` are NOT reserved — each acts special only in the
  phrase `<word> of`.
- Operators: `+ - * / ( )`. Commas are optional separators and are ignored.

**Forgiveness rules (accepted in, never written out by tools):**
- The word `the` is a filler word and is ignored everywhere (`set the score to 5` ≡ `set score to 5`).
  Consequence: `the` cannot be a variable name.
- Singular forms are accepted: `time` ≡ `times`, `second` ≡ `seconds`, `contain` ≡ `contains`.
- Canonical output: tools and AI that *generate* Ember write the canonical forms only
  (`say`, `otherwise`, `box`, `color`, plural `times`/`seconds`, no filler words).

## 3. Statements

| Form | Meaning |
|---|---|
| `say <expr>` | Print the value as text, followed by a newline. Aliases: `print`, `show`. |
| `set <name> to <expr>` | Create or update a variable. |
| `change <name> by <expr>` | Sugar for `set name to name + expr`. The variable must already exist. Negative amounts count down: `change n by -1`. |
| `wait <expr> [seconds]` | Pause that many seconds (fractions fine, e.g. `0.15`). Negative waits 0; capped at 60 per wait. The unit word is optional but natural. |
| `repeat <expr> times … end` | Run the body ⌊expr⌋ times. Expr must be a number. |
| `repeat while <cond> … end` | Run the body while the condition holds. `then` after the condition is optional. |
| `add <expr> to <name>` | Append a value to a list, in place. The name must already hold a list (`set names to a list` first) — adding to anything else is a teaching error. |
| `for each <name> in <expr> … end` | Walk a list in order: set `<name>` to each item in turn and run the body. The loop name follows normal scope rules (an action input of the same name stays private to the action). Walking a non-list is a teaching error; an empty list runs the body zero times. If the body grows the list, the walk sees the new items. |
| `give back <expr>` | Inside `to … end` only (using it elsewhere is a parse-time teaching error): the action stops immediately and hands the value to its caller. See §3 Named actions. |
| `if <cond> then … end` | Branch. Optional `otherwise` (alias `else`) section before `end`. **Chains (4.2, D24):** `otherwise if <cond> then` continues the decision — any number of links, one shared `end` closes the whole chain. Exactly one road runs. Sugar for a nested if in the otherwise branch, so old programs and old tools are untouched; Explain renders a chain flat, the way it was written. |
| `ask <expr> into <name>` | Show the question, store the reply. If the reply parses as a number, it is stored as a number, else as text. Cancel stores `""`. |
| `clear` | Wipe the canvas. |
| `background <expr>` | Paint the entire canvas that color (drawing color is unchanged). |
| `filled circle <x> <y> <r>` / `filled box <x> <y> <w> <h>` | Solid shapes — the recommended way to fill. (`fill` toggle remains accepted: forgiving in, canonical out.) |
| `beep` | A short sound (quiet no-op where sound is unavailable). |
| `write <expr> at <x> <y>` | Put text on the canvas at x, y in the current color. `write "score: " + score at 20 30` |
| `ask the web for <expr> into <name>` | **The one feature that can rot (D17).** Fetch a page's text into a variable. No answer — dead site, no network, a host that can't reach out — is the EMPTY TEXT, never a crash: programs must expect silence (`if reply is not "" then …`). A host with no web ability at all raises the teaching error "The web isn't reachable from here." Replies are capped (64 KB). Works best in the terminal; browsers may be refused by other sites (CORS). |
| `put <expr> into file <expr>` | **Files (4.3, D25) — an optional host power, like the web.** Replace the file's contents with the value's text, ending with a line break. File names are PLAIN: letters, digits, spaces, dashes, dots and underscores only, no folders, no slashes, no `..`, no leading/trailing space or dot, at most 100 characters, and not a name that means hardware on some computers (`con`, `prn`, `aux`, `nul`, `com0`–`com9`, `lpt0`–`lpt9`, with or without an extension) — anything else is a teaching error (the engine owns this fence, identically in every home, so a program means the same thing on every machine). Files live BESIDE the program: the terminal writes to the current folder; the Ember page keeps them in its pocket (localStorage; the Files button lists, downloads and removes them). A home with no file powers raises the teaching error "Files can't be kept from here." A refused write (full disk, locked file) is the teaching error "I couldn't save the file '…'." `put score into file "save.txt"` |
| `add <expr> to file <expr>` | Append the value's text plus a line break to the file (created if missing) — one `add`, one line, the same rhythm as `say`. Multi-line files are built this way, since Ember text can't span lines. Same fences as `put`. `add "dear diary" to file "diary.txt"` |
| `read file <expr> into <name>` | Read the file's whole text into a variable. A missing or unreadable file is the EMPTY TEXT — the kind silence, never a crash: programs must expect it (`if memory is "" then …`). Reading is forgiving: other homes' `\r\n` line endings are normalized to `\n`, and the file's final line break is not part of the text (so `put x into file "f"` then `read file "f" into y` gives back exactly `x`'s text). Reads are capped (64 KB). An empty file and a missing file read the same — both are `""`. Pairs naturally with 4.2's `lines of`: `for each line in lines of memory … end`. `read file "save.txt" into memory` |
| `play <note> [for <n> seconds]` | Music. Notes 1–8 are a major scale; values are rounded and clamped to 1–8. Default length 0.3s (0.05–10s). Notes take their time, so consecutive plays make a tune. `play 5 for 0.5 seconds` |
| `color <expr>` | Set drawing color. Any CSS color text (`"red"`, `"#ff8800"`). Alias: `pen color <expr>`. |
| `fill` | Toggle fill mode: while on, `circle` and `box` draw filled. Starts off. |
| `dot <x> <y>` | Draw a small dot (radius 3). |
| `line <x1> <y1> <x2> <y2>` | Draw a line segment. |
| `circle <x> <y> <r>` | Draw a circle (outline, or filled in fill mode). |
| `box <x> <y> <w> <h>` | Draw a rectangle from top-left corner. Alias: `rect`. |

Block statements (`repeat`, `if`, `for each`, `to`) require their body on following lines and are
closed by a line containing `end`. `repeat`, `if`, and `for each` nest freely; `to` is top-level only.

### Named actions (`to … end`)

```
to draw flower with x y      # teach Ember a new action
  color "hotpink"
  circle x y 14
end

draw flower 100 120          # use it by name
```

Rules:
- **Definition:** `to <name words> [with <input names>] … end`. The name is one to five plain
  words — the limit is enforced at definition time with a teaching error — and name words may
  not be keywords (also enforced, including as the first word). `with` separates the name from
  the inputs.
  Definitions are allowed only at the top level of the program, and each name may be taught
  only once per program (re-teaching is an error naming the first definition's line).
- **Hoisting:** an action may be *used anywhere in the file*, including above the lines that
  teach it. Definitions are learned before the program starts running.
- **Calling:** write the action's name followed by one expression per input
  (`draw flower 100 120`, `draw flower spot + 10 200`). The argument count must match the
  input count exactly — a mismatch is a parse-time error listing the input names.
  Multi-word names are matched longest-first, so `draw flower` wins over a variable `draw`.
- **Scope:** inputs are **private** to the action (they shadow, and writes to them never leak
  out). *All other* variables read and write the shared global environment. One sentence for
  humans: *inputs belong to the action; everything else is shared.*
- **Recursion** is allowed. Call depth is capped at 200 with a teaching error ("…isn't calling
  itself without a way to stop"). The global step limit still applies.
- **Giving back (4.0):** `give back <value>` inside the action stops it immediately and hands
  the value to the caller — from anywhere in the body, including inside `repeat`, `if`, or
  `for each`. An action's name used **where a value is expected IS the call**:
  `set y to double 5`, `add new star to stars`, `say roll` (zero-input actions just name
  themselves). Longest name still wins. **Arguments bind tightly** (multiplicative level — the
  `random` precedent): `say double 3 + 1` means `(double 3) + 1`, but `* /` bind INTO an
  argument — `say double 2 * 3` means `double (2 * 3)`; parenthesize compound
  arguments — `say double (3 + 1)`, `say (double 2) * 3`. An action called for a value that finishes without
  reaching `give back` is a runtime teaching error ("…finished without giving anything back").
  Statement-position calls are unchanged: a given-back value there is quietly dropped, so
  pre-4.0 programs keep their exact meaning. Recursion with `give back` works
  (`give back n * (fact (n - 1))`).

### Reacting to keys (`when … pressed … end`)

```
when "left" pressed
  change paddlex by -25
end
```

- Top-level only, like actions. The key is a quoted string or bare word: `"left"`, `"right"`,
  `"up"`, `"down"`, `"space"`, or a single letter. `is` before `pressed` is optional.
- Handlers run **between statements** of the main program (single-threaded — no interruption
  mid-statement): key presses queue up and are drained in order before each statement executes.
  A typical game is a `repeat while` loop with a short `wait` per frame; handlers update shared
  variables the loop reads.
- Handlers only run while the program is running. Keys typed into the editor are not captured
  (and arrows/space keep working in the editor while a program runs).
- If several handlers match the same key, **all run, in definition order**.
- **`when clicked … end`** reacts to a click or tap on the canvas. Before the body runs, the
  variables `clickx` and `clicky` are set (rounded, in canvas units, scale-corrected). Clicks
  share the same between-statements queue as keys.

Canvas: 400 × 400 units. `0 0` is the top-left; x grows right, y grows down.

**Drawing-command arguments** are space-separated expressions, parsed greedily left to right:
`line x - 8 312 x - 14 - step 330` is four arguments because an expression ends where no
operator follows. When an argument is compound, parentheses make the grouping unmistakable and
are the recommended style: `line x y (x + 30) (y - 30)`. Both forms are valid.

## 4. Expressions

- Values: number, string, variable, `true`, `false`, parenthesized expression, unary `-`, and
  `random <a> to <b>` — a whole number from a to b inclusive (bounds may be expressions;
  reversed bounds are swapped; negative bounds fine; fractional bounds are rounded inward —
  ceiling of the low, floor of the high — and a range containing no whole number is a
  teaching error). `random` binds its bounds tightly
  (multiplicative level): in larger expressions parenthesize for clarity —
  `set x to 5 + (random 1 to 10)`.
- Binary operators with normal precedence: `* /` bind tighter than `+ -`. Left-associative.
- `+` on two numbers adds; if either side is not a number, both sides are converted to text and
  joined. `- * /` require numbers (a numeric-looking string like `"5"` is accepted and converted).
  Chained `+` with mixed text and numbers works left to right — parenthesize any math inside a
  text join: `say "You can vote in " + (18 - age) + " years."`
- Division by zero is an error (see §6).
- **Lists (4.0):** `a list` is a new empty list — `set names to a list` (bare `list` is
  accepted; `a list` is canonical). `count of <list>` is how many items it holds.
  `item <n> of <list>` is the n-th item, **counting from 1**; the index may be any expression
  (`of` ends it cleanly — `item i + 1 of names` works); a fractional index is floored
  (`item 1.5` reads item 1 — same rule as `repeat ⌊expr⌋ times`); an index out of range is a
  teaching error that names the count. Both `count of` and `item … of` take their list operand
  tightly (primary level) — parenthesize anything compound. Lists are values: `set b to a`
  makes both names share ONE list (adding through either is visible through both). A list
  shown as text joins its items with ", " (an empty list is the empty text); a list that
  contains itself prints the inner occurrence as `(the same list)` — never an endless loop;
  `is` comparison on lists compares that text; `- * /` with a list raises the usual number
  errors. Lists may hold lists.
- **The aggregation family (4.2, D24):** `total of <list>`, `highest of <list>`,
  `lowest of <list>`, `average of <list>` — arithmetic over a whole list, in `count of`'s
  quiet pattern (phrases, not keywords; operand binds tightly at primary level). Items must
  be numbers (numeric text like `"4"` is accepted — the house tolerance); a non-number item
  is a teaching error naming the phrase. `total of` an empty list is 0; `highest`/`lowest`/
  `average` of an empty list is a teaching error ("there is no … of nothing"). Forgiving in,
  canonical out: `sum` (→ total), `biggest`/`largest` (→ highest), `smallest` (→ lowest),
  `mean` (→ average) are accepted; tools and Explain write only the canonical four.
- **Text splitting (4.2, D24):** `words of <text>` is a list of the text's words (split on
  any run of whitespace, never an empty word); `lines of <text>` is a list of its lines
  (split on newlines, `\r` shed, blank lines skipped — kindly). Both take any non-list value
  via its text form; a list operand is a teaching error ("needs text, not a list"). They
  compose: `count of words of reply`, `for each w in words of saying`, `item 2 of lines of
  reply`. Plain text typed in programs cannot hold a newline, so `lines of` earns its keep
  on web replies and other host-given text.
- **Character access (4.4, D26):** `letters of <text>` is a list of the text's characters —
  one item per Unicode code point, including spaces and punctuation; an empty text is the empty
  list — the same `<word> of` shape as `words of`/`lines of`. `letter <n> of <text>` is the
  n-th character, counting from 1 (a fractional index is floored, like `item … of`; an index
  out of range is a teaching error naming the count; the operand must be text, not a list).
  They compose with everything: `count of letters of word`, `for each ch in letters of name`,
  `letter count of letters of word of word`. Like the aggregation and split words, `letter`
  and `letters` are NOT reserved — they act special only in these phrases, and a taught action
  named `letter`/`letters` always wins (the oath protects the frozen `grade letters` program,
  which teaches an action called `letter`).
- Number-to-text: integers print without a decimal point; other numbers are rounded to 6 decimal
  places. Booleans print as `yes` / `no`.

## 5. Conditions

A condition is `<expr> <comparator> <expr>`, optionally joined with `and` / `or`
(left-associative, `and` and `or` have equal precedence; evaluation is short-circuit).

Comparators (plain words only):

| Words | Meaning |
|---|---|
| `is` (or `is equal to`) | equal — numeric if both numbers, else text equality |
| `is not` | not equal |
| `is greater than` | > (numbers required) |
| `is less than` | < (numbers required) |
| `is at least` | ≥ (numbers required) |
| `is at most` | ≤ (numbers required) |
| `contains` (4.1) | the left value's text holds the right value's text — **ignoring case, on purpose** (D18): `if reply contains "workforce stability" then` matches "Workforce Stability". Works on any values via their text form: numbers (`12345 contains 234`), booleans (`yes`/`no`), and lists via their written-out, comma-joined form (`names contains "Nero"` — note the match looks at that joined text, so it can also match across item boundaries). Every text contains the empty text. |
| `does not contain` (4.1) | the opposite. (`does not contains` is accepted — forgiving in; the canonical pair is `contains` / `does not contain`.) |

Truthiness (for `repeat while` results): `true`, nonzero numbers, and nonempty text are true.

## 6. Errors — required behavior

Errors are part of the language. Every error MUST:
1. Name the line number,
2. State the problem in plain language (no jargon, first person: "I expected…"),
3. Where possible, suggest the concrete fix, often by example (`→ Like: repeat 5 times`).

**"Did you mean…?" is required behavior:** an unknown word at the start of a line, or an unknown
variable name, MUST be fuzzy-matched (edit distance ≤ 2) against the command words / known
variables and the nearest match suggested ("I don't know the word 'repaet'. → Did you mean
'repeat'?"). Only fall back to the generic error when nothing is close.

Canonical errors: unclosed string; unknown symbol; unknown variable (suggests nearest name, else `set x to 0`);
missing `times` / `then` / `to` / `into`; unclosed block ("This 'repeat' is never closed → add a
line that just says 'end'"); missing comparator in a test; wrong arg count for drawing commands;
non-number where a number is needed; divide by zero; runaway-loop stop (step limit ≈ 2,000,000;
`repeat while` also guards at 1,000,000 iterations); the worlds errors (E058–E069): `give back`
outside an action, `give back` without a value, an expression call that never gives back
(naming the action and the fix), `add` to a non-list, `for each` over a non-list, `item` out
of range (naming the count), `count of` / `item … of` a non-list; the text errors (E070–E071):
`does` without `not contain` (both halves teach the full phrase); the watchlist errors
(E072–E075): aggregation of a non-list / of an empty list / over words, splitting a list;
the file errors (E076–E087): `put` without a value or without `into file`, `read` without
`file` or `into`, a missing file name, the plain-name fence (slashes, `..`, edge spaces/dots,
over-long names, reserved device names like `nul`/`con`), files unavailable from this home,
and a refused write.

## 7. Execution model

Pipeline: `tokenize(source) → parse(tokens) → AST → interpret(AST, host)`.

All side effects go through a **host** interface — the language core never touches the screen
directly. Host methods (as of 1.0): `say(text)`, `clear()`, `color(name)`, `toggleFill()`,
`ask(question) → text`, `draw(shape, args, line, filled)`, `wait(seconds) → Promise`,
`background(color)`, `beep()`, `write(text, x, y)`, `random() → 0..1` (optional; defaults to
Math.random), `nextKey() → key | {click:true,x,y} | null` (optional; the between-statements
queue — click items set `clickx`/`clicky` before click handlers run), `stopped() → bool`
(optional; run cancellation), and — 4.3 — the file powers (all optional, all three or none in
practice): `writeFile(name, text)` (replace), `appendFile(name, text)` (append),
`readFile(name) → text | null` (null/undefined is the kind silence; hosts cap at 64 KB).
The engine validates file names BEFORE calling the host, so the fence speaks the same words in
every home; replayable hosts back the trio with a canned pocket (`opts.files`). The browser host renders to canvas + output pane; a headless host runs the
same core for testing. **Any new effectful feature must go through the host, and this list
must be updated with it.**

**The wish seam (4.2, D23):** the unknown-phrase teaching error (no near-miss guess) carries
the attempted phrase as `err.wish` — pure data on the error, nothing more. Hosts MAY record
it: the page keeps the last 200 wishes in `ember.wishes.v1`; the CLI appends a
`date<TAB>phrase` line to `ember-wishes.log` beside wherever it runs, and says so
("✎ I've made a note that you wanted this"). Recording must never change program-visible
behavior, error text, or the exit story — the wish log is the earn-rule's instrument
(FEATURE_WATCHLIST.md), not a language feature.

**The workshop (3.0, grown in 4.0):** the Learn button opens a built-in 14-lesson curriculum,
from `say "hello, world"` to a working game. Each lesson's check runs the learner's REAL program
against a silent recording host (`runRecorded` — the same host-seam trick the test suites use,
living inside the app) and answers with PASS-praise or a sentence that teaches. Lessons travel
inside every gift. Lesson checks may rerun a program with scripted answers, seeded randomness,
or queued keys. `play` was admitted to the language because the music lesson demanded it
(earn-rule, D11); lists and `give back` entered at 4.0, earned by repeated demand from real
programs (D15) — the "Many things at once" lesson teaches lists.

**Explain (2.0):** `explainProgram(source)` is part of the reference implementation — a pure
AST→plain-language walker, exposed in-app as the Explain button. It is how `.ember` files
literally explain themselves, with no network or AI dependency. New statement kinds must be
added to it (a missing kind renders as a visible `(kind)` placeholder, which is a Step 5 FAIL).

**Gift (2.0):** the Gift button exports the current program as a **standalone webpage** — the
entire Ember environment serialized with the program embedded in a `gift-payload` script. A
gift runs on open, shows a gift banner, never touches the recipient's autosave, and *is* a full
Ember editor — readable, remixable, re-giftable. Re-gifting replaces the old payload.

**The debugger panel (4.0, D16):** ember.html now inlines the bytecode VM (`src/ember-vm.js`)
beside the core — built in by `build.mjs`, still ONE dependency-free file — and the Step
button opens a panel that runs the program one statement at a time: Step / Continue / Stop,
the current line named and quoted, and a live "values right now" view (action inputs marked
private). Gifts inherit the panel. Run, Debug-narration, and Explain still use the reference
walker; the VM is held byte-identical to it by the differential suite.

Variables live in a single global environment (no scoping yet). The interpreter is **async**:
`wait` suspends via `host.wait(seconds)` without blocking the page. Hosts MAY provide
`stopped()`; the interpreter polls it each statement and halts quietly (an `EmberStop`, not an
error) — this is how a newer Run cancels an older animated one.

## 8. Reserved for the future (do not repurpose)

Nothing is currently parked: every reserved word has been earned into the language under the
earn-rule — `play` in 3.0 (the music lesson, D12), `ask the web` in 3.1 (D17), lists and
`give back` in 4.0 (D15). Future features enter this list before they enter the language.
Aliases `print`/`show`/`else`/`rect`/`pen` exist for forgiveness but the cheat sheet teaches one
canonical form: `say`, `otherwise`, `box`, `color`.

## 9. Formal grammar (EBNF)

```ebnf
program     = { line } ;
line        = [ statement ] , [ comment ] , newline ;
comment     = "#" , { any-char } ;
statement   = say | set | change | repeat | if | action | call | ask | wait | play | beep
            | clear | background | color | "fill" | filled | draw | write | when
            | add | foreach | giveback | put | readfile ;
say         = ("say"|"print"|"show") , expr ;
set         = "set" , name , "to" , expr ;
change      = "change" , name , "by" , expr ;
repeat      = "repeat" , ( expr , "times" , block | "while" , condition , ["then"] , block ) ;
if          = "if" , condition , "then" , body ,
              { ("otherwise"|"else") , "if" , condition , "then" , body } ,
              [ ("otherwise"|"else") , body ] , "end" (* one shared end closes the chain *) ;
block       = newline , body , "end" ;
body        = { line } ;
action      = "to" , name , { name } (* ≤5 words, no keywords *) , [ "with" , name , { name } ] , block ;
call        = action-name , { expr } (* one expr per input *) ;
ask         = "ask" , ( "web" , "for" , expr , "into" , name | expr , "into" , name ) ;
add         = "add" , expr , "to" , ( "file" , expr | name ) (* a file append, or the name must hold a list *) ;
put         = "put" , expr , "into" , "file" , expr ;
readfile    = "read" , "file" , expr , "into" , name ;
foreach     = "for" , "each" , name , "in" , expr , block ;
giveback    = "give" , "back" , expr (* inside an action body only *) ;
wait        = "wait" , expr , [ "seconds" ] ;
play        = "play" , expr , [ "for" , expr , [ "seconds" ] ] ;
background  = "background" , expr ;
color       = ("color" | "pen" "color") , expr ;
filled      = "filled" , ( "circle" , expr , expr , expr | ("box"|"rect") , expr , expr , expr , expr ) ;
draw        = "dot" e e | "line" e e e e | "circle" e e e | ("box"|"rect") e e e e   (* e = expr *) ;
write       = "write" , expr , "at" , expr , expr ;
when        = "when" , ( "clicked" | key , ["is"] , "pressed" ) , block ;
key         = string | name ;
condition   = comparison , { ("and"|"or") , comparison } ;
comparison  = expr , ( "is" , [ "not" | "greater" "than" | "less" "than" | "at" ("least"|"most")
            | "equal" ["to"] ] | "contains" | "does" "not" "contains" ) , expr
            (* canonical negative spelling: does not contain *) ;
expr        = mul , { ("+"|"-") , mul } ;
mul         = primary , { ("*"|"/") , primary } ;
primary     = number | string | name | "true" | "false" | "-" primary | "(" expr ")"
            | "random" , mul , "to" , mul
            | [ "a" | "an" ] , "list" (* a new empty list; "a list" is canonical *)
            | "count" , "of" , primary (* "count" is a phrase here, not a keyword *)
            | agg-word , "of" , primary (* 4.2: same quiet pattern; canonical total/highest/lowest/average; sum/biggest/largest/smallest/mean accepted in *)
            | ("words"|"lines"|"letters") , "of" , primary (* 4.2: text→list; 4.4: letters — one item per character *)
            | "item" , expr , "of" , primary
            | "letter" , expr , "of" , primary (* 4.4: the n-th character, 1-indexed; "letter" not reserved *)
            | action-name , { mul } (* one tight argument per input; the action must give back *) ;
string      = '"' { char } '"' | "'" { char } "'" (* no escapes; no newlines *) ;
agg-word    = "total" | "sum" | "highest" | "biggest" | "largest" | "lowest" | "smallest" | "average" | "mean" ;
```
Forgiveness (accepted, not produced): `the` ignored; `time`→`times`, `second`→`seconds`;
commas ignored; names case-insensitive.

## 10. Versioning

This document defines **ember 4.4** (0.1 = kickoff prototype; 0.2 added the .ember format,
save/load, autosave; 0.3 added `wait`, `change … by`, forgiveness rules, did-you-mean errors,
and the async interpreter; 0.4 added named actions — `to … with … end`, hoisting, private
inputs, recursion guard; 1.0 — the magazine-test release — added `random`, `background`,
`filled circle`/`filled box`, `beep`, and `when <key> pressed` event handlers with the
between-statements key queue; 2.0 — the gift-test release — added `write … at x y`,
`when clicked` with `clickx`/`clicky`, the Explain walker, and the Gift standalone-webpage
export; 3.0 — the workshop-test release — added the Learn workshop with 13 engine-checked
lessons, `runRecorded`, and `play` notes 1–8, earned by the music lesson; 3.1 added
`ask the web for … into …` (D17) — earned by Nero reaching for his own website, fenced as an
optional host power with kind empty-text degradation and canned-reply corpus freezing;
4.0 — the worlds release — added **lists** (`a list`, `add … to`, `for each … in`,
`count of`, `item … of`, with `count` deliberately unreserved) and **`give back`** with
action calls inside expressions (D15), the lists lesson (14 total), and the in-page step
debugger riding the inlined VM (D16); 4.1 — the text test — added **`contains`** /
**`does not contain`**, case-blind by design (D18), earned three times over — last by a
fetched homepage the language could only `say`, never search; 4.2 — the watchlist
release — added **`otherwise if`** chains (one shared `end`), the **aggregation family**
(`total of` / `highest of` / `lowest of` / `average of`, aliases accepted in and the
canonical four written out, nothing reserved), and **`words of`** / **`lines of`** —
the first features admitted through the pre-registered `01_OPERATING/FEATURE_WATCHLIST.md`
(D24; otherwise-if's demand evidence was on disk in magic-8-ball's six sequential ifs) —
plus the wish log seam, D23; 4.3 — the files release — added **`put … into file …`**,
**`add … to file …`**, and **`read file … into …`** (D25) — optional host powers on the D17
template, decided by Nero from the four-mission demand ("build a game, have sound, develop a
webpage, create files"), fenced to plain names beside the program, with the kind silence for
missing files and a canned-pocket corpus freeze so the oath never touches a real disk; the
ruling supersedes the watchlist's earlier "files-as-such stay out" framing by Nero's charter
authority, logged in D25); 4.4 — the characters release — added **`letters of`** (text
becomes a list of its characters) and **`letter <n> of`** (the n-th character, 1-indexed),
the `<word> of` grammar reaching into text one character at a time, earned from two independent
external reviews (D26); and made the silent `+` text-join VISIBLE in the Debug narrator (D27) —
when a number is absorbed into text, Debug now says `joined as text: "…"`, changing no
program's output (apparatus, not language). Behavior changes
require: a frozen copy of this spec in `spec_archive/` (append-only, per release), a golden-
corpus replay pass (`02_QUALITY/golden/` — old programs do not break, ever), and a bump of
the version here and in `LANGUAGE_VERSION`
in ember.html, a Build Log entry, and headless tests staying green. Old `.ember` files must keep
running; if a break is ever unavoidable, the `# language:` header tells tools which rules to apply.
