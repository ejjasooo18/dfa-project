<title>SimpCalc DFA — Project Plan</title>

# SimpCalc Scanner DFA — Project Plan

**Course:** CSCI 70 · **Deliverable:** DFA diagram for the SimpCalc Scanner, submitted as a **PDF**
**Group size:** max 3 members · **Submitter:** one member only

> Fill in before submitting: **Surnames:** `______`, `______`, `______` · **Deadline:** `______`

---

## 1. What we are actually submitting

From `Project DFA.pdf` (slides 13–15):

| Requirement | Detail |
|---|---|
| Format | One **PDF** file |
| Content | The **DFA diagram** for the Scanner |
| Names | Surnames of all group members written out |
| COA | Certificate of Authorship — **must be included** |
| Tooling | Draw.io / FigJam preferred; hand-drawn allowed **if legible** |
| Error states | Spec says "about **3–4 lexical errors**" should appear in the DFA |

**Not required this submission:** the parser, and the keyword-lookup step (slide 9 explicitly says keywords don't need to be shown in the DFA — they're recognized as Identifiers first, then re-classified in a separate lookup step).

**Scope note:** this submission is the *diagram only*. The code comes later. Every design decision below should therefore be defensible on paper, not just "whatever the code does."

---

## 2. Ground truth we extracted from the samples

We checked all 9 `sample_input_*.txt` / `sample_output_scan_*.txt` pairs. These behaviors are **confirmed by the reference output** and the DFA must reproduce them. Do not design around assumptions — design around this table.

### 2.1 The four error states (this satisfies the "3–4 lexical errors" requirement)

| # | Error message in reference output | Triggered by | Evidence |
|---|---|---|---|
| E1 | `Lexical Error: Illegal character/character sequence` | A character with no valid transition out of the **start state** (`.`, `#`, `@`, `&`, `$`, `%`, `?`, …) | `sample_input_4.txt:6` `.71` → error, then `Number 71` |
| E2 | `Lexical Error: Invalid number format` | A malformed number lexeme | `73.`, `55.2e`, `1.e`, `3.2ea`, `3e+e` |
| E3 | `Lexical Error: Unterminated` | A `"` that hits newline/EOF before the closing `"` | `sample_input_2.txt:16` `"half a stri` |
| E4 | `Lexical Error reading character !` | `!` **not** followed by `=` | `sample_input_8.txt:26` `IF 2!1:` |

### 2.2 The `.` rule (this one is subtle — get it right)

A bare `.` produces **two different errors** depending on where the DFA is when it sees it:

- `.` reached from the **start state** → **E1 Illegal character**, then scanning resumes.
  `.71` → `E1`, `Number 71`
- `.` reached from **inside/after a number** (maximal munch pulls it into the number lexeme) → **E2 Invalid number format**.
  `111.222e333.444` → `Number 111.222e333`, `E2`, `Number 444`
  `.1e1. 1.e.1` → `E1`, `Number 1e1`, `E2`, `E2`, `E1`, `Number 1`

This falls out naturally from a correct DFA: the number sub-machine has a transition on `.` from its accepting states into an "invalid number" trap. It is **not** a special case to hand-wave — draw the arrow.

### 2.3 Maximal munch + backtracking

The scanner takes the **longest valid prefix**, emits it, then restarts at the offending character:

| Input | Output |
|---|---|
| `1abc` | `Number 1`, `Identifier abc` |
| `5213y` | `Number 5213`, `Identifier y` |
| `abc1` | `Identifier abc1` (one token) |
| `3.2east` | `E2` (consumed `3.2ea`), `Identifier st` |
| `3e+e3` | `E2` (consumed `3e+e`), `Number 3` |
| `** * *` | `Raise`, `Multiply`, `Multiply` |
| `=!=!ouch` | `Equal`, `NotEqual`, `E4`, `Identifier uch` ← note `!o` was consumed |

Our diagram must show which arrows are **lookahead-with-backtrack** (typically annotated `other` / `*` on the return arrow). This is a team-wide convention — see §3.

### 2.4 Sign handling — a common trap

