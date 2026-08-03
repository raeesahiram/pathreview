# Work Journal

## Week 7 — Issue selection

**Issue link:** https://github.com/jamjamgobambam/pathreview/issues/73

**Issue title:** `pii_scrubber.py` test coverage doesn't include address formats

**Tier:** [x] Tier 1  [ ] Tier 2  [ ] Tier 3

**Problem summary:**
The safety layer's PII scrubber (`safety/pii_scrubber.py`) is supposed to redact
personal addresses from generated feedback, but its coverage of real address
formats is incomplete and the one existing address test makes no assertions at
all, so it proves nothing. In practice, numbered street names like "5th Avenue"
or "42nd Street", house numbers with a unit letter like "221B", and PO Box
addresses are not redacted, meaning a user's address could leak through. A
successful fix adds meaningful unit tests covering these address formats and
tightens the `street_address` matching so genuine addresses are caught without
over-redacting ordinary text.

**Branch name:** test/73-pii-scrubber-address-formats

**Setup confirmation:** [x] App runs locally at localhost:5173

**Cohort ledger:** [ ] Issue added to cohort ledger

### Is this right for me? — checklist reasoning

Worked through the CodePath "Is This Issue Right for Me?" checklist:

**Part 1 — Understanding the Issue**

- [x] *I can explain the problem and expected behavior in 2–3 sentences without
  reading the issue.* In my words: the PII scrubber is meant to redact home
  addresses from generated feedback, but its address matching misses common
  real-world formats and the existing address test asserts nothing. After a fix,
  addresses like "123 5th Avenue" or "221B Baker Street" get redacted and there
  are real tests proving it.
- [x] *I've located the relevant files and confirmed they exist.*
  `safety/pii_scrubber.py` and `tests/unit/test_pii_scrubber.py` both exist.
- [x] *I can describe a concrete before-and-after.* Before: `scrub("123 5th
  Avenue")` returns the string unchanged (address leaks). After: it returns
  `"[REDACTED]"`, and ordinary text like "Room 101 upstairs" is left untouched.

**Part 2 — Tier Fit**

- [x] *The tier is a realistic match.* This is a Tier 1 issue (label `tier-1`,
  `good first issue`): a localized change in one module plus its test file, no
  cross-module or system understanding required. Appropriate as an early
  contribution rather than reaching for a Tier 3.

**Part 3 — Codebase Readiness**

- [x] *I've found and read the specific code the issue references.* Read the
  `PIIScrubber` class — the `PII_PATTERNS` dict (specifically the
  `street_address` regex), and the `scrub()` and `detect()` methods.
- [x] *I understand the surrounding code well enough to plan the fix.* The
  scrubber just applies each regex in `PII_PATTERNS` via `re.sub`/`re.finditer`;
  the fix is to broaden the `street_address` pattern (allow digits in
  street-name words, a unit letter on the house number) and add a `po_box`
  pattern — no callers need to change.
- [x] *I've read the test file and at least one test end-to-end.* Read
  `tests/unit/test_pii_scrubber.py`, including `test_address_variations` (which
  runs `scrub()` but makes no assertions) and `test_detect_returns_list_of_pii`.

**Part 4 — Scope and Time**

- [x] *Not already claimed (comments + ledger).* No comments claiming issue #73
  on GitHub as of selection. NOTE: I could not view the cohort ledger myself —
  needs a manual confirm that it isn't taken there.
- [x] *Scope realistic for Weeks 8–9.* This is a small Tier 1 change (~3–6h of
  the regex-and-tests kind) that fits comfortably within the two-week window.
- [x] *No blockers or dependencies.* The issue body references no "blocked by"
  issue and the code is self-contained.

**Scope boundary:** I'll keep this change to address-format coverage in the PII
scrubber. A related-but-separate problem I noticed — the phone-number regex
failing on formats like `(555) 123-4567` — is out of scope and belongs in its
own issue.

## Week 8 — Reproduction & solution planning

**Reproduction commit link:** https://github.com/raeesahiram/pathreview/commit/024b02339727694f5ae7f9ba7822bd50fd620bf4

**Reproduction summary:**
I ran `scrub()` / `detect()` against a set of real address formats and observed
that `123 5th Avenue`, `221B Baker Street`, and `PO Box 1234` pass through
unredacted (and `detect()` returns `[]`), while the existing address tests still
report as passing because `test_address_variations` has no assertions.

**PLAN.md link:** https://github.com/raeesahiram/pathreview/blob/main/PLAN.md

**Walkthrough video (recommended):** _(not recorded)_

**Blockers or open questions:**
Whether ZIP-code redaction is expected as part of "address formats." I've scoped
it out of the fix for now (a bare 5-digit pattern would over-redact any number)
and will confirm before finalizing in Week 9.

### Reproduction detail

