# Week 11, Day 4: The CRM Dashboard

By the end of today, a sales team will be able to open `http://localhost:3000` in a browser and see every WhatsApp lead your bot has captured. They can search by name or phone, filter by pipeline status, click a row to see the full conversation in a slide-out panel, change the status with a single click, and tap "Reply via WhatsApp" to jump straight into a chat. New leads show up within ten seconds of the conversation completing, without a page refresh. This is the payoff day of the week -- the moment where the backend you have been building becomes something a real business could use this afternoon.

Everything you build today uses patterns you already know. Components, props, `useState`, `useEffect`, controlled inputs, `axios`, rendering arrays with `.map()`, conditional rendering. You learned all of these in Weeks 7-9. Today you assemble them into a working CRM and style it with Tailwind. The only genuinely new piece is using Tailwind utility classes end-to-end instead of writing CSS files -- and Week 8 Day 4 already introduced the concept.

**Prior-week concepts you will use today:**
- `create-react-app` and React project structure (Week 7, Week 10 Day 3)
- Components, props, `useState`, `useEffect` (Weeks 7-8)
- Controlled inputs and form state (Week 8, Day 1)
- API service layer pattern -- all `axios` calls in one file (Week 9, Day 1)
- `axios.create` with a `baseURL` (Week 10, Day 3)
- `.map()` to render lists, conditional rendering with `&&` and ternaries (Week 7, Day 4)
- `useCallback` for stable function references across renders (Week 9, Day 3)
- Tailwind utility classes as a styling option (Week 8, Day 4)

**Estimated time:** 4-5 hours

---

## Recap: What You Built Yesterday

On Day 3 you built:

- `GET /api/leads` with search, status filtering, and pagination.
- `GET /api/leads/:id` returning the lead together with its full conversation history.
- `PATCH /api/leads/:id` for status and notes updates with server-side validation.
- `GET /api/stats` with total, today's count, and per-status breakdown.

Keep your server running (`npm run dev` in the `server` folder) and ngrok up in its own terminal. We need the backend live while we build the frontend.

---

## What Makes A CRM Actually Useful

Before you touch React, decide what the screen has to do. Not the colours, not the animations -- the job to be done. Picture three real people who will use this:

- The **receptionist** at a real estate agency in Lavington glances at the dashboard every few minutes and sees "3 new". She clicks the first one, reads the conversation, sees they want a 3-bedroom, changes status to "Contacted", and calls them back.
- The **sales lead** at a Ngong Road car yard opens the dashboard at 5pm, filters to status "Qualified", and makes sure every qualified lead has been followed up today.
- The **owner** of an insurance brokerage in Westlands checks on Saturday morning, filtering to "Converted this month", to see how the week went.

The dashboard has to work for all three without any training. That gives us three real requirements:

1. **Findability.** Can I find a specific lead in under five seconds? That means search by name or phone, and filter by status.
2. **Context on one screen.** When I open a lead, can I see the full conversation, the status, and the notes without clicking around? That is why we build a slide-out panel instead of a separate page.
3. **One-click status updates.** The sales team updates statuses dozens of times a day. If it takes three clicks, they stop doing it, and the data rots.

Anything beyond those three is a bonus. Build the three well before you build anything pretty.

### Real-Time vs Polling

How does the dashboard know a new lead just came in? Three options:

- **Manual refresh.** User clicks a "Refresh" button. Zero complexity. Fine for a prototype.
- **Polling.** The React app calls `GET /api/leads` every 10 seconds in a `useEffect`. Simple, reliable, costs a bit of bandwidth. This is what we use today.
- **Server-Sent Events (SSE) or WebSockets.** The server pushes updates the moment a lead is created. Feels instant. More code. We list this as a stretch goal at the end of the day.

Polling is good enough for five users and fifty leads a day. Upgrade to SSE when the team tells you they are seeing leads show up "a bit late."

---

## Creating The React Project

From the `whatsapp-crm` root folder:

```bash
npx create-react-app client
cd client
npm install react-router-dom axios
```

You have used `create-react-app` since Week 7, and this is the same client setup as the M-Pesa Paylink project from Week 10 Day 3. We install `react-router-dom` for completeness (you may want it for stretch goals) and `axios` for our API service layer -- same reasons as last week.

