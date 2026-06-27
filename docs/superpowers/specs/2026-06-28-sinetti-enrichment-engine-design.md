# Sinetti Enrichment Engine — Design Spec (v1)

> Design spec. Generated 2026-06-28. Status: **approved in brainstorming, pending written review.**
> Builds on `research/01-enrichment-data-sources.md`, `research/02-registries-vs-vies-vs-aggregators.md`,
> and `research/03-sinetti-field-to-api-map.md` in this repo.

---

## 0. Plain-language glossary

- **Enrichment** — adding extra, trustworthy facts about a company we already know about,
  by looking it up in outside databases.
- **Counterparty** — the other company in a deal (the one you're checking before you trade).
- **Due diligence** — the background check you run on a counterparty before trusting them.
- **VAT number** — a business's sales-tax registration number. Italy calls it *partita IVA*.
- **VIES** — the EU's free service that says whether a VAT number is valid and (in most
  countries) returns the registered company name and address.
- **LEI** — "Legal Entity Identifier", a free global 20-character company code (ISO 17442),
  run by **GLEIF**, the non-profit that publishes the LEI database and a free web API.
- **Fuzzy match** — matching by approximate text (a name) rather than an exact code; less
  certain, so it needs a confidence score and the option to *abstain* (return "no match").
- **Provider** — one outside source we consult (VIES is a provider; GLEIF is a provider).
- **Cache** — a local copy of an answer we already fetched, so we don't ask the source again.
- **TTL ("time to live")** — how long a cached answer stays usable before we refresh it.
- **Provenance** — for each fact, *where it came from* and *when* (so the user can trust it).
- **SiS (Streamlit-in-Snowflake)** — the Snowflake-hosted copy of Sinetti. By default it
  **cannot reach the internet**, so live external lookups don't run there.

> Numbers in worked examples below are flagged **[illustrative]** unless they were queried
> from the live Sinetti data, in which case they are flagged **[real]**.

---

## 1. Goal & the two non-negotiable facts it rests on

**Goal:** a self-contained engine that takes one certified company from Sinetti and returns
a **Counterparty Profile** — a verified identity plus a transparent fraud/trust verdict —
by consulting *free* sources on demand and caching the answers. The engine is built clean
enough to later wrap as a public API; v1 surfaces it as a panel in Sinetti's **Verify** section.

**The two facts from research that shape everything:**

1. **Certification data alone is a weak trust signal** (report 01). That is *why* layering
   identity + fraud verification on top of the certificate adds real value.
2. **Sinetti has no single primary identifier** (report 03). Only the Italian **INS** subset
   carries a real tax ID (~22,267 of 22,582 INS rows have a clean 11-digit Italian VAT)
   **[real]**; **ISCC + REDcert give only name + country (+ address)** and store **no LEI
   anywhere**. So enrichment is *exact* for the INS subset and *fuzzy* for the rest.

## 2. The four decisions locked in brainstorming

| Decision | Choice |
|---|---|
| What we build | **Consume free sources now, designed to publish as an API later** |
| Where it lives | **Standalone pure-Python engine + a thin Sinetti UI panel** |
| When lookups run | **On-demand when a company is opened, then cached** |
| v1 primary use case | **Due diligence / counterparty trust** |
| v1 sources | **VIES + GLEIF + internal ISCC fraud flags** |
| Where live lookups run | **Hugging Face (has internet); Snowflake mirror shows cached + internal fraud only** |

## 3. Scope

**In v1:** VIES (VAT validity + official name/address), GLEIF (identity, national reg #,
parent/ownership links), internal ISCC fraud flags (blocklist + compliance + cert status);
on-demand + cache; a Counterparty Profile panel in Verify; a rule-based trust verdict;
per-fact provenance in the UI; a GLEIF match-rate measurement spike.

**Out of v1 (explicitly deferred):** national registries (France INPI, Italy InfoCamere,
Germany Registerportal), EU Union Database (UDB), paid aggregators (Orbis/D&B/Creditsafe for
financials + beneficial ownership), batch pre-enrichment of all holders, public API exposure
(auth/rate-limit/hosting/terms), lead-generation scoring, AI narration of the profile.

