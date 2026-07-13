# PRD: GRN Mismatch Review & Resolution
---
## 1. Problem

When a delivery arrives, an operator logs a GRN capturing what was actually received - quantities and prices - against the PO's confirmed values. Supy already computes the variance between confirmed and received. **What it doesn't do is let anyone act on that variance inside the platform.**

Today, once a variance is recorded, it just sits there. Nobody is notified, nobody owns it, and there's no way to say "we're accepting this," "the supplier owes us a credit," or "this needs a second opinion." So **the actual resolution happens entirely outside the platform** - a phone call, a WhatsApp message, a verbal agreement to adjust the price. The consequence isn't just inconvenience:

- **Nothing reconciles.** The GRN says one thing, the supplier's invoice may say another, and there's no recorded decision explaining the gap. Whoever processes payment either pays the wrong amount or has to go ask around
- **Mismatches get lost.** With no queue, no owner, and no status, a shortfall discovered on a busy receiving day just doesn't get followed up on. **It's not that the resolution is hard - it's that there's no reason it would happen today versus whenever someone remembers**
- **There's no trace.** If a variance was accepted, adjusted, or written off, that decision - and the reasoning behind it - lives in someone's memory or a WhatsApp thread that isn't searchable in an audit six months later

**Who this affects:**
- **Branch/warehouse operators**, who receive deliveries and currently have no way to flag "this isn't right" beyond noting it and separately messaging someone
- **Procurement or accounts staff**, who are accountable for what the business ends up paying, and currently find out about discrepancies informally - usually only when an invoice doesn't match the PO late in the payment cycle

---

## 2. Goals & non-goals


**Goals**
- **Resolve procurement discrepancies entirely inside Supy** - no more WhatsApp/email chasing
- **Give whoever approves spend a structured decision trail** for every mismatch: who decided, what, and why
- **Cut the time between "delivery received" and "resolved"** from indefinite to a matter of days
- **Make the dollar impact of mismatches visible and reportable** - not just to the internal team, but as proof to restaurant buyers that Supy pays for itself (see paragraph 7, Business metrics)

**Non-goals** *(detailed reasoning in paragraph 5)*
- Supplier-facing communication or negotiation - stays outside the platform, exactly as today
- Generating accounting-system credit memos or ledger postings - this feature records intent, not the financial artifact
- Multi-tier approval chains - v1 ships one reviewer tier
- Bulk resolution of multiple GRNs at once
- Per-SKU or per-supplier tolerance configuration - v1 ships one global tolerance

---

## 3. Assumptions

This PRD is written on top of a few things taken as given rather than redesigning:

- **Supy's existing permission system already has, or can map to, a role with resolution authority.** This PRD doesn't invent a new permission - it assumes "reviewer" lands on an existing role (e.g., Procurement Lead, Accounts)
- **GRN capture already computes the confirmed-vs-received variance per line.** This feature adds tolerance-based flagging, a queue, and resolution on top of a comparison that already exists - it doesn't rebuild that math
- **Accounting/AP workflows outside Supy stay unchanged for now.** Credit notes and payment adjustments continue to be processed manually (or via whatever export/integration already exists) - this feature stops at producing a clean, structured record of the decision
- **All supplier communication continues outside the platform**, exactly as it does today - no in-app messaging or negotiation surface
- **A single global tolerance is an acceptable starting point** for every customer in v1, even though real customers will eventually want per-supplier or per-category thresholds

---

## 4. Primary user

### Who resolves mismatches

**The primary user is the person who approves what gets paid** - in most Supy customers, a procurement or accounts lead with visibility across multiple branches and the authority to decide how a variance gets settled.

### Why not the receiving operator

We are optimizing for the approver rather than the operator for one reason: **resolving a mismatch is a decision with financial consequence, not a data-entry step.** Accepting a shortfall, adjusting a price, or logging a credit note changes what the business pays the supplier. That decision needs to sit with whoever is accountable for the spend - not with whoever happened to be at the loading dock that morning, who didn't choose the supplier or negotiate the price.

### The two-role model

The operator is not a bystander, though:
- They **create** the mismatch by logging what was actually received, so the system needs to make it obvious, the moment they submit a GRN, that something is off
- They often hold context the reviewer doesn't ("driver said the rest is coming tomorrow," "box was visibly damaged") - so they need a lightweight way to attach a note, and later, **to be notified how it was resolved** (see paragraph 6.2)

