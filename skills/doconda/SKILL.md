---
name: doconda
description: Integrate Doconda, the API that creates, reviews and edits Word, PowerPoint, Excel and PDF documents (and "artifacts", web pages with their own data) for apps and AI agents. Use when writing code that calls the Doconda API, the @doconda/sdk TypeScript SDK or the @doconda/mcp server; when generating .docx/.pptx/.xlsx/.pdf files from an app or agent; when the code imports @doconda/sdk, uses DOCONDA_API_KEY, an `ak_eu_` key or api.eu.doconda.com, or imports the doconda Python package.
---

# Doconda

One endpoint does almost everything: `POST /v1/documents`. You say **what** you want (a sentence, or your Markdown)
and **which file** (`format`); Doconda writes it (if needed), lays it out, renders it, reviews it, fixes what it can and
returns the files plus a report. Never generate OOXML yourself: send Markdown or a prompt.

Full field tables, error codes and review codes: [reference.md](reference.md). Docs: <https://docs.doconda.com/en>
(index for agents: [llms.txt](https://docs.doconda.com/llms.txt); add `.md` to any page URL to get it as Markdown).

## Setup

- API key per project, from the dashboard (**API keys**), `ak_eu_…`. Every key uses the same API:
  `https://api.eu.doconda.com/v1`; documents and files are stored in the EU.
- Keep it in `DOCONDA_API_KEY`. **Server side only**: the key gives access to every document in the project.
- TypeScript: `npm install @doconda/sdk` (Node ≥ 20; its only runtime dependency is `zod`). Python: `pip install doconda`.
  Other languages: plain HTTP with `Authorization: Bearer $DOCONDA_API_KEY`.
- Messages (error `message`, report texts, `style_unsupported`, a failed document's `error.message`) are in English.
  For Spanish pass `language: "es"` (Python `language="es"`, from `doconda` 0.2.0); the SDK sends it as
  `Accept-Language`. A document's texts keep the language of the request that created it. Branch on `code`, never on
  the text.

```ts
import { readFile, writeFile } from "node:fs/promises"
import { Doconda } from "@doconda/sdk"
const doconda = new Doconda() // reads DOCONDA_API_KEY, retries 429/5xx/network twice
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
| Fix layout problems in an existing file (no AI, €0.03 + VAT) | `{ operation: "review", file }` |
| Change an existing file as asked, rest untouched | `{ operation: "edit", file, prompt }` |
| Turn any file into Markdown for an LLM (free up to 1,000 pages a month, then €0.01 every 5 pages) | `doconda.extract(file)` (`POST /v1/extract`) |
| A web page that stores data (poll, list, calculator) | `{ format: "artifact", prompt }` |

`format`: `docx` (Word + PDF), `pdf`, `pptx` (+ PDF), `xlsx` (+ PDF), `artifact`. Set it explicitly: if omitted it is
inferred from `prompt`, and an unclear prompt fails with `format_unclear`. Don't pick the document kind (report, letter,
contract…): Doconda infers it from the prompt; `doc.type` says what it chose.

If the app already has the text (an LLM wrote it, a template filled it), send `content.markdown`, not `prompt`.
Markdown supports headings, paragraphs, lists, tables, bold/italic, links, page breaks and images
(`![Caption](img:1)` = first image in `sources`; `![Caption](https://…)` = a picture from a link).

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
  footer: Confidential"). Any Google Fonts family works. What can't be applied comes back in `style_unsupported` —
  check it, it is never silently dropped.
- `quality`: `fast` (seconds: basic style, no templates or charts), `standard` (about 3 minutes: any style, templates,
  charts), `best` (more review and design care, about 5–10 minutes) or `auto` (default). With `auto`, a `prompt` gets
  the level it needs (the cheapest when in doubt), your Markdown (`content`) always gets `standard`, and copying a
  file's design gets `best`. For a Markdown document in about 10 seconds, pass `quality: "fast"`. `max_quality` caps
  `auto`. Prices per document (fast / standard / best, + VAT): create from a prompt €0.15 / €0.50 / €1.50, from your
  own Markdown (`content`) €0.05 / €0.20 / €0.60; edit €0.10 / €0.30 / €1.00. Failed and canceled documents are free.
- `sources`: up to 5 documents (material to write from) + 10 images (placed in the document). To copy a template's
  look, put it in `sources` and say so in the prompt ("with the design of our template"): copying works at `standard` and
  `best`; in `fast` the document keeps its own design and the report says so (`design.not_copied`). With `auto`,
  copying any file other than a PowerPoint template for a deck picks `best`.
- `?dry_run=true` checks the request (fields, operation, files) without generating or charging. It doesn't try the
  `style`: unsupported parts only show up in `style_unsupported` of a real run.

## Files

Anywhere a file goes (`file`, `sources[]`) you can pass:
`"file_…"` (uploaded with `POST /v1/files`), `"doc_…"` (a document Doconda made), `{ "url": "https://…" }` (public) or
`{ "data": "<base64>", "filename": "x.docx" }`. Max 20 MB. Files with macros are rejected.

```ts
const file = await doconda.files.upload(await readFile("contract.docx"), "contract.docx")
const reviewed = await doconda.documents.create({ operation: "review", file: file.id })
const edited = await doconda.documents.create({ operation: "edit", file: file.id, prompt: "Add a confidentiality clause" })
```

## Check the result — always

```ts
if (doc.status === "failed") throw new Error(`${doc.error?.code}: ${doc.error?.message}`) // not an exception, not charged
if (doc.status === "needs_review") {
  const report = await doconda.documents.report(doc.id) // issues, ledger (fixes), edits, key_data
}
```

`create` and `wait` return a final `status`: `ready` · `ready_with_warnings` · `needs_review` (done, but something needs
a human look) · `failed` (`error` says why) · `canceled` (`documents.cancel`). Failed and canceled are not charged.
`queued` and `running` only show up from `createInBackground`, `get` or `list`.
Review never changes text or values: it flags them. Edit reports each change (`report.edits`, `rejected` with reason).

## Links expire

`doc.outputs[].url` lasts **5 minutes**. Don't store URLs: store `doc.id` and call `documents.outputs(id)` /
`documents.download(id, kind)` when needed. An artifact's page link never changes and works until its retention ends (`expires_at`).

## Long work, live progress

- `documents.createInBackground(body)` → store `doc.id` → `documents.wait(id)` later (job queues, serverless).
- `documents.stream(body)` yields events: `content.block_delta` (text as written), `preview.page_ready`,
  `document.ready`/`document.failed`/`document.canceled` (final). Many events carry `display` (`{ es, en }`), a ready
  UI label; it is optional (`content.block_delta` has none). Ignore unknown event types.
- Resume with `documents.events(id, { after: lastSequence })`. Disconnecting does not stop the document.
- To show progress in a browser, proxy the stream through your server (SSE); never ship the key to the client.
- Webhooks: register an HTTPS endpoint (`doconda.webhooks.create({ url })`, the `secret` comes once) and get a signed
  POST when a document finishes (`document.ready` / `failed` / `canceled`, no content). Check it with
  `const event = await verifyWebhook(rawBody, headers, secret)` (it throws on a bad signature), dedupe by
  `webhook-id`, then fetch the files.

## Errors and retries

API errors are RFC 9457 problem+json; the SDK throws `DocondaError` with `status`, `code` (stable — branch on it),
`message`, `errors` (422 field paths) and `requestId`. Common: `input_invalid` (fix the field), `insufficient_balance`
(402, top up), `unauthorized`, `not_found`, `rate_limited` (120 documents/min per organization).

The SDK retries network/429/5xx with an automatic `Idempotency-Key`. If **your** code may repeat an operation (job
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
  the remote server `https://api.eu.doconda.com/mcp` with `Authorization: Bearer ak_…`.
- Function calling: `doconda.tools()` returns tools (`create_document`, `upload_file`, `review_document`,
  `edit_document`, `list_documents`, `get_document`, `get_report`, `read_artifact_data`) with `name`, `description`,
  `inputSchema` (JSON Schema) and `run(args)`. Map them to your provider's tool format; return
  `JSON.stringify(await tool.run(input))` as the tool result. If the API answers with an error, `run` doesn't throw: it
  returns `{ error: "<code>", message, fields? }` (`fields` on validation errors) for the model to act on (TypeScript SDK 0.4.0+, Python 0.2.0+).

## Don'ts

- Don't build DOCX/PPTX/XLSX with another library and then send it to Doconda to "make it nice": send Markdown.
- Don't call the API from the browser or commit the key.
- Don't set `baseUrl` in production (`DOCONDA_BASE_URL` only for a local stack).
- Don't treat `status: "failed"` as a thrown error, or ignore `needs_review` / `style_unsupported`.
- Don't persist output URLs.
