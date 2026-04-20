---
name: Reviewer Voting System Design
description: Multi-reviewer yes/no/maybe voting on all applications, displayed per-reviewer in evaluation panel
type: specification
---

# Reviewer Voting System

## Overview

Add a voting feature that allows all reviewers to vote **yes/no/maybe independently on every application**. Votes sit alongside the existing 5-point scoring system and are displayed in the evaluation panel with per-reviewer attribution. Votes sync live via Supabase.

**Purpose:** Finalize the fellowship cohort by collecting consensus votes from all reviewers on applications stuck in "in review" status.

---

## Requirements

### Functional

- Every reviewer can vote independently on all 38+ applications
- Each reviewer's vote is **yes**, **maybe**, **no**, or **empty** (not voted)
- Votes are optional — a reviewer can skip voting on an application
- Votes sync live across all open browser tabs (Supabase realtime)
- Voting system operates **independently** of the existing 5-point scoring system
- Vote tallies appear only when an application detail panel is opened, not in the main table
- Each reviewer can see all reviewers' votes but can only modify their own

### Non-Functional

- Votes persist to Supabase
- No data migration required — new votes table is empty at launch
- Performance: loading and rendering votes should not noticeably slow down the app

---

## Data Model

### Votes Table (Supabase)

```sql
CREATE TABLE votes (
  applicant_id TEXT NOT NULL,
  reviewer_slot INT NOT NULL CHECK (reviewer_slot >= 1 AND reviewer_slot <= 4),
  vote TEXT CHECK (vote IN ('yes', 'maybe', 'no') OR vote IS NULL),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (applicant_id, reviewer_slot)
);
```

**Columns:**
- `applicant_id`: TEXT, foreign key to applicant ID (1–38 as string)
- `reviewer_slot`: INT, which reviewer (1–4, corresponding to `reviewers.slot`)
- `vote`: TEXT, one of 'yes', 'maybe', 'no', or NULL for not voted
- `updated_at`: TIMESTAMPTZ, when the vote was last changed

**RLS:** Same as existing tables — permissive `allow all` for anon key.

**Realtime:** Add to `supabase_realtime` publication so all open browsers get live updates.

---

## UI/UX Design

### Location

In the full-screen evaluation panel (right column, `panel-eval`), **at the bottom**, after all score dimensions and existing evaluation summary.

### Layout

New section titled **"Reviewer Votes"** containing one row per reviewer:

```
┌──────────────────────────────────────┐
│ Reviewer Votes                       │
├──────────────────────────────────────┤
│ You                        [Yes]     │
├──────────────────────────────────────┤
│ Jane Doe              [Maybe]        │
│ Mark Smith              [Yes]        │
│ Sarah Johnson             [—]        │
└──────────────────────────────────────┘
```

**Row Structure:**
- **Reviewer name** (left): "You" for current reviewer, or the reviewer's name from the `reviewers` table
- **Vote label/button** (right): Shows the reviewer's current vote
  - For **your row**: clickable buttons — [Yes] [Maybe] [No]
  - For **other reviewers' rows**: read-only vote display (greyed out, non-clickable)

### Vote Display

- **Your row:** Three buttons (yes, maybe, no) styled like score buttons. Clicked button is highlighted (full color). None highlighted = you haven't voted yet.
- **Other reviewers' rows:** Vote shown as colored label or button (green for Yes, orange for Maybe, red for No). Non-clickable.
- **Empty vote:** Shows "—" (dash) in light grey, non-clickable.

### Interaction

1. **Voting:** Click a vote button in your row to vote. Vote saves immediately to Supabase.
2. **Changing vote:** Click a different button to change your vote. Previous vote is replaced.
3. **Clearing vote:** (Design decision: not yet required; a user can click another option but cannot actively "unvote". If needed later, add a clear/none button.)
4. **Real-time updates:** When another reviewer votes, their row updates live in the panel.

---

## JavaScript Implementation

