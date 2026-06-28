# Sinetti Enrichment Engine — Design Spec (v1)

> Design spec. Generated 2026-06-28. Revised after an adversarial review + web fact-check pass
> (11 agents). Status: **pending written review by the user.**
> Builds on `research/01-enrichment-data-sources.md`, `research/02-registries-vs-vies-vs-aggregators.md`,
> and `research/03-sinetti-field-to-api-map.md` (all at the repo root of `azizouuuu/test2`).
> The engine code is built in the **local `sinetti` repo** (not this one); this repo holds the design record.

---

## 0. Plain-language glossary

- **Enrichment** — adding extra, trustworthy facts about a company we already know about, by
  looking it up in outside databases.
- **Counterparty** — the other company in a deal (the one you check before you trust them).
- **Due diligence** — the background check you run on a counterparty before trusting them.
- **VAT number** — a business's sales-tax registration number. Italy calls it *partita IVA*.
- **VIES** — the EU's free service that says whether a VAT number is valid and (in most countries)
  returns the registered name and address.
- **LEI** — "Legal Entity Identifier", a free global 20-character company code (ISO 17442), run by
  **GLEIF**, the non-profit that publishes the LEI database and a free web API.
- **Fuzzy match** — matching by approximate text (a name) rather than an exact code; less certain,
  so it needs a confidence score and the option to **abstain** (return "no match").
- **Provider** — one outside source we consult (VIES is a provider; GLEIF is a provider).
- **Adapter** — the code that turns a raw Sinetti certificate row into the clean input the engine needs.
- **Cache** — a local copy of an answer we already fetched, so we don't ask the source again.
- **TTL ("time to live")** — how long a cached answer stays usable before we refresh it.
- **Provenance** — for each fact, *where it came from* and *when* (so the user can trust it).
- **Reconciliation** — checking that the name a VAT/LEI resolves to actually matches the name on
  the certificate. A VAT that resolves to a *different* company is a red flag.
- **SiS (Streamlit-in-Snowflake)** — the Snowflake-hosted copy of Sinetti. By default it **cannot
  reach the internet**, so live external lookups don't run there.

> Numbers in examples are flagged **[illustrative]** unless queried from live Sinetti data, where
> they're flagged **[real]**.

---

## 1. Goal & the facts it rests on

**Goal:** a self-contained engine that takes one certified company from Sinetti and returns a
**Counterparty Profile** — a verified identity, a registry-name reconciliation check, and a
transparent trust verdict — by consulting *free* sources on demand and caching the answers. v1
surfaces it inside Sinetti's existing **global verifier** (`render_global_verify`) and the Browse
"Verify" expander. The engine is a pure Python package with **no Streamlit imports**, purely for
reusability and testability (see the publish-later non-goal in §3).

**Load-bearing facts (verified this round):**

1. **Certification data alone is a weak trust signal** (report 01) — so layering identity + a
   name-reconciliation check + fraud flags on top is where the value is.
2. **Sinetti has no single primary identifier** (report 03). Only the Italian **INS** subset
   carries a real tax ID (~22,267 of 22,582 INS rows have a clean 11-digit Italian VAT) **[real]**;
   **ISCC + REDcert give only name + country (+ address)**, and **no LEI is stored anywhere**.
3. **VIES REST is exactly as designed** — `GET …/vies/rest-api/ms/{country}/vat/{vat}` returns JSON
   `{isValid, name, address, userError, …}`; **but every outcome is HTTP 200** and the result is
   carried in `userError` (`VALID` / `INVALID` / `MS_UNAVAILABLE` / `*_MAX_CONCURRENT_REQ`), with
   `name`/`address` = `"---"` when not valid. **[verified live]**
