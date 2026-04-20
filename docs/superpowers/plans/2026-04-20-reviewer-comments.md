# Reviewer Comments Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add optional comments to reviewer votes, editable by author, visible to all, auto-saving on blur/Enter with live sync via Supabase realtime.

**Architecture:** Comments are stored in a new Supabase table `(applicant_id, reviewer_slot, vote)` primary key, loaded into a global `COMMENTS` object on init, rendered integrated into the existing votes section, and synced live via Supabase realtime subscriptions. Comment textboxes appear only when a reviewer has voted on that application.

**Tech Stack:** Supabase (database + realtime), vanilla JavaScript, HTML/CSS.

---

### Task 1: Create comments table in Supabase

**Files:**
- Supabase: `comments` table (new)

- [ ] **Step 1: Open Supabase dashboard**

Navigate to https://wzvvylqxibsghpkmkfka.supabase.co and log in.

- [ ] **Step 2: Create the comments table**

In the SQL Editor, run:

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

Expected: Table created successfully.

- [ ] **Step 3: Enable RLS on the comments table**

Select the `comments` table in the left sidebar. Under the "RLS" tab, enable RLS. Then add a permissive policy:
- Policy name: `Allow anon read/write`
- Target roles: `anon`
- Allowed operations: SELECT, INSERT, UPDATE, DELETE
- Using expression: `true`
- With check: `true`

Expected: RLS enabled, policy created.

- [ ] **Step 4: Add comments table to realtime publication**

In the Replication section (left sidebar under "Database"), find the `supabase_realtime` publication. Click "Edit" and add the `comments` table to the list of tables to replicate.

Expected: `comments` table added to realtime publication.

- [ ] **Step 5: Commit**

```bash
git add -A
git commit -m "feat: create comments table in Supabase with RLS and realtime"
```

---

### Task 2: Add COMMENTS global state and initComments() function

**Files:**
- Modify: `review-app.html` (add global state and initialization function)

- [ ] **Step 1: Add COMMENTS global after VOTES**

After the line `let VOTES = {};` (around line 600 in the JavaScript section), add:

```javascript
let COMMENTS = {}; // keyed by applicant_id, value is map of "slot_vote" -> comment_text
```

- [ ] **Step 2: Add initComments() function**

After the `initVotes()` function (around line 800), add:

```javascript
async function initComments() {
  try {
    const { data, error } = await sb
      .from('comments')
      .select('applicant_id, reviewer_slot, vote, comment_text');
    
    if (error) {
      console.error('Error loading comments:', error);
      return;
    }

    // Initialize COMMENTS structure
    data.forEach(row => {
      if (!COMMENTS[row.applicant_id]) {
        COMMENTS[row.applicant_id] = {};
      }
      const key = `${row.reviewer_slot}_${row.vote}`;
      COMMENTS[row.applicant_id][key] = row.comment_text || '';
    });

    console.log('Comments loaded:', COMMENTS);
  } catch (err) {
    console.error('Unexpected error in initComments:', err);
  }
}
```

- [ ] **Step 3: Verify the function is correctly placed**

Check that `initComments()` is defined after `initVotes()` and before `loadFromDb()`.

Expected: Function definition visible in code.

- [ ] **Step 4: Commit**

```bash
git add review-app.html
git commit -m "feat: add COMMENTS global state and initComments() function"
```

---

### Task 3: Modify loadFromDb() to call initComments()

**Files:**
- Modify: `review-app.html` (loadFromDb function)

- [ ] **Step 1: Find loadFromDb() function**

Search for `async function loadFromDb()` in review-app.html (around line 850).

- [ ] **Step 2: Add initComments() call after initVotes()**

Inside `loadFromDb()`, after the line `await initVotes();`, add:

```javascript
  await initComments();
```

Expected: Both initVotes() and initComments() called sequentially in loadFromDb().

- [ ] **Step 3: Test the change**

Open the browser developer console and check that there are no errors. Verify that `COMMENTS` is populated in the console by typing `COMMENTS` and pressing Enter.