## 4. Architecture — a pure engine + a thin UI

The engine contains **no Streamlit imports**; that is what keeps it reusable and
publishable later. Each module has one job and a well-defined interface.

```
enrich/                         ← pure Python package, no UI, independently testable
  models.py      dataclasses: CompanyInput, Field, ProviderResult, CompanyProfile, Verdict
  config.py      endpoints, timeouts, TTLs, feature flags, online() capability check
  cache.py       TTL cache backed by intel.db (enrichment_cache table)
  matching.py    name/address normalisation + GLEIF candidate scoring (with abstain)
  providers/
    base.py      Provider protocol: lookup(CompanyInput) -> ProviderResult
    vies.py      VAT  → validity + official name/address (REST, SOAP fallback); cleans VAT
    gleif.py     name+country → best LEI match → identity + registeredAs + parent links
    fraud.py     holder_norm → internal blocklist / compliance / cert-status  (no network)
  engine.py      orchestrator: build input → run providers (cache-aware) → assemble profile + verdict

enrichment_ui.py                ← thin Streamlit panel; reads a CompanyProfile, never calls a provider
```

**Module responsibilities & dependencies (each must be understandable in isolation):**

- `models.py` — data shapes only, no logic. Depends on nothing.
- `config.py` — constants + `online()` (probes whether external calls are possible in this
  environment). Depends on nothing app-specific.
- `cache.py` — `get(provider, input_key)` / `put(provider, input_key, result)` with TTL.
  Depends on `intel.db` (via the existing store layer) and `models`.
- `matching.py` — pure functions: `normalise(name)`, `score(candidate, input) -> 0..1`.
  No I/O. Depends on `models`.
- `providers/*` — each takes a `CompanyInput`, returns a `ProviderResult`, never raises
  to the caller (wraps its own errors). Depends on `models`, `config`, `matching` (gleif).
- `engine.py` — the only place that knows about all providers + the cache + the verdict
  rules. Depends on everything above. Exposes `profile(company) -> CompanyProfile`.
- `enrichment_ui.py` — Streamlit only. Depends on `engine`. Knows nothing about HTTP.

## 5. Data model (`enrich/models.py`)

```python
@dataclass
class CompanyInput:               # built from a Sinetti unified record
    scheme: str                   # "ISCC" | "REDcert" | "INS"
    certificate_id: str
    name: str                     # holder_name (ISCC: includes embedded address)
    norm: str                     # holder_norm
    country: str | None
    address: str | None           # ISCC: parsed from holder_name; REDcert/INS: city+postcode
    city: str | None
    postcode: str | None
    vat: str | None               # INS partita_iva, cleaned; else None
    codice_fiscale: str | None    # INS only

@dataclass
class Field:                      # one enriched value WITH provenance
    value: object
    source: str                  # "VIES" | "GLEIF" | "ISCC-lists"
    confidence: float            # 1.0 for exact (VIES/VAT), 0..1 for fuzzy (GLEIF)
    as_of: str                   # ISO date the fact was fetched

@dataclass
class ProviderResult:
    provider: str
    status: str                  # "ok" | "no_match" | "unavailable" | "not_applicable"
    fields: dict[str, Field]
    raw: dict | None             # cached for debugging / future re-parse

@dataclass
class Verdict:
    level: str                   # "verified" | "caution" | "high_risk"
    reasons: list[str]           # each reason carries its source + as_of

@dataclass
class CompanyProfile:
    input: CompanyInput
    identity: dict[str, Field]   # resolved name, address, VAT status, LEI, national reg #
    ownership: list[Field]       # GLEIF parent / child links
    certificates: list[dict]     # the Sinetti certs this entity holds (already local)
    risk: list[Field]            # fraud-list / withdrawn / suspended / VAT-invalid signals
    verdict: Verdict
    providers: list[ProviderResult]  # full per-provider status (incl. unavailable)
```

## 6. Provider designs

### 6.1 `vies.py` — VAT validity + official identity *(exact; INS subset)*
- **Applies when** `input.vat` is present (INS rows).
- **Input cleaning (in this module):** strip a leading country prefix (e.g. `FR `),
  validate the Italian company form (11 digits); reject 16-char individual fiscal codes;
  if it fails validation → `status="not_applicable"`.
