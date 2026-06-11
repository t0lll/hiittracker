# yourfolio Integration — Developer Notes

Notes from integrating yourfolio into a zero-dependency static HIIT timer app.
Written to help improve the DX for future builders.

---

## What worked well

**MCP tool flow is logical.** The sequence `list_use_cases → get_schema → create_folio → write_folio` is sensible and easy to follow. Having the tools guide you through setup step-by-step feels right.

**Schema validation is strict and helpful.** When a write fails, the `fixes` array in the error response tells you exactly what to correct. That's better than a generic 400.

**`app_url` in write response is a nice touch.** Getting back `https://your-app.com/?folio={token}` immediately after a write means you can hand the link straight to the user without constructing it yourself.

**CORS is open (`*`).** No surprises when moving from server-side MCP calls to client-side `fetch()` — it just works.

---

## Friction points and suggested improvements

### 1. The REST API is undiscovered

The MCP tools abstract away the HTTP calls, but any real app needs to make those calls from the browser. There's no documentation, no `/api` route, and the main site returns 403 to automated fetchers — so I had to discover the API empirically with curl:

```
GET  /f/{token}/{use_case}   → {"json": {...}, "schema_version": 1, "updated_at": "..."}
PUT  /f/{token}/{use_case}   ← {"markdown": "...", "json": {...}}
```

**Suggestion:** Expose a `/docs` or `/api` page (even a plain-text one) listing these two endpoints with their request/response shapes. Alternatively, return it from a `list_use_cases`-style endpoint or include it in the `register_use_case` response.

---

### 2. No REST endpoint for folio creation

Users can't get a folio token from within the app — there's no `POST /folios` equivalent. This means apps must tell users to go somewhere else to create a folio, which breaks the flow.

**Suggestion:** Add `POST /folios` returning `{"token": "v1..."}`. Optionally accept a display name. This lets apps auto-provision a folio on first use, with no external step required.

---

### 3. `get_schema` and `list_use_cases` are two calls when they could be one

`list_use_cases` returns just `id`, `name`, and `description`. To understand what you can actually store, you always have to follow up with `get_schema`. In practice, a builder checking if an existing use case fits their data model needs both immediately.

**Suggestion:** Add a `full` flag to `list_use_cases` that inlines the schema and `app_url_template`, or just always include them — the payload is small.

---

### 4. Markdown is write-only from the REST API

The MCP tools frame markdown as the "source of truth," but the GET endpoint only returns `json`. If an app wanted to display or edit the markdown (e.g. a notes field the user wrote), it can't retrieve it over REST.

**Suggestion:** Include `markdown` in the GET response (perhaps gated behind a `?include=markdown` param to keep the default response small).

---

### 5. Schema immutability requires upfront design

Once a use case is registered, the schema is frozen — a change means a new `usecase_id`. This is a reasonable trade-off for stability, but it's easy to register a schema that's subtly wrong before you've actually built the app.

In this integration, the pre-existing `hiit-workout` use case had a weekly-plan schema that didn't fit a single-workout execution model at all. Registering `hiittracker` with the right schema required a builder key and careful upfront design.

**Suggestions:**
- Allow schema updates during a short grace period after registration (e.g. 24 hours, before any folios have written to it).
- Alternatively, expose a `validate_json` dry-run endpoint so builders can test their schema against real data before committing.
- The `description` field on use cases is written "for the AI choosing a use case" — it would help if it also indicated the intended data shape in plain language, so a builder can quickly judge fit without calling `get_schema`.

---

### 6. No way to list or inspect existing folio entries

There's no `list` or `exists` check — you either GET a use case entry (200 or 404) or write one. If a builder wants to show the user "you have 3 saved workouts," that's not possible with the current model (one entry per use case per folio).

This is probably an intentional constraint (one folio = one plan per use case), but it's worth documenting explicitly so builders don't try to work around it in hacky ways.

**Suggestion:** Document the one-entry-per-use-case model prominently. If multi-entry support is planned, note it in the roadmap.

---

## Summary table

| Area | Current state | Suggested change |
|------|--------------|-----------------|
| REST API docs | Undocumented, discoverable only by probing | Add `/docs` or return endpoint info from `list_use_cases` |
| Folio creation | MCP-only | Add `POST /folios` REST endpoint |
| `list_use_cases` | Id + name + description only | Option to inline schema and `app_url_template` |
| Markdown retrieval | Write-only via REST | Return `markdown` in GET response |
| Schema mutability | Frozen on registration | Grace-period updates or a `validate_json` dry-run endpoint |
| Entry model | Undocumented one-entry limit | Document explicitly; note multi-entry roadmap if applicable |