### Adding Tailwind

🆕 NEW CONCEPT (mostly): Week 8 Day 4 introduced Tailwind as one of several styling options, but you did not build anything end-to-end with it. Today you use it for real. If you get stuck on a utility class, the Tailwind docs search bar is your best friend.

Install the Tailwind v3 toolchain (v4 has a different install flow that is not yet the CRA default):

```bash
npm install -D tailwindcss@3 postcss autoprefixer
npx tailwindcss init -p
```

This creates `tailwind.config.js` and `postcss.config.js`. Configure Tailwind's content paths in `client/tailwind.config.js`:

```javascript
// client/tailwind.config.js
module.exports = {
  content: ["./src/**/*.{js,jsx}"],
  theme: { extend: {} },
  plugins: [],
};
```

The `content` array tells Tailwind where to look for class names. Any utility you use in a file outside that glob will be stripped from the final CSS bundle as unused.

Replace the contents of `client/src/index.css` with Tailwind's three directives:

```css
/* client/src/index.css */
@tailwind base;
@tailwind components;
@tailwind utilities;
```

These three lines are Tailwind's entry point. `base` is a CSS reset and sensible defaults. `components` is for any reusable component classes (we do not add any). `utilities` is the pile of `p-4`, `text-red-600`, `flex` classes we will actually use.

### Folder Structure

Organize `client/src` like this -- same shape as Week 10 Day 3:

```
client/src/
  App.js
  index.js
  index.css
  pages/
    Dashboard.js
  components/
    StatsCards.js
    LeadsTable.js
    LeadDetail.js
  services/
    api.js
```

```bash
mkdir src/pages src/components src/services
```

---

## The API Service Layer

Create `client/src/services/api.js`. This follows the pattern from Week 9 Day 1 -- all `axios` calls live in one file so components stay clean.

```javascript
// client/src/services/api.js
import axios from "axios";

const http = axios.create({
  baseURL: "http://localhost:5000/api",
});

export async function listLeads({ search = "", status = "", page = 1 } = {}) {
  const { data } = await http.get("/leads", {
    params: { search, status, page, pageSize: 25 },
  });
  return data;
}

export async function getLead(id) {
  const { data } = await http.get(`/leads/${id}`);
  return data;
}

export async function updateLead(id, patch) {
  const { data } = await http.patch(`/leads/${id}`, patch);
  return data;
}

export async function getStats() {
  const { data } = await http.get("/stats");
  return data;
}
```

Four functions -- one for each backend endpoint you built yesterday. Components import these by name and never touch `axios` directly. If the backend URL ever changes, you update one line in one file.

`axios.create({ baseURL })` is the small pattern you saw in Week 10 Day 3 -- it lets every call below be written as `/leads` instead of `http://localhost:5000/api/leads`.

---

## The App Shell

Replace `client/src/App.js`:

```javascript
// client/src/App.js
import Dashboard from "./pages/Dashboard";

function App() {
  return <Dashboard />;
}

export default App;
```

We are not using React Router today because the whole CRM is one page -- the detail view is a slide-out panel, not a route. If you want to add routes later (for example a dedicated `/leads/:id` page), the Week 8 Day 3 patterns apply unchanged.

---

## The Stats Cards Component

Create `client/src/components/StatsCards.js`:

```javascript
// client/src/components/StatsCards.js
import { useEffect, useState } from "react";
import { getStats } from "../services/api";

export default function StatsCards({ refreshKey }) {
  const [stats, setStats] = useState(null);
  const [error, setError] = useState(null);

  useEffect(() => {
    let cancelled = false;
    getStats()
      .then((data) => !cancelled && setStats(data))
      .catch((err) => !cancelled && setError(err.message));
    return () => {
      cancelled = true;
    };
  }, [refreshKey]);

  if (error) {
    return <div className="text-red-600 text-sm">Stats error: {error}</div>;
  }
  if (!stats) {
    return <div className="text-gray-500 text-sm">Loading stats...</div>;
  }

  const cards = [
    { label: "Total leads", value: stats.total },
    { label: "New today", value: stats.today },
    { label: "Qualified", value: stats.byStatus.qualified },
    { label: "Converted", value: stats.byStatus.converted },
  ];

  return (
    <div className="grid grid-cols-2 md:grid-cols-4 gap-4">
      {cards.map((c) => (
        <div
          key={c.label}
          className="bg-white rounded-lg shadow p-4 border border-gray-100"
        >
          <div className="text-sm text-gray-500">{c.label}</div>
          <div className="text-3xl font-bold text-gray-900 mt-1">
            {c.value}
          </div>
        </div>
      ))}
    </div>
  );
}
```

