# Shakepay Canada Tax Engine
### Complete Technical Documentation — v4.2 — March 29, 2026

> **Scope:** Individual taxpayers · Personal use only · Canada Federal + All Provinces & Territories
>
> **Not in scope:** Corporate tax · Employment income · Business inventory · Non-Shakepay exchanges · Legal or tax advice
>
> **Audience:** Product, backend, parser, tax-engine, QA, and reporting teams
>
> **Status:** Implementation-ready specification for a Shakepay-only personal-use Canadian tax workflow.

---

## Table of Contents

1. [Product Overview](#1-product-overview)
2. [Scope](#2-scope)
3. [Input Contract](#3-input-contract)
4. [System Architecture](#4-system-architecture)
5. [Import Layer](#5-import-layer)
6. [Canonical Data Models](#6-canonical-data-models)
7. [Classification Engine](#7-classification-engine)
8. [Event Linking & Reconciliation](#8-event-linking--reconciliation)
9. [ACB Ledger Engine](#9-acb-ledger-engine)
10. [Annual Tax Calculation](#10-annual-tax-calculation)
11. [Tax Estimation Engine](#11-tax-estimation-engine)
12. [Québec Module](#12-québec-module)
13. [Review Queue](#13-review-queue)
14. [Output Contract](#14-output-contract)
15. [API Reference](#15-api-reference)
16. [Database Schema](#16-database-schema)
17. [Tax Law Data Layer](#17-tax-law-data-layer)
18. [Error Handling](#18-error-handling)
19. [Testing Strategy](#19-testing-strategy)
20. [Policy Positions & Caveats](#20-policy-positions--caveats)
21. [Known Issues & Review Notes](#21-known-issues--review-notes)
22. [Decision Log & Open Decisions](#22-decision-log--open-decisions)
23. [Implementation Order](#23-implementation-order)
24. [Definition of Done](#24-definition-of-done)
25. [Recommended Improvements](#25-recommended-improvements)
26. [Configuration](#26-configuration)
27. [Operations](#27-operations)
28. [ACB Engine Performance](#28-acb-engine-performance)
29. [USD/USDC Policy Roadmap](#29-usdustdc-policy-roadmap)
30. [Engineering Reference](#30-engineering-reference)

---

## 1. Product Overview

### 1.1 Purpose

The Shakepay Tax Engine accepts one or more Shakepay summary export files and produces, for each selected Canadian tax year:

- Taxable reward income
- Interest income
- Capital gains and losses from crypto dispositions
- Estimated federal and provincial or territorial tax impact

This is a **Shakepay-only tax computation layer**. It is not a complete tax return tool. Federal and provincial tax payable depends on the user's total annual income across all sources, not only Shakepay activity.

**Target user:** Canadian individual using Shakepay for personal investing or personal-use transactions.
**Primary jurisdiction:** Canada, with first-class Québec support and a framework that can extend to all provinces and territories.

### 1.2 Outputs per tax year

| Output | Description |
|---|---|
| `taxableRewardIncomeCad` | ShakingSats, SecretSats, referrals, ShakeSquad, interest |
| `interestIncomeCad` | CAD balance interest |
| `capitalGainsCad` | Total gains from dispositions |
| `capitalLossesCad` | Total losses from dispositions |
| `netCapitalGainCad` | Gains minus losses |
| `taxableCapitalGainCad` | Net gain × inclusion rate |
| `federalTaxEstimateCad` | Estimated federal tax effect |
| `provincialTaxEstimateCad` | Estimated provincial/territorial tax effect |
| `completenessStatus` | `complete` / `likely_complete` / `incomplete` / `materially_incomplete` |
| `reviewItemCount` | Number of items requiring user action |
| `quebecDisclosureRequired` | Boolean — QC residents with crypto activity |

### 1.3 What the engine does not produce

- A complete T1 or TP-1 income tax return
- Tax payable on income earned outside Shakepay
- Tax treatment for non-individual taxpayers
- Formal tax advice or legal opinion

### 1.4 Governing tax-policy rules (summary)

| Category | Default treatment | Confidence |
|---|---|---|
| ShakingSats | Taxable income + crypto basis | High |
| SecretSats | Taxable income + crypto basis | High |
| Referral bonus | Taxable income | High |
| Promotional referral bonus | Taxable income | Medium |
| ShakeSquad rewards | Taxable income + crypto basis | Medium |
| CAD interest | Interest income | High |
| Card cashback in BTC | Purchase rebate — basis only, not income | **Prudent default** |
| 3% card intro bonus | Purchase rebate — basis only, not income | **Prudent default** |
| Card purchase | Personal non-deductible outflow | High |
| Shake It Forward received | No tax effect (internal transfer) — see §22 | Implemented |
| Shake It Forward sent | Personal non-deductible outflow | Medium |
| Crypto sends/receives | Flagged for ownership review | High |
| USD events | Review-only — not auto capital gains | Implemented |

Positions marked **Prudent default** are not confirmed published CRA or Revenu Québec positions. They must be disclosed in every report. The SIF received treatment (previously flagged for review in the spec) has been resolved to `no_tax_effect` in the decision log — see §22.1.

### 1.5 Specification vs. production compliance completeness

This document describes both architectural intent and the current state of implementation. Readers should distinguish between **specification completeness** — the degree to which the design covers known tax scenarios — and **production compliance completeness** — the degree to which the running system correctly handles every edge case with appropriate disclosures. The most material gaps between those two planes, as of this version, are: the unimplemented superficial loss rule (see §21.7), the review-only treatment of USD/USDC (see §29), and the schema-level enforcement of reconciliation constraints (see §8.1 and §16.3). These are tracked as active implementation obligations, not stable product choices.

---

## 2. Scope

| In scope | Out of scope |
|---|---|
| Canadian individuals; personal-use tax treatment; annual tax summaries; Québec disclosure support; BTC, ETH, and USDC tracking; bilingual Shakepay import support | Corporate tax; business inventory treatment; self-employment treatment; accountant-grade filing automation; legal advice; non-Shakepay records unless imported separately |

---

## 3. Input Contract

The importer must assume the user uploads a bundle of summary files rather than one monolithic transaction-history CSV. This **dossier model** is the foundation of the entire input contract.

### 3.1 Accepted file families

**The primary production filename format observed in real Shakepay exports is the unsuffixed form.** The `_english` / `_french` suffixed variants are accepted alternates. Detection is based entirely on CSV header signatures (see §5.1) — neither naming convention requires configuration changes.

| File | Family | Language |
|---|---|---|
| `crypto_transactions_summary.csv` | `crypto_summary` | detected from headers |
| `cash_transactions_summary.csv` | `cash_summary` | detected from headers |
| `usd_transactions_summary.csv` | `usd_summary` | detected from headers |
| `crypto_transactions_summary_english.csv` | `crypto_summary` | `en` |
| `crypto_transactions_summary_french.csv` | `crypto_summary` | `fr` |
| `cash_transactions_summary_english.csv` | `cash_summary` | `en` |
| `cash_transactions_summary_french.csv` | `cash_summary` | `fr` |
| `usd_transactions_summary_english.csv` | `usd_summary` | `en` |
| `usd_transactions_summary_french.csv` | `usd_summary` | `fr` |

The system must support 1 to 6 files per import session and must explicitly score completeness when files are missing.

> **Filename variation:** The engine does not rely on filenames for family or language detection — it identifies file family and language exclusively from CSV header signatures (see §5.1). Both naming conventions are accepted without any configuration change.

### 3.2 Minimum viable input set

| File | Requirement |
|---|---|
| Crypto summary | **Required** — needed for dispositions, book cost, market value, asset movement. A session without a crypto summary file may still be uploaded and reach `CLASSIFIED` status, but `completenessStatus` MUST be set to `materially_incomplete` for any tax year with activity. See normative rule below. |
| Cash summary | **Strongly recommended** — needed for CAD rewards and fiat reconciliation |
| USD summary | **Optional** — required only if user has Shakepay USD activity |

> **Normative rule — missing crypto file.** A session without a crypto summary file may still be uploaded and reach `CLASSIFIED` status. However:
> - `completenessStatus` MUST be set to `materially_incomplete` for any tax year with activity.
> - The session MUST NOT advance to `CALCULATED` without at least one crypto summary file unless all events in scope are cash-only (no crypto activity detectable).
> - `POST /api/v1/sessions/:id/process` MUST return a `MissingRequiredFileError` (HTTP 422) if no crypto file is present and the session contains any crypto-indicating signals (e.g. cash Buy rows without a paired crypto file).

### 3.3 Observed column schemas

#### Cash summary

| Canonical field | English header | French header |
|---|---|---|
| `timestamp` | `Date` | `Date` |
| `typeRaw` | `Type` | `Type` |
| `descriptionRaw` | `Description` | `Description` |
| `debit` | `Debit` | `Débit` |
| `credit` | `Credit` | `Crédit` |
| `spotRate` | `Spot Rate` | `Taux au comptant` |
| `buySellRate` | `Buy / Sell Rate` | `Taux d'achat/de vente` |
| `balance` | `Balance` | `Solde` |

#### Crypto / USD summary

| Canonical field | English header | French header |
|---|---|---|
| `timestamp` | `Date` | `Date` |
| `amountDebited` | `Amount Debited` | `Montant débité` |
| `assetDebited` | `Asset Debited` | `Actif débité` |
| `amountCredited` | `Amount Credited` | `Montant crédité` |
| `assetCredited` | `Asset Credited` | `Actif crédité` |
| `marketValueCad` | `Market Value` | `Valeur du marché` |
| `marketValueCurrency` | `Market Value Currency` | `Devise de valeur du marché` |
| `bookCostCad` | `Book Cost` | `Coût comptable` |
| `bookCostCurrency` | `Book Cost Currency` | `Devise du coût comptable` |
| `typeRaw` | `Type` | `Type` |
| `spotRate` | `Spot Rate` | `Taux au comptant` |
| `buySellRate` | `Buy / Sell Rate` | `Taux d'achat/de vente` |
| `descriptionRaw` | `Description` | `Description` |

> **Note:** USD summary files use the identical schema as crypto summary files. The engine distinguishes them by the asset values present (`USD` vs `BTC`/`ETH`/`USDC`), not by filename.

### 3.4 Bilingual type values

| Canonical token | English | French |
|---|---|---|
| `buy` | `Buy` | `Achat` |
| `sell` | `Sell` | `Vente` |
| `send` | `Send` | `Envoi` |
| `receive` | `Receive` | `Recevoir` |
| `reward` | `Reward` | `Récompenses` |
| `interac` | `Interac e-Transfer` | `Virement Interac` |
| `convert` | `Convert` | `Convertir` |
| `withdrawal` | `Withdrawal` | `Retrait` |
| `deposit` | `Deposit` | `Dépôt` |
| `card_purchase` | `Card purchase` | *(not yet observed in production French exports — must be confirmed from a French-language export before implementation; treat unrecognised French equivalents as `unknown_review` pending confirmation — see §22.6)* |

---

## 4. System Architecture

### 4.1 Monorepo package layout

| Package / App | Path | Responsibility |
|---|---|---|
| `apps/api` | `apps/api` | Express API (`/api/v1/sessions/...`) |
| `apps/web` | `apps/web` | Vite + React UI; proxies `/api` to the API |
| `packages/parser` | `packages/parser` | CSV detection, canonicalization, `normalizeAsset` |
| `packages/classification` | `packages/classification` | Rule-based classification engine |
| `packages/linking` | `packages/linking` | Cash/crypto leg matching and `linkGroupId` assignment |
| `packages/basis` | `packages/basis` | ACB pools, gain/loss computation, superficial-loss warnings |
| `packages/tax` | `packages/tax` | Annual summaries and Québec disclosure |
| `packages/review` | `packages/review` | Review queue, grouping, bulk decision handling |
| `packages/db` | `packages/db` | Drizzle ORM schema and repositories |
| `packages/orchestration` | `packages/orchestration` | `processSession` pipeline orchestrator |
| `packages/domain` | `packages/domain` | Shared constants (inclusion rates, policy versions, enums) |

### 4.2 Layer overview

| # | Layer | Responsibility |
|---|---|---|
| 1 | **Import & Detection** | Accept files; detect family and language from headers; validate schema |
| 2 | **Canonicalization** | Map bilingual headers to canonical fields; produce `CanonicalRow` records |
| 3 | **Classification** | Assign `normalizedCode` and `taxTreatment` using the priority hierarchy |
| 4 | **Event Linking** | Group related cash + crypto rows into single economic events via `linkGroupId` |
| 5 | **Basis Ledger** | Maintain per-asset ACB pools; compute acquisition cost and disposition gain/loss |
| 6 | **Annual Ledger** | Aggregate events by tax year into reward, interest, capital gain/loss ledgers |
| 7 | **Tax Estimator** | Apply federal and provincial brackets to taxable amounts |
| 8 | **Output Builder** | Assemble annual summary, audit trail, Québec disclosure, completeness score |
| 9 | **Review Queue** | Surface flagged rows; accept user decisions; log every override |

### 4.3 Processing pipeline

The pipeline is orchestrated by a single call in `packages/orchestration/src/process-session.service.ts` and triggered via `POST /api/v1/sessions/:id/process`. Steps run in sequence:

```
User uploads 1–6 files
  → FileDetector         identifies family + language from headers (two-stage — see §5.2)
  → SchemaValidator      confirms required columns; rejects invalid files
  → ColumnMapper         maps raw headers → CanonicalRow
  → ClassificationEngine assigns normalizedCode + taxTreatment
  → EventLinker          reconciles matching cash + crypto rows (packages/linking)
  → BasisLedger          processes acquisitions and dispositions chronologically (packages/basis)
  → AnnualLedgerBuilder  groups events by tax year
  → CoverageScorer       assesses completeness; emits warnings
  → User provides province + optional other income per year (POST /tax-inputs)
  → TaxEstimator         applies federal + provincial rates
  → OutputBuilder        assembles report + Québec disclosure + audit trail
```

> **Entry point:** `computeBasisWithUpdates` in `packages/basis` is called from `process-session.service.ts` with chronologically sorted events for the session.

### 4.4 Import state machine

```
CREATED
  → FILES_UPLOADED       (at least one file present)
  → FILES_DETECTED       (every uploaded file has identified family + language)
  → CANONICALIZED        (all rows normalized)
  → CLASSIFIED           (all rows have normalizedCode)
  → RECONCILED           (economic events linked across files)
  → CALCULATED           (annual summaries computed)
  → REVIEW_PENDING       (unresolved review items remain — optional interstitial)
  → COMPLETE
  → FAILED
```

**Transition rules:**
- `FILES_UPLOADED → FILES_DETECTED` only if every file family/language is resolved
- `CLASSIFIED → RECONCILED` only if canonical rows exist
- `RECONCILED → CALCULATED` allowed even with warnings
- `CALCULATED → REVIEW_PENDING` when unresolved items remain
- `REVIEW_PENDING → CALCULATED` after user resolves overrides
- Any stage → `FAILED` on unreadable files, missing core columns, contradictory duplicates, or impossible parsing
- `FILES_UPLOADED → CALCULATED` is blocked if no crypto summary file is present and crypto-indicating signals exist in the session (e.g. cash Buy rows). In that case, processing MUST halt with `MissingRequiredFileError` and set `completenessStatus = materially_incomplete`.

### 4.5 Design principles

- The parser must not decide tax payable. It only normalizes, classifies, and computes taxable amounts and ledger effects.
- The tax engine must consume annual taxable outputs and versioned tax tables as a separate layer.
- The reconciler is mandatory. Without it, the product risks double-counting the fiat and crypto legs of the same economic event.
- No row may ever be silently dropped. Any row that cannot be classified must produce a `ReviewItem`.

### 4.6 Source file layout

```
apps/
  api/
    src/
      routes/
        sessions.ts          # CRUD session routes
        files.ts             # Upload / list / delete files
        process.ts           # POST /process — unified pipeline trigger
        review.routes.ts     # GET/PATCH review items; POST /review/bulk
        tax.ts               # POST /tax-inputs; GET annual-summary, audit, disclosure
        export.ts            # GET /export/:year
      middleware/
        auth.ts              # requireAuth; SHAKEPAY_API_KEY
        rate-limiter.js      # global 200 req/min; upload/process route limits
        errorHandler.ts
        validateUpload.ts
  web/
    src/
      api/
        client.ts            # api.bulkReviewDecision(); exponential-backoff retry on 429
      pages/
        ReviewQueuePage.tsx  # CATEGORY_LABELS + CATEGORY_EXPLANATIONS; bulkMutation

packages/
  parser/src/
    detector.ts              # family + language detection (two-stage)
    canonicalizer.ts         # header normalization → CanonicalRow
    headerMap.ts             # HEADER_MAP EN + FR
    typeDictionary.ts        # TYPE_DICTIONARY bilingual values
    helpers.ts               # normalizeAsset + other parse helpers
  classification/src/
    classification-engine.ts
    rule-registry.ts         # ordered priority list
    priority-order.ts        # priority constants
    family-rules/
      crypto.rules.ts        # includes USDC rules, @username SIF detection, two-leg convert
      cash.rules.ts          # includes card_purchase, interac directionality, cash send/receive
      usd.rules.ts           # includes usd_buy_review / usd_sell_review distinction
    description-patterns.ts  # shared DESC_PATTERNS (e.g. convertedAt)
    taxTreatments.ts         # TaxTreatment enum + descriptions
  linking/src/
    heuristics.ts            # getLinkingTimeWindowMs(); configurable via env
    eventLinker.ts           # cross-file event matching; two-leg convert pairing
    matchScorer.ts           # timestamp + amount + type scoring
    linkGroup.ts             # LinkGroup schema + builder
  basis/src/
    assetPool.ts             # BTC/ETH/USD pooled ACB
    acquisitionEngine.ts
    dispositionEngine.ts     # includes native-asset fee parser + spot-rate conversion
    acb-calculator.ts        # processes BTC / ETH / USD only; USDC excluded
    yearBoundary.ts          # opening ACB rollforward per year
  tax/src/
    federalEngine.ts
    provincialEngine.ts      # dispatches to province modules
    provinces/               # one file per jurisdiction
      QC.ts  ON.ts  BC.ts  AB.ts  MB.ts  SK.ts
      NS.ts  NB.ts  NL.ts  PE.ts  YT.ts  NT.ts  NU.ts
    taxLawRegistry.ts        # loads + caches versioned law tables
    inclusionRate.ts         # capital gains inclusion rate by year
  review/src/
    reviewQueue.ts
    review-queue.service.ts  # category assignment; unclassified_event handling
    review-item.ts           # ReviewItem schema + category enum
    overrideLog.ts           # immutable audit of user decisions
  db/src/
    schema/
      index.ts               # Drizzle ORM table definitions — canonical schema
    migrations/
    seeds/
      taxBrackets/           # one JSON per jurisdiction per year
  orchestration/src/
    process-session.service.ts  # computeBasisWithUpdates; pipeline orchestration
  domain/src/
    constants/
      inclusion-rates.ts     # capital gains inclusion rates by year
    enums.ts                 # NormalizedCode, TaxTreatment, and other enums
```

---

## 5. Import Layer

### 5.1 File detection rules

Detection is based entirely on header presence — never on filename.

**Cash summary signature** — requires ALL of:
- `Debit` / `Débit`
- `Credit` / `Crédit`
- `Balance` / `Solde`
- Absence of `Asset Debited` / `Actif débité`

**Crypto / USD summary signature** — requires ALL of:
- `Amount Debited` / `Montant débité`
- `Asset Debited` / `Actif débité`
- `Amount Credited` / `Montant crédité`
- `Asset Credited` / `Actif crédité`
- `Market Value` / `Valeur du marché`
- `Book Cost` / `Coût comptable`

**Language detection** — primary: header names; secondary: known `Type` values.

Unknown schemas must throw a typed `SchemaDetectionError`.

### 5.2 Detection pseudocode

File detection proceeds in two stages. *(Illustrative pseudocode — normative behaviour is defined in `packages/parser/src/detector.ts`.)*

```python
# Stage 1: header-based detection
def detect_family_and_language(headers: list[str]) -> tuple[str, str]:
    hs = set(headers)

    cash_en   = {"Debit", "Credit", "Balance"}
    cash_fr   = {"Débit", "Crédit", "Solde"}
    crypto_en = {"Amount Debited", "Asset Debited", "Amount Credited",
                 "Asset Credited", "Market Value", "Book Cost"}
    crypto_fr = {"Montant débité", "Actif débité", "Montant crédité",
                 "Actif crédité", "Valeur du marché", "Coût comptable"}

    if cash_en <= hs:    return ("cash_summary", "en")
    if cash_fr <= hs:    return ("cash_summary", "fr")
    if crypto_en <= hs:  return ("crypto_or_usd_pending", "en")   # resolved in Stage 2
    if crypto_fr <= hs:  return ("crypto_or_usd_pending", "fr")   # resolved in Stage 2

    raise SchemaDetectionError("Unknown schema")


# Stage 2: asset-content resolution (for files flagged crypto_or_usd_pending)
def resolve_crypto_vs_usd(rows) -> ImportFamily:
    assets = set(row["Asset Debited"] for row in rows if row.get("Asset Debited")) \
           | set(row["Asset Credited"] for row in rows if row.get("Asset Credited"))
    if assets <= {"USD"}:
        return "usd_summary"
    return "crypto_summary"
```

> **Note:** The final `ImportFamily` value assigned to a file is always one of `cash_summary`, `crypto_summary`, or `usd_summary`. The intermediate label `crypto_or_usd_pending` exists only within Stage 1 detection and must **never** be persisted to the database.

### 5.3 Header mapping

```typescript
const HEADER_MAP = {
  en: {
    "Date":                   "timestamp",
    "Type":                   "typeRaw",
    "Description":            "descriptionRaw",
    "Debit":                  "debit",
    "Credit":                 "credit",
    "Balance":                "balance",
    "Amount Debited":         "amountDebited",
    "Asset Debited":          "assetDebited",
    "Amount Credited":        "amountCredited",
    "Asset Credited":         "assetCredited",
    "Market Value":           "marketValueCad",
    "Market Value Currency":  "marketValueCurrency",
    "Book Cost":              "bookCostCad",
    "Book Cost Currency":     "bookCostCurrency",
    "Spot Rate":              "spotRate",
    "Buy / Sell Rate":        "buySellRate",
  },
  fr: {
    "Date":                       "timestamp",
    "Type":                       "typeRaw",
    "Description":                "descriptionRaw",
    "Débit":                      "debit",
    "Crédit":                     "credit",
    "Solde":                      "balance",
    "Montant débité":             "amountDebited",
    "Actif débité":               "assetDebited",
    "Montant crédité":            "amountCredited",
    "Actif crédité":              "assetCredited",
    "Valeur du marché":           "marketValueCad",
    "Devise de valeur du marché": "marketValueCurrency",
    "Coût comptable":             "bookCostCad",
    "Devise du coût comptable":   "bookCostCurrency",
    "Taux au comptant":           "spotRate",
    "Taux d'achat/de vente":      "buySellRate",
  },
};
```

### 5.4 Numeric normalization rules

- Use a decimal-safe library (`decimal.js` or equivalent) — never native `parseFloat` or `Number()` for financial values.
- French locale numbers (e.g. `2 259,81`) must have spaces removed and commas replaced with periods before parsing.
- Empty cells must become `null` — never `0` or `NaN`.
- All parsed numeric fields must be stored as `Decimal` internally.
- Retain original strings for auditability even after canonical numeric parsing.

### 5.5 Asset normalization (`normalizeAsset`)

`normalizeAsset` in `packages/parser/src/helpers.ts` normalizes raw asset strings from `assetDebited` and `assetCredited` columns into canonical uppercase ticker values. This function is the single point where asset identity is established; all downstream classification rules depend on it.

**Supported tickers** (case-insensitive input, uppercase canonical output):

| Input (examples) | Canonical output |
|---|---|
| `BTC`, `btc`, `Btc` | `'BTC'` |
| `ETH`, `eth` | `'ETH'` |
| `USD`, `usd` | `'USD'` |
| `USDC`, `usdc`, `Usdc` | `'USDC'` |
| Any other value | returned as-is (triggers `unknown_review` in classifier) |

**Return type:** `'BTC' | 'ETH' | 'USD' | 'USDC' | string`

**Behaviour contract:**
- Input is trimmed of leading/trailing whitespace before comparison.
- Matching is strictly case-insensitive; no partial matching.
- Unknown tickers are not rejected at this stage — they pass through and are caught by the classification fallback (`unknown_review`).

**Effect on USDC rows:** Before this normalization, `USDC` in an asset column was passed through unnormalized, causing crypto classification rules (which key on exact uppercase strings) to miss the asset and fall through to `unknown_review`. With `normalizeAsset` recognizing `USDC`, the asset fields are set to `'USDC'` and existing rules in `crypto.rules.ts` apply correctly.

---

## 6. Canonical Data Models

### 6.1 `CanonicalRow`

```typescript
type ImportFamily = "cash_summary" | "crypto_summary" | "usd_summary";
type Language     = "en" | "fr";

interface CanonicalRow {
  importId:      string;
  fileId:        string;
  family:        ImportFamily;
  language:      Language;
  rowIndex:      number;

  timestamp:         string;       // ISO 8601
  typeRaw:           string;
  descriptionRaw:    string;

  // Crypto / USD summary fields
  amountDebited?:    Decimal | null;
  assetDebited?:     string  | null;
  amountCredited?:   Decimal | null;
  assetCredited?:    string  | null;
  marketValueCad?:   Decimal | null;
  bookCostCad?:      Decimal | null;
  spotRate?:         Decimal | null;
  buySellRate?:      Decimal | null;

  // Cash summary fields
  debit?:            Decimal | null;
  credit?:           Decimal | null;
  balance?:          Decimal | null;

  raw: Record<string, string>;     // always preserved
}
```

### 6.2 `NormalizedEvent`

```typescript
interface NormalizedEvent {
  eventId:          string;
  linkGroupId?:     string;    // groups related cash + crypto rows
  source:           "shakepay";
  family:           ImportFamily;
  language:         Language;
  fileName:         string;
  rawRowIndex:      number;

  timestamp:        string;
  taxYear:          number;
  typeRaw:          string;
  descriptionRaw:   string;
  typeCanonical:    CanonicalTypeToken;

  normalizedCode:   NormalizedCode;
  taxTreatment:     TaxTreatment;

  assetIn?:         string;
  amountIn?:        Decimal;
  assetOut?:        string;
  amountOut?:       Decimal;

  marketValueCad?:  Decimal;
  bookCostCad?:     Decimal;

  incomeEffectCad:  Decimal;   // amount to include in income
  basisEffectCad:   Decimal;   // change to ACB pool
  gainLossCad?:     Decimal;   // realized gain or loss

  reviewFlag:                boolean;
  reviewReason?:             string;
  policyVersion:             string;  // e.g. "2026-03-29"
  classificationConfidence:  "high" | "medium" | "medium_prudent" | "low_prudent";
}
```

### 6.3 `EconomicEvent`

```typescript
type EconomicEventKind =
  | "cad_to_crypto_buy"
  | "crypto_to_cad_sell"
  | "crypto_reward"
  | "cash_reward"
  | "crypto_send"
  | "crypto_receive"
  | "crypto_swap"
  | "usd_review"
  | "cash_deposit"
  | "cash_withdrawal"
  | "unmatched_review";

interface EconomicEvent {
  economicGroupId:  string;
  eventKind:        EconomicEventKind;
  memberRowIds:     string[];
  timestamp:        string;

  assetIn?:         string;
  amountIn?:        Decimal;
  assetOut?:        string;
  amountOut?:       Decimal;

  marketValueCad?:  Decimal;
  bookCostCad?:     Decimal;

  normalizedCode:   NormalizedCode;
  taxTreatment:     TaxTreatment;
  reviewFlag:       boolean;
}
```

### 6.4 `NormalizedCode` enum

```typescript
type NormalizedCode =
  // Rewards — policy-defined
  | "shakingsats"
  | "referral_bonus"
  | "referral_promo_bonus"
  | "shakesquad_reward"
  | "cad_interest"
  | "card_cashback_btc"
  | "card_intro_bonus_3pct"
  | "sif_received"
  | "sif_sent"
  // Cash events
  | "cad_deposit"
  | "cad_withdrawal"
  | "cad_send"             // CAD send via peer-to-peer transfer (non-SIF)
  | "cad_receive"          // CAD receipt via peer-to-peer transfer (non-SIF, e.g. email counterparty)
  | "card_purchase"        // Shakepay card spending — personal non-deductible outflow
  | "cash_buy_leg"         // provisional reconciliation-only code
  | "cash_sell_leg"        // provisional reconciliation-only code
  // Crypto acquisitions
  | "cad_to_btc"
  | "cad_to_eth"
  // Crypto dispositions
  | "btc_to_cad"
  | "eth_to_cad"
  // Crypto-to-crypto swaps — two-leg model (see §7.4)
  | "convert_out_leg"      // provisional — outgoing leg of a two-row crypto conversion; linked by Event Linker
  | "convert_in_leg"       // provisional — incoming leg of a two-row crypto conversion; linked by Event Linker
  // NOTE: btc_to_eth and eth_to_btc are superseded by the two-leg model above.
  // They are retained in the enum pending migration; new classification rules must not assign them.
  // See open decision in §22.6.
  | "btc_to_eth"           // DEPRECATED — use convert_out_leg / convert_in_leg pair
  | "eth_to_btc"           // DEPRECATED — use convert_out_leg / convert_in_leg pair
  // Crypto transfers (BTC / ETH)
  | "btc_send"
  | "btc_receive"
  | "eth_send"
  | "eth_receive"
  // USDC events
  | "usdc_cad_conversion"   // Send or Receive with "Converted @" description
  | "usdc_send_review"      // On-chain USDC Send (address-based)
  | "usdc_receive_review"   // On-chain USDC Receive (address-based)
  // USD — implementation review codes
  | "usd_receive_review"
  | "usd_send_review"
  | "usd_sell_review"
  | "usd_buy_review"        // CAD → USD purchase — preserved in review ledger pending formal USD policy
  // Fallbacks
  | "unknown_reward_review"
  | "unknown_review";
```

### 6.5 `TaxTreatment` enum

```typescript
type TaxTreatment =
  | "taxable_income"                   // include FMV in income + add basis if crypto
  | "interest_income"                  // interest income
  | "purchase_rebate"                  // no income; add basis to crypto received
  | "non_taxable_review"               // flag for review; do not auto-include
  | "personal_non_deductible_outflow"  // no deduction; informational only
  | "crypto_acquisition"               // ACB increases; no income
  | "taxable_disposition"              // compute gain/loss; reduce ACB
  | "cash_funding_only"                // no direct tax; reconciliation only
  | "cash_withdrawal_only"             // no direct tax; reconciliation only
  | "transfer_or_income_review"        // receive: unknown ownership
  | "transfer_or_disposition_review"   // send: unknown ownership
  | "usd_review"                       // USD event — policy pending
  | "no_tax_effect";                   // neutral
```

---

## 7. Classification Engine

### 7.1 Priority hierarchy

Apply categories in this exact order. The first match wins. Rules live in `packages/classification/src/` — see §30 for code pointers.

| Priority | Category | Rationale |
|---|---|---|
| 1 | `sif_received` / `sif_sent` | Prevent absorption into reward income bucket; detected via `@username` description pattern (primary) or `shake it forward` text (secondary, unconfirmed in production) |
| 1b | `card_purchase` | Flat-rate personal outflow; no ambiguity |
| 2 | `cad_interest` | Clearest treatment — classic interest income |
| 3 | `referral_bonus` / `referral_promo_bonus` | Autonomous conditional bonus |
| 4 | `shakesquad_reward` | Autonomous reward in BTC |
| 5 | `shakingsats` | Primary daily BTC reward |
| 6 | `card_cashback_btc` / `card_intro_bonus_3pct` | Purchase rebate — not income by default |
| 7 | `cad_deposit` / `cad_withdrawal` | Fiat ledger events (includes Interac directionality) |
| 8 | `cad_to_btc` / `cad_to_eth` | Crypto acquisition |
| 9 | `btc_to_cad` / `eth_to_cad` | Crypto disposition |
| 10 | `convert_out_leg` / `convert_in_leg` | Two-leg crypto swap — matched by Event Linker |
| 11 | `usdc_cad_conversion` | USDC Send/Receive matching `converted @` pattern — no tax effect |
| 12 | Send / receive transfers (BTC/ETH on-chain) | Transfer review |
| 13 | USDC on-chain send / receive | USDC transfer review |
| 14 | USD events | Policy review pending |
| 15 | `unknown_reward_review` / `unknown_review` | Final fallback |

### 7.2 Shared description patterns (`description-patterns.ts`)

All description matching must use the shared `DESC_PATTERNS` dictionary. Rules in both `crypto.rules.ts` and `usd.rules.ts` must reference this dictionary rather than defining local regexes.

```typescript
export const DESC_PATTERNS = {
  convertedAt:  /converted\s*@/i,
  sifUsername:  /^@\w+$/,           // primary production SIF signal
  feeNative:    /Fees:\s*([\d.]+)\s*(BTC|ETH)/i,
  usdSoldDesc:  /Sold\s+[\d.]+\s+USD\s+@\s+CA\$/i,
  // ... other shared patterns
};
```

### 7.3 Reward pattern dictionary

All patterns must be case-insensitive regex — not `includes()` string checks. Unknown reward descriptions must route to `unknown_reward_review` and must never be silently dropped.

> **Production SIF format:** The primary description format for SIF transfers observed in real Shakepay exports is a bare `@username` (e.g. `@johsol`, `@moha20`, `@noregretsjohn`). The phrase `shake it forward` has **not** been confirmed in any production export and may be a legacy or internal label. SIF detection must not rely solely on the text-based reward pattern below.
>
> The `@username` detection logic (matching `DESC_PATTERNS.sifUsername`) applies in the classification engine's send/receive handler — not via the reward-pattern dictionary. The reward-pattern dictionary applies only to `Reward`-type rows.

```typescript
const REWARD_PATTERNS: Array<{ re: RegExp; code: NormalizedCode }> = [
  // NOTE: The "shake it forward" text pattern has not been confirmed in production exports.
  // Cash-file and crypto-file SIF detection uses @username + typeCanonical logic — see §7.4 and §7.5.
  { re: /shake\s*it\s*forward/i,              code: "sif_received"          },
  { re: /interest|intérêts/i,                 code: "cad_interest"          },
  { re: /promo.*ref(erral|érencement)/i,      code: "referral_promo_bonus"  },
  { re: /ref(erral|érence|érencement)/i,      code: "referral_bonus"        },
  { re: /shakesquad/i,                        code: "shakesquad_reward"     },
  { re: /shakingsats|secretsats/i,            code: "shakingsats"           },
  { re: /cashback|remise\s*carte/i,           code: "card_cashback_btc"     },
  { re: /3\s*%.*card|card.*3\s*%|intro.*bonus|welcome.*bonus/i,
                                              code: "card_intro_bonus_3pct" },
];
```

> **SecretSats:** The `shakingsats` pattern also matches `secretsats`. Both programs are treated as taxable income at market value with a corresponding BTC cost basis adjustment.

### 7.4 Per-family classification rules

#### Cash summary

| `typeCanonical` | `descriptionRaw` pattern | Direction condition | `normalizedCode` | `taxTreatment` |
|---|---|---|---|---|
| `interac` | any | `credit` is non-null | `cad_deposit` | `no_tax_effect` |
| `interac` | any | `debit` is non-null | `cad_withdrawal` | `no_tax_effect` |
| `interac` | any | both null | *(unclassified)* | → `unknown_review` / `unclassified_event` ReviewItem |
| `buy` | any | — | `cash_buy_leg` *(provisional)* | `cash_funding_only` |
| `sell` | any | — | `cash_sell_leg` *(provisional)* | `cash_withdrawal_only` |
| `card_purchase` | any | — | `card_purchase` | `personal_non_deductible_outflow` |
| `receive` | matches `^@\w+$` (username format) | — | `sif_received` | `no_tax_effect` |
| `receive` | email address format | — | `cad_receive` | `no_tax_effect` |
| `receive` | any other | — | `unknown_reward_review` | `non_taxable_review` |
| `send` | matches `^@\w+$` or email address format | — | `sif_sent` | `personal_non_deductible_outflow` |
| `send` | any other | — | `unknown_review` | `non_taxable_review` |
| `reward` | referral pattern | — | `referral_bonus` | `taxable_income` |
| `reward` | promo referral pattern | — | `referral_promo_bonus` | `taxable_income` |
| `reward` | interest pattern | — | `cad_interest` | `interest_income` |
| `reward` | unknown | — | `unknown_reward_review` | `non_taxable_review` |
| `withdrawal` | any | — | `cad_withdrawal` | `no_tax_effect` |

> **Interac directionality.** Direction is determined by which of `debit` or `credit` is populated for the row. A row where both are null must generate an `unclassified_event` `ReviewItem`.

> **Cash-file Send/Receive vs. crypto-file Send/Receive.** `Send` and `Receive` appear in **both** the cash summary and the crypto summary. In the cash file they represent peer-to-peer CAD transfers (SIF or email-based). In the crypto file they represent on-chain asset transfers. The classification engine must apply the appropriate rule set based on file family.

> **`@username` SIF format.** The `^@\w+$` pattern (a bare `@` followed by alphanumeric/underscore characters with no spaces or other text) is the production description for Shake It Forward peer-to-peer transfers within the Shakepay platform. It is distinct from Interac e-Transfer rows, which use `Interac e-Transfer` as the `Type` value regardless of description. The SIF regex in §7.3 (`shake\s*it\s*forward`) applies only to `Reward`-type rows in the crypto summary file and has not been confirmed in production exports. Cash-file SIF detection uses `typeCanonical = receive` or `typeCanonical = send` combined with the `@username` description pattern — not the §7.3 regex.

> **`cad_send` scope.** The `cad_send` code is currently reachable only via the email-address send path. Non-`@username`, non-email cash sends fall to `unknown_review` until their population is better understood. See open decision §22.6.

#### Crypto summary

| `typeCanonical` | Asset context | `normalizedCode` | `taxTreatment` |
|---|---|---|---|
| `buy` | credited: BTC | `cad_to_btc` | `crypto_acquisition` |
| `buy` | credited: ETH | `cad_to_eth` | `crypto_acquisition` |
| `sell` | debited: BTC | `btc_to_cad` | `taxable_disposition` |
| `sell` | debited: ETH | `eth_to_cad` | `taxable_disposition` |
| `convert` | Asset Debited present, Asset Credited null | `convert_out_leg` *(provisional)* | `taxable_disposition` |
| `convert` | Asset Credited present, Asset Debited null | `convert_in_leg` *(provisional)* | `crypto_acquisition` |
| `reward` | ShakingSats/SecretSats desc | `shakingsats` | `taxable_income` |
| `reward` | ShakeSquad desc | `shakesquad_reward` | `taxable_income` |
| `reward` | cashback desc | `card_cashback_btc` | `purchase_rebate` |
| `reward` | 3% desc | `card_intro_bonus_3pct` | `purchase_rebate` |
| `reward` | SIF desc | `sif_received` | `no_tax_effect` |
| `send` | debited: USDC, desc matches `convertedAt` | `usdc_cad_conversion` | `no_tax_effect` |
| `receive` | credited: USDC, desc matches `convertedAt` | `usdc_cad_conversion` | `no_tax_effect` |
| `send` | debited: USDC, on-chain (address in desc) | `usdc_send_review` | `transfer_or_disposition_review` |
| `receive` | credited: USDC, on-chain (address in desc) | `usdc_receive_review` | `transfer_or_income_review` |
| `send` | debited: BTC, desc matches `^@\w+$` | `sif_sent` | `personal_non_deductible_outflow` |
| `receive` | credited: BTC, desc matches `^@\w+$` | `sif_received` | `no_tax_effect` |
| `send` | debited: BTC | `btc_send` | `transfer_or_disposition_review` |
| `send` | debited: ETH | `eth_send` | `transfer_or_disposition_review` |
| `receive` | credited: BTC | `btc_receive` | `transfer_or_income_review` |
| `receive` | credited: ETH | `eth_receive` | `transfer_or_income_review` |

> **Two-row Convert model.** Conversion events in the crypto summary file are emitted as **two separate rows** sharing the same timestamp: one row records the outgoing asset debit (Asset Debited present, Asset Credited null); one row records the incoming asset credit (Asset Credited present, Asset Debited null). Both carry `Type = Convert`. The earlier single-row assumption (`btc_to_eth`, `eth_to_btc`) does not match production exports. The Event Linker (§8.7) is responsible for pairing `convert_out_leg` and `convert_in_leg` rows into a single `crypto_swap` economic event. A `convert_out_leg` without a corresponding `convert_in_leg` within the timestamp window must generate a `ReviewItem` with category `unclassified_event` and severity `warning`.

> **USD-priced sells.** Some `Sell` rows have USD-denominated proceeds in the description (`Sold @ US$…`). These are still classified as `btc_to_cad` or `eth_to_cad` because Shakepay populates `Market Value` in CAD for all rows, and this CAD value is used as proceeds. The USD price in the description is informational only. Description-matching logic for parsing proceeds must not assume a `CA$` prefix — it must fall back to `bookCostCad` (or `marketValueCad`) when the description does not match the `CA$` pattern. See §9.5.

> **USDC conversion rule priority:** `usdc_cad_conversion` runs at priority 11 — before on-chain transfer rules — so that wallet-plumbing legs are auto-resolved without creating review items.

> **`@username` SIF detection in crypto file.** `Send` and `Receive` rows in the crypto file with descriptions matching `^@\w+$` are classified as `sif_sent` / `sif_received` at a priority above the standard on-chain transfer rules. This prevents them from generating ownership-review items.

> **SIF received:** The implemented treatment is `no_tax_effect` (see Decision Log §22.1), not `non_taxable_review` as originally specified.

#### USD summary

| `typeCanonical` | Asset context | `normalizedCode` | `taxTreatment` |
|---|---|---|---|
| `receive` | credited: USD | `usd_receive_review` | `usd_review` |
| `send` | debited: USD | `usd_send_review` | `usd_review` |
| `sell` | `Amount Credited = USD` non-null, `Amount Debited` null, description null | `usd_buy_review` | `usd_review` |
| `sell` | `Amount Debited = USD` non-null, description matches `Sold … USD @ CA$…` | `usd_sell_review` | `usd_review` |

> **USD `Sell` sub-type distinction.** In the USD summary file, a `Sell` row with null description and USD credited represents the user **purchasing** USD (selling CAD). This is semantically a CAD-to-USD acquisition, not a USD disposal. It must not be labelled as a USD sell in the review queue. The `suggestedActions` in the review item must distinguish these two patterns clearly (see §13.3).

### 7.5 Classification pseudocode

*(Illustrative pseudocode — normative behaviour is defined in the rule registry.)*

```python
def classify_transaction(tx):
    d        = tx.description_raw.lower() if tx.description_raw else ""
    desc_raw = tx.description_raw.strip() if tx.description_raw else ""

    # Priority 1 — SIF
    # Primary production signal: @username description (cash or crypto file send/receive).
    # "shake it forward" text pattern retained as secondary signal; not confirmed in production.
    is_sif_desc = re.search(r"shake\s*it\s*forward", d) or re.match(r"^@\w+$", desc_raw)
    if is_sif_desc:
        if tx.type_canonical == "send":
            return code("sif_sent", "personal_non_deductible_outflow")
        return code("sif_received", "no_tax_effect")  # see Decision Log §22.1

    # Priority 1b — Card purchase (cash file only)
    if tx.type_canonical == "card_purchase":
        return code("card_purchase", "personal_non_deductible_outflow")

    # Priority 2 — Interest
    if re.search(r"interest|intérêts", d):
        return code("cad_interest", "interest_income")

    # Priority 3 — Referral
    if re.search(r"promo.*(referral|référencement)", d):
        return code("referral_promo_bonus", "taxable_income")
    if re.search(r"ref(erral|érence|érencement)", d):
        return code("referral_bonus", "taxable_income")

    # Priority 4 — ShakeSquad
    if re.search(r"shakesquad", d):
        return code("shakesquad_reward", "taxable_income")

    # Priority 5 — ShakingSats / SecretSats
    if re.search(r"shakingsats|secretsats", d):
        return code("shakingsats", "taxable_income")

    # Priority 6 — Card cashback
    if re.search(r"cashback|remise\s*carte", d):
        return code("card_cashback_btc", "purchase_rebate")
    if re.search(r"3\s*%.*card|intro.*bonus", d):
        return code("card_intro_bonus_3pct", "purchase_rebate")

    return classify_standard_transaction(tx)


def classify_standard_transaction(tx):
    # Cash file — Interac directionality
    if tx.type_canonical == "interac":
        if tx.credit is not None:  return code("cad_deposit",   "no_tax_effect")
        if tx.debit  is not None:  return code("cad_withdrawal", "no_tax_effect")
        return code("unknown_review", "non_taxable_review", review=True)

    # Cash file — Receive (non-Reward)
    if tx.family == "cash_summary" and tx.type_canonical == "receive":
        if re.match(r"^@\w+$", tx.description_raw.strip() if tx.description_raw else ""):
            return code("sif_received", "no_tax_effect")
        if is_email(tx.description_raw):
            return code("cad_receive", "no_tax_effect")
        return code("unknown_reward_review", "non_taxable_review", review=True)

    # Cash file — Send (non-Reward)
    if tx.family == "cash_summary" and tx.type_canonical == "send":
        desc = tx.description_raw.strip() if tx.description_raw else ""
        if re.match(r"^@\w+$", desc) or is_email(desc):
            return code("sif_sent", "personal_non_deductible_outflow")
        return code("unknown_review", "non_taxable_review", review=True)

    # Crypto file — Convert two-leg model
    if tx.type_canonical == "convert":
        if tx.asset_debited and not tx.asset_credited:
            return code("convert_out_leg", "taxable_disposition")
        if tx.asset_credited and not tx.asset_debited:
            return code("convert_in_leg", "crypto_acquisition")
        # Both or neither populated — unexpected
        return code("unknown_review", "non_taxable_review", review=True)

    # Priority 11 — USDC conversion leg (before on-chain transfer rules)
    if tx.asset in ("USDC",) and re.search(DESC_PATTERNS["convertedAt"], tx.description_raw or ""):
        return code("usdc_cad_conversion", "no_tax_effect", review=False)

    # Priority 12–13 — BTC/ETH and USDC on-chain transfers
    if tx.type_canonical == "send":
        if tx.asset_debited == "BTC":    return code("btc_send",          "transfer_or_disposition_review", review=True)
        if tx.asset_debited == "ETH":    return code("eth_send",          "transfer_or_disposition_review", review=True)
        if tx.asset_debited == "USDC":   return code("usdc_send_review",  "transfer_or_disposition_review", review=True)
    if tx.type_canonical == "receive":
        if tx.asset_credited == "BTC":   return code("btc_receive",         "transfer_or_income_review", review=True)
        if tx.asset_credited == "ETH":   return code("eth_receive",         "transfer_or_income_review", review=True)
        if tx.asset_credited == "USDC":  return code("usdc_receive_review", "transfer_or_income_review", review=True)

    # Priority 14 — USD events
    if tx.asset in ("USD",):
        return classify_usd(tx)

    # Priority 15 — Fallback
    return code("unknown_review", "non_taxable_review", review=True)
```

### 7.6 Understanding extraction vs. classification

A common source of confusion is interpreting a crowded review queue as evidence that the engine failed to read the CSV. In practice, the two concepts are distinct.

**Extraction** refers to the import and canonicalization stages: detecting the file family, mapping headers, parsing types and amounts, and producing `CanonicalRow` records. If a transaction appears in the audit trail with a correct timestamp, asset, and quantity, extraction succeeded.

**Classification** is what happens after extraction. Many Shakepay rows are ambiguous for Canadian tax purposes even when every field is parsed correctly. The review queue answers: *"We have the row; we are not sure of the tax story without you."*

| Question | Check in CSV | Check in audit trail |
|---|---|---|
| Was the row imported? | Row exists in export | Event exists for that date/type |
| Was type parsed? | `Type` column value | `typeCanonical` / event type |
| Was amount parsed? | Debit/credit/market value | `quantity`, `amountCad`, etc. |
| Is tax treatment final? | N/A — not in CSV | `taxTreatment`, `normalizedCode`, `reviewFlag` |

Disagreement between CSV and audit trail on **amounts or dates** suggests extraction or mapping bugs. Disagreement on **whether review is required** is classification policy, not extraction failure.

> **Operational note:** This distinction gives support and engineering a clean triage model. If timestamps, assets, and quantities are present in the audit trail, import worked. If treatment remains unresolved, the engine is asking for tax context rather than failing to parse.

---

## 8. Event Linking & Reconciliation

### 8.1 Why reconciliation is required

A single economic buy event appears in two files:
- **Cash file:** CAD debit for the purchase (`cash_buy_leg`)
- **Crypto file:** BTC or ETH credit for the same purchase (`cad_to_btc` / `cad_to_eth`)

Without linking, the engine double-counts acquisitions and distorts ACB.

> **This is the most important correctness requirement in the entire engine.**

**Important:** Cash and crypto rows are **linked** via `linkGroupId` — they are **not** merged into a single database row. Both rows are preserved as separate `NormalizedEvent` records; the link group is the join. This design keeps the raw data immutable and preserves the audit chain.

> **Schema-level enforcement required:** The event-linking model is conceptually sound, but the operational guarantee must not rest on application logic alone. The database layer must enforce the intended relationships directly. At minimum, link-group membership must be stored in an explicit join table with referential integrity, and the schema must prevent orphaned rows, duplicate membership entries, and contradictory event assignments. See §16.3.

The session MUST NOT advance to `CALCULATED` without at least one crypto summary file unless all events in scope are cash-only. Attempting to process a session with crypto-indicating signals but no crypto file MUST return `MissingRequiredFileError` (HTTP 422). See §3.2.

### 8.2 Matching algorithm

For each provisional `cash_buy_leg` row, search for a candidate crypto buy row matching:

| Signal | Condition | Weight |
|---|---|---|
| Timestamp proximity | Within the configured time window (default ±120 s) | High |
| Amount match | `abs(cashDebit − bookCostCad) ≤ 0.05` (fallback to `marketValueCad` if `bookCostCad` is null) | High |
| Type pair | `cash_buy_leg` ↔ `cad_to_btc` or `cad_to_eth` | Required |
| Asset | Crypto credited is BTC or ETH | Medium |

> **Book Cost priority in linker.** The linker's CAD amount comparison uses `bookCostCad` from the crypto row where present, falling back to `marketValueCad`. This is consistent with the Book Cost priority established in §9.2. The matching tolerance applies to the same value the ACB engine will use, preventing a match that succeeds at linking but diverges at basis computation.

The time window is configurable via environment variable — see §26. Increase it if your exports show clock skew or delayed settlements.

**Conflict resolution:**
1. Prefer exact timestamp match
2. Then exact value match
3. Then highest composite score
4. Multiple candidates with equal score → `unmatched_review`

### 8.3 Linked event output

When a match is confirmed, both rows receive the same `linkGroupId`. The canonical acquisition is:

```
assetIn:         CAD
amountIn:        cash debit amount
assetOut:        BTC or ETH
amountOut:       crypto credited amount
bookCostCad:     from crypto file's Book Cost field
marketValueCad:  from crypto file's Market Value field
```

### 8.4 Handling unlinked rows

| Scenario | Action |
|---|---|
| Cash buy leg, no matching crypto row | `ReviewItem` — `cash_inflow_review` |
| Crypto buy leg, no matching cash row | Use `bookCostCad` from crypto file; no flag if present |
| Cash reward, no companion row | Normal — cash rewards are cash-file-only events |
| Crypto send, no matching receive | `ReviewItem` — `transfer_or_disposition_review` |

### 8.5 `LinkGroup` schema

```typescript
interface LinkGroup {
  linkGroupId:     string;
  groupType:
    | "cad_crypto_buy"
    | "cad_crypto_sell"
    | "crypto_swap"
    | "reward_only"
    | "usd_flow_cluster"   // USD/USDC legs grouped for review clarity
    | "review";
  memberEventIds:  string[];
  canonicalAcquisition?: {
    timestamp:   string;
    assetOut:    string;
    amountOut:   Decimal;
    costCad:     Decimal;
  };
  canonicalDisposition?: {
    timestamp:    string;
    assetIn:      string;
    amountIn:     Decimal;
    proceedsCad:  Decimal;
    acbCad:       Decimal;
    gainLossCad:  Decimal;
  };
}
```

> **`usd_flow_cluster`:** USD and USDC legs that share timing and amounts may be grouped into a `usd_flow_cluster` link group. This surfaces them as a unit in the review queue rather than as isolated unrelated items.

### 8.6 Internal transfers (send/receive between own wallets)

When a user moves crypto between accounts they own, this is a non-disposition event. The engine carries the existing ACB forward unchanged. No gain or loss is recorded at the time of the transfer. Gain or loss is recognized only when the asset is subsequently disposed of in a taxable event.

Because ownership context cannot be proven automatically, all `btc_send`, `eth_send`, `btc_receive`, `eth_receive`, `usdc_send_review`, and `usdc_receive_review` events default to `transfer_or_disposition_review` or `transfer_or_income_review` until the user confirms them.

> **Design strength — review-first ownership:** Defaulting all on-chain sends and receives into review-first ownership categories is one of the strongest tax-safety decisions in this design. This default must be maintained even as heuristic auto-classification (see §13.9) is added.

When a user confirms a `btc_receive` or `eth_receive` as `internal_transfer`, the event is reclassified to `no_tax_effect` and excluded from basis computation. When confirmed as `taxable_income`, market value at receipt is used as cost basis. Any decision that changes a basis-affecting classification triggers full recomputation from the earliest impacted timestamp — see §22.2.

### 8.7 Crypto-to-crypto conversion linking

A `Convert` event in the crypto summary file is emitted as **two rows** sharing the same timestamp: one debit-only leg (`convert_out_leg`) and one credit-only leg (`convert_in_leg`). The Event Linker must match these two rows using:

- `typeCanonical = convert` on both rows
- Timestamp proximity within the configured window (default ±120 s)
- The debited asset on one row does not appear as the credited asset on the same row

When matched, both rows are assigned the same `linkGroupId` and grouped into a `crypto_swap` economic event. The outgoing leg supplies `proceedsCad` (from `bookCostCad` or `marketValueCad`); the incoming leg supplies the new asset quantity and basis. Both `canonicalDisposition` (from the out-leg) and `canonicalAcquisition` (from the in-leg) must be populated in the `LinkGroup`.

If no match is found within the time window, both unmatched legs independently generate `ReviewItem`s under category `unclassified_event`, severity `warning`.

---

## 9. ACB Ledger Engine

### 9.1 Basis method

Use **pooled average cost basis** per asset — the standard Canadian method for individuals. Maintain **separate pools** for BTC, ETH, and USD. USDC does not have its own pool — see §9.7.

```typescript
interface AssetPool {
  asset:          "BTC" | "ETH" | "USD";
  quantity:       Decimal;
  totalAcbCad:    Decimal;
}
```

Computed property: `perUnitAcb = totalAcbCad / quantity`

**Superficial loss rule:** Not yet implemented. This is a near-term release risk — see §21.7 for full discussion.

### 9.2 Book cost vs. market value (Shakepay spread)

Shakepay exports include both a `Book Cost` and a `Market Value` column. The ACB engine uses them with the following priority:

**Crypto acquisitions:** Adjusted cost base uses **Book Cost** from the crypto CSV when present; otherwise Market Value.

**Crypto dispositions (BTC/ETH sells and swap outgoing legs):** Proceeds default to **Book Cost** first, then Market Value, so CAD legs from the cash file stay consistent with the crypto file where Shakepay reports both.

This priority reflects the fact that Shakepay's `Book Cost` is the value actually transacted at the buy/sell rate, while `Market Value` may include the spread.

> **Design strength — Book Cost priority:** Preferring Book Cost for acquisitions and, where applicable, for disposition proceeds is the correct default for a Shakepay-only engine. This choice and its reasoning must be disclosed in every generated report — see §20.4.

### 9.3 Network fees

Fees on `Send` rows are expressed in the **native crypto asset** (BTC or ETH) in the description field, using the format `Fees: <quantity> <ASSET>.` *(Example: `Fees: 0.000008 BTC.`, `Fees: 0.00013549 ETH.`)*

The engine must:
1. Parse the fee quantity and asset using the pattern `Fees:\s*([\d.]+)\s*(BTC|ETH)` *(illustrative regex — normative behaviour is defined in the fee-parser module).*
2. Convert to CAD: `feeCad = feeQuantity × spotRate`, where `spotRate` is the `Spot Rate` field from the same row.
3. Reduce net disposition proceeds by `feeCad`.

If `Spot Rate` is null for the row, the fee cannot be converted to CAD. The engine must flag the audit trail entry with a `missing_fee_valuation` warning and must **not** silently omit the fee or substitute zero.

Confirm large or ambiguous fees with a tax professional.

### 9.4 Acquisition events

| Source | Income effect | ACB effect |
|---|---|---|
| Taxable reward in crypto | `incomeEffectCad += fmvCad` | `totalAcbCad += fmvCad` |
| Cashback / rebate in crypto | None | `totalAcbCad += fmvCad` |
| CAD-to-crypto buy | None | `totalAcbCad += bookCostCad (or marketValueCad)` |
| Crypto-to-crypto swap (incoming leg) | None | `totalAcbCad += proceedsOfOutgoingLeg` |
| Confirmed internal transfer | None | Preserve existing basis only |
| Unconfirmed receive | None | `ReviewItem` — review required |
| USDC conversion leg (`usdc_cad_conversion`) | None | No ACB effect — excluded from pools |
| USDC on-chain receive (`usdc_receive_review`) | None | `ReviewItem` — excluded from pools pending decision |

### 9.5 Disposition logic

```
perUnitAcb       = pool.totalAcbCad / pool.quantity
acbAllocated     = perUnitAcb × quantityDisposed
netProceeds      = proceedsCad − feeCad   (fee converted from native asset via spotRate — see §9.3)
gainLoss         = netProceeds − acbAllocated

pool.quantity    -= quantityDisposed
pool.totalAcbCad -= acbAllocated
```

**Proceeds source rule for USD-priced sells.** When a `Sell` row description contains a USD price (`Sold @ US$…`), proceeds MUST be taken from `bookCostCad` (or `marketValueCad` as fallback) — not from the USD figure in the description. Applying the USD amount directly as CAD proceeds would overstate or understate the gain/loss depending on the exchange rate. Shakepay populates `Market Value` in CAD for all rows regardless of the description's price denomination.

### 9.6 Crypto-to-crypto swap (two-step)

*(Illustrative pseudocode — normative behaviour is defined in the swap-processing module.)*

```python
def process_swap(from_asset, from_qty, to_asset, to_qty, cad_value, ledger):
    # Step 1 — dispose outgoing asset (convert_out_leg)
    acb_allocated = ledger.allocate_acb(from_asset, from_qty)
    gain_loss = cad_value - acb_allocated
    ledger.record_disposition(from_asset, from_qty, cad_value, acb_allocated)

    # Step 2 — acquire incoming asset at same CAD value (convert_in_leg)
    ledger.add_basis(to_asset, to_qty, cad_amount=cad_value)
```

Both legs are supplied by the Event Linker after pairing the `convert_out_leg` and `convert_in_leg` rows — see §8.7.

### 9.7 USDC exclusion

`acb-calculator.ts` processes BTC, ETH, and USD only. USDC rows (`usdc_cad_conversion`, `usdc_send_review`, `usdc_receive_review`) are intentionally excluded from all ACB pool logic. Any future decision to include USDC in gain/loss calculations requires an explicit basis-pool extension, a policy decision, and a migration path for existing sessions.

### 9.8 Year boundary / rollforward

- Opening quantity and ACB for year Y are computed from **all prior events**, not just the prior year.
- Current-year dispositions must use prior-year ACB where applicable.
- If historical basis is incomplete, mark `completenessStatus = "materially_incomplete"` and label the gain/loss as estimated.
- Negative pool quantity must generate a `ReviewItem` with `severity: "blocking"` — see §13.5 for the consolidated negative-pool rule.

> **Year-boundary timezone rule (normative).** All tax-year assignment and year-boundary ACB rollforward computations use the `timestamp` field as recorded in the Shakepay export, interpreted as **UTC**. Shakepay timestamps are stored in UTC; no local timezone conversion is applied by the engine. If a future Shakepay export format introduces local timestamps, this rule must be revisited. Integration tests for year-boundary rows must use timestamps near `2024-12-31T23:59:59Z` and `2025-01-01T00:00:00Z` to verify correct year assignment.

### 9.9 Acquisition and disposition pseudocode

*(Illustrative pseudocode — normative behaviour is defined in the basis package.)*

```python
def process_crypto_receipt(tx, is_taxable, ledger):
    cost_cad = tx.book_cost_cad or tx.market_value_cad
    qty   = tx.amount_credited
    asset = tx.asset_credited

    if is_taxable:
        ledger.income += cost_cad

    ledger.add_basis(asset=asset, qty=qty, cad_amount=cost_cad)


def process_disposition(tx, asset, qty_disposed, proceeds_cad, fee_qty, fee_asset, spot_rate, ledger):
    fee_cad       = (fee_qty * spot_rate) if spot_rate else None   # native asset → CAD
    net_proceeds  = proceeds_cad - (fee_cad or 0)
    acb_allocated = ledger.allocate_acb(asset, qty_disposed)
    gain_loss     = net_proceeds - acb_allocated

    ledger.record_capital_event({
        "asset":       asset,
        "qty":         qty_disposed,
        "proceeds":    net_proceeds,
        "acb":         acb_allocated,
        "gain_loss":   gain_loss,
        "fee_cad":     fee_cad,
    })
```

---

## 10. Annual Tax Calculation

### 10.1 Calculation flow

For each selected tax year:

1. **Filter** income events and capital events for that calendar year only. All year-boundary computations use UTC timestamps (see §9.8).
2. **Compute opening ACB** per asset from all prior history
3. **Aggregate reward income** — `shakingsats` + `referral_bonus` + `referral_promo_bonus` + `shakesquad_reward`
4. **Aggregate interest income** — `cad_interest`
5. **Aggregate dispositions** — all `taxable_disposition` events in the year
6. **Compute** net capital gain/loss and taxable capital gain
7. **Feed** totals into federal and provincial tax estimators

### 10.2 Taxable capital gain calculation

```
taxableCapitalGain = max(netCapitalGain, 0) × inclusionRate(taxYear, lawVersion)
```

**Current inclusion rates** (source: `packages/domain/src/constants/inclusion-rates.ts`):

| Tax years | Rate | Status |
|---|---|---|
| 2021–2026 | **50%** | In effect. The June 2024 federal budget proposed a 66.67% rate above $250,000 net gain; Finance Canada announced a deferral of that proposal on January 31, 2025; the CRA subsequently stated it was administering the currently enacted one-half rate; and the proposed increase was later cancelled. The enacted 50% rate applies for all years in scope. |

Inclusion rates must be loaded from `packages/domain/src/constants/inclusion-rates.ts` and the active `TaxLawVersion` record for the selected year. Do not hardcode any rate in business logic, tests, or report templates.

**Minimum stored metadata per annual calculation:**
- `federalTaxLawVersionId`
- `provincialTaxLawVersionId`
- `inclusionRateApplied`
- `lawDataLoadedAt`
- `lawDataSourceUrls`

### 10.3 `AnnualSummary` schema

```typescript
interface AnnualSummary {
  taxYear:    number;
  province:   JurisdictionCode;

  taxableAmounts: {
    rewardIncomeCad:         Decimal;
    interestIncomeCad:       Decimal;
    capitalGainsCad:         Decimal;
    capitalLossesCad:        Decimal;
    netCapitalGainCad:       Decimal;
    taxableCapitalGainCad:   Decimal;
    purchaseRebatesCad:      Decimal;
    nonTaxableReviewCad:     Decimal;
    usdLedgerCadEquiv:       Decimal;
  };

  taxEstimates: {
    federalTaxEstimateCad:    Decimal;
    provincialTaxEstimateCad: Decimal;
    totalTaxEstimateCad:      Decimal;
    effectiveMarginalRate?:   Decimal;
    mode:                     "standalone" | "marginal";
    note:                     string;
  };

  fileCoverage: {
    cryptoPresent:          boolean;
    cashPresent:            boolean;
    usdPresent:             boolean;
    languages:              Language[];
    yearCoveragePct:        number;
    completenessStatus:     CompletenessStatus;
    reviewItemCount:        number;
  };

  quebecDisclosureRequired?: boolean;
  federalTaxLawVersionId:    string;
  provincialTaxLawVersionId: string;
  policyVersion:             string;
  classifierVersion:         string;
  computedAt:                string;
  assumptions:               string[];
}
```

---

## 11. Tax Estimation Engine

### 11.1 Required inputs and outputs

```typescript
interface TaxEngineInput {
  taxYear:                        number;
  province:                       JurisdictionCode;
  otherTaxableIncomeCad?:         Decimal;
  shakepayOrdinaryIncomeCad:      Decimal;
  shakepayInterestIncomeCad:      Decimal;
  shakepayTaxableCapitalGainCad:  Decimal;
  mode:                           "standalone" | "marginal";
}

interface TaxEngineOutput {
  federalTaxEstimateCad:    Decimal;
  provincialTaxEstimateCad: Decimal;
  totalTaxEstimateCad:      Decimal;
  effectiveMarginalRate?:   Decimal;
  assumptions:              string[];
}
```

### 11.2 Calculation modes

**Standalone mode** — assumes Shakepay amounts are the only income being taxed. Useful for quick estimates when the user does not provide other income.

**Marginal mode** — user provides other annual income; the engine calculates the incremental tax caused by Shakepay activity. This is the more accurate option and should be the default. Province and other income are submitted via `POST /api/v1/sessions/:id/tax-inputs`.

### 11.3 Table-driven requirement

The tax estimator must be **table-driven** for both federal and provincial or territorial tax. Do not embed rate tables, thresholds, surtaxes, reductions, credits, or inclusion-rate assumptions directly in estimator code.

### 11.4 Supported jurisdictions

| Code | Province / Territory |
|---|---|
| `NL` | Newfoundland and Labrador |
| `PE` | Prince Edward Island |
| `NS` | Nova Scotia |
| `NB` | New Brunswick |
| `QC` | Québec |
| `ON` | Ontario |
| `MB` | Manitoba |
| `SK` | Saskatchewan |
| `AB` | Alberta |
| `BC` | British Columbia |
| `YT` | Yukon |
| `NT` | Northwest Territories |
| `NU` | Nunavut |

### 11.5 Tax-law architecture

Tax-law data must live in versioned records and seed files, not in TypeScript constants.

```
packages/db/src/seeds/taxBrackets/
  FED-2025.json
  QC-2025.json
  ON-2025.json
  BC-2025.json
  AB-2025.json
  ...
```

`taxLawRegistry.ts` loads and caches active law records by `jurisdiction-year` key. `provincialEngine.ts` dispatches via a registry rather than inline branching. Adding a new jurisdiction requires only adding a JSON seed file.

### 11.6 `TaxLawVersion` schema

```typescript
interface TaxLawVersion {
  id:              string;
  taxYear:         number;
  jurisdiction:    JurisdictionCode | "FED";
  sourceUrl:       string;
  effectiveFrom:   string;
  effectiveTo?:    string;
  inclusionRate:   Decimal;
  brackets:        TaxBracket[];
  surtaxes?:       SurtaxRule[];
  reductions?:     ReductionRule[];
  metadata:        Record<string, string>;
}

interface TaxBracket {
  lowerBound:  Decimal;
  upperBound?: Decimal;
  rate:        Decimal;
}
```

### 11.7 Tax law admin layer

An explicit Tax Law Admin layer is required so tax data can be maintained without redeploying business logic. Responsibilities include: loading official yearly tables and validating schema before activation; marking one law version active per jurisdiction-year; patching edge cases through override profiles; exposing the exact law versions used in each report; and providing a comparison view between two law versions.

---

## 12. Québec Module

### 12.1 Legal requirement

Since the 2024 tax year, Revenu Québec requires form **TP-21.4.39** — *Déclaration relative aux cryptoactifs* — when a Québec taxpayer possesses, receives, disposes of, or uses crypto-assets. This obligation applies even in years with no taxable gain.

### 12.2 Product behavior

When `province = "QC"` and `taxYear >= 2024`:
- If any crypto possession, receipt, disposition, or use activity exists → set `quebecDisclosureRequired = true`
- Display a TP-21.4.39 reminder in the UI
- Generate the Québec disclosure support report

CAD-only rows must **not** be treated as crypto events for TP-21.4.39 purposes.

> **Design strength — first-class Québec support:** Québec is treated as a first-class jurisdiction throughout this engine, not a federal-model afterthought.

### 12.3 Québec rate-table handling

Québec tax calculations must be table-driven and versioned. The module resolves the active `QC` tax-law version for the selected year and persists the exact law-version identifier in the annual summary and disclosure support output.

### 12.4 TP-21.4.39 support report

Key form line mappings (from the current version of the form):
- **Line 49:** Taxable capital gain from crypto activity
- **Line 135:** Total taxable amount from crypto activity

These mappings must be maintained in a versioned form-mapping configuration (not hardcoded in core business logic).

```typescript
interface QuebecDisclosure {
  taxYear:               number;
  possessedCrypto:       boolean;
  transactedCrypto:      boolean;
  openingHoldings:       Array<{ asset: string; quantity: Decimal; acbCad: Decimal }>;
  receipts:              Array<{ asset: string; quantity: Decimal; valueCad: Decimal }>;
  dispositions:          Array<{ asset: string; quantity: Decimal; proceedsCad: Decimal }>;
  uses:                  Array<{ asset: string; quantity: Decimal; valueCad: Decimal }>;
  closingHoldings:       Array<{ asset: string; quantity: Decimal; acbCad: Decimal }>;
  usdActivityReviewFlag: boolean;
  completenessStatus:    CompletenessStatus;
  policyVersion:         string;
  notes:                 string[];
}
```

---

## 13. Review Queue

### 13.1 What belongs in the review queue

The review queue surfaces normalized events where classification is ambiguous for Canadian tax purposes — even if every field was parsed correctly. It answers: *"We have the row; we are not sure of the tax story without you."*

| Fact in CSV | What the CSV does not say | Why review may be required |
|---|---|---|
| Crypto **Send** to an address | Whether that address is the user's own wallet | Transfer to self vs. third-party payment |
| Crypto **Receive** from an address | Whether funds are an internal transfer or income | Different tax character |
| **USD** legs | How to treat FX and intermediary USD | Policy pending |
| **USDC** on-chain send/receive | Whether address belongs to user | Same ownership-ambiguity logic as BTC/ETH |
| **Cash Buy** without a matching crypto **Buy** | Whether buy failed, is delayed, or files are incomplete | Unlinked cash buy |

### 13.2 Trigger conditions

| Trigger | Severity |
|---|---|
| Negative asset pool quantity | `blocking` |
| Contradictory duplicate rows | `blocking` |
| Unknown reward description | `warning` |
| Unresolved `btc_send` / `eth_send` | `warning` |
| Unresolved `btc_receive` / `eth_receive` | `warning` |
| Unresolved `usdc_send_review` / `usdc_receive_review` | `warning` |
| Unmatched buy legs | `warning` |
| Incomplete prior-year basis | `warning` |
| USD review codes present | `info` |
| `unknown_review` (no rule matched) | `warning` — category: `unclassified_event` |
| Missing CAD valuation | `warning` |
| Unmatched `convert_out_leg` or `convert_in_leg` | `warning` — category: `unclassified_event` |

> **Note:** `sif_received` events are classified as `no_tax_effect` (Decision Log §22.1) and do not generate review items by default. `card_purchase` events are classified as `personal_non_deductible_outflow` and do not generate review items.

### 13.3 Review item categories

| `category` | UI label | Typical trigger |
|---|---|---|
| `uncertain_send` | Uncertain Send | `btc_send`, `eth_send`, `usdc_send_review` |
| `uncertain_receive` | Uncertain Receive | `btc_receive`, `eth_receive`, `usdc_receive_review` |
| `usd_event` | USD Event | USD review codes (`usd_receive_review`, `usd_send_review`, `usd_sell_review`, `usd_buy_review`) |
| `unlinked_cash_buy` | Unlinked Cash Buy | `cash_inflow_review` |
| `unknown_reward` | Unknown Reward | `unknown_reward_review` |
| `unclassified_event` | Unclassified Transaction | `unknown_review`, unmatched convert legs, or `No classification rule matched` |
| `broken_basis` | Broken Basis | Negative pool or incomplete prior-year basis |

> **`usd_event` `suggestedActions`.** The suggested actions in the review item must differ by direction: `usd_buy_review` items should prompt the user to confirm a CAD-to-USD purchase; `usd_sell_review` items should prompt the user to confirm a USD disposal and its FX gain/loss treatment. Do not use generic USD-event copy for both directions.

> **`unclassified_event` vs `unknown_reward`:** `unknown_reward_review` is the fallback for reward-type rows with unrecognized descriptions. `unknown_review` (category `unclassified_event`) is the fallback for any row that exits the full classification hierarchy without a match, including unmatched convert legs.

### 13.4 `ReviewItem` schema

```typescript
interface ReviewItem {
  id:               string;
  severity:         "info" | "warning" | "blocking";
  category:         ReviewItemCategory;
  reasonCode:       string;
  message:          string;
  relatedEventIds:  string[];
  suggestedActions: string[];
}

type ReviewItemCategory =
  | "uncertain_send"
  | "uncertain_receive"
  | "usd_event"
  | "unlinked_cash_buy"
  | "unknown_reward"
  | "unclassified_event"
  | "broken_basis";
```

### 13.5 Blocking vs non-blocking

**Normative rule — negative pool quantity:**

When a disposition would cause a pool's quantity to go negative, the engine MUST:
1. Halt the ACB computation for that asset at that point.
2. Emit a `NegativePoolError` — surfaced as a `ReviewItem` with `severity: "blocking"` and `category: "broken_basis"`.
3. Prevent the session from advancing to `CALCULATED` until the blocking item is resolved.

A `NegativePoolError` is therefore always manifested as a blocking `ReviewItem`. These are not two separate outcomes — the error produces the review item.

**Other blocking conditions** — prevent `CALCULATED` status until resolved:
- Impossible numeric parse
- Contradictory duplicate rows

**Non-blocking** — allow calculation to proceed with warnings:
- Unmatched internal transfer (BTC, ETH, or USDC)
- Unknown reward description
- Unclassified events (including unmatched convert legs)
- USD review items

### 13.6 Safety rule

**No row may ever be silently dropped.** Any row that cannot be confidently classified or reconciled must produce a `ReviewItem` with a reason code and related row IDs.

### 13.7 Override logging

All user decisions on review items must be audit-logged:

```typescript
interface OverrideLog {
  id:          string;
  importId:    string;
  targetType:  "classified_row" | "economic_event";
  targetId:    string;
  oldValue:    Record<string, unknown>;
  newValue:    Record<string, unknown>;
  reason:      string;
  createdAt:   string;
}
```

### 13.8 Typical high-volume review categories

**Uncertain send / receive:** Every outbound or inbound on-chain transfer requires user confirmation of wallet ownership. Applies to BTC, ETH, and on-chain USDC rows. USDC conversion legs (`converted @` description) and `@username` send/receive rows (SIF) are auto-resolved with no review item.

**USD events:** Preserved in the review ledger. USD/USDC legs sharing timing and amounts may be grouped into a `usd_flow_cluster`. Heavy USD volume can fill the review queue — see §29 for full discussion.

**Unlinked cash buy:** Cash file shows a Buy but the linker cannot pair it to a crypto file Buy within tolerance.

**Unknown rewards:** Reward lines that match no known description pattern route to `unknown_reward_review`.

### 13.9 Suggested mitigations (product & engineering)

**Temporal pairing heuristics:** A Buy → Send within a short window with matching quantity on the same asset often indicates buy-and-withdraw-to-own-wallet. Such rules reduce review volume for strong matches and must remain overrideable.

**Wallet frequency analysis:** If the same counterparty address appears across many sends or receives, it is more likely to be the user's own address.

**USD conversion patterns:** Rows whose descriptions indicate conversion at a fixed peg are often plumbing legs rather than standalone economic events.

**Review UX:** Per-category explanations stating *why* review exists; category filters; bulk actions (`internal_transfer`, `taxable_disposition`, `taxable_income`, `skip`) submitted via the bulk review endpoint.

---

## 14. Output Contract

### 14.1 Completeness scoring

| Score | Conditions |
|---|---|
| `complete` | Crypto + cash files present; full-year coverage; prior-year basis available; no unresolved reviews |
| `likely_complete` | Crypto + cash present; minor gaps or ≤5 review items |
| `incomplete` | Crypto present but cash missing; significant coverage gap; >5 unresolved reviews |
| `materially_incomplete` | Crypto file missing; no files for a year with known activity; basis chain broken. **A session without a crypto file that contains crypto-indicating signals (e.g. cash Buy rows) MUST be scored `materially_incomplete` and MUST NOT advance to `CALCULATED`.** |

> **Current limitation:** "Known activity" detection (e.g., cash buys exist but no crypto file) is partially implemented through the `MissingRequiredFileError` gate. Broader completeness scoring through known-activity inference remains an active improvement obligation — see §25.

### 14.2 Annual report structure

Each annual report must contain:

**Summary** — total taxable reward income, interest income, capital gains/losses, net capital gain/loss, taxable capital gain, estimated federal tax, estimated provincial tax, combined estimated tax.

**Breakdown by category** — ShakingSats/SecretSats, referral bonuses, ShakeSquad rewards, CAD interest, card cashback (excluded from income), card purchases (non-deductible outflows, informational), SIF received (excluded), SIF sent (excluded), acquisitions, dispositions.

**Audit trail** — for every event:

| Field | Description |
|---|---|
| `sourceFile` | File family and filename |
| `rawRowIndex` | Row index in source file |
| `timestamp` | Transaction timestamp (UTC) |
| `typeRaw` | Raw type value |
| `descriptionRaw` | Raw description |
| `normalizedCode` | Assigned code |
| `taxTreatment` | Applied treatment |
| `marketValueCad` | CAD value used |
| `basisEffectCad` | Change to ACB pool |
| `gainLossCad` | Realized gain or loss |
| `reviewFlag` | Whether flagged |
| `linkGroupId` | Economic event group |

**Caveats section** — always include:
- Calculations cover Shakepay records only
- Other exchanges and wallets may change basis and gains
- Cashback treatment is a prudent default, not a confirmed CRA/Revenu Québec position
- Review flags may affect final accuracy
- Tax estimates depend on full annual income
- Policy version and classification version used
- Superficial loss rule not yet implemented
- Book Cost used over Market Value for acquisitions and disposition proceeds where present
- Network fees expressed in native crypto asset and converted to CAD via spot rate; fees cannot be applied where spot rate is null

**File coverage summary** — crypto file: yes/no; cash file: yes/no; USD file: yes/no; year coverage percentage; languages uploaded; review item count; completeness status.

---

## 15. API Reference

All session-scoped routes live under `/api/v1/sessions`. All endpoints require authentication (see §27). The normative contract is maintained in `openapi.yaml`.

### 15.1 Sessions

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/sessions` | Create a new session → `{ sessionId, status, createdAt }` |
| `GET` | `/api/v1/sessions/:id` | Get session status and coverage |

### 15.2 File upload

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/sessions/:id/files` | Upload files (`multipart/form-data`, field `files`) → file metadata |
| `GET` | `/api/v1/sessions/:id/files` | List uploaded files with coverage metadata |
| `DELETE` | `/api/v1/sessions/:id/files` | Delete all files and derived data for the session |
| `DELETE` | `/api/v1/sessions/:id/files/:fileId` | Delete a single file |

### 15.3 Processing

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/sessions/:id/process` | Run the full parse → classify → link → basis → summary pipeline |

The process endpoint runs all pipeline stages in sequence and returns processing statistics. It replaces the earlier separate `parse`, `classify`, `reconcile`, and `calculate` endpoints.

### 15.4 Coverage

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/sessions/:id/coverage` | Per-year completeness scores |

### 15.5 Review queue

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/sessions/:id/review` | Review queue; supports `?grouped=true` and `?year=2025` filters |
| `PATCH` | `/api/v1/sessions/:id/review/:itemId` | Submit a single user decision → `{ decision, decisionNote?, excludedFromGroup? }` |
| `POST` | `/api/v1/sessions/:id/review/bulk` | Submit multiple decisions in one request (see below) |

#### Bulk review endpoint (`POST /api/v1/sessions/:id/review/bulk`)

> **Operational importance:** The bulk review endpoint is more important than it first appears. For sessions with heavy USD/USDC or on-chain transfer volume, the review queue can contain dozens or hundreds of items. Without bulk resolution, each user action requires a separate request and a separate query invalidation. This is a key usability safeguard for review-heavy sessions, not merely a convenience feature.

**Request body:**
```json
{
  "items": [
    {
      "kind": "review_item",
      "targetId": "<reviewItemId>",
      "decision": "internal_transfer",
      "decisionNote": "Confirmed own hardware wallet",
      "year": 2024
    },
    {
      "kind": "group",
      "targetId": "<linkGroupId>",
      "decision": "taxable_disposition"
    }
  ]
}
```

| Field | Type | Required | Description |
|---|---|---|---|
| `kind` | `"review_item" \| "group"` | Yes | Whether `targetId` is a single review item or an event group |
| `targetId` | `string` | Yes | ID of the review item or link group |
| `decision` | `string` | Yes | One of `internal_transfer`, `taxable_disposition`, `taxable_income`, `skip` |
| `decisionNote` | `string` | No | Free-text reason stored in the override log |
| `year` | `number` | No | Tax year — used to scope query invalidation on the client |

**Response:**
```json
{
  "updatedCount": 50,
  "results": [
    { "targetId": "<id>", "status": "resolved" }
  ]
}
```

**Performance:** Selecting 50 review items and clicking "Mark as Internal Transfer" sends **1 HTTP request + 1 refetch = 2 requests** instead of 100 individual requests.

#### Rate limiting and retry

The global rate limiter allows **200 requests per minute**. The client retries on `429` up to **3 times** with exponential backoff (1 s → 2 s → 4 s), or uses the `Retry-After` header if present.

### 15.6 Tax inputs and reports

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/v1/sessions/:id/tax-inputs` | Save province and other income for a tax year → `{ taxYear, provinceCode, otherIncomeCad }` |
| `GET` | `/api/v1/sessions/:id/annual-summary/:year` | Annual tax summary JSON |
| `GET` | `/api/v1/sessions/:id/annual-summary/:year/audit` | Normalized events for year (full audit trail) |
| `GET` | `/api/v1/sessions/:id/annual-summary/:year/quebec-disclosure` | TP-21.4.39 companion JSON (QC only) |

### 15.7 Export

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/v1/sessions/:id/export/:year?format=json\|csv` | Export annual data as JSON or CSV |

### 15.8 Health

| Method | Path | Description |
|---|---|---|
| `GET` | `/health` | API liveness check (no auth required) |

### 15.9 Error response format

```json
{
  "error": {
    "code": "SCHEMA_DETECTION_ERROR",
    "message": "Uploaded file does not match any known Shakepay export schema.",
    "details": { "headersFound": ["Col1", "Col2"] }
  }
}
```

| Error type | HTTP status |
|---|---|
| `SchemaDetectionError` | 422 |
| `ValidationError` | 400 |
| `NegativePoolError` | 409 |
| `MissingRequiredFileError` | 422 |
| `DuplicateRowError` | 409 |
| `TaxLawNotFoundError` | 500 |
| Unexpected server error | 500 |

---

## 16. Database Schema

### 16.1 Source of truth

The **canonical** database schema is defined in code:

```
packages/db/src/schema/index.ts   (Drizzle ORM table definitions)
```

Migrations and generated SQL must match that file. The DDL narrative below (§16.2) is **descriptive** documentation. If the markdown and the Drizzle schema diverge, **trust Drizzle + migrations** and update this document.

### 16.2 Schema (descriptive)

```sql
-- Sessions
CREATE TABLE sessions (
  id                    TEXT PRIMARY KEY,
  user_id               TEXT NOT NULL,
  status                TEXT NOT NULL,
  completeness_status   TEXT,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  policy_version        TEXT NOT NULL,
  classifier_version    TEXT NOT NULL
);

-- Uploaded files
CREATE TABLE session_files (
  id              TEXT PRIMARY KEY,
  session_id      TEXT NOT NULL REFERENCES sessions(id),
  file_name       TEXT NOT NULL,
  sha256          TEXT NOT NULL,
  family          TEXT NOT NULL CHECK (family IN ('crypto_summary','cash_summary','usd_summary')),
  language        TEXT NOT NULL CHECK (language IN ('en','fr')),
  row_count       INTEGER NOT NULL,
  coverage_start  TIMESTAMPTZ,
  coverage_end    TIMESTAMPTZ,
  uploaded_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Raw rows — immutable after upload
CREATE TABLE raw_rows (
  id          TEXT PRIMARY KEY,
  file_id     TEXT NOT NULL REFERENCES session_files(id),
  row_index   INTEGER NOT NULL,
  raw_json    JSONB NOT NULL
);

-- Canonical rows
CREATE TABLE canonical_rows (
  id             TEXT PRIMARY KEY,
  raw_row_id     TEXT NOT NULL REFERENCES raw_rows(id),
  canonical_json JSONB NOT NULL
);

-- Classified rows
CREATE TABLE classified_rows (
  id                TEXT PRIMARY KEY,
  canonical_row_id  TEXT NOT NULL REFERENCES canonical_rows(id),
  normalized_code   TEXT NOT NULL,
  tax_treatment     TEXT NOT NULL,
  confidence        NUMERIC NOT NULL,
  review_flag       BOOLEAN NOT NULL DEFAULT FALSE,
  review_reason     TEXT
);

-- Economic events (reconciled)
CREATE TABLE economic_events (
  id               TEXT PRIMARY KEY,
  session_id       TEXT NOT NULL REFERENCES sessions(id),
  event_kind       TEXT NOT NULL,
  normalized_code  TEXT NOT NULL,
  tax_treatment    TEXT NOT NULL,
  timestamp        TIMESTAMPTZ NOT NULL,
  asset_in         TEXT,
  amount_in        NUMERIC(20,8),
  asset_out        TEXT,
  amount_out       NUMERIC(20,8),
  market_value_cad NUMERIC(20,2),
  book_cost_cad    NUMERIC(20,2),
  review_flag      BOOLEAN NOT NULL DEFAULT FALSE
);

-- Economic event membership (many rows → one event)
CREATE TABLE economic_event_members (
  economic_event_id  TEXT NOT NULL REFERENCES economic_events(id),
  classified_row_id  TEXT NOT NULL REFERENCES classified_rows(id),
  PRIMARY KEY (economic_event_id, classified_row_id)
);

-- Asset ledger entries
CREATE TABLE asset_ledger_entries (
  id                TEXT PRIMARY KEY,
  session_id        TEXT NOT NULL REFERENCES sessions(id),
  tax_year          INTEGER NOT NULL,
  event_id          TEXT NOT NULL REFERENCES economic_events(id),
  asset             TEXT NOT NULL,
  quantity_delta    NUMERIC(20,8) NOT NULL,
  acb_delta_cad     NUMERIC(20,2) NOT NULL,
  proceeds_cad      NUMERIC(20,2) NOT NULL DEFAULT 0,
  acb_allocated_cad NUMERIC(20,2) NOT NULL DEFAULT 0,
  gain_loss_cad     NUMERIC(20,2) NOT NULL DEFAULT 0
);

CREATE INDEX idx_ale_asset_year ON asset_ledger_entries(asset, tax_year);
CREATE INDEX idx_ale_event      ON asset_ledger_entries(event_id);

-- Annual summaries
CREATE TABLE annual_summaries (
  session_id                    TEXT NOT NULL REFERENCES sessions(id),
  tax_year                      INTEGER NOT NULL,
  province                      TEXT NOT NULL,
  reward_income_cad             NUMERIC(20,2) NOT NULL DEFAULT 0,
  interest_income_cad           NUMERIC(20,2) NOT NULL DEFAULT 0,
  capital_gains_cad             NUMERIC(20,2) NOT NULL DEFAULT 0,
  capital_losses_cad            NUMERIC(20,2) NOT NULL DEFAULT 0,
  net_capital_gain_cad          NUMERIC(20,2) NOT NULL DEFAULT 0,
  taxable_capital_gain_cad      NUMERIC(20,2) NOT NULL DEFAULT 0,
  purchase_rebates_cad          NUMERIC(20,2) NOT NULL DEFAULT 0,
  non_taxable_review_cad        NUMERIC(20,2) NOT NULL DEFAULT 0,
  usd_ledger_cad_equiv          NUMERIC(20,2) NOT NULL DEFAULT 0,
  federal_tax_estimate_cad      NUMERIC(20,2),
  provincial_tax_estimate_cad   NUMERIC(20,2),
  total_tax_estimate_cad        NUMERIC(20,2),
  completeness_status           TEXT NOT NULL,
  review_item_count             INTEGER NOT NULL DEFAULT 0,
  federal_tax_law_version_id    TEXT REFERENCES tax_law_versions(id),
  provincial_tax_law_version_id TEXT REFERENCES tax_law_versions(id),
  policy_version                TEXT NOT NULL,
  classifier_version            TEXT NOT NULL,
  assumptions_json              JSONB NOT NULL,
  computed_at                   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (session_id, tax_year)
);

-- Review queue
CREATE TABLE review_items (
  id            TEXT PRIMARY KEY,
  session_id    TEXT NOT NULL REFERENCES sessions(id),
  tax_year      INTEGER,
  event_id      TEXT REFERENCES economic_events(id),
  severity      TEXT NOT NULL CHECK (severity IN ('info','warning','blocking')),
  category      TEXT NOT NULL,
  reason_code   TEXT NOT NULL,
  message       TEXT NOT NULL,
  status        TEXT NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending','resolved','skipped')),
  user_decision TEXT,
  resolved_at   TIMESTAMPTZ,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ri_session_status ON review_items(session_id, status);

-- Override audit log
CREATE TABLE overrides (
  id           TEXT PRIMARY KEY,
  session_id   TEXT NOT NULL REFERENCES sessions(id),
  target_type  TEXT NOT NULL,
  target_id    TEXT NOT NULL,
  old_value    JSONB NOT NULL,
  new_value    JSONB NOT NULL,
  reason       TEXT NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Tax law versions
CREATE TABLE tax_law_versions (
  id             TEXT PRIMARY KEY,
  tax_year       INTEGER NOT NULL,
  jurisdiction   TEXT NOT NULL,
  source_url     TEXT NOT NULL,
  effective_from DATE NOT NULL,
  effective_to   DATE,
  inclusion_rate NUMERIC(6,4) NOT NULL,
  metadata       JSONB NOT NULL DEFAULT '{}',
  UNIQUE (tax_year, jurisdiction)
);

-- Tax brackets
CREATE TABLE tax_brackets (
  id                 TEXT PRIMARY KEY,
  tax_law_version_id TEXT NOT NULL REFERENCES tax_law_versions(id),
  lower_bound        NUMERIC(20,2) NOT NULL,
  upper_bound        NUMERIC(20,2),
  rate               NUMERIC(6,4) NOT NULL
);

CREATE INDEX idx_tb_version ON tax_brackets(tax_law_version_id);

-- Tax-law override profiles
CREATE TABLE tax_override_profiles (
  id             TEXT PRIMARY KEY,
  name           TEXT NOT NULL,
  description    TEXT,
  overrides_json JSONB NOT NULL,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

### 16.3 Required schema-level enforcement for event linking

The application-layer linking logic described in §8 must be backed by schema-level constraints.

**Explicit link group membership table:**

```sql
CREATE TABLE link_groups (
  id           TEXT PRIMARY KEY,
  session_id   TEXT NOT NULL REFERENCES sessions(id),
  group_type   TEXT NOT NULL CHECK (group_type IN (
                 'cad_crypto_buy', 'cad_crypto_sell', 'crypto_swap',
                 'reward_only', 'usd_flow_cluster', 'review')),
  created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE link_group_members (
  link_group_id      TEXT NOT NULL REFERENCES link_groups(id),
  classified_row_id  TEXT NOT NULL REFERENCES classified_rows(id),
  PRIMARY KEY (link_group_id, classified_row_id)
);

-- A single classified row may belong to at most one link group.
CREATE UNIQUE INDEX idx_lgm_row_unique ON link_group_members(classified_row_id);
```

**Enforcement intent:**
- A classified row cannot be a member of two different link groups.
- Link groups belong to a session and carry a declared type.
- Orphaned link groups (no members) must be prevented at the service layer before commit.
- The `economic_event_members` join table enforces N:1 membership of classified rows to economic events; `link_group_members` enforces the earlier linker-stage grouping.

---

## 17. Tax Law Data Layer

### 17.1 Seed file format

Each seed file covers one jurisdiction and one tax year. Example shape:

```json
{
  "taxYear": 2025,
  "jurisdiction": "FED",
  "sourceUrl": "<official source URL>",
  "effectiveFrom": "2025-01-01",
  "effectiveTo": null,
  "inclusionRate": "<decimal>",
  "brackets": [
    { "lowerBound": "<decimal>", "upperBound": "<decimal>", "rate": "<decimal>" }
  ],
  "surtaxes": [],
  "reductions": [],
  "credits": [],
  "metadata": {
    "publishedAt": "<date>",
    "notes": "<free text>"
  }
}
```

> Use placeholders in documentation examples. Active numeric values must come from maintained official seeds, not from prose in this spec.

### 17.2 `taxLawRegistry` behavior

- Loads seed files at startup or on admin refresh.
- Caches by `jurisdiction-year` key: `"FED-2025"`, `"QC-2025"`, etc.
- Throws `TaxLawNotFoundError` if no active table exists for the combination.
- Supports supersession, activation, and rollback of law versions.
- Applies override profiles after base-law load and before estimator execution.

---

## 18. Error Handling

### 18.1 Hard errors — fail and surface

| Error | Condition |
|---|---|
| `SchemaDetectionError` | File headers match no known schema |
| `ValidationError` | Required columns missing; invalid date; invalid numeric |
| `NegativePoolError` | Asset pool quantity becomes negative — always surfaces as a blocking `ReviewItem` (see §13.5) |
| `DuplicateRowError` | Same row ID with conflicting payload |
| `TaxLawNotFoundError` | No bracket table for jurisdiction + year |
| `MissingRequiredFileError` | No crypto summary file present and session contains crypto-indicating signals |

### 18.2 Soft warnings — proceed with review flag

| Warning | Condition |
|---|---|
| Partial-year coverage | Selected year not fully covered |
| Unmatched economic event | Buy legs that cannot be reconciled |
| Missing prior-year basis | Historical data unavailable |
| Unresolved send/receive | Transfer ownership unclear (BTC, ETH, or USDC) |
| USD rows present | Policy treatment pending |
| Unknown reward description | Cannot map to known normalized code |
| Unclassified event | Row exited full classification hierarchy without a match |
| Unmatched convert leg | `convert_out_leg` or `convert_in_leg` without a paired companion within the time window |
| Missing fee valuation | Fee quantity and asset parsed from description, but `Spot Rate` is null — fee cannot be converted to CAD |

### 18.3 Safety invariant

No row is ever silently dropped. If a row cannot be processed at any stage, it must produce a `ReviewItem` and remain in the database in its last-valid stage.

---

## 19. Testing Strategy

### 19.1 Unit tests

| Module | Tests required |
|---|---|
| `detector.ts` | All 6 file families; French/English; unknown schema rejects; two-stage crypto/USD resolution |
| `helpers.ts` (`normalizeAsset`) | `USDC` / `usdc` / trimmed casing → `'USDC'`; BTC/ETH/USD round-trip; unknown tickers pass through unchanged |
| `canonicalizer.ts` | All bilingual field mappings; null for empty cells; decimal-safe parse; French locale numbers |
| `classifyReward.ts` | Each reward code in EN and FR; priority order; SecretSats matches `shakingsats`; unknown → `unknown_reward_review` |
| `classifyTransaction.ts` | All standard codes; buy/sell/send/receive/convert; **`card_purchase` type → `card_purchase` / `personal_non_deductible_outflow`**; Interac with credit → `cad_deposit`; Interac with debit → `cad_withdrawal`; Interac with both null → `unknown_review`; cash `receive` `@username` → `sif_received`; cash `receive` email → `cad_receive`; cash `send` `@username` → `sif_sent`; crypto `send`/`receive` `@username` → `sif_sent`/`sif_received` before on-chain transfer rules |
| `crypto.rules.ts` (USDC) | `usdc_cad_conversion` for Send/Receive + `convertedAt` desc; `usdc_send_review` / `usdc_receive_review` for on-chain rows |
| `crypto.rules.ts` (convert) | `convert_out_leg` for debit-only row; `convert_in_leg` for credit-only row; both-populated → `unknown_review` |
| `review-queue.service.ts` | `unclassified_event` category for `unknown_review`, unmatched convert legs, and `No classification rule matched`; `card_purchase` does not generate review item |
| `assetPool.ts` | Acquisition adds qty + ACB; disposition reduces qty + ACB; negative qty blocks; USDC excluded from pools |
| `dispositionEngine.ts` | Gain/loss math; **native-asset fee parsed and converted via spot rate** (`feeCad = qty × spotRate`); `spotRate = null` → `missing_fee_valuation` warning; no fee → `feeCad = 0`; ACB allocation; prior-year basis; book cost priority over market value; **USD-priced sell uses `bookCostCad` not USD description amount** |
| `eventLinker.ts` | Exact match; near-match within tolerance; ambiguous → review; unmatched → flag; **two `convert` rows same timestamp → `crypto_swap` link group**; single `convert` row no companion → `unclassified_event` ReviewItem |
| `inclusionRate.ts` | 50% for 2021–2026; no hardcoded year assumptions; table-driven |
| `completenessScorer.ts` | All four score levels; each contributing factor; missing crypto file with cash buy signals → `materially_incomplete` |
| `process-session.service.ts` | Orchestration order; pipeline runs in correct sequence; `MissingRequiredFileError` thrown when no crypto file and crypto signals present |

### 19.2 Integration tests

| Fixture | What it validates |
|---|---|
| English-only crypto + cash | Full pipeline; reconciliation; BTC ACB; book cost used for acquisition |
| French-only crypto + cash | Bilingual parsing produces identical output to English fixture |
| Mixed EN/FR session | Bilingual parsing coexistence |
| Multi-year 2022–2025 | Prior-year basis carries forward correctly |
| Crypto-only (no cash) | Completeness warning; cash rewards not counted |
| All 3 families | USD preserved in review ledger; not in crypto ACB; `usd_flow_cluster` grouping |
| Partial year | Coverage warning; `incomplete` completeness status |
| QC resident 2024+ | `quebecDisclosureRequired = true`; TP-21.4.39 report with line 49 and line 135 populated |
| USDC conversion rows | Auto-resolved as `no_tax_effect`; no review items generated |
| USDC on-chain rows | `usdc_send_review` / `usdc_receive_review` in review queue |
| USDC end-to-end pipeline | Conversion leg → `usdc_cad_conversion`; on-chain send → `usdc_send_review`; receive → `usdc_receive_review` |
| Bulk review — 50-item batch | `POST /review/bulk` resolves all 50 items; query invalidation fires once; override log has 50 entries |
| Bulk review — mixed `kind` | Items and groups in the same batch are all resolved; `updatedCount` correct |
| SIF received | `sif_received` → `no_tax_effect`; no review item generated |
| Network fee disposition | Fee parsed from description in native asset; converted via spot rate; net proceeds reduced correctly |
| Network fee — null spot rate | `missing_fee_valuation` warning; fee not applied; audit trail flagged |
| Two `Convert` rows sharing the same timestamp (ETH out, BTC in) | Linked as `crypto_swap`; ETH disposition gain/loss computed; BTC basis established |
| Card purchase rows | `card_purchase` → `personal_non_deductible_outflow`; no review item |
| Interac debit row | `cad_withdrawal` → `no_tax_effect` |
| Cash Receive `@username` | `sif_received` → `no_tax_effect`; no review item |
| Missing crypto file with cash buy signals | `MissingRequiredFileError`; `materially_incomplete` |

### 19.3 Regression fixtures

| Fixture | Expected outcome |
|---|---|
| ShakingSats (crypto EN) | `shakingsats` → income + BTC basis |
| SecretSats | `shakingsats` (same code) → income + BTC basis |
| Referral reward (cash EN) | `referral_bonus` → income |
| Referral reward (cash FR) | Same as above — bilingual parity |
| BTC cashback (crypto EN) | `card_cashback_btc` → basis only, no income |
| BTC sale | `btc_to_cad` → `taxable_disposition` + gain/loss; book cost as proceeds |
| BTC↔ETH swap (two rows) | `convert_out_leg` (BTC) → `taxable_disposition`; `convert_in_leg` (ETH) → `crypto_acquisition`; linked as `crypto_swap` |
| BTC send confirmed internal | No tax event; basis preserved |
| BTC send unconfirmed | `transfer_or_disposition_review` |
| SIF received | `sif_received` → `no_tax_effect`; no review item |
| Cash buy + crypto buy | Reconciled to one `cad_to_crypto_buy` event — no double count |
| USDC Send with `converted @` desc | `usdc_cad_conversion` → `no_tax_effect`, `reviewFlag: false` |
| USDC Receive with `converted @` desc | `usdc_cad_conversion` → `no_tax_effect`, `reviewFlag: false` |
| USDC Send to on-chain address | `usdc_send_review` → `transfer_or_disposition_review`, category `uncertain_send` |
| USDC Receive from on-chain address | `usdc_receive_review` → `transfer_or_income_review`, category `uncertain_receive` |
| Row with no matching classification rule | `unknown_review` → category `unclassified_event` |
| **Cash `Card purchase` row (any merchant)** | `card_purchase` → `personal_non_deductible_outflow`; no review item |
| **Cash `Interac e-Transfer` with Credit populated** | `cad_deposit` → `no_tax_effect` |
| **Cash `Interac e-Transfer` with Debit populated** | `cad_withdrawal` → `no_tax_effect` |
| **Cash `Receive` with description `@moha20`** | `sif_received` → `no_tax_effect`; no review item |
| **Cash `Receive` with email description** | `cad_receive` → `no_tax_effect` |
| **Cash `Send` with description `@johsol`** | `sif_sent` → `personal_non_deductible_outflow` |
| **Crypto `Send` with description `@johsol`** | `sif_sent` → `personal_non_deductible_outflow`; no review item |
| **Crypto `Receive` with description `@noregretsjohn`** | `sif_received` → `no_tax_effect`; no review item |
| **Two `Convert` rows, same timestamp, ETH debited / BTC credited** | Linked as `crypto_swap`; `convert_out_leg` (ETH) → `taxable_disposition`; `convert_in_leg` (BTC) → `crypto_acquisition` |
| **Single `Convert` row with no matching companion within time window** | `unclassified_event` ReviewItem, severity `warning` |
| **USD `Sell` row with USD credited and null description** | `usd_buy_review` → `usd_review` |
| **USD `Sell` row with USD debited and `"Sold X USD @ CA$Y"` description** | `usd_sell_review` → `usd_review` |
| **BTC `Send` with description `"Bitcoin address bc1q.... Fees: 0.000007 BTC."`, `spotRate = 120000`** | `feeCad = 0.84`; net proceeds reduced accordingly |
| **BTC `Send` with fee in description, `spotRate = null`** | `missing_fee_valuation` warning; `feeCad = null`; audit trail flagged |
| **BTC `Sell` with description `"Sold @ US$106,145.25"` and `bookCostCad` populated** | `btc_to_cad` → `taxable_disposition`; proceeds = `bookCostCad`, not USD description amount |

### 19.4 QA matrix summary

| Layer | Test type | Minimum coverage |
|---|---|---|
| File detection | Unit | All 6 known schemas + unknown; two-stage crypto/USD resolution |
| Header normalization | Unit | All bilingual field pairs |
| Asset normalization | Unit | BTC/ETH/USD/USDC; casing; trim; unknown passthrough |
| Classification | Unit | All normalized codes EN + FR, including USDC codes, SecretSats, `card_purchase`, Interac directionality, cash send/receive, `@username` SIF, two-leg convert, USD sub-types |
| Reconciliation | Unit + Integration | Exact, near, ambiguous, unmatched; `usd_flow_cluster` grouping; two-leg convert pairing |
| ACB engine | Unit | BTC + ETH, acquisition + disposition (with native-asset fee + spot-rate conversion) + swap + multi-year; book cost priority; USDC excluded; USD-priced sell proceeds from `bookCostCad` |
| Tax engine | Unit | All federal bracket boundaries; representative province coverage; 50% inclusion rate for all years |
| Review queue | Unit | All trigger conditions; blocking vs non-blocking; `unclassified_event` category; SIF and card purchases do not trigger review; unmatched convert legs trigger `unclassified_event` |
| Bulk review API | Integration | 50-item batch resolves correctly; query invalidation once; retry on 429 |
| API | Contract | All endpoints against openapi.yaml; error codes including `MissingRequiredFileError` |
| E2E | Integration | ≥1 mixed-language multi-year scenario |

---

## 20. Policy Positions & Caveats

### 20.1 High-confidence positions

| Position | Implementation stance |
|---|---|
| Product is a Shakepay-only computation layer | Always disclose limited source scope |
| Crypto dispositions can create capital gains or losses | Treat sales, swaps, and non-internal sends as disposition-capable events |
| Prior-year basis affects current-year gains | Build opening ACB from all available prior history |
| ShakingSats and SecretSats are taxable income at market value | Both map to `shakingsats` normalized code |
| Bitcoin cashback is a purchase rebate (cost basis adjustment, not income) | `card_cashback_btc` → `purchase_rebate` |
| Card purchases are personal non-deductible outflows | `card_purchase` → `personal_non_deductible_outflow` |
| Québec crypto activity can trigger TP-21.4.39 requirements | Show reminder and produce support summary |
| Federal and provincial tax depend on full annual income | Require province per year; support marginal mode |
| ACB method is pooled average cost per asset (BTC, ETH, USD pools separate) | Chronological processing mandatory across all years |
| Superficial loss rule | Not yet implemented — near-term release risk (see §21.7) |
| Network fees are denominated in native crypto asset, converted to CAD via spot rate | Parse `Fees: <qty> <ASSET>` from description; `feeCad = qty × spotRate` |

### 20.2 Prudent default positions

These are implementation defaults, **not** Shakepay-specific published tax rulings. They must be disclosed in every report.

| Position | Default | Trigger for change |
|---|---|---|
| Card cashback in BTC = purchase rebate, not income | `purchase_rebate` | Clear CRA or Revenu Québec guidance specific to this fact pattern |
| 3% card intro bonus = purchase rebate | `purchase_rebate` | Same as above |
| USD events are preserved but excluded from crypto ACB and gain logic | `usd_review` | Formal USD treatment module and policy approval — see §29 |
| USDC `converted @` legs = wallet plumbing, no tax effect | `no_tax_effect` | Formal USDC treatment policy — see §29 |
| USDC on-chain sends/receives = ownership ambiguous | `transfer_or_disposition_review` / `transfer_or_income_review` | User confirmation per review queue |
| Capital gains inclusion rate = 50% flat for 2021–2026 | 50% | When a new rate becomes enacted law; update `inclusion-rates.ts` and activate the new law version |

### 20.3 Resolved positions (see Decision Log §22)

| Position | Decision | Rationale |
|---|---|---|
| SIF received | `no_tax_effect` (implemented) | SIF transfers between Shakepay wallets do not change beneficial ownership |
| Review decision → basis recomputation | Full recompute from earliest impacted event | Ensures ACB integrity after user overrides |
| Materially incomplete detection | File presence + crypto-indicating signal detection | `MissingRequiredFileError` enforced at pipeline entry |
| Convert events | Two-row model; `convert_out_leg` + `convert_in_leg` linked by Event Linker | Matches production export format |
| Interac direction | Determined by populated `credit` or `debit` field | Matches production export format |
| SIF detection | `@username` pattern (`^@\w+$`) is primary; `shake it forward` text unconfirmed in production | Matches production export format |
| Network fees | Native asset denomination; converted via `spotRate`; null spot rate → `missing_fee_valuation` warning | Matches production export format |
| USD `Sell` sub-types | `usd_buy_review` (CAD→USD) vs `usd_sell_review` (USD disposal) distinguished by asset direction | Matches production export format |

### 20.4 Book cost vs. market value disclosure

Reports must disclose that acquisition ACB uses Book Cost from the Shakepay CSV where present, and that disposition proceeds default to Book Cost before Market Value.

### 20.5 Required UI disclosures

The following notices must appear in the UI and in every generated report:

1. "This report covers Shakepay activity only. Other exchanges, wallets, or crypto income sources are not included."
2. "Cashback in BTC is treated by default as a purchase rebate, not ordinary income. This is a prudent default, not a confirmed CRA or Revenu Québec position specific to Shakepay cashback."
3. "Prior-year transactions may affect current-year capital gains through adjusted cost base."
4. "Québec residents may need to file form TP-21.4.39 — Déclaration relative aux cryptoactifs."
5. "Tax estimates are based on the tax-law versions selected for the relevant year and jurisdiction. Total tax payable still depends on the taxpayer's full annual income."
6. "USD activity is preserved but not included in crypto capital-gain calculations pending a formal policy treatment."
7. "USDC conversion legs ('Converted @') are treated as wallet-plumbing events with no tax effect. On-chain USDC transfers require manual review to determine ownership."
8. "The superficial loss rule is not yet implemented. Capital losses may be overstated for repurchases within 30 days before or after a disposition. Do not rely on reported capital losses without professional review if you reacquired the same asset within that window."
9. "Network fees, where parsed from transaction descriptions, are expressed in native crypto asset (BTC or ETH) and converted to CAD using the spot rate from the same row. Where the spot rate is unavailable, the fee cannot be applied and is flagged for review. Confirm large fees with a tax professional."
10. "Acquisition cost and disposition proceeds use Book Cost from Shakepay exports where present, not Market Value. This reflects the transacted buy/sell rate."

### 20.6 Policy and law version tracking

Every `NormalizedEvent`, `AnnualSummary`, and `QuebecDisclosure` must store:
- `policyVersion` — date string of the governing rules document (e.g., `"2026-03-29"`)
- `classifierVersion` — version of the classification rule registry
- `federalTaxLawVersionId`
- `provincialTaxLawVersionId`

---

## 21. Known Issues & Review Notes

### 21.1 `unknown_review` rows

`unknown_review` rows must never be treated as resolved by virtue of reaching the review queue. Every such row must surface as an `unclassified_event` `ReviewItem` with its reason code, suggested actions, and must appear in the audit trail.

### 21.2 USD activity

USD rows carry `usd_review` treatment. Canadian tax treatment of FX gains and losses on USD↔CAD conversions is not trivially zero. Until a formal USD policy module is built, the engine must preserve all USD rows in the review ledger with full monetary data and must never silently assign `no_tax_effect` to USD rows. See §29.

### 21.3 Inconsistent second-leg treatment on `cad_to_btc`

The CAD side of a `cad_to_btc` buy sometimes receives `no_tax_effect` and sometimes `cash_funding_only` in observed exports. The rule registry must be explicit about which sub-type maps to which treatment, and the reconciler must verify consistency across linked legs.

### 21.4 BTC→ETH swap with unresolved ETH leg

In some exports, a conversion event records `taxable_disposition` on the out-leg while the in-leg classifies as `unknown_review`. This asymmetry means the disposition is taxed but the incoming basis is not properly established. The Event Linker should detect and flag this; the ACB engine must not silently proceed with an incomplete swap pair.

### 21.5 Quantity mismatches from fees or partial fills

Small differences between a buy quantity and a subsequent send quantity are normal with pooled ACB. The reconciler's CAD tolerance of ≤ $0.05 addresses most cases; quantity-based matching should be secondary.

### 21.6 Tax-year filter alignment

The year filter in the UI and API must align precisely with the UTC calendar-year boundary used in the ledger engine (`2025-01-01T00:00:00Z`). Integration tests must use timestamps near December 31 / January 1 boundaries in UTC to verify correct year assignment — see §9.8.

### 21.7 Superficial loss rule not implemented — near-term release risk

The superficial loss rule is **not yet implemented**. Active crypto users frequently dispose of and reacquire the same asset within the relevant window. Reported capital losses may be materially overstated for any user with such activity.

**Required mitigations before launch:**
- Display a conspicuous limitation notice on every screen and report section where capital losses are shown.
- Flag individual disposition events where a reacquisition within 30 days is detectable from the imported data.
- Elevate the priority of implementing this rule — it must not remain indefinitely deferred.

### 21.8 USDC basis exclusion

USDC rows are intentionally excluded from all ACB pool logic. Any future decision to include USDC in gain/loss calculations requires an explicit basis-pool extension, a policy decision, and a migration path for existing sessions.

### 21.9 Materially incomplete — known-activity detection partially implemented

A session is blocked from `CALCULATED` when no crypto file is present and crypto-indicating signals exist (enforced via `MissingRequiredFileError`). Broader completeness scoring through known-activity inference — e.g., detecting that cash buys exist but no crypto file was uploaded and scoring accordingly — remains an active improvement obligation. See §25.

### 21.10 `card_purchase` French type value unconfirmed

The French equivalent of the `Card purchase` type value has not been confirmed from a production French-language export. Until confirmed, French-language cash exports may route card purchase rows to `unknown_review`. See open decision §22.6.

### 21.11 `btc_to_eth` / `eth_to_btc` enum migration pending

The `btc_to_eth` and `eth_to_btc` normalized codes are deprecated in favour of the two-leg `convert_out_leg` / `convert_in_leg` model. Any sessions processed before this change may have persisted these codes. A migration strategy is required before removing the deprecated codes from the enum — see §22.6.

---

## 22. Decision Log & Open Decisions

### 22.1 SIF Received Handling

**Status:** Resolved — treated as `no_tax_effect` (internal transfer)
**Rationale:** SIF transfers between Shakepay wallets do not change beneficial ownership. The primary production signal is a bare `@username` description (e.g. `@johsol`), not the text `shake it forward`.

### 22.2 Review Decision Effect on Basis

**Status:** Resolved — any decision that changes a basis-affecting classification triggers full recomputation from the earliest impacted timestamp.

### 22.3 USD Tax Policy Module

**Status:** Deferred — USD events ingested and preserved in review queue with `usd_review` treatment. Not automatically included in capital gains. See §29.

### 22.4 Capital Gains Inclusion Rate

**Status:** Resolved — 50% for all years 2021–2026 (flat rate, enacted). The June 2024 federal budget proposed raising the inclusion rate; Finance Canada deferred the proposal on January 31, 2025; the CRA administered the enacted one-half rate; the proposed increase was later cancelled.

### 22.5 Materially Incomplete Inference

**Status:** Partially resolved — `MissingRequiredFileError` enforced at pipeline entry when crypto-indicating signals exist and no crypto file is present. Broader known-activity completeness scoring is an active improvement obligation — see §25.

### 22.6 Remaining open decisions

| Decision | Options | Recommendation |
|---|---|---|
| **Tax-law source** | Manual JSON seeds vs. third-party rate API | Manual seeds first; add sync later |
| **Tax-law refresh cadence** | On demand vs. scheduled review | Quarterly review plus ad hoc on published changes |
| **Internal transfer detection** | Auto-detect by matching sends/receives vs. user-confirmed only | User-confirmed for MVP; add heuristics later |
| **Default tax mode** | Standalone or marginal | Marginal is more accurate; standalone as fallback |
| **USDC formal policy** | Define now or defer | Defer; conversion legs auto-resolved; on-chain rows reviewed |
| **Row ID scheme** | Random UUID vs. deterministic SHA-256 | Deterministic: `sha256(session_id + file_id + row_index + normalized_payload)` |
| **Province change across years** | Single province only vs. per-year selection | Require per-year selection |
| **Tax-law overrides** | Direct table edits vs. override profiles | Keep immutable base tables; use explicit override profiles |
| **Superficial loss** | Define now or defer | Implement before launch; disclose limitation in every report until complete |
| **`btc_to_eth` / `eth_to_btc` enum migration** | Retain deprecated codes as guards vs. remove and migrate | Retain with `DEPRECATED` annotation; write migration to reprocess sessions using these codes before the two-leg model was introduced |
| **`card_purchase` French type value** | Wait for confirmed French export vs. infer by exclusion | Wait for confirmed French-language export; treat unmatched French rows as `unknown_review` in the interim |
| **`cad_send` scope for non-`@username`, non-email cash sends** | Separate code vs. route to `unknown_review` | Route to `unknown_review` until population of that pattern is better understood |
| **`@username` regex character set** | `^@\w+$` (alphanumeric + underscore) vs. wider set | Confirm Shakepay username character set from account creation rules before shipping |

---

## 23. Implementation Order

Build in this sequence to avoid blocking dependencies:

1. Schema detection + canonicalization + `normalizeAsset` (two-stage crypto/USD detection)
2. Classification engine (reward patterns + standard transactions + USDC rules + `card_purchase` + Interac directionality + cash send/receive + `@username` SIF + two-leg convert + USD sub-types)
3. Shared description patterns (`DESC_PATTERNS` including `sifUsername`, `feeNative`, `usdSoldDesc`)
4. Event linking / reconciler (with configurable time window; two-leg convert pairing)
5. BTC/ETH/USD basis ledger engine (USDC excluded; native-asset fee parser with spot-rate conversion)
6. Annual aggregation + completeness scorer
7. Review queue + override logging (including `unclassified_event` category; `card_purchase` excluded)
8. Unified `process` endpoint orchestration (with `MissingRequiredFileError` gate)
9. Tax law registry + bracket loader
10. Tax estimation adapter (federal + all provinces)
11. Québec disclosure module
12. API routes + middleware (auth, rate limiting)
13. Bulk review endpoint
14. Report export (JSON, CSV)
15. Schema-level link group enforcement (§16.3)
16. `btc_to_eth` / `eth_to_btc` migration (§22.6)

---

## 24. Definition of Done

The feature is complete only when **all** of the following are true.

**✅ All six file types detected correctly**
- `cash_summary/en`, `cash_summary/fr`, `crypto_summary/en`, `crypto_summary/fr`, `usd_summary/en`, `usd_summary/fr` via two-stage detection
- Unknown schemas produce `SchemaDetectionError`

**✅ Bilingual parsing works end-to-end**
- English and French files produce identical canonical schemas
- All bilingual type values and description patterns recognized
- Decimal-safe numeric parsing; French locale handled; empty cells → `null`

**✅ `normalizeAsset` handles all supported tickers**
- BTC, ETH, USD, USDC recognized case-insensitively with whitespace trimming
- Unknown tickers pass through without error

**✅ Cash/crypto buys reconciled without double counting**
- Matching cash debit + crypto credit rows linked via `linkGroupId` (not merged)
- Linker amount comparison uses `bookCostCad` (fallback `marketValueCad`)
- ACB does not count the same acquisition twice; ambiguous matches routed to review

**✅ BTC/ETH ACB rollforward correct across years**
- Opening ACB computed from all prior history; all timestamps interpreted as UTC
- Book Cost used over Market Value where present for acquisitions and dispositions
- Network fees parsed as native asset, converted to CAD via spot rate; null spot rate → `missing_fee_valuation` warning
- USD-priced sells use `bookCostCad` for proceeds, not the USD description amount
- Incomplete history marks result as `materially_incomplete`
- USDC excluded from BTC/ETH pools

**✅ USDC rows classified correctly**
- `usdc_cad_conversion` for Send/Receive + `converted @` description → `no_tax_effect`, no review item
- `usdc_send_review` / `usdc_receive_review` for on-chain rows → review queue, correct categories

**✅ Policy-coded rewards treated per documented rules**
- `shakingsats` and SecretSats → income + basis
- `referral_bonus` / `referral_promo_bonus` → income
- `shakesquad_reward` → income + basis
- `cad_interest` → interest income
- `card_cashback_btc` / `card_intro_bonus_3pct` → basis only
- `sif_received` → `no_tax_effect`, no review item (detected via `@username` pattern)
- `sif_sent` → non-deductible outflow
- `card_purchase` → `personal_non_deductible_outflow`, no review item

**✅ Interac directionality correct**
- Interac with credit → `cad_deposit`; Interac with debit → `cad_withdrawal`; both null → `unknown_review`

**✅ Cash-file send/receive classified correctly**
- `receive` + `@username` → `sif_received`; `receive` + email → `cad_receive`
- `send` + `@username` or email → `sif_sent`; other → `unknown_review`

**✅ Crypto-file convert events use two-leg model**
- Debit-only `convert` row → `convert_out_leg`; credit-only → `convert_in_leg`
- Event Linker pairs matched legs into `crypto_swap`; unmatched legs → `unclassified_event` ReviewItem

**✅ USD Sell sub-types distinguished**
- USD credited + null description → `usd_buy_review`; USD debited + `"Sold … USD @ CA$…"` → `usd_sell_review`
- Review item `suggestedActions` differ by direction

**✅ Review queue uses correct categories**
- `unclassified_event` for `unknown_review`, unmatched convert legs, and unmatched-rule rows
- `unknown_reward` for reward-type rows with unrecognized descriptions
- Per-category UI labels and explanations present for all categories

**✅ Annual summaries include federal and provincial estimates**
- All taxable amount fields present; 50% inclusion rate applied for 2021–2026
- Policy version, classifier version, and active law-version identifiers stored in every summary

**✅ Bulk review decisions work correctly**
- `POST /api/v1/sessions/:id/review/bulk` accepts mixed `review_item` and `group` targets
- 1 request + 1 refetch for any batch size; override log has one entry per resolved item
- Rate limiter at 200 req/min; client retries up to 3× with exponential backoff on 429

**✅ Incomplete dossiers clearly marked**
- Missing crypto file with crypto-indicating signals → `MissingRequiredFileError`; `materially_incomplete`
- Missing files, partial coverage, missing basis → visible completeness status with reason

**✅ All unclassified rows appear in review output**
- Zero rows silently dropped; every unresolvable row → `ReviewItem` with reason code and category

**✅ Schema-level reconciliation constraints enforced**
- Link group membership stored in explicit join table with referential integrity
- A single classified row cannot belong to more than one link group

**✅ Limitations disclosed in every report**
- Superficial loss not implemented — conspicuous notice wherever capital losses are displayed
- Network fees note (native asset denomination; null spot rate warning)
- Inclusion rate applied
- USD/USDC policy status
- Book Cost vs. Market Value choice disclosed

**Release gate** — do not ship unless:
- All acceptance criteria above are covered by automated tests
- `pnpm exec turbo run build --force` and `pnpm exec turbo run test --force` pass
- Fixture-based regression tests pass for all six file types, including USDC, SecretSats, `card_purchase`, Interac directionality, `@username` SIF, two-leg convert, USD sub-types, and native-asset fee fixtures
- At least one mixed-language multi-year E2E test is green
- `openapi.yaml` matches all implemented routes
- Known limitations documented in every report
- Superficial loss disclosure appears on every screen showing capital losses

---

## 25. Recommended Improvements

The following improvements are ordered by priority. The first two are pre-launch obligations; the remainder are active product debt.

**1. Implement the superficial loss rule (pre-launch — high compliance risk)**

This is the highest-priority improvement. Users who repurchase the same crypto asset within 30 days of a disposition will have overstated capital losses. The product must display a conspicuous limitation notice on every screen and report section where capital losses are shown until this rule is implemented.

**2. Strengthen completeness scoring through known-activity inference (pre-launch — high reliability risk)**

Completeness scoring currently enforces a hard gate (no crypto file + crypto signals → `MissingRequiredFileError`). Broader inference — e.g. scoring a session as `incomplete` when cash buys exist but no crypto file is uploaded — remains an active improvement obligation. File-presence-only scoring understates the gap and gives users false confidence.

**3. Confirm `card_purchase` French type value (pre-launch — correctness risk for French users)**

The French equivalent of `Card purchase` has not been confirmed from a production French-language export. Until confirmed, French-language cash exports may route card purchase rows to `unknown_review`, inflating the review queue for French users.

**4. Resolve `btc_to_eth` / `eth_to_btc` enum migration (pre-launch — data integrity risk)**

Sessions processed before the two-leg convert model was introduced may have persisted the deprecated codes. A migration strategy must be designed and executed before the deprecated codes can be safely removed.

**5. Formalize USD/USDC policy as medium-priority product debt**

The current USD/USDC treatment is operationally safe but explicitly incomplete. This should be tracked as medium-priority product debt with a clear owner and exit criteria — see §29.

**6. Add schema-level enforcement for event linking**

The reconciliation architecture is conceptually sound, but correctness currently depends on service-layer discipline. The required additions are specified in §16.3.

**7. Require measured ACB performance profiling under concurrent workloads**

O(n) complexity is necessary but not sufficient evidence of production readiness. See §28.4.

| Priority | Improvement | Why it matters |
|---|---|---|
| Pre-launch | Superficial loss rule | Overstated losses for 30-day repurchases; release risk until implemented or conspicuously disclosed |
| Pre-launch | Completeness scoring — known-activity inference | File-presence-only scoring understates incompleteness |
| Pre-launch | `card_purchase` French type confirmation | French users may see inflated review queues |
| Pre-launch | `btc_to_eth`/`eth_to_btc` migration | Data integrity risk for sessions processed before two-leg model |
| Medium | Formal USD/USDC policy module | Heavy USD/USDC users face disproportionate review burden |
| Medium | Schema-level link group enforcement | Reconciliation correctness depends on service discipline until DB constraints added |
| Medium | Measured ACB performance profiling | O(n) argument necessary but not sufficient before production rollout |
| Medium | Dedicated review inbox with per-category filters and bulk actions | Operationally required for review-heavy sessions |
| Medium | Québec disclosure companion report | Turns TP-21.4.39 into a usable output |
| Low | Wallet frequency analysis heuristic | Reduces review volume for repeated addresses |
| Low | Temporal pairing heuristics for internal transfers | Auto-suggest internal transfer for tight Buy → Send sequences |
| Low | Third-party tax-law rate API sync | Reduces manual maintenance burden |

---

## 26. Configuration

### 26.1 Linking / reconciler

| Variable | Default | Description |
|---|---|---|
| `SHAKEPAY_LINKING_TIME_WINDOW_SECONDS` | `120` | Max time delta (seconds) between cash and crypto legs for timestamp scoring; also used for convert two-leg pairing |
| `LINKING_TIME_WINDOW_SECONDS` | (same) | Alias for the above |

### 26.2 API

| Variable | Description |
|---|---|
| `PORT` | API listen port (default commonly `3001`) |
| `CORS_ORIGIN` | CORS allowlist if set |
| `SHAKEPAY_API_KEY` | API key for `requireAuth` middleware in production |

### 26.3 Web

| Variable | Description |
|---|---|
| `SHAKEPAY_API_PORT` | Port Vite proxy uses for `/api` — must match API `PORT` |

---

## 27. Operations

### 27.1 Ports

- API: `PORT` in `apps/api/.env` (default commonly `3001`)
- Web (Vite): `apps/web` dev server (often `4000`); `SHAKEPAY_API_PORT` in `apps/web/.env.local` must match the API

### 27.2 Rate limiting

The global rate limiter is configured in `apps/api/src/middleware/rate-limiter.js`:
- Global limit: **200 requests per minute**
- Route-specific limits exist for upload and process endpoints

The web client implements client-side retry: on a `429`, retries up to 3 times with exponential backoff (1 s → 2 s → 4 s), or uses the `Retry-After` header if present.

### 27.3 Health

```
GET /health   — API liveness (no auth required)
```

### 27.4 Authentication

`apps/api` uses `requireAuth` middleware for all `/api/v1` routes. Configure `SHAKEPAY_API_KEY` or equivalent per `apps/api/src/middleware/auth.ts` for production deployments.

---

## 28. ACB Engine Performance

### 28.1 Implementation approach

Average cost basis for BTC and ETH is computed **in memory** over **chronologically sorted** normalized events for the session.

**Entry point:** `computeBasisWithUpdates` in `packages/basis`, called from `packages/orchestration/src/process-session.service.ts`.

### 28.2 Complexity

Pool operations are **O(1)** per relevant event. Overall complexity is **O(n)** in the number of classified events that affect basis for the session.

### 28.3 Large histories

For tens of thousands of rows, the dominant costs are typically parsing and classification, timestamp sorting, and in-memory pool updates. Do not optimize without profiling data.

### 28.4 Required performance validation before production

O(n) complexity is a necessary condition, not a sufficient one. Shakepay's product shape generates years of small reward events. Before treating the ACB engine as operationally hardened, the following validation is required:

- **Heap profiling:** Run against a large synthetic fixture (10,000+ events per session) under realistic concurrency (≥10 concurrent sessions). Measure peak heap usage and GC pause duration.
- **GC observation:** Confirm GC pause does not cause P99 latency spikes on the `/process` endpoint under sustained load.
- **Concurrency testing:** Verify concurrent sessions do not share mutable pool state.

---

## 29. USD/USDC Policy Roadmap

### 29.1 Current behavior (MVP)

- USD rows land in the review queue with `usd_review` treatment. Not automatically included in capital gains.
- USDC conversion legs (`converted @` description) are auto-resolved as `no_tax_effect`.
- USDC on-chain rows land in the review queue under `uncertain_send` / `uncertain_receive` categories.
- USD `Sell` rows are distinguished by direction: `usd_buy_review` (CAD→USD) and `usd_sell_review` (USD disposal).

### 29.2 Risk

Heavy USD/USDC volume can fill the review queue and hurt UX even when the review UI groups USD into its own step. This is the **main UX bottleneck** for affected users. The `usd_flow_cluster` grouping design is a useful mitigation but is not a substitute for a settled policy module.

### 29.3 Mitigations (product)

1. **Wizard:** Dedicated USD bucket in Step 3 (Categorize) so users are not scrolling mixed categories.
2. **`usd_flow_cluster` grouping:** USD/USDC legs sharing timing and amounts grouped as a unit in the review queue.
3. **Direction-aware `suggestedActions`:** `usd_buy_review` and `usd_sell_review` items surface different prompts so users are not confused by the CAD↔USD directionality.
4. **Results / exports:** Prominent caveat that USD/USDC policy is conservative and may require professional advice.

### 29.4 Exit criteria (formal USD policy module)

Promote from "review-first" to "ledger rules" only when **all** of the following are agreed and implemented:

1. Written policy for USD/USDC as **capital vs income** and **FX** handling (jurisdiction-specific, including Québec).
2. Classifier updates to apply that policy, with tests covering all affected normalized codes.
3. User-facing copy and audit trail fields updated to reflect the new treatment.
4. Migration path for existing sessions.

Until these criteria are met, treat USD/USDC as **explicitly incomplete** and track as medium-priority product debt with a clear owner and target milestone.

---

## 30. Engineering Reference

### 30.1 Classification rules (code pointers)

| Concern | File |
|---|---|
| Rule registry and priority | `packages/classification/src/rule-registry.ts`, `priority-order.ts` |
| Crypto family rules (incl. USDC, `@username` SIF, two-leg convert) | `packages/classification/src/family-rules/crypto.rules.ts` |
| Cash family rules (incl. `card_purchase`, Interac directionality, cash send/receive) | `packages/classification/src/family-rules/cash.rules.ts` |
| USD family rules (incl. `usd_buy_review` / `usd_sell_review` distinction) | `packages/classification/src/family-rules/usd.rules.ts` |
| Shared description patterns | `packages/classification/src/description-patterns.ts` |
| Classification engine | `packages/classification/src/classification-engine.ts` |

### 30.2 Database schema (code pointer)

```
packages/db/src/schema/index.ts   (Drizzle ORM table definitions)
```

If the markdown in §16 and the Drizzle schema diverge, **trust Drizzle + migrations** and update the doc.

### 30.3 Inclusion rates (code pointer)

```
packages/domain/src/constants/inclusion-rates.ts
```

Update this file when the capital gains inclusion rate changes. Do not hardcode rates in business logic, tests, or report templates.

### 30.4 Normative API contract

```
openapi.yaml   (in the docs/ or project root directory)
```

Keep `openapi.yaml` updated whenever routes change. The narrative in §15 is secondary; the YAML is the source of truth for API consumers and contract tests.

---

## Official References

Verified against official public pages on 2026-03-29:

- Revenu Québec publishes `TP-21.4.39-V` as the cryptoasset return, requiring completion since 2024 for taxpayers who own, receive, dispose of, or use cryptoassets.
- CRA guidance states that, when crypto is disposed of on capital account, the applicable fraction of the capital gain is included in income as a taxable capital gain.
- The June 2024 federal budget proposed raising the capital gains inclusion rate; Finance Canada deferred the proposal on January 31, 2025; the CRA subsequently stated it was administering the enacted one-half rate; and the proposed increase was later cancelled. The currently enacted rate of one-half (50%) applies.
- CRA and Revenu Québec both publish current annual personal tax rate tables, supporting a maintained tax-table source rather than hardcoded values.

Tax tables, brackets, and inclusion rates must always be sourced from the maintained seed layer and official publications — not from values embedded in this document.

---

## Document History

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-01-xx | Initial specification |
| 2.1 | 2026-03-28 | Supplementary chapter: extraction vs. classification gaps; review queue behaviour |
| 3.0 | 2026-03-28 | Merged all sources; USDC classification; shared `DESC_PATTERNS`; `unclassified_event` category; USDC ACB exclusion |
| 3.1 | 2026-03-28 | `normalizeAsset` USDC support; parser unit tests; USDC E2E pipeline test; bulk review endpoint; client retry; rate limiter 200 req/min |
| 4.0 | 2026-03-28 | Integrated operational docs: monorepo layout (§4.1); unified `process` endpoint; corrected API base path to `/api/v1` and routes to `sessions`; book cost vs. market value (§9.2); network fees (§9.3); SecretSats (§1.4, §7.3); superficial loss disclosure (§9.1, §20.5, §21.7); inclusion rate specifics (§10.2); Québec line numbers (§12.4); decision log as §22; new §§26–30; schema source-of-truth note (§16.1); `usd_flow_cluster` link type (§8.5); `cash_sell_leg` code (§6.4); SIF resolved to `no_tax_effect` throughout |
| 4.1 | 2026-03-29 | Integrated review improvements: corrected inclusion rate history (50% flat for 2021–2026, §10.2, §22.4); superficial loss elevated to near-term release risk (§9.1, §21.7, §24, §25); schema-level link group enforcement added (§8.1, §16.3, §23, §24); ACB performance profiling required (§28.4); USD/USDC named as main UX bottleneck (§29.2); bulk review endpoint operational importance noted (§15.5); Book Cost vs. Market Value design strength noted (§9.2); Québec first-class jurisdiction rationale noted (§12.2); extraction vs. classification operational note added (§7.6); §25 reordered and expanded; §20.5 disclosure 10 added; §1.5 specification vs. production compliance framing added |
| 4.2 | 2026-03-29 | CSV-reconciliation and internal-consistency pass: `card_purchase` type added (§3.4, §6.4, §7.1, §7.4, §19.1, §19.3, §20.1, §24, §25); Interac directionality split by credit/debit field (§7.4, §19.3); cash-file `send`/`receive` rules added with `@username` → SIF mapping (§7.4, §6.4, §19.3); `cad_receive` and `cad_send` codes defined (§6.4); SIF `@username` detection added to crypto classifier with note that `shake it forward` text is unconfirmed in production (§7.3, §7.4, §7.5, §19.3); two-row `convert` event model added with Event Linker pairing requirement (§7.4, §6.4, §8.7, §19.2, §19.3); `btc_to_eth`/`eth_to_btc` deprecated in favour of two-leg codes (§6.4, §21.11, §22.6, §23, §24, §25); USD `Sell` sub-types distinguished by direction with `usd_buy_review` code (§6.4, §7.4, §13.3, §19.3, §29.3); fee denomination corrected from CAD to native asset with spot-rate conversion spec and `missing_fee_valuation` warning (§9.3, §9.9, §14.2, §18.2, §19.1, §19.3, §20.5); USD-priced crypto sells documented with `bookCostCad` proceeds rule (§7.4, §9.5, §19.3); filename examples updated to unsuffixed production format as primary (§3.1); file-detection pseudocode made two-stage to resolve `crypto_or_usd_pending` type mismatch (§5.2); missing crypto file behaviour made explicit and normative (§3.2, §4.4, §14.1, §18.1, §22.5); negative pool behaviour consolidated to one model (§13.5); linker amount comparison updated to Book Cost priority (§8.2); year-boundary timezone rule set to UTC (§9.8, §10.1, §19.3, §21.6); `MissingRequiredFileError` added to hard error table (§18.1); `DESC_PATTERNS` extended with `sifUsername`, `feeNative`, `usdSoldDesc` (§7.2); open decisions §22.6 expanded; known issues §21.10, §21.11 added |

---

## Change Log (v4.1 → v4.2)

| Issue fixed | Sections changed | What changed | Type |
|---|---|---|---|
| `Card purchase` type absent from type dictionary and classification rules | §3.4, §6.4, §7.1, §7.4, §19.1, §19.3, §20.1, §24, §25 | Added `card_purchase` canonical token, `NormalizedCode`, priority table entry, cash classification rule, unit test, regression fixture, policy position, DoD criterion, and pre-launch improvement | Alignment to real Shakepay files |
| `Interac e-Transfer` mapped unconditionally to `cad_deposit`; outbound rows misclassified | §7.4, §19.3 | Split into two directional rows keyed on `credit`/`debit` field presence; added note for both-null edge case; added two regression fixtures | Alignment to real Shakepay files |
| Cash-file `Send` and `Receive` types absent from cash classification table | §6.4, §7.4, §19.3 | Added classification rules for `receive`/`send` in cash file with `@username` → SIF, email → `cad_receive`/`cad_send`, fallback → review; defined `cad_receive` and `cad_send` NormalizedCodes; added regression fixtures | Alignment to real Shakepay files |
| SIF regex `shake\s*it\s*forward` does not match `@username` production format | §7.3, §7.4, §7.5, §19.3 | Added `@username` `^@\w+$` pattern as primary SIF signal in pseudocode, rule notes, and crypto-file table; noted that text-based regex is unconfirmed in production; added regression fixtures | Alignment to real Shakepay files |
| Crypto-file `Convert` assumed single row; production uses two rows | §6.4, §7.1, §7.4, §8.7, §19.2, §19.3, §21.11, §22.6, §23, §24, §25 | Added two-leg model (`convert_out_leg`/`convert_in_leg`); deprecated `btc_to_eth`/`eth_to_btc`; added Event Linker pairing subsection §8.7; added integration and regression fixtures; added known issue §21.11 and open decision | Alignment to real Shakepay files |
| USD `Sell` not distinguished by direction | §6.4, §7.4, §13.3, §19.3, §29.3 | Split into `usd_buy_review` and `usd_sell_review` based on asset direction and description; added directional `suggestedActions` note; added direction-aware mitigation to §29.3; added regression fixtures | Alignment to real Shakepay files |
| Network fees described as "in CAD"; production fees are in native crypto asset | §9.3, §9.9, §14.2, §18.2, §19.1, §19.3, §20.5 | Rewrote §9.3 to specify native-asset fee parsing + spot-rate conversion; updated pseudocode in §9.9; added `missing_fee_valuation` soft warning; updated disclosure §20.5 item 9; added unit test cases and regression fixtures | Alignment to real Shakepay files |
| USD-priced `Sell` descriptions (`Sold @ US$…`) not handled; proceeds source ambiguous | §7.4, §9.5, §19.3 | Added note to §7.4 that USD-priced sells still use `bookCostCad`; added normative proceeds source rule to §9.5; added regression fixture | Alignment to real Shakepay files |
| Filename examples implied `_english`/`_french` suffix as primary format | §3.1 | Updated §3.1 table to show unsuffixed production filenames as primary; suffixed variants listed as accepted alternates | Alignment to real Shakepay files |
| File-detection pseudocode returned `crypto_or_usd_summary` which does not match `ImportFamily` type | §5.2 | Made detection two-stage; Stage 1 returns `crypto_or_usd_pending` (internal only); Stage 2 resolves to `crypto_summary` or `usd_summary`; added note that intermediate label must never be persisted | Internal consistency |
| Missing crypto file behaviour inconsistent across sections | §3.2, §4.4, §14.1, §18.1, §22.5 | Added normative rule; `MissingRequiredFileError` added to hard error table; §22.5 updated | Internal consistency |
| `NegativePoolError` and blocking `ReviewItem` described as alternatives | §13.5, §18.1 | Consolidated into one normative rule; language suggesting alternatives superseded | Internal consistency |
| Linker used `marketValueCad` for amount match; ACB engine prefers `bookCostCad` | §8.2 | Updated amount match condition to `bookCostCad` (fallback `marketValueCad`); added rationale note | Internal consistency |
| Timezone for year-boundary computation unresolved | §9.8, §10.1, §19.3, §21.6 | Added normative UTC rule; updated integration test guidance; noted in §21.6 that the existing risk is now resolved | Internal consistency |
| `DESC_PATTERNS` missing entries for new detection needs | §7.2 | Extended with `sifUsername`, `feeNative`, `usdSoldDesc` | Internal consistency |

---

## Remaining Open Questions (v4.2)

| Issue | Why it remains open | Decision needed | Temporary wording used in doc |
|---|---|---|---|
| French equivalent of `Card purchase` type value | No French-language production export containing card purchase rows has been confirmed. | Obtain and analyse a French-language cash summary export from a user who has made card purchases. Confirm the exact `Type` column string before adding it to `typeDictionary.ts` and §3.4. | "*(not yet observed in production French exports — must be confirmed from a French-language export before implementation; treat unrecognised French equivalents as `unknown_review` pending confirmation)*" in §3.4; known issue §21.10 |
| `btc_to_eth` / `eth_to_btc` enum migration | These codes may exist in persisted session data from before the two-leg model was introduced. Removing them requires a migration strategy. | Decide: (a) retain as guards mapping to `unknown_review`; (b) write a one-time migration to reprocess affected sessions; (c) keep both code paths with a feature flag. | `// DEPRECATED` annotation in §6.4; known issue §21.11; open decision §22.6 |
| Non-`@username`, non-email cash `Send` description patterns | Unknown what other patterns exist in production for cash `Send` rows. The current rule routes them to `unknown_review`. | Analyse a broader sample of cash `Send` rows to identify recurring patterns that warrant dedicated rules. | Routes to `unknown_review` / `non_taxable_review` with a ReviewItem; §7.4 note on `cad_send` scope |
| `@username` pattern scope — character set beyond `\w` | Shakepay usernames may permit hyphens or other characters not covered by `\w`. | Confirm the full Shakepay username character set from account creation rules or API docs and update the regex accordingly before shipping. | `^@\w+$` used throughout; noted as illustrative in pseudocode; open decision §22.6 |
| `cad_send` classification for non-`@username`, non-email cash sends | `cad_send` is defined in the enum but currently only reachable via the email-address send path. Whether it should have a broader use is undecided. | Determine if there are other legitimate CAD P2P send patterns that should map to `cad_send`. If not, consider whether `cad_send` should remain in the enum. | `cad_send` exists in enum; scope limited per §7.4 note |
| USD `Sell` row with both amount fields null | A `Sell` row where both `Amount Credited` and `Amount Debited` are null is an edge case not yet assigned a code. Whether such rows exist in production is unknown. | Determine whether such rows exist and what they represent. | Falls to `unknown_review`; no explicit rule |

---

*This document does not replace legal or tax advice. Tax treatment depends on individual facts; the engine encodes product policy and user choices, not a binding CRA or Revenu Québec position.*
