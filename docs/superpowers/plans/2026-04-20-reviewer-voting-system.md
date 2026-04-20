# Reviewer Voting System Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a yes/no/maybe voting feature allowing all reviewers to independently vote on all applications, with live per-reviewer vote tallies in the evaluation panel.

**Architecture:** Add a new `votes` table to Supabase, extend the app's global state to load and subscribe to votes, add rendering logic for a "Reviewer Votes" section at the bottom of the eval panel, and wire up vote buttons to save votes and trigger live updates.

**Tech Stack:** Supabase (votes table, realtime), pure HTML/CSS/JS (no build step).

---

## File Structure

**Modified:**
- `review-app.html` — all changes to this file (single-file app)

**No new files** — all logic stays in `review-app.html`; after editing, sync to `index.html`.

---

## Task 1: Create Votes Table in Supabase

**Files:**
- Supabase Console: Create table `votes`

- [ ] **Step 1: Open Supabase Console**

Navigate to https://wzvvylqxibsghpkmkfka.supabase.co in your browser. Log in if needed.

- [ ] **Step 2: Create the votes table**

In the SQL Editor, run:

```sql
CREATE TABLE votes (
  applicant_id TEXT NOT NULL,
  reviewer_slot INT NOT NULL CHECK (reviewer_slot >= 1 AND reviewer_slot <= 4),
  vote TEXT CHECK (vote IN ('yes', 'maybe', 'no') OR vote IS NULL),
  updated_at TIMESTAMPTZ DEFAULT NOW(),
  PRIMARY KEY (applicant_id, reviewer_slot)
);
```

Expected: Table created successfully.

- [ ] **Step 3: Enable RLS**

In the Supabase Console, go to **Tables > votes > RLS**. 

Click **Enable RLS** if not already enabled.

Then click **New Policy** and add a permissive `allow all` policy:
- Policy name: `Allow all`
- SELECT: ✓
- INSERT: ✓
- UPDATE: ✓
- DELETE: ✓
- WITH CHECK: (leave blank)

Expected: Policy created.

- [ ] **Step 4: Add votes to realtime publication**

In the Supabase Console, go to **Database > Publications > supabase_realtime**.

Click **Edit** and toggle **votes** table to **ON**.

Expected: votes table now publishes realtime events.

---

## Task 2: Add Global Vote State

**Files:**
- Modify: `review-app.html` (add global state variable after `APPLICANTS` const)

- [ ] **Step 1: Add VOTES global variable**

In `review-app.html`, find the line with `const APPLICANTS = [...]`. After that block (after the closing `];`), add:

```javascript
// Global vote state: { applicant_id: { slot: 'yes'|'maybe'|'no'|null } }
let VOTES = {};
```

Expected: Global variable added. No visual change yet.

- [ ] **Step 2: Commit**

```bash
cd "c:\Users\musma\Documents\Mulago Review App\mulago-review-app"
git add review-app.html
git commit -m "feat: add global VOTES state object"
```

---

## Task 3: Implement initVotes() Function

**Files:**
- Modify: `review-app.html` (add new function in the JS section, before `init()`)

- [ ] **Step 1: Write initVotes() function**

In `review-app.html`, find the `async function init()` declaration. Before it, add:

```javascript
async function initVotes() {
  try {
    const { data, error } = await supabase
      .from('votes')
      .select('*');
    
    if (error) throw error;
    
    // Build VOTES object: { applicant_id: { slot: vote } }
    VOTES = {};
    data.forEach(row => {
      if (!VOTES[row.applicant_id]) {
        VOTES[row.applicant_id] = {};
      }
      VOTES[row.applicant_id][row.reviewer_slot] = row.vote;
    });
    
    console.log('Loaded votes:', VOTES);
  } catch (err) {
    console.error('Error loading votes:', err.message);
  }
}
```

Expected: Function defined. No errors in console.

- [ ] **Step 2: Commit**

```bash
git add review-app.html
git commit -m "feat: add initVotes() to load votes from Supabase"
```

---

## Task 4: Call initVotes() from loadFromDb()

**Files:**
- Modify: `review-app.html` (update `loadFromDb()` function)

- [ ] **Step 1: Find loadFromDb() function**

In `review-app.html`, locate the `async function loadFromDb()` function.

- [ ] **Step 2: Add initVotes() call**

At the **end** of `loadFromDb()` (after all existing `const { data, error }` blocks but before the function closes), add:

```javascript
await initVotes();
```

The function should now end with:
```javascript
  APPLICANT_LIST = data.map(row => row.id);
  await initVotes();
}
```

Expected: No visual change. Votes load when app initializes.

