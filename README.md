# jointechlabs-ai-assistant-

Jointechlabs AI Assistant

A physician-facing assistant for Jointechlabs. It answers clinical and product questions about the MiniTC Adipose Processing System and related microfat technology. Every answer is grounded in Jointechlabs' own published content and cites its source.

Who it's for: surgeons, physicians and clinical staff evaluating or using MiniTC. What it is not: a substitute for the Instructions for Use (IFU), clinical training, or medical judgment. It does not give patient-specific advice.

Features
Grounded Q&A. Retrieves relevant passages from jointechlabs.com (pages, products, industry research, events, blog, PDFs) and answers only from them, with links back to the source.
Fast responses. Inference runs on the Groq API.
Follow-up email. Physicians can have the answer, with its sources, emailed to them via Nodemailer, so the conversation can continue with the Jointechlabs team.
Refreshable knowledge base. A scraper pulls content through the WordPress REST API. Re-runs can be incremental.
Architecture
                ┌──────────────────────────── ingestion (offline) ───────────────────────────┐
jointechlabs.com│  WordPress REST API ──► scraper/scrape.js ──► chunks.jsonl ──► embed + index │
                └────────────────────────────────────────────────────────────────┬───────────┘
                                                                                 │
                ┌──────────────────────────── serving (online) ──────────────────▼───────────┐
 Physician ───► │ Express API ──► retrieve top-k chunks ──► prompt + context ──► Groq LLM    │
   (web UI)  ◄──│            ◄── answer + citations ◄────────────────────────────────┘       │
                │            └──► /api/email-followup ──► Nodemailer ──► physician inbox     │
                └────────────────────────────────────────────────────────────────────────────┘
Layer	Technology
Server	Node.js 18+, Express
LLM	Groq API
Email	Nodemailer (SMTP)
Content source	WordPress REST API (/wp-json/wp/v2/...)
Retrieval	Chunked Markdown + embeddings (see Knowledge base)
Project layout
<!-- Adjust to match the repo -->
jointechlabs-ai-assistant/
├── server.js              # Express app entry point
├── routes/
│   ├── chat.js            # POST /api/chat
│   └── email.js           # POST /api/email-followup
├── lib/
│   ├── groq.js            # Groq client + prompt assembly
│   ├── retrieval.js       # load index, top-k search
│   └── mailer.js          # Nodemailer transport + templates
├── prompts/
│   └── system.md          # system prompt and clinical guardrails
├── public/                # chat UI
├── scraper/
│   ├── scrape.js          # WordPress → RAG chunks
│   └── output/            # documents.json, chunks.jsonl, manifest.json
├── .env.example
└── package.json
Getting started
Prerequisites
Node.js 18 or later
A Groq API key
SMTP credentials for the sending mailbox
Install
bash
git clone <repo-url> jointechlabs-ai-assistant
cd jointechlabs-ai-assistant
npm install
cp .env.example .env   # then fill in the values below
Configure
<!-- Variable names are suggested; align with the code -->
Variable	Required	Description
GROQ_API_KEY	yes	Groq API key
GROQ_MODEL	no	Model ID to use (set to your chosen Groq model)
PORT	no	HTTP port (default 3000)
SMTP_HOST	yes	SMTP server host
SMTP_PORT	yes	Usually 587 (STARTTLS) or 465 (TLS)
SMTP_USER	yes	SMTP username
SMTP_PASS	yes	SMTP password or app password
MAIL_FROM	yes	Sender, e.g. "Jointechlabs Assistant" <assistant@jointechlabs.com>
MAIL_BCC	no	Internal address copied on every follow-up (sales/clinical team)
RETRIEVAL_TOP_K	no	Chunks passed to the model per question (default 6)

Never commit .env.

Build the knowledge base
bash
cd scraper
npm install
node scrape.js            # full crawl, including PDFs
cd ..

Then embed scraper/output/chunks.jsonl into your index (see below).

Run
bash
npm run dev     # local development
npm start       # production

Open http://localhost:3000.

Knowledge base

The scraper (scraper/scrape.js) reads public content from the WordPress REST API rather than scraping HTML, which gives clean, structured data.

Content	REST base
Pages	pages
Blog posts	posts
Products	upcp_product
Industry research	industry-research
Events	events
PDFs (IFUs, brochures, papers)	media?mime_type=application/pdf

If a page built with a page builder returns almost no text through the API, the scraper fetches the rendered page instead.

Outputs (scraper/output/):

documents.json: one cleaned Markdown record per item
chunks.jsonl: ~600-token chunks split at headings, with 300-character overlap. Each chunk has:
text_for_embedding: section breadcrumb plus text. Embed this.
text: the passage alone. Send this to the model.
url, section, modified, chunk_id: for citations and updates
manifest.json: counts, duplicates skipped, page fallbacks, failures

Refreshing:

bash
node scrape.js --since 2026-10-01   # only content modified since a date
node scrape.js --no-pdf             # skip PDFs

Upsert chunks by chunk_id. Run a full crawl periodically and delete chunks whose doc_id no longer appears, so removed pages stop being cited.

API
<!-- Confirm paths and payloads against routes/ -->
POST /api/chat
json
// request
{ "message": "What volume of lipoaspirate does MiniTC process?", "history": [] }

// response
{
  "answer": "…",
  "sources": [
    { "title": "MiniTC", "section": "MiniTC > Specifications", "url": "https://jointechlabs.com/…" }
  ]
}
POST /api/email-followup
json
{
  "email": "dr.smith@example.org",
  "name": "Dr. Smith",
  "question": "…",
  "answer": "…",
  "sources": [ … ]
}

Sends the answer and its sources to the physician, and to MAIL_BCC if it is set.

Clinical guardrails

The system prompt (prompts/system.md) should enforce these rules. Treat them as requirements, not style choices.

Answer only from retrieved Jointechlabs content. If the context doesn't cover the question, say so and offer the follow-up email instead of guessing.
Cite every factual claim with a source link.
Stay within cleared indications. Don't suggest off-label uses or make efficacy claims beyond what the published material states.
Defer to the IFU for procedure steps, contraindications and warnings, and link to it.
No patient-specific advice. Don't diagnose, select patients, or advise on individual cases.
Don't collect patient information. The chat and email flows should hold no PHI. Remind users not to enter any.
Escalate. Questions about adverse events, complaints or regulatory matters go to the Jointechlabs team, not the model.

Have Jointechlabs' clinical or regulatory reviewer approve the system prompt and a sample of answers before launch.

Testing

Keep a small set of evaluation questions with expected source pages:

product specifications (should cite the product page or IFU)
an off-label question (should decline and redirect)
a question the site doesn't answer (should say so and offer email)
a patient-specific question (should decline)

Re-run the set after each content refresh or prompt change.

Troubleshooting
Symptom	Check
Answers miss obvious content	scraper/output/manifest.json for failures; confirm the page's type is in the scraper's contentTypes
A page's text is empty or just shortcodes	Look for it under used_page_fallback in the manifest; adjust extractMainFromPage selectors
Groq 401	GROQ_API_KEY
Groq 429	Rate limit; add retry with backoff, or lower traffic
Emails not delivered	SMTP credentials, port/TLS pairing, and SPF/DKIM for the MAIL_FROM domain
Ownership
Content: Jointechlabs. Website updates reach the assistant on the next scraper run.
Clinical/regulatory review: Jointechlabs clinical team
Engineering: <!-- maintainer name/contact -->
