# Doconda reference

## Endpoints (`https://api.{eu|us}.doconda.com/v1`)

| Method | Path | SDK |
|---|---|---|
| POST | `/documents` | `documents.create` / `createInBackground` / `stream` |
| GET | `/documents?status=&limit=&cursor=` | `documents.list` (async iterator) / `page` |
| GET | `/documents/{id}` | `documents.get` |
| POST | `/documents/{id}/cancel` | `documents.cancel` |
| POST | `/documents/{id}/versions` (multipart `file`) | — saves a hand-edited version: new doc, `parent_id`, free |
| GET | `/documents/{id}/events` (`Last-Event-ID` or `?after=`) | `documents.events` / `wait` |
| GET | `/documents/{id}/outputs` | `documents.outputs` / `download` |
| GET | `/documents/{id}/report` | `documents.report` |
| GET | `/documents/{id}/data[/{collection}]` | `documents.data` |
| POST | `/extract` | — any file → `{ id, markdown, chars, truncated, pages }`; nothing kept; past 120 s `202` → `GET /extract/{id}` |
| POST | `/files` (multipart `file`, or JSON `url`/`data`) | `files.upload` |
| GET / DELETE | `/files/{id}` | `files.get` / `files.delete` |

MCP: `https://api.{eu|us}.doconda.com/mcp` (Streamable HTTP) or `npx -y @doconda/mcp` (stdio, `DOCONDA_API_KEY`).

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
| `store` | bool, default true | false → content deleted 15 min after finishing |

File ref: `"file_…"`, `"doc_…"`, `{ url, filename? }`, `{ data (base64), filename }`. 20 MB max. Accepted: Word,
PowerPoint, Excel (also `.doc/.ppt/.xls`, returned as modern formats), PDF, TXT, Markdown, PNG/JPEG/GIF/WebP.

## Document (response)

Key fields: `id`, `status` (`queued` · `running` · `ready` · `ready_with_warnings` · `needs_review` · `failed` ·
`canceled`), `format`, `type`, `pages`, `outputs[] { kind, url, expires_at }`, `style_applied`, `style_unsupported[]`,
`validation_summary { errors, warnings }`, `quality { requested, level, reason }`, `route`, `error { code, message }`.

Report: `issues[] { code, severity, message, fixable, location { pages | sheet, cells } }`, `ledger[]` (fixes applied or
rejected), `edits[]` (edit changes, before/after), `changes` (versions), `key_data`, `sanitized.removed`.

## HTTP errors (`code`)

| code | HTTP | Action |
|---|---|---|
| `input_invalid` | 422 | fix `errors[].path` |
| `unauthorized` | 401 | bad/revoked key |
| `region_mismatch` | 401 | use the key's region URL |
| `insufficient_balance` | 402 | top up in the dashboard |
| `session_required` | 403 | dashboard-only route |
| `not_found` | 404 | wrong project/organization |
| `report_not_ready` | 409 | wait for the document to finish |
| `idempotency_request_in_progress` | 409 | retry in seconds |
| `idempotency_key_reused` | 422 | same key, different body |
| `not_an_artifact` / `artifact_is_local` / `too_many_records` | 409 | artifact data |
| `content_deleted` / `events_expired` | 410 | retention ended / events > 7 days |
| `file_rejected` | 422 | unsupported or macro file |
| `file_too_large` / `record_too_large` | 413 | 20 MB / 16 KB |
| `retention_none_shared` | 422 | with `retention: none`, an artifact must be `local` |
| `rate_limited` | 429 | 120 docs/min per organization; artifacts 1200 reads, 120 writes/min |
| `feature_unavailable` | 501 | AI disabled in that deployment |
| `internal_error` | 500 | retry; report `request_id` |

## Failed document (`status: "failed"`, `error.code`, never charged)

`content_empty`, `file_rejected`, `docx_invalid`, `file_not_found`, `document_too_long` (edit > ~50 pages),
`format_unclear` (set `format`), `edit_not_understood` / `edit_not_applied` (be more precise), `source_unreadable`,
`generation_blocked` / `request_refused` (rephrase; harmful requests refused), `page_too_large`, `page_too_long`, `processing_failed`
(retry later).

## Review issue codes (examples)

- Word: `table.overflow_width`, `heading.orphaned`, `signature.split_across_pages`, `numbering.discontinuous`,
  `crossref.broken`, `content.empty_section`.
- PowerPoint: `pptx.text_overflow`, `pptx.shape_off_slide`, `pptx.text_too_small`, `pptx.no_title`, `pptx.image_low_resolution`.
- Excel: `xlsx.formula_error`, `xlsx.broken_reference`, `xlsx.number_as_text`, `xlsx.date_as_text`, `xlsx.hidden_data`.
- PDF: `pdf.no_text_layer` (OCR added), `pdf.fonts_not_embedded`, `pdf.blank_page`, `pdf.untagged`.
- Any: `money.words_mismatch`, `font.missing`.
- Content checks on AI-written text: `content.unsupported_fact`, `content.unsupported_citation`, `content.unclear`,
  `content.contradiction`, `quality.check_failed`, `image.web_source`.

## Events

Shape: `{ id, type, version, document_id, sequence, timestamp, display: { es, en }, data }`. `sequence` is gapless per
document and is the SSE `id:`. Heartbeat every 15 s; the server closes after the final event.

Lifecycle: `document.created`, `document.started`, `document.ready` ⏹, `document.failed` ⏹, `document.canceled` ⏹.
Content: `style.resolved`, `content.block_started`, `content.block_delta { text }`, `content.block_completed`,
`edit.planned`, `edit.operation_applied`, `edit.operation_rejected`. Process: `render.completed`,
`validation.issue_found`, `validation.completed`, `repair.applied`, `repair.rejected`, `preview.page_ready { page, total }`,
`output.available`. Agent: `quality.chosen`, `agent.step`, `agent.checked`, `agent.fallback`, `design.chosen`,
`artifact.checked`. New types can appear: ignore unknown ones.

## Artifact page API (`window.doconda.db`)

`set(collection, key, data)`, `get(collection, key)`, `list(collection)` → `[{ key, data, updated_at }]`,
`delete(collection, key)`, `subscribe(collection, cb)` → unsubscribe. Names: letters, digits, `_ - .` (≤ 64). Record ≤ 16 KB
JSON, ≤ 5000 records per artifact. The page has no cookies or `localStorage`; it can use Tailwind, Chart.js, d3, dayjs, no
external APIs. Embed: `<iframe src="…" sandbox="allow-scripts allow-same-origin allow-forms">`.