- [ ] **Step 3: Commit**

```bash
git add review-app.html
git commit -m "feat: call initVotes() in loadFromDb()"
```

---

## Task 5: Implement saveVoteToDb() Function

**Files:**
- Modify: `review-app.html` (add new function after `initVotes()`)

- [ ] **Step 1: Write saveVoteToDb() function**

After the `initVotes()` function, add:

```javascript
async function saveVoteToDb(applicantId, reviewerSlot, vote) {
  try {
    const { error } = await supabase
      .from('votes')
      .upsert({
        applicant_id: applicantId,
        reviewer_slot: reviewerSlot,
        vote: vote,
        updated_at: new Date().toISOString()
      }, {
        onConflict: 'applicant_id,reviewer_slot'
      });
    
    if (error) throw error;
    
    console.log(`Saved vote for app ${applicantId}, slot ${reviewerSlot}: ${vote}`);
  } catch (err) {
    console.error('Error saving vote:', err.message);
  }
}
```

Expected: Function defined. No errors in console.

- [ ] **Step 2: Commit**

```bash
git add review-app.html
git commit -m "feat: add saveVoteToDb() to upsert votes to Supabase"
```

---

## Task 6: Subscribe to Vote Changes in subscribeRealtime()

**Files:**
- Modify: `review-app.html` (update `subscribeRealtime()` function)

- [ ] **Step 1: Find subscribeRealtime() function**

In `review-app.html`, locate `function subscribeRealtime()`.

- [ ] **Step 2: Add votes channel subscription**

At the **end** of the function (before the closing brace), add:

```javascript
// Subscribe to votes table changes
const votesChannel = supabase
  .channel('public:votes')
  .on(
    'postgres_changes',
    { event: '*', schema: 'public', table: 'votes' },
    (payload) => {
      // Update VOTES state
      const { applicant_id, reviewer_slot, vote } = payload.new;
      if (!VOTES[applicant_id]) {
        VOTES[applicant_id] = {};
      }
      VOTES[applicant_id][reviewer_slot] = vote;
      console.log('Vote updated:', applicant_id, reviewer_slot, vote);
      
      // Re-render votes section if a panel is open
      const panelContent = document.getElementById('panel-content');
      if (panelContent) {
        const currentApplicantId = panelContent.getAttribute('data-applicant-id');
        if (currentApplicantId) {
          renderVotesSection(currentApplicantId);
        }
      }
    }
  )
  .subscribe();
```

Expected: Subscription added. No visual change yet.

- [ ] **Step 2: Commit**

```bash
git add review-app.html
git commit -m "feat: subscribe to votes realtime channel for live updates"
```

---

## Task 7: Implement renderVotesSection() Function

**Files:**
- Modify: `review-app.html` (add new function after `saveVoteToDb()`)

- [ ] **Step 1: Write renderVotesSection() function**

After `saveVoteToDb()`, add:

```javascript
function renderVotesSection(applicantId) {
  const votesSection = document.getElementById('votes-section');
  if (!votesSection) return;
  
  const currentSlot = CURRENT_REVIEWER_SLOT;
  if (!currentSlot) return; // No reviewer selected
  
  // Build reviewer rows
  let html = '<div style="margin-top: 1.5rem;">';
  html += '<div style="font-size: 0.75rem; color: var(--on-surface-variant); text-transform: uppercase; font-weight: 700; margin-bottom: 0.75rem;">Reviewer Votes</div>';
  
  // Get all reviewers from REVIEWERS array
  const reviewerRows = REVIEWERS.filter(r => r.name); // Only show if name is set
  
  reviewerRows.forEach(reviewer => {
    const isCurrentReviewer = reviewer.slot === currentSlot;
    const voteValue = VOTES[applicantId] ? VOTES[applicantId][reviewer.slot] : null;
    const reviewerName = isCurrentReviewer ? 'You' : reviewer.name;
    
    html += '<div style="display: flex; align-items: center; justify-content: space-between; padding: 0.625rem; background: var(--surface-container-low); border-radius: 0.375rem; margin-bottom: 0.5rem;">';
    html += `<div style="font-size: 0.875rem; font-weight: 600; color: var(--on-surface);">${reviewerName}</div>`;
    
    if (isCurrentReviewer) {
      // Current reviewer: show clickable buttons
      html += '<div style="display: flex; gap: 0.375rem;">';
      const votes = ['yes', 'maybe', 'no'];
      const colors = { yes: '#4caf50', maybe: '#ff9800', no: '#f44336' };
      const labels = { yes: 'Yes', maybe: 'Maybe', no: 'No' };
      
      votes.forEach(v => {
        const isActive = voteValue === v;
        const bgColor = isActive ? colors[v] : 'white';
        const textColor = isActive ? 'white' : colors[v];
        const borderColor = colors[v];
        
        html += `<button onclick="handleVoteClick('${applicantId}', ${reviewer.slot}, '${v}')" style="padding: 0.375rem 0.75rem; border: 1px solid ${borderColor}; background: ${bgColor}; color: ${textColor}; border-radius: 0.25rem; cursor: pointer; font-size: 0.75rem; font-weight: 600; flex: 1;">${labels[v]}</button>`;
      });
      html += '</div>';
    } else {
      // Other reviewers: show read-only vote display
      if (voteValue) {
        const colors = { yes: '#4caf50', maybe: '#ff9800', no: '#f44336' };
        const labels = { yes: 'Yes', maybe: 'Maybe', no: 'No' };
        const color = colors[voteValue];
        const label = labels[voteValue];
        html += `<div style="padding: 0.375rem 0.75rem; border-radius: 0.25rem; background: ${color}; color: white; font-size: 0.75rem; font-weight: 600;">${label}</div>`;
      } else {
        html += '<div style="padding: 0.375rem 0.75rem; color: var(--on-surface-variant); font-size: 0.75rem; font-weight: 600;">—</div>';
      }
    }
    
    html += '</div>';
  });
  
  html += '</div>';
  votesSection.innerHTML = html;
}
```