4. **GLEIF API verified** — `api.gleif.org/api/v1`, `lei-records` with
   `filter[entity.legalName]` + `filter[entity.legalAddress.country]` (ISO alpha-2),
   `fuzzycompletions`, `/lei-records/{LEI}`, and parent endpoints that **return HTTP 404 when there
   is no reported parent** (a normal outcome, not an error). National reg # is
   `entity.registeredAs`; the registry is `entity.registeredAt.id` (an **object**, RA code). GLEIF
   data is **CC0** (redistributable). **[verified live]**
5. **VIES results may NOT be cached and re-served to third parties** — the VIES disclaimer forbids
   retransmission/copying. A user validating *their own* counterparties is permitted; a public API
   re-serving cached VIES verdicts is **not**. This is a hard constraint, not an open question. **[verified]**
6. **SiS has no outbound internet by default** — enabling it needs a Snowflake External Access
   Integration. We do **not** do that in v1. **[verified]**

## 2. The decisions locked in brainstorming

| Decision | Choice |
|---|---|
| What we build | Consume free sources now; pure engine kept clean for possible reuse later |
| Where it lives | Standalone pure-Python engine + a thin Sinetti UI panel in the global verifier |
| When lookups run | On-demand when a company is opened, then cached |
| v1 primary use case | Due diligence / counterparty trust |
| v1 sources | VIES + GLEIF (identity) + internal ISCC fraud flags |
| Where live lookups run | Hugging Face (internet); Snowflake mirror shows cached + internal fraud only |

## 3. Scope

