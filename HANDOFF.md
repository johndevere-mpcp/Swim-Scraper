# Swim Yardage Parser — Handoff & Lessons Learned

> Written for whoever (human or agent) picks this up next. The previous
> implementation got the *plumbing* right but kept getting the *yardage
> totals* wrong. This document explains why, what we learned, and how to
> get the logic right from the start. **Read the "Start here" section
> before writing any code.**

---

## 0. Start here — do these THREE things first, in order

1. **Resolve the counting definition with the boss (see §3).** This is the
   reason every prior attempt drifted. Do not write a parser until you can
   answer: *"When 3 coaches each write a block for the same morning session
   (different squads/lanes), is the session's total the SUM of all blocks,
   or one representative block?"* Everything downstream depends on this.

2. **Treat the manual Week 1–3 tally (§2) as your test fixture.** Build to
   reproduce those numbers exactly. Do not guess and eyeball — assert
   against ground truth. The prior attempt had no fixture and validated by
   "does the total look plausible," which is how it stayed ~7% off.

3. **Reconsider the input format (§6).** The source document is the real
   problem, not the parser. Decide format before architecture.

---

## 1. What the boss actually wants

Per-group yardage (Sprint / Mid / Distance) broken down:
- across the whole season (per training group),
- per **week** (microcycle),
- per **day** (mini-microcycle = a single day's total),
- *not* per mesocycle.

Team structure / coach→group mapping (this is stable and correct):

| Coach | Group |
|---|---|
| Dave | **Sprint** |
| Josh | **Mid** (covers "Mid" and "Mid-Distance") |
| Noah | **Distance** |

"Whole-team" sets (no group prescribed) were decided to **fold into every
group** (each group swims them), so the three displayed group columns
intentionally sum to MORE than the unique session total. The "unique
total" counts every yard once.

---

## 2. GROUND TRUTH — manual tally, Weeks 1–5 (from the boss)

These are hand-counted by the boss. **Use as regression fixtures.**

```
WEEK 1  (35,800 yd + 3,800 m)
  Mon AM   4,000 yd
  Tue AM   3,800 m            <-- METERS (LCM). keep yards & meters separate
  Wed AM   5,400 + 3,200 yd
  Wed PM   4,300 + 2,550 + 3,300 + 600 + 300 + 300 + 600 + 450 yd
  Thu AM   (skip — dryland, no swim yardage)
  Fri AM   3,500 + 4,700 yd
  Sat AM   450 + 150 + 2,000 yd

WEEK 2  (32,750 yd)
  Mon      (skip — no practice)
  Tue AM   300 + 400 + 500 + 600 + 700 + 200 yd
  Wed AM   4,600 yd
  Wed PM   1,050 + 400 + 300 + 500 + 450 + 300 yd
  Thu AM   3,350 + 300 + 150 + 150 + 150 + 100 + 100 + 50 + 100 + 750 + 100 + 150 + 100 + 100 yd
  Fri AM   4,500 + 4,100 + 1,300 yd
  Sat AM   5,000 + 1,200 + 700 yd

WEEK 3  (73,575 yd)
  Mon AM   5,800 + 3,900 yd
  Mon PM   5,000 + 3,150 yd
  Tue AM   4,400 + 3,200 + 400 + 150 + 150 + 100 + 100 + 100 + 250 + 100 + 150 yd
  Wed AM   2,200 + 4,400 + 3,900 + 6,000 yd
  Wed PM   3,150 + 2,100 yd
  Thu AM   5,700 + 2,775 yd
  Thu AM   4,800 yd
  Fri PM   4,800 yd
  Sat AM   400 + 400 + 200 + 600 + 1,200 + 800 + 600 + 400 + 200 + 600 + 600 + 600 + 200 yd

WEEK 4  (86,675 yd)
  Mon AM   6,000 + 4,300 + 5,000 yd
  Mon PM   5,600 + 300 + 100 + 200 + 300 + 200 + 50 + 200 + 150 + 200 + 2,150 yd
  Tue AM   2,000 + 3,200 + 500 + 625 + 75 + 75 yd
  Wed AM   6,500 + 5,300 yd
  Wed PM   3,300 + 5,400 + 3,200 yd
  Thu AM   2,800 + 450 + 200 + 450 + 7,000 + 1,100 yd
  Fri AM   2,800 + 7,000 + 1,100 + 3,150 + 5,000 + 700 yd
    (NOTE: Thu AM & Fri AM share 2,800 + 7,000 + 1,100 — confirm this is
     real recurrence, not a number carried between days, before trusting Wk4.)

WEEK 5  (114,050 yd)
  Mon AM   6,700 + 5,000 + 5,500 yd
  Mon PM   2,700 + 3,000 + 5,600 yd     (boss read "2703,000" as 2,700 + 3,000)
  Tue AM   7,600 + 3,600 + 3,200 + 700 + 200 yd
  Wed AM   5,300 + 5,100 + 6,500 + 6,550 yd
  Wed PM   6,550 + 2,700 yd
  Thu AM   5,250 yd
  Fri AM   6,700 + 4,650 yd              (boss noted a leading-digit/typo assumption)
  Fri PM   5,200 + 5,500 yd
  Sat AM   4,050 + 6,200 yd

TOTAL Weeks 1-5: 342,850 yd + 3,800 m
  (Wk1 35,800y+3,800m · Wk2 32,750y · Wk3 73,575y · Wk4 86,675y · Wk5 114,050y)
```

### Where the old parser diverged from this ground truth

| Week | Ground truth | Old parser | Diff |
|---|---:|---:|---:|
| 1 | 35,800 yd + 3,800 m | 35,350 yd + 3,800 m | −450 |
| 2 | 32,750 yd | 39,600 yd | **+6,850** |
| 3 | 73,575 yd | 77,750 yd | **+4,175** |
| 4 | 86,675 yd | 82,300 yd | **−4,375** |
| 5 | 114,050 yd | 117,250 yd | **+3,200** |
| **1–5** | **342,850 yd** | 352,250 yd | **+9,400 (~2.7% over)** |

**Diagnosis of the divergence — this is the key lesson:**

- **Week 1 (−450):** parser slightly *under*-counted Wed PM. The boss
  broke Wed PM into 8 sub-pieces (12,400 total); the parser merged them
  into 3 coach blocks (11,950). Minor, parsing-granularity.

- **Weeks 2 & 3 (+11k over):** parser *over*-counted by including
  **secondary parallel-coach blocks** the boss did NOT tally. Examples:
  - W2 Tue: parser counted a main block **+ a separate Coach Josh 2,200
    block**; boss counted only the main 2,700.
  - W2 Thu: parser counted 5,550 **+ a Coach Noah 5,100 block**; boss
    counted only ~5,650.

- **Weeks 4 & 5 (mixed):** the error flips sign — Wk4 the parser is
  *under* by 4,375, Wk5 *over* by 3,200. The divergence is **not a
  consistent bias**, which rules out a single fixable offset. It's the
  combination of (a) the parallel-block counting ambiguity and (b)
  parsing-granularity differences in how a session's sub-pieces are
  grouped. The signed errors partially cancel, so the 5-week total
  (+2.7%) looks better than any individual week — do NOT be fooled by a
  good-looking grand total; validate per week.

  So the parser sums *every* coach's block for a session; the boss's
  manual count does **not** (consistently). **This is the unresolved
  definition in §3.** Neither is obviously "right" — it depends on what
  the boss means by a session total. THIS ambiguity, not regex bugs, is
  the dominant error source.

---

## 3. THE core ambiguity (resolve before coding)

The document frequently has **multiple coaches writing blocks for the same
session slot** (e.g. Wednesday AM has Noah's lane, Dave's lane, Josh's
lane — each a different squad doing different yardage). There are two
defensible interpretations and they differ by ~7%+:

- **(A) Sum all blocks** — every squad swims its own yardage, so the
  session total = Noah's + Dave's + Josh's. Correct for "total athlete-
  yards swum" and for per-group breakdown. (What the old parser did.)

- **(B) One representative block per slot** — the session is "one practice"
  and you count a single block. (Closer to what the boss's manual tally
  did in Weeks 2–3, though not consistently — in W1 Wed AM the boss counted
  both Noah 5,400 AND Dave 3,200.)

The boss's own manual numbers are **internally inconsistent** between weeks
on this point, which tells you the rule was never explicitly defined. **Get
a one-sentence ruling and write it at the top of the spec.** Suggested
questions to ask:

1. "If Noah's distance lane swims 5,000 and Dave's sprint lane swims 3,000
   in the same hour, is that session 8,000 (both) or 5,000 (one)?"
2. "Do optional / small-group secondary blocks (a coach adding a short set
   for 2 swimmers) count toward the week total?"
