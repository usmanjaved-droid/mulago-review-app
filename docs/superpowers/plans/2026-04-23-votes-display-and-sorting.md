# Votes Display & Smart Sorting Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Display vote counts in the main applications table and implement intelligent sorting that prioritizes shortlisted apps, then sorts by Yes votes descending, with proper tie-breaking.

**Architecture:** Add a helper function to count votes per applicant, insert a votes column into the table HTML, and replace the current sort logic in `getFilteredSorted()` with a new priority-based comparator that respects status order first, then applies secondary sorts by vote counts.

**Tech Stack:** Pure HTML/JS, no build step. Votes data already in Supabase `VOTES` global state object.

---

## File Structure

- **Modify:** `review-app.html` — add helper function, update table headers, modify sorting logic, update row rendering

---

## Task 1: Create Vote Counting Helper Function

**Files:**
- Modify: `review-app.html` (insert new function before `getFilteredSorted()`)

- [ ] **Step 1: Locate insertion point**

Open `review-app.html` and find line ~2806 where `getFilteredSorted()` is defined. We'll insert the new helper function right before it.

- [ ] **Step 2: Add `getVoteCounts()` helper function**

Insert this code at line 2806 (before `function getFilteredSorted()`):

```javascript
function getVoteCounts(applicantId) {
  const votes = VOTES[applicantId] || {};
  const yes = Object.values(votes).filter(v => v === 'yes').length;
  const maybe = Object.values(votes).filter(v => v === 'maybe').length;
  const no = Object.values(votes).filter(v => v === 'no').length;
  return { yes, maybe, no, total: yes + maybe + no };
}
```

- [ ] **Step 3: Test the function exists**

Open browser console at http://localhost:3000, run:
```javascript
console.log(getVoteCounts(1));
```

Expected: Returns an object like `{ yes: 2, maybe: 1, no: 0, total: 3 }` (or empty if no votes).

- [ ] **Step 4: Commit**

```bash
cd "c:\Users\musma\Documents\Mulago Review App\mulago-review-app"
git add review-app.html
git commit -m "feat: add getVoteCounts() helper to count yes/maybe/no votes per applicant"
```

---

## Task 2: Add Votes Column Header to Table

**Files:**
- Modify: `review-app.html:725-726` (table thead)

- [ ] **Step 1: Locate the Score and Status headers**

Find lines 725-726:
```html
<th>Score</th>
<th>Status</th>
```

- [ ] **Step 2: Insert Votes column header**

Replace those lines with:
```html
<th>Score</th>
<th>Votes</th>
<th>Status</th>
```

The votes column now appears between Score and Status.

- [ ] **Step 3: Verify in browser**

Reload http://localhost:3000. The table header should now show: Applicant | Sector | Budget | Team | Score | **Votes** | Status | [Assigned To]

- [ ] **Step 4: Commit**

```bash
git add review-app.html
git commit -m "feat: add Votes column header to applications table"
```

---

## Task 3: Create Vote Rendering Helper Function

**Files:**
- Modify: `review-app.html` (insert new function before `renderTable()`)

- [ ] **Step 1: Locate insertion point**

Find line ~2846 where `function renderTable()` is defined. We'll insert the new helper right before it.

- [ ] **Step 2: Add `renderVotesCell()` helper function**

Insert this code at line 2846 (before `function renderTable()`):

```javascript
function renderVotesCell(applicantId) {
  const counts = getVoteCounts(applicantId);
  if (counts.total === 0) {
    return `<span style="font-size:0.75rem;color:var(--outline-variant);font-style:italic">—</span>`;
  }
  return `<span style="font-size:0.75rem;color:var(--on-surface);font-weight:500">✓ ${counts.yes} | ⊙ ${counts.maybe} | ✗ ${counts.no}</span>`;
}
```

- [ ] **Step 3: Test the function**

Open browser console and run:
```javascript
console.log(renderVotesCell(1));
```

Expected: Returns a string like `<span...>✓ 2 | ⊙ 1 | ✗ 0</span>` or the dash if no votes.

- [ ] **Step 4: Commit**

```bash
git add review-app.html
git commit -m "feat: add renderVotesCell() to format vote display"
```

---

## Task 4: Update Table Row Rendering to Include Votes Column

**Files:**
- Modify: `review-app.html:2895-2896` (in the row template inside `renderTable()`)

- [ ] **Step 1: Locate the row template**

Find the `return \`<tr onclick=...` template starting at line 2885, specifically lines 2895-2896:
```javascript
      <td>${scoreCell}</td>
      <td>${renderBadge(ev.status)}</td>
```

- [ ] **Step 2: Insert votes cell between score and status**

Replace those two lines with:
```javascript
      <td>${scoreCell}</td>
      <td>${renderVotesCell(a.id)}</td>
      <td>${renderBadge(ev.status)}</td>
```

- [ ] **Step 3: Verify in browser**

Reload http://localhost:3000. The table should now show vote breakdowns for each applicant (or dashes if no votes yet).

Test by clicking on an app, voting yes/maybe/no, and seeing the votes appear in the table.

- [ ] **Step 4: Commit**

```bash
git add review-app.html
git commit -m "feat: render vote counts in applications table"
```

---

## Task 5: Replace Sorting Logic in `getFilteredSorted()`

**Files:**
- Modify: `review-app.html:2806-2844` (replace entire `getFilteredSorted()` function)

- [ ] **Step 1: Locate the current sorting section**

