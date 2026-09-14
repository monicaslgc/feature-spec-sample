# Feature Spec: Automated Identity Document Verification

**Status:** Draft for review
**Owner:** Product (Monica Costa)
**Stakeholders:** Engineering, Compliance, Support, Design
**Target release:** Q2 (phased)

> Heads up: this is a sample spec I wrote for my portfolio, not a real one. "Northfield" is a made-up consumer fintech app, the vendor's made up, every number in here is made up. The format and the level of detail are real, though — this is genuinely close to how I'd scope and write something like this.

## 1. Problem

Right now, new users on Northfield verify their identity by uploading a photo of an ID and a selfie, and someone on ops reviews it by hand. That's fine at 200 signups a day. It's not fine at the 1,500+/day we're projecting once the referral program launches in Q2 — and honestly it's already showing: median review time is 14 hours, and it spikes past 48 hours on weekends when ops is short-staffed. Users who don't get verified within a day mostly just don't come back.

There's also a compliance deadline sitting on top of this: our banking partner wants us on an auditable, timestamped verification process with a documented decision trail by end of Q2, regardless of volume. Manual review with notes in a shared spreadsheet doesn't get us there.

## 2. Goals

- Automate the common case (clear photo, valid ID, face match) so it resolves in under 2 minutes, no human involved.
- Route anything the system isn't confident about to manual review, and hand reviewers the system's reasoning, not just the raw images.
- Produce a decision record for every verification, automated or manual, that satisfies the audit requirement.
- Get median time-to-verified under 5 minutes and same-day resolution to 95%+, even at Q2 volume.

## 3. Non-goals

- Not changing what documents are accepted (still passport, driver's license, national ID card). New document types are their own spec.
- Not building our own OCR or face-matching models. This assumes a third-party verification vendor — vendor selection is a separate doc.
- Not removing the manual review queue. It gets smaller and better-informed, not eliminated. There's always going to be a tail of damaged IDs, name mismatches, expired documents that need a person.
- Re-verification (say, a user's ID expires) is out of scope for this phase. I know that's going to come up — see open questions.

## 4. Proposed solution

### 4.1 Flow overview

1. User uploads ID photo + selfie in-app (unchanged from today).
2. We send both to the verification vendor's API.
3. Vendor comes back with: a document authenticity score, extracted fields (name, DOB, document number, expiry), a face match score, and a list of flags ("expired," "low image quality," "font irregularity," etc.).
4. Our decision engine runs those through the thresholds in 4.3 — auto-approve, auto-reject, or manual review.
5. Every outcome gets written to a verification record: input images (retained per 4.4), vendor response, decision, decision reason, reviewer if any, timestamp.
6. User sees a status update in-app. Auto-approved users see it within ~2 minutes; queued users see "under review" with an expected timeframe.

### 4.2 What changes for the manual review team

Right now a reviewer just gets the raw images and has to make a call with basically no context. After this ships, they only see cases that already failed the automated thresholds, and they get the reasoning alongside the photos — which flags fired, what the scores were. My guess is that alone cuts average review time per case roughly in half, since nobody's starting from a blank slate anymore.

### 4.3 Decision thresholds (initial, tunable)

| Condition | Outcome |
|---|---|
| Authenticity score ≥ 0.92 AND face match ≥ 0.90 AND no flags | Auto-approve |
| Authenticity score < 0.50 OR "document expired" flag OR "likely fraudulent" flag | Auto-reject |
| Everything else | Manual review, with vendor flags attached |

These are a starting point, not gospel. Compliance signs off before launch, and I'd expect us to tune them in the first few weeks once we see real false-positive/negative rates — see open questions.

### 4.4 Data handling

- ID images and extracted PII are encrypted at rest, same as today.
- Retention: verification records (decision + metadata) kept 7 years per the banking partner's requirement. Raw images kept 90 days post-decision, then deleted, unless a case is under active dispute.
- Every access to a verification record — reviewer or anyone pulling records for audit — gets logged with who, when, why.

## 5. Requirements

### Functional

- FR1: System calls the verification vendor synchronously on upload and handles timeouts gracefully (retry once, then fall back to manual queue).
- FR2: Auto-approve and auto-reject decisions are reversible — a compliance admin can override any automated decision, and the override itself gets logged.
- FR3: Manual reviewers see vendor flags and scores alongside the images, not just the images.
- FR4: Every verification record is exportable in a fixed format for audit requests, no engineering involvement needed.
- FR5: Auto-rejected users get a clear next step (resubmit, contact support), not a dead end.

### Non-functional

- NFR1: Vendor API call completes or times out within 15 seconds. The "verifying..." state should never just hang.
- NFR2: Vendor downtime falls back to manual review, doesn't block signups.
- NFR3: Threshold tuning is deployable without a full app release — we're going to be iterating on these post-launch.

## 6. Success metrics

- Median time-to-verified: from ~14 hours to under 5 minutes.
- Same-day resolution rate: from current baseline (~78%) to 95%+.
- Manual review queue volume: down 70%+ from today's fully-manual flow, even as signups keep growing.
- False-negative rate (fraudulent ID auto-approved): tracked from day one, hard cap agreed with Compliance before we launch. This is the one metric that can block a rollout — nothing else does.

## 7. Rollout plan

- **Phase 1:** 10% of new signups, but manual review still gets a copy of every case (shadow mode) so we can compare automated decisions against human ones without any user-facing risk.
- **Phase 2:** If false-negative rate holds under the agreed cap after 2 weeks in shadow mode, go live for real on that 10%, expand to 50%.
- **Phase 3:** Full rollout, once thresholds have settled and Compliance has signed off on the shadow-mode results.

Each phase gets a go/no-go review with Compliance before we move to the next one.

## 8. Risks

- **Vendor accuracy on non-US documents.** The vendor's numbers are strongest on US-issued IDs, and a meaningful chunk of our users aren't US-based. I want document-type-level accuracy stats from them before launch so we can set more conservative thresholds anywhere their coverage is weaker, instead of finding out the hard way.
- **The first few weeks post-launch are going to look messy.** The queue will look different, not necessarily smaller, while thresholds settle. Worth briefing Support ahead of time so nobody reads that as a regression.
- **Compliance sign-off is the real risk to the timeline.** The Q2 date isn't moving, so this only works if Compliance review starts now, alongside engineering, not after something's already built and sitting there waiting on a stamp.

## 9. Open questions

- Who owns threshold tuning after launch, Product or a dedicated trust & safety function? This affects NFR3.
- We've scoped re-verification out, but is support going to need some kind of stopgap for expired-ID cases regardless?
- What's an acceptable false-positive rate (a real user gets auto-rejected)? We've agreed a hard cap on false negatives. We haven't agreed one for false positives, and a high rate there just pushes people into a support queue instead of a review queue — still a cost, just a different one.

## 10. Acceptance criteria

- [ ] A new signup with a clear, valid ID and matching selfie is auto-approved in under 2 minutes in 95%+ of cases in shadow-mode testing.
- [ ] A verification record exists for every signup attempt, automated or manual, and is exportable without engineering support.
- [ ] Compliance has reviewed and signed off on the thresholds in section 4.3 before Phase 1 begins.
- [ ] Manual reviewers report (short survey after 2 weeks) that flag context actually speeds up their review process.
- [ ] Vendor downtime falls back to manual review with no user-facing error state.