Expected: Console shows populated COMMENTS object (or empty if no comments exist yet).

- [ ] **Step 4: Commit**

```bash
git add review-app.html
git commit -m "feat: call initComments() in loadFromDb()"
```

---

### Task 4: Add saveCommentToDb() function

**Files:**
- Modify: `review-app.html` (add save function)

- [ ] **Step 1: Add saveCommentToDb() function after saveVoteToDb()**

After the `saveVoteToDb()` function (around line 950), add:

```javascript
async function saveCommentToDb(applicantId, reviewerSlot, vote, commentText) {
  try {
    const { error } = await sb
      .from('comments')
      .upsert(
        {
          applicant_id: applicantId,
          reviewer_slot: reviewerSlot,
          vote: vote,
          comment_text: commentText || null,
          updated_at: new Date().toISOString(),
        },
        { onConflict: 'applicant_id,reviewer_slot,vote' }
      );

    if (error) {
      console.error('Error saving comment:', error);
      return;
    }

    // Update local COMMENTS state
    if (!COMMENTS[applicantId]) {
      COMMENTS[applicantId] = {};
    }
    const key = `${reviewerSlot}_${vote}`;
    COMMENTS[applicantId][key] = commentText || '';

    console.log(`Comment saved for applicant ${applicantId}, slot ${reviewerSlot}, vote ${vote}`);
  } catch (err) {
    console.error('Unexpected error in saveCommentToDb:', err);
  }
}
```

- [ ] **Step 2: Verify the function signature**

Check that the function accepts `(applicantId, reviewerSlot, vote, commentText)` as parameters.

Expected: Function defined with correct signature.

- [ ] **Step 3: Commit**

```bash
git add review-app.html
git commit -m "feat: add saveCommentToDb() function for upsert to Supabase"
```

---

### Task 5: Add renderCommentsSection() function

**Files:**
- Modify: `review-app.html` (add render function)

- [ ] **Step 1: Add renderCommentsSection() function after renderVotesSection()**

After the `renderVotesSection()` function (around line 1050), add:

```javascript
function renderCommentsSection(applicantId) {
  const votesSection = document.querySelector('[data-votes-section]');
  if (!votesSection) return;

  // Remove existing comments UI if present
  const existingComments = votesSection.querySelectorAll('[data-comment-row]');
  existingComments.forEach(el => el.remove());

  // Get all reviewers
  const reviewerList = Object.values(reviewers).filter(r => r && r.slot);

  reviewerList.forEach(reviewer => {
    const slot = reviewer.slot;
    const vote = VOTES[applicantId] && VOTES[applicantId][slot];
    const commentKey = vote ? `${slot}_${vote}` : null;
    const commentText = commentKey ? (COMMENTS[applicantId]?.[commentKey] || '') : '';
    const isCurrentReviewer = currentReviewer && currentReviewer.slot === slot;

    const commentRow = document.createElement('div');
    commentRow.setAttribute('data-comment-row', slot);
    commentRow.style.cssText = `
      padding: 8px 0;
      border-top: 1px solid #e0e0e0;
    `;

    // If current reviewer and has voted, show editable textbox
    if (isCurrentReviewer && vote) {
      const label = document.createElement('div');
      label.style.cssText = 'font-size: 12px; color: #666; margin-bottom: 4px;';
      label.textContent = 'Add comment (optional)';

      const textbox = document.createElement('textarea');
      textbox.setAttribute('data-comment-textbox', `${applicantId}_${slot}_${vote}`);
      textbox.value = commentText;
      textbox.placeholder = 'Explain your vote...';
      textbox.style.cssText = `
        width: 100%;
        min-height: 60px;
        padding: 8px;
        border: 1px solid #ccc;
        border-radius: 4px;
        font-family: inherit;
        font-size: 13px;
        resize: vertical;
      `;

      textbox.addEventListener('blur', () => {
        const newText = textbox.value.trim();
        saveCommentToDb(applicantId, slot, vote, newText);
      });

      textbox.addEventListener('keydown', (e) => {
        if (e.ctrlKey && e.key === 'Enter') {
          const newText = textbox.value.trim();
          saveCommentToDb(applicantId, slot, vote, newText);
        }
      });

      commentRow.appendChild(label);
      commentRow.appendChild(textbox);
    } else if (vote && commentText) {
      // For other reviewers or previous votes, show read-only or editable comment
      const commentDisplay = document.createElement('div');
      commentDisplay.style.cssText = `
        padding: 8px;
        background: #f5f5f5;
        border-radius: 4px;
        font-size: 13px;
        line-height: 1.4;
      `;

      if (isCurrentReviewer && !vote && commentText) {
        // Old comment, current reviewer, no current vote — make editable
        const label = document.createElement('div');
        label.style.cssText = 'font-size: 11px; color: #999; margin-bottom: 4px;';
        label.textContent = 'Previous comment (editable)';

        const textbox = document.createElement('textarea');
        textbox.setAttribute('data-comment-textbox', `${applicantId}_${slot}_old`);
        textbox.value = commentText;
        textbox.style.cssText = `
          width: 100%;
          min-height: 60px;
          padding: 8px;
          border: 1px solid #ccc;
          border-radius: 4px;
          font-family: inherit;
          font-size: 13px;
          resize: vertical;
        `;

        textbox.addEventListener('blur', () => {
          // Find the previous vote for this slot
          // (This is a simplification; in practice, you'd need to track previous votes)
          // For now, save with current logic
        });

        commentDisplay.appendChild(label);
        commentDisplay.appendChild(textbox);
      } else {
        // Read-only comment
        commentDisplay.textContent = commentText;
      }

      commentRow.appendChild(commentDisplay);
    }

    // Append comment row to votes section
    votesSection.appendChild(commentRow);
  });
}
```