Confirmed the issue is real on an untouched `main` (working tree clean, no code
changes made during reproduction).

**Where it lives:**
- `safety/pii_scrubber.py:18` — the `street_address` regex in `PII_PATTERNS`.
- `tests/unit/test_pii_scrubber.py:193` — `test_address_variations`, which has
  no assertions.

**How to reproduce (functional leak):**

```python
from safety.pii_scrubber import PIIScrubber
s = PIIScrubber()
print(s.scrub("123 5th Avenue"))     # -> "123 5th Avenue"   (should be [REDACTED])
print(s.scrub("221B Baker Street"))  # -> "221B Baker Street" (should be [REDACTED])
print(s.scrub("PO Box 1234"))        # -> "PO Box 1234"       (should be [REDACTED])
print(s.detect("221B Baker Street")) # -> []                  (nothing flagged)
```

Observed vs. expected:

| Input | `scrub()` today | Expected |
|---|---|---|
| `123 Main St` | `[REDACTED]` | `[REDACTED]` (already works) |
| `123 5th Avenue` | `123 5th Avenue` | `[REDACTED]` |
| `123 42nd Street` | `123 42nd Street` | `[REDACTED]` |
| `221B Baker Street` | `221B Baker Street` | `[REDACTED]` |
| `PO Box 1234` | `PO Box 1234` | `[REDACTED]` |

**Root cause:** the name portion `[A-Za-z\s]+` rejects digits, so numbered
street names (`5th`, `42nd`) never match; `\d+` alone won't accept a lettered
house number (`221B`); and there is no PO-box pattern at all.

**How to reproduce (test-coverage gap):**

```
$ .venv/bin/python -m pytest tests/unit/test_pii_scrubber.py -q -k "address"
3 passed
```

The address tests pass even though the leaks above exist, because
`test_address_variations` calls `scrub()` in a loop but never asserts anything —
so there is effectively no real address-format coverage.

Reproduction is deterministic (pure regex; no network or LLM involved).

## Week 9 — Solution building & PR submission

### Check-in 1 (mid-week)

**Current progress:**
Fix is fully implemented against `PLAN.md`. Done:
- Step 1 (tests first) — replaced the assertion-less `test_address_variations`
  with real assertions and added tests for numbered streets, lettered house
  numbers, abbreviated suffixes, PO boxes, an embedded address, negative
  (no-false-positive) cases, and `detect()` coverage for `street_address` /
  `po_box`.
- Step 2 (broaden `street_address`) — the name portion now allows digits
  (`5th`, `42nd`), the house number takes an optional unit letter (`221B`), the
  match is bounded to 1–4 name words, and a trailing `\b` stops a suffix from
  matching inside a longer word (this also fixed the pre-existing over-redaction
  where `Pl` matched inside "applications").
- Step 3 (add `po_box` pattern) — covers `PO Box` / `P.O. Box` and case variants.
- Step 4 (verify) — the reproduction snippet from Week 8 now redacts all leaking
  formats; `pii_scrubber` tests went from 5 failed / 20 passed to 4 failed /
  29 passed (the 4 are the out-of-scope phone bug).

**Next steps:**
Open the PR (ready, not draft) with the template filled in, then submit the
branch URL via the portal.

**Blockers:**
The ZIP-code question from Week 8 — resolved by keeping ZIP out of scope (a bare
5-digit pattern over-redacts any number); noted as a possible follow-up rather
than added blind.

---

### Check-in 2 (end of week)

**PR link:** _(not submitted)_

**Branch:** `test/73-pii-scrubber-address-formats`

**What you built:**
Broadened the `street_address` regex in `safety/pii_scrubber.py` so it redacts
numbered street names (`123 5th Avenue`), lettered house numbers (`221B Baker
Street`), and abbreviated suffixes (`10 Downing St.`), and added a `po_box`
pattern for `PO Box` / `P.O. Box`. A trailing word boundary keeps a street-type
suffix from matching inside ordinary words, so genuine addresses are caught
without over-redacting prose.

**Tests added or updated:**
`tests/unit/test_pii_scrubber.py` — rewrote `test_address_variations` to actually
assert, and added `test_numbered_street_names_redacted`,
`test_lettered_house_number_redacted`,
`test_abbreviated_suffix_with_period_redacted`, `test_po_box_redacted`,
`test_address_embedded_in_sentence`, `test_address_no_false_positives`,
`test_detect_street_address_pii`, and `test_detect_po_box_pii`.

**Self-review confirmation:** [x] make check passes  [x] make test-unit passes