3. "For per-group numbers specifically — you clearly want each group
   attributed — so isn't summing all blocks required anyway?"

My read: the boss wants per-group totals, which *requires* counting each
group's block (interpretation A). The Week 2–3 manual undercount is likely
the boss missing secondary blocks while hand-tallying — i.e. the **parser
may be more right than the manual tally there.** But confirm; don't assume.

---

## 4. Domain rules & document conventions we reverse-engineered

These are real and hard-won. Whatever you build, it must handle them.

### Units
- Yards by default.
- **"LCM" in a title does NOT mean meters.** Only an explicit meter bracket
  like `[1,500m]` or `[2,100m / 3,600m]` means meters. Keep yards and
  meters in separate tallies; never add them.

### Cumulative checkpoints (the coach's own running total)
The doc marks running totals in FIVE different notations:
- `1500/3000`  — plain slash (most common; right number = cumulative)
- `[1,500m / 2,600m]` — meter bracket (pair or solo `[1,500m]`)
- `[1,500yds / 2,600yds]` — **yard bracket, Coach Josh's style** (commas + "yds")
- ` - 3200` — tail-dash running total at end of a line
- `1200` — a bare number alone on its own line (≥ 500 to avoid noise)

### Group section headers (inside one coach's block)
Lines starting with `SPRINT` / `MID` / `DISTANCE` / `MID-DISTANCE`
(case-insensitive, colon optional). `MID-DISTANCE` and `MID` both → **mid**.
When a block has these, it's a **parallel-group session**: the shared
warm-up (content before the first section) is whole-team → "all"; each
section's sets → that group.