`-` and `+` are **always** their own tokens (`Minus` / `Plus`), *never* part of a number literal — **except** immediately after `e`/`E` inside an exponent.

- `3 - 2 - 1` → `Number 3`, `Minus`, `Number 2`, `Minus`, `Number 1`
- `(-10 + ...` → `LeftParen`, `Minus`, `Number 10`, `Plus`, …
- `7E+8` → **one** token `Number 7E+8`
- `9.8E+7+6.5e4+3e+21` → `Number 9.8E+7`, `Plus`, `Number 6.5e4`, `Plus`, `Number 3e+21`

### 2.5 `e` / `E` is shared between Identifier and Number

`e+e` → `Identifier e`, `Plus`, `Identifier e`. An `e` at the **start state** is a letter (identifier path). An `e` **after digits** is the exponent marker. The two sub-machines must not fight over it.

### 2.6 Keywords are case-sensitive

`IF ELSE ENDIF SQRT PRINT AND OR NOT` → keyword tokens.
`if else endif sqrt print and or not` → **eight plain Identifiers.**
(Handled by post-scan lookup, not the DFA — but state it in the doc so the grader knows we knew.)

### 2.7 Strings

- `"` … any non-newline … `"` → `String`, quotes **included** in the lexeme: `"done"`, `" aa // "`.
- `//` **inside** a string is not a comment: `" aa // "` is one `String`.
- `&%@#$` inside a string is fine.
- `aaa" + "aaa` → `Identifier aaa`, `String " + "`, `Identifier aaa`.

### 2.8 Comments and EOF

- `//` → discard through end of line, **emit no token**. The comment start shares the `/` prefix with `Divide` — needs a lookahead state.
- Every run ends with a single `EndofFile` token.
- Whitespace (space, tab, newline) is skipped; newline also increments the line counter.

### 2.9 Complete token list (21, from slide 5)

`Identifier` · `Number` · `String` · `Assign :=` · `Semicolon ;` · `Colon :` · `Comma ,` · `LeftParen (` · `RightParen )` · `Plus +` · `Minus -` · `Multiply *` · `Divide /` · `Raise **` · `LessThan <` · `Equal =` · `GreaterThan >` · `LTEqual <=` · `NotEqual !=` · `GTEqual >=` · `EndOfFile`

---

## 3. Conventions to lock in BEFORE splitting up (Phase 1, everyone together)

Do not skip this. If three people draw in three notations, integration will cost more than the drawing did.

1. **Canvas:** one shared **Draw.io** file (`.drawio`, synced via Google Drive) or one FigJam board. Everyone works in their own labeled region; we merge into a single connected diagram at the end.
2. **State naming:** `S0` = start. Each member gets a reserved number block so IDs never collide:
   - Member 1 → `S1`–`S19`
   - Member 2 → `S20`–`S39`
   - Member 3 → `S40`–`S59`
   - Error states → `E1`–`E4` (shared, drawn once)
3. **Shapes:** single circle = non-accepting · double circle = accepting/emit · double circle with a red border = error state.
4. **Accepting-state labels:** every accepting state is labeled with the token it emits, e.g. `S20 → Number`.
5. **Backtracking notation:** an arrow that consumes no input and pushes the character back is drawn **dashed** and labeled `other / *` (star = retract one char). Put this in a legend.
6. **Transition alphabet vocabulary** (slide 8 permits these): `letter`, `digit`, `other`, plus literal characters. Also allowed: `ws` (space/tab/newline), `not-newline`, `not-quote`.
7. **Legend box** — mandatory, top-left of the diagram. Defines `letter`, `digit`, `other`, `ws`, the dashed-arrow rule, and the shape key.
8. **Everything routes back to `S0`** after emitting a token or an error. One hub, no orphan sub-machines.

---

## 4. Phase timeline

