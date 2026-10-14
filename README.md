# Kanban6

IT PMO Kanban board demo (single-file vanilla HTML/CSS/JS). It's a training and demo tool for the internal IT Project Management Office of a **fictitious** bank. It is not affiliated with, endorsed by, or an official system of any real bank.

**Live demo:** https://merrickkw.github.io/Kanban6/

![IT PMO Kanban board showing Backlog, In Progress, Blocked and Done columns with seeded demo tasks](docs/screenshot.png)

## Features

- **Four columns:** Backlog, In Progress, Blocked, Done. Each column shows a count of the tasks currently visible in it.
- **Add tasks** through a dialog with validation: title, description, project/workstream, category, assignee, priority, due date and status. Task IDs follow the format `UOB-ITPM-####`.
- **Move tasks** by drag and drop, or with the "Move ▸" menu on each card (works with the keyboard).
- **Delete tasks**, with a confirmation step.
- **Filters** by project/workstream, assignee (text match) and priority.
- **Large KPI tiles** for completion %, Backlog, In Progress, Blocked, Done and Overdue, across all tasks.
- **Status colour coding:** each status has its own colour and icon on columns, card edges, status chips and charts.
- **Project status visuals:** a status donut, an overall delivery bar and a stacked status bar per project (inline SVG).
- **Three designs** selectable from the header dropdown: Professional, Fun and Futuristic. The choice is memory only and resets on refresh.
- **Overdue highlighting** on cards whose due date has passed. Seed due dates are relative to today, so some are always overdue.
- **Email notification** via [FormSubmit](https://formsubmit.co/) when a task is added. If sending fails, you only see a warning; the board keeps working.

## Run locally

Open `index.html` in a browser. There's no build step, server or install.

To turn on email notifications, change the address in `FORMSUBMIT_ENDPOINT` near the top of the `<script>` in `index.html`. FormSubmit sends a one-time activation email to that address the first time it's used. Note: any address you put there is publicly visible in the repo and on the live site.

## No persistence

Board data lives in memory only. Refreshing the page resets it to the seeded demo data. The app doesn't use localStorage, sessionStorage, IndexedDB or cookies.

## Tech constraints

- Vanilla HTML, CSS and JavaScript in one file (`index.html`).
- No frameworks, bundlers, npm packages, CDN scripts, web fonts or image files. It uses the system font stack and inline SVG/Unicode for icons.
- The only network call is the FormSubmit AJAX endpoint.
- Hardened: a strict Content-Security-Policy meta tag pins the inline script and style by SHA-256 hash and limits connections to formsubmit.co; input is sanitised and validated, and task creation is rate limited and capped. After editing the inline script or style, recompute the hashes in the CSP meta tag.

## Deployment

The site is deployed to GitHub Pages by the GitHub Actions workflow in [`.github/workflows/pages.yml`](.github/workflows/pages.yml). It runs on every push to `main`, or manually from the Actions tab. Only `index.html` is published.

## Disclaimer

All names, projects, tasks and data are fictitious. This is for demo and training use only.