### Sub-session boundaries
- `Week N` resets the week.
- A date header (`Wednesday, August 27, 2025 - ...`) starts a session.
- A `Day TIME` header (`MON AM`, `WED PM 16`, `Tuesday AM`) starts a session.
- A `Coach: NAME` line under an existing date starts a parallel sub-block.
- **Underscores (`____`) are NOT a boundary.** The doc uses them BOTH as
  separators before a Coach: line AND as internal section dividers inside
  Coach Josh's prescriptions. Splitting on them shredded one block into
  900/400/300 fragments. Let the Coach:/date/day headers do the splitting.

### Things that are NOT yardage
- Drylands days (skip — no swim).
- `Coach: SelectCoach` blocks (empty templates).
- Drill ratios like `2/2/2`, `2/2/1` (these are stroke counts, not sets;
  sub-25 "distances" should be ignored).
- `90 minutes in the water` (a duration, not 90 yards).
- Interval/time annotations `@ 1:30`, `:35`, `2:00`.

### Plain-distance lines need permissive matching
`100 kick` is easy, but the doc also writes `100 steady kick`,
`200 FAST kick`, `100 social kick`, `100 - no paddles - mostly underwater
kick`. A strict "number immediately followed by stroke word" misses ~88
real lines. Match a leading distance + any stroke/modifier word anywhere
on the line, excluding duration lines.

### Multipliers
- Standalone `2x` / `4x` on its own line multiplies the block that follows.
- `3 rounds of:` does the same.
- Inline `3x100`, `6x50` (also `1x400 + 1x200` — multiple per line).

### Abbreviated weekdays / late-season format drift
Early weeks: full date headers. Late weeks: `MON AM 16`, `TU AM`, `WED PM`.
Handle MON/TU/TUE/WED/THU/FRI/SAT/SUN with optional AM/PM and trailing week
number. Watch for a UTF-8 BOM at file start (read with `utf-8-sig`).

### De-duplication
The doc has copy-paste artifacts (same workout pasted twice). Dedupe on a
hash of **(normalized header + normalized body)** — NOT body alone. Body-
only hashing collapsed legitimately-similar workouts on different days and
silently dropped ~11k yd.

---

## 5. Architecture lessons (what worked, what didn't)

**What worked:**
- The coach→group attribution model with section-header overrides.
- The hybrid `total = max(manual NxDIST sum, checkpoint cumulative)` for
  *single-group* sessions — the checkpoint catches prose warm-ups the
  NxDIST matcher can't, and `max` means a missed checkpoint never lowers a
  good manual sum.
- Per-(week, day) rollups for the microcycle/mini-microcycle views.
- A validation pane that flags (a) sub-sessions < 2,000 yd and (b) any
  group reading 0 in a week (a near-certain attribution failure).

**What did NOT work / kept biting us:**
- **Reactive regex patching.** Each boss-reported wrong number led to a
  new regex. We never had a ground-truth fixture, so fixes were whack-a-
  mole and we never knew the true error rate. (We do now — see §2. USE IT.)
- **Checkpoints in parallel-group sessions.** There, the running total
  tracks ONE squad's cumulative, not the multi-squad sum. Using it
  mangles the split. For parallel-group sessions, trust the manual sum.
- **Treating the txt as if it had a schema.** It doesn't (see §6).

**Mistakes I (the previous agent) made — learn from these:**
1. Started coding before getting ground truth. Should have asked the boss
   for a hand-tally of 2–3 weeks on day one and built to match it.
2. Let the "what counts as a session total" ambiguity (§3) stay implicit
   for the entire project. It's THE central question and it was never
   nailed down.
3. Over-trusted regex for a fundamentally prose document. The variety of
   notations means regex coverage is always "good enough until the next
   edge case."