Standard `useEffect` + `useState` pattern from Week 8. The `refreshKey` prop is a trick: whenever the parent changes the number, this effect re-runs and refetches. We will use it to drive polling.

The `cancelled` flag in the effect is the cleanup pattern from Week 9 Day 1 -- if the component unmounts (or the effect re-runs) before the fetch resolves, we skip the `setState` call to avoid a warning.

---

## The Leads Table Component

Create `client/src/components/LeadsTable.js`:

```javascript
// client/src/components/LeadsTable.js
const STATUS_STYLES = {
  new: "bg-blue-100 text-blue-800",
  contacted: "bg-yellow-100 text-yellow-800",
  qualified: "bg-purple-100 text-purple-800",
  converted: "bg-green-100 text-green-800",
  lost: "bg-gray-200 text-gray-700",
};

export default function LeadsTable({ leads, onSelect }) {
  if (leads.length === 0) {
    return (
      <div className="text-center text-gray-500 py-12 bg-white rounded-lg shadow">
        No leads yet. Message your WhatsApp number to create one.
      </div>
    );
  }

  return (
    <div className="bg-white rounded-lg shadow overflow-hidden">
      <table className="w-full text-sm">
        <thead className="bg-gray-50 text-gray-600 text-left">
          <tr>
            <th className="px-4 py-3">Name</th>
            <th className="px-4 py-3">Phone</th>
            <th className="px-4 py-3">Inquiry</th>
            <th className="px-4 py-3">Status</th>
            <th className="px-4 py-3">Received</th>
          </tr>
        </thead>
        <tbody>
          {leads.map((lead) => (
            <tr
              key={lead.id}
              className="border-t border-gray-100 hover:bg-gray-50 cursor-pointer"
              onClick={() => onSelect(lead)}
            >
              <td className="px-4 py-3 font-medium text-gray-900">
                {lead.name || "(no name yet)"}
              </td>
              <td className="px-4 py-3 text-gray-700">+{lead.wa_phone}</td>
              <td className="px-4 py-3 text-gray-700">
                {lead.inquiry_type || "-"}
              </td>
              <td className="px-4 py-3">
                <span
                  className={
                    "px-2 py-1 rounded-full text-xs font-medium " +
                    (STATUS_STYLES[lead.status] || "bg-gray-100 text-gray-700")
                  }
                >
                  {lead.status}
                </span>
              </td>
              <td className="px-4 py-3 text-gray-500">
                {new Date(lead.created_at.replace(" ", "T") + "Z").toLocaleString()}
              </td>
            </tr>
          ))}
        </tbody>
      </table>
    </div>
  );
}
```

Colour-coded status badges are the smallest useful feature of a CRM dashboard. One glance tells you where every lead is. `STATUS_STYLES` is the single source of truth for colours -- if you ever want to change "qualified" to orange, change it in one place.

The `new Date(lead.created_at.replace(" ", "T") + "Z")` dance converts SQLite's `YYYY-MM-DD HH:MM:SS` format into a proper JavaScript `Date` object. SQLite stores timestamps as strings with a space separator; JS `Date` prefers the ISO 8601 `T` separator, and the trailing `Z` tells it to parse as UTC. Without this, some browsers parse the raw string as local time and you get weird results.

---

## The Lead Detail Panel

Create `client/src/components/LeadDetail.js`:

```javascript
// client/src/components/LeadDetail.js
import { useEffect, useState } from "react";
import { getLead, updateLead } from "../services/api";

const STATUSES = ["new", "contacted", "qualified", "converted", "lost"];

export default function LeadDetail({ leadId, onClose, onUpdated }) {
  const [lead, setLead] = useState(null);
  const [saving, setSaving] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    if (!leadId) return;
    let cancelled = false;
    setLead(null);
    setError(null);
    getLead(leadId)
      .then((data) => !cancelled && setLead(data))
      .catch((err) => !cancelled && setError(err.message));
    return () => {
      cancelled = true;
    };
  }, [leadId]);

  // Close on Escape key
  useEffect(() => {
    if (!leadId) return;
    const onKey = (e) => {
      if (e.key === "Escape") onClose?.();
    };
    window.addEventListener("keydown", onKey);
    return () => window.removeEventListener("keydown", onKey);
  }, [leadId, onClose]);

  async function changeStatus(newStatus) {
    setSaving(true);
    setError(null);
    try {
      const updated = await updateLead(leadId, { status: newStatus });
      setLead((prev) => ({ ...prev, ...updated }));
      onUpdated?.();
    } catch (err) {
      setError(err.response?.data?.error || err.message);
    } finally {
      setSaving(false);
    }
  }

  if (!leadId) return null;

  return (
    <div className="fixed inset-0 z-40 flex">
      <div className="flex-1 bg-black/30" onClick={onClose} />
      <aside className="w-full max-w-lg bg-white shadow-xl p-6 overflow-y-auto">
        <div className="flex items-start justify-between">
          <h2 className="text-xl font-semibold text-gray-900">
            {lead?.name || "Loading..."}
          </h2>
          <button
            onClick={onClose}
            className="text-gray-500 hover:text-gray-800"
          >
            Close
          </button>
        </div>

        {error && <div className="mt-3 text-sm text-red-600">{error}</div>}

        {lead && (
          <>
            <div className="mt-4 space-y-1 text-sm">
              <div>
                <span className="text-gray-500">Phone:</span> +{lead.wa_phone}
              </div>
              <div>
                <span className="text-gray-500">Email:</span>{" "}
                {lead.email || "-"}
              </div>
              <div>
                <span className="text-gray-500">Inquiry:</span>{" "}
                {lead.inquiry_type || "-"}
              </div>
              <div>
                <span className="text-gray-500">Created:</span>{" "}
                {lead.created_at}
              </div>
            </div>

            <div className="mt-4">
              <div className="text-sm text-gray-500 mb-1">Status</div>
              <div className="flex flex-wrap gap-2">
                {STATUSES.map((s) => (
                  <button
                    key={s}
                    disabled={saving || lead.status === s}
                    onClick={() => changeStatus(s)}
                    className={
                      "px-3 py-1 rounded-full text-xs font-medium border " +
                      (lead.status === s
                        ? "bg-gray-900 text-white border-gray-900"
                        : "bg-white text-gray-700 border-gray-300 hover:bg-gray-50")
                    }
                  >
                    {s}
                  </button>
                ))}
              </div>
            </div>

            <div className="mt-6">
              <a
                href={`https://wa.me/${lead.wa_phone}`}
                target="_blank"
                rel="noreferrer"
                className="inline-block bg-green-600 text-white text-sm px-4 py-2 rounded-lg hover:bg-green-700"
              >
                Reply via WhatsApp
              </a>
            </div>

            <div className="mt-8">
              <h3 className="text-sm font-semibold text-gray-700 mb-2">
                Conversation
              </h3>
              <div className="space-y-2">
                {lead.messages.map((m) => (
                  <div
                    key={m.id}
                    className={
                      "rounded-lg px-3 py-2 text-sm max-w-[85%] " +
                      (m.direction === "inbound"
                        ? "bg-gray-100 text-gray-900"
                        : "bg-blue-600 text-white ml-auto")
                    }
                  >
                    <div>{m.body}</div>
                    <div
                      className={
                        "text-[10px] mt-1 " +
                        (m.direction === "inbound"
                          ? "text-gray-500"
                          : "text-blue-100")
                      }
                    >
                      {m.created_at}
                    </div>
                  </div>
                ))}
              </div>
            </div>
          </>
        )}
      </aside>
    </div>
  );
}
```

This is the longest single file of the week. Read it in pieces.

**Two `useEffect` hooks.** The first fetches the lead whenever `leadId` changes. The second attaches a keyboard listener for Escape to close the panel. Both have cleanup functions -- the fetch one sets `cancelled`, the keyboard one removes the listener. If you miss those cleanups you get memory leaks and stale-state warnings.

**`changeStatus` is an async handler** that calls the PATCH endpoint, merges the response into the local state, and tells the parent via `onUpdated` so the table row re-fetches too. The `setSaving` flag disables buttons during the request -- standard "don't double-submit" UX.

**The `wa.me/<phone>` link** is a WhatsApp feature with no API call involved. Clicking it opens WhatsApp on the user's device with a chat to that number pre-filled. Zero backend work, huge productivity gain for the sales team.

**The conversation list** renders messages with two visual lanes: inbound messages (grey, left-aligned) and outbound messages (blue, right-aligned). The same visual shorthand WhatsApp itself uses, so nobody has to learn anything.

---

## The Dashboard Page

Create `client/src/pages/Dashboard.js`:

```javascript
// client/src/pages/Dashboard.js
import { useCallback, useEffect, useState } from "react";
import { listLeads } from "../services/api";
import StatsCards from "../components/StatsCards";
import LeadsTable from "../components/LeadsTable";
import LeadDetail from "../components/LeadDetail";

