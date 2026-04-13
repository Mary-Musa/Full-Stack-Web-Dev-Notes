# Lesson 1.1: Introduction to JavaScript & Development Environment

> **AI boundaries this week:** 80% manual / 20% AI. Habit: *The 15-minute rule -- try for 15 minutes before you prompt.* See [ai.md](../ai.md).

## Learning Objectives
By the end of this lesson, you will be able to:
- Explain what JavaScript is and its role in web development.
- Understand the difference between running JavaScript in the browser vs. on the server (Node.js).
- Set up a professional development environment using Visual Studio Code.
- Scaffold your first JavaScript project.
- Write and execute your first JavaScript code.

---

## 1. What is JavaScript?
JavaScript (JS) is a versatile programming language initially created to make web pages interactive. Today, it is one of the most popular programming languages in the world, capable of powering both the front-end (what users see) and the back-end (the server logic).

### JavaScript in the Browser (Client-Side)
JavaScript is often called the "language of the web." Every modern web browser (Chrome, Firefox, Safari, Edge) has a built-in **JavaScript engine** (like V8 in Google Chrome) that reads and executes JavaScript code.

In the browser, JavaScript allows you to:
- **Manipulate the DOM (Document Object Model):** Change HTML and CSS dynamically (e.g., hiding a menu, updating text based on user input).
- **Handle User Events:** Respond to clicks, key presses, and mouse movements.
- **Communicate with Servers (HTTP/AJAX/Fetch):** Load new data without refreshing the page (e.g., fetching new tweets or loading more images).

### JavaScript on the Server (Node.js)
**Node.js** is a runtime environment that uses Chrome's V8 engine to run JavaScript *outside* the browser environment.

With Node.js, JavaScript can:
- Read and write files to your computer's file system.
- Connect to databases (like MongoDB or PostgreSQL).
- Build back-end APIs, microservices, and web servers.
- Run powerful build tools and automation scripts.

> **Key Takeaway:** JavaScript in the browser controls the user interface and interactions. JavaScript in Node.js controls the server hardware and backend logic.

---

## 2. Setting Up Your Editor: Visual Studio Code

To write code efficiently, we need a good code editor. We will use **Visual Studio Code (VS Code)**, which is the industry standard for web development.

