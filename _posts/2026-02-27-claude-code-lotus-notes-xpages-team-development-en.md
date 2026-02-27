---
layout: post
title: "Using Claude Code to develop Lotus Notes and XPages with Team Development"
date: 2026-02-27
categories: [domino, xpages, claude-code, team-development, git]
lang: en
---

If you develop on HCL Domino and have not yet tried using AI to assist with LotusScript, Java, XPages, and formula programming, this post will show you how we integrated **Claude Code** into the Domino development workflow using **Team Development** and **Git**.

The result: an AI assistant that understands the complete context of your Domino application — forms, views, agents, XPages, script libraries — and can suggest, fix, and create code directly in the project files.

## Prerequisites

- HCL Domino Designer (with Team Development support)
- Git installed on your machine
- GitHub account (or GitLab, Bitbucket)
- Claude Code installed (`npm install -g @anthropic-ai/claude-code`)

## The complete workflow

```
NSF (Domino Designer)
    |
    v
[1] Team Development: Export NSF → ODP (On-Disk Project)
    |
    v
[2] Create a Git repository in the ODP directory
    |
    v
[3] Commit + Push to GitHub
    |
    v
[4] Open Claude Code in the project directory
    |
    v
[5] Code with AI assistance (LotusScript, Java, XPages, formulas)
    |
    v
[6] Commit + Push changes
    |
    v
[7] Sync ODP back to Domino Designer
    |
    v
[8] Clean + Build (for XPages)
    |
    v
[9] Test the application
```

## Step 1: Export the NSF to ODP via Team Development

In Domino Designer:

1. Open the NSF database you want to version control
2. Go to **File → Application → Properties**
3. On the **Design** tab, check **"Enable source control integration"** (if not already enabled). When you click OK, the Designer will create the ODP in a local directory
4. The Designer asks you to choose a destination directory. Choose a location like:

```
C:\projects\my-application\ntf\
```

After the export, the directory will have the typical ODP structure:

```
ntf/
├── AppProperties/
├── Code/
│   ├── Agents/
│   ├── ScriptLibraries/
│   └── Java/
├── CustomControls/
├── Forms/
├── Pages/
├── Resources/
│   ├── Files/
│   ├── Images/
│   └── StyleSheets/
├── SharedElements/
│   └── Subforms/
├── Views/
├── XPages/
└── META-INF/
    └── MANIFEST.MF
```

Each design element becomes one or more files on disk:
- **Forms** → `.form` files
- **Views** → `.view` files
- **Agents** → `.lsa` (LotusScript), `.ja` (Java), `.fa` (Formula) files
- **Script Libraries** → `.lss` (LotusScript), `.javalib` (Java) files
- **XPages** → `.xsp` files
- **Custom Controls** → `.xsp` files

## Step 2: Create the Git repository

In the terminal, inside the project directory:

```bash
cd C:\projects\my-application
git init
```

Create a `.gitignore` to exclude unnecessary files:

```
# .gitignore for Domino ODP projects
*.nsf
*.ntf
*.ns5
*.ns6
*.ns7
*.ns8
build/
local.properties
plugin.xml
.project
.classpath
.settings/
```

Make the initial commit:

```bash
git add .
git commit -m "Initial ODP export from Domino Designer"
```

Create the repository on GitHub and push:

```bash
git remote add origin https://github.com/your-username/my-application.git
git push -u origin main
```

## Step 3: Open Claude Code in the project

With the repository ready, open Claude Code in the project directory:

```bash
cd C:\projects\my-application
claude
```

Claude Code will index all ODP files and understand the Domino application structure. It can read and modify:

- **LotusScript** in agents (`.lsa`) and script libraries (`.lss`)
- **Java** in agents (`.ja`) and Java libraries (`.javalib`)
- **XPages XML** in `.xsp` files
- **JavaScript (SSJS/CSJS)** embedded in XPages
- **Formulas** in views, forms, and agents
- **Properties** of forms and views (DXL/XML)

### Example: ask for help with a LotusScript agent

```
> Analyze the agent "SendEmail" in Code/Agents/SendEmail.lsa and suggest
  improvements for error handling and logging
```

### Example: create an XPage Custom Control

```
> Create a Custom Control called ccDatePicker.xsp that uses the
  xe:djDateTextBox component with date validation in dd/MM/yyyy format
```

