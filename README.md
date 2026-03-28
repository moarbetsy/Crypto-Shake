# Shakepay Canada Tax Engine
### Complete Technical Documentation — v2.1 — March 28, 2026

> **Scope:** Individual taxpayers · Personal use only · Canada Federal + All Provinces & Territories
>
> **Not in scope:** Corporate tax · Employment income · Business inventory · Non-Shakepay exchanges · Legal or tax advice
>
> **Audience:** Product, backend, parser, tax-engine, QA, and reporting teams
>
> **Status:** Implementation-ready baseline specification for a Shakepay-only personal-use Canadian tax workflow.

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
22. [Open Decisions](#22-open-decisions)
23. [Implementation Order](#23-implementation-order)
24. [Definition of Done](#24-definition-of-done)
25. [Recommended Improvements](#25-recommended-improvements)

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
| `taxableRewardIncomeCad` | ShakingSats, referrals, ShakeSquad, interest |
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
| Referral bonus | Taxable income | High |
| Promotional referral bonus | Taxable income | Medium |
| ShakeSquad rewards | Taxable income + crypto basis | Medium |
| CAD interest | Interest income | High |
| Card cashback in BTC | Purchase rebate — basis only, not income | **Prudent default** |
| 3% card intro bonus | Purchase rebate — basis only, not income | **Prudent default** |
| Shake It Forward received | Not auto-included — review flag | **Prudent default** |
| Shake It Forward sent | Personal non-deductible outflow | Medium |

Positions marked **Prudent default** are not confirmed published CRA or Revenu Québec positions specific to Shakepay cashback. They must be disclosed in every report.

---

## 2. Scope

| In scope | Out of scope |
|---|---|
| Canadian individuals; personal-use tax treatment; annual tax summaries; Québec disclosure support; BTC and ETH ACB tracking; bilingual Shakepay import support | Corporate tax; business inventory treatment; self-employment treatment; accountant-grade filing automation; legal advice; non-Shakepay records unless imported separately |

---

## 3. Input Contract

The importer must assume the user uploads a bundle of summary files rather than one monolithic transaction-history CSV. This **dossier model** is the foundation of the entire input contract.

### 3.1 Accepted file families

| File | Family | Language |
|---|---|---|
| `crypto_transactions_summary_english.csv` | `crypto_summary` | `en` |
| `crypto_transactions_summary_french.csv` | `crypto_summary` | `fr` |
| `cash_transactions_summary_english.csv` | `cash_summary` | `en` |
| `cash_transactions_summary_french.csv` | `cash_summary` | `fr` |
| `usd_transactions_summary_english.csv` | `usd_summary` | `en` |
| `usd_transactions_summary_french.csv` | `usd_summary` | `fr` |

The system must support 1 to 6 files per import dossier and must explicitly score completeness when files are missing.

### 3.2 Minimum viable input set

| File | Requirement |
|---|---|
| Crypto summary | **Required** — needed for dispositions, book cost, market value, asset movement |
| Cash summary | **Strongly recommended** — needed for CAD rewards and fiat reconciliation |
| USD summary | **Optional** — required only if user has Shakepay USD activity |

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

> **Note:** USD summary files use the identical schema as crypto summary files. The engine distinguishes them by the asset values present (`USD` vs `BTC`/`ETH`), not by filename.

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

---

## 4. System Architecture

### 4.1 Layer overview

| # | Layer | Responsibility |
|---|---|---|
| 1 | **Import & Detection** | Accept files; detect family and language from headers; validate schema |
| 2 | **Canonicalization** | Map bilingual headers to canonical fields; produce `CanonicalRow` records |
| 3 | **Classification** | Assign `normalizedCode` and `taxTreatment` using the priority hierarchy |
| 4 | **Event Linking** | Group related cash + crypto rows into single `EconomicEvent` records |
| 5 | **Basis Ledger** | Maintain per-asset ACB pools; compute acquisition cost and disposition gain/loss |
| 6 | **Annual Ledger** | Aggregate events by tax year into reward, interest, capital gain/loss ledgers |
| 7 | **Tax Estimator** | Apply federal and provincial brackets to taxable amounts |
| 8 | **Output Builder** | Assemble annual summary, audit trail, Québec disclosure, completeness score |
| 9 | **Review Queue** | Surface flagged rows; accept user decisions; log every override |

### 4.2 Processing pipeline

```
User uploads 1–6 files
  → FileDetector         identifies family + language from headers
  → SchemaValidator      confirms required columns; rejects invalid files
  → ColumnMapper         maps raw headers → CanonicalRow
  → ClassificationEngine assigns normalizedCode + taxTreatment
  → EventLinker          reconciles matching cash + crypto rows
  → BasisLedger          processes acquisitions and dispositions chronologically
  → AnnualLedgerBuilder  groups events by tax year
  → CoverageScorer       assesses completeness; emits warnings
  → User provides province + optional other income per year
  → TaxEstimator         applies federal + provincial rates
  → OutputBuilder        assembles report + Québec disclosure + audit trail
```

### 4.3 Import state machine

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

### 4.4 Design principles

- The parser must not decide tax payable. It only normalizes, classifies, and computes taxable amounts and ledger effects.
- The tax engine must consume annual taxable outputs and versioned tax tables as a separate layer.
- The reconciler is mandatory. Without it, the product risks double-counting the fiat and crypto legs of the same economic event.

### 4.5 Folder structure

```
src/
  import/
    detector.ts              # family + language detection
    canonicalizer.ts         # header normalization → CanonicalRow
    headerMap.ts             # HEADER_MAP EN + FR
    typeDictionary.ts        # TYPE_DICTIONARY bilingual values
  classifier/
    classifyReward.ts        # reward pattern matching
    classifyTransaction.ts   # standard transaction classification
    ruleRegistry.ts          # ordered priority list
    taxTreatments.ts         # TaxTreatment enum + descriptions
  reconciler/
    eventLinker.ts           # cross-file event matching
    matchScorer.ts           # timestamp + amount + type scoring
    linkGroup.ts             # LinkGroup schema + builder
  ledger/
    assetPool.ts             # BTC/ETH pooled ACB
    acquisitionEngine.ts
    dispositionEngine.ts
    yearBoundary.ts          # opening ACB rollforward per year
  tax/
    federalEngine.ts
    provincialEngine.ts      # dispatches to province modules
    provinces/               # one file per jurisdiction
      QC.ts  ON.ts  BC.ts  AB.ts  MB.ts  SK.ts
      NS.ts  NB.ts  NL.ts  PE.ts
      YT.ts  NT.ts  NU.ts
    taxLawRegistry.ts        # loads + caches versioned law tables
    inclusionRate.ts         # capital gains inclusion rate by year
  review/
    reviewQueue.ts
    overrideLog.ts           # immutable audit of user decisions
  reporting/
    annualSummary.ts
    auditTrail.ts
    quebecDisclosure.ts      # TP-21.4.39 support report
    completenessScorer.ts
  models/
    CanonicalRow.ts
    NormalizedEvent.ts
    EconomicEvent.ts
    AnnualSummary.ts
    ReviewItem.ts
    TaxLawVersion.ts
  api/
    routes/
      imports.ts  files.ts  classify.ts
      reconcile.ts  calculate.ts  review.ts  reports.ts
    middleware/
      validateUpload.ts
      errorHandler.ts
  db/
    schema.sql               # single source of truth for all tables
    migrations/
    seeds/
      taxBrackets/           # one JSON per jurisdiction per year
  utils/
    decimal.ts               # decimal-safe numeric helpers
    dateHelpers.ts
    hashRow.ts               # deterministic row ID hashing
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

```python
def detect_family(headers: list[str]) -> tuple[str, str]:
    hs = set(headers)

    cash_en  = {"Debit", "Credit", "Balance"}
    cash_fr  = {"Débit", "Crédit", "Solde"}
    crypto_en = {"Amount Debited", "Asset Debited", "Amount Credited",
                 "Asset Credited", "Market Value", "Book Cost"}
    crypto_fr = {"Montant débité", "Actif débité", "Montant crédité",
                 "Actif crédité", "Valeur du marché", "Coût comptable"}

    if cash_en <= hs:    return ("cash_summary", "en")
    if cash_fr <= hs:    return ("cash_summary", "fr")
    if crypto_en <= hs:  return ("crypto_or_usd_summary", "en")
    if crypto_fr <= hs:  return ("crypto_or_usd_summary", "fr")

    raise SchemaDetectionError("Unknown schema")
```

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
  policyVersion:             string;  // e.g. "2026-03-28"
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
  | "cad_send"
  | "cad_receive"
  | "cash_buy_leg"          // provisional reconciliation-only code
  // Crypto acquisitions
  | "cad_to_btc"
  | "cad_to_eth"
  // Crypto dispositions
  | "btc_to_cad"
  | "eth_to_cad"
  // Crypto-to-crypto swaps
  | "btc_to_eth"
  | "eth_to_btc"
  // Crypto transfers
  | "btc_send"
  | "btc_receive"
  | "eth_send"
  | "eth_receive"
  // USD — implementation review codes
  | "usd_receive_review"
  | "usd_send_review"
  | "usd_sell_review"
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

Apply categories in this exact order. The first match wins.

| Priority | Category | Rationale |
|---|---|---|
| 1 | `sif_received` / `sif_sent` | Prevent absorption into reward income bucket |
| 2 | `cad_interest` | Clearest treatment — classic interest income |
| 3 | `referral_bonus` / `referral_promo_bonus` | Autonomous conditional bonus |
| 4 | `shakesquad_reward` | Autonomous reward in BTC |
| 5 | `shakingsats` | Primary daily BTC reward |
| 6 | `card_cashback_btc` / `card_intro_bonus_3pct` | Purchase rebate — not income by default |
| 7 | `cad_deposit` / `cad_withdrawal` | Fiat ledger events |
| 8 | `cad_to_btc` / `cad_to_eth` | Crypto acquisition |
| 9 | `btc_to_cad` / `eth_to_cad` | Crypto disposition |
| 10 | `btc_to_eth` / `eth_to_btc` | Crypto swap |
| 11 | Send / receive transfers | Transfer review |
| 12 | USD events | Policy review pending |
| 13 | `unknown_reward_review` / `unknown_review` | Final fallback |

### 7.2 Reward pattern dictionary

All patterns must be case-insensitive regex — not `includes()` string checks. Unknown reward descriptions must route to `unknown_reward_review` and must never be silently dropped.

```typescript
const REWARD_PATTERNS: Array<{ re: RegExp; code: NormalizedCode }> = [
  { re: /shake\s*it\s*forward/i,              code: "sif_received"          },
  { re: /interest|intérêts/i,                 code: "cad_interest"          },
  { re: /promo.*ref(erral|érencement)/i,      code: "referral_promo_bonus"  },
  { re: /ref(erral|érence|érencement)/i,      code: "referral_bonus"        },
  { re: /shakesquad/i,                        code: "shakesquad_reward"     },
  { re: /shakingsats/i,                       code: "shakingsats"           },
  { re: /cashback|remise\s*carte/i,           code: "card_cashback_btc"     },
  { re: /3\s*%.*card|card.*3\s*%|intro.*bonus|welcome.*bonus/i,
                                              code: "card_intro_bonus_3pct" },
];
```

### 7.3 Per-family classification rules

#### Cash summary

| `typeCanonical` | `descriptionRaw` pattern | `normalizedCode` | `taxTreatment` |
|---|---|---|---|
| `interac` | any | `cad_deposit` | `no_tax_effect` |
| `buy` | any | `cash_buy_leg` *(provisional)* | `cash_funding_only` |
| `reward` | referral pattern | `referral_bonus` | `taxable_income` |
| `reward` | promo referral pattern | `referral_promo_bonus` | `taxable_income` |
| `reward` | interest pattern | `cad_interest` | `interest_income` |
| `reward` | unknown | `unknown_reward_review` | `non_taxable_review` |
| `withdrawal` | any | `cad_withdrawal` | `no_tax_effect` |

#### Crypto summary

| `typeCanonical` | Asset context | `normalizedCode` | `taxTreatment` |
|---|---|---|---|
| `buy` | credited: BTC | `cad_to_btc` | `crypto_acquisition` |
| `buy` | credited: ETH | `cad_to_eth` | `crypto_acquisition` |
| `sell` | debited: BTC | `btc_to_cad` | `taxable_disposition` |
| `sell` | debited: ETH | `eth_to_cad` | `taxable_disposition` |
| `convert` | debited: BTC, credited: ETH | `btc_to_eth` | `taxable_disposition` + `crypto_acquisition` |
| `convert` | debited: ETH, credited: BTC | `eth_to_btc` | `taxable_disposition` + `crypto_acquisition` |
| `reward` | ShakingSats desc | `shakingsats` | `taxable_income` |
| `reward` | ShakeSquad desc | `shakesquad_reward` | `taxable_income` |
| `reward` | cashback desc | `card_cashback_btc` | `purchase_rebate` |
| `reward` | 3% desc | `card_intro_bonus_3pct` | `purchase_rebate` |
| `reward` | SIF desc | `sif_received` | `non_taxable_review` |
| `send` | debited: BTC | `btc_send` | `transfer_or_disposition_review` |
| `send` | debited: ETH | `eth_send` | `transfer_or_disposition_review` |
| `receive` | credited: BTC | `btc_receive` | `transfer_or_income_review` |
| `receive` | credited: ETH | `eth_receive` | `transfer_or_income_review` |

#### USD summary

USD events are ingested, normalized, and preserved in a separate review ledger. They are not included in BTC or ETH ACB pools by default. The UI must display a visible notice that USD activity requires manual review.

| `typeCanonical` | Asset context | `normalizedCode` | `taxTreatment` |
|---|---|---|---|
| `receive` | credited: USD | `usd_receive_review` | `usd_review` |
| `send` | debited: USD | `usd_send_review` | `usd_review` |
| `sell` | USD context | `usd_sell_review` | `usd_review` |

### 7.4 Classification pseudocode

```python
def classify_transaction(tx):
    d = tx.description_raw.lower()

    # Priority 1 — SIF (must come before referral check)
    if re.search(r"shake\s*it\s*forward", d):
        if "sent" in d:
            return code("sif_sent", "personal_non_deductible_outflow")
        return code("sif_received", "non_taxable_review", review=True)

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

    # Priority 5 — ShakingSats
    if re.search(r"shakingsats", d):
        return code("shakingsats", "taxable_income")

    # Priority 6 — Card cashback
    if re.search(r"cashback|remise\s*carte", d):
        return code("card_cashback_btc", "purchase_rebate")
    if re.search(r"3\s*%.*card|intro.*bonus", d):
        return code("card_intro_bonus_3pct", "purchase_rebate")

    # Standard transaction fallthrough
    return classify_standard_transaction(tx)
```

---

## 8. Event Linking & Reconciliation

### 8.1 Why reconciliation is required

A single economic buy event appears in two files:
- **Cash file:** CAD debit for the purchase
- **Crypto file:** BTC or ETH credit for the same purchase

Without linking, the engine double-counts acquisitions and distorts ACB.

> **This is the most important correctness requirement in the entire engine.**

### 8.2 Matching algorithm

For each provisional `cash_buy_leg` row, search for a candidate crypto buy row matching:

| Signal | Condition | Weight |
|---|---|---|
| Timestamp proximity | Within ±120 seconds | High |
| Amount match | `abs(cashDebit − cryptoMarketValueCad) ≤ 0.05` | High |
| Type pair | `cash_buy_leg` ↔ `cad_to_btc` or `cad_to_eth` | Required |
| Asset | Crypto credited is BTC or ETH | Medium |

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

### 8.6 Internal transfers (send/receive between own wallets)

When a user moves crypto between accounts they own (e.g., Shakepay → external wallet → Shakepay), this is treated as a non-disposition event. The engine carries the existing ACB forward unchanged. No gain or loss is recorded at the time of the transfer. Gain or loss is only recognized when the asset is subsequently disposed of in a taxable event (e.g., sold to CAD, swapped, or sent to a third party).

Because ownership context cannot be proven automatically, all `btc_send`, `eth_send`, `btc_receive`, and `eth_receive` events default to `transfer_or_disposition_review` or `transfer_or_income_review` until the user confirms them as internal.

---

## 9. ACB Ledger Engine

### 9.1 Basis method

Use **pooled average cost basis** per asset — the standard Canadian method for individuals. Maintain separate pools for BTC and ETH. USD must not enter either pool.

```typescript
interface AssetPool {
  asset:          "BTC" | "ETH";
  quantity:       Decimal;
  totalAcbCad:    Decimal;
}
```

Computed property: `perUnitAcb = totalAcbCad / quantity`

### 9.2 Acquisition events

| Source | Income effect | ACB effect |
|---|---|---|
| Taxable reward in crypto | `incomeEffectCad += fmvCad` | `totalAcbCad += fmvCad` |
| Cashback / rebate in crypto | None | `totalAcbCad += fmvCad` |
| CAD-to-crypto buy | None | `totalAcbCad += purchaseCostCad` |
| Crypto-to-crypto swap (incoming leg) | None | `totalAcbCad += proceedsOfOutgoingLeg` |
| Confirmed internal transfer | None | Preserve existing basis only |
| Unconfirmed receive | None | `ReviewItem` — review required |

### 9.3 Disposition logic

```
perUnitAcb       = pool.totalAcbCad / pool.quantity
acbAllocated     = perUnitAcb × quantityDisposed
gainLoss         = proceedsCad − acbAllocated

pool.quantity    -= quantityDisposed
pool.totalAcbCad -= acbAllocated
```

### 9.4 Crypto-to-crypto swap (two-step)

```python
def process_swap(from_asset, from_qty, to_asset, to_qty, cad_value, ledger):
    # Step 1 — dispose outgoing asset
    acb_allocated = ledger.allocate_acb(from_asset, from_qty)
    gain_loss = cad_value - acb_allocated
    ledger.record_disposition(from_asset, from_qty, cad_value, acb_allocated)

    # Step 2 — acquire incoming asset at same CAD value
    ledger.add_basis(to_asset, to_qty, cad_amount=cad_value)
```

### 9.5 Year boundary / rollforward

- Opening quantity and ACB for year Y are computed from **all prior events**, not just the prior year.
- Current-year dispositions must use prior-year ACB where applicable.
- If historical basis is incomplete, mark `completenessStatus = "materially_incomplete"` and label the gain/loss as estimated.
- Negative pool quantity must generate a `ReviewItem` with `severity: "blocking"`.

> Prior-year history cannot be ignored when it affects the cost of assets disposed of in the current year.

### 9.6 Acquisition and disposition pseudocode

```python
def process_crypto_receipt(tx, is_taxable, ledger):
    fmv   = tx.market_value_cad
    qty   = tx.amount_credited
    asset = tx.asset_credited

    if is_taxable:
        ledger.income += fmv

    ledger.add_basis(asset=asset, qty=qty, cad_amount=fmv)


def process_disposition(tx, asset, qty_disposed, proceeds_cad, ledger):
    acb_allocated = ledger.allocate_acb(asset, qty_disposed)
    gain_loss = proceeds_cad - acb_allocated

    ledger.record_capital_event({
        "asset":     asset,
        "qty":       qty_disposed,
        "proceeds":  proceeds_cad,
        "acb":       acb_allocated,
        "gain_loss": gain_loss,
    })
```

---

## 10. Annual Tax Calculation

### 10.1 Calculation flow

For each selected tax year:

1. **Filter** income events and capital events for that calendar year only
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

**Implementation rule:** the capital-gains inclusion rate must always be loaded from the active federal tax-law record for the relevant year. It must not be hardcoded in business logic, tests, or report templates. Inclusion-rate rules have been politically and legislatively volatile; the engine must remain table-driven so that a tax-law update changes data, not parser or ledger code.

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
    note:                     string;    // always include disclaimer
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

**Marginal mode** — user provides other annual income; the engine calculates the incremental tax caused by Shakepay activity. This is the more accurate option and should be the default.

### 11.3 Table-driven requirement

The tax estimator must be **table-driven** for both federal and provincial or territorial tax. Do not embed rate tables, thresholds, surtaxes, reductions, credits, or inclusion-rate assumptions directly in estimator code. The estimator loads the active law version for:
- the selected tax year,
- the selected province or territory for that year, and
- the current inclusion-rate regime for that year.

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
db/seeds/taxBrackets/
  FED-2025.json
  QC-2025.json
  ON-2025.json
  BC-2025.json
  AB-2025.json
  ...
```

`taxLawRegistry.ts` loads and caches active law records by `jurisdiction-year` key (e.g., `"FED-2025"`, `"QC-2025"`). `provincialEngine.ts` dispatches via a registry rather than inline branching.

The selected law record schema must be able to represent: tax brackets, surtaxes, reductions, non-refundable credit rates, province-specific adjustments, capital-gains inclusion rates, and metadata about source URLs and effective dates.

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

### 11.7 Tax Law Admin layer

An explicit **Tax Law Admin** layer is required so tax data can be maintained without redeploying parser or ledger logic. Responsibilities include:
- loading official yearly tables and validating schema before activation,
- marking one law version active per jurisdiction-year and supporting supersession,
- patching edge cases through override profiles,
- exposing the exact law versions used in each report,
- providing a comparison view between two law versions.

---

## 12. Québec Module

### 12.1 Legal requirement

Since the 2024 tax year, Revenu Québec requires form **TP-21.4.39** — *Déclaration relative aux cryptoactifs* — when a Québec taxpayer possesses, receives, disposes of, or uses crypto-assets in a transaction. This obligation applies even in years with no taxable gain.

### 12.2 Product behavior

When `province = "QC"` and `taxYear >= 2024`:
- If any crypto possession, receipt, disposition, or use activity exists → set `quebecDisclosureRequired = true`
- Display a TP-21.4.39 reminder in the UI
- Generate the Québec disclosure support report

CAD-only rows must **not** be treated as crypto events for TP-21.4.39 purposes, though they remain important for reconciliation.

### 12.3 Québec rate-table handling

Québec tax calculations must also be table-driven and versioned. Do not hardcode Québec brackets. The module resolves the active `QC` tax-law version for the selected year and persists the exact law-version identifier in the annual summary and disclosure support output.

### 12.4 TP-21.4.39 support report

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

### 12.5 Form-mapping policy

Do **not** hardcode TP-21.4.39 line mappings in core business logic. The form can change over time. Maintain a versioned form-mapping configuration that links disclosure support report fields to the current version of the form. If a future implementation adds direct form-field export, that mapping must be versioned separately from the parser, classifier, and ledger rules.

---

## 13. Review Queue

### 13.1 Trigger conditions

| Trigger | Severity |
|---|---|
| Negative asset pool quantity | `blocking` |
| Contradictory duplicate rows | `blocking` |
| Unknown reward description | `warning` |
| Unresolved `btc_send` / `eth_send` | `warning` |
| Unresolved `btc_receive` / `eth_receive` | `warning` |
| `sif_received` event | `warning` |
| Unmatched buy legs | `warning` |
| Incomplete prior-year basis | `warning` |
| USD review codes present | `info` |
| Missing CAD valuation | `warning` |

### 13.2 `ReviewItem` schema

```typescript
interface ReviewItem {
  id:               string;
  severity:         "info" | "warning" | "blocking";
  reasonCode:       string;
  message:          string;
  relatedEventIds:  string[];
  suggestedActions: string[];
}
```

### 13.3 Blocking vs non-blocking

**Blocking** — prevents `CALCULATED` status until resolved:
- Negative asset pool
- Impossible numeric parse
- Contradictory duplicate rows

**Non-blocking** — allows calculation to proceed with warnings:
- Unmatched internal transfer
- Unknown reward description
- USD review items

### 13.4 Safety rule

**No row may ever be silently dropped.** Any row that cannot be confidently classified or reconciled must produce a `ReviewItem` with a reason code and related row IDs.

### 13.5 Override logging

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

---

## 14. Output Contract

### 14.1 Completeness scoring

| Score | Conditions |
|---|---|
| `complete` | Crypto + cash files present; full-year coverage; prior-year basis available; no unresolved reviews |
| `likely_complete` | Crypto + cash present; minor gaps or ≤5 review items |
| `incomplete` | Crypto present but cash missing; significant coverage gap; >5 unresolved reviews |
| `materially_incomplete` | Crypto file missing; no files for a year with known activity; basis chain broken |

### 14.2 Annual report structure

Each annual report must contain:

**Summary** — total taxable reward income, interest income, capital gains/losses, net capital gain/loss, taxable capital gain, estimated federal tax, estimated provincial tax, combined estimated tax.

**Breakdown by category** — ShakingSats, referral bonuses, ShakeSquad rewards, CAD interest, card cashback (excluded from income), SIF received (flagged), SIF sent (excluded), acquisitions, dispositions.

**Audit trail** — for every event:

| Field | Description |
|---|---|
| `sourceFile` | File family and filename |
| `rawRowIndex` | Row index in source file |
| `timestamp` | Transaction timestamp |
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
- Cashback treatment is a prudent default, not a confirmed specific CRA/Revenu Québec published position
- Review flags may affect final accuracy
- Tax estimates are based on Shakepay income only — total tax depends on full annual income
- Policy version and classification version used

**File coverage summary** — crypto file: yes/no; cash file: yes/no; USD file: yes/no; year coverage percentage; languages uploaded; review item count; completeness status.

---

## 15. API Reference

All endpoints are authenticated. Base path: `/v1`.

### 15.1 Import session

| Method | Path | Description |
|---|---|---|
| `POST` | `/v1/imports` | Create a new import session → `{ importId, status, createdAt }` |
| `GET` | `/v1/imports/:id` | Get session status and coverage |

### 15.2 File upload

| Method | Path | Description |
|---|---|---|
| `POST` | `/v1/imports/:id/files` | Upload one file (`multipart/form-data`) → `{ fileId, detectedFamily, detectedLanguage, rowCount, coverageStart, coverageEnd }` |
| `GET` | `/v1/imports/:id/files` | List files with coverage metadata |
| `DELETE` | `/v1/imports/:id/files/:fileId` | Remove a file and its canonical rows |

### 15.3 Processing

| Method | Path | Description |
|---|---|---|
| `POST` | `/v1/imports/:id/parse` | Detect family/language; produce canonical rows |
| `POST` | `/v1/imports/:id/classify` | Run classification on canonical rows |
| `POST` | `/v1/imports/:id/reconcile` | Link cross-file economic events |
| `POST` | `/v1/imports/:id/calculate` | Compute annual ledgers and tax outputs |

**Calculate request:**
```json
{
  "years": [2023, 2024, 2025],
  "provinceByYear": {
    "2023": "QC",
    "2024": "ON",
    "2025": "BC"
  },
  "otherTaxableIncomeByYear": {
    "2023": "62000.00",
    "2024": "70000.00",
    "2025": "81000.00"
  },
  "mode": "marginal"
}
```

### 15.4 Review queue

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/imports/:id/review-items` | List pending review items (`?year=2025` filter) |
| `PATCH` | `/v1/imports/:id/review-items/:itemId` | Submit user decision → `{ decision, reason }` |

### 15.5 Reports

| Method | Path | Description |
|---|---|---|
| `GET` | `/v1/imports/:id/reports/:year` | Annual summary JSON |
| `GET` | `/v1/imports/:id/reports/:year/audit` | Full audit trail |
| `GET` | `/v1/imports/:id/reports/:year/quebec-disclosure` | TP-21.4.39 support report (QC only) |
| `GET` | `/v1/imports/:id/reports/:year/export` | Download PDF or CSV |
| `GET` | `/v1/imports/:id/coverage` | Per-year completeness scores |

### 15.6 Error response format

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
| Unexpected server error | 500 |

---

## 16. Database Schema

```sql
-- Import sessions
CREATE TABLE imports (
  id                    TEXT PRIMARY KEY,
  user_id               TEXT NOT NULL,
  status                TEXT NOT NULL,
  completeness_status   TEXT,
  created_at            TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  policy_version        TEXT NOT NULL,
  classifier_version    TEXT NOT NULL
);

-- Uploaded files
CREATE TABLE import_files (
  id              TEXT PRIMARY KEY,
  import_id       TEXT NOT NULL REFERENCES imports(id),
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
  id          TEXT PRIMARY KEY,  -- sha256(import_id + file_id + row_index + raw_json)
  file_id     TEXT NOT NULL REFERENCES import_files(id),
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
  import_id        TEXT NOT NULL REFERENCES imports(id),
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
  import_id         TEXT NOT NULL REFERENCES imports(id),
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
  import_id                     TEXT NOT NULL REFERENCES imports(id),
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
  PRIMARY KEY (import_id, tax_year)
);

-- Review queue
CREATE TABLE review_items (
  id            TEXT PRIMARY KEY,
  import_id     TEXT NOT NULL REFERENCES imports(id),
  tax_year      INTEGER,
  event_id      TEXT REFERENCES economic_events(id),
  severity      TEXT NOT NULL CHECK (severity IN ('info','warning','blocking')),
  reason_code   TEXT NOT NULL,
  message       TEXT NOT NULL,
  status        TEXT NOT NULL DEFAULT 'pending'
                  CHECK (status IN ('pending','resolved','skipped')),
  user_decision TEXT,
  resolved_at   TIMESTAMPTZ,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ri_import_status ON review_items(import_id, status);

-- Override audit log
CREATE TABLE overrides (
  id           TEXT PRIMARY KEY,
  import_id    TEXT NOT NULL REFERENCES imports(id),
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

---

## 17. Tax Law Data Layer

### 17.1 Seed file format

Each seed file covers one jurisdiction and one tax year.

**Example shape: `db/seeds/taxBrackets/FED-2025.json`**
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

> Use placeholders in documentation examples. Active numeric values must come from maintained official seeds, not from prose copied into the spec.

### 17.2 `taxLawRegistry` behavior

- Loads seed files at startup or on admin refresh.
- Caches by `jurisdiction-year` key: `"FED-2025"`, `"QC-2025"`, `"ON-2025"`, etc.
- Returns typed `TaxLawVersion`.
- Throws `TaxLawNotFoundError` if no active table exists for the combination.
- Supports supersession, activation, and rollback of law versions.
- Applies override profiles after base-law load and before estimator execution.

### 17.3 Province and territory extensibility

Adding or updating a jurisdiction must require only:
1. Adding or updating the JSON seed file.
2. Loading or activating the new law version.
3. No parser, classifier, or ledger code changes.

---

## 18. Error Handling

### 18.1 Hard errors — fail and surface

| Error | Condition |
|---|---|
| `SchemaDetectionError` | File headers match no known schema |
| `ValidationError` | Required columns missing; invalid date; invalid numeric |
| `NegativePoolError` | Asset pool quantity becomes negative |
| `DuplicateRowError` | Same row ID with conflicting payload |
| `TaxLawNotFoundError` | No bracket table for jurisdiction + year |

### 18.2 Soft warnings — proceed with review flag

| Warning | Condition |
|---|---|
| Partial-year coverage | Selected year not fully covered |
| Unmatched economic event | Buy legs that cannot be reconciled |
| Missing prior-year basis | Historical data unavailable |
| Unresolved send/receive | Transfer ownership unclear |
| USD rows present | Policy treatment pending |
| Unknown reward description | Cannot map to known normalized code |

### 18.3 Safety invariant

No row is ever silently dropped. If a row cannot be processed at any stage, it must produce a `ReviewItem` and remain in the database in its last-valid stage.

---

## 19. Testing Strategy

### 19.1 Unit tests

| Module | Tests required |
|---|---|
| `detector.ts` | All 6 file families; French/English; unknown schema rejects |
| `canonicalizer.ts` | All bilingual field mappings; null for empty cells; decimal-safe parse; French locale numbers |
| `classifyReward.ts` | Each reward code in EN and FR; priority order; unknown → `unknown_reward_review` |
| `classifyTransaction.ts` | All standard codes; buy/sell/send/receive/convert |
| `assetPool.ts` | Acquisition adds qty + ACB; disposition reduces qty + ACB; negative qty blocks |
| `dispositionEngine.ts` | Gain/loss math; ACB allocation; prior-year basis |
| `eventLinker.ts` | Exact match; near-match within tolerance; ambiguous → review; unmatched → flag |
| `inclusionRate.ts` | Loads the active inclusion-rate record for the selected law version; no hardcoded year assumptions |
| `completenessScorer.ts` | All four score levels; each contributing factor |

### 19.2 Integration tests

| Fixture | What it validates |
|---|---|
| English-only crypto + cash | Full pipeline; reconciliation; BTC ACB |
| French-only crypto + cash | Bilingual parsing produces identical output to English fixture |
| Mixed EN/FR dossier | Bilingual parsing coexistence |
| Multi-year 2022–2025 | Prior-year basis carries forward correctly |
| Crypto-only (no cash) | Completeness warning; cash rewards not counted |
| All 3 families | USD preserved in review ledger; not in crypto ACB |
| Partial year | Coverage warning; `incomplete` completeness status |
| QC resident 2024+ | `quebecDisclosureRequired = true`; TP-21.4.39 report generated |

### 19.3 Regression fixtures

| Fixture | Expected outcome |
|---|---|
| ShakingSats (crypto EN) | `shakingsats` → income + BTC basis |
| Referral reward (cash EN) | `referral_bonus` → income |
| Referral reward (cash FR) | Same as above — bilingual parity |
| BTC cashback (crypto EN) | `card_cashback_btc` → basis only, no income |
| BTC sale | `btc_to_cad` → `taxable_disposition` + gain/loss |
| BTC↔ETH swap | BTC disposition + ETH acquisition |
| BTC send confirmed internal | No tax event; basis preserved |
| BTC send unconfirmed | `transfer_or_disposition_review` |
| SIF received | `sif_received` → `non_taxable_review` flagged |
| Cash buy + crypto buy | Reconciled to one `cad_to_crypto_buy` event — no double count |

### 19.4 QA matrix summary

| Layer | Test type | Minimum coverage |
|---|---|---|
| File detection | Unit | All 6 known schemas + unknown |
| Header normalization | Unit | All bilingual field pairs |
| Classification | Unit | All normalized codes EN + FR |
| Reconciliation | Unit + Integration | Exact, near, ambiguous, unmatched |
| ACB engine | Unit | BTC + ETH, acquisition + disposition + swap + multi-year |
| Tax engine | Unit | All federal bracket boundaries; representative province coverage plus smoke tests for all supported jurisdictions |
| Review queue | Unit | All trigger conditions; blocking vs non-blocking |
| API | Contract | All endpoints; error codes |
| E2E | Integration | ≥1 mixed-language multi-year scenario |

---

## 20. Policy Positions & Caveats

### 20.1 High-confidence positions

| Position | Implementation stance |
|---|---|
| Product is a Shakepay-only computation layer | Always disclose limited source scope |
| Crypto dispositions can create capital gains or losses | Treat sales, swaps, and non-internal sends as disposition-capable events |
| Prior-year basis affects current-year gains | Build opening ACB from all available prior history |
| Québec crypto activity can trigger TP-21.4.39 requirements | Show reminder and produce support summary for Québec years with crypto activity |
| Federal and provincial tax depend on the user's year-specific total income and residence | Require province per year; support marginal mode |

### 20.2 Prudent default positions

These are implementation defaults, **not** Shakepay-specific published tax rulings. They must be disclosed in every report.

| Position | Default | Trigger for change |
|---|---|---|
| Card cashback in BTC = purchase rebate, not income | `purchase_rebate` | Clear CRA or Revenu Québec guidance specific to this fact pattern |
| 3% card intro bonus = purchase rebate | `purchase_rebate` | Same as above |
| Shake It Forward received = not auto-included | `non_taxable_review` | Clear authority on peer-transfer gift or windfall treatment |
| USD events are preserved but excluded from crypto ACB and gain logic | `usd_review` | Formal USD treatment module and policy approval |

### 20.3 Required UI disclosures

The following notices must appear in the UI and in every generated report:

1. "This report covers Shakepay activity only. Other exchanges, wallets, or crypto income sources are not included."
2. "Cashback in BTC is treated by default as a purchase rebate, not ordinary income. This is a prudent default, not a confirmed CRA or Revenu Québec position specific to Shakepay cashback."
3. "Shake It Forward received is flagged for manual review and is not automatically included in income."
4. "Prior-year transactions may affect current-year capital gains through adjusted cost base."
5. "Québec residents may need to file form TP-21.4.39 — Déclaration relative aux cryptoactifs."
6. "Tax estimates are based on the tax-law versions selected for the relevant year and jurisdiction. Total tax payable still depends on the taxpayer's full annual income."
7. "USD activity is preserved but not included in crypto capital-gain calculations pending a formal policy treatment."

### 20.4 Policy and law version tracking

Every `NormalizedEvent`, `AnnualSummary`, and `QuebecDisclosure` must store:
- `policyVersion` — date string of the governing rules document (e.g., `"2026-03-28"`)
- `classifierVersion` — version of the classification rule registry
- `federalTaxLawVersionId`
- `provincialTaxLawVersionId`

This enables identification of which outputs were produced under prudent assumptions and which official law tables were active at calculation time.

---

## 21. Known Issues & Review Notes

The following flags were identified during review of real Shakepay export data. They represent weak spots in the current pipeline that warrant attention during implementation and QA.

### 21.1 `unknown_review` rows

`unknown_review` rows — sometimes appearing as duplicates within the same second, occasionally with no clearly resolved asset or quantity — are a signal that the classifier could not cleanly map the row. These must never be treated as resolved by virtue of reaching the review queue. Every `unknown_review` row must surface a `ReviewItem` with its reason code and must appear in the audit trail.

### 21.2 USD activity

USD rows classified as `no_tax_effect` may be correct for wallet-plumbing events, but Canadian tax treatment of FX gains and losses on USD↔CAD conversions is not trivially zero. Until a formal USD treatment module is built, the engine must:
- Preserve all USD rows in the review ledger with full monetary data.
- Display a visible UI notice that USD activity is not included in capital gain calculations.
- Never silently assign `no_tax_effect` without logging the reason.

### 21.3 Inconsistent second-leg treatment on `cad_to_btc`

In observed exports, the CAD side of a `cad_to_btc` buy sometimes receives `no_tax_effect` and sometimes `cash_funding_only`. The correct treatment depends on the subflow (e.g., card round-up vs. manual buy). The rule registry must be explicit about which sub-type maps to which treatment, and the reconciler must verify that the selected treatment is consistent across both legs of the linked economic event. Inconsistent treatments between linked legs are a source of double-counting or mis-bucketing.

### 21.4 BTC→ETH swap with unresolved ETH leg

In some exports, a `btc_to_eth` event records `taxable_disposition` on the BTC leg while the ETH acquisition leg is classified as `unknown_review`. This creates an asymmetric swap where the disposition is taxed but the incoming basis is not properly established. The reconciler should detect and flag this condition; the ACB engine should not silently proceed with an incomplete swap pair.

### 21.5 Quantity mismatches from fees or partial fills

Small differences between a buy quantity and a subsequent send quantity (due to exchange fees or partial fills) are normal when the engine uses pooled ACB. These should not generate false positives in the review queue. The reconciler's CAD tolerance of ≤ $0.05 addresses most cases; quantity-based matching should be secondary and should not block reconciliation when the CAD values align within tolerance.

### 21.6 Tax-year filter alignment

The year filter used in the UI and API must align precisely with the calendar-year boundary used in the ledger engine (`Jan 1 00:00:00` local time). If the filter and the ledger engine use different timezone assumptions, transactions at year boundaries may be attributed to the wrong year. This must be verified in integration tests that include rows timestamped near December 31 / January 1.

---

## 22. Open Decisions

| Decision | Options | Recommendation |
|---|---|---|
| **Tax-law source** | Manual JSON seeds vs. third-party rate API | Manual seeds first; add sync later |
| **Tax-law refresh cadence** | On demand vs. scheduled review | Quarterly review plus ad hoc refresh on published changes |
| **Internal transfer detection** | Auto-detect by matching sends/receives vs. user-confirmed only | User-confirmed for MVP; add heuristics later |
| **Missing basis policy** | Block calculation or mark as estimated and proceed | Mark estimated and proceed; escalate with review severity |
| **Default tax mode** | Standalone or marginal | Marginal is more accurate; standalone as fallback |
| **USD formal policy** | Define now or defer | Defer; ingest and review-flag; add dedicated module later |
| **Timestamp tolerance for reconciliation** | 60 s / 120 s / configurable | 120 s default, configurable per dossier |
| **Row ID scheme** | Random UUID vs. deterministic SHA-256 | Deterministic: `sha256(import_id + file_id + row_index + normalized_payload)` |
| **Province change across years** | Single province only vs. per-year selection | Require per-year selection |
| **Tax-law overrides** | Direct table edits vs. override profiles | Keep immutable base tables; use explicit override profiles |

---

## 23. Implementation Order

Build in this sequence to avoid blocking dependencies:

1. Schema detection + canonicalization
2. Classification engine (reward patterns + standard transactions)
3. Event linking / reconciler
4. BTC/ETH basis ledger engine
5. Annual aggregation + completeness scorer
6. Review queue + override logging
7. Tax law registry + bracket loader
8. Tax estimation adapter (federal + all provinces)
9. Québec disclosure module
10. API routes + middleware
11. Report export (JSON, PDF, CSV)

---

## 24. Definition of Done

The feature is complete only when **all** of the following are true.

**✅ All six file types detected correctly**
- `cash_summary/en`, `cash_summary/fr`
- `crypto_summary/en`, `crypto_summary/fr`
- `usd_summary/en`, `usd_summary/fr`
- Unknown schemas produce `SchemaDetectionError`

**✅ Bilingual parsing works end-to-end**
- English and French files produce identical canonical schemas
- All bilingual type values and description patterns recognized
- Decimal-safe numeric parsing; French locale handled
- Empty cells → `null`

**✅ Cash/crypto buys reconciled without double counting**
- Matching cash debit + crypto credit rows merged into one `EconomicEvent`
- ACB does not count the same acquisition twice
- Ambiguous matches routed to review

**✅ BTC/ETH ACB rollforward correct across years**
- Opening ACB computed from all prior history
- Prior-year acquisitions affect current-year dispositions
- Incomplete history marks result as `materially_incomplete`

**✅ Policy-coded rewards treated per documented rules**
- `shakingsats` → income + basis
- `referral_bonus` / `referral_promo_bonus` → income
- `shakesquad_reward` → income + basis
- `cad_interest` → interest income
- `card_cashback_btc` / `card_intro_bonus_3pct` → basis only
- `sif_received` → review flag, not auto-income
- `sif_sent` → non-deductible outflow

**✅ Annual summaries include federal and provincial estimates**
- All taxable amount fields present
- Federal + provincial tax estimate for every selected year and jurisdiction
- Policy version, classifier version, and active law-version identifiers stored in every summary

**✅ Incomplete dossiers clearly marked**
- Missing files, partial coverage, missing basis → visible completeness status
- UI shows reason for incompleteness

**✅ All unclassified rows appear in review output**
- Zero rows silently dropped
- Every unresolvable row → `ReviewItem` with reason code

**Release gate** — do not ship unless:
- All acceptance criteria above are covered by automated tests
- Fixture-based regression tests pass for all six file types
- At least one mixed-language multi-year E2E test is green
- Policy version and classifier version included in all report outputs
- Known limitations documented in the caveats section of every report

---

## 25. Recommended Improvements

| Priority | Improvement | Why it matters |
|---|---|---|
| High | Dossier-based import model everywhere | Removes the stale single-CSV assumption and aligns every layer with the actual upload bundle |
| High | Mandatory cross-file event linking | Prevents double counting and basis distortion |
| High | Completeness scoring per year | Lets users and QA judge whether the output is reliable enough to use |
| High | Policy and classifier version tracking | Separates fixed rules from prudent assumptions and supports reproducibility |
| Medium | Dedicated review inbox and reconciliation views | Makes ambiguous transfers and unmatched legs operationally manageable |
| Medium | Québec disclosure companion report | Turns the Québec requirement into a usable output instead of a generic reminder |
| Medium | Formal USD policy module | Moves USD from review-only handling into explicit tax logic once policy is decided |

---

## Official References

The following time-sensitive points were verified against official public pages on 2026-03-28:

- Revenu Québec publishes `TP-21.4.39-V` as the cryptoasset return and states that, since 2024, taxpayers who own, receive, dispose of, or use cryptoassets must complete it when applicable.
- CRA guidance states that, when crypto is disposed of on capital account, the applicable fraction of the capital gain is included in income as a taxable capital gain.
- CRA and Revenu Québec both publish current annual personal tax rate tables, which supports a maintained tax-table source rather than hardcoded parser logic.

Tax tables, brackets, and inclusion rates must always be sourced from the maintained seed layer and official publications — not from values embedded in this document.

---

*End of document — Shakepay Canada Tax Engine v2.1 — March 28, 2026*