| Phase | What happens | Who | Suggested time |
|---|---|---|---|
| **0. Kickoff** | Read the PDF together, confirm group, skim §2 of this plan | All 3 | 30 min |
| **1. Conventions** | Agree on §3 in a call; create the shared Draw.io file with the legend + `S0` + the 4 error states already placed | All 3 | 45 min |
| **2. Build sub-DFAs** | Each member draws their assigned sub-machine in their own region — **in parallel** | Split (§5) | 2–3 days |
| **3. Integration** | Wire all sub-machines to `S0`, resolve the 3 shared-character conflicts (§6) | All 3 | 1 session |
| **4. Validation** | Hand-trace the sample inputs against the diagram, fix what breaks (§7) | Split (§7) | 1 session |
| **5. Assembly** | Export to PDF, add title page + surnames + COA, final read-through | Member 1 leads, all review | 1 hour |
| **6. Submit** | One member uploads | 1 person | — |

---

## 5. Task division — 3 members

The split is by **sub-machine**, sized so the three loads are roughly even. Member 2 gets fewer states but the hardest logic; Member 3 gets many states but each is simple.

---

### Member 1 — Core hub, Identifiers, Whitespace, Comments, EOF
*Role: owns the skeleton everyone else plugs into. Start first — Members 2 and 3 depend on `S0` existing.*

**Deliverables**

- [ ] **`S0` start state** — the central hub with the complete outgoing dispatch: `letter`/`_` → identifier, `digit` → number, `"` → string, `/` → comment-or-divide, each operator char, `ws` → self-loop, `EOF` → accept, anything else → `E1`.
- [ ] **Identifier sub-DFA** — `(letter | _)(letter | digit | _)*`
  - `S0 --letter|_--> S1`, `S1 --letter|digit|_--> S1` (self-loop), `S1 --other--> ACCEPT Identifier` (dashed, backtrack).
  - Verify: `_`, `_x`, `x_`, `_x_`, `numb3Rs`, `under_sc0res` are all single valid Identifiers.
- [ ] **Whitespace handling** — `S0 --space|tab|newline--> S0` self-loop. Annotate that newline bumps the line counter (needed for error messages).
- [ ] **Comment sub-DFA** — `S0 --/--> S2`; `S2 --/--> S3` (in comment); `S3 --not-newline--> S3`; `S3 --newline--> S0` **emitting nothing**. The `S2 --other--> ACCEPT Divide` (backtrack) arrow is **Member 3's**, coordinate at integration (§6.1).
- [ ] **EndOfFile** — `S0 --EOF--> ACCEPT EndofFile`. Also decide what happens on EOF mid-comment (falls out to EOF cleanly).
- [ ] **Legend box** and the diagram title/header block.
- [ ] **Keyword note** — a small annotation near the Identifier accept state: *"Identifiers are checked against the keyword table (PRINT, IF, ELSE, ENDIF, SQRT, AND, OR, NOT — case-sensitive) in a post-scan step; not shown in DFA per spec."*

**Watch out for**

- The `/` conflict with Member 3's `Divide` (§6.1).
- A comment that is the last line with no trailing newline (`sample_input_2.txt:17`).
- `_` alone is a valid Identifier (confirmed in `sample_output_scan_2.txt:91`).

**Also owns:** final PDF assembly in Phase 5.

---

### Member 2 — Numbers (whole / float / exponent) + `E2` Invalid Number Format
*Role: the hardest single piece. Fewest states, most edge cases. Give this to whoever is most comfortable with regex-to-DFA conversion.*

**Deliverables**

- [ ] **Whole number** — `digit digit*`. Accepting, emits `Number`.
- [ ] **Float** — `digit+ . digit+`. Note: the dot **must** be followed by at least one digit. `73.` is an error, not `Number 73`.
- [ ] **Exponent** — `(whole | float)(e|E)(ε | + | -)(digit+)`. The sign is optional; digits after it are **not**.
- [ ] **`E2` Invalid Number Format trap** with every arrow that leads into it:
  - from `digit+ .` on a non-digit → `73.`, `1.e`
  - from `… e/E` on a non-digit-non-sign → `55.2e`, `3.2ea`
  - from `… e/E (+|-)` on a non-digit → `3e+e`
  - from any **number-accepting** state on `.` → `111.222e333.444`, `.1e1.` (see §2.2 — this is the arrow people forget)
- [ ] **Backtrack arrows** from all three accepting states on `other`.
- [ ] A short written note on why `-` is not part of a number literal (§2.4) — the grader will look for this.