- **Call:** `GET https://ec.europa.eu/taxation_customs/vies/rest-api/ms/{country}/vat/{vat}`
  (JSON REST); SOAP `checkVatService` as documented fallback.
- **Returns →** `Field`s: `vat_status` (valid/invalid), `registered_name`, `registered_address`,
  all `source="VIES"`, `confidence=1.0`.
- **Caveats:** Germany & Spain return validity only — moot (we hold no DE/ES VAT).
  Rate-limited, no bulk → cache (TTL ~30 days). On error/timeout → `status="unavailable"`.

### 6.2 `gleif.py` — identity, national reg #, ownership *(fuzzy; all schemes)*
- **Applies when** we have `name` + `country` (always).
- **Discover:** `GET https://api.gleif.org/api/v1/lei-records?filter[entity.legalName]={name}`
  `&filter[entity.legalAddress.country]={country}` (and/or `fuzzycompletions`).
- **Pick best candidate** via `matching.score()` (§7); if below the abstain threshold →
  `status="no_match"` (we do **not** guess).
- **Pull record:** `GET https://api.gleif.org/api/v1/lei-records/{LEI}` →
  `legal_name`, `registered_address`, **`registeredAs`** (national registration number),
  **`registeredAt`** (issuing registry).
- **Ownership:** `…/lei-records/{LEI}/direct-parent` and `…/ultimate-parent` →
  `Field`s with `source="GLEIF"`, `confidence=`match score.
- **Caveats:** only firms that hold an LEI are found (large traders skew in, small UCO
  collectors skew out — quantified by the §10 spike). GLEIF data is **CC0** (re-distributable,
  relevant to the publish-later option). Parent links are *legal* parent, **not** beneficial
  ownership (UBO is paid/restricted — report 02). TTL ~90 days.

### 6.3 `fraud.py` — internal screening *(no network; runs everywhere incl. SiS)*
- **Applies always.** Joins `input.norm` against the local overlays already in Sinetti:
  `data/iscc/blocklist.json.gz` (630 fake/excluded entities **[real]**),
  `data/iscc/compliance.json.gz` (677 suspended/withdrawn certs **[real]**), and the
  certificate's own status from `iscc_certs` / scheme data.
- **Returns →** risk `Field`s: `on_fraud_list` (fake/excluded), `cert_status`
  (valid/suspended/withdrawn/expired), each `source="ISCC-lists"`, `confidence=1.0`.
- **Why it matters:** this is the only provider that works with no internet, so the Snowflake
  mirror still delivers a real fraud screen.

## 7. Matching & confidence (`enrich/matching.py`)

- `normalise(name)` — lowercase, strip legal suffixes (SRL/GmbH/Ltd/SpA…), drop punctuation
  and the embedded address tail (ISCC), collapse whitespace.