const STATUS_OPTIONS = [
  { value: "", label: "All statuses" },
  { value: "new", label: "New" },
  { value: "contacted", label: "Contacted" },
  { value: "qualified", label: "Qualified" },
  { value: "converted", label: "Converted" },
  { value: "lost", label: "Lost" },
];

export default function Dashboard() {
  const [search, setSearch] = useState("");
  const [status, setStatus] = useState("");
  const [leads, setLeads] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [selectedId, setSelectedId] = useState(null);
  const [refreshKey, setRefreshKey] = useState(0);

  const fetchLeads = useCallback(async () => {
    try {
      setError(null);
      const data = await listLeads({ search, status });
      setLeads(data.leads);
    } catch (err) {
      setError(err.response?.data?.error || err.message);
    } finally {
      setLoading(false);
    }
  }, [search, status]);

  // Re-fetch when filters change or refreshKey bumps
  useEffect(() => {
    fetchLeads();
  }, [fetchLeads, refreshKey]);

  // Poll every 10 seconds
  useEffect(() => {
    const id = setInterval(() => setRefreshKey((k) => k + 1), 10000);
    return () => clearInterval(id);
  }, []);

  return (
    <div className="min-h-screen bg-gray-50">
      <header className="bg-white border-b border-gray-200">
        <div className="max-w-6xl mx-auto px-6 py-4">
          <h1 className="text-2xl font-bold text-gray-900">Mctaba CRM</h1>
          <p className="text-sm text-gray-500">
            WhatsApp lead capture -- Nairobi
          </p>
        </div>
      </header>

      <main className="max-w-6xl mx-auto px-6 py-6 space-y-6">
        <StatsCards refreshKey={refreshKey} />

        <div className="flex flex-col md:flex-row gap-3">
          <input
            type="text"
            value={search}
            onChange={(e) => setSearch(e.target.value)}
            placeholder="Search by name, phone, or email"
            className="flex-1 px-4 py-2 border border-gray-300 rounded-lg text-sm focus:outline-none focus:ring-2 focus:ring-blue-500"
          />
          <select
            value={status}
            onChange={(e) => setStatus(e.target.value)}
            className="px-4 py-2 border border-gray-300 rounded-lg text-sm bg-white"
          >
            {STATUS_OPTIONS.map((opt) => (
              <option key={opt.value} value={opt.value}>
                {opt.label}
              </option>
            ))}
          </select>
        </div>

        {error && <div className="text-sm text-red-600">Error: {error}</div>}
        {loading ? (
          <div className="text-gray-500 text-sm">Loading leads...</div>
        ) : (
          <LeadsTable
            leads={leads}
            onSelect={(lead) => setSelectedId(lead.id)}
          />
        )}
      </main>

      <LeadDetail
        leadId={selectedId}
        onClose={() => setSelectedId(null)}
        onUpdated={() => setRefreshKey((k) => k + 1)}
      />
    </div>
  );
}
```

The dashboard wires everything together. Let us walk through the state:

- `search` and `status` -- the two filter inputs. Whenever either changes, `useCallback` rebuilds `fetchLeads`, the first `useEffect` runs, and the table refetches.
- `leads` -- the current page of results.
- `loading` / `error` -- the same dual-flag pattern you used throughout Week 9.
- `selectedId` -- which lead the detail panel should show (or `null` for closed).
- `refreshKey` -- a number we bump to force re-fetches. Both filter changes and the polling interval can trigger it.

The second `useEffect` runs once on mount (`[]` dependency) and sets up the polling interval. `setInterval` fires every 10,000 ms, bumping `refreshKey` by 1, which triggers the first effect to re-run `fetchLeads`. The return value clears the interval on unmount. This is the standard "interval in React" pattern -- forget the cleanup and you get leaked timers.

When the detail panel updates a lead via PATCH, it calls `onUpdated`, which also bumps `refreshKey` -- so the table in the background refreshes to show the new status the instant you click a button.

---

## Running The Whole System

You now need **three terminals** running at once:

```bash
# Terminal 1 -- backend
cd server
npm run dev