**In v1:** VIES (VAT validity + official name/address), GLEIF (identity + national reg #),
internal ISCC fraud flags, a **registry-name reconciliation** check, a four-state trust verdict,
on-demand + cache, per-fact provenance, and a GLEIF match-rate measurement spike.

**Fast-follow (v1.1), explicitly out of v1:** GLEIF **ownership** (direct/ultimate parent links).
It is decorative — it does *not* feed the verdict — needs extra calls + its own UI, and legal
parent is **not** beneficial ownership. It is the first thing to defer if the GLEIF phase runs long.

**Deferred (later):** national registries (FR INPI, IT InfoCamere, DE Registerportal), EU UDB,
paid aggregators (financials + real beneficial ownership), batch pre-enrichment, lead-gen scoring,
AI narration.

**Non-goal for v1 (one line, to avoid premature generality):** *publishing a public API.* We keep
the pure-engine boundary (justified by reuse/testability alone), but no implementer should build
auth, rate-limit headers, or versioned response envelopes now. We carry exactly one forward-guard:
a per-provider **`redistributable`** flag (GLEIF=true/CC0, VIES=false/self-use-only) that documents
fact 5 so a future API can't accidentally re-serve VIES.

## 4. Architecture — a pure engine + a thin UI

No module under `enrich/` imports Streamlit. External I/O lives only in `providers/`. App-specific
data access (the fraud overlays, the cache DB) is reached through **injected ports**, so the engine
runs headless and is testable with in-memory fakes.

```
enrich/                         ← pure Python package, no UI, no direct file/DB/sqlite access
  models.py      dataclasses: CompanyInput, Field, ProviderResult, NameMatch, Verdict, CompanyProfile
  config.py      endpoints, timeouts, TTLs (status-aware), abstain_threshold, KEY_VERSION
  adapter.py     from_sinetti_record(row, scheme) -> CompanyInput   (all scheme-quirk parsing here)
  matching.py    normalise(name); score(candidate, input) -> 0..1   (pure; NO threshold inside)
  ports.py       Protocols: CacheStore (get/put), ScreeningSource (blocklist/compliance/cert status)
  providers/
    base.py      Provider protocol: lookup(CompanyInput) -> ProviderResult; carries .redistributable
    vies.py      VAT → validity + official name/address (parses userError); HTTP only
    gleif.py     name+country → discover → score → pick → /lei-records/{LEI} → identity fields
    fraud.py     wraps an injected ScreeningSource; NO file/DB access of its own; no network
  engine.py      orchestrates: adapter → providers (cache+capability aware) → reconcile → verdict

sinetti side (thin, Streamlit-aware):
  enrich_store.py     SQLite + Snowflake CacheStore impls + the enrichment_history table
  enrich_screening.py ScreeningSource backed by existing iscc.blocklist_match()/compliance/cert lookups
  enrichment_ui.py    the Counterparty Profile panel; reads a CompanyProfile; calls no provider
```

**Module responsibilities & dependencies:**

- `models.py` — shapes only. Depends on nothing.
- `config.py` — constants + `KEY_VERSION` (bump to invalidate caches when cleaning/normalise logic
  changes). No I/O.
- `adapter.py` — the one place that knows the three scheme shapes (§6). Pure functions; depends on
  `models` + Sinetti's existing parse helpers (imported, not re-implemented — see §6).
- `matching.py` — pure `normalise` + `score`. No I/O, no threshold. Depends on `models`.
- `ports.py` — the two interfaces the engine needs from the host app, so the engine never imports
  `iscc.py`, `store_iscc.py`, sqlite3, or touches `.gz` files.
- `providers/*` — each takes a `CompanyInput`, returns a `ProviderResult`, **never raises** to the
  caller. `vies`/`gleif` do HTTP; `fraud` uses the injected `ScreeningSource`. Depend on `models`,
  `config`, `matching` (gleif).
- `engine.py` — the only module that knows all providers + cache + capability flag + verdict rules.
  Exposes `profile(input, *, external_allowed, cache, screening) -> CompanyProfile`.
- `enrichment_ui.py` — Streamlit only; depends on `engine`; knows nothing about HTTP or DB shape.

## 5. Data model (`enrich/models.py`)

```python
@dataclass
class CompanyInput:
    scheme: str                   # "ISCC" | "REDcert" | "INS"
    certificate_id: str
    name: str                     # cleaned company name (no embedded address)
    norm: str                     # holder_norm (reuses Sinetti's _holder_norm)
    country: str | None           # ISO-3166 alpha-2, normalised here
    address: str | None
    city: str | None
    postcode: str | None
    vat: str | None               # cleaned digits-only IT VAT, or None
    cert_status: str              # canonical enum: valid|suspended|withdrawn|expired|unknown
    anonymised: bool              # True for INS OPERATORE000xxx placeholders (name unusable)

@dataclass
class Field:
    value: object
    source: str                   # "VIES" | "GLEIF" | "ISCC-lists"
    confidence: float             # 1.0 exact; 0..1 fuzzy (GLEIF identity = LEI match score)
    as_of: str                    # ISO date fetched

@dataclass
class NameMatch:                  # registry-name reconciliation result
    level: str                    # "match" | "partial" | "mismatch" | "n/a"
    score: float
    compared_to: str              # which source's name we compared against the cert

@dataclass
class ProviderResult:
    provider: str
    status: str                   # "ok" | "no_match" | "invalid" | "unavailable" | "not_applicable"
    fields: dict[str, Field]
    redistributable: bool         # provenance for the publish-later guard

@dataclass
class Verdict:
    level: str                    # "high_risk" | "caution" | "verified" | "insufficient_data"
    reasons: list[str]            # each reason carries its source + as_of

@dataclass
class CompanyProfile:
    input: CompanyInput
    identity: dict[str, Field]    # resolved name, address, VAT status, LEI, national reg #
    name_match: NameMatch
    risk: list[Field]             # fraud-list / cert-status / VAT-invalid signals
    verdict: Verdict
    providers: list[ProviderResult]  # full per-provider status incl. unavailable/not_applicable
    # ownership: list[Field]  ← v1.1 fast-follow only
```

## 6. The Sinetti→CompanyInput adapter (`enrich/adapter.py`)

This is the real coupling seam, so it's a named, single-responsibility, per-scheme module with its
own tests. It **reuses existing Sinetti helpers** rather than inventing parsers:

- **ISCC** (rows from `iscc_certs`): `name` via `iscc._company_name(holder)`, `address` via
  `iscc._address_part(holder)` / `iscc._norm_addr(holder)` (the embedded address is split on the
  first comma; `_norm_addr` drops addresses < 12 chars). `vat=None`. `cert_status` mapped from the
  row's `raw_status`/`suspended_date`.
- **REDcert** (`data/redcert/certs.json.gz`): `name=holder_name`, structured `city`+`postcode`
  (+`holder_full`). `vat=None`. `cert_status` from the row's `status`.
- **INS** (`ins.load_ins_certs()`, `data/ins/certs.json.gz`): the operator snapshot has
  `status="active"` hardcoded and **blank validity** — real lifecycle status lives only in
  `ins.load_cert_pdf_index()`, joined by tax-code keys. The adapter **must** perform that join to
  set `cert_status` (else INS cert status is `unknown`). `vat` = digits-only of `partita_iva` (else
  `codice_fiscale`); **country guard**: only keep `vat` usable for VIES when `country=="IT"` and the
  value isn't foreign-prefixed; set `anonymised=True` and skip GLEIF name use for the 4 `OPERATORE000xxx`
  placeholders **[real]**.
- **All schemes:** `norm` reuses the scheme's existing `_holder_norm`; `country` is normalised to
  ISO-3166 **alpha-2** before it can reach the GLEIF filter (a wrong code silently returns zero hits
  and would inflate the spike's no-match rate).

**Canonical `cert_status` enum:** `valid | suspended | withdrawn | expired | unknown`. Each scheme's
raw status is mapped onto it **here**, so the verdict (§9) reasons over one closed domain.

## 7. Provider designs

### 7.1 `vies.py` — VAT validity + official identity *(exact; INS-with-VAT only; redistributable=False)*
- **Applies when** `input.vat` is present and the country guard passed. Else `not_applicable`.
- **Call:** `GET https://ec.europa.eu/taxation_customs/vies/rest-api/ms/{country}/vat/{vat}`
  (path-param GET, no body). SOAP `checkVatService` WSDL is a documented fallback (its boolean field
  is `valid`, **not** `isValid` — both paths map to the same `Field`s).
- **Parse every HTTP 200 body:** read `isValid` + `userError`. Map `VALID`→`status="ok"` with
  `vat_status=valid`, `registered_name`, `registered_address`; `INVALID`→`status="invalid"`;
  `MS_UNAVAILABLE` / `*_MAX_CONCURRENT_REQ`→`status="unavailable"` (**not** invalid). Treat
  `name`/`address == "---"` as "no fields returned." Reserve the try/except→`unavailable` path for
  true transport errors. `confidence=1.0`.
- **Caveats:** rate-limited, no bulk; cache TTL ~30d (status-aware, §8). DE/ES return validity only
  (moot — we hold no DE/ES VAT). **Redistribution forbidden (fact 5):** the cache is for the user's
  own interactive re-use, never a third-party re-serving channel.

### 7.2 `gleif.py` — identity & national reg # *(fuzzy; all schemes; redistributable=True/CC0)*
- **Discovery algorithm (fixed, and the §11 spike MUST use this identical path):** call the
  structured endpoint `…/lei-records?filter[entity.legalName]={name}&filter[entity.legalAddress.country]={alpha2}`;
  **if it returns zero candidates, fall back** to `…/fuzzycompletions?field=entity.legalName&q={name}`;
  union + de-dupe candidates by LEI.
- **Pick:** `matching.score()` each candidate; take the best. If `best < config.abstain_threshold`
  → `status="no_match"` (we do **not** guess).
- **Pull:** `GET …/lei-records/{LEI}` → `legal_name`, `registered_address`,
  `national_reg = data.attributes.entity.registeredAs` (string),
  `registry = data.attributes.entity.registeredAt.id` (RA code; `.other` as nullable fallback; resolve
  the RA code to a name via GLEIF's Registration Authorities list for display). Identity `Field`s carry
  `confidence = best match score`.
- **Caveats:** only LEI-registered firms are found (small UCO collectors skew out — quantified by the
  §11 spike). Don't display the GLEIF name/logo or imply endorsement (trademark, separate from CC0
  data). TTL ~90d (status-aware). Ownership endpoints are **v1.1**; note 404 = "no parent" there.

### 7.3 `fraud.py` — internal screening *(no network; runs everywhere incl. SiS; redistributable=True)*
- **Depends only on an injected `ScreeningSource` port** — it does **not** read `.gz` files or query
  `iscc_certs` itself. In Sinetti the port (`enrich_screening.py`) wraps the existing
  `iscc.blocklist_match(name)` (exact normalised match, 630 entries **[real]**),
  `iscc._load_iscc_compliance()` (677 suspended/withdrawn **[real]**), and the cert-status lookup; in
  tests it's an in-memory fake.
- **Returns** risk `Field`s: `on_fraud_list` (fake/excluded), `cert_status` (canonical enum), each
  `source="ISCC-lists"`, `confidence=1.0`.
- **Honest scope limit (stated in the UI):** the blocklist + compliance overlays are **ISCC-only**. A
  REDcert or INS holder gets cert-status screening but **no fraud-list coverage** — the panel must say
  so rather than imply a clean fraud screen.

## 8. Matching, caching, capability

- **`matching.py`** — `normalise(name)` (lowercase, strip legal suffixes SRL/GmbH/Ltd/SpA…,
  punctuation, the ISCC embedded-address tail). `score(candidate, input) -> 0..1` combines name
  token-set similarity + a **country hard-gate** (different country caps the score low) + city/postcode
  agreement + address-token overlap. **No threshold lives here** — the single `abstain_threshold`
  lives in `config.py` and is injected to `gleif.py`. Starting value **0.85 [illustrative]**, set by
  the §11 spike.
- **Cache (`CacheStore` port, two impls):** the engine depends on a `get(provider, input_key)` /
  `put(provider, input_key, status, payload, ttl)` interface — **not** on sqlite. `enrich_store.py`
  provides a SQLite impl (the `store_iscc` pattern) **and** a Snowflake impl (the
  `store_iscc_snowflake` pattern), selected by `runtime.in_snowflake()`, so the SiS "cached + fraud
  only" read path actually works.
  - **Table:** `enrichment_cache(provider, input_key, status, payload_json, fetched_at,
    PRIMARY KEY(provider, input_key))`.
  - **`input_key` (one precise definition):** `sha256_hex(f"{KEY_VERSION}|{canonical}")` where
    `canonical` is the cleaned VAT for VIES and `f"{norm}|{country}"` for GLEIF. `KEY_VERSION` bumps
    invalidate stale keys when cleaning logic changes.
  - **Status-aware TTL:** `ok` → long provider TTL (VIES 30d / GLEIF 90d); `no_match` → short (e.g.
    7d **[illustrative]**) so absences don't outlive data refreshes; `unavailable` → **not cached**
    (retry next time). Freshness check reads the stored `status`.
  - **`raw` provider JSON is NOT persisted** — kept in-memory for the current request only, so cache
    rows stay decoupled from upstream response shape.
  - **`fraud` is not cached** — it reads local in-memory overlays already; caching would only add
    staleness.
- **Capability (no probe):** reuse the existing `ui.live_fetch_available()` ==
  `not runtime.in_snowflake()` — a static, no-I/O check. The engine receives `external_allowed: bool`;
  when false it runs only `fraud` + serves any cache, and live providers report
  `status="unavailable"`. A genuinely-offline non-SiS host is handled by the providers' own
  try/except, not by a network probe.

## 9. Reconciliation + trust verdict (`engine.py`)

**Registry-name reconciliation (the core trust signal):** compare the resolved registered name
(VIES preferred, else GLEIF) against the certificate's `name` via `matching.normalise`/`score` →
`NameMatch{level: match|partial|mismatch|n/a}`. A counterparty whose VAT/LEI resolves to a *different*
company is exactly the fraud signal this tool exists to surface.

**Verdict — four states, evaluated as first-match short-circuit** (evaluate high_risk, then caution,
then verified, else insufficient_data; the parenthetical guards are redundant given short-circuit and
exist only for readability). **Every reason records its source + as_of.**

- 🔴 **high_risk** — entity on fake/excluded list **OR** `cert_status ∈ {withdrawn, suspended}`.
- 🟠 **caution** — any *adverse* signal: VAT present but `INVALID` **OR** `cert_status == expired`
  **OR** `name_match == mismatch` **OR** two positive sources disagree on the registered name.
- 🟢 **verified** — a positive identity confirmation (`vat_status == valid` **OR** GLEIF
  `status == ok`) **AND** no adverse signal **AND** `cert_status == valid` **AND**
  `name_match ∈ {match, partial, n/a}`.
- ⚪ **insufficient_data** — clean (no adverse signal, cert not withdrawn/suspended) but **no positive
  external confirmation** available — e.g. a non-EU, no-VAT company with GLEIF `no_match`, or an INS
  row with `cert_status == unknown` and no usable VAT. **"We couldn't confirm" is not "suspicious."**

**Why the 4th state:** small UCO collectors systematically lack an LEI (§7.2) and most are outside the
VAT subset, so without `insufficient_data` a large, predictable, *legitimate* population would be
mis-flagged 🟠. `cert_status == unknown` (the common INS case) is **neutral**, never a caution trigger.

## 10. Error handling

Every provider wraps its own I/O (per-provider timeout: ~5s connect / 10s read) and returns a
`ProviderResult`, **never raises**. A failed provider degrades the profile, never breaks it — the
panel always renders at least the local certs + fraud screen. **Rate-limiting:** v1 calls providers
**sequentially** (no concurrency); retry once with backoff on HTTP 429 / timeout, then
`status="unavailable"`; a hard per-session call cap. The §11 spike adds a token-bucket / minimum
inter-request delay so it can't trip GLEIF/VIES limits.

## 11. De-risking the one uncertain part — the GLEIF spike (build phase 1)

A throwaway script that: (1) samples N real ISCC + REDcert holders (e.g. **N=300 [illustrative]**,
stratified by country, **country verified alpha-2** first), (2) runs each through the **exact §7.2
discovery path** + `matching.score()` (throttled per §10), (3) reports **hit-rate** (share with
`status==ok`), **abstain-rate**, and a hand-checked **false-match estimate** on a labelled subset.

**Explicit decision gate after the spike:** if the confident-match rate is below an agreed floor
(e.g. **40% [illustrative]**), the fuzzy GLEIF tier ships as **clearly-labelled best-effort** (no
ownership, reduced UI) rather than as a headline feature. The spike also fixes the `abstain_threshold`.

> **Spike result (run 2026-06-28, live GLEIF, 0 HTTP errors):** confident-match (name-similarity ≥ 0.85)
> on ISCC + REDcert holders was **~25% at N=80** (43% on a noisier N=30). **Below the 40% gate → GLEIF
> ships as best-effort for ISCC/REDcert.** Decision: keep the engine and `ABSTAIN_THRESHOLD = 0.85`
> (a conservative bar avoids wrong identities in a due-diligence tool); the UI labels GLEIF identity as
> best-effort; for non-INS holders the verdict will commonly be ⚪ `insufficient_data` — which is the
> honest outcome the four-state verdict was designed for, **not** a defect. INS/VIES remains the
> high-confidence path. **No build changes required** — the architecture already absorbs low GLEIF
> coverage by treating a no-match as neutral.

## 12. Testing strategy

Pure engine → deterministic, no live network in CI:
- **Contract tests** pinning the exact request URL/params each provider builds (catches filter/URL
  drift offline).
- **Recorded cassettes** (one per provider) so the full discover→pick→pull / parse chain runs in CI
  against real recorded bytes; live smoke tests stay opt-in (upstream-drift only).
- **GLEIF multi-call fixtures:** discovery-empty (→ fuzzycompletions fallback), below-threshold
  (→ abstain), and a hit; **not** just one happy path.
- **VIES fixtures:** `VALID`, `INVALID`, `MS_UNAVAILABLE`, and the `not_applicable` paths (blank
  VAT, foreign/empty country, 16-char codice fiscale).
- **Adapter fixtures per scheme:** ISCC holder with/without embedded address; INS with no VAT; INS
  foreign `country==""`; INS status resolved only via the cert-PDF join; an `OPERATORE000xxx`
  placeholder.
- **Round-trip cache test:** `put` then `get` hit the **same `input_key`** for the same
  `CompanyInput` (catches writer/reader key drift that would cause 100% live cache misses).
- **Verdict truth table** over (`on_fraud_list` × `vat_status` × `cert_status` incl. `unknown` ×
  `name_match` × gleif `status`) — every cell assigned to exactly one of the four states, including
  the `insufficient_data` cases.
- Fits Sinetti's existing pytest suite (~179 test functions today). Target: **no regression, all green.**

## 13. Audit trail

A lightweight **`enrichment_history(holder_norm, scheme, verdict_level, reasons_json, produced_at,
produced_by)`** table (same `store_iscc` init pattern, `produced_by` best-effort from the runtime
user) so a due-diligence check is a **defensible, re-displayable record** without re-fetching. This
is small and aligns with existing Sinetti history tables (`iscc_compliance_history`, `iscc_sync_log`);
it is in v1, not deferred.

## 14. UI integration (concrete, against the real app)

There is **no "Verify" nav section** — `app.py:123` folds Verify into a Browse expander, and the
verifier is `render_global_verify()` (a substring search over all three registers rendering compact
match rows via `_verify_match_rows`, with risk/blocklist banners). The trigger:

- Add a **"Run due-diligence profile"** affordance per matched holder in the verifier results,
  storing the selected `holder_norm` + `scheme` in `session_state`.
- A profile is produced for **one `holder_norm` within one scheme** (a `group_entities` entity); when
  a holder has several certs, use the most recent/most-scoped cert for `cert_status`. Specify the
  zero-match and many-match cases (the verifier already disambiguates; the panel renders after the
  user picks a row).
- `enrichment_ui.py` renders the `CompanyProfile` inline beneath the chosen result: identity with
  source chips + as_of, the `NameMatch` reconciliation line, the risk/fraud signals (with the
  ISCC-only honesty note for REDcert/INS), and the four-state verdict with its reasons. Offline/SiS
  shows fraud + cache only, with a plain "live lookups unavailable here" note.

## 15. Build phases (hand-off to the implementation plan)

1. **GLEIF spike** (throwaway) → hit-rate + abstain threshold + the §11 decision gate.
2. **`models.py` + `config.py` + `adapter.py`** + `ports.py` — *foundation; verified by unit tests
   only (not user-facing — acknowledged).*
3. **`enrich_screening.py` + `fraud.py`** → ships the **offline fraud screen first** (works on SiS).
4. **`enrich_store.py` (CacheStore + history) + `vies.py`** — cache built alongside its first consumer.
5. **`matching.py` + `gleif.py`** (identity only) using the spike's threshold + discovery path.
6. **`engine.py`** — reconciliation + four-state verdict + the truth-table tests.
7. **`enrichment_ui.py`** wired into `render_global_verify`; provenance chips; offline states.
8. Full suite green; live-verify on Hugging Face; confirm SiS degrades to fraud + cache.
9. *(v1.1 fast-follow)* GLEIF ownership links.

## 16. Open questions (track, don't block v1)

- Final `abstain_threshold` + the spike's go/no-go floor — set by phase 1, not guessed.
- The agreed `enrichment_history` retention + whether `produced_by` is available in all deploys.
- EU UDB access model (the biggest unknown; research before any deferred-tier wiring).
