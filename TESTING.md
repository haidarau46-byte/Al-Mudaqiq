# تحليل تغطية الاختبارات / Test Coverage Analysis

**الحالة الحالية: لا توجد اختبارات إطلاقاً — التغطية صفر ٪.**

**Current state: there are no tests at all — coverage is 0%.**

The repository contains exactly two files: `README.md` and `index.html` (2,163 lines,
~1,550 of them JavaScript inside a single `<script>` block). There is no `package.json`,
no test runner, no CI workflow, no linter, and no assertions of any kind.

This document proposes where to start. Every defect listed below was **reproduced by
extracting the functions and executing them**, not inferred by reading — each one is a
concrete, ready-made first test case.

---

## 0. The structural blocker (fix this first)

All logic lives inside `<script>` in `index.html` with no module boundary and no exports,
so nothing can be imported by a test. Worse, the top-level code is not inert: line 801
runs `el("input")`, `el("checkBtn")`, … at parse time, and dozens of
`el(...).addEventListener(...)` calls follow. Any attempt to load the script outside a
fully-built DOM throws immediately.

**Recommendation — preserve the single-file property, it is a stated product feature.**
Do *not* split the app into modules just to make it testable. Instead:

1. Move the pure logic into one clearly-delimited region of the script, fenced by
   sentinel comments (`/* --- PURE:START --- */ … /* --- PURE:END --- */`), containing
   only functions with no DOM or network access.
2. Add a test harness that reads `index.html`, slices out that region, and `eval`s it.
   This is exactly how the defects below were reproduced, and it costs one small helper:

   ```js
   // test/harness.js
   const fs = require('fs');
   const html = fs.readFileSync(require.resolve('../index.html'), 'utf8');
   const pure = html.split('/* --- PURE:START --- */')[1].split('/* --- PURE:END --- */')[0];
   module.exports = new Function(pure + '; return {' + [
     'runLocalChecks','applyLocalFixes','splitChunks','extractJson','salvageByMode',
     'mergeResults','harvestTextFields','detectLangName','isNonArabic','detectQuran',
     'stripQ','formatDur','esc','chunkSizes','unescapeJsonStr'
   ].join(',') + '};')();
   ```

3. Use Node's built-in `node:test` + `node:assert`. Zero dependencies, no build step, no
   `node_modules` — which keeps the repo as light as it is today.

Roughly 15 functions become testable this way with **no change to how the app ships**.
That is where the highest return is, and everything in §1–§3 below depends on it.

---

## 1. Highest priority — text-transformation correctness

This is a proofreading tool. If it silently corrupts the user's text, that is the worst
possible failure, and it is currently untested end to end.

### 1.1 `applyLocalFixes` destroys paragraph structure — **confirmed**

The rule at line 669, `out.replace(/\s+([،؛.!؟:])/g,"$1")`, treats `\s` as including
newlines:

| Input | Output |
|---|---|
| `"سطر أول\n. سطر ثان"` | `"سطر أول. سطر ثان"` |

The line break is gone. Any user who clicks «إصلاح الكل» on a multi-paragraph document
where a line happens to begin with punctuation loses that break, with no warning and no
undo. The fix is to restrict the class to horizontal whitespace (`[^\S\n]+`), but the
test should come first.

**Tests to add:** newline preservation, and an idempotence property — `applyLocalFixes(applyLocalFixes(x)) === applyLocalFixes(x)`
for a corpus of inputs. (Idempotence currently *does* hold on everything I tried; that is
worth locking in before someone adds a rule that breaks it.)

### 1.2 Word-boundary set misses standard Arabic quotation marks — **confirmed**

`BOUND` (line 617) omits the guillemets `«»` that are the normal quoting convention in
Arabic typography, and omits the hyphen:

| Input | `applyLocalFixes` output | Expected |
|---|---|---|
| `"لاكن"` | `"لكن"` ✅ | `"لكن"` |
| `"«لاكن»"` | `"«لاكن»"` ❌ | `"«لكن»"` |
| `"لاكن-النص"` | `"لاكن-النص"` ❌ | `"لكن-النص"` |

`runLocalChecks` misses these identically, so the quick-check panel does not even flag
them. Every one of the 20 entries in `COMMON_MISTAKES` is affected.

**Tests to add:** a table-driven test running all 20 `COMMON_MISTAKES` keys through every
boundary context — start/end of string, space, `،`, `.`, `«»`, `()`, hyphen, digit,
newline. That is one loop and ~200 assertions, and it pins the highest-traffic logic in
the app.

