# Full Stack Web Development Marathon: Onboarding & Tool Setup Guide

> **AI boundaries this week:** 90% manual / 10% AI. Habit: *Ask AI to explain, not to do.* Read [ai.md](../ai.md) before starting -- it explains exactly what you can and cannot use AI for this week.

Welcome to the **Full-Stack Web Development Marathon**!

This guide is designed to take you from "zero to coding" by helping you set up a professional development environment. We'll be using the same tools used by industry professionals: **Visual Studio Code**, **Git**, **GitHub**, and **Node.js**.

---

## 1. Setting Up Your Development Environment

A craftsman is only as good as their tools. Let's ensure your "digital workshop" is ready for action.

### 1.1 Visual Studio Code (VS Code)
**What is it?** VS Code is the world's most popular code editor. It's lightweight, fast, and highly customizable.

**Why use it?** It offers "IntelliSense" (smart code completion), built-in terminal access, and thousands of extensions that make coding easier and faster.

**Installation Steps:**
1. **Download:** Visit [code.visualstudio.com](https://code.visualstudio.com/) and download the installer for your OS (Windows, macOS, or Linux).
2. **Install:** Run the installer and follow the prompts. (Tip: On Windows, check "Add to PATH" if prompted).
3. **Essential Extensions:** Open VS Code, click the Extensions icon (four squares on the left), and install these:
   - **Prettier:** Automatically cleans up your code formatting when you save.
   - **ESLint:** Points out potential errors in your JavaScript.
   - **Live Server:** Lets you see your web page changes in real-time as you code.
   - **Tailwind CSS IntelliSense:** Provides autocompletion for styling your apps.

### 1.2 Git (Version Control)
**What is it?** Git is like a "Save Game" system for your code. It tracks every change you make, allowing you to go back in time if something breaks.

**Installation Steps:**
1. **Download:** Get the latest version from [git-scm.com](https://git-scm.com/).
2. **Setup:** Open your terminal (or "Git Bash" on Windows) and tell Git who you are:
   ```bash
   git config --global user.name "Your Name"
   git config --global user.email "your-email@example.com"
   ```
3. **Why it matters:** Without Git, you'd be saving files like `index_v1.html`, `index_final.html`, `index_REALLY_final.html`. Git solves this by keeping a clean history.

### 1.3 Node.js & NPM
**What is it?** Node.js allows you to run JavaScript on your computer (instead of just inside a web browser). **NPM** (Node Package Manager) comes with it and lets you download tools and libraries created by other developers.

**Installation Steps:**
1. **Download:** Go to [nodejs.org](https://nodejs.org/) and choose the **LTS (Long Term Support)** version.
2. **Verify:** Open your terminal and type:
   ```bash
   node -v
   npm -v
   ```
   If you see version numbers, you're good to go!

---

## 2. Mastering GitHub

GitHub is where your code lives in the cloud. It's the standard for collaborating with other developers.

### 2.1 The Workflow
1. **Create a Repository:** Think of this as a project folder on GitHub.
2. **Clone:** Download that folder to your computer.
3. **Commit:** Save a "snapshot" of your work locally.
4. **Push:** Upload your local snapshots to GitHub.

### 2.2 Your First Repository
1. Log in to [GitHub](https://github.com/).
2. Click the **+** icon -> **New repository**.
3. Name it `my-first-project`, check "Add a README file", and click **Create**.
4. To bring it to your computer, click the green **Code** button, copy the URL, and run this in your terminal:
   ```bash
   git clone https://raw-url-here
   ```

---

## 3. High-Level Overview: The Full-Stack

As a Full-Stack developer, you work on both the "Front" and the "Back" of an application.

| Layer | Technology | Role |
| :--- | :--- | :--- |
| **Front-End** | HTML, CSS, JavaScript | The "Face" - What the user sees and clicks on. |
| **Back-End** | Node.js, Express | The "Brain" - Handles logic, security, and data processing. |
| **Database** | SQL (Postgres) or NoSQL (MongoDB) | The "Memory" - Where user accounts and posts are stored. |

### 3.1 The Front-End Trio
- **HTML:** The skeleton (headings, paragraphs, buttons).
- **CSS:** The skin and clothes (colors, fonts, layouts).
- **JavaScript:** The muscles (animations, form handling, fetching data).

---

## 4. Understanding the DOM (Document Object Model)

The DOM is the most important concept in Front-End development. It is how JavaScript "talks" to your HTML.

Imagine your HTML as a tree:
- The `<html>` tag is the root.
- The `<body>` and `<head>` are branches.
- Every `<p>`, `<h1>`, and `<div>` are leaves.

**Common JavaScript DOM Commands:**
- `document.querySelector('#id')`: Find an element by its ID.
- `element.textContent = 'Hello'`: Change the text inside an element.
- `element.style.color = 'red'`: Change the color of an element.
- `element.addEventListener('click', ...)`: Make something happen when the user clicks!

---

## 5. Next Steps

Now that your tools are ready, here is your mission:
1. **Hello World:** Create a file called `index.html`, add some text, and open it using VS Code's **Live Server**.
2. **Daily Practice:** Spend 30 minutes every day coding. Consistency is the secret sauce.
3. **Join the Community:** Don't learn in a vacuum. Ask questions in our Discord/Slack channels!

**Happy Coding!**