### Data Flow

**Initialization:**
1. When `init()` runs, `loadFromDb()` fetches all votes from the `votes` table into a global object: `let VOTES = {}` (keyed by `applicant_id`, value is map of `slot -> vote`).
2. `subscribeRealtime()` adds a listener on the `votes` table channel. When votes change, update `VOTES` and call `renderVotesSection(currentApplicantId)`.

**Rendering:**
1. When `openPanel(applicantId)` opens an application, call `renderVotesSection(applicantId)` to render the votes section at the bottom of `panel-eval`.
2. `renderVotesSection(applicantId)` iterates through all reviewers (from `reviewers` table), looks up their vote in `VOTES[applicantId]`, and renders a row per reviewer.

**Saving:**
1. When user clicks a vote button, call `saveVoteToDb(applicantId, reviewerSlot, vote)`.
2. `saveVoteToDb()` upserts the row to Supabase: `INSERT INTO votes (applicant_id, reviewer_slot, vote) VALUES (...) ON CONFLICT (applicant_id, reviewer_slot) DO UPDATE SET vote = ...`
3. Supabase realtime notifies all clients, triggering a re-render via the subscription listener.

### New Functions

| Function | Purpose | Input | Output |
|---|---|---|---|
| `renderVotesSection(applicantId)` | Render the "Reviewer Votes" section in the eval panel | applicant ID | HTML string (or return element) |
| `saveVoteToDb(applicantId, reviewerSlot, vote)` | Upsert vote to Supabase | applicant ID, reviewer slot (1–4), vote ('yes'\|'maybe'\|'no') | Promise (resolves when saved) |
| `initVotes()` | Load all votes from `votes` table into `VOTES` global | none | Promise |

### Modified Functions

| Function | Change |
|---|---|
| `loadFromDb()` | Call `initVotes()` after fetching evaluations |
| `subscribeRealtime()` | Add listener for `votes` table channel; on change, re-render current votes section if a panel is open |
| `openPanel(applicantId)` | After rendering score blocks, call `renderVotesSection(applicantId)` to append votes section to `panel-eval` |

---

## Data Integrity

- **Vote uniqueness:** The primary key `(applicant_id, reviewer_slot)` ensures each reviewer can vote once per application.
- **Vote validity:** The `CHECK` constraint ensures vote is one of the three valid options or NULL.
- **Reviewer slot consistency:** `reviewer_slot` must match a row in the `reviewers` table (enforced by app logic; no foreign key constraint needed on free tier).

---

## Edge Cases & Constraints

1. **No reviewer assigned to a slot:** If a reviewer slot is empty (no name entered), that slot doesn't appear in the votes section.
2. **Reviewer removed:** If a reviewer is deleted (via "remove reviewer" button), their votes remain in the `votes` table but won't display (since no reviewer row exists). This is acceptable.
3. **Concurrent votes:** Two reviewers voting simultaneously should work fine — Supabase handles concurrency, and realtime will notify both clients.
4. **Vote history:** This design does not track vote history (changes over time). If needed, an audit log can be added later.

---

## Testing Strategy

- **Manual:** Open the app with multiple reviewers logged in simultaneously. Vote on an application and verify the vote appears live in other tabs.
- **Vote persistence:** Refresh the page and verify votes are loaded correctly.
- **Vote changes:** Change your vote and verify it updates immediately.
- **Empty votes:** Verify empty votes display as "—".

---

## Rollout

1. Create `votes` table in Supabase with RLS and realtime enabled.
2. Implement JavaScript functions (render, save, subscribe).
3. Sync `review-app.html` → `index.html`.
4. Push to GitHub (`main` branch).
5. Verify on live URL within ~60 seconds.

---

## Future Enhancements (Out of Scope)

- Vote tally summary in the main applications table
- Vote history/audit log
- Consensus metrics (e.g., "2/4 agree on Yes")
- "Clear vote" / "Unvote" button
- Vote statistics across all applications