**Suggested state skeleton** (`S20`–`S39` block)

```
S0  --digit--> S20 (accept Number)         whole
S20 --digit--> S20
S20 --.-->     S21                          seen dot, need digit
S21 --digit--> S22 (accept Number)         float
S21 --other--> E2                           "73." / "1.e"
S22 --digit--> S22
S20 --e|E-->   S23                          exponent marker (from whole)
S22 --e|E-->   S23                          exponent marker (from float)
S23 --+|--->   S24                          signed exponent
S23 --digit--> S25 (accept Number)
S23 --other--> E2                           "55.2e"
S24 --digit--> S25
S24 --other--> E2                           "3e+e"
S25 --digit--> S25
S20 --.--> / S22 --.--> / S25 --.--> E2     see §2.2
```

**Validate against:** `12321`, `98.76`, `1.2e45`, `9e87`, `7E+8`, `6E-5`, `9.8E+7`, `6.5e4`, `3e+21`, `111.222e333`, `123211231`, `98.3243246` (all valid) and `73.`, `55.2e`, `.1e1.`, `1.e`, `3.2east`, `3e+e3`, `111.222e333.444` (all errors).

---

### Member 3 — Operators, Punctuation, Strings + `E1`, `E3`, `E4`
*Role: highest state count, but each branch is short and mechanical.*

**Deliverables**

- [ ] **Single-char, no lookahead needed** — `;` Semicolon · `,` Comma · `(` LeftParen · `)` RightParen · `+` Plus · `-` Minus · `=` Equal. One state each, straight to accept.
- [ ] **Two-char lookahead operators** — each needs an intermediate state plus a backtracking `other` exit:
  - `:` → `:=` **Assign**, else **Colon**
  - `*` → `**` **Raise**, else **Multiply**
  - `<` → `<=` **LTEqual**, else **LessThan**
  - `>` → `>=` **GTEqual**, else **GreaterThan**
  - `!` → `!=` **NotEqual**, else → **`E4`** (there is no standalone `!` token)
  - `/` → **Divide** on `other`; the `//` branch is Member 1's (§6.1)
- [ ] **String sub-DFA** — `S0 --"--> Sx`; `Sx --not-newline-and-not-quote--> Sx`; `Sx --"--> ACCEPT String`; `Sx --newline|EOF--> E3 Unterminated`.
- [ ] **`E1` Illegal Character** trap — the catch-all from `S0` for any character with no transition (`#`, `@`, `&`, `$`, `%`, `.` at start state, …).
- [ ] **`E3` Unterminated** and **`E4` reading character `!`** traps.

**Watch out for**

- `**` vs `* *` — `** * *` must give `Raise, Multiply, Multiply`, so the `*` intermediate state needs a proper backtracking exit.
- `=!=!ouch` → `Equal, NotEqual, E4, Identifier uch`. Note the reference consumed `!o` (two chars) on the `E4` path — flag this at integration; we need to decide whether to draw `E4` as consuming the lookahead char or backtracking it. **Recommend: draw it as consuming, matching the reference.** Add a note.
- `:;=:=;:` → `Colon, Semicolon, Equal, Assign, Semicolon, Colon` — good stress test for the `:` lookahead.
- The string sub-machine must swallow `//` (§2.7).

---

## 6. Integration checklist (Phase 3, all three together)

Three characters are claimed by two members each. Resolve these explicitly, in one sitting, with all three present:

1. **`/`** — Member 1 (Comment `//`) vs Member 3 (`Divide`).
   Resolution: **one** shared intermediate state after `S0 --/-->`. `--/-->` continues to Member 1's comment loop; `--other-->` is Member 3's backtracking exit to `Divide`. Draw the state once, both members agree on its ID.
2. **`e` / `E`** — Member 1 (letter → Identifier) vs Member 2 (exponent marker).
   Resolution: no conflict at `S0` — `e` from the start state is always a letter. The exponent transition only exists **from digit states**. Confirm on the canvas that no `e` arrow leaves `S0` toward the number sub-machine.
