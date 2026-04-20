---
name: Reviewer Comments Feature Design
description: Optional comments on votes explaining reviewer reasoning, editable by author, visible to all
type: specification
---

# Reviewer Comments Feature

## Overview

Add optional comments to the voting system. Each reviewer can comment on their vote, explaining their reasoning. Comments are editable (if the reviewer changes their mind or vote), and visible to all reviewers but only editable by the author.

---

## Requirements

### Functional

- Every reviewer can add an optional comment to each of their votes
- Comments are tied to a specific vote ('yes', 'maybe', 'no')
- Each reviewer can only edit their own comments
- All reviewers can view all comments (read-only)
- When a reviewer changes their vote, their old comment becomes editable (not deleted)
- Comments sync live across all open browser tabs (Supabase realtime)
- Comment textbox appears only when a reviewer has voted on that application

### Non-Functional

- Comments persist to Supabase
- No data migration required — new comments table is empty at launch
- Performance: loading and rendering comments should not noticeably slow down the app

---

## Data Model

### Comments Table (Supabase)

```sql
CREATE TABLE comments (
  applicant_id TEXT NOT NULL,
  reviewer_slot INT NOT NULL CHECK (reviewer_slot >= 1 AND reviewer_slot <= 4),
  vote TEXT NOT NULL CHECK (vote IN ('yes', 'maybe', 'no')),
  comment_text TEXT,
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (applicant_id, reviewer_slot, vote)
);
```

**Columns:**
- `applicant_id`: TEXT, foreign key to applicant ID (1–38 as string)
- `reviewer_slot`: INT, which reviewer (1–4, corresponding to `reviewers.slot`)
- `vote`: TEXT, which vote this comment belongs to ('yes', 'maybe', 'no')
- `comment_text`: TEXT, the comment content (can be NULL or empty string)
- `updated_at`: TIMESTAMPTZ, when last updated

**RLS:** Same as existing tables — permissive `allow all` for anon key.

**Realtime:** Add to `supabase_realtime` publication so all open browsers get live updates.

---

## UI/UX Design

### Location

In the "Reviewer Votes" section (at the bottom of the eval panel), integrated with each reviewer row.

### Layout per Reviewer Row

**Your row (you have voted):**
```
┌──────────────────────────────────────┐
│ You                        [Yes]     │
│ ┌──────────────────────────────────┐ │
│ │ Add comment (optional)           │ │
│ │ [Textbox for your comment]       │ │
│ └──────────────────────────────────┘ │
└──────────────────────────────────────┘
```

**Your row (you have not voted):**
- No textbox visible; vote buttons only.

**Other reviewer's row (they have voted):**
```
┌──────────────────────────────────────┐
│ Jane Doe                  [Yes]       │
│ Comment: "Strong product-market fit" │
└──────────────────────────────────────┘
```

**Other reviewer's row (they changed their vote, old comment exists):**
```
┌──────────────────────────────────────┐
│ Mark Smith               [Maybe]      │
│ Previous vote: No                     │
│ ┌──────────────────────────────────┐ │
│ │ "Needs more evidence"            │ │
│ │ [Editable by Mark]               │ │
│ └──────────────────────────────────┘ │
│ Current vote: Maybe                  │
│ Comment: [empty]                     │
└──────────────────────────────────────┘
```

### Interaction

1. **Adding a comment:** Type in the textbox under your vote. Saves automatically on blur or Enter.
2. **Editing your comment:** Click/focus the textbox, edit, and it saves automatically.
3. **Viewing others' comments:** Read-only; no interaction.
4. **Changing your vote:** Old comment becomes editable for you; new vote shows empty textbox for a new comment.

---

## JavaScript Implementation

### Data Flow

**Initialization:**
1. When `init()` runs, `initComments()` fetches all comments from the `comments` table into a global object: `let COMMENTS = {}` (keyed by `applicant_id`, value is map of `slot_vote -> comment_text`).
2. `subscribeRealtime()` adds a listener on the `comments` table channel. When comments change, update `COMMENTS` and call `renderCommentsSection(currentApplicantId)`.

