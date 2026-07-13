# PRD: GRN Mismatch Review & Resolution
---
## 1. Problem

When a delivery arrives, an operator logs a GRN capturing received quantities and prices against the PO's confirmed values. Supy already computes the variance between confirmed and received, but no one can act on it inside the platform. Resolution happens off-platform — a phone call, a WhatsApp message, a verbal price adjustment. Consequences:

- **Nothing reconciles.** The GRN, the supplier's invoice, and the payment can all disagree, with no recorded decision explaining the gap
- **Mismatches get lost.** With no queue, owner, or status, follow-up depends on someone remembering
- **No trace.** Accepted, adjusted, or written-off variances live in memory or unsearchable chat threads, useless in an audit months later

**Who this affects:**
- **Branch/warehouse operators**, who have no way to flag "this isn't right" beyond messaging someone
- **Procurement or accounts staff**, accountable for what gets paid, who find out about discrepancies informally and late in the payment cycle

---

## 2. Goals & non-goals

**Goals**
- Resolve procurement discrepancies entirely inside Supy
- Give whoever approves spend a structured decision trail per mismatch: who decided, what, and why
- Cut the time from delivery to resolution from indefinite to days
- Make the dollar impact of mismatches reportable — internally and as ROI proof to restaurant buyers (section 8)

**Non-goals** *(reasoning in section 5)*
- Supplier-facing communication or negotiation — stays off-platform
- Generating accounting-system credit memos or ledger postings — this feature records intent, not the financial artifact
- Multi-tier approval chains — v1 ships one reviewer tier
- Bulk resolution of multiple GRNs at once
- Per-SKU or per-supplier tolerance configuration — v1 ships one global tolerance ("global" = not per-SKU/per-supplier; there are still two independent dimensions, a quantity % and a price %)

---

## 3. Assumptions

- **Supy's existing permission system can map "reviewer" to an existing role** with resolution authority (e.g., Procurement Lead, Accounts) — no new permission is invented
- **GRN capture already computes the confirmed-vs-received variance per line.** This feature adds tolerance-based flagging, a queue, and resolution on top
- **Accounting/AP workflows outside Supy stay unchanged.** Credit notes and payment adjustments continue to be processed manually; this feature stops at a clean, structured record of the decision
- **All supplier communication continues outside the platform**
- **A single global tolerance is an acceptable v1 starting point** (platform-wide thresholds; independent quantity % and price % dimensions), even though customers will eventually want per-supplier or per-category thresholds

---

## 4. Primary user

**The primary user is the person who approves what gets paid** — typically a procurement or accounts lead with cross-branch visibility.

**Why not the receiving operator:** resolving a mismatch is a decision with financial consequence, not a data-entry step. It belongs with whoever is accountable for the spend, not whoever was at the loading dock.

**Two-role model:** operators surface and contextualize mismatches as a byproduct of receiving — they attach a note when something looks off and are notified how it was resolved (section 6.2). A designated reviewer works a dedicated queue to resolve them. Which title holds resolution authority is a per-customer mapping (section 3).

---

## 5. Scope

### In scope

- Automatic flagging of a GRN line when received quantity or price deviates from confirmed PO values beyond a **configurable tolerance**
- A **severity tier** (Small / Medium / Critical) per mismatch, driven by dollar impact (section 6.1)
- A **Mismatch Queue**: one place a reviewer works from, across branches and suppliers
- A **notification** when a mismatch is flagged or escalated (section 6.2)
- A **review view** per GRN: confirmed vs. received side by side, variance size and direction, who received it
- **Four resolution actions**, applied per line item (a GRN can have some lines resolved and others open):
  - **Settle at received quantity** — pure quantity variance only (price matches confirmed): the corrected line total (received qty × confirmed unit price) is **calculated automatically and locked**. Covers a shortfall written off or an overage kept
  - **Return surplus** — over-delivery only, price as confirmed: settle at the confirmed quantity, i.e. pay only for what was ordered. Also auto-calculated and locked
  - **Record debit/credit note** — the one action with a manually entered amount: offered whenever price varies, and the **sole action (besides Escalate) when both quantity and price are off** — one note covers the line's full dollar impact. **Credit note** when the business was overcharged (supplier owes money back); **debit note** when undercharged (business owes more). The amount defaults to the system-calculated variance and is editable
  - **Escalate** — hand the line to the **Accounts Manager** with a note. Single fixed destination, not a pick-list (section 6.9)