Find lines 2832-2841 (the sort comparator inside `getFilteredSorted()`):
```javascript
  const sort = currentSort || 'name';
  list.sort((a, b) => {
    if (sort === 'name') return a.name.localeCompare(b.name);
    if (sort === 'org') return a.org.localeCompare(b.org);
    if (sort === 'budget') return b.budget - a.budget;
    if (sort === 'score') return totalScore(b.id).total - totalScore(a.id).total;
    if (sort === 'sector') return a.sector.localeCompare(b.sector);
    if (sort === 'team') return b.team - a.team;
    return a.id - b.id;
  });
```

- [ ] **Step 2: Define the status priority order**

Before the sort logic (after line 2830, after the search filter), insert:
```javascript
  const statusPriority = { 'shortlisted': 0, 'review': 1, 'rejected': 2, 'pending': 3 };
```

- [ ] **Step 3: Replace the sort comparator**

Replace the entire sort block (lines 2832-2841) with:
```javascript
  const sort = currentSort || 'name';
  list.sort((a, b) => {
    const aStatus = getEval(a.id).status || 'pending';
    const bStatus = getEval(b.id).status || 'pending';
    const aPriority = statusPriority[aStatus] ?? 999;
    const bPriority = statusPriority[bStatus] ?? 999;

    // First, sort by status priority
    if (aPriority !== bPriority) {
      return aPriority - bPriority;
    }

    // Within same status, apply user's chosen sort
    if (sort === 'name') return a.name.localeCompare(b.name);
    if (sort === 'org') return a.org.localeCompare(b.org);
    if (sort === 'budget') return b.budget - a.budget;
    if (sort === 'score') return totalScore(b.id).total - totalScore(a.id).total;
    if (sort === 'sector') return a.sector.localeCompare(b.sector);
    if (sort === 'team') return b.team - a.team;

    // Default sort: by votes (Yes desc, Maybe desc, then ID asc)
    const aVotes = getVoteCounts(a.id);
    const bVotes = getVoteCounts(b.id);
    if (aVotes.yes !== bVotes.yes) return bVotes.yes - aVotes.yes;
    if (aVotes.maybe !== bVotes.maybe) return bVotes.maybe - aVotes.maybe;
    return a.id - b.id;
  });
```

- [ ] **Step 4: Test sorting in browser**

Reload http://localhost:3000. Verify:
- All shortlisted apps appear at the top
- Then in-review apps
- Then rejected apps
- Then pending apps at the bottom
- Within each status group, apps with more "Yes" votes appear higher

Test by changing the `currentSort` in console: `currentSort = 'name'; renderTable();` and verify status priority still overrides.

- [ ] **Step 5: Commit**

```bash
git add review-app.html
git commit -m "feat: implement priority-based sorting (status, votes, id)"
```

---

## Task 6: Sync Changes to index.html

**Files:**
- Copy: `review-app.html` → `index.html`

- [ ] **Step 1: Sync the files**

Per CLAUDE.md instructions, after every edit to `review-app.html`, copy to `index.html`:

```bash
cd "c:\Users\musma\Documents\Mulago Review App\mulago-review-app"
cp "review-app.html" "index.html"
```

- [ ] **Step 2: Verify both files are identical**

```bash
diff review-app.html index.html
```

Expected: No output (files are identical).

- [ ] **Step 3: Commit**

```bash
git add index.html review-app.html
git commit -m "chore: sync index.html with review-app.html"
```

---

## Task 7: Test Full Feature End-to-End

**Files:**
- Test: Browser at http://localhost:3000

- [ ] **Step 1: Start dev server**

```bash
cd "c:\Users\musma\Documents\Mulago Review App\mulago-review-app"
npx serve -l 3000 .
```

- [ ] **Step 2: Open app and verify initial state**

Navigate to http://localhost:3000. The table should display:
- Votes column header between Score and Status
- All apps showing "—" in the Votes column (no votes yet)
- Apps ordered by status (shortlisted > review > rejected > pending)

- [ ] **Step 3: Add votes and verify sorting updates**

1. Open an app in "pending" status
2. Vote "yes" on it
3. Close the panel and check: the app should move up in the list
4. Open another pending app and vote "maybe"
5. Verify the first app (with "yes" vote) appears above it

- [ ] **Step 4: Test tie-breaking**

1. Give two "pending" apps the same number of "yes" votes
2. Give one of them a "maybe" vote
3. Verify the one with "maybe" vote appears below (or if they have 0 maybe votes, verify they're sorted by ID)

- [ ] **Step 5: Test status priority override**

1. Move a pending app to "shortlisted"
2. Verify it immediately jumps to the top of the list, regardless of vote count

- [ ] **Step 6: Commit final state**

```bash
git add index.html review-app.html
git commit -m "feat: complete votes display and smart sorting implementation"
```

---

## Self-Review Checklist

✅ **Spec coverage:**
- Vote breakdown display (Yes/Maybe/No) in main table — Task 2, 3, 4
- Vote counting helper function — Task 1
- Priority sorting (Shortlisted > Review > Rejected > Pending) — Task 5
- Secondary sorts (Yes votes desc, Maybe votes desc, ID asc) — Task 5
- Votes column inserted into table — Task 4
- Files synced — Task 6

✅ **No placeholders:** All code blocks complete and runnable.

✅ **Type consistency:** 
- `getVoteCounts()` returns `{ yes, maybe, no, total }` consistently
- `renderVotesCell()` uses the same object structure
- `statusPriority` mapping is clear and complete

✅ **Architecture:** Helper functions isolated, sorting logic self-contained, no dependencies on external libraries.

---

## Execution

Plan complete and saved to this file. Two execution options:

**1. Subagent-Driven (recommended)** — Fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** — Execute all tasks in this session using executing-plans, batch execution with checkpoints

Which approach would you prefer?
