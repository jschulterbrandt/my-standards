---
description: Guides the creation of new API endpoints. Use when adding routes, handlers, or controllers to the project
---

# API Endpoints

How to add a new API endpoint. The codebase follows a layered `routes/` → `services/` → `db/queries/` architecture — each layer has exactly one job, and nothing skips a layer.

## Where route files live

- Routes live under `src/routes/`, one file per resource.
- Filename is kebab-case and matches the resource name, lowercase and plural where the resource itself is plural (e.g. `src/routes/tasks.js`, `src/routes/comments.js`). Single-word filenames still count as kebab-case (`users.js`, `tags.js`).
- Each route file exports a named Express `Router`:

```js
// src/routes/widgets.js
const express = require('express');
const router = express.Router();

router.get('/', listWidgets);
router.post('/', createWidget);

module.exports = router;
```

- Register the new router in `src/index.js` with `app.use(<resource-path>, <importedRouter>)`. Don't define routes inline in `src/index.js` — even trivial routes (health checks, root) get their own file under `src/routes/`.
- A resource that spans more than one URL namespace (e.g. nested routes like `/tasks/:id/tags`) declares its full paths in its own route file and is mounted at the app root, rather than at a single resource prefix.

## Naming conventions

- All files use kebab-case: `error-handler.js`, `send-email.js`, `tasks.js`.
- A new route file `src/routes/<resource>.js` is paired with `src/services/<resource>.js` and `src/db/queries/<resource>.js` — same resource name across all three layers.
- Test files follow `<resource>.test.js` under `tests/`, matching the `tests/*.test.js` glob in `package.json`'s `testMatch` — no config change needed.

## Handlers call services, services call queries

- Route handlers parse the request and shape the response only — no business logic, no direct `db.prepare(...)` calls.
- Each handler calls a corresponding function in `src/services/<resource>.js`.
- Services own business logic: validation beyond basic presence, existence checks, uniqueness/conflict handling, status-transition logic. Services call `src/db/queries/<resource>.js` for all database access.
- A service never calls another service. If a service needs data owned by another resource (e.g. `services/tasks.js` checking a project exists), it calls that resource's query module directly.
- Query modules contain raw SQL only — no validation, no thrown errors, just functions returning rows.

```js
// src/routes/widgets.js
const { listWidgets } = require('../services/widgets');

router.get('/', (req, res, next) => {
  try {
    res.json(listWidgets(req.query));
  } catch (err) {
    next(err);
  }
});
```

## Error handling pattern

- Services throw a plain `Error` with a `.status` property attached (e.g. `err.status = 404`, `err.status = 400`). Small local helpers like `badRequest(message)` / `notFound(message)` that build and return such an error keep this consistent within a service file.
- Route handlers wrap the service call in `try { ... } catch (err) { next(err); }` and never build an error response themselves.
- A single central middleware (`src/middleware/error-handler.js`'s `errorHandler`) is the only place that turns an error into a response: `res.status(err.status || 500).json({ error: err.message || 'Internal server error' })`.
- The one exception: when a service signals "not found" by returning `null` rather than throwing (e.g. `GET /:id`), the route itself sends the 404 — that's response-shaping, not business logic:

```js
router.get('/:id', (req, res, next) => {
  try {
    const widget = getWidgetById(parseInt(req.params.id));
    if (!widget) return res.status(404).json({ error: 'Widget not found' });
    res.json(widget);
  } catch (err) {
    next(err);
  }
});
```

## Response shape

- Success responses return the resource (or array of resources) directly as JSON — no `{ data: ... }` envelope. Creation (`POST`) responds with `res.status(201).json(resource)`; everything else defaults to 200.
- Error responses are always `{ error: message }`, produced only by the central `errorHandler`, with the HTTP status carried on the thrown error's `.status` (defaulting to 500).
- List endpoints accept pagination via `page` / `page_size` query params (`page_size` capped at 100) and return a bare array of rows, not a pagination envelope.

## Before committing

- New route, service, and query files exist as a matched trio named after the same resource, and no layer is skipped (route never calls a query module or contains `db.prepare(...)` directly).
- The new router is registered in `src/index.js`.
- A test file `tests/<resource>.test.js` exists and covers the happy path plus at least two error cases (e.g. missing required field, not-found, invalid enum, conflict/duplicate) — see the testing-standards conventions for this project.
- Any route that should require auth uses the project's auth middleware explicitly — auth is opt-in per route, not applied globally.
- No new file lives outside `src/routes/`, `src/services/`, `src/db/`, `src/middleware/`, or `src/utils/`, and no new helper was dropped into `src/utils/` without checking it isn't really service logic.
- The full test suite passes with no manual setup beyond installing dependencies.