- A **mandatory short note** on every resolution, and a full **audit trail** (who, what, when, before/after values)
- **Status** at both the line and GRN level
- A **read-only view** for the operator to see what happened to a mismatch they flagged

### Explicitly out of scope, and why

| Cut | Why |
|---|---|
| **Supplier communication** | Stays exactly as today. This feature ends at "we decided X" — nothing is sent to the supplier. |
| **Generating financial documents** | "Record credit note" captures intent and amount, not an accounting-system credit memo. That means AP-system integration — a separate, larger project. |
| **Multi-step approval chains** | Almost every customer starts with a single reviewer tier. Variance-size routing would be a contained addition later, not a redesign. |
| **Bulk resolution** | A real efficiency want at volume, but it multiplies audit-trail and permission edge cases. Single-item resolution first. |
| **Per-SKU/per-supplier tolerance** | Config surface with no evidence yet that customers need the granularity. |
| **Operator-side resolution** | Resolution authority stays with the reviewer (section 4). If reviewers rubber-stamp trivial cases, raise the tolerance rather than delegate approval down. |

### Edge cases handled

| Case | Handling |
|---|---|
| Under-delivery (received < confirmed qty, price as confirmed) | Flagged if outside tolerance; resolved via **Settle at received quantity** — total is arithmetic (received qty × confirmed price), calculated and locked |
| Over-delivery (received > confirmed qty, price as confirmed) | Flagged the same way — paying for unordered stock is a liability, not upside. Two auto-calculated outcomes: **Settle at received quantity** (keep the extra) or **Return surplus** (pay only for the confirmed amount; pickup arranged off-platform) |
| Price higher than confirmed | Flagged; **credit note** — the business was overcharged, supplier owes the difference |
| Price lower than confirmed | Flagged; **debit note** if the business genuinely owes more (e.g. supplier corrects a stale low price) — still a decision, just not automatic |
| Both qty **and** price off on the same line | Resolved with **one action: Record debit/credit note**, covering the line's full dollar impact. Auto-settle actions aren't offered — a locked arithmetic total can't absorb a price judgment, and splitting the line into two decisions would break the one-action-per-line model |
| Multiple mismatched lines, one GRN | Each line resolved independently; GRN status reflects the mix ("2 of 5 lines resolved") |
| Mismatch within tolerance | No flag, no queue entry — variance still stored on the GRN for reporting |
| GRN with no mismatch | No change from today's behavior |
| Escalation target unavailable/leaves | **Out of scope for v1** — escalation is a flat reassignment, not a managed workflow with SLAs |
| Split delivery (partial now, remainder later) | **Out of scope** — v1 assumes one GRN reflects the final state of a delivery; a receiving-flow question, not a resolution one |

---

## 6. Solution

### 6.1 Detection & severity

At GRN submission, each line's received qty/price is compared to the PO's confirmed values. If either falls outside the tolerance band, the line is marked `mismatch_status = open`.

Every flagged line gets a severity tier — Small, Medium, or Critical — based on **dollar impact**, not percentage: a small percentage on a large order can matter more than a large percentage on a $2 item. The queue defaults to sorting by severity, then age. (Thresholds are placeholders, tuned per customer like tolerance.)

Severity is derived from current thresholds while a line is open, and **snapshotted onto the record when the line is resolved or escalated**. Threshold changes re-tier only still-open lines; closed lines keep the tier they were decided under.

Tolerance and severity configuration is an **admin setting, not part of the reviewer's day-to-day surface** — the prototype exposes it in the toolbar purely as a demo control. Inputs are validated: no negative values, and the Critical threshold must be higher than Medium (otherwise the Medium tier would be unreachable).

### 6.2 Notifications

A queue nobody checks is the same failure mode as no queue:
- The reviewer gets a notification (and a live badge count) when a line is flagged or escalated to them
- The operator gets a notification when a mismatch they flagged is resolved

