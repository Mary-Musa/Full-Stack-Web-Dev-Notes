# Week 8 - Day 3 Assignment

## Title
React Router -- Multi-Page Apps With Dynamic Routes

## Overview
Today you add routing. Your assignment is to take your Task Manager (or any multi-page idea) and give it three routes: a home list, a detail page for a single task, and an about page. You will use React Router v6 and meet `useNavigate` and `useParams`.

## Learning Objectives Assessed
- Install and configure React Router v6
- Define routes with `createBrowserRouter` or `<BrowserRouter>`
- Use `<Link>` for internal navigation
- Use dynamic route parameters (`:id`)
- Use `useNavigate` to navigate programmatically

## Prerequisites
- Days 1-2 completed

## AI Usage Rules

**Ratio:** 55/45. **Habit:** AI as pair partner. See [../ai.md](../ai.md).

- **ALLOWED FOR:** Pairing on the route definition structure after you made an initial attempt.
- **NOT ALLOWED FOR:** Generating the entire routing setup.
- **AUDIT REQUIRED:** Yes.

## Tasks

### Task 1: Install and mount React Router

**What to do:**
```bash
npm install react-router-dom
```

In `src/main.jsx` (or `index.jsx`), wrap your `App` in a router:

```jsx
import { BrowserRouter } from "react-router-dom";

ReactDOM.createRoot(document.getElementById("root")).render(
  <BrowserRouter>
    <App />
  </BrowserRouter>
);
```

**Expected output:**
App still renders. No routing yet but Router is wired in.

### Task 2: Define three routes

**What to do:**
In `App.jsx`:

```jsx
import { Routes, Route, Link } from "react-router-dom";

function Home() { return <h1>Home</h1>; }
function About() { return <h1>About</h1>; }
function TaskDetail() { return <h1>Task Detail</h1>; }

function App() {
  return (
    <div>
      <nav>
        <Link to="/">Home</Link> | <Link to="/about">About</Link>
      </nav>
      <Routes>
        <Route path="/" element={<Home />} />
        <Route path="/about" element={<About />} />
        <Route path="/tasks/:id" element={<TaskDetail />} />
      </Routes>
    </div>
  );
}
```

**Expected output:**
Clicking Home and About changes the visible component without a full page reload.

### Task 3: Use useParams for dynamic routes

**What to do:**
In the `TaskDetail` component, read the `:id` from the URL with `useParams`:

```jsx
import { useParams } from "react-router-dom";

function TaskDetail() {
  const { id } = useParams();
  return <h1>Task {id}</h1>;
}
```

Visit `/tasks/42` in the browser. See "Task 42".

**Expected output:**
Different `:id` values show different text.

### Task 4: Programmatic navigation with useNavigate

**What to do:**
On the Home page, add a button that navigates to a specific task detail:

```jsx
import { useNavigate } from "react-router-dom";

function Home() {
  const navigate = useNavigate();
  return (
    <div>
      <h1>Home</h1>
      <button onClick={() => navigate("/tasks/1")}>Go to task 1</button>
    </div>
  );
}
```

**Expected output:**
Clicking the button navigates to `/tasks/1`.

### Task 5: Connect to your Task Manager

**What to do:**
Extend: make the Task Manager's Home list of tasks clickable. Each task links to `/tasks/:id`. The TaskDetail page reads the id from URL and displays the corresponding task (from the same state -- use context or lift state up, your choice).

```jsx
<Link to={`/tasks/${task.id}`}>{task.text}</Link>
```

**Expected output:**
Clicking a task in the list opens its detail page.

## Stretch Goals (Optional - Extra Credit)

- Add a 404 page for unknown routes: `<Route path="*" element={<NotFound />} />`
- Use nested routes: `/tasks/:id/edit` as a child of `/tasks/:id`.
- Add a loader that pre-fetches data for the route (React Router v6.4+ data APIs).

## Submission Requirements

- **What to submit:** Repo with routing set up, `AI_AUDIT.md`.
- **Deadline:** End of Day 3

## Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| Router installed and wired | 10 | BrowserRouter wraps App. |
| Three routes defined | 20 | Home, About, Task Detail all reachable. |
| Links navigate without reload | 10 | Link components used, no full page reload. |
| useParams reads :id | 15 | Different IDs show different content. |
| useNavigate programmatic | 15 | Button-triggered navigation works. |
| Task Manager integration | 20 | Tasks are clickable and open their detail. |
| Clean commits | 5 | Conventional messages. |
| AI Audit | 5 | Pair conversation if used. |
| **Total** | **100** | |

## Common Mistakes to Avoid

- **Using `<a href>` instead of `<Link to>`.** Plain anchor tags cause full page reloads, losing state. Always use `<Link>` for internal navigation.
- **Forgetting to wrap in BrowserRouter.** Routes only work inside a Router. If hooks throw "useNavigate must be used within a Router", this is usually why.
- **Mixing v5 and v6 syntax.** React Router v6 uses `element={<Comp />}`, not `component={Comp}`. Check your version.

## Resources

- Day 3 reading: [Routing and Multi-Page React Apps.md](./Routing%20and%20Multi-Page%20React%20Apps.md)
- Week 8 AI boundaries: [../ai.md](../ai.md)
- React Router docs: https://reactrouter.com/en/main
