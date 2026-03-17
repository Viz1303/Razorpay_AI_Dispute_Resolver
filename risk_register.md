# Risk Register: AI Dispute Resolver
**Product:** Razorpay Merchant Dashboard — AI Dispute Resolver
**Version:** 1.0
**Status:** Draft
**Last Updated:** March 2026

---

## Risk Rating Scale

| Rating | Likelihood | Description |
|---|---|---|
| H | High | Likely to occur without intervention; observed in similar deployments |
| M | Medium | Possible; requires specific circumstances |
| L | Low | Unlikely; would require multiple failures to materialise |

| Rating | Impact | Description |
|---|---|---|
| H | High | Significant financial loss, regulatory action, or material merchant trust damage |
| M | Medium | Measurable NPS decline, moderate financial exposure, reputational risk |
| L | Low | Operational inconvenience, recoverable with standard support response |

**Risk Priority Score** = Likelihood × Impact (HH = Critical, HM/MH = High, MM/HL/LH = Medium, ML/LM = Low, LL = Minimal)

---

## Risk 1: Model Error Causes Merchant to Lose a Winnable Dispute

**Category:** AI / Model Quality
**Priority:** Critical (Likelihood: H, Impact: H)

### Description
The AI agent assigns a high confidence score to a dispute and auto-submits a rebuttal that is factually incomplete or incorrect. The card network rules in the cardholder's favour. The merchant loses revenue they would have recovered if a human agent had handled the case. This could occur because:
- The model fails to detect an address mismatch between delivery and billing records
- A key piece of evidence (e.g., customer's signed acceptance for digital goods) is present but not parsed correctly from an uploaded PDF
- The reason code requires a specific legal declaration (e.g., "services rendered") that the AI generates in a non-compliant format

**Likelihood:** High — Evidence parsing from unstructured documents (PDFs, images) is inherently error-prone. Even at 95% accuracy, at scale this affects thousands of merchants.

**Impact:** High — Merchants lose real revenue. If systematic, destroys trust in the AI feature. Liability questions if Razorpay's ToS implies a duty of care in dispute representation.

### Mitigation
1. **Conservative thresholds at launch:** Auto-submission only above 85% confidence; threshold raised further (90%) during Phase 3 rollout until win rate data confirms calibration.
2. **Address and key-field hard checks:** Structured mismatch detection (address, amount, date) is rule-based, not ML-dependent — any mismatch forces confidence below 65%.
3. **Outcome feedback loop:** Every dispute result is tagged (AI-handled vs. human-handled) and win rates tracked by reason code and confidence band weekly. Divergence triggers model review.
4. **Merchant redress:** If a post-analysis review confirms AI error caused a merchant loss, Razorpay compensates the merchant for the lost dispute amount. This policy should be defined and committed to before launch.
5. **Opt-out remains available:** Merchants can disable auto-submission at any time, reducing exposure for merchants who prefer human handling.

**Owner:** ML Engineering (model quality), Product (thresholds), Legal (redress policy)
**Review Cadence:** Weekly during Phase 2 pilot; monthly post-GA

---

## Risk 2: Regulatory Non-Compliance (RBI, Card Networks, IT Act)

**Category:** Regulatory / Legal
**Priority:** Critical (Likelihood: M, Impact: H)

### Description
The AI Dispute Resolver touches several regulatory boundaries simultaneously:

- **RBI data localisation:** All payment data must be stored within India. If any model inference happens on infrastructure outside India (e.g., a US-hosted LLM API), transaction metadata sent to that model may violate RBI's 2018 circular on payment data storage.
- **Card network operating regulations:** Visa and Mastercard have specific rules on who can certify a chargeback rebuttal. If their operating regulations require a human to attest to the evidence submitted, AI-generated rebuttals may be rejected or constitute a violation of the merchant agreement.
- **IT Act / DPDP Act 2023:** The Digital Personal Data Protection Act creates obligations around processing personal data (which cardholder names, addresses, and transaction details constitute). Automated decisions affecting merchant finances may require explainability and a right-to-contest mechanism.
- **Consumer protection:** If a legitimate customer dispute is incorrectly rejected (because the AI "won" for the merchant), the customer's right to redress is affected. RBI has consumer protection mandates in the payments space.

**Likelihood:** Medium — These are known constraints; compliance is achievable but requires deliberate architecture decisions that are not yet made.

**Impact:** High — RBI enforcement actions can include fines, operational restrictions, and in severe cases, license suspension. Card network violations can result in fines and elevated interchange rates. Reputational damage is severe.

### Mitigation
1. **Compliance review as Phase 0 gate:** Legal and compliance sign-off is a hard prerequisite before any merchant-facing work begins (see Rollout Plan, Phase 0).
2. **India-only infrastructure for model inference:** If an external LLM is used for evidence drafting, all inference must occur on India-region cloud infrastructure. No payment data to be sent to US/EU-based endpoints. Explore alternatives: Azure India (OpenAI), AWS ap-south-1 (Bedrock), or self-hosted model.
3. **Card network pre-clearance:** Engage Visa/Mastercard/NPCI partnership contacts in Phase 0 to confirm AI-generated rebuttals are permissible and understand certification requirements.
4. **DPDP explainability:** Every AI decision is logged with the reason for the outcome in human-readable form. Merchants (and potentially cardholders via support) can request a decision explanation.
5. **Legal counsel engaged:** Retain external fintech-regulatory counsel familiar with RBI payments regulation for pre-launch review.

**Owner:** General Counsel, Head of Compliance, Card Network Partnerships
**Review Cadence:** Monthly; immediate escalation on any new regulatory communication from RBI or card networks

---

## Risk 3: Merchant Trust Erosion After High-Profile AI Error

**Category:** Trust / Reputation
**Priority:** High (Likelihood: M, Impact: H)

### Description
Even if the overall AI win rate is strong, a single high-visibility error — a large merchant losing a significant dispute due to AI mishandling — can rapidly erode trust across the merchant base. This is amplified by:
- Social media amplification (merchant Twitter/LinkedIn posts about AI making them lose ₹5 lakh)
- Press coverage framing Razorpay as "letting AI make financial decisions for merchants"
- Merchant community forums (many SMBs are in WhatsApp groups where fintech experiences spread quickly)

Trust damage is disproportionate to the actual error rate because of availability bias: one negative story outweighs many successful automated resolutions.

**Likelihood:** Medium — Errors will occur at scale; the question is whether any single error will be prominent enough to trigger a trust cascade.

**Impact:** High — Merchant opt-out rates could spike, reducing the feature's value. Competitor FUD ("Razorpay let AI lose our disputes") could damage new merchant acquisition. NPS impact could be significant.

### Mitigation
1. **High-value disputes never auto-submitted:** Disputes above ₹1,00,000 always go through human or merchant-review tier. This keeps the most visible failures out of the AI-only path.
2. **Proactive communication:** Before GA, publish a clear "How AI Dispute Resolver works" explainer in the dashboard and Help Centre. Set expectations: "AI handles straightforward cases; human experts handle complex ones."
3. **Merchant control narrative:** Frame the feature as a "co-pilot" not an "autopilot." Emphasise that merchants can always review what was submitted, opt out, or request human review.
4. **Rapid response protocol:** Pre-define a crisis response plan for a high-profile AI error. Includes: immediate manual review of all similar cases, proactive outreach to affected merchants, compensation policy, and a public statement template.
5. **Trust metrics monitoring:** Track weekly merchant sentiment on dispute handling (NPS sub-driver) and social listening for "Razorpay dispute" mentions.

**Owner:** Product Marketing, Merchant Success, PR/Comms
**Review Cadence:** Weekly NPS review during pilot; ongoing social listening post-GA

---

## Risk 4: Adversarial Merchants Gaming the AI System

**Category:** Fraud / Security
**Priority:** High (Likelihood: M, Impact: M)

### Description
Once merchants understand that the AI will auto-submit a rebuttal if evidence meets certain criteria, sophisticated or dishonest merchants may attempt to game the system:
- Uploading forged delivery receipts (fake PDFs with correct-looking metadata)
- Submitting screenshots of "order confirmations" that were generated/edited after the fact
- Deliberately structuring transactions to fall below the ₹1,00,000 human-review threshold
- Coordinating with associates to file fake disputes that the AI "defeats," generating a fraudulent chargeback win pattern

This transforms Razorpay's AI tool from a dispute management system into a merchant fraud enabler — which carries card network fines and potential regulatory action.

**Likelihood:** Medium — Document forgery is technically accessible. The financial incentive exists, especially for high-dispute-volume merchants. This has been observed in Western markets with automated dispute tools (Stripe's chargeback protection fraud cases, 2022–2023).

**Impact:** Medium — Individual fraud cases are manageable; systematic exploitation is a network risk. Card networks monitor aggregators for anomalous win rate patterns and will investigate Razorpay if merchant fraud spikes.

### Mitigation
1. **Document forensics on uploads:** Automated metadata analysis of all uploaded PDFs and images. Flags: creation date post-dates transaction, GPS metadata inconsistency, editing software fingerprint in EXIF, pixel-level manipulation indicators.
2. **Win rate anomaly detection:** If a merchant's dispute win rate increases > 20 percentage points post-AI adoption (vs. their own historical baseline), flag for manual fraud review.
3. **Third-party verification:** For disputes above ₹25,000, cross-check delivery confirmation with the logistics partner API directly — not just the merchant's uploaded document.
4. **Merchant risk scoring multiplier:** Merchants with elevated risk flags (new, high dispute volume, flagged MCC) have a lower effective confidence ceiling — AI cannot auto-submit above a reduced threshold until they build trust history.
5. **Card network reporting:** Unusual win rate patterns are reported proactively to card networks, demonstrating Razorpay's controls. This protects Razorpay from network-level penalties.

**Owner:** Fraud Risk, ML Engineering, Compliance
**Review Cadence:** Weekly automated monitoring; manual review triggered by anomaly flags

---

## Risk 5: Model Performance Degrades as Dispute Landscape Shifts

**Category:** Technical / Operational
**Priority:** High (Likelihood: H, Impact: M)

### Description
The dispute ecosystem is not static. Conditions that will cause the AI model's performance to degrade over time include:
- **Card network rule changes:** Visa updated its Compelling Evidence rules in April 2023 (CE 3.0); Mastercard similarly updates chargeback reason code definitions periodically. A model trained on pre-CE3.0 data will underperform on new reason code structures.
- **New fraud patterns:** Fraudsters adapt. A new category of friendly fraud (e.g., "item not received" abuse for digital gift cards) may emerge that the model was not trained on.
- **Merchant mix shift:** As Razorpay adds new merchant categories (healthcare, B2B SaaS, cross-border), the dispute patterns for those categories won't be represented in training data.
- **Silent degradation:** Unlike a software bug that causes an obvious error, model degradation is gradual. Win rates may drift down 3–5% over 6 months before anyone notices.

**Likelihood:** High — Rule changes are certain; fraud pattern evolution is certain. The question is detection and response speed.

**Impact:** Medium — Merchants don't lose money catastrophically, but the feature's competitive advantage erodes. If the AI win rate drops to below the human baseline, the feature has negative value.

### Mitigation
1. **Monthly performance dashboards:** Win rate tracked by reason code × confidence band. Automated alert if any bucket drops > 5pp month-over-month or > 10pp vs. 3-month rolling average.
2. **Card network change monitoring:** Dedicated person (or service) monitors Visa/Mastercard operating regulation updates. Confirmed rule changes trigger an immediate model review for affected reason codes, with auto-submit paused for those codes pending revalidation.
3. **Continuous feedback loop:** Every dispute outcome (won/lost/expired/withdrawn) tagged and ingested as a training signal. Model fine-tuning scheduled quarterly minimum.
4. **Kill switch:** If overall AI win rate drops below the trailing 90-day human baseline for 2 consecutive months, auto-submission is automatically suspended across all merchants until root cause is identified and model is retrained.
5. **Champion/challenger framework:** At GA, run 5–10% of eligible disputes through an alternative model version ("challenger") to continuously test whether the current model is still best-in-class.

**Owner:** ML Engineering, Analytics
**Review Cadence:** Monthly automated review; immediate action on alert triggers

---

## Risk Summary Matrix

| # | Risk | Likelihood | Impact | Priority | Mitigation Status |
|---|---|---|---|---|---|
| 1 | Model error → merchant loses winnable dispute | H | H | Critical | Mitigated (thresholds, hard checks, redress policy) |
| 2 | Regulatory non-compliance (RBI, card networks, DPDP) | M | H | Critical | Phase 0 gate; requires legal sign-off |
| 3 | Merchant trust erosion after high-profile AI error | M | H | High | Mitigated (amount threshold, comms plan, opt-out) |
| 4 | Adversarial merchants gaming the AI | M | M | High | Mitigated (forensics, anomaly detection, risk scoring) |
| 5 | Model performance degrades over time | H | M | High | Mitigated (monitoring, kill switch, feedback loop) |

**Residual risk after mitigations:** Medium. No mitigation eliminates risk entirely. The most significant residual risk is #2 (regulatory), which depends on external parties (RBI, card networks) whose positions may change. This risk should be reviewed at every major milestone.