### 6.3 Mismatch Queue

Scoped to the reviewer's visibility (branches/suppliers, per existing permissions). Each row: supplier, branch, GRN date, lines flagged, severity, **$ exposure**, age, status. Exposure is the **sum of absolute line impacts** — signed variances are never netted, so offsetting lines can't hide each other. A row's severity is the tier of the GRN's total exposure; individual lines carry their own tier in the detail view. Sortable and filterable, because reviewers triage by money and by how long it's been sitting.

### 6.4 Review view

Per mismatched line: item, confirmed qty/price, received qty/price, variance (% and $), and any operator note. A comparison, not a form — the reviewer's job is to understand what happened before deciding.

### 6.5 Resolution

Each line takes exactly one action, and which actions are offered depends on the variance type: quantity-only never shows a manual dollar field, price-only never shows a quantity settlement, both-off gets only the debit/credit note (covering the full impact), and pure-quantity over-delivery is the only case with two outcomes (keep vs. return). For the auto-calculated actions the amount is arithmetic — nothing to mis-enter or fake. The debit/credit note is the only human-entered dollar amount, because it's the one case with negotiated judgment.

Residual risk on the note is managed, not eliminated: the field is pre-filled with the system-calculated variance as an anchor, and a note is always required — the "why" is exactly what gets skipped under time pressure, and exactly what matters in a dispute months later. This doesn't make fraud impossible (that takes real financial controls: approval thresholds, reconciliation against the supplier-issued document); it moves the business from zero traceability to full attribution for the one action that still requires a human number.

### 6.6 Trace

Every resolution appends to the GRN's activity log: actor, timestamp, action, note, before/after values. A record that survives staff turnover and supplier disputes.

### 6.7 Status model

- **Line status:** `Open` → `Resolved` (terminal), or `Open` → `Escalated` → `Resolved`. Escalation is recorded separately from the resolution, so resolving a previously escalated line keeps both entries in the audit trail
- **GRN status is derived, not stored:** `No mismatch` / `Needs review` (≥1 open line) / `Escalated` (no open lines, ≥1 escalated) / `Resolved` (all lines resolved). A GRN can't look done while a line is still open

### 6.8 Resolution flow

```
GRN submitted
      │
      ▼
Received vs. confirmed compared automatically
      │
      ▼
Within tolerance? ──yes──▶ No mismatch. Nothing further happens.
      │
      no
      ▼
Flagged, severity assigned → lands in the Mismatch Queue
Reviewer notified (section 6.2)
      │
      ▼
Reviewer opens the GRN, compares confirmed vs. received side by side
      │
      ▼
Reviewer resolves the line - the options offered depend on the variance type:
  [ Settle at received qty ]   [ Return surplus* ]   [ Record debit/credit note** ]   [ Escalate ]
   *only on over-delivery with price as confirmed
   **offered whenever price varies; the sole action (besides Escalate) when both qty and price are off
      │
      ▼
Resolution logged to the audit trail: who, what, why, when
Operator notified
```

### 6.9 Key decisions and why

- **Resolve at the line level, not the GRN level.** A rounding issue on one item shouldn't force the same resolution as a real shortfall on another, and partial progress stays visible
- **Auto-calculate and lock the amount for pure quantity variances.** The corrected total is arithmetic, not judgment — no typeable number to mis-enter or game
- **One "debit/credit note" action, both directions.** Credit when overcharged, debit when undercharged — the accounting terms an accounts team expects
- **Mandatory note on every action, including auto-calculated ones.** An optional "why" is a skipped "why"
- **The note captures intent, not the accounting artifact.** Keeps the feature inside Supy's boundary (section 5)
- **A global tolerance and severity scale, not per-item.** Platform-wide thresholds with two independent dimensions (quantity % and price %). Per-item granularity has no demand evidence yet; ship the simple version to find out
- **Escalate routes to one fixed destination (the Accounts Manager).** v1 is a flat, single-tier handoff — a pick-list would imply routing this version deliberately doesn't support. If usage shows escalations need different targets, that's a contained addition

---

## 7. User stories