# Terminal 2 -- ngrok (for the WhatsApp webhook)
ngrok http 5000

# Terminal 3 -- frontend
cd client
npm start
```

`npm start` opens `http://localhost:3000` automatically. You should see:

- The "Mctaba CRM" header.
- Stats cards showing real counts from your database.
- A search input and status dropdown.
- A table of whatever leads you captured on Day 2, sorted newest first.

---

## End-To-End Test

The moment of truth. Keep the dashboard open in a browser tab, and do the following:

1. From your phone, message the test number. Type "restart" first if you are reusing a number that already completed the flow.
2. Complete a full conversation: name, email, tap an inquiry type, reply "yes".
3. Switch to the dashboard tab. **Do not refresh.** Wait up to 10 seconds.
4. A new row should appear at the top of the table.
5. The "Total leads" and "New today" counts should increment.
6. Click the new row. The slide-out panel opens. You should see the full conversation, including the interactive list tap.
7. Click "Qualified". The status badge changes to purple. The table row in the background updates.
8. Press `Escape`. The panel closes.
9. Type part of the lead's name into the search box. The table filters to just that lead.
10. Clear the search. Select "Qualified" in the dropdown. Only qualified leads remain.
11. Click the new lead again, then click "Reply via WhatsApp". Your device's WhatsApp opens with a chat to that number.

✅ CHECKPOINT: All eleven steps work. Week 11's core build is complete.

---

## Common Errors

| Error | Cause | Fix |
|---|---|---|
| `has been blocked by CORS policy` in the browser console | `app.use(cors())` missing or below the route registrations in `server/index.js` | `cors()` must come before `app.use("/api/leads", ...)`. Same rule as Week 10. |
| `Network Error` in the dashboard, no request reaches the server | Server not running, or wrong port in `api.js` | Confirm `npm run dev` is running in the server terminal and `baseURL` in `services/api.js` matches. |
| Tailwind classes have no effect (unstyled page) | Content paths wrong in `tailwind.config.js`, or `index.css` missing the three directives | Check `content: ["./src/**/*.{js,jsx}"]` and the `@tailwind base/components/utilities` lines. Restart `npm start` after the first install. |
| The table shows "No leads yet" even though the server has data | Reading `data` instead of `data.leads` in `fetchLeads` | The API returns `{ total, page, pageSize, leads }`. Destructure correctly. |
| Dashboard shows a React error about timestamps on some browsers | SQLite date string not converted to a proper `Date` | Use the `replace(" ", "T") + "Z"` trick shown in `LeadsTable.js`. |
| The detail panel opens blank and never loads | `leadId` prop is not changing, so the effect never runs | Make sure you call `setSelectedId(lead.id)` in the table's `onSelect`, not `setSelectedId(lead)`. |
| Polling appears to do nothing | `setRefreshKey((k) => k + 1)` not using the functional updater form | Always use the functional form inside `setInterval` so you do not close over a stale value. |
| New leads take 30-60 seconds to appear | Polling interval too long, or the bot's webhook is taking too long | 10 seconds is what we set. If the bot side is slow, look for blocking work *before* the `res.sendStatus(200)` in `routes/webhook.js`. |