So this PRD designs a **two-role workflow**: operators surface and contextualize mismatches as a byproduct of the receiving flow they already do; a designated reviewer works a dedicated queue to resolve them. *(Which specific title - procurement vs. accounts - holds resolution authority is a per-customer question; assumed to map to an existing role, per paragraph 3.)*

---

## 5. Scope

### In scope

- Automatically flagging a GRN line as a mismatch when received quantity or price deviates from confirmed PO values beyond a **configurable tolerance**
- A **severity tier** (Small / Medium / Critical) per mismatch, driven by dollar impact, so the queue can be scanned by what matters most, not just sorted by date (paragraph 6.1)
- A **Mismatch Queue**: one place a reviewer works from, across branches and suppliers
- A **notification** the moment a mismatch is flagged or escalated, so a reviewer doesn't have to remember to go check (paragraph 6.2)
- A **review view** per GRN: confirmed vs. received side by side, variance size and direction, who received it
- **Four resolution actions**, applied per line item (a GRN can have some lines resolved and others still open), split cleanly by what kind of decision each one actually is:
  - **Settle at received quantity** - for a pure quantity variance, the corrected line total (received qty × the already-agreed unit price) is **calculated automatically and locked**, not typed in. Covers a shortfall being written off, or an overage being kept
  - **Return surplus** - over-delivery only: settle at the confirmed quantity instead, i.e. pay only for what was ordered. Also auto-calculated and locked
  - **Record debit/credit note** - the one action with a manually entered amount, because it's the only one with genuine judgment in it: any price variance, in either direction. A **credit note** when the business was overcharged (supplier owes money back); a **debit note** when the business was undercharged (owes the supplier more). The suggested amount defaults to the system-calculated variance and is editable, since real negotiations rarely land on the exact calculated number
  - **Escalate** - hand the line to the **Accounts Manager** for a second opinion, with a note on why. Single fixed destination, not a pick-list - see paragraph 6.9 on why
- A **mandatory short note** on every resolution, and a full **audit trail** (who, what, when, before/after values)
- **Status** at both the line and GRN level, so a reviewer can tell partial progress from full resolution at a glance
- A **read-only view** for the operator to see what happened to a mismatch they flagged

### Explicitly out of scope, and why

| Cut | Why |
|---|---|
| **Supplier communication** | Per the brief, all supplier-facing conversation stays exactly as it is today. This feature ends at "we decided X" - it doesn't send anything to the supplier. |
| **Generating financial documents** | "Record credit note" captures intent and amount, not an accounting-system credit memo. Doing that correctly means integrating with whatever AP system each customer uses - a separate, larger project. This version gives finance a clean record to act on manually. |
| **Multi-step approval chains** | Almost every customer starts with a single reviewer tier. If variance-size-based routing turns out to matter, it's a contained addition on top of this model, not a redesign. |
| **Bulk resolution** | Resolving many GRNs in one action is a real efficiency want at high volume, but it multiplies audit-trail and permission edge cases. Shipping single-item resolution first tells us if bulk actions are worth the complexity. |
| **Per-SKU/per-supplier tolerance** | Some categories genuinely warrant tighter or looser thresholds, but that's config surface with no evidence yet that customers need that granularity. |
| **Operator-side resolution** | Resolution authority stays with the reviewer, even for small variances - see paragraph 4. If reviewers end up rubber-stamping trivial cases, raising the tolerance is the first lever, not delegating approval down. |

### Edge cases handled

| Case | Handling |
|---|---|
| Under-delivery (received < confirmed qty) | Flagged if outside tolerance; resolved via **Settle at received quantity** - the corrected total is arithmetic (received qty × confirmed price), so it's calculated and locked, not typed in |
| Over-delivery (received > confirmed qty) | Flagged the same way - **paying for unordered stock is its own liability**, not free upside. Two valid outcomes, both auto-calculated: **Settle at received quantity** (keep the extra, pay for it) or **Return surplus** (pay only for the confirmed amount; physical pickup is arranged off-platform, same as any supplier coordination) |
| Price higher than confirmed | Flagged; resolved via **Record debit/credit note** as a **credit note** - the business was overcharged, supplier owes the difference |
| Price lower than confirmed | Flagged; resolved via **Record debit/credit note** as a **debit note** if the business genuinely owes more (e.g. supplier corrects a stale low price) - still requires a decision, it's just not automatic |
| Both qty **and** price off on the same line | Both variances shown together; the quantity dimension can still be auto-settled, and any remaining price difference on the kept quantity goes through **Record debit/credit note** - **one resolution action covers the whole line's price dimension**, it doesn't need to be split into two separate decisions |
| Multiple mismatched lines, one GRN, different resolutions needed | Each line resolved independently; GRN status reflects the mix ("2 of 5 lines resolved") |
| Mismatch within tolerance | No flag, no queue entry - variance still stored on the GRN for reporting, just doesn't demand action |
| GRN with no mismatch | No change from today's behavior |
| Escalation target unavailable/leaves | **Out of scope for v1** - escalation is a flat reassignment, not a managed workflow with SLAs. Worth revisiting if escalated items visibly stall. |
| Split delivery (partial now, remainder later) | **Out of scope** - this version assumes one GRN reflects the final state of a delivery. Split/partial deliveries are a receiving-flow question, not a mismatch-resolution one. |