Expected: Function defined. No visual change yet (needs to be called).

- [ ] **Step 2: Commit**

```bash
git add review-app.html
git commit -m "feat: add renderVotesSection() to display reviewer votes"
```

---

## Task 8: Add Vote Handling Function

**Files:**
- Modify: `review-app.html` (add new function after `renderVotesSection()`)

- [ ] **Step 1: Write handleVoteClick() function**

After `renderVotesSection()`, add:

```javascript
async function handleVoteClick(applicantId, reviewerSlot, vote) {
  await saveVoteToDb(applicantId, reviewerSlot, vote);
  
  // Update local state
  if (!VOTES[applicantId]) {
    VOTES[applicantId] = {};
  }
  VOTES[applicantId][reviewerSlot] = vote;
  
  // Re-render votes section
  renderVotesSection(applicantId);
}
```

Expected: Function defined.

- [ ] **Step 2: Commit**

```bash
git add review-app.html
git commit -m "feat: add handleVoteClick() to handle vote button clicks"
```

---

## Task 9: Add Votes Section Container to Eval Panel

**Files:**
- Modify: `review-app.html` (update `openPanel()` function)

- [ ] **Step 1: Find openPanel() function**

In `review-app.html`, locate `function openPanel(applicantId)`.

- [ ] **Step 2: Find where eval panel content is appended**

Within `openPanel()`, find the line where `panel-eval` content is built. Look for lines that append score blocks and eval summary. You're looking for the section that builds the HTML for `panel-eval`.

Look for a pattern like:
```javascript
panelEval.innerHTML = `...`
// followed by appending elements
```

- [ ] **Step 3: Add votes section container**

At the **end** of the `panel-eval` content building (after the eval summary is appended but before the section closes), add:

```javascript
// Add votes section container
const votesContainer = document.createElement('div');
votesContainer.id = 'votes-section';
panelEval.appendChild(votesContainer);
```

Then, after appending the container, call:

```javascript
renderVotesSection(applicantId);
```

Expected: Votes section renders in the panel when opened. Buttons should be clickable.

- [ ] **Step 4: Store applicant ID for vote re-renders**

At the start of `openPanel()`, add:

```javascript
panelContent.setAttribute('data-applicant-id', applicantId);
```

This allows the realtime subscription handler to know which applicant's votes are currently displayed.

Expected: Attribute set on `panel-content` div.

- [ ] **Step 5: Commit**

```bash
git add review-app.html
git commit -m "feat: add votes section to eval panel and render votes"
```

---

## Task 10: Sync and Test Locally

**Files:**
- `review-app.html` (source of truth)
- `index.html` (sync copy)

- [ ] **Step 1: Sync review-app.html to index.html**

```bash
cd "c:\Users\musma\Documents\Mulago Review App\mulago-review-app"
cp "review-app.html" "index.html"
```

Expected: `index.html` updated with all changes.

- [ ] **Step 2: Start dev server**

```bash
npx serve -l 3000 .
```

Expected: Server running on http://localhost:3000

- [ ] **Step 3: Test locally**

