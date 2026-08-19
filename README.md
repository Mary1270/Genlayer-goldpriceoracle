![License](https://img.shields.io/badge/license-MIT-blue)
![Tests](https://img.shields.io/badge/tests-113%20passing-brightgreen)
![GenLayer](https://img.shields.io/badge/GenLayer-Studio%20Deployed-6c5ce7)
![Python](https://img.shields.io/badge/python-3.x-blue)

# GoldPriceOracle — Multi-Source, Karat/Unit-Normalized Gold Price Consensus

A GenLayer Intelligent Contract that resolves two-party gold-price agreements ("party A wins if 24K gold is above $2,000/troy oz, party B wins otherwise") using multi-source, provenance-checked, freshness-checked price consensus, with **real deterministic conversion across units and purities** — not a single caller-chosen page, and not a match-or-reject gate.

> **This contract does not determine an absolute, real-time gold price.** It deterministically decides, given a caller-submitted set of candidate sources — each of which may legitimately quote gold in a *different* unit and karat — whether enough independent, reputable, fresh evidence exists, after converting every source onto one common basis, to say a price is Above, Below, or Equal to a threshold, and if so, records the resulting settlement outcome for a two-party agreement.

---

## 1. Why This Is Not a Renamed Copy of OilPriceOracle

[OilPriceOracle](https://github.com/Mary1270/Genlayer-oilpriceoracle) (an earlier, accepted Intelligent Contract by the same author) settles agreements on crude oil, which trades in exactly **one** unit (USD per barrel) everywhere. Its numeric-normalization problem is a **match-or-reject gate**: parse the number, and if the instrument/currency/unit doesn't match exactly, exclude the source.

Gold is a genuinely different, harder problem. Reputable bullion-market sources legitimately quote gold in **different units** (troy ounce, gram, kilogram, tola) **and** **different purities/karats** (24K pure, 22K, 21K, 18K, 14K, 10K) — simultaneously, for the same real-world metal, at the same moment. A match-or-reject gate would make most reputable sources unusable. This contract instead **converts** every source's price onto one canonical basis (USD per gram of 24-karat gold) using fixed physical conversion constants, so sources quoted in different units/purities can still be validly compared.

What *is* deliberately reused from OilPriceOracle, because it is genuinely commodity-agnostic infrastructure: domain-provenance annotation, duplicate-domain detection, content-quality classification, prompt-injection guardrails, the `prompt_comparative`/`EQUIVALENCE_PRINCIPLE` consensus pattern, the two-party settlement workflow, and — having learned directly from a GenLayer Portal steward's review of OilPriceOracle — the `required_source_domains` source-policy commitment, **included here from the start** rather than retrofitted after a rejection.

---

## 2. The Core Design Decision: What Gets Converted, What Gets Rejected

Two kinds of "mismatch" are deliberately **not** treated the same way:

| | Treatment | Why |
|---|---|---|
| **Karat / Unit** | Converted deterministically, in Python, using fixed physical constants (1 troy oz is *always* 31.1034768g; 22K is *always* 22/24 pure, by definition) | These ratios never change and are identical for every validator — safe for the contract to compute |
| **Currency** | Match-or-reject (same as OilPriceOracle's instrument/currency field) — a non-USD source is excluded as `currency_mismatch` | A USD/EUR exchange rate is a *live, fluctuating* market quantity, not a fixed constant — letting the contract (or the LLM) silently apply "today's" rate would reintroduce exactly the kind of consensus-fragile, possibly-wrong conversion this whole design pattern exists to avoid |

**A direct consequence:** unlike OilPriceOracle, this contract asks the model for **no self-reported COMPARISON field** — asking the model to compare a price across different karats/units would require it to perform exactly the conversion arithmetic this design deliberately keeps out of the LLM's hands. This means there is no analogous "LLM self-consistency check" (OilPriceOracle's `comparison_mismatch` flag) here — see §7 Known Limitations for the disclosed trade-off and its mitigation (multi-source corroboration).

---

## 3. Architecture

```
create_agreement(party_a, party_b, karat, unit, threshold_price, comparison,
                  description, required_source_domains=None)
        │
        └─ validates inputs, normalizes karat ("24k"→"24K") and unit
           ("oz"→"TroyOunce") against fixed vocabularies, validates
           threshold_price is parseable, OPTIONALLY validates a
           committed source policy (§5), stores an "open" agreement

resolve_agreement(agreement_id, source_urls)
        │
        ├─ 1. Deterministic validation: agreement exists & not resolved,
        │      3-6 sources, ≥2 distinct REPUTABLE domains, AND - if a
        │      source policy was committed - every committed domain
        │      present among the submitted sources (§5)
        │
        ├─ 2. Deterministic provenance annotation (_annotate_sources)
        │
        ├─ 3. ONE non-deterministic closure (gl.eq_principle.prompt_comparative)
        │      per source: fetch → classify → LLM reports CURRENCY / KARAT /
        │      UNIT / FRESHNESS / PRICE (verbatim, no conversion) →
        │      contract DETERMINISTICALLY normalizes both the source's
        │      price and the agreement's threshold onto USD-per-gram-24K
        │      (_normalize_to_24k_per_gram) → quality_flag (ok /
        │      currency_mismatch / karat_unrecognized / unit_unrecognized /
        │      stale_or_unknown_freshness / price_unparseable) →
        │      deterministic aggregation (_aggregate) → final_verdict →
        │      deterministic winner derivation from the STORED agreement's
        │      comparison direction (resolve_agreement itself accepts no
        │      parameter that could influence the winner)
        │
        └─ 4. Persist final_verdict + winner + full evidence trail +
               increment resolution_attempts; mark "resolved" only if
               winner != "unresolved"
```

---

## 4. The Deterministic Conversion — Worked Example

`_normalize_to_24k_per_gram(price, karat, unit)` is the only place any unit/purity arithmetic happens in the whole contract:

```
price_per_gram_this_purity = price / UNIT_TO_GRAMS[unit]
price_per_gram_24k          = price_per_gram_this_purity / KARAT_PURITY_FRACTION[karat]
```

Example: a source quotes **"$2,000 per troy ounce, 22-karat"**:

```
2000 / 31.1034768   = 64.2986... USD/gram (at 22K purity)
64.2986... / (22/24) = 70.1471... USD/gram (normalized to 24K/pure basis)
```

A different source quoting **"$80.00 per gram, 24-karat"** normalizes to exactly `80.00` on the same basis — directly comparable to the first source's `70.15`, despite neither unit nor karat matching between them. `test_cross_karat_and_unit_sources_are_correctly_normalized_and_agree` in `tests/test_end_to_end.py` verifies three sources quoted in three different karat/unit combinations, deterministically converted to the same normalized value, all correctly agree.

---

## 5. Source Policy Commitment

`create_agreement` accepts an optional `required_source_domains: list[str]` — identical mechanism to OilPriceOracle's, included here **proactively**, not after a rejection. If given, every listed domain must already be on `REPUTABLE_PRICE_DOMAINS`, must be distinct, and there must be 2–6 entries. At `resolve_agreement` time, every committed domain must be present among the submitted sources (extra reputable domains are still allowed — a floor, not a ceiling) or the attempt is rejected before any fetch, naming the missing domain(s). See `TestSourcePolicyCommitmentValidation` / `TestSourcePolicyCommitmentEnforcement` in `tests/test_end_to_end.py`, and OilPriceOracle's README §3a for the full original design rationale (why commitment was chosen over restricting caller identity) — that reasoning applies unchanged here.

---

## 6. Consensus Model

Same pattern as OilPriceOracle: `gl.eq_principle.prompt_comparative(nondet, principle=EQUIVALENCE_PRINCIPLE)`, never `strict_eq` (GenLayer's own guidance is explicit that `strict_eq` must never be used for LLM-derived output). `EQUIVALENCE_PRINCIPLE` restricts cross-validator equivalence to categorical fields only (`final_verdict`, `winner`, `independent_source_count`, each record's `fetch_status`/`quality_flag`/`comparison`) and explicitly excludes the audit-only `price`, `karat`, and `unit` fields, since different validators may legitimately extract slightly different exact figures from a live page. `test_not_strict_eq` and `test_explicitly_excludes_price_karat_unit_from_equivalence` in `tests/test_prompt_and_consensus.py` verify this.

---

## 7. Known Limitations (Disclosed, Not Hidden)

- **No LLM self-consistency cross-check on the comparison result.** OilPriceOracle can catch a self-inconsistent LLM response (extracted price contradicts its own stated Above/Below conclusion) because it asks for both. This contract deliberately does *not* ask the model to compare (see §2), so that check does not exist here. **Mitigation:** the same `>=2` independent, agreeing, reputable sources requirement `_aggregate` already enforces — a single source's mis-extracted karat/unit/price is far less likely to be replicated identically by two *independent* sources.
- **No currency conversion.** A non-USD source is excluded outright (`currency_mismatch`), not converted — see §2 for why. Multi-currency support would need a live, trusted, on-chain-verifiable FX rate feed, which does not exist in this design.
- **`REPUTABLE_PRICE_DOMAINS` is a small, static, hand-maintained allowlist**, not a live reputation feed — same deliberate determinism trade-off OilPriceOracle makes, for the same reason (every validator must see an identical list).
- **No full Public Suffix List** for registrable-domain extraction — same `KNOWN_MULTI_PART_SUFFIXES` approximation as OilPriceOracle, for the same determinism reasons.
- **Freshness detection** depends on the source page stating or implying a current timestamp — no independent trusted clock exists inside GenVM to cross-check against.
- **No deadline/expiry** on agreements; **re-resolution overwrites prior evidence** (only the most recent attempt's `records` are retained, though `resolution_attempts` is a durable counter) — identical disclosed trade-offs to OilPriceOracle, same rationale (unbounded storage growth avoidance).
- **`required_source_domains` is a floor, not a full lock**, and **a committed domain becoming permanently unreachable can strand an agreement** — identical disclosed trade-offs to OilPriceOracle's §3a/§9.
- **No actual fund transfer** — this contract produces the authoritative, auditable settlement decision; moving funds based on it would require payable-method patterns not verified against a live GenLayer SDK in this development environment.

---

## 8. Public Interface

```python
create_agreement(party_a: str, party_b: str, karat: str, unit: str,
                  threshold_price: str, comparison: str, description: str,
                  required_source_domains: list[str] = None) -> str   # returns agreement_id
resolve_agreement(agreement_id: str, source_urls: list[str]) -> str   # returns full JSON record
get_agreement(agreement_id: str) -> str    # full JSON evidence + settlement record
total_agreements() -> int
```

`karat` accepts `"24k"`, `"22k"`, `"21k"`, `"18k"`, `"14k"`, `"10k"` (case-insensitive). `unit` accepts `"troy_ounce"`/`"oz"`, `"gram"`/`"g"`, `"kilogram"`/`"kg"`, `"tola"`.

Example `get_agreement` result after resolution (`comparison="above"`, normalized price found above threshold):

```json
{
  "agreement_id": "0",
  "status": "resolved",
  "party_a": "alice",
  "party_b": "bob",
  "karat": "24K",
  "unit": "TroyOunce",
  "threshold_price": "1800",
  "comparison": "above",
  "required_source_domains": [],
  "final_verdict": "Above",
  "winner": "party_a",
  "independent_source_count": 3,
  "resolution_attempts": 1,
  "records": [
    {
      "url": "https://kitco.com/a",
      "domain": "kitco.com",
      "is_duplicate_domain": false,
      "is_reputable": true,
      "fetch_status": "ok",
      "quality_flag": "ok",
      "price": 2000.0,
      "karat": "24K",
      "unit": "TroyOunce",
      "comparison": "Above"
    }
  ]
}
```

`price`/`karat`/`unit` are audit metadata only (see §6) — `price` is `null` for any source excluded before price parsing.

---

## 9. Testing

**113/113 offline tests passing**, run via:

```bash
python3 -m unittest discover -s tests -p "test_*.py" -v
```

| File | Tests | Covers |
|---|---|---|
| `test_aggregation.py` | 60 | Domain extraction + allowlist round-trip regression guard, content classification, labeled-field parsing, `_parse_price` (integer/decimal/`$`/comma/negative/ambiguous formats), karat/unit input alias normalization, **`_normalize_to_24k_per_gram`** (identity case, troy-ounce conversion, karat conversion, the combined worked example, kilogram/tola internal consistency, `None`/`Unspecified` handling, and a regression guard that every non-"Unspecified" vocabulary word is actually convertible), and every branch of `_aggregate` |
| `test_prompt_and_consensus.py` | 10 | The prompt asks for exactly the 5 labeled fields and explicitly does **not** ask for a COMPARISON field (the central §2 design decision), contains the injection guardrail, tells the model not to convert; `EQUIVALENCE_PRINCIPLE` matches the real schema and excludes `price`/`karat`/`unit`; confirms `prompt_comparative` (not `strict_eq`) is used |
| `test_end_to_end.py` | 43 | Full `create_agreement`→`resolve_agreement`→`get_agreement` pipeline: input/karat/unit validation, party A/B winning in both directions, Equal-stays-open, **cross-karat-and-cross-unit sources correctly normalizing to agreement**, currency/karat/unit/freshness/price exclusion paths, fetch-failure handling, prompt injection, `resolution_attempts`, winner-manipulation resistance, and the full `TestSourcePolicyCommitmentValidation`/`TestSourcePolicyCommitmentEnforcement` suites (included from day one — see §5) |

These run fully offline against a local stub of the `genlayer` SDK — no GenLayer node, network access, or real LLM required.

---

## 10. Live Deployment

**Contract address:** `0x1763E5C8f4966D2d60e4774a348F46C50fF6AD72`
**Public explorer:** https://explorer-studio.genlayer.com/address/0x1763E5C8f4966D2d60e4774a348F46C50fF6AD72

Deployment reached `FINALIZED`/`SUCCESS` on the first attempt — no GenVM lint issues this time (tx `0xb3b71dc9b884aaf4dc69d028b05ab783329ccfec7c291e99d57e719d84063542`).

Three live transactions on this address exercise both the karat/unit source-policy mechanism and the settlement pipeline end-to-end:

1. **`create_agreement`** (tx `0x17beceec129e3c895526173ad4d5a749a5c43d82532fdc177e5bdb90f3d7c602`, `FINALIZED`/`SUCCESS`) — created agreement `"0"`: `karat="24k"`, `unit="troy_ounce"`, `threshold_price="1800"`, `comparison="above"`, with `required_source_domains=["kitco.com", "bloomberg.com"]` committed at creation.

2. **`resolve_agreement` with a committed domain deliberately omitted** (tx `0x8f8651d3b28dee404ff0d62031f30f7f5ed670e284f77d25082574e8e41f8e3e`, `FINALIZED`/`ERROR`) — submitted `kitco.com`, `tradingeconomics.com`, `reuters.com` (no `bloomberg.com`). Every validator that executed independently rolled back with the identical error naming the missing domain, confirming the source-policy-commitment mechanism (§5) behaves deterministically live, exactly as it does in OilPriceOracle.

3. **`resolve_agreement` with all committed domains present** (tx `0xf28c75058b7ee5beed2a3a1225b53287e2a409916b4c1316dd87c8683f66e26d`, `FINALIZED`/`SUCCESS`) — submitted `kitco.com`, `bloomberg.com`, `tradingeconomics.com` (both committed domains present, one extra). The pre-flight domain check **passed** this time and the pipeline proceeded to fetch/LLM/aggregation, reaching `final_verdict: "Indeterminate"` because the three sample URLs weren't real, fetchable live pages (`fetch_status: "inaccessible"` for all three, `quality_flag: "price_unparseable"`) — a fetch-layer outcome unrelated to the karat/unit normalization logic itself, the same class of result OilPriceOracle's live deployment documented. `get_agreement("0")` afterward confirms the full record, including `"required_source_domains": ["bloomberg.com", "kitco.com"]` and `"status": "open"` (not force-resolved, per the existing Indeterminate-stays-open behavior).

**What this confirms:** deployment with no lint issues, the source-policy commitment validation/enforcement (both the rejection and floor-not-ceiling acceptance paths), and correct karat/unit normalization at `create_agreement` time (`"karat": "24K"`, `"unit": "TroyOunce"` both stored correctly from the `"24k"`/`"troy_ounce"` input) all behave live exactly as the 113 offline tests predict. **What this does NOT confirm:** a successful `Above`/`Below`/`Equal` resolution with real cross-karat/cross-unit sources correctly normalizing against each other on live GenVM execution — that depends on submitting real, currently-fetchable reputable pages quoting gold in different karats/units, which these three sample URLs were not. The underlying `_normalize_to_24k_per_gram` conversion arithmetic itself is verified deterministically by the offline test suite (§9), including the specific worked cross-karat/cross-unit example from §4.