### 1.3 `runLocalChecks` ↔ `applyLocalFixes` must agree

These two functions encode the same rule set twice, independently. They can drift: one
can report a problem the other cannot fix, stranding the user with a warning chip that
never clears.

**Test to add (property):** for any input, `runLocalChecks(applyLocalFixes(t))` must be
empty. This currently passes on my samples — capture it before it regresses.

---

## 2. High priority — API response parsing

`extractJson` → `salvageByMode` → `mergeResults` is the pipeline that turns a model reply
into what the user sees. It is defensive, hand-rolled, regex-based parsing of untrusted
input, and it is completely untested. Two confirmed content-corruption bugs:

### 2.1 `extractJson` strips backticks from legitimate content — **confirmed**

Line 1045 runs `raw.replace(/```json|```/g,"")` across the **entire** response, including
inside JSON string values:

| Raw response | Parsed `corrected_text` |
|---|---|
| `{"corrected_text":"يقول: ```code``` هنا"}` | `"يقول: code هنا"` ❌ |

Any user proofreading text that contains a triple backtick — a code snippet, a technical
document, a Markdown draft — gets it silently deleted from the corrected output. The fix
is to strip fences only at the string's edges.

### 2.2 The salvage path converts real newlines to spaces — **confirmed**

Line 1050's control-character fallback, `m[0].replace(/[\u0000-\u001f]+/g," ")`, is
reached whenever a response contains a literal newline inside a string:

| Raw response | Parsed |
|---|---|
| `{"corrected_text":"سطر\nمكسور"}` | `"سطر مكسور"` ❌ |

Multi-paragraph proofreading results collapse into a single run-on paragraph.

### 2.3 `salvageByMode` loses everything on nested objects — **confirmed**

The regex `/\{[^{}]*?"original"[\s\S]*?\}/g` cannot match an error object containing a
nested object:

```js
salvageByMode('imla', '{"errors":[{"original":"a","meta":{"x":1},"correction":"b"}]}')
// → null   (whole truncated response discarded, user sees a generic failure)
```

**Tests to add for §2:** a fixture corpus of real-world response shapes — clean JSON,
fenced JSON, JSON with prose on both sides, truncated mid-string, truncated mid-array,
SSE/streaming bodies (for `harvestTextFields`), an `error` envelope, an empty `content`
array, and nested-object payloads. Each of the six modes has a different salvage branch
and a different merge branch; none is exercised today. This is the densest concentration
of untested branching in the codebase.

### 2.4 `mergeResults` spacing and `splitChunks` sizing

- `mergeResults('tashkeel', [{tashkeel_text:'أوّل'}, {}, {tashkeel_text:'ثانٍ'}])`
  → `"أوّل  ثانٍ"` (double space, because a failed chunk contributes `""` and is still
  joined with `" "`). Confirmed.
- `splitChunks` emits chunks up to **1.6×** `maxLen` by design (line 1070) — a `maxLen`
  of 450 produced a 650-character chunk. That is fine if intended, but it is an
  undocumented invariant that token-budget assumptions depend on.

**Tests to add:** losslessness (`splitChunks(t, n).join('') === t` — this holds today and
is the single most valuable invariant in the function), max-size bounds, empty input,
input with no sentence terminators, and a single unbroken 2,000-character token.

---

## 3. Medium priority — output escaping

`esc()` (line 805) escapes `&`, `<`, `>`, `"` but **not** `'`. The app is safe today only
because every interpolated HTML attribute happens to use double quotes. That is an
invariant no test enforces, and the app renders attacker-influenceable strings — model
output, imported lexicon files, and a third-party API — straight into `innerHTML`.

Two places bypass `esc()` entirely:

| Location | Unescaped value | Source |
|---|---|---|
| `index.html:1910` | `x.numberInSurah` | third-party `api.alquran.cloud` response |
| `index.html:1999` | `MODE_NAMES[x.mode] \|\| x.mode` | `localStorage` history entry |

Neither is trivially exploitable right now, but both are unescaped interpolation of data
the app does not control, and the history one round-trips through storage that another
script on the same origin could write.

**Tests to add:** `esc()` character-by-character coverage including `'`; a render test
per mode feeding `<img src=x onerror=alert(1)>` and `" onload="` through every field
(`original`, `correction`, `explanation`, `rule`, `speaker`, `delivery`, `glossary.fusha/lahja`,
`notes`, `improvements`) asserting no element is created; and the same for a hostile
lexicon import file and a hostile Quran-API fixture. These need a DOM — `jsdom`, or
Playwright if you would rather test the real thing.

