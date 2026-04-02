# Mulago Scale-Up Fellowship Review App

## Project Overview
A single-file HTML application for reviewing 38 fellowship applications for the **Mulago Scale-Up Fellowship 2026** (June 6–12). Supports multiple reviewers with real-time score sync via Supabase.

**Live URL:** https://usmanjaved-droid.github.io/mulago-review-app
**Local dev:** `npx serve -l 3000 .` → http://localhost:3000

---

## Architecture

### Files
- **`index.html`** — the app served by the browser (always keep in sync with `review-app.html`)
- **`review-app.html`** — the working source file; always edit this, then copy to `index.html`
- **`applicants_data.json`** — 38 applicants extracted verbatim from the original CSV
- **`patch_fullscreen.py`**, **`patch_ux.py`** — historical patch scripts (do not re-run)

### Rule: always sync after editing
After every edit to `review-app.html`, run:
```bash
cp "review-app.html" "index.html"
```
The server serves `index.html`. Forgetting this sync is the most common source of "my change isn't showing up" bugs.

### Stack
- Pure HTML/CSS/JS — no build step, no npm, no bundler
- **Supabase** (free tier) for database + real-time sync
- Supabase JS SDK loaded via CDN: `https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2`

---

## Supabase

**Project URL:** `https://wzvvylqxibsghpkmkfka.supabase.co`
**Anon key:** stored directly in `review-app.html` as `SUPABASE_KEY`

### Tables
| Table | Purpose |
|---|---|
| `reviewers` | Up to 4 reviewer slots (`slot` 1–4, `name`) |
| `assignments` | Maps `applicant_id` → `slot` |
| `evaluations` | Scores, status, recommendation, notes per applicant |

All three tables have RLS enabled with permissive `allow all` policies (anon key has full read/write).
All three tables are added to `supabase_realtime` publication for live sync.

---

## Data Model

### Applicants
38 applicants embedded as `const APPLICANTS = [...]` in the HTML. Each has:
- `id` (integer 1–38), `name`, `title`, `email`, `org`, `founded`, `budget`, `team`, `sector`, `stage`
- `reach`, `problem`, `solution`, `metrics`, `evidence`, `scale`, `extra`, `source` (verbatim text fields)
- `sector` — one of: Education, Health, Mental Health, Livelihoods, Environment, Women & Girls, Tech & Innovation, Youth, Disability

**Do not truncate or modify applicant text.** All fields must remain verbatim from the original CSV.

### Evaluations (Supabase)
```
applicant_id  TEXT PRIMARY KEY
status        TEXT  -- 'pending' | 'review' | 'shortlisted' | 'rejected'
problem       INT   -- 1–5
solution      INT   -- 1–5
evidence      INT   -- 1–5
scalability   INT   -- 1–5
scalestrategy INT   -- 1–5
recommendation TEXT -- 'Strong Yes' | 'Yes' | 'Maybe' | 'No'
notes         TEXT
updated_at    TIMESTAMPTZ
```

### Score Bands
| Score | Band |
|---|---|
| 21–25 | Strong Yes |
| 15–20 | Yes |
| 9–14 | Maybe |
| < 9 | No |

---

## Key JS Functions

| Function | Purpose |
|---|---|
| `init()` | Loads from Supabase, subscribes to realtime, restores reviewer identity, renders UI |
| `loadFromDb()` | Fetches all evaluations, assignments, reviewers into memory |
| `subscribeRealtime()` | Supabase channel listener — updates local state and re-renders on remote changes |
| `saveEvalToDb(id)` | Upserts evaluation for one applicant to Supabase |
| `openPanel(id)` | Opens full-screen split panel for an applicant; checks score gating |
| `renderTable()` | Renders the main application list with filters applied |
| `getFilteredSorted()` | Returns filtered + sorted applicant array (status, sector, reviewer filters) |
| `assignAndSave()` | Randomly splits unassigned apps across filled reviewer slots, saves to Supabase |
| `removeReviewer(slot)` | Deletes reviewer, clears their assignments and scores from Supabase |
| `renderScoreButtons(dim, canScore)` | Renders 1–5 score buttons for a dimension; disables if `canScore=false` |
| `updateEvalSummary()` | Updates total score display and suggested band in the eval panel |
| `renderReviewerProgress()` | Renders per-reviewer progress bars in sidebar |
| `selectReviewer(slot)` | Sets current reviewer identity, persists to localStorage |

---

## Reviewer Flow
1. Admin clicks **⚙ Reviewers** → enters names → **Assign & Save**
2. App randomly divides 38 apps into 4 equal slots (9/9/10/10). Unfilled slots stay unassigned.
3. Each reviewer opens the URL, picks their name from the name picker
4. Reviewer identity persists in `localStorage` as `mulago_reviewer`
5. Reviewers can **view** all applications but can only **score** their assigned ones
6. Scores sync live via Supabase realtime across all open browsers
7. Admin can remove a reviewer — clears their slot, scores, and assignments

---

## UI Layout
- **Full-screen split panel** when an application is opened:
  - Left column (`panel-content`): application text, reach callout, sections
  - Right column (`panel-eval`): status buttons, 5 score blocks, eval summary at natural bottom (not sticky)
- Score guides collapse by default, expand on click or when a score is saved
- `onmouseenter`/`onmouseleave` (not `mouseover`/`mouseout`) on score buttons to prevent bubble glitch
- Score guide stays open if manually expanded (`data-manually-expanded` attribute) — hover-out does not collapse it

---

## General Behaviour
- Before starting implementation on any UI change, propose a written design approach and wait for approval
- Before starting a new feature, ask clarifying questions until requirements are fully understood — do not write code until confirmed
- Before debugging a reported issue, verify the issue actually exists in the app (not just the preview tool)
- Always produce polished results — never leave UI in an ugly or unfinished state
- After every edit, sync `review-app.html` → `index.html`
- After significant changes, push to GitHub: `git add index.html review-app.html && git commit && git push`

---

## Deployment
- **GitHub repo:** https://github.com/usmanjaved-droid/mulago-review-app
- **GitHub Pages:** https://usmanjaved-droid.github.io/mulago-review-app (auto-deploys from `main` branch)
- To deploy: push to `main` — GitHub Pages picks it up within ~60 seconds
