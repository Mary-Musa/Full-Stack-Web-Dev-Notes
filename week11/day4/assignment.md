# Week 11 - Day 4 Assignment

## Title
CRM Dashboard -- React Frontend For Your Leads

## Overview
Day 4 is pre-weekend polish day. Today you build a React dashboard that reads from your Week 11 leads API, lets the sales team search and filter, and updates lead status. Most of this is repeated React patterns you have built before, so AI help is welcome. The weekend project ships the full system end-to-end.

## Learning Objectives Assessed
- Fetch paginated data from your own API
- Build a searchable, filterable table view
- Open a detail view in a side panel or new route
- Update lead status via PATCH
- Ship a usable CRM demo by end of Week 11

## Prerequisites
- Days 1-3 completed

## AI Usage Rules

**Ratio:** 45/55. **Habit:** Repeated patterns = AI. See [../ai.md](../ai.md).

- **ALLOWED FOR:** React dashboard scaffolding, table UI, pagination controls, status dropdowns. You have built all of this before.
- **NOT ALLOWED FOR:** Deciding what the CRM looks like or which columns to show -- that is your product decision.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Scaffold the dashboard

**What to do:**
Create a new React project or add to your existing `react-lab`. Build:

- A table that lists leads from `GET /api/leads`
- Columns: Name, Phone, Email, Inquiry, Status, Created
- A search input (hooked to `?q=`)
- A status filter dropdown (hooked to `?status=`)
- Pagination controls (Previous / Next)

Use `useFetch` or a simple `useEffect` pattern.

**Expected output:**
Table loads real leads from your API.

### Task 2: Loading, empty, and error states

**What to do:**
Ensure the table renders sensibly when:
- Loading (show "Loading...")
- Empty (show "No leads yet. Send a WhatsApp to your test number to get started.")
- Error (show a clear error message)

**Expected output:**
All three states visible. Screenshot `day4-states.png`.

### Task 3: Detail view

**What to do:**
Clicking a row opens a detail view (side panel OR new route `/leads/:id`). The detail shows:
- Lead info (name, phone, email, inquiry, status, notes)
- Conversation history (list of messages from the `messages` table)
- A dropdown to change status (PATCH to the API)
- A notes textarea (PATCH to the API, save on blur)

**Expected output:**
Clicking a lead shows its full details and conversation. Changing status persists.

### Task 4: Pre-weekend checklist

**What to do:**
Create `CHECKLIST.md`:

```markdown
## Week 11 Day 4 Pre-Weekend Checklist

- [ ] Webhook verification working
- [ ] HMAC signature check catches tampering
- [ ] Bot completes a full conversation and creates a lead
- [ ] REST API returns leads with pagination, search, status filter
- [ ] React dashboard lists leads in a searchable table
- [ ] Detail view shows conversation history
- [ ] Status updates persist via PATCH
- [ ] AI_AUDIT.md has split log for every feature
- [ ] Repo pushed and clean
```

Tick every box honestly.

**Expected output:**
`CHECKLIST.md` committed.

### Task 5: One "think" task

**What to do:**
In `design-notes.md`, answer in 5-8 sentences:
- What would break first if 100 conversations happened simultaneously? Why?
- If a sales rep says "I need to see which leads have not replied in 3 days", what feature would you add and where?
- How would you prevent one rep from accidentally overwriting another rep's status update?

No AI. Your own words.

**Expected output:**
`design-notes.md` committed.

## Stretch Goals (Optional - Extra Credit)

- Add a WebSocket or polling update so the dashboard refreshes when a new lead arrives.
- Add CSV export of the filtered view.
- Add a "Assign to me" button that stores `assigned_to` on the lead.

## Submission Requirements

- **What to submit:** Repo, `CHECKLIST.md`, `design-notes.md`, screenshots.
- **Deadline:** End of Day 4

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Table loads real leads | 20 | Pagination + search + filter all wired. |
| Three UI states | 10 | Loading, empty, error. |
| Detail view with conversation | 25 | Full lead info + messages list + status + notes. |
| PATCH persistence | 15 | Status and notes save. Refresh confirms. |
| Pre-weekend checklist | 10 | All boxes honest. |
| Design thinking notes | 15 | Three questions answered. Own words. |
| Clean commits | 5 | Conventional messages. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Optimistic updates without rollback.** If you update the UI before the API call confirms, have a rollback path for failures.
- **Fetching on every keystroke in search.** Debounce the search input or wait for Enter to avoid hammering your API.
- **Hardcoding the API base URL in every file.** Put it in one config file so you can switch between dev and prod easily.

## Resources

- Day 4 reading: [The CRM Dashboard.md](./The%20CRM%20Dashboard.md)
- Week 11 AI boundaries: [../ai.md](../ai.md)