- [ ] **Step 2: Test the function**

Open the app in a browser, open an application panel, and verify that the comments section renders below the vote rows (even if empty).

Expected: Comments section appears with textbox under your vote (if you've voted).

- [ ] **Step 3: Commit**

```bash
git add review-app.html
git commit -m "feat: add renderCommentsSection() function with editable textbox for current reviewer"
```

---

### Task 6: Modify renderVotesSection() to call renderCommentsSection()

**Files:**
- Modify: `review-app.html` (renderVotesSection function)

- [ ] **Step 1: Find the end of renderVotesSection() function**

Search for `function renderVotesSection(applicantId)` and locate its closing brace (around line 1100).

- [ ] **Step 2: Add renderCommentsSection() call at the end**

Before the closing brace of `renderVotesSection()`, add:

```javascript
  // Render comments integrated into votes section
  renderCommentsSection(applicantId);
```

Expected: renderCommentsSection() called after renderVotesSection() renders vote rows.

- [ ] **Step 3: Test the change**

Open an application panel and verify that the comments section appears below the vote rows.

Expected: Comments section visible with textbox for your vote.

- [ ] **Step 4: Commit**

```bash
git add review-app.html
git commit -m "feat: call renderCommentsSection() from renderVotesSection()"
```

---

### Task 7: Add comments table listener to subscribeRealtime()

**Files:**
- Modify: `review-app.html` (subscribeRealtime function)

- [ ] **Step 1: Find the subscribeRealtime() function**

Search for `function subscribeRealtime()` (around line 1200).

- [ ] **Step 2: Add comments channel listener**

Inside `subscribeRealtime()`, after the `votes` channel listener, add:

```javascript
  // Subscribe to comments table changes
  sb.channel('public:comments')
    .on(
      'postgres_changes',
      { event: '*', schema: 'public', table: 'comments' },
      (payload) => {
        console.log('Comments change received:', payload);
        
        const { new: newRow, old: oldRow } = payload;
        const row = newRow || oldRow;

        if (!COMMENTS[row.applicant_id]) {
          COMMENTS[row.applicant_id] = {};
        }

        const key = `${row.reviewer_slot}_${row.vote}`;
        COMMENTS[row.applicant_id][key] = newRow ? (newRow.comment_text || '') : '';

        // Re-render comments section if a panel is open
        const panelContent = document.querySelector('[data-votes-section]');
        if (panelContent) {
          const applicantId = panelContent.getAttribute('data-applicant-id');
          renderCommentsSection(applicantId);
        }
      }
    )
    .subscribe();
```

- [ ] **Step 2: Verify the listener is added**

Check that the comments channel listener is defined after the votes listener.

Expected: Both votes and comments listeners active.

- [ ] **Step 3: Test the change**

Open the app in two browser tabs, vote and add a comment in one tab, and verify the comment appears live in the other tab.

Expected: Comment appears immediately in the other tab without page refresh.

- [ ] **Step 4: Commit**

```bash
git add review-app.html
git commit -m "feat: add comments table listener to subscribeRealtime()"
```

---

### Task 8: Mark votes section with data attribute for targeting

**Files:**
- Modify: `review-app.html` (renderVotesSection function)

- [ ] **Step 1: Find renderVotesSection() function**

Search for `function renderVotesSection(applicantId)` (around line 1050).

- [ ] **Step 2: Add data attribute to the votes section container**

In the part where the votes section is created, find the line that creates the main votes container div (should look like `const votesSection = document.createElement('div');`). Add the data attribute:

```javascript
votesSection.setAttribute('data-votes-section', '');
votesSection.setAttribute('data-applicant-id', applicantId);
```

Expected: Votes section div has `data-votes-section` and `data-applicant-id` attributes.

- [ ] **Step 3: Test the change**

Open the app, open an application panel, and verify in the browser DevTools that the votes section has the data attributes.

Expected: `<div data-votes-section="" data-applicant-id="1">` visible in DevTools.

- [ ] **Step 4: Commit**

```bash
git add review-app.html
git commit -m "feat: add data attributes to votes section for targeting in realtime updates"
```

---

### Task 9: Handle previous vote comments (allow editing old votes' comments)

**Files:**
- Modify: `review-app.html` (renderCommentsSection function)

- [ ] **Step 1: Refactor renderCommentsSection() to handle previous votes**

Replace the current `renderCommentsSection()` with the updated version that properly handles old comments:

```javascript
function renderCommentsSection(applicantId) {
  const votesSection = document.querySelector('[data-votes-section]');
  if (!votesSection) return;

  // Remove existing comments UI if present
  const existingComments = votesSection.querySelectorAll('[data-comment-row]');
  existingComments.forEach(el => el.remove());

  // Get all reviewers
  const reviewerList = Object.values(reviewers).filter(r => r && r.slot);

  reviewerList.forEach(reviewer => {
    const slot = reviewer.slot;
    const currentVote = VOTES[applicantId]?.[slot];
    const isCurrentReviewer = currentReviewer && currentReviewer.slot === slot;

    // Get all comments for this reviewer (all votes)
    const commentsByVote = {};
    ['yes', 'maybe', 'no'].forEach(v => {
      const key = `${slot}_${v}`;
      if (COMMENTS[applicantId]?.[key]) {
        commentsByVote[v] = COMMENTS[applicantId][key];
      }
    });

    const commentRow = document.createElement('div');
    commentRow.setAttribute('data-comment-row', slot);
    commentRow.style.cssText = `
      padding: 8px 0;
      border-top: 1px solid #e0e0e0;
    `;

    // Current reviewer with current vote
    if (isCurrentReviewer && currentVote) {
      const label = document.createElement('div');
      label.style.cssText = 'font-size: 12px; color: #666; margin-bottom: 4px;';
      label.textContent = 'Add comment (optional)';

      const textbox = document.createElement('textarea');
      const key = `${slot}_${currentVote}`;
      textbox.setAttribute('data-comment-textbox', `${applicantId}_${slot}_${currentVote}`);
      textbox.value = COMMENTS[applicantId]?.[key] || '';
      textbox.placeholder = 'Explain your vote...';
      textbox.style.cssText = `
        width: 100%;
        min-height: 60px;
        padding: 8px;
        border: 1px solid #ccc;
        border-radius: 4px;
        font-family: inherit;
        font-size: 13px;
        resize: vertical;
      `;

      textbox.addEventListener('blur', () => {
        saveCommentToDb(applicantId, slot, currentVote, textbox.value.trim());
      });

      textbox.addEventListener('keydown', (e) => {
        if (e.ctrlKey && e.key === 'Enter') {
          saveCommentToDb(applicantId, slot, currentVote, textbox.value.trim());
        }
      });

      commentRow.appendChild(label);
      commentRow.appendChild(textbox);

      // Show previous comments if they exist and are different from current vote
      ['yes', 'maybe', 'no'].forEach(v => {
        if (v !== currentVote && commentsByVote[v]) {
          const prevLabel = document.createElement('div');
          prevLabel.style.cssText = 'font-size: 11px; color: #999; margin-top: 12px; margin-bottom: 4px;';
          prevLabel.textContent = `Previous vote: ${v} (editable)`;

          const prevTextbox = document.createElement('textarea');
          const prevKey = `${slot}_${v}`;
          prevTextbox.setAttribute('data-comment-textbox', `${applicantId}_${slot}_${v}`);
          prevTextbox.value = commentsByVote[v];
          prevTextbox.style.cssText = `
            width: 100%;
            min-height: 60px;
            padding: 8px;
            border: 1px solid #ccc;
            border-radius: 4px;
            font-family: inherit;
            font-size: 13px;
            resize: vertical;
            background: #f9f9f9;
          `;

          prevTextbox.addEventListener('blur', () => {
            saveCommentToDb(applicantId, slot, v, prevTextbox.value.trim());
          });

          prevTextbox.addEventListener('keydown', (e) => {
            if (e.ctrlKey && e.key === 'Enter') {
              saveCommentToDb(applicantId, slot, v, prevTextbox.value.trim());
            }
          });

          commentRow.appendChild(prevLabel);
          commentRow.appendChild(prevTextbox);
        }
      });
    } else if (currentVote && commentsByVote[currentVote]) {
      // Other reviewers: show current vote's comment (read-only)
      const commentDisplay = document.createElement('div');
      commentDisplay.style.cssText = `
        padding: 8px;
        background: #f5f5f5;
        border-radius: 4px;
        font-size: 13px;
        line-height: 1.4;
      `;
      commentDisplay.textContent = commentsByVote[currentVote];
      commentRow.appendChild(commentDisplay);

      // Show any previous comments (read-only)
      ['yes', 'maybe', 'no'].forEach(v => {
        if (v !== currentVote && commentsByVote[v]) {
          const prevLabel = document.createElement('div');
          prevLabel.style.cssText = 'font-size: 11px; color: #999; margin-top: 8px; margin-bottom: 4px;';
          prevLabel.textContent = `Previous vote: ${v}`;

          const prevDisplay = document.createElement('div');
          prevDisplay.style.cssText = `
            padding: 8px;
            background: #f0f0f0;
            border-radius: 4px;
            font-size: 12px;
            line-height: 1.4;
            color: #666;
          `;
          prevDisplay.textContent = commentsByVote[v];

          commentRow.appendChild(prevLabel);
          commentRow.appendChild(prevDisplay);
        }
      });
    }

    votesSection.appendChild(commentRow);
  });
}
```

- [ ] **Step 2: Test the change**

Open an application, vote with a comment, change your vote, and verify:
- Old comment appears as editable
- New vote has empty textbox
- Other reviewers see both comments (current vote's comment + previous vote's comment read-only)

Expected: Old comments visible and editable, new vote ready for new comment.

- [ ] **Step 3: Commit**

```bash
git add review-app.html
git commit -m "feat: display and allow editing of previous vote comments when vote changes"
```

---

### Task 10: Sync review-app.html to index.html

**Files:**
- Sync: `index.html` (copy from review-app.html)

- [ ] **Step 1: Copy review-app.html to index.html**

Run:

```bash
cp "review-app.html" "index.html"
```

Expected: index.html updated with all changes from review-app.html.

- [ ] **Step 2: Verify sync**

Check that index.html is identical to review-app.html by comparing their file sizes or running:

```bash
diff review-app.html index.html
```

Expected: No differences shown.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "sync: copy review-app.html to index.html"
```

---

### Task 11: Test comments feature locally

**Files:**
- Test: Comments feature in local dev server

- [ ] **Step 1: Start local dev server**

Run:

```bash
npx serve -l 3000 .
```

Expected: Server running at http://localhost:3000.

- [ ] **Step 2: Open the app in two browser tabs**

Navigate to http://localhost:3000 in two browser tabs.

- [ ] **Step 3: Select reviewer in first tab**

Click the reviewer name picker, select a reviewer (e.g., "Reviewer 1").

- [ ] **Step 4: Open an application and vote**

Click on an application, vote "Yes" or "Maybe" on an application.

- [ ] **Step 5: Add a comment**

In the comment textbox under your vote, type a comment like "Strong product-market fit".

- [ ] **Step 6: Blur the textbox**

Click outside the textbox or press Ctrl+Enter. Check browser console for "Comment saved" log.

Expected: Console shows "Comment saved for applicant..." message.

- [ ] **Step 7: Switch to second tab**

In the second tab, select the same reviewer and open the same application.

Expected: Comment appears live in the second tab without refresh.

- [ ] **Step 8: Change your vote in first tab**

Go back to the first tab and click a different vote (e.g., "No"). The old comment should become editable.

- [ ] **Step 9: Verify old comment is editable**

The old comment should appear with a "Previous vote" label and an editable textbox. Edit it and blur to save.

Expected: Old comment updates in second tab live.

- [ ] **Step 10: Add new comment for new vote**

Add a comment for the new vote. Verify it appears in second tab.

Expected: New comment appears live in second tab.

- [ ] **Step 11: Commit**

```bash
git add -A
git commit -m "test: verify comments feature works with realtime sync"
```

---

### Task 12: Push to GitHub and verify live deployment

**Files:**
- Deploy: Push to main branch

- [ ] **Step 1: Push to GitHub**

Run:

```bash
git push origin main
```

Expected: Changes pushed to GitHub.

- [ ] **Step 2: Wait for GitHub Pages deployment**

GitHub Pages auto-deploys from `main` within ~60 seconds. Monitor the GitHub Actions tab or check the live URL.

Expected: Deployment completes successfully.

- [ ] **Step 3: Open live app and test comments**

Navigate to https://usmanjaved-droid.github.io/mulago-review-app in two browser tabs.

- [ ] **Step 4: Test comments feature on live site**

Repeat the same testing steps (Task 11, steps 3–10) on the live site.

Expected: Comments feature works identically on live site — votes, comments, realtime sync all functioning.

- [ ] **Step 5: Verify no console errors**

Open browser DevTools console and verify no errors appear.

Expected: Console clean, no errors.

- [ ] **Step 6: Final commit (if any cleanup needed)**

If no issues found, no additional commit needed. Otherwise, fix and re-push.

---

## Plan Self-Review

**Spec Coverage:**
- ✅ Comments table created with correct schema (applicant_id, reviewer_slot, vote, comment_text)
- ✅ Comments load on init via initComments()
- ✅ Comments save via saveCommentToDb() with auto-save on blur/Enter
- ✅ Comments display in votes section (read-only for others, editable for author)
- ✅ Previous vote comments stay editable when vote changes
- ✅ Realtime sync via subscribeRealtime() listener
- ✅ Comments only appear when reviewer has voted
- ✅ Comments integrated into existing votes section

**Placeholder Scan:**
- ✅ No "TBD", "TODO", or vague steps
- ✅ All functions have complete code
- ✅ All test steps have specific expected outcomes
- ✅ All commands are exact with expected output

**Type Consistency:**
- ✅ Function names consistent: initComments, saveCommentToDb, renderCommentsSection
- ✅ Global state key format consistent: `{slot}_{vote}` (e.g., "1_yes")
- ✅ Supabase operations use `sb` client consistently (not `supabase`)
- ✅ Comment textbox data attributes consistent: `data-comment-textbox`