- `score(candidate, input) -> 0..1` — combine: name similarity (token-set ratio) **+** country
  agreement (hard gate: different country caps the score low) **+** postcode/city agreement
  when available **+** address-token overlap (ISCC's embedded address is a strong signal).
- **Abstain threshold** (tuned by the §10 spike, start **0.85 [illustrative]**): below it,
  return `no_match` rather than a wrong identity. A wrong match is worse than no match in a
  due-diligence tool.

## 8. Trust verdict (`engine.py` rules) — transparent, not a black box

Evaluated in order; **every reason records its source + as_of**:

- 🔴 **high_risk** if: entity on fake/excluded list **OR** certificate withdrawn/suspended.
- 🟠 **caution** if (not high risk and any of): VAT present but invalid/unverifiable **OR**
  GLEIF `no_match` for a non-INS company **OR** certificate expired.
- 🟢 **verified** otherwise: (VAT valid **OR** confident GLEIF match) **AND** no fraud flag
  **AND** certificate currently valid.

The verdict is rule-based and inspectable — the UI lists each contributing reason so the user
sees *why*, and can disagree.

## 9. Caching, environment, error handling

- **Cache table (intel.db):**
  `enrichment_cache(provider TEXT, input_key TEXT, status TEXT, payload_json TEXT,
  fetched_at TEXT, PRIMARY KEY(provider, input_key))`. `input_key` = a stable hash of the
  normalised provider input (VAT for VIES; norm+country for GLEIF). TTL per provider (§6).
  Created via the existing `store_iscc` init path (SiS-safe, no auto-exec on import).
- **Environment (`online()`):** probes whether outbound calls are possible. On SiS it returns
  false → engine runs only `fraud.py` + serves any cache; live providers report
  `status="unavailable"` and the UI explains why. On Hugging Face it returns true.
- **Error handling:** every provider wraps its own I/O (timeout + try/except) and returns a
  `ProviderResult`, never raises. A failed provider degrades the profile, never breaks it.
  The profile always renders (at minimum: the local certs + fraud screen).

## 10. De-risking the one uncertain part — the GLEIF spike (build step 1)

The fuzzy GLEIF match is the only part whose reliability is unknown. **Before wiring it deeply,
build a throwaway measurement script** that:
1. samples N real ISCC + REDcert holders (e.g. **N = 300 [illustrative]**, stratified by country),
2. runs each through GLEIF discovery + `matching.score()`,
3. reports **hit-rate** (share with a confident match), **abstain-rate**, and a hand-checked
   **false-match estimate** on a small labelled subset.

Output: a real number that (a) sets the abstain threshold and (b) tells us whether the fuzzy
tier deserves deep investment or stays clearly-labelled best-effort. This directly tests the
"medium reliability" assumption instead of trusting it.

## 11. Testing strategy

- **Pure engine → deterministic tests, no live network in CI:**
  - provider tests run against **saved real JSON fixtures** (one VIES valid, one VIES invalid,
    one GLEIF hit, one GLEIF no-match) — captured once, replayed.
  - `matching.score()` tested on a labelled set of name/address pairs (true + false matches).
  - verdict tested as a **truth table** over (fraud flag × VAT status × cert status × match).
  - cache TTL tested (fresh vs expired vs miss).
- **Opt-in live smoke tests** (skipped in CI) hit VIES/GLEIF once to catch upstream drift.
- Fits Sinetti's existing pytest suite (146+ tests); target: no regression, all green.

## 12. Worked example (the output shape)

**VENANZIEFFE SRL** — INS, IT, `partita_iva = 10002290152` **[real holder]**:
1. `fraud.py`: `norm="venanzieffesrl"` not on fake/excluded list; cert status valid →
   no risk flag *(ISCC-lists, today)*.
2. `vies.py`: clean 11-digit IT VAT → `GET …/ms/IT/vat/10002290152` → **valid**, official
   name + address *(VIES, today, conf 1.0)* **[illustrative result]**.
3. `gleif.py`: discover by name+country → best candidate scores 0.91 ≥ 0.85 → LEI + national
   reg # + parent link *(GLEIF, today, conf 0.91)* **[illustrative result]**.
4. `engine.py` verdict: VAT valid AND no fraud flag AND cert valid → **🟢 verified**, with the
   three reasons + sources listed.

## 13. Build phases (hand-off to the implementation plan)

1. **GLEIF spike** (throwaway) → hit-rate number + abstain threshold.
2. **`models.py` + `config.py` + `cache.py`** + the Sinetti→`CompanyInput` adapter.
3. **`fraud.py`** (no network; immediate value, works on SiS) + tests.
4. **`vies.py`** (exact, INS subset) + input-cleaner + tests.
5. **`matching.py` + `gleif.py`** (fuzzy, all schemes) + tests, using the spike's threshold.
6. **`engine.py`** orchestration + verdict + truth-table tests.
7. **`enrichment_ui.py`** Counterparty Profile panel wired into Verify; provenance chips;
   offline/unavailable states.
8. Full-suite green; live-verify on Hugging Face; confirm SiS degrades gracefully.

## 14. Open questions (track, don't block v1)

- Final abstain threshold — set by the spike, not guessed.
- VIES usage terms for the eventual *publish* step (GLEIF is CC0; VIES redistribution TBD).
- Whether to add a resolved `company_identity` table now or after the cache proves out (YAGNI: start with the cache).
- EU UDB access model (the biggest unknown; research before any v2 wiring).