4. Repointed the git remote from the live-app repo to a new repo mid-
   stream, which split the deployment from the code. Keep one source of
   truth.

---

## 6. Is a .txt file the problem? (format recommendation)

**The format is a symptom; the root problem is that the source has no
structure.** It's free-form coaching prose with per-coach conventions.
Switching file formats only helps if it adds structure. Three real paths,
best to worst for accuracy:

1. **Structured data entry (best, if the coaches will do it).** A Google
   Sheet / Airtable with one row per (week, day, session, coach, group,
   set, reps, distance) — or even just (week, day, coach, group,
   total_yards). If yardage is entered as data, parsing becomes trivial
   and ~100% accurate. The blocker is workflow change for the coaches.
   Even a lighter version — coaches writing a single `TOTAL: 5400` line
   per block in a consistent place — would remove 90% of the pain.

2. **LLM extraction per day (best, if the source can't change).** Feed
   each day's raw text to an LLM with a strict output schema (per coach:
   group, yards, meters) AND the cumulative checkpoints as a cross-check.
   An LLM reads "100 steady kick" and "Sequence 2: Up Fly vs Low Fly"
   (parallel branches) far better than regex, and can explain its count.
   Cost is ~cents per day. We validated by hand that on hard cases the
   right answer needs *reading*, not pattern-matching. **This is likely
   the correct architecture given the source won't change.** Still assert
   the LLM's per-week totals against the §2 fixtures.

3. **Keep regex on txt (current; not recommended to extend).** You'll keep
   hitting new notations. If you must, at least add the §2 fixtures as
   automated tests so you know your error rate.

**Recommendation:** if the boss can get even semi-structured entry (#1),
do that. Otherwise build #2 (LLM-per-day with schema + checkpoint
validation + the §2 fixtures as regression tests). Do **not** invest more
in extending the regex parser.

---

## 7. Repos / where things are

- Source docs: `~/Desktop/Career/Swim Data/Weeks 1-10 ....txt` and
  `Weeks 11-17 ....txt` (read with `utf-8-sig` for the BOM).
- Old parser code: `swim_parser.py` (parsing) + `app.py` (Streamlit UI).
- GitHub: `johndevere-mpcp/Swim-Scraper` (current) and
  `johndevere-mpcp/Yardage-Breakdown-per-Group` (older; had the live
  Streamlit deploy). Pick one and delete/redirect the other.
- Latest parser season total (for reference, NOT ground truth): ~1.60M yd.

---

## 8. Copy-paste prompt for the next agent

> I'm building a tool that totals swim-practice yardage for a college team,
> broken down by training group (Sprint=Dave, Mid=Josh, Distance=Noah), per
> week and per day. The input is a free-form coaching document (currently
> .txt exported from a Google Doc; I'm open to changing the format).
>
> **Before writing any code, do these in order:**
>
> 1. Read `HANDOFF.md` in full. It documents the domain rules, the document's
>    conventions, and the specific mistakes the last attempt made.
>
> 2. The hardest part is NOT parsing — it's a definition I must settle first:
>    when multiple coaches write separate blocks for the same session slot
>    (different squads in different lanes), does the session total SUM all
>    blocks or count one? Ask me this explicitly and wait for my answer.
>    Restate the rule back to me in one sentence before proceeding.
>
> 3. Use the Week 1–3 hand-tally in `HANDOFF.md §2` as a hard test fixture.
>    Your per-week totals must reproduce: Week 1 = 35,800 yd + 3,800 m,
>    Week 2 = 32,750 yd, Week 3 = 73,575 yd (subject to the §3 ruling, which
>    may legitimately revise Weeks 2–3 upward — confirm with me). Assert
>    against these numbers in automated tests; do not validate by eyeballing.
>
> 4. Recommend an architecture. Given the source is unstructured prose with
>    per-coach notations, seriously evaluate (a) getting the data entered in
>    a structured sheet, or (b) LLM-per-day extraction with a strict schema
>    and the doc's cumulative checkpoints as a cross-check — over (c)
>    extending regex. Justify your choice against the §6 analysis.
>
> 5. Keep yards and meters separate (an LCM day is meters; "LCM" in a title
>    does not by itself mean meters — only `[1,500m]` brackets do).
>
> Do not start coding until steps 2 and 3 are settled with me. When you do,
> build the smallest thing that reproduces the §2 fixtures, then expand.
> Treat any training group reading 0 yards for a whole week as a bug to
> investigate, not a number to display.

---

*Generated as a handoff for a fresh start. The plumbing (UI, rollups,
exports, validation) from the old version is reusable; the parsing core
is what needs rethinking — ideally not as regex.*