> In this codebase both commands have documented pre-existing failures unrelated
> to this issue, so per the course guidance "passes" means my change introduces
> **no new failures**. Verified:
> - `make test-unit`: 53 failed on clean `main` → 52 with my change. My change
>   only touches `pii_scrubber`, whose tests improved from 5 failed to 4 failed
>   (I fixed `test_mixed_pii_and_text` and added 8 passing tests). The remaining
>   4 `pii_scrubber` failures — and all 48 failures in other modules — are
>   pre-existing. The 4 are the parenthesized-phone bug (`(555) 123-4567`), which
>   I scoped out in Week 7 as a separate phone-regex issue.
> - `make check`: ~181 pre-existing lint errors codebase-wide. On my two files
>   ruff went from 6 errors to 5 (I removed one, added none), `mypy safety/` is
>   clean, and black's only diff is in `detect()` — a method I did not touch.

**Draft PR feedback received from:** none

## Week 10 — Iteration & reflection

### Reviewer feedback

**Feedback received:** [ ] Yes  [x] No — still awaiting review

**Summary of feedback:**
No review came in. The work is committed on
`test/73-pii-scrubber-address-formats` (the fix in `8647732`, the tests in
`ea7e170`), but I never opened the PR against the upstream repo — I added a PR
link to my Week 9 check-in and then removed it because it didn't point to a real
submitted PR. So there was nothing for a maintainer to review, and the honest
state at the close of the module is: change finished and self-reviewed, not yet
submitted upstream.

**How you responded:**
Rather than paper over that, I'm leaving the check-in accurate ("not submitted")
and treating the missing submission as the main process lesson of the module
(see below). The branch is in a state where opening the PR is a mechanical next
step — the template content is already written into the Week 9 entry.

---

### Reflection

**What was harder than you expected?**
Two things. First, the regex itself was deceptively fiddly. The visible bug was
"numbered streets don't redact," but the fix that actually made the tests green
was a *trailing* `\b` on the street-suffix alternation — without it, `Pl` matched
inside "applications" and the scrubber over-redacted ordinary prose. So the hard
part wasn't catching more addresses, it was catching more without catching less,
and that only showed up because I wrote the negative ("no false positives") test
before I was confident the pattern was done. Second, and harder: deciding what
"passes" even means in a repo that ships with ~53 failing unit tests and ~181
lint errors on a clean `main`. I expected a green baseline to measure against and
there wasn't one. I had to redefine success as "introduces no new failures" and
then actually prove it by diffing counts before and after (53 → 52 unit, 6 → 5
ruff on my files), which is a much more annoying thing to stand behind than a
green checkmark.

**What did you learn about working in a large codebase?**
That the most important skill isn't writing the fix, it's drawing the boundary
around it. On my own project everything is in scope by definition. Here, I kept
tripping over adjacent broken things — the parenthesized-phone-number bug
(`(555) 123-4567`), the question of whether ZIP codes count as "address
formats" — and the discipline was to *not* fix them. I scoped the phone bug out
in Week 7 as its own issue and kept ZIP out because a bare 5-digit pattern
over-redacts any number. Both were tempting one-liners; both would have made the
change harder to review and muddied what the PR was actually claiming. In
someone else's production code, a small honest diff that does exactly what it
says is worth more than a clever big one.

**How did AI tools help — and where did they fall short?**
Most useful for the mechanical middle: drafting regex variants, enumerating
address formats to test (numbered streets, lettered house numbers, abbreviated
suffixes with periods, embedded-in-a-sentence), and generating the reproduction
snippet quickly so I could confirm the leak on an untouched `main`. Where it fell
short was exactly the judgment calls that made this a real contribution: it
couldn't tell me that the repo's baseline was already red (I had to run it and
see), it couldn't decide the scope boundary for me, and it happily would have
"finished" by declaring tests pass without me insisting on the before/after count
diff. The over-redaction-of-"applications" behavior also came from actually
running the suite, not from reasoning about the pattern. AI compressed the typing;
the verification and the scoping were on me.

**What would you do differently if you started over?**
Open the PR early as a draft, in Week 8, right after reproduction — before the
fix even exists. The single concrete failure of this module is that a finished,
self-reviewed change never got submitted, and that happened because I treated PR
submission as a final ceremony instead of a container I could fill incrementally.
A draft PR from the start would have given me a real URL to put in every
check-in, surfaced CI behavior against the actual baseline sooner, and made
submission a non-event. Everything technical went fine; the thing I'd change is
purely process sequencing.

**What are you most proud of from this module?**
That I didn't hide the messy parts. The existing `test_address_variations`
"passed" while asserting nothing — it was green and worthless — and the easy path
was to leave that illusion in place. Instead I made the coverage real, wrote a
negative test that caught my own over-redaction, and documented an honest
"no new failures" standard against a genuinely broken baseline rather than
claiming a clean pass I couldn't back up. The redaction of one leaked address
matters, but what I'm actually proud of is the habit of preferring an accurate
uncomfortable status over a tidy false one.

