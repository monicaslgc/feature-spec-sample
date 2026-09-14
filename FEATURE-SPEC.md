# Feature Spec: Automated Identity Document Verification

**Status:** Draft for review
**Owner:** Product (Monica Costa)
**Stakeholders:** Engineering, Compliance, Support, Design
**Target release:** Q2 (phased)

> Note: this is a sample spec written for a portfolio. The product ("Northfield," a fictional consumer fintech app), the vendor, and every number in here are invented. The format and level of detail are real — this is close to how I'd actually scope and write a spec like this.

## 1. Problem

Right now, new users on Northfield verify their identity by uploading a photo of an ID and a selfie, which a member of our ops team reviews by hand. That's fine at 200 signups a day. It is not fine at the 1,500+/day we're projecting once the referral program launches in Q2, and it already shows: median review time is 14 hours, and it spikes past 48 hours on weekends when ops is short-staffed. Users who don't get verified within a day mostly don't come back.

We also have a compliance deadline: our banking partner requires us to move to an auditable, timestamped verification process with a documented decision trail by end of Q2, regardless of volume. Manual review with review notes in a shared spreadsheet doesn't meet that bar.

## 2. Goals

- Automate the common case (clear photo, valid ID, face match) so it resolves in under 2 minutes with no human in the loop.
- Route anything the system isn't confident about to manual review, with the system's reasoning attached, not just the raw images.
- Produce a decision record for every verification (automated or manual) that satisfies our banking partner's audit requirement.
- Get median time-to-verified under 5 minutes and same-day resolution to 95%+, even at projected Q2 volume.

## 3. Non-goals

- We are not changing what documents are accepted (still passport, driver's license, national ID card). Adding new document types is a separate spec.
- We are not building our own OCR or face-matching models. This spec assumes a third-party verification vendor; vendor selection is covered in a separate doc, not here.
- We are not removing the manual review queue. It gets smaller and better-informed, not eliminated — there will always be a tail of cases (damaged IDs, name mismatches, expired documents) that need a person.
- Re-verification (e.g. if a user's ID expires) is out of scope for this phase.

## 4. Proposed solution

### 4.1 Flow overview

1. User uploads ID photo + selfie in-app (unchanged from today).
2. We send both to the verification vendor's API.
3. Vendor returns: document authenticity score, extracted fields (name, DOB, document number, expiry), face match score, and a list of flags (e.g. "expired," "low image quality," "font irregularity").
4. Our decision engine applies thresholds (see 4.3) to auto-approve, auto-reject, or route to manual review.
5. Every outcome — automated or manual — is written to a verification record with: input images (retained per policy, see 4.4), vendor response, decision, decision reason, reviewer (if any), and timestamp.
6. User sees a status update in-app. Auto-approved users see it within ~2 minutes; queued users see "under review" with an expected timeframe.

### 4.2 What changes for the manual review team

Today reviewers look at raw images and make a judgment call with no structured context. After this change, reviewers only see cases that already failed automated thresholds, and they see *why*: the specific flags and scores that pushed it into the queue, not just the photos. The expectation is this cuts average manual review time per case roughly in half, because reviewers aren't starting from zero.

### 4.3 Decision thresholds (initial, tunable)

| Condition | Outcome |
|---|---|
| Authenticity score ≥ 0.92 AND face match ≥ 0.90 AND no flags | Auto-approve |
| Authenticity score < 0.50 OR "document expired" flag OR "likely fraudulent" flag | Auto-reject |
| Everything else | Manual review, with vendor flags attached |

These thresholds are a starting point, not final. Compliance needs to sign off before launch, and we should expect to tune them in the first few weeks based on false-positive/negative rates — see open questions.

### 4.4 Data handling

- ID images and extracted PII are encrypted at rest, same as today.
- Retention: verification records (decision + metadata) are kept for 7 years per our banking partner's requirement. Raw images are retained for 90 days post-decision, then deleted, unless a case is under active dispute.
- Every access to a verification record (by a reviewer, or by anyone pulling records for audit) is logged with who, when, and why.

## 5. Requirements

### Functional

- FR1: System must call the verification vendor synchronously on upload and handle timeouts gracefully (retry once, then fall back to manual queue).
- FR2: Auto-approve and auto-reject decisions must be reversible — a compliance admin can override any automated decision, and that override is itself logged.
- FR3: Manual reviewers must see vendor flags and scores alongside the images, not just the images.
- FR4: Every verification record must be exportable in a fixed format for audit requests, without engineering involvement.
- FR5: Users who are auto-rejected must get a clear next step (resubmit, contact support) — not a dead end.

### Non-functional

- NFR1: Vendor API call must complete (or time out) within 15 seconds; the in-app "verifying..." state should never hang indefinitely.
- NFR2: System must handle vendor downtime by falling back to manual review, not by blocking signups.
- NFR3: Decision engine changes (threshold tuning) must be deployable without a full app release, since we expect to iterate on thresholds post-launch.

## 6. Success metrics

- Median time-to-verified: from ~14 hours to under 5 minutes.
- Same-day resolution rate: from current baseline (~78%) to 95%+.
- Manual review queue volume: reduced by 70%+ compared to today's fully-manual flow, even as total signups grow.
- False-negative rate (fraudulent ID auto-approved): tracked from day one, hard cap agreed with Compliance before launch — this metric can block a rollout, nothing else can.

## 7. Rollout plan

- **Phase 1:** Ship to 10% of new signups, manual review still gets a copy of every case (shadow mode) so we can compare automated decisions against human ones without any user-facing risk.
- **Phase 2:** If false-negative rate holds under the agreed cap after 2 weeks in shadow mode, go live for that 10% for real, expand to 50%.
- **Phase 3:** Full rollout, once thresholds are stable and Compliance has signed off on the shadow-mode results.

Each phase has a go/no-go review with Compliance before moving to the next one.

## 8. Risks

- **Vendor accuracy on non-US documents.** Our current vendor's stated accuracy is strongest on US-issued IDs. We serve a meaningfully international user base. Mitigation: pull document-type-level accuracy stats from the vendor before launch and set more conservative thresholds for document types with weaker coverage.
- **Threshold tuning period creates a rocky first few weeks.** We should expect the manual queue to look different (not necessarily smaller) right after launch while thresholds settle. Support should be briefed so this doesn't look like a regression.
- **Compliance sign-off could slip the timeline.** The Q2 deadline is fixed; this spec's timeline assumes Compliance review starts now, in parallel with engineering, not after a build is done.

## 9. Open questions

- Who owns threshold tuning after launch — Product or a dedicated trust & safety function? Affects NFR3's "no full release needed" requirement.
- Do we need a distinct flow for re-verification when this phase explicitly excludes it, or will support end up needing a stopgap regardless?
- What's an acceptable false-positive rate (real user auto-rejected)? We've agreed a hard cap on false negatives; we haven't agreed one for false positives, and a high rate there just pushes people into a support queue instead of a review queue.

## 10. Acceptance criteria

- [ ] A new signup with a clear, valid ID and matching selfie is auto-approved in under 2 minutes in 95%+ of cases in shadow-mode testing.
- [ ] A verification record exists for every signup attempt, automated or manual, and is exportable without engineering support.
- [ ] Compliance has reviewed and signed off on the thresholds in section 4.3 before Phase 1 begins.
- [ ] Manual reviewers report (via a short survey after 2 weeks) that flag context measurably speeds up their review process.
- [ ] Vendor downtime falls back to manual review with no user-facing error state.
