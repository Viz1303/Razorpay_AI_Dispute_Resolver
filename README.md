# Portfolio Project: AI Dispute Resolver for Razorpay

**Type:** AI Product Management portfolio case study
**Industry:** Indian Fintech / Payment Processing
**Format:** Simulated internal PM deliverable
**Target audience:** Hiring managers and PMs at AI-first product companies

---

## What This Project Is

This repository contains a complete, production-grade product spec for a hypothetical AI feature built on top of Razorpay's merchant platform — written as if it were a real internal deliverable ready for engineering and leadership review.

The feature — **AI Dispute Resolver** — is an AI agent embedded in Razorpay's merchant dashboard that automatically triages, investigates, and resolves payment chargebacks, escalating only edge cases to human agents.

This is not a slide deck. It is not a blog post. It is the kind of document a PM would actually write before kicking off a sprint.

---

## Why Razorpay, Why Disputes?

Razorpay was chosen because:
- It is one of the few Indian fintechs operating at sufficient scale to make an AI automation investment this large financially viable
- It operates in a regulatory environment (RBI, NPCI, Visa/Mastercard) that adds genuine complexity to the product and trust design — not just a technical problem
- The dispute management surface is well-understood enough in the industry that the spec can be grounded in real data and real process constraints
- It has a clear dual-stakeholder tension: merchant experience vs. risk/compliance — which makes for interesting product tradeoffs

Payment disputes were chosen because:
- They are high-frequency, structured enough for AI automation, and have clear measurable outcomes (won/lost/expired)
- The failure modes are meaningful: a wrong AI decision has a direct financial consequence for a real small business
- This is an area where AI genuinely outperforms today's process (deadline tracking, evidence completeness, formatting) — not just a "let's add AI" exercise

---

## Document Guide

### [spec.md](spec.md) — Feature Specification
The full product spec. Designed to be handed to an engineering lead and generate real pushback on scope, data access, and system design. Covers:
- Problem statement with evidence (not assumptions)
- Three-stage AI agent architecture in plain English
- Confidence scoring and escalation logic with specific thresholds
- Five distinct failure modes, each with concrete guardrails
- Phased rollout plan with specific go/no-go criteria
- Open questions that a real PM would need answered before engineering begins

### [risk_register.md](risk_register.md) — Risk Register
Five top risks assessed by likelihood and impact, with mitigations:
1. AI model error causing merchant loss
2. Regulatory non-compliance (RBI, card networks, DPDP Act)
3. Merchant trust erosion after a high-profile error
4. Adversarial merchants gaming the AI
5. Silent model degradation over time

### [business_case.md](business_case.md) — Business Case
A bottom-up cost and savings model. All assumptions are explicitly labelled (A1–A13) and traced to public sources. Covers:
- Baseline cost model using industry benchmarks and public Razorpay data
- Projected operational savings broken down by category
- Merchant NPS and retention impact, with sensitivity haircuts
- 3-year simplified P&L and payback analysis
- Bear/base/bull case sensitivity analysis

---

## AI PM Skills This Project Demonstrates

The work in this repo is best read as evidence of specific PM capabilities rather than a self-assessment of them. A few things worth noting for context:

- **Human-in-the-loop design:** The confidence threshold model (Section 6 of the spec) was the hardest design decision — deciding exactly *when* to let AI act autonomously vs. defer, and making that a product choice with measurable guardrails rather than an engineering default.
- **Honest business cases:** The business case separates a defensible operational savings model (₹22 crore/year, bottom-up, cited sources) from a speculative retention hypothesis (₹130 crore/year, directional only). Most portfolio business cases don't make this distinction. This one does, and explains what data would be needed to validate the speculative number.
- **Failure mode thinking as product work:** The five failure modes in the spec — particularly adversarial merchants gaming the AI and silent model degradation — are not engineering concerns. They are product design constraints that determine the feature's trust architecture.
- **Regulatory awareness:** The risk register covers RBI data localisation, DPDP Act 2023, and Visa CE 3.0 as first-class product risks, not legal footnotes. Compliance constraints shaped the architecture (e.g., India-only inference infrastructure, Phase 0 legal gate).

---

## What This Project Does Not Claim

- This is not based on inside knowledge of Razorpay's actual systems, infrastructure, or roadmap
- The financial estimates are directionally useful, not precise — all assumptions are clearly stated
- This spec would require significant iteration with engineering, legal, and design teams before it became a real sprint backlog
- The AI architecture described is conceptual; actual implementation choices (which LLM, which infrastructure, training pipeline) would be made by engineering

---

## About

Created as a portfolio project to demonstrate AI product management thinking for PM roles at AI-first companies. Built without access to internal Razorpay data — all market figures are from public sources cited in business_case.md.

**Author:** Vishal Arasu
**LinkedIn:** https://www.linkedin.com/in/vishal-arasu-dataanalyst/
**Other projects:** [Link to portfolio]
