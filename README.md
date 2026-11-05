# Avenor AI

A self-hosted Telegram AI assistant that answers in your language, reads the live web, builds real
files (HTML, PDF, CSV, code) and improves its own source code — with the owner approving every
change. One Python file, one JSON store, no database to run.

- **Bot:** [@AvenorOrbitBot](https://t.me/AvenorOrbitBot)
- **Group:** [Avenor AI Community](https://t.me/AvenorAICommunity)
- **Repository:** [github.com/bydenvik/avenor-ai-telegram-bot](https://github.com/bydenvik/avenor-ai-telegram-bot)
- **Developer:** [bydenvik](https://github.com/bydenvik)
- **Licence:** MIT — see [LICENSE](LICENSE)

---

## What it does

**Talks like a person, in the person's language.** Replies to any private message without a command,
and in a group when it is mentioned or replied to. It mirrors Hindi, Hinglish or English from the
message itself, and it never introduces itself or repeats the question back. If a message is aimed at
someone else, it stays quiet.

**Reads the live web.** A factual question triggers a real lookup — DuckDuckGo, Wikipedia, Google News,
arXiv — and the answer is built from what came back, with the sources named. If a model ever claims it
has no internet access, the reply is discarded and rebuilt from the fetched data.

**Builds real files.** `[file:]` is a first-class output: Markdown, text, CSV, JSON, Python, HTML, CSS,
SQL, YAML — and `.pdf`, written by a small PDF writer inside the bot, so "make me a PDF" produces a
document you can open, not text with a `.pdf` name. A reply that promises a file always carries one.

**Reads links, images and documents.** YouTube (title, views, comments, thumbnail — the thumbnail is
passed to the model as an image), Instagram, X/Twitter, Reddit, GitHub, and any ordinary page. Photos
and PDFs/DOCX sent to the bot are read before answering.

**Remembers carefully, forgets on request.** Per-user and per-chat memory in one JSON file: name,
language, notes, reminders, projects. `/memory` shows what is stored, `/forget` deletes all of it in
one step.

**Improves itself — with the owner in the loop.** `/brain` shows the failures it recorded about its
own behaviour; `/improve` turns them into one surgical code change, which the owner approves with a
button before anything is written. See [Self-improvement](#self-improvement).

**Stays safe in groups.** Spam scoring with strikes → warn → mute → ban, a malicious-file and link
scan, and an SSRF guard on every outbound fetch.

---

## Stack

| Layer | Choice |
|---|---|
| Language | Python 3.10+ (3.12 recommended) |
| Telegram | python-telegram-bot 22.8 |
| AI | `google-genai` — Gemini chain, Groq fallback, Pollinations last resort |
| Storage | One JSON file, debounced atomic writes |
| Hosting | Render, Docker, or any VPS |

| File | What lives there |
|---|---|
| `bot.py` | Handlers, commands, model chain, research, file building, self-improvement |
| `memory.py` | The JSON store: users, chats, notes, reminders, profiles |
| `.env.example` | Every setting, with its default |
| `privacy-policy.md` | What is stored and what leaves the server |

---

## Quick start

```bash
git clone https://github.com/bydenvik/avenor-ai-telegram-bot.git
cd avenor-ai-telegram-bot
pip install -r requirements.txt
cp .env.example .env          # then fill in BOT_TOKEN and GEMINI_API_KEY
python bot.py
```

Two variables are enough to start: `BOT_TOKEN` from [@BotFather](https://t.me/BotFather) and a free
`GEMINI_API_KEY` from [Google AI Studio](https://aistudio.google.com/app/apikey). Everything else has a
working default.

`.env.example` is a **template** — editing it changes nothing. The bot reads the real environment
(Render → Environment, or the `.env` file you just created).

---

## Configuration

| Variable | What to put |
|---|---|
| `BOT_TOKEN` | Token from [@BotFather](https://t.me/BotFather) |
| `GEMINI_API_KEY` | Free key from Google AI Studio |
| `OWNER_IDS` | Your Telegram ID — `/id` gives it to you |
| `GROQ_API_KEY` | Optional but recommended: a second provider when Gemini is rate-limited |
| `GITHUB_TOKEN` | Only for self-improvement commits (fine-grained, one repo, Contents: write) |

The full list — model chain, effort budgets, voice, group behaviour, memory limits — is in
[`.env.example`](.env.example) with a note on every line.

**Why a model chain instead of one model.** Free tiers rate-limit. The bot walks
`GEMINI_MODELS` in order, drops models that answer 404 or repeat a 500, then falls back to Groq, then
to Pollinations. `/diag` shows which are alive right now and prints the exact `GEMINI_MODELS=` line to
paste once you have pruned the dead ones.

---

## Commands

**Everyone** — `/start` `/help` `/skills` `/about` `/id` `/info` `/ai` `/research` `/image` `/voice`
`/voicemessage` `/setvoice` `/voicepreview` `/remind` `/reminders` `/delremind` `/feed` `/papers`
`/define` `/holiday` `/myphoto` `/logo` `/memory` `/proactive` `/privacy` `/clear` `/forget` `/rules`
`/setup` `/welcome` `/stats` `/safety` `/members`

**Owner** — `/improve` `/brain` `/diag` `/voicecreate` `/voicelist` `/voicefind`

`/help` shows each role only the commands it can use, so a member never sees a list they cannot run.

---

## Effort routing

Every message is classified before a model is called, so a "hello" does not pay for a research run.

| Effort | Trigger | Budget |
|---|---|---|
| `fast` | short chat, greetings | minimal thinking, small output |
| `standard` | a normal question | low thinking |
| `deep` | code, debugging, architecture, builds | high thinking, largest budget |

---

## Research and links

A lookup runs only when the message needs one — a factual question, a link, or an explicit request.
The bot fetches, keeps the useful lines, and hands the model a compact block instead of raw HTML.

When every provider is rate-limited the reply is short and honest ("the model is busy, try again in a
minute"). It never pastes a wall of raw search results into the chat.

---

## Self-improvement

The bot can propose a change to its own source. It can never apply one on its own.

1. `/brain` (owner, private chat) shows the failures it recorded: a denied capability, a wrong skip, a
   promised file that was never attached, a provider outage.
2. `/improve [topic]` writes **one** hunk — exact old lines, exact new lines, one line of reasoning.
3. The hunk is validated before you see it: the file must be on an allowlist (`bot.py`, `memory.py`,
   `README.md`, `.env.example`), the old block must match exactly once, the hunk must be under
   `IMPROVE_MAX_LINES`, and it may not contain `subprocess`/`eval`/`exec` or anything secret-shaped.
4. You get a diff with **Apply & commit** / **Reject**.
5. Apply re-validates, applies to a temp copy, runs `ast.parse` and `py_compile`, and only then commits
   through the GitHub API. Any failure means nothing is written.

**Which repository?** Exactly one, and it is not the model's choice. `GITHUB_REPO` is a single value you
set; the code can only call `/repos/{GITHUB_REPO}/contents/{allowlisted file}`. There is no
repository-creation call anywhere, and no chat message can change the target.

**Credentials.** A GitHub App (`GITHUB_APP_ID`, `GITHUB_INSTALLATION_ID`, `GITHUB_PRIVATE_KEY`) mints a
token that expires in an hour and reaches one repository. A fine-grained personal token with
`Contents: Read and write` on that single repository also works. `SELF_IMPROVE_DRY_RUN=true` validates
and compiles without writing anything.

**Is it on?** `SELF_IMPROVE` is three-state: `true` forces it on, `false` forces it off, and unset means
"on when a GitHub credential exists". `/brain` always shows which is in effect, and `/improve on` /
`/improve off` override it for the current run.

**What it learns.** Only de-identified failure patterns — "the model denied having web access", not who
said what. IDs, @usernames, emails and phone numbers become `[id]` before anything is stored, and the
chat it came from is not recorded.

---

## Privacy

`/privacy` explains what is stored; `/forget` deletes it. In short: memory lives in one JSON file on
your server, message text goes to the AI provider to produce an answer, and nothing is sold or used to
train a model. Full detail in [privacy-policy.md](privacy-policy.md).

---

## Group safety

Spam is scored on length, links, repetition, emoji runs and forwards; strikes escalate to mute and
ban. Every uploaded file and every link is scanned before it is opened. The SSRF guard resolves the
host and refuses private, loopback and link-local addresses, with a port allowlist.

`/safety` shows the rules and the most recent blocks. `/rules` posts and pins the group rules.

---

## Project layout

```
avenor-ai-telegram-bot/
├── bot.py              Handlers, commands, model chain, research, files, self-improvement
├── memory.py           The JSON store
├── requirements.txt    Dependencies
├── .env.example        Environment template (a template — the bot reads the real environment)
├── privacy-policy.md   What is stored and what leaves the server
├── .gitignore          Must be named exactly ".gitignore", or git reads none of it
├── LICENSE.txt         MIT
└── README.md
```

---

## Troubleshooting

| Symptom | Cause |
|---|---|
| "Avenor is a little busy" | every provider is rate-limited — run `/diag`, check quotas, add `GROQ_API_KEY` |
| `/improve` says the feature is off | no GitHub credential, or `SELF_IMPROVE=false` — `/brain` says which |
| A model shows ❌ in `/diag` | 404/500 on Google's side — paste the `GEMINI_MODELS=` line it prints |
| Files not arriving | `MAX_FILE_BYTES`, or the person is over the daily action cap |
| Voice replies silent | `ffmpeg` missing on the host |

---

## Licence

MIT. Keep the copyright notice. See [LICENSE](LICENSE).

---

<sub>telegram ai bot · gemini telegram bot · groq fallback · python telegram bot · self-hosted ai
assistant · telegram chatbot hindi · telegram pdf generator bot · telegram image bot · ai agent
telegram · open source telegram bot · render deploy telegram bot</sub>