Good time to commit: `git add . && git commit -m "feat: React + Tailwind CRM dashboard"`

---

## Part 5: Stretch Goals (Pick Whatever Interests You)

None of these are required for a passing build. Do the ones that sound fun or that you can imagine the sales team actually wanting.

### A `notes` Field In The Detail Panel

The `notes` column already exists on `leads`, and the PATCH route already accepts a `notes` field. Add a `<textarea>` to the detail panel that saves on blur by calling `updateLead(leadId, { notes: e.target.value })`. Two lines of JSX, one line of handler. The sales team will use this constantly.

### Assign Leads To Team Members

Add a text input to the detail panel that writes to `assigned_to` via the existing PATCH route. Show the assignee as a small avatar or initials in the table column. Full user accounts wait until Week 12 (Phase 2 continues with authentication), but "assign a name string" is a useful first step -- the backend was built for it.

### Export Leads To CSV

Add `GET /api/leads/export.csv` on the server. Query all leads, build a CSV string with plain string concatenation (no package needed at this size), set `Content-Type: text/csv` and `Content-Disposition: attachment; filename="leads-<date>.csv"`, and return it. Add a "Download CSV" button on the dashboard. The sales team lives in Excel -- they will love you.

### Last Message Preview In The Table

Add the last inbound message to each row in the table (truncated to 60 characters). Requires updating `GET /api/leads` to include a subquery for the most recent message per lead. More useful than it sounds -- the receptionist can triage without opening each lead.

### Server-Sent Events For Real-Time Leads

Replace polling with SSE. Add a `GET /api/leads/stream` route on the server that sets `Content-Type: text/event-stream` and holds the connection open, pushing an event whenever a new lead is created (have `bot.js` notify a shared emitter). On the client, use the built-in `EventSource` API instead of `setInterval`. New leads appear in under 100ms instead of up to 10 seconds. This is the right upgrade when the team tells you polling feels laggy.

### Light / Dark Theme Toggle

Tailwind has a `dark:` variant. Toggle a `dark` class on the `<html>` element and prefix every colour utility with `dark:bg-gray-900`, `dark:text-gray-100`, etc. Good practice for everywhere else you will build UI.

---

## What You Built This Week

You now have a complete, two-sided WhatsApp Lead Capture CRM:

- **The bot.** A WhatsApp number running on Meta's Cloud API, verified webhook, HMAC-signed requests, a state machine that tracks a multi-turn conversation in SQLite, text and interactive-list messages, graceful handling of voice notes and garbage input, and a restart command.
- **The REST API.** Pagination, search across three columns, status filtering, lead detail with full conversation history, PATCH updates with enum validation, and an aggregate stats endpoint.
- **The dashboard.** React + Tailwind with live stats cards, a searchable filterable table, colour-coded status badges, a slide-out detail panel with one-click status updates, a keyboard shortcut, a direct reply link, and 10-second polling so new leads appear without a refresh.

### How This Connects To Real Businesses

Every piece maps onto something a Nairobi business actually needs. The bot is the reason leads stop falling through the cracks -- by the time a salesperson looks at the dashboard, every inquiry already has a name, email, and category. The search and filter are what turn a pile of WhatsApp chats into a workable list. The status pipeline is what lets a sales lead hold the team accountable: "why are there 14 leads in 'new' older than 48 hours?" is a question you can now *ask* instead of just feel.

Point your test number at a friend who runs a real estate office in Kileleshwa or a car yard on Ngong Road and let them try it for an afternoon. One hour of real feedback beats a week of polishing.

### What's Next

**Week 12** layers authentication on top of this CRM. Real accounts for sales team members, login and session handling, and per-user `assigned_to` that actually means something. The backend you wrote this week already has the column waiting for it -- we built it that way on purpose.

The WhatsApp Lead Capture CRM is your Phase 2 anchor project. Come back to it when Week 12 adds auth, when later weeks add deployment, and when you eventually pick it up as a portfolio piece for interviews. Everything you put in this week compounds from here.