**Rendering:**
1. When `openPanel(applicantId)` opens an application, `renderVotesSection(applicantId)` is called, which then calls `renderCommentsSection(applicantId)` to render comment textboxes and read-only comments for each reviewer.
2. `renderCommentsSection(applicantId)` iterates through all reviewers, looks up their vote and comment in `COMMENTS[applicantId]`, and renders a comment UI per reviewer.

**Saving:**
1. When user types in a comment textbox and blurs or presses Enter, call `saveCommentToDb(applicantId, reviewerSlot, vote, commentText)`.
2. `saveCommentToDb()` upserts the row to Supabase: `INSERT INTO comments (applicant_id, reviewer_slot, vote, comment_text) VALUES (...) ON CONFLICT (applicant_id, reviewer_slot, vote) DO UPDATE SET comment_text = ..., updated_at = NOW()`
3. Supabase realtime notifies all clients, triggering a re-render via the subscription listener.

### New Functions

| Function | Purpose | Input | Output |
|---|---|---|---|
| `initComments()` | Load all comments from `comments` table into `COMMENTS` global | none | Promise |
| `renderCommentsSection(applicantId)` | Render comment textboxes and read-only comments integrated into votes section | applicant ID | (modifies DOM in votes section) |
| `saveCommentToDb(applicantId, reviewerSlot, vote, commentText)` | Upsert comment to Supabase | applicant ID, reviewer slot (1–4), vote ('yes'\|'maybe'\|'no'), comment text | Promise |

### Modified Functions

| Function | Change |
|---|---|
| `loadFromDb()` | Call `initComments()` after fetching evaluations and votes |
| `subscribeRealtime()` | Add listener for `comments` table channel; on change, re-render current comments section if a panel is open |
| `renderVotesSection(applicantId)` | After rendering vote rows, call `renderCommentsSection(applicantId)` to add comments UI |

---

## Data Integrity

- **Comment uniqueness:** The primary key `(applicant_id, reviewer_slot, vote)` ensures each reviewer has at most one comment per vote per application.
- **Vote validity:** Comments are only tied to valid votes ('yes', 'maybe', 'no').
- **Reviewer slot consistency:** `reviewer_slot` must match a row in the `reviewers` table (enforced by app logic).

---

## Edge Cases & Constraints

1. **No comment entered:** Empty string or NULL in DB — treated as "no comment" and displays as blank.
2. **Vote changed, old comment exists:** Old comment row stays in DB tied to old vote; appears editable to author, read-only to others.
3. **Comment cleared by user:** Save empty string or NULL to DB; old comment row remains but displays as empty.
4. **Reviewer removed:** If a reviewer is deleted, their comments remain in the `comments` table but won't display (since no reviewer row exists). This is acceptable.
5. **Concurrent comments:** Two reviewers commenting simultaneously should work fine — Supabase handles concurrency, and realtime will notify both clients.

---

## Testing Strategy

- **Manual:** Open the app with multiple reviewers logged in simultaneously. Vote on an application, add a comment, verify the comment appears live in other tabs.
- **Comment persistence:** Refresh the page and verify comments are loaded correctly.
- **Comment editing:** Edit your comment and verify it updates immediately.
- **Vote changes:** Change your vote and verify old comment becomes editable, new vote shows empty textbox.
- **Empty comments:** Verify empty comments display as blank.

---

## Rollout

1. Create `comments` table in Supabase with RLS and realtime enabled.
2. Implement JavaScript functions (render, save, subscribe).
3. Sync `review-app.html` → `index.html`.
4. Push to GitHub (`main` branch).
5. Verify on live URL within ~60 seconds.

---

## Future Enhancements (Out of Scope)

- Comment count or summary in main applications table
- Comment history/edit log
- Timestamp display on comments
- Comment threading or replies
- Rich text formatting in comments
