# Doconda reference

## Endpoints (`https://api.eu.doconda.com/v1`)

| Method | Path | SDK |
|---|---|---|
| POST | `/documents` | `documents.create` / `createInBackground` / `stream` |
| GET | `/documents?status=&limit=&cursor=` | `documents.list` (async iterator) / `page` |
| GET | `/documents/{id}` | `documents.get` |
| POST | `/documents/{id}/cancel` | `documents.cancel` |
| DELETE | `/documents/{id}` | `documents.delete` — files, report, events (and an artifact's page and data); a running document must be canceled first |
| PUT | `/documents/{id}/file` (multipart `revision` then `file`, or JSON) | — saves a hand-edited file over the same document: `revision` + 1, new PDF/report, free |
| GET | `/documents/{id}/events` (`Last-Event-ID` or `?after=`) | `documents.events` / `wait` |
| GET | `/documents/{id}/outputs` | `documents.outputs` / `download` |
| GET | `/documents/{id}/report` | `documents.report` |
| GET | `/documents/{id}/data[/{collection}]` | `documents.data` — collections, or a collection's 1,000 most recently updated records |
| POST | `/extract` | `extract` — any file → `{ id, markdown, chars, truncated, pages }`; nothing kept; past 120 s `202` → `GET /extract/{id}` (the SDKs wait) |
| POST | `/files` (multipart `file`, or JSON `url`/`data`) | `files.upload` |
| GET / DELETE | `/files/{id}` | `files.get` / `files.delete` |
| POST / GET | `/webhook-endpoints` | `webhooks.create` (the `secret` comes once) / `webhooks.list` |
| DELETE | `/webhook-endpoints/{id}` | `webhooks.delete` |
| POST | `/editor/collab/token` (`{ room }`) | `editor.collabToken` / `editor.collab_token` — `{ token, expires_at }` for the editor plugin's real-time co-editing on Doconda's server (`collab={{ token }}`); one room, the licence's domains, 10 minutes; needs an active editor licence |

MCP: `https://api.eu.doconda.com/mcp` (Streamable HTTP; OAuth sign-in or `Authorization: Bearer ak_…`) or
`npx -y @doconda/mcp` (stdio, `DOCONDA_API_KEY`).

## `POST /documents` body

| Field | Type | Notes |
|---|---|---|
| `operation` | `create` (default) · `review` · `edit` | reading is `POST /extract` |
| `format` | `docx` · `pdf` · `pptx` · `xlsx` · `artifact` | create; inferred from `prompt` if omitted |
| `prompt` | string 3–8000 | create (AI writes) or edit (what to change) |
| `content.markdown` | string ≤ 500 000 | create from your text; one of `prompt`/`content` |
| `style` | string ≤ 1000 | free text; unsupported parts → `style_unsupported` |
| `sources` | ≤ 15 file refs | ≤ 5 documents + ≤ 10 images |
| `file` | file ref | review / edit |
| `quality` | `auto` · `fast` · `standard` · `best` | create / edit |
| `max_quality` | `fast` · `standard` · `best` | cap for `auto` |
| `stream` | bool | SSE on this connection |
| `background` | bool | `202` at once |
| `name` | string ≤ 200 | also names downloaded files |
| `metadata` | `{ string: string ≤ 500 }` | returned as is |
| `artifact.storage` | `shared` · `local` | artifacts only |
| `retention` | `none` · `1d` · `7d` · `30d` · `90d` | how long its content is kept once finished (`none` = 15 min); default: the project's (`30d` unless changed) |

File ref: `"file_…"`, `"doc_…"`, `{ url, filename? }`, `{ data (base64), filename }`. 20 MB max. Accepted: Word,
PowerPoint, Excel (also `.doc/.ppt/.xls`, returned as modern formats), PDF, TXT, Markdown, HTML (as a source only), PNG/JPEG/GIF/WebP.

## Document (response)

Key fields: `id`, `status` (`queued` · `running` · `ready` · `ready_with_warnings` · `needs_review` · `failed` ·
`canceled`), `format`, `type`, `pages`, `outputs[] { kind, url, expires_at }`, `style_applied`, `style_unsupported[]`,
`validation_summary { errors, warnings }`, `quality { requested, level, reason }`, `route`, `error { code, message }` (`null`
unless `failed`), `retention`, `expires_at`.

Report: `issues[] { code, severity, message, fixable, location { pages | sheet, cells } }`, `ledger[]` (fixes applied or
rejected), `changes[]` (what an edit or a hand-saved file added, changed or removed, before/after; `edits[]` only on
edits before 2026-10-28), `key_data`, `sanitized.removed`.

## HTTP errors (`code`)

| code | HTTP | Action |
|---|---|---|
| `input_invalid` | 422 | fix `errors[].path` |
| `unauthorized` | 401 | bad/revoked key |
| `insufficient_balance` | 402 | top up in the dashboard |
| `session_required` | 403 | dashboard-only route |
| `account_blocked` | 403 | organization blocked by Doconda; don't retry |
| `not_found` | 404 | wrong project/organization |
| `report_not_ready` | 409 | wait for the document to finish |
| `idempotency_request_in_progress` | 409 | retry in seconds |
| `idempotency_key_reused` | 422 | same key, different body |
| `not_an_artifact` / `artifact_is_local` / `too_many_records` | 409 | artifact data |
| `content_deleted` / `events_expired` | 410 | retention ended / events > 7 days |
| `file_rejected` | 422 | unsupported file type (a file with macros uploads, then its document fails with `file_rejected`) |
| `file_too_large` / `record_too_large` | 413 | 20 MB / 16 KB |
| `file_unreachable` | 422 | the `url` must be public and answer with the file |
| `fst_…` | 400 / 413 / 415 | unreadable request: invalid JSON, wrong `Content-Type` or body too large (upload big files with `POST /files` first) |
| `document_has_no_file` | 409 | the `doc_…` used as a file has no Word/PowerPoint/Excel/PDF (or isn't finished) |
| `document_not_finished` | 409 | cancel a running document before deleting it |
| `revision_conflict` / `document_busy` / `file_format_mismatch` | 409 / 409 / 422 | hand-saved file: someone saved after you opened it · still being made or saved · not the document's format |
| `webhook_url_invalid` | 422 | webhook URLs must be public `https` |
| `editor_unlicensed` | 403 | `/editor/collab/token`: the organization has no active editor licence |
| `retention_none_shared` | 422 | with `retention: none`, an artifact must be `local` |
| `rate_limited` | 429 | 120 docs/min per organization; artifacts 1200 reads, 120 writes/min |
| `feature_unavailable` | 501 | AI disabled in that deployment |
| `internal_error` | 500 | retry; report `request_id` |

## Failed document (`status: "failed"`, `error.code`, never charged)

`content_empty`, `file_rejected` (macros, abnormal structure, a password-protected PDF, a PDF over 50 pages to review
or edit), `docx_invalid`, `file_not_found`, `format_unclear` (set `format`), `design_needs_quality` (copying a file's
design at `fast`: use `standard` or `best`), `agent_failed` / `agent_not_delivered` (the agent couldn't finish; retry or
rephrase), `agent_unavailable` (retry in a few minutes), `source_unreadable`,
`generation_blocked` / `request_refused` (rephrase; harmful requests refused), `page_too_large`, `page_too_long`, `processing_failed`
(retry later).

## Review issue codes (examples)

- Word: `table.overflow_width`, `heading.orphaned`, `signature.split_across_pages`, `numbering.discontinuous`,
  `crossref.broken`, `content.empty_section`.
- PowerPoint: `pptx.text_overflow`, `pptx.shape_off_slide`, `pptx.text_too_small`, `pptx.no_title`, `pptx.image_low_resolution`.
- Excel: `xlsx.formula_error`, `xlsx.broken_reference`, `xlsx.number_as_text`, `xlsx.date_as_text`, `xlsx.hidden_data`.
- PDF: `pdf.no_text_layer` (OCR added), `pdf.fonts_not_embedded`, `pdf.blank_page`, `pdf.untagged`.
- Any: `money.words_mismatch`, `font.missing`, `agent.unresolved` (the agent ran out of steps with problems left; check the file).
- Content checks on AI-written text: `content.unsupported_fact`, `content.unsupported_citation`, `content.unclear`,
  `content.contradiction`, `image.web_source`.

## Events

Shape: `{ id, type, version, document_id, sequence, timestamp, display?: { es, en }, data }` (`display` only on some events). `sequence` is gapless per
document and is the SSE `id:`. Heartbeat every 15 s; the server closes after the final event.

Lifecycle: `document.created`, `document.started`, `document.ready` ⏹, `document.failed` ⏹, `document.canceled` ⏹. Hand-saved file (`PUT /documents/{id}/file`): `document.updated` ⏹ or `document.update_failed` ⏹ (the document stays as it was).
Content: `style.resolved`, `content.block_started`, `content.block_delta { text }`, `content.block_completed`. Process: `render.completed`,
`validation.issue_found`, `validation.completed`, `repair.applied`, `repair.rejected`, `preview.page_ready { page, total }`,
`output.available`. Routing: `request.understood`, `quality.chosen`, `route.chosen`. Agent: `agent.started`, `agent.step`,
`agent.checked`, `agent.delivered`, `design.chosen`, `artifact.checked`. New types can appear: ignore unknown ones.

## Artifact page API (`window.doconda.db`)

`set(collection, key, data)`, `get(collection, key)`, `list(collection)` → `[{ key, data, updated_at }]`,
`delete(collection, key)`, `subscribe(collection, cb)` → unsubscribe. Names: letters, digits, `_ - .` (≤ 64). Record ≤ 16 KB
JSON, ≤ 5000 records per artifact. The page saves only through `doconda.db` (with `storage: "local"`, in the visitor's
`localStorage`); it runs sandboxed on its own subdomain and can use Tailwind, Chart.js, d3, dayjs, no external APIs. Embed: `<iframe src="…" sandbox="allow-scripts allow-same-origin allow-forms">`.
