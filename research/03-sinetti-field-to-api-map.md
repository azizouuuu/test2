# Sinetti (CERT) → Free-API Enrichment Map

> Design doc. Generated 2026-06-28.
> Inputs: the live Sinetti project at `C:\Users\AZIZ\sinetti` — SQLite `intel.db`
> (table `iscc_certs`) plus the gzip data files `data/redcert/certs.json.gz` and
> `data/ins/certs.json.gz`. All counts below were queried directly from that data
> (real, not illustrative).

---

## 0. Plain-language glossary (terms used below)

- **Primary identifier** — the one field that reliably and *uniquely* names a company,
  so a computer can look it up automatically without guessing.
- **VAT number** — the tax-registration number a business uses for sales tax. In Italy
  it is called *partita IVA*. Format for Italian companies: 11 digits.
- **Codice fiscale** — the Italian "fiscal code", a national tax/registration ID. For a
  *company* it is usually the same 11 digits as the partita IVA; for an *individual* it
  is a 16-character letter+number code.
- **LEI** — "Legal Entity Identifier", a free global 20-character code (e.g.
  `529900T8BM49AURSDO55`) issued under an ISO standard. Maintained by GLEIF.
- **VIES** — the EU's free online service that checks whether a VAT number is valid and
  (in most countries) returns the registered name and address.
- **GLEIF** — the global non-profit that runs the free LEI database and a free REST API.
- **Fuzzy match** — matching on approximate text (a name) instead of an exact code.
  Lower reliability: "OLFAR S/A" vs "Olfar SA Alimento e Energia" must be reconciled.

---

## 1. Headline answer: what primary identifier does Sinetti store?

**There is no single primary identifier across all of Sinetti.** What we store depends
entirely on which certification scheme the row came from. The picture splits cleanly:

| Scheme | Rows | Stored identifiers | VAT? | LEI? | Nat. reg. #? | Address fields |
|---|---:|---|:--:|:--:|:--:|---|
| **ISCC** | 16,202 | `certificate_id`, `holder_name` (name **with full address baked in**), `holder_norm`, `country` | ❌ | ❌ | ❌ | embedded inside `holder_name` |
| **REDcert** | 32,715 | `certificate_id`, `holder_name`, `holder_full`, `country`, `city`, `postcode` | ❌ | ❌ | ❌ | city + postcode (structured) |
| **INS (Italy)** | 22,582 | `certificate_id`, **`partita_iva`**, **`codice_fiscale`**, `holder_name`, `country`, `city`, `postcode` | ✅ | ❌ | ✅ (codice fiscale) | city + postcode |

**The common denominator we *always* have is: `certificate_id` + `holder_norm`
(normalised name) + `country`.** That is the real, dependable key.

**Strong tax identifiers exist for the Italian INS subset only.** Of 22,582 INS rows,
**22,267 carry a clean 11-digit Italian VAT** (partita IVA); 13 carry a foreign-prefixed
VAT (e.g. `FR 71449427145`); only 4 holders are anonymised (`OPERATORE0000xx`).

**No LEI is stored anywhere.** That matters because the handoff's decision rule was:

> Has VAT or LEI → robust automated free endpoints. Only name + country → fuzzy, manual.

So Sinetti sits in **both** tiers at once:
- **Italy/INS (~22k operators): robust tier.** We have a real VAT to drive VIES directly.
- **ISCC + REDcert (~49k certificates): fuzzy tier.** We have only name + country (plus an
  address), so enrichment must *discover* a clean identifier before it can do anything
  automated. **GLEIF name-search is the key that unlocks this tier** (see §3.2).

> ⚠️ Data-quality gotcha, INS VAT field: in 4,341 rows `partita_iva == codice_fiscale`.
> For companies that is *correct and expected* (Italian firms reuse the 11-digit code for
> both), so those are usable. But you must still (a) strip any country prefix and (b)
> reject 16-char individual codice-fiscale values before calling VIES — VIES expects the
> 11-digit company form.

---

## 2. Where the data physically lives (for the build)