---

## 4. Medium priority — persistence and state

`localStorage` is the app's only durable store and every access is wrapped in
`try{}catch(e){}` that swallows the error silently.

### 4.1 Favourites are silently evicted — **confirmed**

`storeHist` hard-truncates to 30 entries (line 1970) with no regard for the `fav` flag.
Simulating a user who stars a result and then runs 30 more checks:

```
entries kept: 30
favourites surviving: 0   → the starred item is gone
```

The favourites view promises «اضغط ☆ على أي نتيجة في السجل لتثبيتها هنا» — *pin it here* —
but pinning does not survive. Favourites should be exempt from the cap, or capped
separately.

### 4.2 Quota exhaustion is invisible

Each history entry stores the full result object plus up to 8,000 characters of source
text, ×30 entries. On overflow, `storeHist`'s catch block swallows `QuotaExceededError`
and the user's history simply stops saving with no message.

**Tests to add:** history cap behaviour with favourites present; round-trip of every
mode's result shape; recovery from corrupt/partial JSON in each of the three storage keys
(`mudaqqiq_settings`, `mudaqqiq_hist`, `mudaqqiq_lex`); `lexTogglePair` add/remove/dedupe
and its 80-entry cap; and the lexicon import parser (line 927), which splits on four
different separators (`←`, `->`, `—`, `:`) and has never been exercised against a
malformed file.

---

## 5. Lower priority — locale correctness

`formatDur` produces grammatically incorrect Arabic, which is a poor look in a tool whose
entire purpose is grammatical correctness. Confirmed outputs:

| Input | Output | Correct Arabic |
|---|---|---|
| `3600000` | `"60 دقائق"` | `"60 دقيقة"` — 11+ takes the singular |
| `125000` | `"دقيقتين و5 ثانية"` | `"دقيقتين و5 ثوانٍ"` — 3–10 takes the plural |
| `undefined` | `"NaN دقائق"` | should degrade gracefully |

The `NaN` case is reachable: `renderHistInto` calls `formatDur(x.dur)` on stored entries,
and any entry written before `dur` existed — or hand-edited — renders «NaN دقائق» in the
history list.

**Tests to add:** a table covering the Arabic number-agreement boundaries (1, 2, 3, 10,
11, 60, 100) plus `undefined`/`null`/`NaN`/negative.

Also worth pinning: `detectLangName` returns `""` below 4 characters and falls back to
`"اللاتينية"` when no stop-words match; `detectQuran` returns `false` for undiacriticised
Quranic text (`"بسم الله الرحمن الرحيم"` → `false`), so the do-not-modify guard in
`buildBody` does **not** engage for plainly-written verses. That may be an acceptable
trade-off, but it is a significant behaviour that should be asserted deliberately rather
than left implicit.

---

## 6. Not worth unit-testing — cover with a few browser tests instead

`callClaude`'s retry/fallback ladder (fetch → XHR → retry → XHR again), `readWholeBody`'s
stream pump, the cancellation path through `RUN.cancelled`, the mic/OCR/PDF integrations,
and the two-pass `lahja` deepening are all heavily DOM- and network-bound. Unit-testing
them would mean mocking more than you assert.

**Recommendation:** three or four Playwright tests against a stubbed `api.anthropic.com`,
covering the flows that actually break in production — happy path per mode, a truncated
response reaching the salvage path, a mid-run cancel, and an auth error surfacing the
right Arabic message. Chromium is already available in this environment.

---

## Suggested order

| # | Area | Effort | Why first |
|---|---|---|---|
| 1 | Pure-logic harness (§0) | S | Nothing else is possible without it |
| 2 | `applyLocalFixes` / `runLocalChecks` table + properties (§1) | M | Two confirmed corruption bugs; core promise of the app |
| 3 | Response-parsing fixture corpus (§2) | M | Two more confirmed corruption bugs; densest untested branching |
| 4 | Escaping tests (§3) | M | Needs a DOM; closes two unescaped sinks |
| 5 | Storage round-trip + favourites cap (§4) | S | Confirmed data-loss bug |
| 6 | `formatDur` locale table (§5) | S | Quick, visible, embarrassing to ship |
| 7 | Playwright smoke flows (§6) | M | Catches wiring regressions the units cannot |

Items 1, 2, 5 and 6 alone are a day's work, require no dependencies beyond Node's built-in
test runner, and would already cover four confirmed defects that ship today.