3. **`+` / `-`** — Member 3 (Plus/Minus) vs Member 2 (exponent sign).
   Resolution: no conflict at `S0` — the sign transition only exists **from the `e`/`E` state**. Confirm visually.

Also at integration:

- [ ] Every accepting state has a **return arrow to `S0`**.
- [ ] All four error states are drawn **once**, shared, with every inbound arrow attached.
- [ ] Every state ID is unique (the number-block convention in §3.2 should guarantee it).
- [ ] The legend matches what was actually drawn.
- [ ] No unreachable states, no dead ends that aren't error traps.

---

## 7. Validation — hand-trace the samples (Phase 4)

Split the 9 sample pairs so each is traced by **someone who did not design the relevant sub-machine** — you catch more that way.

| Sample | Why it matters | Assigned to |
|---|---|---|
| `sample_input_1` | Baseline realistic program | Member 3 |
| **`sample_input_2`** | **The torture test.** Every error type, keyword casing, unterminated strings, operator soup | All 3 together |
| `sample_input_3` | Normal program | Member 1 |
| **`sample_input_4`** | **Number edge cases.** `.71`, `73.`, `55.2e`, `111.222e333.444`, `.1e1. 1.e.1`, `e+e` | Member 1 & 3 (not Member 2) |
| `sample_input_5` | Clean small program | Member 2 |
| `sample_input_6` | Normal program | Member 2 |
| `sample_input_7` | Normal program | Member 3 |
| **`sample_input_8`** | The `!` error (`IF 2!1:`), nested strings with `#` | Member 2 |
| `sample_input_9` | Normal program | Member 1 |

**Method:** walk the input character-by-character through the drawn diagram and write down the token sequence. Diff it against the matching `sample_output_scan_N.txt`. Any mismatch is a diagram bug — log it, don't patch it silently.

**Known discrepancy — ignore it:** the reference outputs' **line numbers are off** in places (`sample_output_scan_4.txt` reports `line 7` for errors on input lines 8 and 9). That's a line-counter bug in the reference implementation, not a DFA issue. It does not affect our diagram. Match the **token/error sequence**, not the line numbers.

---

## 8. Final submission checklist (Phase 5)

- [ ] Diagram exported to **PDF**, fits legibly — if it's cramped, split across 2 pages (main hub page + sub-machine detail page) rather than shrinking the text
- [ ] Text is readable at 100% zoom
- [ ] Legend present and accurate
- [ ] **All three surnames** written out on the document
- [ ] **COA (Certificate of Authorship)** included
- [ ] 4 error states clearly marked (meets the "3–4 lexical errors" requirement)
- [ ] Keyword note present (shows we understood slide 9)
- [ ] One member submits — confirm in the group chat who, so nobody double-submits
- [ ] Keep the source `.drawio` file — **we will reuse this diagram for the Scanner implementation next phase**

---

## 9. Open questions for the instructor

Ask early — the answers could change the diagram.

1. Should the **error states** be drawn as a single generic `ERROR` state with four labeled inbound arrows, or as four visually distinct states? (This plan assumes four distinct states, which reads better and matches "3–4 lexical errors.")
2. Should the **keyword lookup** appear in the diagram at all? Slide 9 says we don't have to — we plan to add it only as a text annotation.
3. Is **backtracking / lookahead** expected to be drawn explicitly (dashed `other/*` arrows), or is it assumed understood?
4. Does the `EndOfFile` token need its own state in the diagram, or is it implicit?

---

## 10. Quick reference — who owns what

| Area | Owner |
|---|---|
| `S0` hub, dispatch table | Member 1 |
| Identifier | Member 1 |
| Whitespace, line counting | Member 1 |
| Comment `//` | Member 1 |
| EndOfFile | Member 1 |
| Legend, title block, PDF assembly | Member 1 |
| Whole / Float / Exponent numbers | Member 2 |
| `E2` Invalid Number Format | Member 2 |
| All operators + punctuation | Member 3 |
| Two-char lookahead operators | Member 3 |
| String literal | Member 3 |
| `E1` Illegal Character | Member 3 |
| `E3` Unterminated | Member 3 |
| `E4` reading character `!` | Member 3 |