- **ISCC** → SQLite table `iscc_certs` in `intel.db`.
- **REDcert** → `data/redcert/certs.json.gz` (loaded at runtime; not in SQLite).
- **INS** → `data/ins/certs.json.gz` (loaded at runtime; not in SQLite).
- **Fraud / status overlays (ISCC):** `data/iscc/blocklist.json.gz` (630 fake/excluded
  entities, keyed by `norm`), `data/iscc/compliance.json.gz` (677 suspended/withdrawn
  certs), and the SQLite tables `iscc_compliance_history` + `iscc_sync_log`.

So the "company table" is effectively three sources unified by the shared columns
`{certificate_id, holder_name, holder_norm, country, scope, feedstocks, valid_until, …}`.
The enrichment layer should read from a **unified view** over the three, not from
`iscc_certs` alone.

---

## 3. Field-by-field → free API endpoint map

Each enrichment is `Sinetti input → API call → useful output → caveat`.

### 3.1 VIES — VAT validation & official name/address  *(robust tier; INS only)*

- **Sinetti input:** `country` (`IT`) + `partita_iva` (cleaned to 11 digits).
- **Endpoint (REST, JSON):**
  `GET https://ec.europa.eu/taxation_customs/vies/rest-api/ms/{countryCode}/vat/{vatNumber}`
  e.g. `…/ms/IT/vat/10002290152`
- **Endpoint (SOAP, fallback):** `checkVatService` at
  `https://ec.europa.eu/taxation_customs/vies/services/checkVatService`
- **Returns:** `{ isValid, requestDate, name, address }`. Italy returns name + address.
- **Use for:** confirming the certified operator is a live, validly-registered VAT entity
  today (a due-diligence "is this a real, active company?" check) and getting an
  authoritative registered name/address to reconcile against the certificate's holder name.
- **Caveats:** Germany & Spain return validity only (no name/address) — but we have no DE/ES
  VAT anyway, so moot here. Rate-limited; no bulk method — cache results. Clean the input
  first (strip `FR `-style prefixes; reject 16-char individual codes).

### 3.2 GLEIF — LEI discovery, legal name, address, ownership, *and* national reg #  *(unlocks the fuzzy tier)*

GLEIF is the highest-leverage free source because we have **no LEI stored**, yet GLEIF lets
us go **name + country → LEI**, and an LEI record then hands back a clean national
registration number and parent/child ownership links — bootstrapping a proper identifier
out of just a name.

- **Step 1 — discover the LEI from name + country:**
  `GET https://api.gleif.org/api/v1/lei-records?filter[entity.legalName]={name}&filter[entity.legalAddress.country]={country}`
  (or `…/fuzzycompletions?field=entity.legalName&q={name}` for autocomplete-style matching).
- **Step 2 — pull the full record:** `GET https://api.gleif.org/api/v1/lei-records/{LEI}`
  → legal name, registered address, **`entity.registeredAs`** (the national registration
  number — e.g. an Italian REA / a French SIREN) and **`entity.registeredAt`** (which
  registry issued it).
- **Step 3 — ownership:** `…/lei-records/{LEI}/direct-parent` and `…/ultimate-parent`
  → the *legally-reported* parent links (this is the only **free** ownership signal we get;
  it is **not** beneficial ownership / UBO — see §4).
- **Use for:** corporate-identity enrichment across all three schemes; deriving the
  national reg number that then feeds §3.4; partial ownership graph for due diligence.
- **Caveats:** coverage is only firms that have an LEI (large/trading entities skew in,
  small UCO collectors skew out). Name matching is fuzzy — score candidates using the
  address we already hold (ISCC bakes the full address into `holder_name`; REDcert/INS give
  city + postcode) to disambiguate. CC0-licensed, free, bulk download also available.

### 3.3 ISCC public database — live certification status & fraud screening  *(mostly internal already)*

- **Sinetti input:** `certificate_id` and/or `holder_norm`.
- **Source:** primarily our **own** overlays — join `holder_norm` against
  `blocklist.json.gz` (`norm` key → fake/excluded) and `compliance.json.gz`
  (suspended/withdrawn). For *live* re-checks, the ISCC public DB / certificate lookup.
- **Returns:** status ∈ {valid, suspended, withdrawn, fake, excluded} + dates.
- **Use for:** the fraud-screening leg of due diligence — flagging a counterparty whose
  certificate is withdrawn or whose entity is on the fake/excluded list. (2024 context from
  report 01: 97 certs withdrawn, 130 companies suspended, 78 excluded.)
