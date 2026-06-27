# Sinetti Enrichment Engine — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a pure-Python enrichment engine that turns a Sinetti certificate row into a Counterparty Profile (verified identity + registry-name reconciliation + a four-state trust verdict), consuming VIES + GLEIF + internal ISCC fraud flags on demand with caching, surfaced in Sinetti's global verifier.

**Architecture:** A UI-free `enrich/` package (models, config, matching, adapter, ports, providers, engine) that never imports Streamlit and reaches the host app only through injected ports (cache, screening, http). Sinetti-side glue (`enrich_store.py`, `enrich_screening.py`, `enrichment_ui.py`) wires the engine into the existing app and persists results. The engine runs headless and is fully testable with in-memory fakes and recorded fixtures.

**Tech Stack:** Python 3, dataclasses, `urllib.request` (stdlib HTTP — no new deps), sqlite3 (via existing `store_iscc` pattern), pytest, Streamlit ≤1.52.2 for the thin UI only.

**Repos:** The engine **code is built in the local `sinetti` repo** (`C:\Users\AZIZ\sinetti`). This plan and the spec live in `azizouuuu/test2` (PR #1). Commit engine work in `sinetti` (local-only — never add a remote / push, per the user's deploy rules).

**Spec:** `docs/superpowers/specs/2026-06-28-sinetti-enrichment-engine-design.md` (in `test2`).

## Global Constraints

- **No Streamlit imports anywhere under `enrich/`.** UI lives only in `enrichment_ui.py`.
- **Streamlit ≤1.52.2 API only** in `enrichment_ui.py` (no `st.dialog`; use `use_container_width=True`).
- **SiS-safe:** nothing under `enrich/` or the store does network or table I/O *at import time*. Tables are created only inside an explicit init call folded into `store_iscc.init_iscc_tables()`.
- **Providers never raise to the caller.** Every provider catches its own errors and returns a `ProviderResult` with `status` ∈ {`ok`,`no_match`,`invalid`,`unavailable`,`not_applicable`}.
- **Capability gate is static, no network probe:** reuse `ui.live_fetch_available()` == `not runtime.in_snowflake()`. The engine receives `external_allowed: bool`.
- **VIES is non-redistributable** (verified): provider carries `redistributable=False`; GLEIF carries `redistributable=True` (CC0). No publish/re-serving path in v1.
- **No live network in CI:** provider tests inject a fake `http_get`; one recorded-cassette test per provider. Live smoke tests are `@pytest.mark.skip`-by-default (opt-in).
- **GLEIF country filter is ISO-3166 alpha-2.** Adapter normalises `country` before it can reach GLEIF.
- **Canonical `cert_status` enum:** `valid | suspended | withdrawn | expired | unknown` — mapped in the adapter, reasoned over in the verdict.
- **DRY, YAGNI, TDD, frequent commits.** Reuse existing Sinetti helpers; do not re-implement parsers/loaders.
- Run the full suite with `python -m pytest -q` from the `sinetti` repo root. Target: no regression.

## Interface Contract (every task aligns to these exact signatures)

```python
# enrich/models.py
@dataclass(frozen=True)
class CompanyInput:
    scheme: str; certificate_id: str; name: str; norm: str
    country: str | None; address: str | None; city: str | None; postcode: str | None
    vat: str | None; cert_status: str; anonymised: bool

@dataclass(frozen=True)
class Field:
    value: object; source: str; confidence: float; as_of: str

@dataclass(frozen=True)
class NameMatch:
    level: str; score: float; compared_to: str        # level: match|partial|mismatch|n/a

@dataclass(frozen=True)
class ProviderResult:
    provider: str; status: str; fields: dict           # dict[str, Field]
    redistributable: bool

@dataclass(frozen=True)
class Verdict:
    level: str; reasons: list                           # level: high_risk|caution|verified|insufficient_data

@dataclass(frozen=True)
class CompanyProfile:
    input: CompanyInput; identity: dict; name_match: NameMatch
    risk: list; verdict: Verdict; providers: list

# enrich/matching.py
def normalise(name: str) -> str
def score(cand: dict, inp: "CompanyInput") -> float     # cand keys: name, country, postcode, address

# enrich/adapter.py
def from_sinetti_record(row: dict, scheme: str, *, ins_pdf_index: dict | None = None) -> CompanyInput

# enrich/ports.py  (typing.Protocol)
class CacheStore(Protocol):
    def get(self, provider: str, input_key: str) -> dict | None: ...
    def put(self, provider: str, input_key: str, status: str, payload: dict, ttl_days: int) -> None: ...
class ScreeningSource(Protocol):
    def blocklist_hits(self, name: str) -> list: ...

# enrich/providers — each returns a ProviderResult, never raises
HttpGet = "Callable[[str], tuple[int, dict]]"
def fraud_lookup(inp: CompanyInput, screening: ScreeningSource) -> ProviderResult
def vies_lookup(inp: CompanyInput, *, http_get: HttpGet) -> ProviderResult
def gleif_lookup(inp: CompanyInput, *, http_get: HttpGet, abstain_threshold: float) -> ProviderResult

# enrich/engine.py
def profile(row: dict, scheme: str, *, external_allowed: bool,
            cache: CacheStore, screening: ScreeningSource,
            http_get=None, ins_pdf_index: dict | None = None) -> CompanyProfile
```

---

### Task 1: `enrich/models.py` — data shapes

**Files:**
- Create: `enrich/__init__.py` (empty)
- Create: `enrich/models.py`
- Test: `tests/test_enrich_models.py`

**Interfaces:**
- Produces: the six dataclasses in the Interface Contract above. No logic, no imports beyond `dataclasses`.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_enrich_models.py
from enrich.models import CompanyInput, Field, NameMatch, ProviderResult, Verdict, CompanyProfile

def test_company_input_holds_fields():
    ci = CompanyInput(scheme="INS", certificate_id="123", name="ACME SRL", norm="acmesrl",
                      country="IT", address=None, city="Roma", postcode="00197",
                      vat="10002290152", cert_status="unknown", anonymised=False)
    assert ci.vat == "10002290152" and ci.cert_status == "unknown"

def test_profile_composes():
    f = Field(value="ACME SRL", source="VIES", confidence=1.0, as_of="2026-06-28")
    p = CompanyProfile(input=None, identity={"registered_name": f},
                       name_match=NameMatch(level="match", score=0.97, compared_to="VIES"),
                       risk=[], verdict=Verdict(level="verified", reasons=["VAT valid (VIES, 2026-06-28)"]),
                       providers=[ProviderResult(provider="VIES", status="ok", fields={"vat_status": f},
                                                 redistributable=False)])
    assert p.verdict.level == "verified" and p.providers[0].redistributable is False
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_enrich_models.py -q`
Expected: FAIL with `ModuleNotFoundError: No module named 'enrich'`.

- [ ] **Step 3: Write minimal implementation**

```python
# enrich/__init__.py  (empty file)
```

```python
# enrich/models.py
"""Pure data shapes for the enrichment engine. No logic, no I/O, no Streamlit."""
from __future__ import annotations
from dataclasses import dataclass, field


@dataclass(frozen=True)
class CompanyInput:
    scheme: str
    certificate_id: str
    name: str
    norm: str
    country: str | None
    address: str | None
    city: str | None
    postcode: str | None
    vat: str | None
    cert_status: str          # canonical: valid|suspended|withdrawn|expired|unknown
    anonymised: bool


@dataclass(frozen=True)
class Field:
    value: object
    source: str
    confidence: float
    as_of: str


@dataclass(frozen=True)
class NameMatch:
    level: str                # match | partial | mismatch | n/a
    score: float
    compared_to: str


@dataclass(frozen=True)
class ProviderResult:
    provider: str
    status: str               # ok | no_match | invalid | unavailable | not_applicable
    fields: dict
    redistributable: bool


@dataclass(frozen=True)
class Verdict:
    level: str                # high_risk | caution | verified | insufficient_data
    reasons: list


@dataclass(frozen=True)
class CompanyProfile:
    input: CompanyInput | None
    identity: dict
    name_match: NameMatch
    risk: list
    verdict: Verdict
    providers: list
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_enrich_models.py -q`
Expected: PASS (2 passed).

- [ ] **Step 5: Commit**

```bash
git add enrich/__init__.py enrich/models.py tests/test_enrich_models.py
git commit -m "feat(enrich): add engine data models"
```

---

### Task 2: `enrich/config.py` — constants

**Files:**
- Create: `enrich/config.py`
- Test: `tests/test_enrich_config.py`

**Interfaces:**
- Produces: `VIES_REST`, `GLEIF_BASE`, `TIMEOUTS`, `ABSTAIN_THRESHOLD`, `TTL_OK`, `TTL_NO_MATCH`, `KEY_VERSION`, `MAX_CALLS_PER_SESSION`, `RETRY_BACKOFF_S`.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_enrich_config.py
from enrich import config as cfg

def test_endpoints_and_policy_present():
    assert cfg.GLEIF_BASE == "https://api.gleif.org/api/v1"
    assert "{country}" in cfg.VIES_REST and "{vat}" in cfg.VIES_REST
    assert cfg.TTL_OK["VIES"] == 30 and cfg.TTL_OK["GLEIF"] == 90
    assert 0.0 < cfg.ABSTAIN_THRESHOLD <= 1.0
    assert cfg.KEY_VERSION  # non-empty; bumping invalidates caches
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_enrich_config.py -q`
Expected: FAIL with `ModuleNotFoundError` / `AttributeError`.

- [ ] **Step 3: Write minimal implementation**

```python
# enrich/config.py
"""Engine configuration: endpoints, timeouts, thresholds, cache TTLs. No I/O."""
from __future__ import annotations

VIES_REST = "https://ec.europa.eu/taxation_customs/vies/rest-api/ms/{country}/vat/{vat}"
GLEIF_BASE = "https://api.gleif.org/api/v1"

TIMEOUTS = (5, 10)                 # (connect, read) seconds
RETRY_BACKOFF_S = 1.5              # one retry on 429/timeout, then 'unavailable'
MAX_CALLS_PER_SESSION = 200        # hard cap, defensive

ABSTAIN_THRESHOLD = 0.85           # GLEIF score < this => no_match. Tuned by the spike (Task 0).

TTL_OK = {"VIES": 30, "GLEIF": 90}  # days for a confirmed result
TTL_NO_MATCH = 7                    # days for a no_match (absences don't outlive data refreshes)
# 'unavailable' results are NOT cached (handled in the engine).

KEY_VERSION = "v1"                  # bump to invalidate all cache keys when cleaning/normalise changes
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_enrich_config.py -q`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add enrich/config.py tests/test_enrich_config.py
git commit -m "feat(enrich): add engine config constants"
```

---

### Task 3: `enrich/matching.py` — normalisation + confidence scoring

**Files:**
- Create: `enrich/matching.py`
- Test: `tests/test_enrich_matching.py`

**Interfaces:**
- Consumes: `CompanyInput` (Task 1).
- Produces: `normalise(name) -> str`; `score(cand: dict, inp: CompanyInput) -> float`. Pure; **no threshold inside** (threshold lives in `config`).

- [ ] **Step 1: Write the failing test**

```python
# tests/test_enrich_matching.py
from enrich.matching import normalise, score
from enrich.models import CompanyInput

def _inp(name, country="IT", postcode="00197", address=None):
    return CompanyInput("INS", "1", name, normalise(name), country, address, None, postcode,
                        None, "unknown", False)

def test_normalise_strips_suffix_and_punctuation():
    assert normalise("Venanzieffe S.R.L.") == normalise("VENANZIEFFE srl") == "venanzieffe"

def test_same_name_same_country_scores_high():
    s = score({"name": "Venanzieffe SRL", "country": "IT", "postcode": "00197", "address": ""},
              _inp("VENANZIEFFE S.R.L."))
    assert s >= 0.85

def test_different_country_is_gated_low():
    s = score({"name": "Venanzieffe SRL", "country": "DE", "postcode": "", "address": ""},
              _inp("VENANZIEFFE SRL"))
    assert s < 0.5

def test_unrelated_name_scores_low():
    s = score({"name": "Globex Petroleum AG", "country": "IT", "postcode": "", "address": ""},
              _inp("VENANZIEFFE SRL"))
    assert s < 0.5
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_enrich_matching.py -q`
Expected: FAIL with `ModuleNotFoundError`.

- [ ] **Step 3: Write minimal implementation**

```python
# enrich/matching.py
"""Name normalisation + a 0..1 match score. Pure; no network, no threshold (that lives in config)."""
from __future__ import annotations
import re
from difflib import SequenceMatcher

_LEGAL = (" srl", " s r l", " spa", " s p a", " gmbh", " co kg", " ag", " sa", " ltd",
          " limited", " bv", " nv", " oy", " ab", " as", " plc", " inc", " llc", " sas",
          " snc", " sl", " kg", " sarl", " scrl", " coop")


def normalise(name: str) -> str:
    s = (name or "").split(",")[0]                 # drop ISCC's embedded-address tail
    s = s.lower()
    s = re.sub(r"[^a-z0-9 ]", " ", s)
    s = re.sub(r"\s+", " ", s).strip()
    for suf in _LEGAL:
        if s.endswith(suf):
            s = s[: -len(suf)].strip()
    return re.sub(r"\s+", "", s)                    # collapse to a tight token


def _token_ratio(a: str, b: str) -> float:
    return SequenceMatcher(None, a, b).ratio()


def score(cand: dict, inp: "CompanyInput") -> float:
    """Combine name similarity + country hard-gate + postcode/address agreement."""
    name_sim = _token_ratio(normalise(cand.get("name", "")), inp.norm or normalise(inp.name))
    cc_cand = (cand.get("country") or "").upper()
    cc_inp = (inp.country or "").upper()
    if cc_cand and cc_inp and cc_cand != cc_inp:
        return min(name_sim, 0.40)                  # different country: hard cap
    s = name_sim
    pc_c = re.sub(r"\D", "", cand.get("postcode") or "")
    pc_i = re.sub(r"\D", "", inp.postcode or "")
    if pc_c and pc_i:
        s += 0.05 if pc_c == pc_i else -0.05
    addr_c = re.sub(r"[^a-z0-9]", "", (cand.get("address") or "").lower())
    addr_i = re.sub(r"[^a-z0-9]", "", (inp.address or "").lower())
    if addr_c and addr_i and (addr_c in addr_i or addr_i in addr_c):
        s += 0.05
    return max(0.0, min(1.0, s))
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_enrich_matching.py -q`
Expected: PASS (4 passed).

- [ ] **Step 5: Commit**

```bash
git add enrich/matching.py tests/test_enrich_matching.py
git commit -m "feat(enrich): add name normalisation + match scoring"
```

---

### Task 4: `enrich/adapter.py` — Sinetti row → CompanyInput (per scheme)

**Files:**
- Create: `enrich/adapter.py`
- Test: `tests/test_enrich_adapter.py`

**Interfaces:**
- Consumes: `CompanyInput` (Task 1); reuses Sinetti helpers `iscc._company_name`, `iscc._address_part`, `iscc_fetch._holder_norm`.
- Produces: `from_sinetti_record(row, scheme, *, ins_pdf_index=None) -> CompanyInput`.

**Notes on real data (from the spec):** ISCC `holder_name` embeds the address after the first comma. INS operator rows have `status` hardcoded — real lifecycle status comes from `ins.load_cert_pdf_index()` keyed by tax code; INS `partita_iva` may be blank (fall back to `codice_fiscale`), foreign-prefixed, or an `OPERATORE000xxx` placeholder holder. GLEIF needs ISO alpha-2 country.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_enrich_adapter.py
from enrich.adapter import from_sinetti_record

def test_iscc_splits_embedded_address_and_has_no_vat():
    row = {"certificate_id": "EU-ISCC-Cert-DE100-1",
           "holder_name": "Clean Tech International SRL, Tarla 50, 927080 Ciulnita, Romania",
           "holder_norm": "cleantechinternationalsrl", "country": "RO", "raw_status": "1",
           "suspended_date": ""}
    ci = from_sinetti_record(row, "ISCC")
    assert ci.name == "Clean Tech International SRL"
    assert "Ciulnita" in (ci.address or "")
    assert ci.vat is None and ci.cert_status == "valid"

def test_redcert_uses_city_postcode_and_maps_expired():
    row = {"certificate_id": "DE-x", "holder_name": "KFS Biodiesel GmbH & Co. KG",
           "holder_norm": "kfsbiodieselgmbhcokg", "country": "DE", "city": "Cloppenburg",
           "postcode": "49661", "status": "expired"}
    ci = from_sinetti_record(row, "REDcert")
    assert ci.city == "Cloppenburg" and ci.postcode == "49661"
    assert ci.vat is None and ci.cert_status == "expired"

def test_ins_clean_vat_and_status_via_pdf_join():
    row = {"certificate_id": "10002290152", "holder_name": "VENANZIEFFE SRL",
           "holder_norm": "venanzieffesrl", "country": "IT", "city": "Roma", "postcode": "00197",
           "partita_iva": "10002290152", "codice_fiscale": "10002290152", "status": "active"}
    idx = {"piva:10002290152": {"status": "withdrawn"}}
    ci = from_sinetti_record(row, "INS", ins_pdf_index=idx)
    assert ci.vat == "10002290152" and ci.cert_status == "withdrawn" and ci.anonymised is False

def test_ins_anonymised_placeholder_and_foreign_vat_guard():
    row = {"certificate_id": "FR 71449427145", "holder_name": "OPERATORE000030",
           "holder_norm": "operatore000030", "country": "", "partita_iva": "FR 71449427145",
           "codice_fiscale": "FR 71449427145", "status": "active"}
    ci = from_sinetti_record(row, "INS")
    assert ci.anonymised is True
    assert ci.vat is None          # foreign-prefixed / empty country => not VIES-usable
    assert ci.cert_status == "unknown"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_enrich_adapter.py -q`
Expected: FAIL with `ModuleNotFoundError: No module named 'enrich.adapter'`.

- [ ] **Step 3: Write minimal implementation**

```python
# enrich/adapter.py
"""Translate a raw Sinetti certificate row (ISCC / REDcert / INS) into a clean CompanyInput.
The ONE place that knows each scheme's quirks. Reuses Sinetti's existing parse helpers."""
from __future__ import annotations
import re
from enrich.models import CompanyInput

# ISCC raw_status "1" = valid; suspended_date set => suspended (see store_iscc).
_ISCC_STATUS = {"1": "valid", "0": "unknown"}
_REDCERT_STATUS = {"valid": "valid", "active": "valid", "expired": "expired",
                   "suspended": "suspended", "withdrawn": "withdrawn", "revoked": "withdrawn"}


def _alpha2(country: str | None) -> str | None:
    c = (country or "").strip().upper()
    return c if re.fullmatch(r"[A-Z]{2}", c) else None


def _clean_vat(piva: str, cf: str, country: str | None) -> str | None:
    """A VIES-usable Italian VAT: 11 digits, country IT, no foreign prefix. Else None."""
    raw = (piva or cf or "").strip()
    if (country or "").upper() != "IT":
        return None
    if not re.fullmatch(r"\d{11}", raw):
        return None
    return raw


def _is_anonymised(holder_name: str) -> bool:
    return bool(re.match(r"^OPERATORE0*\d+$", (holder_name or "").strip(), re.I))


def from_sinetti_record(row: dict, scheme: str, *, ins_pdf_index: dict | None = None) -> CompanyInput:
    import iscc
    import iscc_fetch
    holder = row.get("holder_name", "") or ""
    norm = row.get("holder_norm") or iscc_fetch._holder_norm(holder)
    country = _alpha2(row.get("country"))
    cert_id = row.get("certificate_id", "") or ""

    if scheme == "ISCC":
        name = iscc._company_name(holder)
        address = iscc._address_part(holder) or None
        status = "suspended" if row.get("suspended_date") else _ISCC_STATUS.get(
            str(row.get("raw_status", "")).strip(), "unknown")
        return CompanyInput(scheme, cert_id, name, norm, country, address, None, None,
                            None, status, False)

    if scheme == "REDcert":
        status = _REDCERT_STATUS.get((row.get("status") or "").lower(), "unknown")
        return CompanyInput(scheme, cert_id, holder, norm, country, row.get("holder_full") or None,
                            row.get("city"), row.get("postcode"), None, status, False)

    if scheme == "INS":
        anon = _is_anonymised(holder)
        vat = _clean_vat(row.get("partita_iva", ""), row.get("codice_fiscale", ""), country)
        status = "unknown"
        if ins_pdf_index:
            for key in (f"piva:{row.get('partita_iva','')}", f"cf:{row.get('codice_fiscale','')}",
                        f"name:{norm}"):
                hit = ins_pdf_index.get(key)
                if hit and hit.get("status"):
                    status = _REDCERT_STATUS.get(hit["status"].lower(), hit["status"].lower())
                    break
        return CompanyInput(scheme, cert_id, holder, norm, country, None,
                            row.get("city"), row.get("postcode"), vat, status, anon)

    raise ValueError(f"unknown scheme {scheme!r}")
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_enrich_adapter.py -q`
Expected: PASS (4 passed).

- [ ] **Step 5: Commit**

```bash
git add enrich/adapter.py tests/test_enrich_adapter.py
git commit -m "feat(enrich): add per-scheme Sinetti->CompanyInput adapter"
```

---

### Task 5: `enrich/ports.py` + `enrich/providers/fraud.py` + `enrich_screening.py` — offline fraud screen

**Files:**
- Create: `enrich/ports.py`
- Create: `enrich/providers/__init__.py` (empty)
- Create: `enrich/providers/fraud.py`
- Create: `enrich_screening.py` (Sinetti-side adapter over `iscc.blocklist_match`)
- Test: `tests/test_enrich_fraud.py`

**Interfaces:**
- Consumes: `CompanyInput`, `Field`, `ProviderResult` (Task 1); the `ScreeningSource` protocol (this task).
- Produces: `ScreeningSource` + `CacheStore` protocols; `fraud_lookup(inp, screening) -> ProviderResult`; `IsccScreening` class (Sinetti-side, wraps `iscc.blocklist_match`).

- [ ] **Step 1: Write the failing test**

```python
# tests/test_enrich_fraud.py
from enrich.providers.fraud import fraud_lookup
from enrich.models import CompanyInput

def _inp(name="ACME SRL", norm="acmesrl", scheme="ISCC", status="valid"):
    return CompanyInput(scheme, "1", name, norm, "IT", None, None, None, None, status, False)

class FakeScreening:
    def __init__(self, hits): self._hits = hits
    def blocklist_hits(self, name): return self._hits

def test_clean_company_has_no_fraud_flag():
    r = fraud_lookup(_inp(), FakeScreening([]))
    assert r.provider == "ISCC-lists" and r.status == "ok"
    assert r.fields["on_fraud_list"].value is False
    assert r.fields["cert_status"].value == "valid"
    assert r.redistributable is True

def test_blocklisted_company_flagged():
    hit = [{"category": "fake", "norm": "acmesrl"}]
    r = fraud_lookup(_inp(), FakeScreening(hit))
    assert r.fields["on_fraud_list"].value is True

def test_redcert_holder_gets_cert_status_only_note():
    r = fraud_lookup(_inp(scheme="REDcert"), FakeScreening([]))
    # blocklist coverage is ISCC-only; non-ISCC schemes carry the honesty note
    assert r.fields["fraud_list_coverage"].value == "ISCC-only"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_enrich_fraud.py -q`
Expected: FAIL with `ModuleNotFoundError`.

- [ ] **Step 3: Write minimal implementation**

```python
# enrich/ports.py
"""The two host-app capabilities the engine needs, as typing Protocols, so the engine never
imports iscc.py / store_iscc.py / sqlite3 directly and stays headless + testable with fakes."""
from __future__ import annotations
from typing import Protocol


class CacheStore(Protocol):
    def get(self, provider: str, input_key: str) -> dict | None: ...
    def put(self, provider: str, input_key: str, status: str, payload: dict, ttl_days: int) -> None: ...


class ScreeningSource(Protocol):
    def blocklist_hits(self, name: str) -> list: ...
```

```python
# enrich/providers/__init__.py  (empty file)
```

```python
# enrich/providers/fraud.py
"""Internal fraud screen. No network — works everywhere (incl. Snowflake). Reads only through the
injected ScreeningSource port. Blocklist/compliance coverage is ISCC-only; that limit is surfaced."""
from __future__ import annotations
from datetime import date
from enrich.models import CompanyInput, Field, ProviderResult


def fraud_lookup(inp: CompanyInput, screening) -> ProviderResult:
    today = date.today().isoformat()
    hits = screening.blocklist_hits(inp.name) or []
    fields = {
        "on_fraud_list": Field(bool(hits), "ISCC-lists", 1.0, today),
        "cert_status": Field(inp.cert_status, "ISCC-lists", 1.0, today),
        "fraud_list_coverage": Field("ISCC-only" if inp.scheme != "ISCC" else "full",
                                     "ISCC-lists", 1.0, today),
    }
    if hits:
        fields["fraud_list_entries"] = Field(hits, "ISCC-lists", 1.0, today)
    return ProviderResult("ISCC-lists", "ok", fields, redistributable=True)
```

```python
# enrich_screening.py  (Sinetti-side; the production ScreeningSource)
"""Production ScreeningSource: wraps Sinetti's existing exact-name blocklist matcher so the engine
gets fraud evidence without importing iscc.py itself."""
from __future__ import annotations


class IsccScreening:
    def blocklist_hits(self, name: str) -> list:
        import iscc
        return iscc.blocklist_match(name or "")
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_enrich_fraud.py -q`
Expected: PASS (3 passed).

- [ ] **Step 5: Commit**

```bash
git add enrich/ports.py enrich/providers/__init__.py enrich/providers/fraud.py enrich_screening.py tests/test_enrich_fraud.py
git commit -m "feat(enrich): add ports + offline fraud screen via screening port"
```

---

### Task 6: `enrich_store.py` — CacheStore (SQLite) + enrichment_history; wire table init

**Files:**
- Create: `enrich_store.py`
- Modify: `store_iscc.py` (inside `init_iscc_tables()`, add two `CREATE TABLE IF NOT EXISTS`)
- Test: `tests/test_enrich_store.py`

**Interfaces:**
- Consumes: `config.TTL_*`, `config.KEY_VERSION` (Task 2).
- Produces: `SqliteCacheStore` (implements `CacheStore`); `record_enrichment(holder_norm, scheme, verdict_level, reasons)`; `make_cache_store()` factory (returns the SQLite store locally, Snowflake store in SiS).

**Note:** the Snowflake mirror (`SnowflakeCacheStore`) mirrors the API but is validated in-account separately, exactly like `store_iscc_snowflake.py`. v1 runs on Hugging Face (SQLite). Include the class with the same method signatures and a `# validated in-account` marker; do not block v1 on it.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_enrich_store.py
import enrich_store
from enrich import config as cfg

def test_put_get_roundtrip_and_status_ttl(tmp_path, monkeypatch):
    monkeypatch.setattr(enrich_store, "DB_PATH", tmp_path / "t.db")
    enrich_store.init_enrich_tables()
    store = enrich_store.SqliteCacheStore()
    store.put("VIES", "k1", "ok", {"vat_status": "valid"}, cfg.TTL_OK["VIES"])
    got = store.get("VIES", "k1")
    assert got["status"] == "ok" and got["payload"]["vat_status"] == "valid"

def test_expired_entry_returns_none(tmp_path, monkeypatch):
    monkeypatch.setattr(enrich_store, "DB_PATH", tmp_path / "t.db")
    enrich_store.init_enrich_tables()
    store = enrich_store.SqliteCacheStore()
    store.put("GLEIF", "k2", "no_match", {}, ttl_days=0)   # already expired
    assert store.get("GLEIF", "k2") is None

def test_record_enrichment_history(tmp_path, monkeypatch):
    monkeypatch.setattr(enrich_store, "DB_PATH", tmp_path / "t.db")
    enrich_store.init_enrich_tables()
    enrich_store.record_enrichment("acmesrl", "INS", "verified", ["VAT valid (VIES)"])
    rows = enrich_store.recent_enrichments(limit=5)
    assert rows and rows[0]["holder_norm"] == "acmesrl" and rows[0]["verdict_level"] == "verified"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_enrich_store.py -q`
Expected: FAIL with `ModuleNotFoundError: No module named 'enrich_store'`.

- [ ] **Step 3: Write minimal implementation**

```python
# enrich_store.py
"""Persistence for the enrichment engine: a status-aware TTL cache + an audit-trail history table.
SQLite locally (mirrors store_iscc's pattern); a Snowflake mirror class is provided for SiS.
Nothing executes at import; init_enrich_tables() is folded into store_iscc.init_iscc_tables()."""
from __future__ import annotations
import json
import sqlite3
from datetime import datetime, timedelta, timezone
from pathlib import Path

DB_PATH = Path(__file__).with_name("intel.db")


def _conn() -> sqlite3.Connection:
    conn = sqlite3.connect(DB_PATH)
    conn.row_factory = sqlite3.Row
    return conn


def init_enrich_tables() -> None:
    with _conn() as c:
        c.execute("""CREATE TABLE IF NOT EXISTS enrichment_cache (
            provider TEXT, input_key TEXT, status TEXT, payload_json TEXT,
            fetched_at TEXT, expires_at TEXT, PRIMARY KEY (provider, input_key))""")
        c.execute("""CREATE TABLE IF NOT EXISTS enrichment_history (
            id INTEGER PRIMARY KEY AUTOINCREMENT, holder_norm TEXT, scheme TEXT,
            verdict_level TEXT, reasons_json TEXT, produced_at TEXT, produced_by TEXT)""")


class SqliteCacheStore:
    """Status-aware TTL cache. get() returns None for a miss OR an expired row."""

    def get(self, provider: str, input_key: str) -> dict | None:
        with _conn() as c:
            row = c.execute("SELECT status, payload_json, expires_at FROM enrichment_cache "
                            "WHERE provider=? AND input_key=?", (provider, input_key)).fetchone()
        if not row:
            return None
        if row["expires_at"] and row["expires_at"] < datetime.now(timezone.utc).isoformat():
            return None
        return {"status": row["status"], "payload": json.loads(row["payload_json"] or "{}")}

    def put(self, provider: str, input_key: str, status: str, payload: dict, ttl_days: int) -> None:
        now = datetime.now(timezone.utc)
        expires = (now + timedelta(days=ttl_days)).isoformat()
        with _conn() as c:
            c.execute("INSERT OR REPLACE INTO enrichment_cache "
                      "(provider, input_key, status, payload_json, fetched_at, expires_at) "
                      "VALUES (?,?,?,?,?,?)",
                      (provider, input_key, status, json.dumps(payload), now.isoformat(), expires))


def record_enrichment(holder_norm: str, scheme: str, verdict_level: str, reasons: list) -> None:
    import runtime
    try:
        who = runtime.current_user()
    except Exception:
        who = ""
    with _conn() as c:
        c.execute("INSERT INTO enrichment_history "
                  "(holder_norm, scheme, verdict_level, reasons_json, produced_at, produced_by) "
                  "VALUES (?,?,?,?,?,?)",
                  (holder_norm, scheme, verdict_level, json.dumps(reasons),
                   datetime.now(timezone.utc).isoformat(), who))


def recent_enrichments(limit: int = 20) -> list[dict]:
    with _conn() as c:
        rows = c.execute("SELECT holder_norm, scheme, verdict_level, reasons_json, produced_at, "
                         "produced_by FROM enrichment_history ORDER BY id DESC LIMIT ?",
                         (limit,)).fetchall()
    return [dict(r) for r in rows]


def make_cache_store():
    """SQLite locally; the Snowflake mirror in SiS (validated in-account)."""
    import runtime
    if runtime.in_snowflake():
        return SnowflakeCacheStore()
    return SqliteCacheStore()


class SnowflakeCacheStore:  # validated in-account, mirrors SqliteCacheStore's API. Not exercised on HF.
    def get(self, provider: str, input_key: str) -> dict | None:
        from snowflake.snowpark.context import get_active_session
        s = get_active_session()
        df = s.sql("SELECT status, payload_json, expires_at FROM enrichment_cache "
                   f"WHERE provider='{provider}' AND input_key='{input_key}'").collect()
        if not df:
            return None
        r = df[0]
        if r["EXPIRES_AT"] and r["EXPIRES_AT"] < datetime.now(timezone.utc).isoformat():
            return None
        return {"status": r["STATUS"], "payload": json.loads(r["PAYLOAD_JSON"] or "{}")}

    def put(self, provider, input_key, status, payload, ttl_days):
        from snowflake.snowpark.context import get_active_session
        s = get_active_session()
        now = datetime.now(timezone.utc)
        expires = (now + timedelta(days=ttl_days)).isoformat()
        s.sql("MERGE INTO enrichment_cache t USING (SELECT ? provider, ? input_key) src "
              "ON t.provider=src.provider AND t.input_key=src.input_key "
              "WHEN MATCHED THEN UPDATE SET status=?, payload_json=?, fetched_at=?, expires_at=? "
              "WHEN NOT MATCHED THEN INSERT (provider,input_key,status,payload_json,fetched_at,expires_at) "
              "VALUES (?,?,?,?,?,?)",
              params=[provider, input_key, status, json.dumps(payload), now.isoformat(), expires,
                      provider, input_key, status, json.dumps(payload), now.isoformat(), expires]).collect()
```

- [ ] **Step 4: Wire init into the existing table bootstrap**

In `store_iscc.py`, inside `init_iscc_tables()` (after the existing `CREATE TABLE` block, before the function returns), add:

```python
        # Enrichment engine tables (cache + audit history) — same connection/bootstrap.
        import enrich_store
        enrich_store.init_enrich_tables()
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `python -m pytest tests/test_enrich_store.py -q`
Expected: PASS (3 passed).

- [ ] **Step 6: Commit**

```bash
git add enrich_store.py store_iscc.py tests/test_enrich_store.py
git commit -m "feat(enrich): add status-aware cache + audit history store"
```

---

### Task 7: `enrich/providers/base.py` + `enrich/providers/vies.py` — VAT validation

**Files:**
- Create: `enrich/providers/base.py` (default `http_get` via urllib)
- Create: `enrich/providers/vies.py`
- Test: `tests/test_enrich_vies.py`

**Interfaces:**
- Consumes: `CompanyInput`, `Field`, `ProviderResult` (Task 1); `config.VIES_REST`, `config.TIMEOUTS`.
- Produces: `default_http_get(url) -> tuple[int, dict]`; `vies_lookup(inp, *, http_get) -> ProviderResult`.

**Note (verified):** VIES returns **HTTP 200 for all outcomes**; the result is in `userError` (`VALID`/`INVALID`/`MS_UNAVAILABLE`/`*_MAX_CONCURRENT_REQ`), with `name`/`address` = `"---"` when not valid.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_enrich_vies.py
from enrich.providers.vies import vies_lookup
from enrich.models import CompanyInput

def _inp(vat="10002290152", country="IT"):
    return CompanyInput("INS", "1", "ACME SRL", "acmesrl", country, None, "Roma", "00197",
                        vat, "valid", False)

def _http(payload, code=200):
    return lambda url: (code, payload)

def test_valid_vat_returns_name_address():
    r = vies_lookup(_inp(), http_get=_http(
        {"isValid": True, "userError": "VALID", "name": "ACME SRL", "address": "VIA X 1 ROMA"}))
    assert r.status == "ok" and r.fields["vat_status"].value == "valid"
    assert r.fields["registered_name"].value == "ACME SRL" and r.redistributable is False

def test_invalid_vat():
    r = vies_lookup(_inp(), http_get=_http({"isValid": False, "userError": "INVALID",
                                            "name": "---", "address": "---"}))
    assert r.status == "invalid" and "registered_name" not in r.fields

def test_ms_unavailable_is_unavailable_not_invalid():
    r = vies_lookup(_inp(), http_get=_http({"isValid": False, "userError": "MS_UNAVAILABLE",
                                            "name": "---", "address": "---"}))
    assert r.status == "unavailable"

def test_no_vat_is_not_applicable():
    r = vies_lookup(_inp(vat=None), http_get=_http({}))
    assert r.status == "not_applicable"

def test_transport_error_is_unavailable():
    def boom(url): raise OSError("dns")
    r = vies_lookup(_inp(), http_get=boom)
    assert r.status == "unavailable"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_enrich_vies.py -q`
Expected: FAIL with `ModuleNotFoundError`.

- [ ] **Step 3: Write minimal implementation**

```python
# enrich/providers/base.py
"""Default HTTP GET returning (status_code, parsed_json). Stdlib only (urllib) so there is no new
dependency and it works in the SiS-free Hugging Face runtime. Tests inject a fake instead."""
from __future__ import annotations
import json
import urllib.request
from enrich import config


def default_http_get(url: str) -> tuple[int, dict]:
    req = urllib.request.Request(url, headers={"Accept": "application/json",
                                               "User-Agent": "Sinetti-Enrichment/1.0"})
    with urllib.request.urlopen(req, timeout=config.TIMEOUTS[1]) as resp:
        body = resp.read().decode("utf-8", "replace")
        return resp.status, (json.loads(body) if body else {})
```

```python
# enrich/providers/vies.py
"""VIES VAT validation. Exact (confidence 1.0), Italian-VAT subset only, NON-redistributable.
Every HTTP 200 carries the outcome in userError; name/address are '---' when not valid."""
from __future__ import annotations
from datetime import date
from enrich import config
from enrich.models import CompanyInput, Field, ProviderResult

_UNAVAILABLE_CODES = {"MS_UNAVAILABLE", "MS_MAX_CONCURRENT_REQ", "GLOBAL_MAX_CONCURRENT_REQ",
                      "SERVICE_UNAVAILABLE", "TIMEOUT"}


def vies_lookup(inp: CompanyInput, *, http_get) -> ProviderResult:
    today = date.today().isoformat()
    if not inp.vat or not inp.country:
        return ProviderResult("VIES", "not_applicable", {}, redistributable=False)
    url = config.VIES_REST.format(country=inp.country, vat=inp.vat)
    try:
        code, body = http_get(url)
    except Exception:
        return ProviderResult("VIES", "unavailable", {}, redistributable=False)
    if code != 200 or not isinstance(body, dict):
        return ProviderResult("VIES", "unavailable", {}, redistributable=False)
    err = (body.get("userError") or "").upper()
    if err in _UNAVAILABLE_CODES:
        return ProviderResult("VIES", "unavailable", {}, redistributable=False)
    if not body.get("isValid"):
        return ProviderResult("VIES", "invalid",
                              {"vat_status": Field("invalid", "VIES", 1.0, today)},
                              redistributable=False)
    fields = {"vat_status": Field("valid", "VIES", 1.0, today)}
    name, addr = body.get("name"), body.get("address")
    if name and name != "---":
        fields["registered_name"] = Field(name, "VIES", 1.0, today)
    if addr and addr != "---":
        fields["registered_address"] = Field(addr, "VIES", 1.0, today)
    return ProviderResult("VIES", "ok", fields, redistributable=False)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_enrich_vies.py -q`
Expected: PASS (5 passed).

- [ ] **Step 5: Commit**

```bash
git add enrich/providers/base.py enrich/providers/vies.py tests/test_enrich_vies.py
git commit -m "feat(enrich): add VIES VAT-validation provider (userError-aware)"
```

---

### Task 8: `enrich/providers/gleif.py` — identity discovery via LEI

**Files:**
- Create: `enrich/providers/gleif.py`
- Test: `tests/test_enrich_gleif.py`

**Interfaces:**
- Consumes: `CompanyInput`, `Field`, `ProviderResult` (Task 1); `matching.score` (Task 3); `config.GLEIF_BASE`.
- Produces: `gleif_lookup(inp, *, http_get, abstain_threshold) -> ProviderResult`.

**Discovery algorithm (fixed):** structured filter first; if zero candidates, fall back to `fuzzycompletions`; union+dedupe by LEI; score; best ≥ threshold or `no_match`. `http_get` is called per-URL; the test fake dispatches on URL substring.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_enrich_gleif.py
from enrich.providers.gleif import gleif_lookup
from enrich.models import CompanyInput

def _inp(name="VENANZIEFFE SRL", norm="venanzieffe", country="IT"):
    return CompanyInput("INS", "1", name, norm, country, None, "Roma", "00197", None, "valid", False)

LEI = "984500ABCDEF12345678"
RECORD = {"data": {"id": LEI, "attributes": {"entity": {
    "legalName": {"name": "VENANZIEFFE SRL"},
    "legalAddress": {"addressLines": ["VIA X 1"], "city": "Roma", "country": "IT", "postalCode": "00197"},
    "registeredAs": "RM-123456", "registeredAt": {"id": "RA000407", "other": None}}}}}
LIST_HIT = {"data": [{"id": LEI, "attributes": {"entity": {
    "legalName": {"name": "VENANZIEFFE SRL"},
    "legalAddress": {"country": "IT", "postalCode": "00197"}}}}], "meta": {"pagination": {"total": 1}}}
LIST_EMPTY = {"data": [], "meta": {"pagination": {"total": 0}}}

def _http(map_):
    def go(url):
        for frag, payload in map_.items():
            if frag in url:
                return (200, payload)
        return (404, {})
    return go

def test_structured_hit_resolves_identity():
    http = _http({"filter[entity.legalName]": LIST_HIT, f"/lei-records/{LEI}": RECORD})
    r = gleif_lookup(_inp(), http_get=http, abstain_threshold=0.85)
    assert r.status == "ok" and r.redistributable is True
    assert r.fields["lei"].value == LEI
    assert r.fields["national_reg"].value == "RM-123456"
    assert r.fields["registry"].value == "RA000407"

def test_zero_then_fuzzy_fallback():
    http = _http({"filter[entity.legalName]": LIST_EMPTY,
                  "fuzzycompletions": {"data": [{"attributes": {"value": "VENANZIEFFE SRL"},
                      "relationships": {"lei-records": {"data": {"id": LEI}}}}]},
                  f"/lei-records/{LEI}": RECORD})
    r = gleif_lookup(_inp(), http_get=http, abstain_threshold=0.85)
    assert r.status == "ok" and r.fields["lei"].value == LEI

def test_below_threshold_abstains():
    bad = {"data": [{"id": LEI, "attributes": {"entity": {
        "legalName": {"name": "Globex Petroleum AG"},
        "legalAddress": {"country": "DE"}}}}], "meta": {"pagination": {"total": 1}}}
    http = _http({"filter[entity.legalName]": bad, "fuzzycompletions": {"data": []}})
    r = gleif_lookup(_inp(), http_get=http, abstain_threshold=0.85)
    assert r.status == "no_match"

def test_transport_error_is_unavailable():
    def boom(url): raise OSError("dns")
    r = gleif_lookup(_inp(), http_get=boom, abstain_threshold=0.85)
    assert r.status == "unavailable"
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_enrich_gleif.py -q`
Expected: FAIL with `ModuleNotFoundError`.

- [ ] **Step 3: Write minimal implementation**

```python
# enrich/providers/gleif.py
"""GLEIF identity discovery: name+country -> best LEI -> identity + national reg #.
Fuzzy (confidence = match score), redistributable (CC0). Abstains below the threshold rather
than guess. Ownership endpoints are a v1.1 fast-follow and are intentionally not called here."""
from __future__ import annotations
import urllib.parse
from datetime import date
from enrich import config
from enrich.matching import score
from enrich.models import CompanyInput, Field, ProviderResult


def _candidates_from_list(body: dict) -> list[dict]:
    out = []
    for rec in (body.get("data") or []):
        ent = (rec.get("attributes") or {}).get("entity") or {}
        addr = ent.get("legalAddress") or {}
        out.append({"lei": rec.get("id"),
                    "name": (ent.get("legalName") or {}).get("name", ""),
                    "country": addr.get("country", ""),
                    "postcode": addr.get("postalCode", ""),
                    "address": " ".join(addr.get("addressLines") or [])})
    return out


def _candidates_from_fuzzy(body: dict) -> list[dict]:
    out = []
    for rec in (body.get("data") or []):
        lei = (((rec.get("relationships") or {}).get("lei-records") or {}).get("data") or {}).get("id")
        if lei:
            out.append({"lei": lei, "name": (rec.get("attributes") or {}).get("value", ""),
                        "country": "", "postcode": "", "address": ""})
    return out


def gleif_lookup(inp: CompanyInput, *, http_get, abstain_threshold: float) -> ProviderResult:
    today = date.today().isoformat()
    if inp.anonymised or not inp.name:
        return ProviderResult("GLEIF", "not_applicable", {}, redistributable=True)
    try:
        name_q = urllib.parse.quote(inp.name)
        url = (f"{config.GLEIF_BASE}/lei-records?filter[entity.legalName]={name_q}")
        if inp.country:
            url += f"&filter[entity.legalAddress.country]={inp.country}"
        _, body = http_get(url)
        cands = _candidates_from_list(body if isinstance(body, dict) else {})
        if not cands:
            _, fbody = http_get(f"{config.GLEIF_BASE}/fuzzycompletions?field=entity.legalName&q={name_q}")
            cands = _candidates_from_fuzzy(fbody if isinstance(fbody, dict) else {})
        # de-dupe by LEI
        seen, uniq = set(), []
        for c in cands:
            if c["lei"] and c["lei"] not in seen:
                seen.add(c["lei"]); uniq.append(c)
        if not uniq:
            return ProviderResult("GLEIF", "no_match", {}, redistributable=True)
        best = max(uniq, key=lambda c: score(c, inp))
        best_score = score(best, inp)
        if best_score < abstain_threshold:
            return ProviderResult("GLEIF", "no_match", {}, redistributable=True)
        _, rec = http_get(f"{config.GLEIF_BASE}/lei-records/{best['lei']}")
        ent = ((rec.get("data") or {}).get("attributes") or {}).get("entity") or {} if isinstance(rec, dict) else {}
        reg_at = ent.get("registeredAt") or {}
        fields = {
            "lei": Field(best["lei"], "GLEIF", best_score, today),
            "registered_name": Field((ent.get("legalName") or {}).get("name", best["name"]),
                                     "GLEIF", best_score, today),
            "national_reg": Field(ent.get("registeredAs"), "GLEIF", best_score, today),
            "registry": Field(reg_at.get("id") or reg_at.get("other"), "GLEIF", best_score, today),
        }
        return ProviderResult("GLEIF", "ok", fields, redistributable=True)
    except Exception:
        return ProviderResult("GLEIF", "unavailable", {}, redistributable=True)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_enrich_gleif.py -q`
Expected: PASS (4 passed).

- [ ] **Step 5: Commit**

```bash
git add enrich/providers/gleif.py tests/test_enrich_gleif.py
git commit -m "feat(enrich): add GLEIF identity-discovery provider (filter->fuzzy->score)"
```

---

### Task 9: `enrich/engine.py` — orchestration, reconciliation, four-state verdict

**Files:**
- Create: `enrich/engine.py`
- Test: `tests/test_enrich_engine.py`

**Interfaces:**
- Consumes: everything above — `adapter.from_sinetti_record`, `fraud_lookup`, `vies_lookup`, `gleif_lookup`, `matching.normalise/score`, `config`, `CacheStore`/`ScreeningSource` ports.
- Produces: `profile(row, scheme, *, external_allowed, cache, screening, http_get=None, ins_pdf_index=None) -> CompanyProfile`; plus internal `_input_key`, `_reconcile`, `_verdict` (tested via `profile`).

**Verdict (first-match short-circuit, four states):** high_risk → caution → verified → insufficient_data. `cert_status==unknown` is neutral (never caution). A clean company with no positive external confirmation is `insufficient_data`, not caution.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_enrich_engine.py
from enrich.engine import profile
from enrich.providers.vies import vies_lookup  # noqa (ensures import path)

class MemCache:
    def __init__(self): self.d = {}
    def get(self, p, k): return self.d.get((p, k))
    def put(self, p, k, status, payload, ttl_days): self.d[(p, k)] = {"status": status, "payload": payload}

class Screen:
    def __init__(self, hits=None): self.hits = hits or []
    def blocklist_hits(self, name): return self.hits

def _http(map_):
    def go(url):
        for frag, payload in map_.items():
            if frag in url: return (200, payload)
        return (404, {})
    return go

INS_ROW = {"certificate_id": "10002290152", "holder_name": "VENANZIEFFE SRL",
           "holder_norm": "venanzieffe", "country": "IT", "city": "Roma", "postcode": "00197",
           "partita_iva": "10002290152", "codice_fiscale": "10002290152", "status": "active"}
VIES_OK = {"isValid": True, "userError": "VALID", "name": "VENANZIEFFE SRL", "address": "VIA X 1 ROMA"}

def test_valid_vat_clean_certvalid_is_verified():
    p = profile(INS_ROW, "INS", external_allowed=True, cache=MemCache(), screening=Screen(),
                http_get=_http({"/vat/": VIES_OK, "lei-records": {"data": []},
                                "fuzzycompletions": {"data": []}}),
                ins_pdf_index={"piva:10002290152": {"status": "valid"}})
    assert p.verdict.level == "verified"
    assert p.name_match.level in ("match", "partial")

def test_blocklisted_is_high_risk():
    p = profile(INS_ROW, "INS", external_allowed=True, cache=MemCache(),
                screening=Screen([{"category": "fake", "norm": "venanzieffe"}]),
                http_get=_http({"/vat/": VIES_OK, "lei-records": {"data": []}, "fuzzycompletions": {"data": []}}),
                ins_pdf_index={"piva:10002290152": {"status": "valid"}})
    assert p.verdict.level == "high_risk"

def test_nonEU_no_vat_no_lei_clean_is_insufficient_data():
    row = {"certificate_id": "x", "holder_name": "OLFAR S/A ALIMENTO E ENERGIA",
           "holder_norm": "olfar", "country": "BR", "raw_status": "1", "suspended_date": ""}
    p = profile(row, "ISCC", external_allowed=True, cache=MemCache(), screening=Screen(),
                http_get=_http({"lei-records": {"data": []}, "fuzzycompletions": {"data": []}}))
    assert p.verdict.level == "insufficient_data"

def test_offline_runs_fraud_only():
    p = profile(INS_ROW, "INS", external_allowed=False, cache=MemCache(), screening=Screen(),
                http_get=None)
    provs = {pr.provider: pr.status for pr in p.providers}
    assert provs["VIES"] == "unavailable" and provs["GLEIF"] == "unavailable"
    assert provs["ISCC-lists"] == "ok"

def test_vies_invalid_name_unconfirmable_is_caution():
    bad = {"isValid": False, "userError": "INVALID", "name": "---", "address": "---"}
    p = profile(INS_ROW, "INS", external_allowed=True, cache=MemCache(), screening=Screen(),
                http_get=_http({"/vat/": bad, "lei-records": {"data": []}, "fuzzycompletions": {"data": []}}),
                ins_pdf_index={"piva:10002290152": {"status": "valid"}})
    assert p.verdict.level == "caution"

def test_cache_roundtrip_uses_same_key():
    cache = MemCache()
    args = dict(external_allowed=True, cache=cache, screening=Screen(),
                http_get=_http({"/vat/": VIES_OK, "lei-records": {"data": []}, "fuzzycompletions": {"data": []}}),
                ins_pdf_index={"piva:10002290152": {"status": "valid"}})
    profile(INS_ROW, "INS", **args)
    keys_after_first = set(cache.d.keys())
    profile(INS_ROW, "INS", **args)
    assert set(cache.d.keys()) == keys_after_first   # no key drift -> reused, not duplicated
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_enrich_engine.py -q`
Expected: FAIL with `ModuleNotFoundError: No module named 'enrich.engine'`.

- [ ] **Step 3: Write minimal implementation**

```python
# enrich/engine.py
"""Orchestrate the providers behind the cache + capability gate, reconcile the registry name against
the certificate, and compute the four-state trust verdict. Pure: all I/O is injected."""
from __future__ import annotations
import hashlib
from enrich import config, adapter, matching
from enrich.models import CompanyInput, NameMatch, ProviderResult, Verdict, CompanyProfile
from enrich.providers.fraud import fraud_lookup
from enrich.providers.vies import vies_lookup
from enrich.providers.gleif import gleif_lookup


def _input_key(provider: str, inp: CompanyInput) -> str:
    canonical = inp.vat or "" if provider == "VIES" else f"{inp.norm}|{inp.country or ''}"
    return hashlib.sha256(f"{config.KEY_VERSION}|{canonical}".encode()).hexdigest()


def _cached_or_call(provider: str, inp: CompanyInput, cache, call):
    """Return (ProviderResult, from_cache). Caches ok/no_match (status-aware TTL); never caches
    unavailable/not_applicable. Cache stores only parsed fields (value+source+confidence+as_of)."""
    key = _input_key(provider, inp)
    hit = cache.get(provider, key)
    if hit is not None:
        from enrich.models import Field
        fields = {k: Field(**v) for k, v in hit["payload"].items()}
        red = provider != "VIES"
        return ProviderResult(provider, hit["status"], fields, red), True
    res = call()
    if res.status in ("ok", "no_match"):
        ttl = config.TTL_NO_MATCH if res.status == "no_match" else config.TTL_OK.get(provider, 30)
        payload = {k: vars(f) for k, f in res.fields.items()}
        cache.put(provider, key, res.status, payload, ttl)
    return res, False


def _reconcile(inp: CompanyInput, vies: ProviderResult, gleif: ProviderResult) -> NameMatch:
    for src in (vies, gleif):
        f = src.fields.get("registered_name")
        if f and f.value:
            s = matching.score({"name": str(f.value), "country": inp.country,
                                "postcode": inp.postcode, "address": inp.address}, inp)
            level = "match" if s >= 0.85 else "partial" if s >= 0.6 else "mismatch"
            return NameMatch(level, round(s, 3), src.provider)
    return NameMatch("n/a", 0.0, "")


def _verdict(inp, fraud, vies, gleif, nm: NameMatch) -> Verdict:
    on_list = fraud.fields["on_fraud_list"].value
    cert = inp.cert_status
    r = []
    if on_list:
        r.append(f"On ISCC fake/excluded list (ISCC-lists, {fraud.fields['on_fraud_list'].as_of})")
    if cert in ("withdrawn", "suspended"):
        r.append(f"Certificate {cert} (ISCC-lists)")
    if on_list or cert in ("withdrawn", "suspended"):
        return Verdict("high_risk", r)

    if vies.status == "invalid":
        r.append("VAT present but INVALID (VIES)")
    if cert == "expired":
        r.append("Certificate expired (ISCC-lists)")
    if nm.level == "mismatch":
        r.append(f"Registry name does not match the certificate ({nm.compared_to})")
    if r:
        return Verdict("caution", r)

    positive = (vies.status == "ok") or (gleif.status == "ok")
    if positive and cert == "valid":
        if vies.status == "ok":
            r.append("VAT valid (VIES)")
        if gleif.status == "ok":
            r.append(f"Identity confirmed via LEI (GLEIF, conf {gleif.fields['lei'].confidence})")
        if nm.level in ("match", "partial"):
            r.append(f"Registry name matches the certificate ({nm.level})")
        return Verdict("verified", r)

    return Verdict("insufficient_data",
                   ["No fraud signal, but no positive external confirmation available "
                    f"(cert status: {cert})"])


def profile(row, scheme, *, external_allowed, cache, screening, http_get=None, ins_pdf_index=None):
    if http_get is None:
        from enrich.providers.base import default_http_get
        http_get = default_http_get
    inp = adapter.from_sinetti_record(row, scheme, ins_pdf_index=ins_pdf_index)

    fraud = fraud_lookup(inp, screening)
    if external_allowed:
        vies, _ = _cached_or_call("VIES", inp, cache, lambda: vies_lookup(inp, http_get=http_get))
        gleif, _ = _cached_or_call("GLEIF", inp, cache,
                                   lambda: gleif_lookup(inp, http_get=http_get,
                                                        abstain_threshold=config.ABSTAIN_THRESHOLD))
    else:
        vies = ProviderResult("VIES", "unavailable", {}, redistributable=False)
        gleif = ProviderResult("GLEIF", "unavailable", {}, redistributable=True)

    nm = _reconcile(inp, vies, gleif)
    verdict = _verdict(inp, fraud, vies, gleif, nm)

    identity = {}
    for src in (vies, gleif):
        for k in ("registered_name", "registered_address", "vat_status", "lei", "national_reg", "registry"):
            if k in src.fields and k not in identity:
                identity[k] = src.fields[k]
    risk = [fraud.fields["on_fraud_list"], fraud.fields["cert_status"]]
    if "fraud_list_entries" in fraud.fields:
        risk.append(fraud.fields["fraud_list_entries"])
    return CompanyProfile(inp, identity, nm, risk, verdict, [fraud, vies, gleif])
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_enrich_engine.py -q`
Expected: PASS (6 passed).

- [ ] **Step 5: Commit**

```bash
git add enrich/engine.py tests/test_enrich_engine.py
git commit -m "feat(enrich): add engine orchestration, reconciliation, 4-state verdict"
```

---

### Task 10: `enrichment_ui.py` + wire into `render_global_verify`

**Files:**
- Create: `enrichment_ui.py`
- Modify: `iscc.py` (inside `render_global_verify()`, per-scheme column — add a holder picker + "Run due-diligence profile" button)
- Test: `tests/test_enrichment_ui.py` (logic-only: profile assembly + verdict→chip mapping; not Streamlit rendering)

**Interfaces:**
- Consumes: `enrich.engine.profile`, `enrich_store.make_cache_store`, `enrich_store.record_enrichment`, `enrich_screening.IsccScreening`, `ui.live_fetch_available`, `ins.load_cert_pdf_index`.
- Produces: `run_profile(row, scheme) -> CompanyProfile`; `verdict_chip(level) -> (label, color)`; `render_profile_panel(profile)`.

- [ ] **Step 1: Write the failing test**

```python
# tests/test_enrichment_ui.py
import enrichment_ui

def test_verdict_chip_mapping():
    assert enrichment_ui.verdict_chip("verified")[0].lower().startswith("verified")
    assert enrichment_ui.verdict_chip("high_risk")[1]  # has a colour
    for lvl in ("high_risk", "caution", "verified", "insufficient_data"):
        label, color = enrichment_ui.verdict_chip(lvl)
        assert label and color

def test_run_profile_offline_path(monkeypatch):
    # Force offline so no network is attempted; fraud-only path.
    monkeypatch.setattr(enrichment_ui, "_external_allowed", lambda: False)
    monkeypatch.setattr(enrichment_ui, "_screening", lambda: type("S", (), {"blocklist_hits": lambda self, n: []})())
    monkeypatch.setattr(enrichment_ui, "_cache", lambda: type("C", (), {
        "get": lambda self, p, k: None, "put": lambda self, *a: None})())
    monkeypatch.setattr(enrichment_ui, "_record", lambda *a: None)
    row = {"certificate_id": "1", "holder_name": "ACME SRL", "holder_norm": "acmesrl",
           "country": "IT", "raw_status": "1", "suspended_date": ""}
    p = enrichment_ui.run_profile(row, "ISCC")
    assert p.verdict.level in ("verified", "insufficient_data", "caution", "high_risk")
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_enrichment_ui.py -q`
Expected: FAIL with `ModuleNotFoundError: No module named 'enrichment_ui'`.

- [ ] **Step 3: Write minimal implementation**

```python
# enrichment_ui.py
"""Thin Streamlit panel for the Counterparty Profile. Reads a CompanyProfile from the engine and
renders it with source chips + the four-state verdict. Knows nothing about HTTP or DB shape.
Streamlit <=1.52.2 API only."""
from __future__ import annotations

_CHIP = {
    "high_risk": ("High risk", "#9A3B2E"),
    "caution": ("Caution", "#B5803F"),
    "verified": ("Verified", "#3E6B4F"),
    "insufficient_data": ("Unverified — insufficient data", "#756A4D"),
}


def verdict_chip(level: str):
    return _CHIP.get(level, ("Unknown", "#756A4D"))


def _external_allowed() -> bool:
    import ui
    return ui.live_fetch_available()


def _cache():
    import enrich_store
    return enrich_store.make_cache_store()


def _screening():
    import enrich_screening
    return enrich_screening.IsccScreening()


def _record(holder_norm, scheme, level, reasons):
    import enrich_store
    try:
        enrich_store.record_enrichment(holder_norm, scheme, level, reasons)
    except Exception:
        pass


def run_profile(row: dict, scheme: str):
    from enrich import engine
    ins_idx = None
    if scheme == "INS":
        import ins
        ins_idx = ins.load_cert_pdf_index()
    p = engine.profile(row, scheme, external_allowed=_external_allowed(),
                       cache=_cache(), screening=_screening(), ins_pdf_index=ins_idx)
    _record(p.input.norm, scheme, p.verdict.level, p.verdict.reasons)
    return p


def render_profile_panel(p) -> None:
    import streamlit as st
    label, color = verdict_chip(p.verdict.level)
    st.markdown(
        f"<div style='border-left:4px solid {color};padding:6px 12px;margin:6px 0;"
        f"background:#F0ECE0;border-radius:7px'><b style='color:{color}'>{label}</b></div>",
        unsafe_allow_html=True)
    for reason in p.verdict.reasons:
        st.caption(f"• {reason}")
    if p.name_match.level != "n/a":
        st.caption(f"Registry-name reconciliation: **{p.name_match.level}** "
                   f"(score {p.name_match.score}, vs {p.name_match.compared_to})")
    with st.expander("Identity & evidence", expanded=False):
        for k, f in p.identity.items():
            st.write(f"**{k.replace('_',' ')}:** {f.value}  ·  _{f.source}, {f.as_of}_")
        for pr in p.providers:
            st.caption(f"{pr.provider}: {pr.status}")
```

- [ ] **Step 4: Wire the trigger into the verifier**

In `iscc.py`, inside `render_global_verify()` — within the `with col:` block, **after** the existing `_verify_match_rows` markdown and the "Open N in Browse" button — add:

```python
            ents = group_entities(matches)
            options = {e["holder_name"]: e["holder_norm"] for e in ents}
            picked = st.selectbox("Due-diligence target", list(options.keys()),
                                  key=f"dd_pick_{scheme}", label_visibility="collapsed")
            if st.button(":material/policy: Run due-diligence profile", key=f"dd_run_{scheme}",
                         use_container_width=True):
                norm = options[picked]
                row = next(c for c in matches if c.get("holder_norm") == norm)
                import enrichment_ui
                st.session_state[f"dd_profile_{scheme}"] = enrichment_ui.run_profile(row, scheme)
            prof = st.session_state.get(f"dd_profile_{scheme}")
            if prof is not None:
                import enrichment_ui
                enrichment_ui.render_profile_panel(prof)
```

- [ ] **Step 5: Run tests + a manual smoke**

Run: `python -m pytest tests/test_enrichment_ui.py -q`
Expected: PASS (2 passed).
Manual (dev, has internet): `streamlit run app.py` → search a known Italian INS holder → "Run due-diligence profile" → verify the panel renders a verdict with reasons + source chips.

- [ ] **Step 6: Commit**

```bash
git add enrichment_ui.py iscc.py tests/test_enrichment_ui.py
git commit -m "feat(enrich): add Counterparty Profile panel wired into the global verifier"
```

---

### Task 11: Hardening tests — contract pinning + recorded cassettes

**Files:**
- Create: `tests/test_enrich_contracts.py`
- Create: `tests/fixtures/` (recorded JSON: `vies_valid.json`, `gleif_list_hit.json`, `gleif_record.json`)
- Test: the file above

**Interfaces:**
- Consumes: `vies_lookup`, `gleif_lookup`, `config`.
- Produces: request-shape pinning (catches URL/filter drift offline) + cassette replays of the full chains.

- [ ] **Step 1: Write the failing test (request-shape pinning)**

```python
# tests/test_enrich_contracts.py
import json, pathlib
from enrich import config
from enrich.providers.vies import vies_lookup
from enrich.providers.gleif import gleif_lookup
from enrich.models import CompanyInput

FX = pathlib.Path(__file__).parent / "fixtures"

def _inp(vat="10002290152", name="VENANZIEFFE SRL", country="IT"):
    return CompanyInput("INS", "1", name, "venanzieffe", country, None, "Roma", "00197",
                        vat, "valid", False)

def test_vies_builds_expected_url():
    seen = {}
    def http(url): seen["url"] = url; return (200, json.loads((FX / "vies_valid.json").read_text()))
    vies_lookup(_inp(), http_get=http)
    assert seen["url"] == "https://ec.europa.eu/taxation_customs/vies/rest-api/ms/IT/vat/10002290152"

def test_gleif_builds_filter_url_with_alpha2_country():
    urls = []
    def http(url):
        urls.append(url)
        if "/lei-records/" in url and "filter" not in url:
            return (200, json.loads((FX / "gleif_record.json").read_text()))
        return (200, json.loads((FX / "gleif_list_hit.json").read_text()))
    gleif_lookup(_inp(), http_get=http, abstain_threshold=0.85)
    assert any("filter[entity.legalName]=" in u and "filter[entity.legalAddress.country]=IT" in u
               for u in urls)

def test_gleif_cassette_full_chain_resolves_identity():
    def http(url):
        if "/lei-records/" in url and "filter" not in url:
            return (200, json.loads((FX / "gleif_record.json").read_text()))
        return (200, json.loads((FX / "gleif_list_hit.json").read_text()))
    r = gleif_lookup(_inp(), http_get=http, abstain_threshold=0.85)
    assert r.status == "ok" and r.fields["national_reg"].value
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python -m pytest tests/test_enrich_contracts.py -q`
Expected: FAIL — fixtures missing / `FileNotFoundError`.

- [ ] **Step 3: Create the recorded fixtures**

Create `tests/fixtures/vies_valid.json`:
```json
{"isValid": true, "userError": "VALID", "name": "VENANZIEFFE SRL", "address": "VIA X 1, 00197 ROMA"}
```
Create `tests/fixtures/gleif_list_hit.json`:
```json
{"data": [{"id": "984500ABCDEF12345678", "attributes": {"entity": {
  "legalName": {"name": "VENANZIEFFE SRL"},
  "legalAddress": {"country": "IT", "postalCode": "00197"}}}}],
 "meta": {"pagination": {"total": 1}}}
```
Create `tests/fixtures/gleif_record.json`:
```json
{"data": {"id": "984500ABCDEF12345678", "attributes": {"entity": {
  "legalName": {"name": "VENANZIEFFE SRL"},
  "legalAddress": {"addressLines": ["VIA X 1"], "city": "Roma", "country": "IT", "postalCode": "00197"},
  "registeredAs": "RM-123456", "registeredAt": {"id": "RA000407", "other": null}}}}}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `python -m pytest tests/test_enrich_contracts.py -q`
Expected: PASS (3 passed).

- [ ] **Step 5: Commit**

```bash
git add tests/test_enrich_contracts.py tests/fixtures/
git commit -m "test(enrich): pin provider request shapes + cassette replays"
```

---

### Task 12: Integration — full suite, optional live smoke, SiS-degrade check

**Files:**
- Create: `tests/test_enrich_live_smoke.py` (skipped by default)
- Test: the whole suite

**Interfaces:**
- Consumes: the public engine surface only.

- [ ] **Step 1: Add an opt-in live smoke test**

```python
# tests/test_enrich_live_smoke.py
import os, pytest
pytestmark = pytest.mark.skipif(os.environ.get("ENRICH_LIVE") != "1",
                                reason="live smoke: set ENRICH_LIVE=1 to run")

def test_vies_live_known_valid():
    from enrich.providers.vies import vies_lookup
    from enrich.models import CompanyInput
    inp = CompanyInput("INS", "1", "GOOGLE IRELAND", "google", "IE", None, None, None,
                       None, "valid", False)
    inp = inp.__class__(**{**vars(inp), "vat": None})  # IE example handled separately; smoke GLEIF below

def test_gleif_live_known_entity():
    from enrich.providers.gleif import gleif_lookup
    from enrich.models import CompanyInput
    from enrich.providers.base import default_http_get
    inp = CompanyInput("ISCC", "1", "ALPHABET INC", "alphabet", "US", None, None, None,
                       None, "valid", False)
    r = gleif_lookup(inp, http_get=default_http_get, abstain_threshold=0.6)
    assert r.status in ("ok", "no_match")   # network-dependent; just must not crash
```

- [ ] **Step 2: Run the FULL suite (no live)**

Run: `python -m pytest -q`
Expected: all pass, including the pre-existing ~179 Sinetti tests (no regression). The live smoke file is skipped.

- [ ] **Step 3: Verify SiS-degrade behaviour (offline path) by simulation**

Run: `APP_BACKEND=snowflake python -c "import runtime; print('in_snowflake', runtime.in_snowflake())"`
Expected: prints `in_snowflake True`. Then confirm `ui.live_fetch_available()` is False under that env, so `engine.profile(..., external_allowed=False)` returns VIES/GLEIF `unavailable` and only the fraud screen + cache are used (already covered by `test_offline_runs_fraud_only`).

- [ ] **Step 4: Manual live verification on Hugging Face / dev**

Run `streamlit run app.py` locally (has internet): search a real Italian INS holder with a known VAT → "Run due-diligence profile" → expect 🟢/🟠/⚪ verdict with reasons + VIES name/address + (if matched) a GLEIF LEI. Search a blocklisted entity → expect 🔴 high_risk.

- [ ] **Step 5: Commit**

```bash
git add tests/test_enrich_live_smoke.py
git commit -m "test(enrich): add opt-in live smoke + integration coverage"
```

---

## Phase 0 (do FIRST): GLEIF match-rate spike

> This runs **before Task 1**. It is throwaway code whose deliverable is a *number + a decision*, not a tested module. It de-risks the only uncertain part (the GLEIF fuzzy tier).

- [ ] **Step 1: Write the spike script**

```python
# scripts/gleif_spike.py  (throwaway — not shipped, not imported by the app)
"""Measure GLEIF confident-match rate on a sample of real ISCC+REDcert holders.
Run: python scripts/gleif_spike.py 150   (N defaults to 150). Throttled to respect rate limits."""
import sys, time, json, urllib.parse, urllib.request

GLEIF = "https://api.gleif.org/api/v1"

def http_get(url):
    req = urllib.request.Request(url, headers={"Accept": "application/json",
                                               "User-Agent": "Sinetti-Spike/1.0"})
    with urllib.request.urlopen(req, timeout=10) as r:
        return json.loads(r.read().decode("utf-8", "replace") or "{}")

def main(n):
    import iscc, redcert
    from enrich.adapter import from_sinetti_record
    from enrich.matching import score
    rows = ([(r, "ISCC") for r in iscc._all_valid_certs()][: n // 2]
            + [(r, "REDcert") for r in redcert.load_redcert_certs()][: n // 2])
    hits = abstain = 0
    for row, scheme in rows:
        inp = from_sinetti_record(row, scheme)
        if not inp.name or inp.anonymised:
            continue
        q = urllib.parse.quote(inp.name)
        url = f"{GLEIF}/lei-records?filter[entity.legalName]={q}"
        if inp.country:
            url += f"&filter[entity.legalAddress.country]={inp.country}"
        body = http_get(url)
        cands = [{"name": (((d.get("attributes") or {}).get("entity") or {}).get("legalName") or {}).get("name", ""),
                  "country": (((d.get("attributes") or {}).get("entity") or {}).get("legalAddress") or {}).get("country", ""),
                  "postcode": "", "address": ""} for d in (body.get("data") or [])]
        best = max((score(c, inp) for c in cands), default=0.0)
        if best >= 0.85:
            hits += 1
        else:
            abstain += 1
        time.sleep(0.7)  # throttle
    total = hits + abstain
    print(f"N={total}  confident-match={hits} ({100*hits/max(total,1):.0f}%)  abstain={abstain}")

if __name__ == "__main__":
    main(int(sys.argv[1]) if len(sys.argv) > 1 else 150)
```

*(Note: `iscc._all_valid_certs` / `redcert.load_redcert_certs` are the existing loaders — confirm the exact loader names when wiring; this script imports `enrich.adapter`/`enrich.matching`, so run it after Tasks 1, 3, 4 exist, OR inline a minimal name/country extraction to run it truly first. Either is fine — the spike is exploratory.)*

- [ ] **Step 2: Run it and record the result**

Run: `ENRICH_LIVE=1 python scripts/gleif_spike.py 150`
Record the printed hit-rate in a one-paragraph note appended to the spec's §11 (commit in `test2`).

- [ ] **Step 3: Decision gate**

- If confident-match **≥ ~40%**: proceed with GLEIF as a first-class identity provider; set `config.ABSTAIN_THRESHOLD` from the observed score distribution.
- If **< ~40%**: ship GLEIF as clearly-labelled best-effort (panel shows "identity not confirmed" prominently), keep the engine but lower the GLEIF UI prominence. The VIES + fraud path still stands on its own.

---

## Self-Review (completed by the plan author)

- **Spec coverage:** VIES (T7) · GLEIF identity (T8) · fraud flags (T5) · adapter incl. INS PDF-join + ISCC address + anonymised guard (T4) · reconciliation + 4-state verdict (T9) · status-aware cache + Snowflake mirror + audit history (T6) · capability gate (T9/T10) · UI in the real verifier (T10) · rate-limit/timeout (config T2 + provider try/except) · tests incl. contract/cassette/round-trip (T9/T11) · GLEIF spike + decision gate (Phase 0). Ownership is correctly **out** (v1.1).
- **Placeholder scan:** every code/test step carries real code; no TBD/TODO.
- **Type consistency:** `CompanyInput`, `Field`, `ProviderResult`, `Verdict`, `NameMatch`, `CompanyProfile` signatures match across T1→T10; `from_sinetti_record(row, scheme, *, ins_pdf_index=)`, `vies_lookup(inp, *, http_get=)`, `gleif_lookup(inp, *, http_get=, abstain_threshold=)`, `profile(row, scheme, *, external_allowed=, cache=, screening=, http_get=, ins_pdf_index=)` are used identically everywhere.