*Prototype scope note: the prototype demonstrates the reviewer flow end to end. Operator note **capture** (part of GRN submission) and event-driven flag persistence are receiving-flow concerns it stubs — notes appear as seeded data, and flags are recomputed live from the current tolerance.*

| User Story | Acceptance Criteria | Design Notes | Technical Corner |
|---|---|---|---|
| As a **receiving operator**, I want to add a note when something looks off on a delivery, so the reviewer has context without me messaging anyone separately. | • Note field is optional at GRN submission, never blocking.<br>• Note is visible to the reviewer on the review view.<br>• Note persists and shows in the operator's own read-only view later. | A single free-text field, not a structured form — operators are moving fast during receiving. | Note stored as a per-line field on the GRN line record; no separate table for v1; default to empty string, not null. |
| As a **reviewer**, I want lines outside an agreed tolerance to be automatically flagged, so I don't have to manually check every GRN. | • Line status = "Open" when qty% or price% variance exceeds configured tolerance.<br>• Within-tolerance lines create no flag and no queue entry.<br>• Qty and price tolerance are configured independently. | Evaluated per line, not per GRN — one bad line shouldn't hide inside an otherwise-clean delivery. | Flag should be a stored, queryable field set on submission (event-driven), not recomputed on page load — matters at volume. |
| As a **reviewer**, I want every flagged mismatch to show a severity tier based on $ impact, so I can tell what matters without doing the math. | • Each flagged line shows Small/Medium/Critical from configured $ thresholds.<br>• Thresholds are independent of tolerance %.<br>• Changing thresholds re-evaluates only still-open lines; resolved/escalated lines keep their historical tier. | Dollar-based, not percentage-based — a small % on a large order can matter more than a large % on a $2 item. | Severity derived at read time from stored $ impact + current thresholds while open; snapshotted onto the resolution/escalation record when the line closes. |
| As a **reviewer**, I want one queue showing every open mismatch across branches and suppliers, sorted by severity then age, so I can triage without hunting through GRNs. | • Queue lists every GRN with ≥1 flagged line, scoped to the reviewer's permitted branches/suppliers.<br>• Default sort: worst severity first, then oldest first within a tier.<br>• Filterable by status, searchable by supplier/branch/GRN ID. | Age and $ impact are the two things a reviewer triages by. | Needs an indexed view joining lines by status + severity + branch scope. Pagination isn't in the v1 prototype but will matter at real volume. |
| As a **reviewer**, I want to be notified the moment something needs my review, so I don't have to remember to check the queue. | • Notification fires when a line is newly flagged or newly escalated to this reviewer.<br>• Notification includes GRN ID, supplier, $ impact.<br>• A live badge reflects the current count needing review. | Closes the "queue nobody knows to check" failure mode (section 6.2). | Channel (in-app vs. email, section 9) is open; assumes an existing Supy notification service. |
| As a **reviewer**, I want a pure quantity shortfall or a kept overage to auto-calculate the corrected payable amount, so there's no manually typed number to get wrong or dispute. | • "Settle at received quantity" only offered when price matches confirmed and quantity doesn't (combined qty+price routes to the debit/credit note, per section 5).<br>• Line total shown = received qty × confirmed unit price, not editable.<br>• Note still required before submitting. | Removes the fraud/error surface for this class entirely — the math is deterministic. | Compute at resolution time; persist the settled qty + total on the resolution record itself so the audit trail is self-contained. |
| As a **reviewer**, I want to choose between keeping or returning surplus stock on an over-delivery, so the payable amount reflects what the business actually keeps. | • "Return surplus" only offered when received qty > confirmed qty and price matches confirmed.<br>• Selecting it locks the line total to confirmed qty × confirmed unit price.<br>• Note prompts for the pickup/return arrangement. | Physical pickup stays off-platform per the non-goals — this records the financial decision only. | No returns/logistics integration in v1 — the note is a free-text record, not a tracked task. |
| As a **reviewer**, I want to record a credit or debit note for a genuine price discrepancy, pre-filled but editable, so I'm not starting from a blank field but still have room for what was negotiated. | • Only offered when received unit price differs from confirmed; the sole action (besides Escalate) when quantity is also off, in which case the pre-filled amount covers the line's full $ impact.<br>• Direction (credit/debit) auto-labels from overcharged vs. undercharged.<br>• Amount pre-fills with the calculated variance, editable; reference number optional at entry.<br>• Note required. | The one action with an unavoidable human-entered number — the calculated anchor reduces, not eliminates, room for error (section 6.5). | Store direction, amount, and reference as distinct fields, not one signed number — so section 8's metric can filter to credit notes without re-deriving direction. |
| As a **reviewer**, I want to escalate a mismatch to the Accounts Manager when I need a second opinion, so I'm not forced to make every call alone. | • Escalate is available regardless of variance type.<br>• No destination picker — always routes to the Accounts Manager role.<br>• Escalated lines remain actionable later (can still be resolved once feedback returns).<br>• The escalation entry survives in the audit trail after the line is resolved — stored alongside, not replaced by, the final resolution. | A fixed single destination matches the flat, single-tier model (section 6.9). | "Accounts Manager" should resolve to an assigned user/role in Supy's permission system; the prototype uses a fixed label (role mapping is a section 9 open question). |
| As an **operator**, I want to be notified once my flagged mismatch is resolved, so I know the outcome without checking back. | • Operator is notified when any line they flagged reaches a resolved state.<br>• Their read-only view shows resolution type, amount (if any), and the reasoning note.<br>• No resolve actions are available to the operator. | Keeps the two-role boundary (section 4) visible in the UI itself. | Notification scoped to the GRN's `receivedBy` field; an operator who's left the company is an out-of-scope edge case (section 5). |
| As a **Supy customer success or sales stakeholder**, I want to see total $ recovered via credit notes per store per month, so I can prove the platform's ROI to that customer. | • Metric sums only debit/credit-note resolutions where direction = credit.<br>• Excludes settle/return actions (arithmetic, not a negotiated win) and debit notes (money owed out).<br>• Reportable per store per month; prototype demos a rolling 30-day view. | Deliberately narrow — a number inflated with routine corrections wouldn't hold up as a credibility claim (section 8). | Aggregation across resolved lines by type + direction + date. Client-side 30-day rollup in the prototype; a real version needs a reporting/BI layer. |

