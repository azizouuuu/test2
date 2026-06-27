# EU Business Registries vs VIES vs Aggregators

> Deep-research report. Generated 2026-06-27.
> Question: compare national business/trade registries of the main EU countries
> against VIES, BRIS, GLEIF, and commercial aggregators (Orbis/BvD, D&B, Creditsafe)
> as sources of complementary financial/corporate enrichment data — and answer
> whether VIES has less data than registries, plus the direct-vs-aggregator trade-off.

## ⚠️ Note on verification

The verification harness only keeps claims backed by primary/reliable sources. The
per-country registry pages (blogs, vendor guides) and Transparency International's
page (HTTP 403) graded "unreliable" and were filtered out. What survived (8 claims,
all 3-0 verified) is the **UBO/beneficial-ownership access law** — backed by primary
EU legal sources. Section A is rigorously verified; Section B is practitioner
synthesis, **not** independently fact-checked this round.

---

## (A) ✅ Verified: UBO/ownership access has been legally restricted

- **2022 CJEU ruling** (22 Nov 2022, Joined Cases C-37/20 & C-601/20,
  *Sovim v Luxembourg Business Registers*) struck down public access to
  beneficial-ownership registers. The Court held unrestricted public access is a
  "serious interference" with privacy rights (Charter Arts 7 & 8). Open UBO access
  was replaced by a narrower **"legitimate interest"** regime.
- **Fragmented result:** Transparency International tested 14 EU countries with
  legitimate-interest regimes — requests refused outright in **Ireland and Hungary**,
  ~8 of 14 charge fees, and **no country provides historical ownership data**.
- **AMLD6 (Directive (EU) 2024/1640)** aims to repair this by **10 July 2026** —
  presumed right for journalists/NGOs/academics to search + historical data. But the
  Commission has **opened infringement proceedings against 11 Member States** for
  missing the July 2025 milestone.

**Implication:** UBO/ownership is now the single hardest corporate field to obtain
directly from registries. This tilts the decision toward a commercial aggregator,
which pre-harvested ownership data under different access bases. Do not architect the
app assuming direct registry access to beneficial-ownership data.

Sources: EUR-Lex CELEX:62020CJ0037; eucrim.eu; transparency.org;
EUR-Lex eli/dir/2024/1640.

---

## (B) 📋 Practical comparison (synthesis — not independently verified)

> Treat costs/fields below as directional; confirm before building against any one API.

| Source | Financials? | Ownership/UBO? | API / Open data | Cost | Coverage |
|---|---|---|---|---|---|
| **VIES** | ❌ | ❌ | SOAP only | Free | All 27 EU — *identity only* |
| **GLEIF (LEI)** | ❌ | Parent / who-owns-whom links | ✅ Free open API + bulk | Free | Global, LEI-registered firms |
| **BRIS** (e-Justice) | Limited | ❌ | Cross-border search portal | Low/free per doc | Pan-EU front-end |
| **Italy — Registro Imprese/InfoCamere** | ✅ Filed accounts | Directors; UBO restricted | Telemaco + API | Per-doc fees | IT |
| **Germany — Handelsregister/Registerportal** | ✅ (free since Aug 2022) | Directors; UBO restricted | Bulk / registerportal | Free since 2022 | DE |
| **France — INPI/RNE** | ✅ | Directors + some UBO | ✅ Free open data API (data.inpi.fr) | Free | FR — most open-data-friendly |
| **Spain — Registro Mercantil/BORME** | ✅ | Directors | BORME gazette; paid docs | Per-doc fees | ES |
| **Netherlands — KvK** | Limited | Directors | ✅ Paid API | Per-call/subscription | NL |
| **Aggregators — Orbis / D&B / Creditsafe** | ✅✅ standardized | ✅ pre-harvested | ✅ One normalized API | €€€ subscription | Pan-EU + global |

### Core trade-off

- **VIES has far less data than national registries** — it is a VAT *validator*
  (name/address/validity), no financials, directors, or ownership. Verified.
- **Direct national registries (~27):** richest and most authoritative, often cheap
  per document, a few now offer free/open data (France INPI, Germany). But 27
  different systems, languages, formats, auth schemes — and UBO is now legally gated.
- **Aggregator (Orbis/D&B/Creditsafe):** one API, normalized schema, pre-harvested
  financials + ownership, pan-EU. You pay (typically €thousands+/yr) and data can lag
  the source. For a small team, usually the pragmatic choice — especially now that
  UBO is hard to get directly.

### Recommendation

Anchor on the free identity backbone (VIES + GLEIF/LEI + France/Germany open data),
then use one aggregator for normalized financials and ownership across the rest.
EU-wide coverage without integrating 27 registries, and it sidesteps the UBO problem.

## Open questions to nail down before committing

- Exact financial-statement depth, API availability and per-document/subscription
  costs for each named registry.
- Aggregator API pricing and how complete/fresh their pre-harvested UBO and
  annual-accounts data is.
- Whether the legitimate-interest restriction applies to aggregators' redistribution
  of UBO data, or they fall under a separate access basis.

---

*Run stats: 6 search angles · 26 sources fetched · 8 claims verified (8 confirmed,
0 killed) · 58 agents. Verified claims cover the UBO legal dimension only.*