- **Caveats:** this is screening, not enrichment — it adds *risk flags*, not new company
  facts. Already largely built inside Sinetti; expose it as an endpoint rather than rebuild.

### 3.4 National registries — deep financials & directors  *(per-country, partial free)*

Drive these with the identifier discovered in §3.1/§3.2, not with the raw certificate name.

- **France — INPI / RNE (most open):**
  `https://registre-national-entreprises.inpi.fr/api/…` keyed by **SIREN** (9 digits). We
  have no SIREN, so obtain it first from GLEIF `registeredAs` (French LEIs) — then pull
  filed accounts + directors free. Requires a free account/token.
- **Germany — Registerportal / Handelsregister:** free since Aug 2022; name/HRB search and
  bulk, but no clean REST API — treat as a scrape/bulk source.
- **Italy — Registro Imprese / InfoCamere:** keyed by `codice_fiscale` (which **we have**
  for INS). No fully-free official API; documents via Telemaco are per-doc paid. The
  codice fiscale is still valuable as the exact join key for any Italian lookup.

### 3.5 EU Union Database (UDB) — open question

- No public third-party API confirmed (see report 01/handoff). Flag as the biggest unknown;
  investigate access model before designing against it. Do not assume it is free/open.

---

## 4. Walls to respect (don't design as if these are free)

- **Beneficial ownership (UBO):** restricted since the 2022 CJEU ruling (report 02). GLEIF
  parent/child links are *legal* parent data, **not** UBO. Real UBO needs a paid aggregator.
- **EU customs / trade data:** structurally limited; no clean free EU bill-of-lading source.
- **27-state deep financials:** free per country but 27 different systems — a paid
  aggregator (Orbis/D&B/Creditsafe) is the pragmatic normaliser for everything outside
  FR/DE/IT.

---

## 5. Worked example (numbers flowing end-to-end)

**Case A — INS / Italy, robust path.** Take the real INS row
`VENANZIEFFE SRL`, `partita_iva = 10002290152`, `country = IT`:
1. **VIES:** `GET …/rest-api/ms/IT/vat/10002290152` → `{ isValid: true, name, address }`.
   ✅ confirms the operator is a live, validly-registered Italian VAT entity, with an
   authoritative name/address to reconcile against the certificate.
2. **GLEIF:** `…/lei-records?filter[entity.legalName]=VENANZIEFFE SRL&filter[entity.legalAddress.country]=IT`
   → if matched, an LEI + `registeredAs` (REA/codice fiscale) + any parent link.
3. **Status:** join `holder_norm = "venanzieffesrl"` against the blocklist/compliance
   overlays → no flag → clean. **Result:** a fully-enriched, screened company record from a
   single VAT number.

**Case B — ISCC, fuzzy path.** Take `OLFAR S/A ALIMENTO E ENERGIA`, `country = BR`
(no VAT, no LEI stored, address embedded in `holder_name`):
1. **VIES:** not applicable (not EU; no VAT).
2. **GLEIF:** fuzzy-search `"OLFAR ... ALIMENTO E ENERGIA"` filtered to Brazil, scored
   against the embedded street address → resolve LEI → legal name + national reg (CNPJ) +
   ownership. This is the *only* automated identifier we can recover here.
3. **Status:** screen `holder_norm` against the fraud overlays.
   **Result:** enrichment is possible but reliability hinges on the fuzzy match — manual
   review fallback needed when no confident GLEIF hit.

---

## 6. Recommended build order

1. **VIES wrapper (INS subset).** Highest confidence, exact key already in hand
   (~22.3k clean Italian VATs). Build the input-cleaner (strip prefix, validate 11-digit)
   as part of it.
2. **GLEIF wrapper (all schemes).** The identifier-bootstrap that turns name+country into an
   LEI + national reg # + ownership. Unlocks ISCC/REDcert and feeds §3.4.
3. **Fraud/status endpoint.** Expose the existing blocklist + compliance joins as an API.
4. **National registries (FR INPI first, then IT via codice fiscale).** Only after GLEIF
   supplies the SIREN/REA keys.
5. **Investigate EU UDB access model** — the biggest open unknown.
6. **Build-vs-buy decision** for the paid gaps (UBO, 27-state financials, EU trade data).
