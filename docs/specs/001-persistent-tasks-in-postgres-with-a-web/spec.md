---
spec_id: 001-persistent-tasks-in-postgres-with-a-web
title: Persistent tasks in Postgres with a web task list
status: done
priority: P1
shape: gitops-app
---

# 001 — Persistent tasks in Postgres with a web task list

## Problem

The Task API keeps tasks in its own memory. Every restart or redeploy loses them, and two running instances each show a different board. People also have no way to use the board except by calling the API directly, because the website has no task features yet. This spec makes tasks durable and shared, stored in a Postgres database, and gives people a web page where they can see the board and add tasks to it.

## Stories

As a team member, I want tasks to survive restarts and redeploys, so that the board I rely on doesn't empty itself.
**Repos:** taskboard21-api, taskboard21-gitops

As a team member, I want to see all tasks on a web page, so that I can read the board without calling the API.
**Repos:** taskboard21-web, taskboard21-api, taskboard21-gitops

As a team member, I want to add a task with a title and an optional due date from the web page, so that I can record work without calling the API.
**Repos:** taskboard21-web, taskboard21-api

## Acceptance criteria

- **AC-001.1** Given a task created through `POST /tasks` or the web page, when the Task API is restarted or redeployed, then `GET /tasks` still returns that task with the same `id`, `title` and `due_date` it had when it was created, in the same position in creation order.
- **AC-001.2** Given the Task API runs as two instances in one environment, when a task is created through one instance, then `GET /tasks` served by either instance includes it, and no two tasks in the environment share an `id`.
- **AC-001.3** Given tasks A, B and C exist, created in that order, when a person opens the website's start page, then it shows the three tasks in creation order, each with its title and its due date as `YYYY-MM-DD`, or "No due date" when it has none.
- **AC-001.4** Given no task exists in the environment, when a person opens the website's start page, then it shows the text "No tasks yet" and the form for adding a task.
- **AC-001.5** Given the start page is open, when a person enters the title "Buy milk", optionally a due date, and submits the form, then within 2 seconds the task appears at the end of the list without a manual page reload, the form is emptied, and `GET /tasks` returns the task with the same title and due date.
- **AC-001.6** Given the start page is open, when a person submits the form with a title that is empty, only whitespace, or longer than 200 characters, then the page shows "Title is required" or "Title must be at most 200 characters" next to the title field, keeps what was typed, and no task is created.
- **AC-001.7** Given the Task API cannot reach its database, when a client sends `GET /tasks` or `POST /tasks`, then the response is 503 with a JSON error body, no task is created, and the API's readiness check reports not ready until the database is reachable again; once it is, all tasks created before the outage are listed again.
- **AC-001.8** Given the Task API answers with an error or cannot be reached, when a person opens the start page or submits the form, then the page shows "Tasks are unavailable right now. Please try again later." instead of the list or above the form, keeps what was typed in the form, and the page itself still loads.

## Non-goals

- Reading, editing, completing or deleting a single task, in the API or on the web page.
- Filtering, sorting options, search and pagination.
- Fields beyond title and due date.
- Users, sign-in and ownership: anyone who can reach the website or the API may see and add tasks (as decided in the Task API's spec 001).
- Carrying over tasks held in a running API instance's memory at the switch-over: they are lost once, as they are on every restart today.
- Database backups and restore, and running the database in staging or prod; the database is declared for dev, like the components today.
- Live updates of the page when someone else adds a task (a reload shows them).

## Assumptions

- **A1** The Task API's contract stays as it is: `POST /tasks` and `GET /tasks`, fields `id`, `title`, `due_date`, the same validation and 422 errors; only where tasks are kept changes (other option: a new versioned API; affects AC-001.1, AC-001.2, AC-001.5).
- **A2** The website talks to the Task API, never to the database directly; the database is used by the Task API alone (other option: the website reads the database itself; affects AC-001.3, AC-001.5).
- **A3** The board is the website's start page, with the list and the form on one page (other option: a separate `/tasks` page; affects AC-001.3 to AC-001.6).
- **A4** The web form checks the title the same way the API does (1 to 200 characters, not only whitespace) before sending, with the messages in AC-001.6; the due date is picked with a date field, so it can only be a valid date or empty (other option: rely on the API's 422 alone; affects AC-001.6).
- **A5** Due dates are shown as `YYYY-MM-DD` and a missing one as "No due date"; tasks are listed oldest first, as the API returns them (other option: localised dates, newest first; affects AC-001.3).
- **A6** A database outage gives 503 with a JSON error body and fails the readiness check (other option: 500; affects AC-001.7).
- **A7** Error and empty-state wording is English, as written in AC-001.4, AC-001.6 and AC-001.8 (other option: other wording or languages; affects those criteria).
- **A8** "Within 2 seconds" in AC-001.5 is measured in dev with a healthy API and database (other option: no time limit; affects AC-001.5).
- **A9** The database is the platform's in-cluster Postgres resource, which costs nothing beyond the cluster (other option: a managed database service; affects AC-001.1, AC-001.2).

## Open questions

## Changelog

- 2026-11-16 created
- 2026-11-16 approved
- 2026-11-16 building
- 2026-11-16 done