---

## 6. Solution

### 6.1 Detection & severity

At GRN submission, each line's received qty/price is compared to the PO's confirmed values. If either falls outside the tolerance band, the line is marked `mismatch_status = open`.

**Every flagged line also gets a severity tier - Small, Medium, or Critical - based on dollar impact**, not just percentage, because a small percentage on a large order can matter more than a large percentage on a $2 item. The queue defaults to sorting by severity, then age, so **the biggest, longest-waiting problems surface automatically** instead of getting lost in a chronological list. (Thresholds are placeholders, tuned per customer the same way tolerance is.)

### 6.2 Notifications

**A queue nobody knows to check is the same failure mode as no queue at all** - so this isn't just a page that exists, it's a page reviewers are pushed toward:
- The reviewer gets a **notification** (and a live badge count) the moment a line is flagged or escalated to them.
- The operator gets a **notification when a mismatch they flagged is resolved**, so they're not left wondering, or checking back out of habit.

### 6.3 Mismatch Queue

A screen scoped to whatever the reviewer has visibility over (branches/suppliers, per existing permissions). Each row: supplier, branch, GRN date, lines flagged, **severity**, $ variance, age, status. Sortable/filterable - because the two things a reviewer triages by are **how much money, and how long it's been sitting.**

### 6.4 Review view

Opening a GRN shows, per mismatched line: item, confirmed qty/price, received qty/price, variance (% and $), and any operator note. **This is a comparison, not a form** - the reviewer's job here is to understand what happened before deciding what to do about it.

### 6.5 Resolution

Each line has its own action, and **which actions are even offered depends on what kind of variance the line actually has** - a quantity-only variance never shows a manual dollar field, a price-only variance never shows a quantity settlement, and over-delivery is the only case offering two valid outcomes (keep vs. return). This isn't just UI tidiness: it's a direct answer to "couldn't a reviewer just type in whatever number they want and call it fraud or error?" For the two auto-calculated actions (Settle at received quantity, Return surplus), **the amount is arithmetic, not a decision** - there's nothing to manually enter, so there's nothing to manually get wrong or fake. The only place a human-entered dollar amount exists at all is the debit/credit note, because that's the one case with genuine negotiated judgment in it.

That doesn't eliminate risk on the debit/credit note entirely - a reviewer could still mis-enter or misrepresent that number. What the design does instead: the field is pre-filled with the system-calculated variance as an anchor (not a blank slate), and **a note is always required, even when the reviewer isn't changing anything** (e.g. accepting a shortfall outright), because the "why" is the part that isn't obvious from the numbers alone - and it's exactly what gets skipped under time pressure if it's optional, which is precisely when it matters most (a real dispute, months later). The honest framing: this doesn't make fraud impossible - no v1 workflow feature does that without real financial controls (approval thresholds, reconciliation against the actual supplier-issued document). It moves the business from **zero traceability today** (an unrecorded WhatsApp decision) to **full attribution** (who, what, when, why, permanently) for the one action that still requires a human number.

### 6.6 Trace

Every resolution appends an entry to the GRN's activity log: actor, timestamp, action, note, before/after values. **This is what turns "someone decided something once" into a record that survives staff turnover** and holds up if a supplier disputes a payment later.

### 6.7 Status model

- **Line status:** `Open` → `Resolved` (terminal), or `Open` → `Escalated` → `Resolved`.
- **GRN status is derived, not stored independently:** `No mismatch` / `Needs review` (≥1 open line) / `Escalated` (no open lines, ≥1 escalated) / `Resolved` (all lines resolved). **A GRN can't look "done" while a line is quietly still open.**

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
Reviewer notified (paragraph 6.2)
      │
      ▼
