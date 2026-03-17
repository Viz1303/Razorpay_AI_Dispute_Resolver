# Feature Spec: AI Dispute Resolver
**Product:** Razorpay Merchant Dashboard
**Feature:** AI-Powered Payment Dispute Resolution Agent
**Author:** Vishal Arasu
**Version:** 1.0
**Status:** Draft — for engineering and design review
**Last Updated:** March 2026

---

## Table of Contents
1. [Executive Summary](#1-executive-summary)
2. [Problem Statement](#2-problem-statement)
3. [User Research & Merchant Pain Points](#3-user-research--merchant-pain-points)
4. [Proposed Solution](#4-proposed-solution)
5. [Data Inputs](#5-data-inputs)
6. [Confidence Thresholds & Escalation Logic](#6-confidence-thresholds--escalation-logic)
7. [Failure Modes & Trust Guardrails](#7-failure-modes--trust-guardrails)
8. [Evaluation Metrics](#8-evaluation-metrics)
9. [Rollout Plan](#9-rollout-plan)
10. [Open Questions](#10-open-questions)
11. [Appendix: Out of Scope](#11-appendix-out-of-scope)

---

## 1. Executive Summary

Razorpay processes over 10 million transactions daily across 8 million+ merchants. A small but operationally expensive fraction of these result in payment disputes — initiated by cardholders via their issuing banks. Today, dispute resolution is manual: a Razorpay support agent reviews evidence, communicates with merchants, and submits rebuttals to card networks. This process averages 7–12 business days, consumes significant agent bandwidth, and consistently ranks among the top three sources of merchant dissatisfaction.

This spec defines the **AI Dispute Resolver**: an AI agent embedded in the merchant dashboard that automatically triages incoming disputes, gathers and evaluates evidence, and either resolves the dispute autonomously or prepares a structured handoff to a human agent — with full audit trail. The goal is to resolve 60–70% of disputes end-to-end without human intervention within 48 hours, while maintaining or improving resolution accuracy.

---

## 2. Problem Statement

### 2.1 What is a payment dispute?

A payment dispute (also called a chargeback) occurs when a cardholder contacts their bank to reverse a completed transaction. The issuing bank then opens a dispute with the card network (Visa, Mastercard, RuPay), which Razorpay — as the payment aggregator — must respond to within a strict deadline (typically 7–21 days depending on the network and reason code). If Razorpay fails to respond in time, the merchant automatically loses the chargeback and the amount is debited from their settlement.

### 2.2 Current process flow

```
Dispute received from card network
        ↓
Razorpay support queue (manual triage)
        ↓
Agent reviews transaction + merchant history
        ↓
Agent contacts merchant via email for evidence
        ↓
Merchant uploads proof (delivery receipt, screenshot, etc.)
        ↓
Agent compiles rebuttal document
        ↓
Rebuttal submitted to card network
        ↓
Outcome: Won / Lost / Expired
```

**Average end-to-end time:** 7–12 business days
**Human touchpoints per dispute:** 3–5 interactions
**Deadline miss rate (estimated):** 8–12% of disputes (source: industry benchmarks; Razorpay-specific figure is an assumption — see Business Case)

### 2.3 Why this is worth solving now

- Dispute volumes in India are growing with UPI and card adoption (RBI Annual Report FY24: card transactions grew 24% YoY)
- Razorpay's merchant base has grown 4× since 2020; support headcount has not scaled proportionally
- Card network mandates (Visa Compelling Evidence 3.0, effective April 2023) now require structured, evidence-based responses — a format AI is well-suited to generate
- Competitors (Stripe, PayPal) already offer automated dispute management; this is a growing table-stakes expectation

---

## 3. User Research & Merchant Pain Points

**Primary users:** Small and medium merchants (SMBs) on Razorpay who lack dedicated finance/ops teams to manage disputes.

**Secondary users:** Large enterprise merchants with high dispute volumes who want faster resolution and reporting.

**Support users:** Razorpay dispute resolution agents who will receive AI-escalated cases.

### Pain Point 1: Merchants don't know a dispute exists until it's too late

Dispute notifications today arrive via email and a dashboard banner. Email open rates for transactional fintech emails average 18–22% (Mailchimp 2024 benchmarks). Merchants running ops on mobile often miss these entirely. By the time they see the notification, the evidence-upload window may have closed.

**User research gap:** This pain point is supported by proxy evidence (low transactional email open rates, Razorpay community forum complaint threads, Trustpilot reviews citing "no notification before settlement deduction") but has not been validated through structured merchant interviews. A targeted user research sprint — 8–10 interviews with SMB merchants who have experienced disputes in the past 6 months — is recommended before finalising the notification UX. The research should confirm: which channel merchants actually monitor for operational alerts, at what point in a dispute they first want to be notified, and whether SMS/WhatsApp outperforms push/email for time-sensitive alerts in this context.

### Pain Point 2: Evidence collection is friction-heavy and unclear

Merchants are asked to upload "proof of delivery" or "order confirmation" with no guidance on what format is acceptable or what makes evidence compelling. First-submission rejection rates in the industry run at 30–40% (Chargebacks911, 2023 State of Chargebacks report) because evidence is incomplete.

### Pain Point 3: Outcome opacity — merchants don't understand why they lost

When Razorpay loses a dispute, merchants receive a generic outcome notification. There is no explanation of what evidence was missing or what they could do differently next time. This creates a sense that the system is arbitrary and erodes trust.

### Pain Point 4: Long resolution timelines block working capital

For SMB merchants, a disputed ₹10,000–₹50,000 transaction is real working capital. A 10-day hold is proportionally much more painful than for a large merchant. Faster resolution directly impacts cash flow.

### Pain Point 5: High cognitive load for infrequent disputes

Most SMBs encounter disputes rarely — perhaps once a quarter. Each occurrence feels novel and stressful. There is no institutional memory or workflow to fall back on, unlike a large merchant with a dedicated chargebacks team.

---

## 4. Proposed Solution

### 4.1 Feature Overview

The AI Dispute Resolver is a three-stage agent embedded in the merchant dashboard. It operates as a background service that activates on dispute receipt and presents findings to both the merchant and (where needed) a human Razorpay agent.

### Stage 1: Intelligent Triage (fully automated, seconds)

On dispute receipt from the card network, the agent immediately:
- Parses the dispute reason code (there are ~30 standard reason codes across Visa/Mastercard/RuPay) and maps it to a resolution strategy
- Pulls all structured data about the transaction: amount, merchant category, device fingerprint, IP, shipping address, order history for that customer
- Cross-references the disputing customer against Razorpay's fraud signal database (is this card associated with prior fraudulent disputes?)
- Classifies the dispute into one of three categories:
  - **Likely Merchant Win** — strong evidence already exists in structured data
  - **Needs Evidence** — outcome is uncertain, merchant input required
  - **Likely Loss / Low Priority** — dispute has high probability of being legitimate (e.g., genuinely duplicate charge); negotiated settlement may be optimal

### Stage 2: Automated Investigation (fully automated, minutes to hours)

For disputes classified as **Likely Merchant Win** or **Needs Evidence**, the agent:
- Automatically pulls evidence from connected systems where available: delivery partner APIs (Shiprocket, Delhivery, Ecom Express), Razorpay order notes, email/SMS delivery confirmation webhooks
- Drafts a structured evidence package in the format required by the relevant card network
- For digital goods / SaaS merchants: checks login logs, IP-matched usage events, and session data if the merchant has connected their system via Razorpay webhooks
- For disputes that still need merchant input: sends a guided, conversational prompt inside the dashboard ("We found your delivery confirmation from Shiprocket. Can you also upload the customer's signed order or screenshot of their confirmation email?") — not a blank upload box

### Stage 3: Resolve or Escalate (semi-automated)

Based on the agent's confidence score (see Section 6):
- **High confidence (>85%):** Agent auto-submits the rebuttal to the card network without merchant or human agent review. Merchant receives a notification: "We've automatically responded to this dispute on your behalf. Here's what we submitted."
- **Medium confidence (50–85%):** Agent presents the evidence package to the merchant for one-click approval before submission. The merchant sees: evidence gathered, AI recommendation, and a plain-English explanation of the case. Estimated merchant time: < 2 minutes.
- **Low confidence (<50%) or edge case flag:** Full handoff to human Razorpay agent with an AI-generated case summary, evidence gathered so far, and recommended next steps.

### 4.2 Merchant-Facing UX Principles

- **Proactive, not reactive:** Merchants should never discover a dispute for the first time from a settlement deduction. Push notification + in-app alert + SMS.
- **Plain English:** Dispute reason codes should never be shown raw. "Reason: 4853 – Merchandise / Services Not as Described" becomes "The customer claims the product was not what they ordered."
- **One-click actions:** Any merchant action should be completable in under 3 minutes. No multi-step forms.
- **Full audit trail:** Every AI action, decision, and submission is logged and visible to the merchant. No black box.

### 4.3 Agent-Facing UX (for escalated cases)

When a dispute is escalated to a human agent, they receive:
- AI case summary (2–3 sentences)
- All evidence gathered, annotated with source and confidence
- Recommended action with reasoning
- Quick-accept or override control

This is not an AI assistant that agents query — it is a structured handoff that replaces the blank-slate case queue agents currently work from.

---

## 5. Data Inputs

The model needs access to the following data inputs. Each input has a designated owner and sensitivity classification.

| Input | Source | Sensitivity | Notes |
|---|---|---|---|
| Dispute reason code | Card network / Razorpay webhook | Low | Structured; one of ~30 standard codes |
| Transaction metadata | Razorpay DB | Medium | Amount, timestamp, MCC, payment method |
| Customer device fingerprint | Razorpay fraud DB | High | PII-adjacent; masked for model input |
| Order history (customer × merchant) | Razorpay DB | Medium | Prior disputes, refund history |
| Delivery confirmation | Logistics partner APIs | Low–Medium | Available for ~60% of physical goods merchants |
| Merchant category + dispute win rate | Razorpay internal analytics | Low | Historical win rate by MCC × reason code |
| Card network dispute history for this card | Razorpay fraud consortium | High | Requires RBI data localisation compliance |
| Merchant-uploaded evidence | Merchant dashboard | High | PDFs, images — requires secure storage and retention policy |
| Razorpay fraud signal score | Internal fraud ML model | High | Composite score; model-to-model dependency |
| Session/login logs (digital goods) | Merchant-connected system via API | High | Optional; only if merchant has opted in |

**Data retention note:** Evidence documents are subject to card network retention requirements (minimum 18 months for Visa). All PII must be handled under RBI's data localisation framework (storage within India mandatory).

---

## 6. Confidence Thresholds & Escalation Logic

### 6.1 Confidence Score

The AI agent produces a single **Dispute Confidence Score** (0–100) representing the probability that the merchant will win if the current evidence package is submitted. This score is a weighted combination of four signals:

| Signal | Initial Weight | Rationale |
|---|---|---|
| Reason code + merchant category base win rate | 30% | Historical win rates by MCC × reason code are the single strongest predictor of outcome in chargeback datasets (cf. Chargebacks911, 2023); anchors the prior |
| Evidence completeness score | 35% | Card network rulings are primarily evidence-driven; missing required fields (e.g., no proof of delivery for "item not received") are nearly deterministic losses |
| Fraud signal match | 20% | Disputes from cardholders with prior fraud flags are won at ~2× the baseline rate in Razorpay's own portfolio; moderately strong but not decisive alone |
| Deadline proximity penalty | 15% | Urgency affects decision quality, not case merit; penalty is applied to reflect increased risk of error when time-compressed, not to change the win probability estimate |

**Important:** These are **initial heuristic weights**, not empirically derived. They reflect the relative importance of each signal based on industry research and analogous chargeback models (Stripe's dispute ML documentation, 2023). They must be treated as a starting point for calibration, not as fixed design.

The Phase 1 shadow mode (see Section 9) will generate a labeled dataset of dispute outcomes. After 3–4 months of data accumulation (~20,000+ labeled disputes), logistic regression will be used to fit empirical weights and replace these heuristics. Weights will be recalibrated quarterly thereafter. Any weight shift > 10 percentage points from the initial heuristic triggers a mandatory product review before deployment.

### 6.2 Decision Bands

| Score | Decision | Merchant action required | Human agent involved |
|---|---|---|---|
| 85–100 | **Auto-resolve:** Submit rebuttal automatically | None (notified post-submission) | No |
| 65–84 | **Merchant review:** One-click approve or edit | < 2 min | No |
| 50–64 | **Assisted resolution:** AI drafts, merchant completes evidence gaps | 5–10 min | No |
| 30–49 | **Soft escalation:** AI summary sent to human agent + merchant | None | Yes (AI-assisted) |
| 0–29 | **Full escalation:** Human agent takes over | None | Yes (standard queue) |

### 6.3 Hard Escalation Triggers (override confidence score)

Regardless of confidence score, the following conditions trigger immediate full escalation to a human agent:

1. Dispute amount > ₹1,00,000 (configurable threshold)
2. Dispute involves a merchant flagged for elevated regulatory risk
3. The disputing customer has filed > 5 disputes with Razorpay merchants in the past 90 days (potential friendly fraud ring)
4. Reason code is "Fraud – Card Not Present" and transaction amount > ₹25,000 (higher stakes + nuanced evidence requirements)
5. Merchant has explicitly opted out of AI auto-submission in their settings
6. Legal hold or active investigation flag on the merchant account

### 6.4 Deadline-Aware Processing

The agent checks time remaining against the card network deadline at each stage. If deadline < 72 hours and confidence score < 50, the agent immediately escalates with a URGENT flag rather than waiting for the standard queue. A missed deadline is always worse than a lower-quality human response.

---

## 7. Failure Modes & Trust Guardrails

### Failure Mode 1: Model submits incorrect evidence, merchant loses a winnable dispute

**Scenario:** The AI confidence score is high but based on incomplete evidence. For example, a delivery confirmation exists but the address on the record doesn't match the dispute. The model misses this mismatch and auto-submits. The card network rules in favor of the cardholder.

**Impact:** Merchant loses legitimate revenue; damages trust in AI automation.

**Guardrails:**
- Address mismatch between delivery record and billing address is a hard check that forces confidence score below 65 (no auto-submit)
- All auto-submitted rebuttals are logged with the evidence snapshot used; post-hoc analysis can identify systematic errors
- Merchants receive a "dispute report card" after outcome: what was submitted, what the network ruled, and whether AI vs. human handled it — this data feeds model improvement
- Merchants can toggle off auto-submission globally or per-dispute type in settings

### Failure Mode 2: Adversarial merchant inflates evidence

**Scenario:** A merchant uploads a forged delivery receipt or fabricated screenshot as "evidence." The AI treats it as legitimate, auto-submits, and the merchant fraudulently wins a legitimate customer dispute.

**Impact:** Razorpay becomes complicit in merchant fraud; regulatory and reputational risk; card network fines.

**Guardrails:**
- Document authenticity checks: metadata analysis on uploaded PDFs/images (creation date, modification history, geolocation EXIF data)
- Anomaly detection: if a merchant's dispute win rate suddenly spikes post-AI adoption, flag for manual review
- For disputes above ₹50,000, human spot-check of evidence even if AI confidence is high (5% random sample + all flagged anomalies)
- Razorpay's existing merchant risk scoring feeds a "merchant trustworthiness" multiplier into the confidence calculation
- Card networks have their own fraud detection; egregious cases will be caught at that layer too

### Failure Mode 3: Model confidently resolves a dispute that has a regulatory or legal dimension

**Scenario:** A dispute involves a transaction that is under investigation by the ED or related to a frozen account. The AI, unaware of the legal context, proceeds with a standard rebuttal.

**Impact:** Razorpay takes an official position in an active legal matter without legal review; compliance violation.

**Guardrails:**
- Legal hold flag in the merchant account system is a hard blocker — any dispute associated with a flagged merchant or transaction is immediately routed to the compliance team, never to AI
- The compliance team maintains a blocklist of transaction IDs under investigation; this is checked at triage (Stage 1) before any AI action
- Integration with Razorpay's AML/KYC system is required before go-live; this is a launch dependency

### Failure Mode 4: Notification failure — merchant misses AI action confirmation

**Scenario:** The AI auto-submits a rebuttal, sends a notification, but the merchant never sees it (email bounced, push notification disabled). The merchant later contacts support not knowing what was submitted on their behalf, creating confusion and eroding trust.

**Impact:** Merchant dissatisfaction; support tickets increase; trust in the feature degrades.

**Guardrails:**
- Multi-channel notification: in-app alert + push notification + email + SMS (for auto-resolved disputes). Delivery confirmation tracked.
- If notification delivery fails across all channels, the dispute is downgraded to "Merchant review" tier so the merchant must actively approve before submission
- Dashboard landing page shows a persistent "Recent AI Actions" widget for 72 hours after any AI-automated activity
- Audit trail is always accessible in dispute history: "This dispute was resolved automatically by AI on [date]. Click to see what was submitted."

### Failure Mode 5: Model degrades over time as dispute patterns shift

**Scenario:** Card networks change reason code definitions (as Visa did in 2023 with CE 3.0). Or a new category of fraud emerges that the model was not trained on. The model continues to apply old patterns and its win rate silently decays.

**Impact:** Resolution quality drops without anyone noticing until merchant complaints spike.

**Guardrails:**
- Monthly model performance review: track win rate by reason code × confidence band. Alert if any band's win rate drops > 5 percentage points month-over-month.
- Card network rule changes trigger an automatic model revalidation gate — no AI auto-submission for affected reason codes until revalidation passes
- Feedback loop: every dispute outcome (won/lost/expired) is fed back as a labeled training signal
- Hard sunset: if overall AI win rate drops below the historical human baseline for 2 consecutive months, auto-submission is paused and a model retraining sprint is triggered

---

## 8. Evaluation Metrics

### 8.1 Primary Success Metrics

| Metric | Definition | Target (6 months post-GA) | Baseline |
|---|---|---|---|
| **AI Automation Rate** | % of disputes handled end-to-end by AI (no human) | 60% | 0% |
| **AI Win Rate** | % of AI-handled disputes won | ≥ human baseline (est. 55–65%) | 55–65% (human) |
| **Resolution Time (median)** | Days from dispute receipt to rebuttal submission | < 2 days (AI) | 7–12 days (human) |
| **False Positive Rate** | % of AI auto-submits that the merchant disputes as wrong | < 2% | N/A |
| **Deadline Miss Rate** | % of disputes where rebuttal is not submitted before network deadline | < 2% | 8–12% (est.) |

### 8.2 Secondary Metrics

| Metric | Definition | Target |
|---|---|---|
| **Merchant CSAT (dispute flow)** | Post-resolution survey score (1–5) | ≥ 4.0 |
| **Escalation Rate** | % of disputes escalated to human agent | < 35% at 6 months |
| **Merchant opt-out Rate** | % of merchants disabling AI auto-submission | < 15% |
| **Evidence completeness on first submission** | % of submissions accepted by card network without additional evidence request | > 80% |
| **Support ticket volume (dispute-related)** | Tickets per 1,000 disputes | Reduction of 40% vs. baseline |

### 8.3 Guardrail Metrics (must not regress)

- Human agent dispute win rate must not decline (AI should not cream-skim easy cases and leave agents with harder ones without additional support)
- Merchant NPS overall must not decline more than 1 point attributable to dispute handling
- Regulatory findings related to dispute handling: zero

### 8.4 Measurement Plan

- Dispute outcomes are reported by card networks; integrate into Razorpay analytics pipeline
- CSAT survey triggered 48 hours after dispute closure, in-app
- A/B test during pilot: 50% of eligible disputes go to AI flow, 50% to human queue (randomised by merchant ID). Compare win rate, resolution time, and merchant CSAT.

---

## 9. Rollout Plan

### Phase 0: Foundation (Weeks 1–8)
**Goal:** Build data infrastructure and internal tooling before any merchant-facing feature.

- Integrate dispute reason codes + card network webhooks into a unified dispute data model
- Build evidence ingestion pipeline (logistics APIs, merchant webhooks)
- Implement audit logging layer (every AI action logged with inputs, outputs, confidence score, timestamp)
- Legal review: confirm auto-submission is permissible under Razorpay's card network agreements
- Compliance check: confirm data inputs comply with RBI data localisation and PCI-DSS

**Go/no-go criteria:** All data pipelines operational; legal and compliance sign-off.

### Phase 1: Internal Dogfood (Weeks 9–12)
**Goal:** Run AI agent in shadow mode — AI makes decisions but humans submit. Compare AI recommendation vs. human outcome.

- AI agent runs in parallel with human agents on 100% of disputes
- No merchant-facing UI changes; disputes still handled by humans
- Track: how often would AI have made the same decision as the human? Win rate if AI had submitted?
- Threshold: AI recommendation agreement rate with human outcome > 70% before proceeding

**Go/no-go criteria:** Shadow mode win rate ≥ human baseline; zero compliance flags.

### Phase 2: Merchant Pilot (Weeks 13–20)
**Goal:** Limited merchant-facing launch with AI in the "Merchant review" tier only (no auto-submission yet).

- Select 500 pilot merchants: mix of SMBs and mid-market; diverse MCCs; voluntary opt-in
- AI drafts evidence package; merchant reviews and approves before submission
- Merchant NPS and CSAT collected weekly
- Human agents remain on standby; escalations go to dedicated pilot support pod

**Go/no-go criteria:** Merchant CSAT ≥ 3.8; AI-assisted win rate ≥ human baseline; false positive rate < 5%; no regulatory issues.

### Phase 3: Staged GA with Auto-Submit (Weeks 21–32)
**Goal:** Enable AI auto-submission (high confidence tier) for opted-in merchants at increasing scale.

- Week 21: Enable auto-submit for top-tier merchants (>100 disputes/month, longest-tenured, highest trust score). 10% of auto-submit eligible volume.
- Week 25: Expand to 30% of eligible volume, all merchant tiers.
- Week 29: Full GA — all eligible merchants enrolled by default, with opt-out available.
- Hard escalation triggers and guardrail metrics monitored in real time.

### Phase 4: Optimisation (Weeks 33+)
- Model retraining with accumulated outcome data
- Add new data sources (UPI dispute data, NACH return reasons)
- Expand to international dispute handling (Stripe-competitive feature)

---

## 10. Open Questions

| # | Question | Owner | Priority | Status |
|---|---|---|---|---|
| 1 | Does auto-submitting a rebuttal on behalf of a merchant require explicit informed consent, or is it covered under Razorpay's existing merchant ToS? | Legal | P0 | Unresolved |
| 2 | Do card networks (Visa, Mastercard) permit AI-generated rebuttals, or must a human certify the submission? | Card network partnerships team | P0 | Unresolved |
| 3 | What is the minimum data retention period for AI decision audit logs? Does it differ from evidence retention? | Compliance | P1 | Unresolved |
| 4 | How do we handle disputes on transactions processed via third-party aggregators (PayU, CCAvenue) that are settled through Razorpay? | Engineering | P1 | Unresolved |
| 5 | Can we share dispute outcome data (anonymised) with external ML providers for model improvement, or does RBI data localisation prevent this? | Compliance + Data Privacy | P1 | Unresolved |
| 6 | What is the escalation SLA for human agents receiving AI-escalated cases? Should AI-escalated cases get priority queue treatment? | Operations | P2 | Needs stakeholder input |
| 7 | How do we handle disputes for merchants who use Razorpay in white-label mode (partners who have rebranded the product)? | Partnerships | P2 | Not started |
| 8 | Should merchants be able to see the confidence score the AI assigned? Or is this too confusing / gameable? | Design | P2 | Needs user testing |

---

## 11. Appendix: Out of Scope

The following are explicitly **not** in scope for v1:

- UPI dispute resolution (separate regulatory framework under NPCI; different process)
- Merchant-initiated refunds (not the same as bank-initiated chargebacks)
- International transactions outside India
- Disputes involving merchants under active legal investigation (always human)
- Building a proprietary ML model from scratch — v1 uses a rules-based scoring engine + a hosted LLM for evidence drafting; full ML model is a v2 investment
- Proactive fraud prevention (a separate product surface; this feature is reactive)
