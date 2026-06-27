# Project Handoff — EU Biofuel Company Data Enrichment APIs

> Paste this into a fresh Claude Code session to continue the work.
> Last updated: 2026-06-27.

## TL;DR for the next session

I'm building an app centered on **sustainability certificates for bioenergy**
(REDcert, ISCC, and the Italian National System "INS" / Sistema Nazionale, plus
2BSvs and the EU Union Database / UDB). I **already have the certification-linked
data** (certificate numbers, scope, validity) in a project/database called
**Sinetti (CERT)**.

**Goal:** build **free, public APIs** that enrich the certified companies with
*complementary* data — financial/corporate, operational/production,
contact/commercial, and ESG/reputation — for two use cases:
(a) **due diligence / counterparty trust** (reduce biofuel-trading fraud risk), and
(b) **business development / lead generation**.

## What's already been done

Two deep-research reports live in this repo under `research/`:
- `research/01-enrichment-data-sources.md` — where to source the four categories of
  enrichment data (free + paid), with the key finding that **certification data
  alone is a weak trust signal**.
- `research/02-registries-vs-vies-vs-aggregators.md` — EU business registries vs
  VIES vs commercial aggregators, including the **2022 CJEU ruling** that restricted
  public access to beneficial-ownership (UBO) registers.

Read those two files first — they contain the verified facts this plan rests on.

## The immediate blocker / first task

**The Sinetti (CERT) database schema is NOT yet available to the assistant.**
The remote session was scoped to GitHub repo `azizouuuu/test2` only, which does not
contain Sinetti. To proceed, the next session needs Sinetti's **company-info fields**.

**First action for the next session:** locate Sinetti (CERT) and read its
company-info schema. Ask the user for one of:
- the table/column list for company info, or
- a sample CSV/JSON export, or
- the SQL schema, or the repo/path where Sinetti lives.

**The single most important thing to determine:** what is the **primary identifier**
Sinetti reliably stores per company? This decides how clean the enrichment APIs can be:
- Has **VAT number** or **LEI** → robust automated free-tier endpoints are possible.
- Only **name + country** → fuzzy matching, lower reliability, more manual work.

## Identifier → free-API map (the core design)

| If Sinetti has… | …it unlocks (free) |
|---|---|
| VAT number | VIES → live validity + name/address (most EU states; not DE/ES) |
| LEI | GLEIF free REST API + bulk → legal name, address, parent/ownership links |
| National reg. number (e.g. Italian REA / Codice Fiscale) | BRIS / national registry lookups |
| Country + legal name | BRIS cross-border search; ISCC certificate DB cross-ref |
| ISCC/REDcert cert number | ISCC public DB → status, validity, fraud/withdrawn flags |

## Free public APIs worth wrapping (realistic free tier)

1. **VIES** (VAT) — free SOAP/XML API. Identity-verification microservice.
   Caveat: Germany & Spain return validity only (no name/address); SOAP, not REST.
2. **GLEIF** (LEI) — free REST API + bulk download. Ownership/parent enrichment.
3. **ISCC certificate database** — free, searchable, CSV export. Certification status
   + fraud screening (Withdrawn / Suspended / Fake / Excluded lists).
4. **France INPI open data** (data.inpi.fr) and **German Registerportal** — free,
   where financials are open.
5. **EU Union Database (UDB)** — OPEN QUESTION: unclear what it exposes to third
   parties. Worth investigating; potentially high-value traceability data.

## Known walls (don't design around these as if free)

- **Beneficial ownership (UBO):** since the 2022 CJEU ruling, public access is
  restricted to a "legitimate interest" regime — fragmented, often refused, no
  historical data. Practically requires a paid aggregator (Orbis/BvD, D&B,
  Creditsafe). Do NOT assume you can pull UBO directly from registries.
- **EU customs / trade data:** structurally limited (EU confidentiality). ImportGenius
  (EU is a paid add-on) and Panjiva (no EU member states) won't give clean EU
  bill-of-lading data.
- **Deep financials across all 27 states:** free per-country, but 27 different
  systems/languages/formats. A paid aggregator is the pragmatic normalizer.

## Suggested next steps (in order)

1. Get the Sinetti (CERT) company-info schema; identify the primary identifier(s).
2. Map each Sinetti field to a concrete free-API endpoint (input → output → caveats).
3. Prototype the highest-confidence wrappers first: **VIES** (VAT validation) and
   **GLEIF** (LEI/ownership), since both are free and well-defined.
4. Cross-reference the **ISCC database** for live certification status + fraud flags.
5. Investigate the **EU UDB** access model (the biggest open unknown).
6. Decide build-vs-buy for the paid gaps (UBO, deep financials, EU trade data).

## Repo / workflow context

- GitHub repo: `azizouuuu/test2`
- Work branch: `claude/github-integration-question-n8h7hy` → open as **PR #1**
  (https://github.com/azizouuuu/test2/pull/1). New commits to this branch update PR #1.
- The research files and this handoff are committed on that branch.