1. Open http://localhost:3000 in a browser
2. Enter reviewer names in the ⚙ Reviewers panel (e.g., "Alice", "Bob", "Charlie")
3. Click "Assign & Save"
4. Pick a reviewer name from the dropdown
5. Open any application
6. Scroll to the bottom of the eval panel
7. **Expected:** "Reviewer Votes" section visible with:
   - Your row showing [Yes] [Maybe] [No] buttons
   - Other reviewers' rows showing empty (—) votes
8. Click a vote button for yourself
9. **Expected:** Button highlights, vote saves to Supabase
10. Open the app in a second browser tab (or incognito)
11. Select a different reviewer
12. Open the same application
13. **Expected:** Your vote from the first tab shows in the second tab's panel
14. Click a vote button in the second tab
15. **Expected:** New vote appears live in the first tab

- [ ] **Step 4: Commit sync**

```bash
git add index.html
git commit -m "sync: index.html from review-app.html after voting feature"
```

---

## Task 11: Test Edge Cases

**Files:**
- No file changes; manual testing only

- [ ] **Step 1: Test empty reviewer slots**

1. Remove all reviewers via "Remove" button
2. Open an application
3. **Expected:** Votes section still renders but no reviewer rows appear
4. Add reviewers again and test

- [ ] **Step 2: Test page refresh**

1. Vote on an application with one reviewer
2. Switch to another reviewer
3. Refresh the page
4. Switch back to the first reviewer
5. Open the same application
6. **Expected:** First reviewer's vote is still there (persisted)

- [ ] **Step 3: Test concurrent votes**

1. Open the app in two tabs
2. In Tab 1: select Reviewer A
3. In Tab 2: select Reviewer B
4. In Tab 1: open Application 1 and vote "Yes"
5. **Expected:** In Tab 2, Application 1 shows Reviewer A's vote as "Yes" live
6. In Tab 2: vote "Maybe" on the same application
7. **Expected:** In Tab 1, Reviewer B's vote updates live to "Maybe"

- [ ] **Step 4: Commit edge case testing results**

No code changes; just confirm tests passed.

```bash
git log --oneline -5
```

Expected: Recent commits visible showing voting implementation.

---

## Task 12: Push to GitHub and Verify Live

**Files:**
- No changes; deployment only

- [ ] **Step 1: Push to GitHub**

```bash
git push origin main
```

Expected: `main` branch updated on GitHub.

- [ ] **Step 2: Wait for GitHub Pages deploy**

Wait ~60 seconds for GitHub Pages to deploy.

- [ ] **Step 3: Test live URL**

Navigate to https://usmanjaved-droid.github.io/mulago-review-app

1. Set up reviewers and open an application
2. Vote on it
3. Refresh the page
4. **Expected:** Vote persists
5. Open in a second browser/incognito tab, select a different reviewer, open the same app
6. **Expected:** See the first reviewer's vote live

Expected: Voting system fully functional on live site.

---

## Self-Review Checklist

**Spec Coverage:**
- ✓ Votes table created with correct schema
- ✓ VOTES global state loaded on init and subscribed to realtime
- ✓ renderVotesSection() displays per-reviewer votes at bottom of eval panel
- ✓ Your row shows clickable buttons; other rows read-only
- ✓ Vote buttons styled with colors (green/orange/red)
- ✓ Empty votes show as "—"
- ✓ Votes only appear in detail panel, not main table
- ✓ saveVoteToDb() upserts to Supabase
- ✓ Realtime subscription re-renders votes when any reviewer votes
- ✓ Sync to index.html and GitHub push included

**Placeholder Scan:**
- ✓ All code shown in full
- ✓ All SQL, HTML, CSS, JS concrete and runnable
- ✓ All file paths exact
- ✓ All commands with expected output

**Type Consistency:**
- ✓ applicantId: always passed as string
- ✓ reviewerSlot: always 1–4 (int)
- ✓ vote: always 'yes', 'maybe', 'no', or null
- ✓ VOTES structure: { applicant_id: { slot: vote } }
- ✓ CURRENT_REVIEWER_SLOT used consistently

**Edge Cases Handled:**
- ✓ Empty reviewer slots (filtered in renderVotesSection)
- ✓ Reviewer removed but votes exist (won't render, acceptable)
- ✓ Concurrent votes (Supabase + realtime handles)
- ✓ Null votes (render as "—")

---

## Execution Path

**Plan complete and saved to `docs/superpowers/plans/2026-04-20-reviewer-voting-system.md`.**

Two execution options:

**1. Subagent-Driven (recommended)** — I dispatch a fresh subagent per task, review between tasks, fast iteration with checkpoint reviews.

**2. Inline Execution** — Execute tasks in this session using executing-plans, batch execution with checkpoints for your review.

Which approach?
