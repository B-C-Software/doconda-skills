---
name: doconda
description: Integrate Doconda, the API that converts, creates and edits Word, PowerPoint, Excel and PDF documents (and "artifacts", web pages with their own data) for apps and AI agents. Use when writing code that calls the Doconda API, the @doconda/sdk TypeScript SDK or the @doconda/mcp server; when generating .docx/.pptx/.xlsx/.pdf files from an app or agent; when the code imports @doconda/sdk, uses DOCONDA_API_KEY, an `ak_eu_` key or api.eu.doconda.com, or imports the doconda Python package.
---

# Doconda

One endpoint does almost everything: `POST /v1/documents`. You say **what** you want (a sentence, or your Markdown)
and **which file** (`format`); Doconda writes it (if needed), lays it out, renders it, checks and fixes it, and
returns the files. Never generate OOXML yourself: send Markdown or a prompt.

Full field tables and error codes: [reference.md](reference.md). Docs: <https://docs.doconda.com>
(index for agents: [llms.txt](https://docs.doconda.com/llms.txt); add `.md` to any page URL to get it as Markdown).

## Setup

- API key per project, from the dashboard (**API keys**), `ak_eu_…`. Every key uses the same API:
  `https://api.eu.doconda.com/v1`; documents and files are stored in the EU.
- Keep it in `DOCONDA_API_KEY`. **Server side only**: the key gives access to every document in the project.
- TypeScript: `npm install @doconda/sdk` (Node ≥ 20; its only runtime dependency is `zod`). Python: `pip install doconda`.
  Other languages: plain HTTP with `Authorization: Bearer $DOCONDA_API_KEY`.
- Messages (error `message`, a failed document's `error.message`) are in English.
  For Spanish pass `language: "es"` (Python `language="es"`, from `doconda` 0.2.0); the SDK sends it as
  `Accept-Language`. A document's texts keep the language of the request that created it. Branch on `code`, never on
  the text.

```ts
import { readFile, writeFile } from "node:fs/promises"
import { Doconda } from "@doconda/sdk"
const doconda = new Doconda() // reads DOCONDA_API_KEY; retries reads and creates on 429/5xx/network twice
```

Python has the same methods, synchronous, with the body as keyword arguments and dicts as responses:

```python
from pathlib import Path
from doconda import Doconda

doconda = Doconda()
doc = doconda.documents.create(format="docx", content={"markdown": "# Note\n\nHello"}, quality="fast")
Path("note.docx").write_bytes(doconda.documents.download(doc["id"], "docx"))
```

## Pick the operation

| Goal | Request |
|---|---|
| New document from a sentence (AI writes it) | `{ format, prompt }` |
| New document from your text (AI only lays it out: cheaper, faster, text kept as is) | `{ format, content: { markdown } }` |
| Write from the user's files | `{ format, prompt, sources: [...] }` |
| Change an existing file as asked, rest untouched | `{ operation: "edit", file, prompt }` |
| Turn a file into another format, no AI: any file → Markdown for an LLM, Markdown → Word, PDF → Word, anything → PDF (free up to 1,000 a month, then €0.01 each) | `doconda.convert(file, to)` (`POST /v1/convert`) |
| A web page that stores data (poll, list, calculator) | `{ format: "artifact", prompt }` |

`format`: `docx` (Word + PDF), `pdf`, `pptx` (+ PDF), `xlsx` (+ PDF), `artifact`. Set it explicitly: if omitted it is
inferred from `prompt`, and an unclear prompt fails with `format_unclear`. Don't pick the document kind (report, letter,
contract…): Doconda infers it from the prompt; `doc.type` says what it chose.

If the app already has the text (an LLM wrote it, a template filled it), send `content.markdown`, not `prompt`.
Markdown supports headings, paragraphs, lists, tables, bold/italic, links, page breaks and images
(`![Caption](img:1)` = first image in `sources`; `![Caption](https://…)` = a picture from a link, up to 10 per
document, 50 MB and 2 minutes of downloads in total; past that they stay as plain links).

## Create

```ts
const doc = await doconda.documents.create({
  format: "docx",
  content: { markdown: "# Internal note\n\nMonday's meeting moves to 10:00." },
  style: "Lora font, dark blue headings, page numbers",
  name: "Meeting note",
})
await writeFile("note.docx", await doconda.documents.download(doc.id, "docx"))
await writeFile("note.pdf", await doconda.documents.download(doc.id, "pdf"))
```

```bash
curl https://api.eu.doconda.com/v1/documents \
  -H "Authorization: Bearer $DOCONDA_API_KEY" -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{"format":"docx","prompt":"Residential lease for €900 a month","style":"blue headings"}'
```

- `create` waits until it finishes (like an LLM call). Over 120 s the raw API answers `202`; the SDK keeps waiting and
  reconnects on its own if the connection drops.
- `style` is free text ("Lora 11, centred title, justified, cover page, table of contents, 2 cm margins, landscape,
  footer: Confidential"). Any Google Fonts family works; a font that isn't available is replaced by the
  closest one.
- `quality`: every level is made by an AI agent in a sandbox, with any style, templates and charts; levels differ in
  steps and how often it looks at the rendered pages. `fast` (about a minute; can't copy a file's design), `standard`
  (a few minutes), `best` (most care, longer than `standard`) or `auto` (default). With `auto`, a `prompt` gets the level
  it needs (the cheapest when in doubt), your Markdown (`content`) always gets `standard`, and copying a file's design
  gets at least `standard` (`best` unless it's a PowerPoint template for a deck). For a quicker Markdown document, pass
  `quality: "fast"`. `max_quality` caps
  `auto`. Prices per document (fast / standard / best, + VAT): create from a prompt €0.15 / €0.50 / €1.50, from your
  own Markdown (`content`) €0.05 / €0.20 / €0.60; edit €0.10 / €0.30 / €1.00. Failed and canceled documents are free.
- `sources`: up to 5 documents (material to write from) + 10 images (placed in the document). To copy a template's
  look, put it in `sources` and say so in the prompt ("with the design of our template"): copying works at `standard` and
  `best`; with `quality: "fast"` the document fails with `design_needs_quality` (not charged).
- `?dry_run=true` checks the request (fields, operation, files) without generating or charging. It doesn't try the
  `style`.

## Files

Anywhere a file goes (`file`, `sources[]`) you can pass:
`"file_…"` (uploaded with `POST /v1/files`), `"doc_…"` (a document Doconda made), `{ "url": "https://…" }` (public) or
`{ "data": "<base64>", "filename": "x.docx" }`. Max 20 MB. Files with macros are rejected.

```ts
const file = await doconda.files.upload(await readFile("contract.docx"), "contract.docx")
const markdown = (await doconda.convert(file.id, "md")).text // md/txt/html/csv come back as text; docx/pdf/… as output.url (5 min)
const edited = await doconda.documents.create({ operation: "edit", file: file.id, prompt: "Add a confidentiality clause" })
const saved = await doconda.documents.updateFile(doc.id, editedBytes, { filename: "report.docx", revision: doc.revision }) // a file edited by hand, saved over the same document
```

## Check the result — always

```ts
if (doc.status === "failed") throw new Error(`${doc.error?.code}: ${doc.error?.message}`) // not an exception, not charged
const { changes } = await doconda.documents.report(edited.id) // what an edit changed, before/after
```

`create` and `wait` return a final `status`: `ready` · `failed` (`error` says why) · `canceled` (`documents.cancel`).
Failed and canceled are not charged. `queued` and `running` only show up from `createInBackground`, `get` or `list`.

## Links expire

`doc.outputs[].url` lasts **5 minutes**. Don't store URLs: store `doc.id` and call `documents.outputs(id)` /
`documents.download(id, kind)` when needed. An artifact's page link never changes and works until its retention ends (`expires_at`).

## Long work, live progress

- `documents.createInBackground(body)` → store `doc.id` → `documents.wait(id)` later (job queues, serverless).
- Step-by-step progress (logs, progress bars): `documents.events(id, { after: lastSequence })`. Disconnecting does not
  stop the document.
- To show progress in a browser, proxy the events through your server (SSE); never ship the key to the client.
- Webhooks: register an HTTPS endpoint (`doconda.webhooks.create({ url })`, the `secret` comes once) and get a signed
  POST when a document finishes (`document.ready` / `failed` / `canceled`) or a hand-save ends (`document.updated` /
  `update_failed`), with no content. Check it with
  `const event = await verifyWebhook(rawBody, headers, secret)` (it throws on a bad signature), dedupe by
  `webhook-id`, then fetch the files.

## Errors and retries

API errors are RFC 9457 problem+json; the SDK throws `DocondaError` with `status`, `code` (stable — branch on it),
`message`, `errors` (422 field paths) and `requestId`. Common: `input_invalid` (fix the field), `insufficient_balance`
(402, top up), `unauthorized`, `not_found`, `rate_limited` (120 documents/min per organization).

The SDK retries reads and `create` (with an automatic `Idempotency-Key`) on network/429/5xx, waiting `Retry-After`;
`convert`, `files.upload`, `updateFile` and `webhooks.create` go out once. If **your** code may repeat an operation (job
retries, double clicks), pass a stable key: `` create(body, { idempotencyKey: `invoice-${id}` }) `` (24 h). With raw HTTP,
always send `Idempotency-Key` on `POST /documents`.

## Retention

Every document (artifacts too) is kept for its `retention` once it finishes: `none` (15 minutes), `1d`, `7d`, `30d`
(default) or `90d`; the project sets the default, a request can ask for another. Then its content is deleted
(`expires_at` says when; `/report` and `/events` return `410 content_deleted`). Download the files before, or have a
webhook take them as soon as a document finishes. `documents.delete(id)` deletes it sooner.

## Artifacts

```ts
const doc = await doconda.documents.create({ format: "artifact", prompt: "Poll for Friday's menu with live results" })
const [page] = await doconda.documents.outputs(doc.id) // embed page.url in an <iframe>; it works until doc.expires_at
const votes = await doconda.documents.data(doc.id, "votes") // [{ key, data, updated_at }], the 1,000 most recent
```

The page stores data with `window.doconda.db` (`set/get/list/delete/subscribe`). `artifact: { storage: "local" }` keeps
data in each visitor's browser (your program can't read it: `data()` answers 409 `artifact_is_local`). `sources` work
for artifacts too: the page is built from the documents' data and the pictures go inside it.

## Giving Doconda to an AI agent

- MCP (no code): `claude mcp add --transport stdio doconda --env DOCONDA_API_KEY=ak_eu_… -- npx -y @doconda/mcp`, or
  the remote server `https://api.eu.doconda.com/mcp`: clients with MCP authorization sign in with the user's Doconda
  account (OAuth, no key); anything else sends `Authorization: Bearer ak_…`.
- Function calling: `doconda.tools()` returns tools (`create_document`, `upload_file`, `convert_file`,
  `edit_document`, `list_documents`, `get_document`, `get_report`, `read_artifact_data`) with `name`, `description`,
  `inputSchema` (JSON Schema) and `run(args)`. Map them to your provider's tool format; return
  `JSON.stringify(await tool.run(input))` as the tool result. If the API answers with an error, `run` doesn't throw: it
  returns `{ error: "<code>", message, fields? }` (`fields` on validation errors) for the model to act on (TypeScript SDK 0.4.0+, Python 0.2.0+).
  `create_document` and `edit_document` wait up to about 4 minutes; a document that takes longer comes back with its
  `id`, `status` and a `note`, and the model calls `get_document` later (TypeScript SDK 0.9.0+, Python 0.7.0+, MCP
  0.6.0+). Give MCP clients a per-call timeout of 300 s.

## Don'ts

- Don't build DOCX/PPTX/XLSX with another library and then send it to Doconda to "make it nice": send Markdown.
- Don't call the API from the browser or commit the key.
- Don't set `baseUrl` in production (`DOCONDA_BASE_URL` only for a local stack).
- Don't treat `status: "failed"` as a thrown error.
- Don't persist output URLs.