Reviewer opens the GRN, compares confirmed vs. received side by side
      │
      ▼
Reviewer resolves the line - the options offered depend on the variance type:
  [ Settle at received qty ]   [ Return surplus* ]   [ Record debit/credit note** ]   [ Escalate ]
   *only offered on over-delivery      **only offered when price is involved
      │
      ▼
Resolution logged to the audit trail: who, what, why, when
Operator notified
```

### 6.9 Key decisions and why

- **Resolve at the line level, not the GRN level.** A trivial rounding issue on one item shouldn't force the same resolution as a real shortfall on another. This also keeps partial progress visible instead of blocking on the hardest line.
- **Auto-calculate and lock the amount for pure quantity variances, instead of a manual dollar field.** The corrected total (received or confirmed qty × the already-agreed price) is arithmetic, not judgment - there's no reason to expose a typeable number that could be mis-entered or gamed when the system already knows the right answer. This is a direct, structural answer to the fraud/error risk of free-text financial fields, not just a policy against it.
- **Consolidate price corrections into one "debit/credit note" action, in both directions.** A credit note when the business was overcharged, a debit note when it was undercharged - using the accounting terms an accounts team would actually expect, instead of a generic "adjust price" action that only implicitly assumed one direction.
- **Mandatory note on every action, including auto-calculated ones.** A resolution's "why" is worthless if it's optional - see paragraph 6.5.
- **The debit/credit note captures intent, not the accounting artifact.** Keeps this feature inside Supy's boundary instead of turning it into an integrations project - see paragraph 5.
- **A global tolerance and severity scale, not per-item.** More correct in theory, but config surface with no evidence yet that customers need that granularity. Shipping the simple version first is how we find out.
- **Escalate always routes to one fixed destination (the Accounts Manager), not a pick-list.** v1 is deliberately a single-tier, flat handoff (not an approval chain) - offering a choice of escalation targets would imply a routing decision that contradicts that design. If real usage shows escalations need to go to different people for different reasons, that's a contained addition on top of this model, not a redesign.

---

## 7. How we'd know it's working

### Product (adoption & speed)
- **Share of GRN mismatches reaching a terminal state inside the platform**, trending toward ~100% within a few weeks of rollout. If reviewers are still resolving things over WhatsApp and just not touching the queue, this stays flat - the failure mode to watch for.
- **Median time from a line being flagged to being resolved.** Watch the trend, not just the number - a high starting median that comes down over the first month tells a different story than one that stays high.
- **% of resolutions with a substantive note** (not "ok" or "-"). A soft signal, but a big share of low-effort notes means the mandatory-note decision isn't achieving its purpose.

### Business (the number that justifies the feature)
- **Margin Recovered / Financial Value of Resolutions: total $ of credit notes issued through the platform, per month, per customer.** Deliberately scoped to **credit notes only** - settling at received or confirmed quantity is correct arithmetic, not a negotiated win, and a debit note is money going out, not back. Counting those would inflate the number with routine corrections and undercut its credibility; keeping it to genuine credit notes is what makes it a defensible, revenue-adjacent metric to show a restaurant buyer - it's money back in their pocket, not workflow polish being relabeled as savings.
- **Average $ recovered per resolved mismatch** - gives customer success a concrete ROI number per account.

### Operational (is the machine calibrated correctly)
- **Backlog: count of lines open more than 7 days.** A queue that never empties means it exists but isn't being worked - a different problem than "reviewers are slow."
- **% of GRN lines auto-flagged vs. total lines with any variance.** Too high, and tolerance is too tight - reviewers drown in noise and start ignoring the queue. Too low with continued email complaints, and it's too loose.
- **Severity mix of the queue (e.g., % Critical).** A leading indicator that severity/tolerance thresholds need recalibrating, before it shows up as reviewer fatigue.

**Leading indicator before there's volume:** in the first two weeks, whether reviewers open the queue at all without being prompted. If nobody's checking it unprompted, the notification path (paragraph 6.2) - not the resolution flow itself - is the thing to fix first.

---

## 8. Open questions for the team

- Does "reviewer" map to an existing Supy role/permission today, or do we need a new permission bit? (Assumed yes, per paragraph 3 - needs confirming with the platform/permissions owner.)
- What's a sane default tolerance and severity scale (placeholders used throughout), and should either differ by customer vertical (restaurant vs. central kitchen)?
- What channel should the notifications in paragraph 6.2 ride on - existing in-app notifications, email, or both? (Assumed in-app for v1, pending what Supy already has.)
