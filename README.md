# WhisperByMeena

A local Jyotish (Vedic astrology) consultation tool for personal study, built as a Node.js Express server with browser UI.

## Philosophy

This tool is for a learner, by intention. Rules are visible, not hidden. Classical sources are attributed. Predictions are probabilistic, not deterministic. The system supports self-knowledge, not authority.

## What it does (Phase 1)

- Parses JHora PDF exports into structured JSON
- Extracts: native data, panchanga, planetary positions, chara karakas, dashas
- **Analysis layer** (not raw diagrams — the Learner already has the source PDF for visual reference):
  - **Planetary Relationships (Graha Maitri)**: Naisargika (natural, fixed) + Tatkalika (temporal, chart-dependent) combined into the classical five-fold Panchadha Maitri (Adhi Mitra / Mitra / Sama / Shatru / Adhi Shatru)
  - **Functional Nature**: Benefic/malefic classification specific to this Lagna — Yogakaraka, Functional Benefic, Neutral, Functional Malefic, Mild Functional Malefic, with Maraka and **Badhakesh** (obstacle-house lord) flagging, each with reasoning and a confidence level
  - **Aspects (Graha Drishti)**: Which planets cast aspects on which houses/planets in D1, per BPHS Ch. 24
  - **Dignity (Avastha)**: Exalted / Moolatrikona / Own Sign / Friend's-Neutral's-Enemy's Sign / Debilitated, per BPHS Ch. 3
  - **Arudha Padas (A1-A12)**: Full Jaimini Bhava Arudha computation, including Arudha Lagna (AL) and Upapada Lagna (UL)
  - **Yogas**: A focused, classically well-defined subset — Parivartana (sign exchange), Gajakesari, Kendra-Trikona Raja Yoga, Dhana Yoga, Vipreet Raja Yoga (Harsha/Sarala/Vimala)
  - **D9 (Navamsa) Confirmation**: Vargottama detection, D9 dignity, and a per-planet D1-vs-D9 convergence read (confirmed strong / confirmed weak / mixed / lean-positive / lean-negative / no-signal) — flagged with a birth-time-reliability caveat, consistent with the Learner's own framework for treating D9 as a supporting layer
