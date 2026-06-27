# Sourcing Complementary Enrichment Data for EU Biofuel Companies

> Deep-research report. Generated 2026-06-27.
> Question: where to source complementary enrichment data (financial, operational,
> contact, ESG/reputation) about EU bioenergy/biofuel companies that already hold
> sustainability certifications (ISCC, REDcert, INS, 2BSvs, EU UDB), for
> due-diligence and business-development use cases.

## Bottom line

Use **free official sources** (VIES, ISCC database, fraud signals) for verification
and screening, and **paid commercial providers** for financial, operational and
contact enrichment. Key strategic finding: **certification data alone is a weak
trust signal** — which is exactly why layering complementary data adds real value.

---

## ✅ Free sources (verification & fraud screening)

### 1. VIES (EU VAT verification) — *Financial/corporate identity*
- Free EC service, all 27 EU states + Northern Ireland, live from national VAT databases
- Returns VAT validity + (in most states) registered name & address
- Free SOAP/XML API for automated enrichment — no native bulk method, rate-limited
- ⚠️ Germany & Spain return validity only (no name/address) — needs a separate step
- **Cost: free | Update: real-time**

### 2. ISCC Certificate Database — *Operational metadata + fraud screening*
- Free, searchable; CSV bulk export of valid certificates (good for table-level joins)
- Facility type (refinery, trader, oil mill) + location — but no financial/capacity/contact fields
- Dedicated Withdrawn / Suspended / Fake / Excluded lists = a ready-made fraud blacklist
- 2024: 97 certificates withdrawn, 130 companies suspended, 78 excluded
- **Cost: free | Update: immediate on suspension/withdrawal**

### 3. Transport & Environment fraud signals — *ESG/reputation/red flags*
- Mass-balance accounting is falsifiable; UCO feedstock often self-declared
  (physical audit only above 5 t/month)
- Only ~9% of certified UCO collection points in China/Malaysia/Indonesia had origin audited
- Malaysia exports ~3× more UCO than it collects
- ~1.8M tons of fraudulent ISCC-certified POME estimated to have entered the EU in 2023
- ⚠️ T&E is an advocacy NGO, but the load-bearing facts were independently corroborated

---

## 💰 Paid sources (financial, operational, commercial enrichment)

### 4. ImportGenius — *Trade/contact intelligence*
- Bill-of-lading import/export records, 25+ countries, JSON API
- **$125–$899/user/month** | US daily, non-US weekly
- ⚠️ EU customs data is a paid add-on — core product is US-centric

### 5. S&P Global Panjiva — *Trade + corporate enrichment*
- 2B+ shipment records, 9M+ companies; enriched with D&B/ZoomInfo/Kompass data
- ⚠️ No EU member states in its trade-data coverage (EU customs confidentiality)
- Enterprise pricing (not disclosed)

---

## ⚠️ The recurring EU blind spot

Both trade-data providers are structurally limited for EU customs data due to EU
confidentiality rules. There is no clean EU bill-of-lading equivalent. This is the
hardest of the four categories to fill for an EU-focused app.

## 🔍 Gaps the run could not independently verify (still your best routes)

- **Financial/corporate:** GLEIF (LEI), national registries (Italy InfoCamere,
  German Handelsregister, French Infogreffe via BRIS), Orbis/Bureau van Dijk,
  Dun & Bradstreet, Creditsafe — see report 02 for the follow-up.
- **EU Union Database (UDB):** unclear what it exposes to third parties for due diligence.
- **Other scheme databases:** whether REDcert, the Italian INS, and 2BSvs have public
  lookups + fraud lists comparable to ISCC's.
- **ESG-score providers:** coverage of typically private/SME biofuel firms is uncertain.

## ❌ Claims checked and refuted (do not rely on)

- "~1/3 of UCO entering the EU is fake" — refuted (0-3)
- "VIES has no REST/JSON API" — refuted (overstated; SOAP confirmed)

---

*Run stats: 5 search angles · 22 sources fetched · 24 claims extracted ·
22 confirmed, 2 killed · 101 agents.*
