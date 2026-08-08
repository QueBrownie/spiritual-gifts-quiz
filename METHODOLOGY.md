# Spiritual Gifts Inventory — Statistical Methodology

Scope: this document describes the scoring logic implemented in `index.html`
(single-file app, no backend). It is written so a reviewer can check every
claim against the code and reproduce every number by hand. All line numbers
refer to `index.html` as of the version reviewed; re-check them if the file
has changed.

Status: **not a validated psychometric instrument.** This document describes
what the code computes and why, not a claim that the instrument has been
normed, factor-analyzed, or validated against external criteria. See
[§7 Limitations](#7-known-limitations--things-a-reviewer-should-push-on).

---

## 1. Instrument design

- **16 constructs** ("gifts"), defined in the `GIFTS` array (`index.html:397-526`).
- **4 items per gift** → 64 items total (`ITEMS_PER_GIFT = 4`, `index.html:528`).
- Per gift: **3 positively-keyed items + 1 reverse-keyed item** (flagged `r: true`
  in the item object, e.g. `index.html:404`).
- **Response scale:** 5-point, integer-coded 0–4 (`MAX = 4`, `index.html:391`),
  labeled Never / Rarely / Sometimes / Often / Almost always
  (`SCALE_LABELS`, `index.html:392`).
- Items are behaviorally anchored ("I have done X"), not trait self-labels
  ("I am an X kind of person") — a deliberate design choice noted in the
  source comment at `index.html:394-396`.

## 2. Item presentation

`buildQuestions()` (`index.html:613-639`):

1. Fisher–Yates shuffle of all 64 items.
2. A repair pass (up to 4 iterations) swaps items so that **no two
   consecutive items belong to the same gift**, reducing local response-set
   effects (e.g. anchoring one gift's answer to the previous one).
3. Gift names are never shown during the quiz — only the item text.

This affects *presentation order only*; it has no effect on scoring, which is
grouped by gift regardless of display order.

## 3. Scoring pipeline

All scoring happens in `computeResults()` (`index.html:738-777`), called once
all 64 items are answered (`finish()`, `index.html:779`).

### 3.1 Reverse scoring

```js
const scored = questions.map((q, i) => q.rev ? (MAX - answers[i]) : answers[i]);
```
`index.html:739`

For a reverse-keyed item, raw answer `a` becomes `4 - a`. This is the only
per-item transformation applied before any aggregation.

### 3.2 Central tendency and dispersion

```js
function mean(a) { return a.reduce((s, x) => s + x, 0) / a.length; }
function sd(a)   { const m = mean(a); return Math.sqrt(mean(a.map(x => (x - m) * (x - m)))); }
```
`index.html:735-736`

Note: `sd()` is the **population standard deviation** (divides by `n`, not
`n-1`). This is a deliberate simplification, not sample-based inference — see
[§7.2](#72-population-vs-sample-sd).

### 3.3 Person-level statistics

```js
const personMean = mean(scored);   // mean of all 64 reverse-corrected answers
const personSD   = sd(scored);     // population SD of all 64 reverse-corrected answers
```
`index.html:747-748`

### 3.4 Per-gift statistics

For each gift `g`, over its 4 (reverse-corrected) item scores:

```js
raw  = sum of the gift's 4 scored items      // range 0–16
mean = raw / 4
n    = 4
```
`index.html:750-753`

### 3.5 Ipsative standardization ("z-score", self-referenced)

```js
const usableSpread = personSD > 0.25;
r.z = usableSpread ? (r.mean - personMean) / personSD : 0;
```
`index.html:756-757`

Each gift's score is expressed as **distance from that respondent's own mean,
in units of that respondent's own SD** — not compared to any population norm
or other respondent. If `personSD ≤ 0.25` (near-zero variance — the
respondent rated almost everything identically), standardization is skipped
and every gift is scored 0 rather than dividing by ~0.

### 3.6 Ranking and tie-breaking

```js
// shuffle before sorting so equal means don't inherit array order
for (let i = byGift.length - 1; i > 0; i--) { ... Fisher-Yates ... }
byGift.sort((a, b) => b.mean - a.mean);
```
`index.html:759-764`

Gifts are pre-shuffled, then sorted descending by mean. `Array.prototype.sort`
is stable per the ES2019+ spec, so gifts with exactly equal means keep the
random pre-shuffle order rather than always favoring, say, index 0
("Leadership") on ties. **This is a testable claim** — see
[§8](#8-test-plan--reviewer-checklist).

### 3.7 Standard error of the difference, and the "top band"

```js
const seDiff = personSD * Math.sqrt(2 / ITEMS_PER_GIFT);   // = personSD * sqrt(0.5)
const topMean = byGift[0].mean;
r.top = (topMean - r.mean) <= seDiff + 1e-9;
const topBand = byGift.filter(r => r.top);
```
`index.html:766-771`

`seDiff` is the standard error of the difference between two 4-item gift
means, under the assumption that each gift's items share the same variance as
the respondent's overall item variance (`personSD`) — i.e. it is **not** a
per-gift SD, but a global plug-in estimate. Formula:
`SE_diff = σ · √(2/n)` with `n = 4` items per gift, `σ = personSD`.

Any gift whose mean is within one `seDiff` of the top gift's mean is placed
in the "top band" and reported as tied with the leader, rather than ranked
strictly above/below it. The `+1e-9` epsilon exists so the top gift is always
included in its own band even when `seDiff = 0` (i.e., every scored answer is
identical).

### 3.8 "Flat profile" detection

```js
const spread = topMean - byGift[byGift.length - 1].mean;
const flat = !usableSpread || topBand.length > 8 || spread < 0.5;
```
`index.html:773-774`

A profile is flagged "flat" (no gift is reported as standing out) if **any**
of:
- `usableSpread` is false (`personSD ≤ 0.25`), or
- more than half the 16 gifts (>8) land in the top band, or
- the raw spread between the highest and lowest gift mean is < 0.5 (on the
  0–4 scale).

These three thresholds (`0.25`, `8`, `0.5`) are fixed constants with no
derivation shown in code or comments — see [§7.4](#74-uncalibrated-thresholds).

## 4. Response-quality diagnostics (separate from the gift scores)

`index.html:741-745`, consumed in `finish()` at `index.html:786-796`.

These operate on **raw, pre-reverse-scoring answers**, deliberately —
per the code comment at `index.html:741`: *"response-style diagnostics use
RAW answers (straight-lining is a raw pattern)."* This is an intentional
separation from the ipsative scoring in §3, which uses reverse-corrected
scores.

| Diagnostic | Formula | Trigger | Message shown |
|---|---|---|---|
| `rawSD` | population SD of all 64 **raw** answers | `rawSD < 0.35` | "gave nearly the same answer to almost every statement" |
| `modalShare` | (count of most frequent raw value) / 64 | `modalShare > 0.85` | same as above |
| `flat` (§3.8) | see above | true, and straight-line not already triggered | "answers came out unusually even across all 16 gifts" |
| `personMean` (reverse-corrected) | mean of scored answers | `≥ 3.2` | "rated most statements high... compared to your own average" |
| `personMean` (reverse-corrected) | mean of scored answers | `≤ 0.9` | "rated most statements low..." |

Straight-lining and flatness notices are mutually exclusive (`else if` at
`index.html:789`); the two `personMean` baseline notices can co-occur with
either.

## 5. What gets displayed

- A diverging bar chart of each gift's `z` (`finish()`, `index.html:799-825`),
  axis scaled to `max(1.0, ceil(max|z| * 2)/2)`.
- A table of raw score (`x`/16), relative score (`z`, signed, 2dp in the
  table / 1dp in the chart), and a "Strongest" band flag
  (`index.html:827-837`).
- Cards for each gift in the top band, only if `!flat`
  (`index.html:842-861`).
- A downloadable/shareable PNG snapshot rendered by hand on `<canvas>`
  (`buildSnapshotCanvas()`, `index.html:935-1060`) reproducing the same chart
  and numbers — this is a rendering path, not an independent computation; it
  consumes the same `R` object from `computeResults()`.

## 6. Formal summary of the model

For respondent with raw answers $a_1, \dots, a_{64} \in \{0,\dots,4\}$ and
reverse flags $r_1,\dots,r_{64}\in\{0,1\}$:

- $s_i = r_i \cdot (4 - a_i) + (1-r_i)\cdot a_i$ (reverse-corrected score)
- $\bar{s} = \frac{1}{64}\sum s_i$ (`personMean`)
- $\sigma_s = \sqrt{\frac{1}{64}\sum (s_i - \bar s)^2}$ (`personSD`, population SD)
- For gift $g$ with item set $I_g$ ($|I_g|=4$): $\bar{s}_g = \frac{1}{4}\sum_{i \in I_g} s_i$
- $z_g = \dfrac{\bar s_g - \bar s}{\sigma_s}$ if $\sigma_s > 0.25$, else $z_g = 0$
- $\mathrm{SE}_{\text{diff}} = \sigma_s \sqrt{2/4}$
- Top band $= \{g : \max_h \bar s_h - \bar s_g \le \mathrm{SE}_{\text{diff}} \}$

This is an **ipsative, within-person standardization** — comparable in spirit
to a centered/scaled Q-sort or an ipsatized personality profile, not to a
norm-referenced test with population percentiles.

## 7. Known limitations (things a reviewer should push on)

### 7.1 Ipsative dependency between gifts
Because every gift's $z_g$ is computed relative to the *same* $\bar s$ and
$\sigma_s$ derived from *all 64 items including that gift's own 4*, the 16
$z_g$ values are **not independent measurements**. Raising one gift's answers
mechanically pulls $\bar s$ up and depresses the relative standing of every
other gift, even if the respondent's absolute behavior in those other areas
is unchanged. This is a structural property of ipsative scoring, not a bug —
but it means gift scores cannot be meaningfully compared *across
respondents* (e.g. "my Teaching z is higher than yours") — only *within* one
respondent's own profile. The in-app disclosure (`index.html:358`) states the
self-referenced design but does not name this dependency explicitly.

### 7.2 Population vs. sample SD
`sd()` divides by $n$, not $n-1$. For $n=64$ (person-level) the Bessel
correction is negligible ($\sqrt{64/63} \approx 1.008$). It is more relevant
at the conceptual level — `personSD` blends variance from 16 different
constructs into one pooled figure used for every gift's standardization and
for `seDiff` (§7.3).

### 7.3 `seDiff` assumes homogeneous variance across all gifts
`seDiff` uses the *global* `personSD` (variance pooled across all 16
constructs) as a stand-in for each gift's own item-level variance. If a
respondent is highly consistent on some gifts and erratic on others, a
single global SE will over- or under-state precision for individual gifts.
There is no per-gift SD or reliability estimate (e.g. Cronbach's alpha,
split-half) computed anywhere in the code.

### 7.4 Uncalibrated thresholds
The constants `0.25` (usable-spread cutoff), `8` (max top-band size before
"flat"), `0.5` (min spread before "flat"), `0.35` (rawSD straight-line
cutoff), `0.85` (modal-share cutoff), and `3.2`/`0.9` (baseline notices) are
all hard-coded with no stated derivation, simulation, or citation. They are
plausible engineering heuristics, not calibrated statistical thresholds.
Worth empirically testing (e.g. simulate random response vectors and check
false-positive/negative rates for each flag) before treating the flags as
reliable.

### 7.5 Small item count per subscale
4 items per gift (3 keyed + 1 reverse) is small for estimating a stable mean
or variance. The app's own "Sources" disclosure (`index.html:361`) already
states this ("enough to be suggestive, not conclusive").

### 7.6 No test-retest or convergent validity data
There is no mechanism in the app to assess reliability over time or against
an external criterion (e.g. peer ratings, behavioral observation). The
in-app copy (`index.html:347-350`) explicitly recommends peer confirmation
for this reason, which is a reasonable mitigation but not a substitute for
psychometric validation.

## 8. Test plan / reviewer checklist

These are exact, hand-computable inputs a reviewer can enter into the live
quiz (statement text is quoted so the correct items are identifiable
regardless of shuffle order and hidden gift labels) to verify the formulas
above.

### 8.1 Degenerate case — literal straight-lining
**Input:** answer `2` ("Sometimes") to all 64 statements.
**Predicted output:**
- `scored[i] = 2` for every item (reverse items: `4 - 2 = 2`).
- `personMean = 2`, `personSD = 0` → `usableSpread = false` → every `z = 0`.
- `rawSD = 0`, `modalShare = 1.0` → triggers the straight-lining notice
  (`rawSD < 0.35` and `modalShare > 0.85`, both true).
- `flat = true` (via `!usableSpread`).
- Every gift ties; `topBand` = all 16 gifts (also `> 8`, a second independent
  reason `flat` is true).
- No "strongest gift" cards should render (`index.html:842`, gated on `!R.flat`).

### 8.2 Degenerate case — reverse-scoring isolates from raw diagnostics
**Input:** answer `4` ("Almost always") to the three positively-keyed
statements of *every* gift, and `0` ("Never") to the one reverse-keyed
statement of *every* gift.
**Predicted output:**
- Raw answers are **not** constant (mix of 4s and 0s: 48 items at 4, 16 at 0),
  so:
  - `modalShare = 48/64 = 0.75` (< 0.85 → does **not** trigger straight-lining by this route)
  - `rawSD = sqrt(0.75·1² + 0.25·3²) = sqrt(3) ≈ 1.732` (> 0.35 → does **not** trigger straight-lining by this route either)
- But after reverse-correction, `scored[i] = 4` for *every* item (the reverse
  item flips `0 → 4-0 = 4`), so `personSD = 0` again → `usableSpread = false`
  → `flat = true`, this time surfacing the **"answers came out unusually
  even"** message (the `else if` branch, `index.html:789`), not the
  straight-lining message.
- This is the key test that the two diagnostics genuinely read different
  data (raw vs. reverse-corrected), as the code comment at `index.html:741`
  claims.

### 8.3 Single clear winner — verify `z`, `seDiff`, and band membership
**Input:**
- Answer `2` ("Sometimes") to all statements **except** the four Evangelism
  items.
- For the three positively-keyed Evangelism statements — "I bring up my
  faith in conversation with people who don't share it," "I have asked
  someone directly where they stand with God," "I build friendships with
  people outside the church with their faith in mind" — answer `4` ("Almost
  always").
- For the reverse-keyed Evangelism statement — "I steer away from spiritual
  topics with people who aren't already believers" — answer `0` ("Never").

**Predicted output (exact):**
- Scored values: 60 items at `2`, 4 items (Evangelism) at `4`.
- `personMean = 136/64 = 2.125`
- `personSD = √(15/64) = √15/8 ≈ 0.48412`
- `usableSpread = true` (0.484 > 0.25)
- Evangelism: `mean = 4`, `z = (4 − 2.125)/0.48412 = √15 ≈ +3.873`
- Every other gift: `mean = 2`, `z = (2 − 2.125)/0.48412 = −1/√15 ≈ −0.258`
- `seDiff = personSD · √0.5 = (√15/8)·(1/√2) = √7.5/8 ≈ 0.342`
- Band test for a non-Evangelism gift: `topMean − mean = 4 − 2 = 2`, which is
  **not** `≤ 0.342` → excluded from the band.
- `topBand = {Evangelism}` only (length 1) → `flat = false`
  (`usableSpread` true, band length 1 ≤ 8, `spread = 2 ≥ 0.5`).
- Results screen should show exactly one "Your strongest gift" card:
  Evangelism, raw `16/16`, `+3.9` vs. average. All other 15 gifts show raw
  `8/16`, approximately `−0.3`.

### 8.4 Tie-order randomness (stability of `sort`)
Construct an input where two or more gifts land on *exactly* equal means
(e.g. extend §8.3's method to give two gifts identical `4,4,4,4` scored
patterns). Run the quiz multiple times with the same answer pattern (item
order is re-shuffled each run by `buildQuestions()`, but that shouldn't
matter to gift-level scoring). Confirm:
- Both tied gifts appear together in the top band (per §3.7's `seDiff` logic,
  since their means are within `0 ≤ seDiff`).
- Their relative order within the band varies from run to run (verifying the
  pre-sort Fisher–Yates at `index.html:760-763` actually randomizes
  tie-order, rather than the sort silently defaulting to array/declaration
  order).

### 8.5 Boundary conditions worth unit-testing directly against the code
(Not requiring the UI — these can be checked by calling `computeResults()`
in a console with a crafted `answers`/`questions` state, or by code
inspection.)
- `personSD` exactly `0.25` → `usableSpread` should be `false` (strict `>`,
  `index.html:756`), i.e. the boundary itself is treated as unusable.
- `topBand.length` exactly `8` vs. `9` → confirms the `> 8` cutoff
  (`index.html:774`) is the documented "more than half" behavior, not "at
  least half."
- All answers missing except one — `next()` (`index.html:722-731`) should
  route back to the first unanswered question (`findIndex(a => a === null)`,
  `index.html:725`) rather than allow `finish()` to run on an incomplete
  `answers` array; confirm `computeResults()` is never reachable with a
  `null` in `answers`.
- `localStorage` unavailable (private browsing) — `save()`/`loadSaved()`
  (`index.html:648-671`) should degrade to in-memory-only state without
  throwing, per the `try/catch` wrapping.

## 9. Traceability table

| Concept | Formula location | Consumption / display location |
|---|---|---|
| Reverse scoring | `index.html:739` | — |
| Raw response diagnostics (`rawSD`, `modalShare`) | `index.html:742-745` | `index.html:787-788` |
| `personMean`, `personSD` | `index.html:747-748` | `index.html:756-757, 766-767, 792-796` |
| Per-gift raw/mean | `index.html:750-753` | table/chart, `index.html:808-837` |
| `z` (ipsative standardization) | `index.html:756-757` | chart, `index.html:800-825`; table, `index.html:834` |
| Tie-safe ranking | `index.html:759-764` | display order throughout results screen |
| `seDiff` / top band | `index.html:766-771` | `index.html:842-861` |
| `flat` | `index.html:773-774` | `index.html:789-791, 842` |
| Quality notices | `index.html:786-797` | `#qualityNotice`, `index.html:303, 797` |

## 10. Sources for item content (not for the statistical method)

Per the in-app "Sources" panel (`index.html:369-374`): gift *definitions* are
adapted from Gene Wilkes, *Jesus on Leadership* (LifeWay, 1998), drawing on
Ken Hemphill (1995) and C. Peter Wagner (Regal, 1979); "sign gifts" are
excluded following that source. The 64 rating *statements* are original to
this app, not drawn from any published, validated inventory — so no external
reliability/validity data exists for the items as written, even though the
construct definitions themselves are drawn from established devotional
literature.