### Example: optimize a view formula

```
> The view "SalesByMonth" in Views/SalesByMonth.view is slow.
  Analyze the selection formula and columns and suggest optimizations
```

### Example: migrate LotusScript to Java

```
> Convert the LotusScript agent "ProcessImport.lsa" to an equivalent
  Java agent, maintaining the same business logic
```

## Tip: create a CLAUDE.md in the project

The `CLAUDE.md` file at the project root provides permanent context to Claude Code. For Domino projects, include:

```markdown
# Project: My Application

## Type
HCL Domino/Notes application with XPages

## Structure
- ntf/ - On-Disk Project (ODP) exported from Domino Designer
- Forms: main data entities
- Views: indexes and queries
- Agents: automated business logic
- XPages: web interface
- Custom Controls: reusable components

## Conventions
- LotusScript follows the existing naming pattern
- Scheduled agents use prefix "Sched_"
- Internal views use prefix "_intra"
- XPages use the project's custom control framework

## Server
- Version: HCL Domino 14.0
- XPages runtime active

## Notes
- Do not modify MANIFEST.MF manually
- Forms and views are XML (DXL) - edit carefully
- Always test via Clean + Build before deploy
```

## Step 4: Code and commit

After Claude Code makes changes to the ODP files, review and commit:

```bash
git add .
git commit -m "Add error handling to SendEmail agent"
git push
```

## Step 5: Sync back to Domino Designer

Now comes the part that closes the loop. Changes made by Claude Code to the ODP files need to go back to the NSF in Domino Designer:

1. Open **Domino Designer**
2. Open the NSF database linked to the ODP
3. The Designer should detect that files on disk have changed
4. If it does not detect automatically, right-click the project and select **Team Development → Sync with On-Disk Project**
5. The Designer will import the ODP changes into the NSF

## Step 6: Clean and Build (for XPages)

If you modified XPages or Custom Controls, you need to Clean and Build:

1. In Domino Designer, go to **Project → Clean**
2. Select the project and click **OK**
3. Wait for the clean to complete
4. Go to **Project → Build All** (or automatic build takes care of it)

Clean removes previously compiled artifacts. Build recompiles all XPages, Custom Controls, and Java code. Without this step, you may see old versions of the code running.

**Important**: if you only changed LotusScript or formulas (no XPages), Clean/Build is not necessary — changes take effect immediately after synchronization.

## Step 7: Test

- **XPages**: open your browser and navigate to the XPage URL (e.g., `http://domino-server/app.nsf/page.xsp`)
- **Notes Client**: open the database in the Notes client to test forms, views, and agents
- **Scheduled agents**: check the server log or run manually from the Designer

## Day-to-day workflow

After the initial setup, the development cycle looks like this:

```
1. Open Claude Code in the project directory
2. Describe what you need (fix, new feature, refactoring)
3. Claude Code edits the ODP files
4. Review changes (git diff)
5. Commit + Push
6. In Designer: Sync with On-Disk Project
7. Clean + Build (if XPages)
8. Test
9. Go back to step 1
```

## What works well with Claude Code + Domino

| Task | Effectiveness |
|---|---|
| Analyze and fix LotusScript | Excellent — understands the language and context |
| Create/modify XPages XML | Very good — generates valid XHTML with XSP components |
| Optimize view formulas | Good — suggests better approaches for selection and columns |
| Create Java agents | Excellent — generates complete Java code with imports |
| Refactor Script Libraries | Very good — reorganizes and modularizes code |
| Write SSJS for XPages | Good — knows the XPages API (SSJS runtime) |
| Edit form/view properties (DXL) | With care — DXL is sensitive to formatting |

## Important precautions

- **Always back up the NSF** before syncing extensive changes
- **Do not manually edit MANIFEST.MF** — it is managed by the Designer
- **Review DXL** for forms and views carefully — a wrong attribute can corrupt the element
- **Test in a development environment** before deploying to production
- **Clean deletes and rebuilds** — it may take a few minutes on large projects

---

Team Development in Domino Designer, combined with Git and Claude Code, transforms Notes/XPages development into a modern experience without leaving the platform. Code is versioned, history is traceable, and you have an AI assistant that understands LotusScript, Java, XPages, and Domino formulas.