- **Planetary Strength** (kept as distinct, non-conflated dimensions): Shadbala, Vimsopaka, Ashtakavarga (Sarvashtakavarga computed from the 8 Bhinnashtakavarga rows, not parsed from JHora's own SAV row — see `strengths.js` for why), Bhava Bala, Avastha (age/alertness/mood) + Activity. **Known limitation**: Pinda values (Sodhya/Rasi/Graha) are NOT extracted — JHora's text export glues them into an unbroken integer string per planet with no reliable separator to split on; rather than guess, this is left unparsed.
- **Conditional Dashas** (11 systems beyond Vimshottari/Narayana): Ashtottari, Moola, Vimshottari Tribhagi, Yogini (planet-based), Kalachakra (+ Paksha/Paramayush/Deha/Jiva metadata), Sudasa, Lagna Kendradi, Drigdasa, Shoola, Niryana Shoola (rasi-based). All reuse the same generic, tested tokenizer/grouper logic built for Vimshottari/Narayana — see `conditional-dashas.js`.
- **Multi-Dasha Convergence**: every personal event now shows which period was active in ALL 12 dasha systems at once (not just Vimshottari) — directly serves the Learner's analytical standard that conclusions should be supported by repeated indications across independent factors, applied to timing verification.
- **Indicator Convergence (the reasoning layer)**: for any event date, lays out — per active dasha lord — which houses it rules, occupies, and aspects, plus its natural significations, then counts how many *independent* active lords point at each life-theme/house. Deliberately NOT a prediction engine: it surfaces convergent evidence and the Learner interprets. Crucially, it ranks by *dynamic* connections (rulership/aspect of the currently-active lords) and excludes static occupancy (planets sitting in a house look identical on every date and would otherwise create false signal). Rasi-dasha lords are read via their sign ruler (flagged as one step removed).
- **Atmakaraka & Karakamsa Lagna (Jaimini)**: identifies the AK (soul-significator), its nakshatra + lord, the Karakamsa Lagna (AK's navamsa sign, read as a special soul-Lagna) with its occupants, and the Ishta Devata *indicators* (12th-from-Karakamsa, its lord/occupants — shown as indicators, not a named deity, since traditions differ). AK periods are also surfaced as a classical rule in the event convergence. Carries explicit birth-time-sensitivity and AK-scheme caveats (see `jaimini.js`).
- **Classical Rule Library**: encodes classical statements (house-lord placements + specific well-known rules) so the tool can say not just "there's a signal here" but "classical text X associates this placement with Y." Every rule carries its **source** (BPHS/Phaladeepika/etc.) and a **confidence flag** — `direct` (widely-agreed), `interpretive` (paraphrase, verify wording), or `synthesized` (combined/modern practice). Conflicting positions are shown, never silently resolved. Rules are integrated into the event convergence view (explaining *what classical texts say about each active lord*) and shown chart-wide in a dedicated card. Explicitly a curated subset of testable hypotheses, not the whole of any text, and never a fixed prediction — see `rules/schema.js` for the full honesty contract.
- **Personal Events Timeline**: log life events and see the active Mahadasha/Antardasha/**Pratyantardasha** (computed via classical proportional subdivision — JHora's text export only goes to Antardasha) at that date, for verifying classical principles against lived experience
- Renders a browser UI with active-dasha highlighting
- **Predictive Engine (3-layer, per the original build-spec)**: this is the piece the project had drifted away from — forward-looking event-timing with a convergence-percentage, not just past-event verification. Covers **7 life-domains**: Marriage, Career, Foreign Travel, Wealth, Disease/Health Vulnerability, Addiction/Loss of Control, and Children. Three layers:
  - **Layer 1 — Promise** (`promise.js`): checks 6 independent classical indicators (house-lord placement, karaka strength, occupants, aspects, secondary-house connections, and a relevant Saham where one exists) to see whether the D1 chart shows a credible promise at all — reported as a checklist, not a verdict. **Domain-aware polarity**: 'growth' domains (marriage, career, foreign, wealth, children) read benefics/strength as "supports"; 'affliction' domains (disease, addiction) correctly INVERT this — a strong, well-placed 6th/12th lord is classically protective (reduces vulnerability), while malefic occupancy/aspects increase it. Getting this backwards would have been a serious, misleading bug, so it's tested explicitly.
  - **Layer 2 — Dasha Activation** (`dasha-activation.js`): walks forward through future Vimshottari Antardasha windows (future-only, chronological) and finds which ones connect to each event via its MD/AD lords' rulership or karakatva. **Spec correction**: the original build-spec is explicit that timing runs on "Vimsottari + Ashtottari dasha" — an earlier version of this engine used only Vimshottari as primary with Narayana as the sole cross-check, a real deviation caught by the Learner. Ashtottari is now a genuine **co-primary** system: at every window, Ashtottari's own active MD/AD lords are independently checked for primary-house rulership, weighted higher (+15) than the tertiary Narayana check (+8) in Layer 3's scoring. **Two leniency bugs found and fixed during this correction**: (1) the primary Vimshottari relevance check itself was matching 36/36 future windows for several event categories (using secondary houses + occupancy/aspect made nearly anything "relevant," giving zero discriminating power) — fixed by restricting Layer 2 to the *primary house only* (secondary houses stay appropriate for the broader Promise/Layer 1) and requiring 2+ independent reasons, not 1; (2) a crash when the walk reached the final, open-ended Antardasha window (no `endDate`) computing an `Infinity` midpoint — now guarded.
  - **Layer 3 — Convergence** (`predictive-engine.js`): combines both into a score **always clamped to 5-95%** (never certain, by construction — Jyotish describes tendencies, leaving room for free will) with a **fully transparent, itemized breakdown** behind every number (no hidden multiplier) — exactly the spec's "narrative > raw score" rule. **Domain-aware labeling**: affliction domains get caution-framed labels ("Elevated indication of vulnerability — worth attention") and an inverted color scale (high score = red/caution, not green/celebratory) — the same wording used for a growth domain would be misleading and potentially alarming if applied unchanged to a health/addiction reading. An explicit note clarifies this is a classical-tendency reading, not medical or psychological advice, with a prompt to consult a qualified professional for live concerns.
- **Sahams** (`sahams.js`): BPHS Tajaka sensitive-point calculations (Vidya/education, Paradesa/foreign, Artha & Labha/wealth, Roga & Jadya/disease, Vyapara & Karma/career) — day/night birth is detected from the Sun's house (7-12 = day, 1-6 = night) to apply the correct classical formula variant, verified by hand-calculation against the chart data. Feeds into the relevant Promise-layer domains as an additional independent indicator.
- **Path Classification** (Education, Career, Foreign — in `promise.js`): goes beyond "is this promised" to suggest WHICH broad path the chart leans toward.
  - **Education**: Academic/Higher, Skill-Based/Vocational (trades like farming, carpentry, plumbing), Commerce/Trade, or Disruption/Break tendency — built on the documented 4th=basic/9th=bachelors/2nd=masters house-from-house chain.
  - **Career**: Government/Authority, Business/Entrepreneurship, Structured Service/Employment, Technical/Engineering, Creative/Arts, Healing/Counseling/Caregiving, Defense/Discipline-Based — built on documented case patterns (e.g. Mars+6th-lord in surgeon charts, Moon-in-10th in pediatrician charts, 9th-lord-in-Lagna for independent/business orientation).
  - **Foreign**: Short-Term Travel, Long-Term Settlement/Relocation, Foreign-Linked Career/Education, Badhakesh-Linked Connection — built directly on the documented principle "9th and 12th houses are the important houses to see foreign travel; even Badhakesha can show foreign travel."
  - All leanings carry explicit `confidence` flags (`interpretive` or `synthesized`, never silently presented as `direct`) since these are modern/case-pattern synthesis, not single named classical yogas — several leanings can apply to one chart at once, and that's reported plainly rather than forced into one answer.
- **Character Portrait** (`narrative-portrait.js`): a flowing, narrative character reading implementing the "Jyotish — Evidence se Insaan Tak" methodology as actual logic (pure local JavaScript, no external API call — by design, to keep the tool fully local). Three rules drive it: (1) **convergence-weighted confidence** — a claim is phrased plainly only when 2+ independent indicators support it; a single isolated indicator is phrased tentatively ("ek jhukav dikhai deta hai..."), reusing the same convergence-counting principle already used elsewhere in this tool (`convergence.js`, `promise.js`); (2) **genuine-tension detection only** — internal paradoxes (e.g. one planet holding both Yogakaraka and Badhakesh) are surfaced only when the chart actually shows them, with each role-*pair* getting its own accurate description (Yogakaraka+Badhakesh is a genuine push-pull; Badhakesh+Maraka are both cautionary roles with no "good pull" — conflating these into one generic template was a real bug, caught via cross-chart testing, and fixed); (3) **static vs. dynamic** — a stellium (3+ planets sharing a house) is treated as a valid *character* signal (since character is static) but is never claimed as event-specific timing, which stays the job of `convergence.js`. This module will not match an LLM's fluency, but implements the spec's actual methodology rather than being a flat fact-list in sentence clothing.
- **Gochar (Transit) — Manual Input** (`transit.js`): the one analytical layer this tool deliberately does NOT compute itself. Transit needs to know where planets are on an arbitrary date — that's a real astronomical (ephemeris) calculation, fundamentally different from every other module here, which is pure classification on the fixed birth-moment data JHora already provides. No JS ephemeris library has been validated here against a known-accurate reference, and a silently-wrong position (especially near a sign boundary) would produce a wrong house and a wrong classical result — unacceptable given this tool's data-integrity standard. So: the Learner manually supplies transit positions (from JHora or any trusted source) via a form in the UI, and this module does the classification — counting each transiting planet's house **from the natal Moon** (Chandra Lagna, the primary classical transit reference) and matching it against the classical Good/Bad result tables (Tables 53-59 in the reference textbook; Rahu follows Saturn's table and Ketu follows Mars's, per the source's own explicit equivalence). Verified against the reference text's own worked example (Moon in Gemini, June 1999 transit chart) before being trusted. An automatic, ephemeris-computed version may be added later as an *additional* option, not a replacement — manual input stays the zero-computation-risk baseline.
- **Character Analysis** (`character.js` + `lagna-archetypes.js`): reads character as a CONVERGENCE of independent layers rather than any one alone — (1) **Lagna Archetype**: the structural template for the ascendant sign itself (sourced from a 12-sign reference resource using the Kal Purush/natural-zodiac comparison technique — what's true for *everyone* with this Lagna before looking at the individual chart), (2) **Lagna lord's actual placement** (individual refinement of the archetype), (3) **Moon** — sign, nakshatra, house (the deepest classical character layer; Janma Nakshatra is taken from the Moon, not the Lagna, because the Moon represents the inner mind), (4) **Sun** — sign, nakshatra, house (soul/willpower), (5) **Dasha timing** — which layer (Lagna-lord-driven, Moon-driven, Sun-driven, or other) is most active right now. For Moon's/Sun's specific nakshatra, this module surfaces reliable structural facts (sign, nakshatra, pada, nakshatra lord) rather than inlining a full psychological deep-dive from memory — pointing to the Learner's own nakshatra reference material for that level of detail, rather than risking a drifted paraphrase of a large source.
- **Report Export**: one-click download of a single, self-contained HTML report consolidating every layer (positions, functional nature, dignity, D9 confirmation, Jaimini, yogas, arudhas, classical rules, and all logged events with their multi-dasha context + convergence). Print-friendly CSS means the browser's "Save as PDF" produces a clean document — no heavy server-side PDF engine needed. Every source citation, confidence flag, and caveat from the live UI is preserved in the report (not stripped for brevity).
- **Excel Workbook Export**: one-click download of a real multi-sheet `.xlsx` (opens directly in Excel *and* Google Sheets — no import step) built for *analysis* rather than reading. Sheets: Summary; Planets (one row per planet, every attribute as a filterable/sortable column); Yogas; Arudhas; Classical Rules (with confidence/polarity/source/caveat columns — filter to just `direct` rules, etc.); Events (wide — each dasha system a column); **Events × Dashas (long)** and **Event Convergence (long)** — pivot-ready fact tables for asking questions like "which dasha lords recur across my 'career' events?" This is the format that plugs into the Learner's broader research-database approach.

## Personal Events & Pratyantar Dasha

JHora's PDF export only lists Mahadasha and Antardasha explicitly. This tool computes **Pratyantardasha** on the fly using the standard classical proportional-subdivision algorithm (each period divides into 9 sub-periods in fixed Vimshottari order, starting with the period's own lord, each share proportional to that planet's total Vimshottari years out of 120).

Events are tied to whichever chart is currently loaded (keyed by native name) and persisted in `data/events.json`. Adding an event automatically computes and stores which MD/AD/PD was active on that date.

## Setup

### Prerequisites

- Node.js v18 or higher ([download](https://nodejs.org))
- npm (comes with Node.js)

### Installation

```bash
# In the project root
npm install
```

> **Updating from an earlier version?** Re-run `npm install` — this version adds the `exceljs` dependency (used for the Excel workbook export). Without it, the `.xlsx` download will fail while everything else keeps working.

### Run the server

```bash
npm start
```

Then open [http://localhost:3000](http://localhost:3000) in your browser.

### Upload a JHora PDF

1. Open JHora on your machine
2. Generate a chart for the native you want to analyze
3. Export as PDF
4. In the browser UI, click the file input and select the PDF
5. Click "Parse Chart"

## Project Structure

```
whisperbymeena/
├── server.js                  Express server (parse, events, dasha-context endpoints)
├── package.json
├── src/
│   ├── parser/
│   │   ├── jhora-parser.js    Main orchestrator
│   │   ├── extractors/        Per-section parsers
│   │   │   ├── header.js
│   │   │   ├── bodies.js
│   │   │   ├── vimshottari.js  (incl. Pratyantar computation)
│   │   │   ├── narayana.js
│   │   │   ├── conditional-dashas.js  Ashtottari, Kalachakra, Moola, Sudasa, etc. (reuses generic logic)
│   │   │   └── strengths.js    Ashtakavarga, Shadbala, Bhava Bala, Avastha, Vimsopaka
│   │   └── utils/
│   │       ├── pdf-text-extract.js
│   │       ├── section-router.js
│   │       └── constants.js
│   └── analysis/               Analysis layer
│       ├── index.js            Orchestrator
│       ├── relationships.js    Graha Maitri (friend/enemy)
│       ├── functional-nature.js Benefic/malefic + Badhakesh for this Lagna
│       ├── aspects.js          Graha Drishti
│       ├── dignity.js          Exaltation/debilitation/own/friend-enemy sign
│       ├── arudhas.js          Bhava Arudhas (A1-A12)
│       ├── yogas.js            Parivartana, Gajakesari, Raja/Dhana/Vipreet yogas
│       ├── navamsa.js          D9 dignity, Vargottama, D1-D9 confirmation
│       ├── jaimini.js          Atmakaraka, Karakamsa Lagna, Ishta Devata indicators
│       ├── convergence.js      Indicator convergence (reasoning layer for events)
│       ├── promise.js          Predictive Layer 1 — D1 promise checklist per event category (7 domains)
│       ├── dasha-activation.js Predictive Layer 2 — future dasha-window relevance
│       ├── predictive-engine.js Predictive Layer 3 — convergence% (clamped 5-95%, full breakdown)
│       ├── sahams.js           BPHS Tajaka Sahams (sensitive points), day/night-aware
│       ├── lagna-archetypes.js Structural character template per ascendant (12 signs)
│       ├── character.js        Character convergence: Lagna archetype + lord + Moon + Sun + dasha timing
│       ├── narrative-portrait.js  Flowing narrative portrait — convergence-weighted confidence, genuine-tension detection, pure local generation
│       └── transit.js          Gochar (transit) — manual position input, classical Moon-based house counting + result tables
│       └── rules/              Classical Rule Library
│           ├── schema.js       Sourcing standard + honesty contract
│           ├── house-lord-rules.js   Principle-based bhavesh-in-bhava rules
│           ├── specific-rules.js     Specific well-known classical rules
│           └── rule-engine.js        Matcher: library → this chart
├── public/                    Browser UI
│   ├── index.html
│   ├── style.css
│   └── app.js
├── data/
│   ├── analyzed_charts/       Parsed JSONs are saved here (keyed by native name)
│   ├── events.json            Personal events log, with dasha context attached
│   └── test_input/            Sample test data
└── tests/
    ├── test_parser.js         Parser tests (Jatin sample)
    └── test_meena.js          Parser + analysis tests (Meena sample, real PDF format)
```

## Testing

A sample JHora text extract is bundled at `data/test_input/jatin_sample.txt`. Run the parser test:

```bash
node tests/test_parser.js
```

You should see successful extraction of all sections with assertions passing.

## What's NOT in Phase 1

The following are deliberately deferred to subsequent phases:

### Phase 2 — Vargas and Strengths
- D10 (Dashamsa) and D12 (Dwadashamsa) house assignments — require varga computation or diagram parsing
- Arudha Lagna and A2-A12 arudhas — computable from sign + house arithmetic
- Special lagnas: Sree Lagna, Pranapada, Bhrigu Bindu, Upapada
- Shadbala, Bhava Bala, Ashtakavarga, Vimsopaka, Vaiseshikamsas extraction
- Planet avastha (age, alertness, mood, activity)

### Phase 3 — Conditional Dashas
- Ashtottari, Kalachakra, Moola
- Sudasa, Lagna Kendradi, Drigdasa
- Shoola, Niryana Shoola, Yogini
- Vimshottari Tribhagi variation

### Phase 4 — Rule Library and Analysis
- Curated classical rule library (BPHS → Phaladeepika → Jaimini → PVR hierarchy)
- Multi-indicator convergence detection
- Personal events timeline mapping against dashas
- Character analysis synthesis (Parashari + Jaimini)
- Optional Claude API integration for narrative synthesis

## Design Principles

1. **Transparency over filtration** — every rule must show its classical source
2. **Multi-indicator convergence** — conclusions emerge from repeated signals, not single rules
3. **Probabilistic framing** — themes and tendencies, never deterministic predictions
4. **Backend computes, frontend explains** — numerical work hidden in code, language in UI
5. **Local-first** — no data sent anywhere, no cloud, no telemetry

## Sources Referenced

- Brihat Parashara Hora Shastra (BPHS) — foundational framework
- Phaladeepika (Mantreswara) — practical interpretation
- Jaimini principles — Chara Karakas, Arudhas, Rasi Dashas
- PVR Narasimha Rao — modern methodological framework

## License

MIT — for personal use and study