### Installation
1. Download and install VS Code from [code.visualstudio.com](https://code.visualstudio.com/).
2. Open the application.

### Recommended Extensions
Extensions add super-powers to your editor. Open the Extensions pane (`Ctrl+Shift+X` on Windows/Linux or `Cmd+Shift+X` on Mac) and install the following:

| Extension Name | ID | Purpose |
| -------------- | -- | ------- |
| **ESLint** | `dbaeumer.vscode-eslint` | Linting. It acts as a spell-checker for your code, catching bugs and syntax errors before you even run them. |
| **Prettier - Code formatter** | `esbenp.prettier-vscode` | Automatically formats your code (spacing, quotes, etc.) so it always looks clean and consistent. |
| **JavaScript (ES6) code snippets** | `xabikos.javascriptsnippets`| Provides shortcuts to quickly type common JavaScript commands, saving time. |

> **Common Mistake:** Forgetting to enable "Format on Save" in VS Code. Go to Settings (`Cmd+,` or `Ctrl+,`), search for "Format on Save", and check the box. Ensure Prettier is selected as your default formatter. This will auto-organize your code every time you save!

---

## 3. Creating Your First Project

Let's set up a workspace for our code. Organization is key for a developer.

1. **Create a folder** on your computer named `js-foundations`.
2. **Open the folder in VS Code:** Go to `File` → `Open Folder...` (or drag and drop the folder into the VS Code window).

### Project Structure
A standard beginner JavaScript project consists of an HTML entry point and an external JavaScript file.

```text
js-foundations/
├── index.html   # The structure of your web page
└── main.js      # The logic and interactivity
```

---

## 4. How to Connect JavaScript to HTML

There are two main ways to run JavaScript in the browser: **Inline** and **External**.

### Approach 1: Inline Script (Not Recommended)
You can write JavaScript directly inside your HTML file using the `<script>` tag.

```html
<script>
  console.log('This is an inline script!');
</script>
```
*Why avoid this?* As your code grows, putting thousands of lines of JavaScript inside HTML makes it incredibly hard to read, format, and maintain.

### Approach 2: External Script (Recommended)
You write your JavaScript in a separate `.js` file and link it inside your HTML.

```html
<script src="main.js"></script>
```

**Why is this much better?**
- **Separation of Concerns:** HTML handles structure, CSS handles design, and JS handles logic.
- **Reusability:** Multiple HTML pages can link to the very same JS file.
- **Performance:** Browsers cache external files, making your website load faster on return visits.
- **Easier Tooling:** Linters (like ESLint) and formatters work flawlessly on dedicated `.js` files.

---

## 5. Writing Your First Code

Let's create our files! Inside your `js-foundations` folder, create the following two files:

### `index.html`
```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <title>JS Foundations – Lesson 1.1</title>
</head>
<body>
  <h1>Lesson 1.1: Hello, JavaScript!</h1>

  <!-- Include your JS at the end of the body so the HTML elements load before the JS runs! -->
  <script src="main.js"></script>
</body>
</html>
```

### `main.js`
```javascript
// This tells the browser to print text into the Developer Console
console.log('Hello, JS!');
```

---

## 6. Activity: Run and Debug Your Code

Now, let's see our code in action and learn how to inspect and fix it.

### Step 1: Open the App
1. Open `index.html` in your Chrome browser. (You can drag the file from your folder directly into an open Chrome tab, or use an extension like Live Server).
2. You should see "Lesson 1.1: Hello, JavaScript!" on the screen.

### Step 2: Open Developer Tools (DevTools)
DevTools is your best friend as a web developer. It allows you to see errors, inspect elements, and run JS on the fly.
- **Windows/Linux:** Press `F12` or `Ctrl+Shift+I`
- **Mac:** Press `Cmd+Option+I` (`⌥⌘I`)

### Step 3: Check the Console
Click on the **Console** tab in DevTools. You should see the following printed:
`Hello, JS!`
*If you don't see it, ensure you saved your `main.js` and `index.html` files, and that your `<script>` tag is spelled correctly.*

### Step 4: Setting a Breakpoint (Debugging)
Breakpoints allow you to pause your code mid-execution to inspect variables and see exactly what the computer is doing step-by-step.

1. In DevTools, switch to the **Sources** tab.
2. In the left panel (file tree), find and click on `main.js`.
3. Click the line number (e.g., line 2) where your `console.log` is. A marker will appear. This is your breakpoint.
4. **Reload the page** (`Ctrl+R` or `Cmd+R`).
5. Notice how the page freezes? The execution is paused right before running that line.
6. Use the **Step Over** button (`F10`) in the DevTools toolbar to run the line and watch the output appear in the console.

> **Pro Tip:** Debugging using breakpoints is vastly superior to just guessing what went wrong in your code. Getting comfortable with the Sources panel early on will save you hundreds of hours in the future!

---

## 7. Node.js Verification (Optional)

If you want to see JavaScript run without a browser, you can use Node.js to execute the file directly.

*Assuming you have Node installed (download at [nodejs.org](https://nodejs.org/)):*
Open your terminal in VS Code (`Ctrl+~` or `Cmd+~`), ensure you are deeply inside your folder, and run:

```bash
cd js-foundations
node main.js
```

You should see `Hello, JS!` printed directly in your terminal. This demonstrates that JavaScript code can be executed natively on your operating system!

---

## 8. Checkpoint Assignment: Professional Setup

To wrap up, let's configure your project folder professionally using **npm** (Node Package Manager). We will set up ESLint and Prettier for the project.

### Instructions

1. **Initialize your project:**
   Open the VS Code integrated terminal (`Ctrl+~` or `Cmd+~`) and run:
   ```bash
   npm init -y
   ```
   *(This creates a `package.json` file which keeps track of your project settings and installed packages).*

2. **Install Dev Dependencies:**
   Run the following command to install our formatting and linting tools:
   ```bash
   npm install --save-dev eslint prettier
   ```

3. **Configure ESLint:**
   Create a new file named `.eslintrc.json` in your project folder and paste the following:
   ```json
   {
     "env": {
       "browser": true,
       "es2021": true,
       "node": true
     },
     "extends": "eslint:recommended",
     "parserOptions": {
       "ecmaVersion": 12,
       "sourceType": "module"
     },
     "rules": {}
   }
   ```

4. **Configure Prettier:**
   Create another new file named `.prettierrc` (notice it starts with a dot) and paste:
   ```json
   {
     "semi": true,
     "singleQuote": true,
     "printWidth": 80
   }
   ```

5. **Verify your setup:**
   Open your `main.js` file. Try making a mess (remove a semicolon, add weird spacing). Save the file. If everything is set up correctly in VS Code, Prettier will snap your code back into perfect formatting instantly!

---

**Next Steps:** In the next lesson, we will dive into JavaScript variables, data types, and operators!