---

## 8. How we'd know it's working

### Product (adoption & speed)
- **Share of GRN mismatches reaching a terminal state inside the platform**, trending toward ~100% within a few weeks. If reviewers still resolve over WhatsApp and skip the queue, this stays flat — the failure mode to watch
- **Median time from flagged to resolved.** Watch the trend, not just the number
- **% of resolutions with a substantive note** (not "ok" or "-"). A big share of low-effort notes means the mandatory-note decision isn't achieving its purpose

### Business (the number that justifies the feature)
- **Margin Recovered: total $ of credit notes issued through the platform, per month, per customer.** Scoped to credit notes only — settling at received or confirmed quantity is correct arithmetic, not a negotiated win, and a debit note is money going out. Counting those would inflate the number and undercut its credibility with a restaurant buyer
- **Average $ recovered per credit-note resolution** (numerator and denominator both scoped to credit notes) — a concrete ROI number per account

### Operational (is the machine calibrated correctly)
- **Backlog: count of lines open more than 7 days.** A queue that never empties isn't being worked — a different problem than "reviewers are slow"
- **% of GRN lines auto-flagged vs. total lines with any variance.** Too high → tolerance too tight, reviewers drown in noise. Too low with continued complaints → too loose
- **Severity mix of the queue (e.g., % Critical).** A leading indicator that thresholds need recalibrating before it shows up as reviewer fatigue

**Leading indicator before there's volume:** whether reviewers open the queue unprompted in the first two weeks. If not, fix the notification path (section 6.2) before touching the resolution flow.

---

## 9. Open questions for the team

- Does "reviewer" map to an existing Supy role/permission today, or do we need a new permission bit? (Assumed yes, per section 3 — needs confirming with the platform/permissions owner.)
- What's a sane default tolerance and severity scale (placeholders used throughout), and should either differ by customer vertical (restaurant vs. central kitchen)?
- What channel should the notifications in section 6.2 ride on — existing in-app notifications, email, or both? (Assumed in-app for v1, pending what Supy already has.)
