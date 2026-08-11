# Connect your Lofty to Claude — the solo-agent quickstart

*From Lofty Master Class. Free. The easy on-ramp: your own Lofty, your own API key, no developer
account.*

This is the shorter of our two guides. It uses the API key that already lives **inside your Lofty
CRM** — the same one Lofty gives you for Zapier — so there is nothing to register and no developer
portal to sign up for. In about fifteen minutes you will be able to ask Claude
*"which of this week's leads still have no follow-up?"* and get a live answer from your own Lofty.

If your account does not show an API key where this guide points, or the key does not work, there is
a second path in the companion guide, [connect-lofty-to-claude-code-developer-key.md](connect-lofty-to-claude-code-developer-key.md), which uses
Lofty's developer portal. Start here first; it is the simplest route for a solo agent.

---

## What you are building, in plain terms

A small program that runs on your own computer — an **MCP server** — and acts as a translator: it
holds your Lofty API key in a private file, and when you ask Claude a question about your CRM, it
fetches the real answer from Lofty and hands it back. You are not giving Claude your Lofty password.
You are giving one small program you control a scoped key, and letting Claude ask that program
questions in plain English.

---

## Read this before you touch the key — four rules

Your Lofty API key can read your whole book of business. Treat it like your online-banking login.

1. **Never paste the key into a chat, a message, or anywhere public.** Not into Claude in the
   browser, not into a document, not into a GitHub commit. It goes in exactly **one** place: a local
   secret file on your own computer.
2. **Never commit the key.** When Claude Code sets up the folder, make sure the secret file (`.env`)
   is ignored by git before you save anything. Ask Claude Code to confirm it.
3. **Start read-only.** Build the tools that only *read* first — list leads, read notes, search
   listings. Live with it a week before you add anything that *changes* your data.
4. **If the key ever leaks, reset it in Lofty right away.** A reset key is a closed door.

---

## Step 1 — Find your Lofty API key

This path is verified in the live Lofty CRM (August 2026). Labels can shift between plans, but this
is where it lives:

1. In Lofty, click the **gear / Settings icon** in the narrow toolbar on the **top right** of the
   screen.
2. In the left menu under **My User Settings**, click **Integrations**.
3. Scroll down to the **API** area (Lofty's own help text calls this
   *"Setting → Integrations → API"* — it is the same section that powers the Zapier connection).
4. Copy your **API Key**. If there is a **Reset** button beside it, do not press it unless you
   intend to — resetting invalidates any existing connection using the old key (like a Zapier zap).
   You only need to *copy* the current key.

Keep that key somewhere safe for the next few minutes — a password manager, not a sticky note and
not a text message to yourself.

> ⚠ **If there is no API key here**, your seat or brokerage may not expose one. Do not force it —
> switch to the developer-portal route in [connect-lofty-to-claude-code-developer-key.md](connect-lofty-to-claude-code-developer-key.md), or, if
> you would rather we build it for you, see the bottom of this page.

---

## Step 2 — Hand the build to Claude Code

Open Claude Code in an empty folder (`mkdir my-lofty-mcp && cd my-lofty-mcp && claude`) and paste
this prompt. It tells Claude Code exactly what to build, and — importantly — to **verify the key
works on the very first call** so you are not guessing.

> **Prompt to paste into Claude Code:**
>
> Build me a local MCP server that lets Claude read my own Lofty CRM through the Lofty API. Details:
>
> - Python, using the official MCP SDK (`mcp` / FastMCP) and `httpx`. Keep it to `server.py`,
>   `requirements.txt`, and `.env.example`.
> - Read my key from an environment variable `LOFTY_API_KEY` in a local `.env` file. The API base is
>   `https://api.lofty.com`; send `Authorization: Bearer ${LOFTY_API_KEY}` on every request. Never
>   hard-code the key. Create a `.gitignore` that ignores `.env` first, and tell me you did.
> - **Before building the tools, add a `verify_connection` tool that makes one cheap read call**
>   (for example `GET /v1.0/members`, the team-members endpoint, or the current-user profile) and
>   reports back the HTTP status and a one-line result. I want to run this first to confirm my key
>   authenticates. If it returns 401 or 403, tell me plainly that this key is not accepted by the
>   API and I should try the developer-portal key instead — do not silently continue.
> - Then, **read-only** tools only, each wrapping one real endpoint and returning clean JSON:
>   `list_leads` (GET `/v1.0/leads`, pass through documented query filters), `get_lead`
>   (GET `/v1.0/leads/{leadId}`), `list_notes` (GET `/v1.0/notes`), `list_team_members`
>   (GET `/v1.0/members`), `search_listings` (POST `/v2.0/listings/search` with a JSON body).
> - Do NOT add any tool that writes, updates, or deletes yet. Do NOT call anything on
>   `cms.lofty.com` — that is a separate internal surface and is out of scope.
> - Fetch and read the current Lofty API reference at developer.lofty.com so paths and parameter
>   names are correct as of today. If a parameter is uncertain, make the tool accept a generic
>   `params` dict and tell me which ones the endpoint documents.
> - On any non-200 response, return the status code and Lofty's error body instead of crashing, so I
>   can see what went wrong.
> - When done, show me the exact command to register this server with Claude Code and one example
>   question to test it.
>
> Do not put my key anywhere except `.env`, and confirm `.env` is gitignored before you finish.

The `verify_connection` step is the important part. The key from inside your CRM is meant to work as
a normal Bearer token, but accounts and plans differ. Running one cheap read first tells you in ten
seconds whether you are good to go, instead of debugging five broken tools.

---

## Step 3 — Put the key in and register the server

1. Copy `.env.example` to `.env` and paste your key:
   ```
   LOFTY_API_KEY=your-key-here
   ```
2. Confirm `.gitignore` lists `.env`. Ask Claude Code to double-check.
3. Register it (Claude Code will give you the exact path):
   ```bash
   claude mcp add lofty -- python /full/path/to/my-lofty-mcp/server.py
   ```

---

## Step 4 — Verify, then use it

Start a fresh Claude Code session and first ask:

- *"Run verify_connection and tell me if my Lofty key works."*

If that comes back OK, you are connected. Now ask in plain English:

- *"List my leads from the last 7 days and flag the ones with no notes."*
- *"Show me everything on the lead named ⟨name⟩."*
- *"Search my active listings between 400k and 600k with 3+ beds."*
- *"Who is on my team?"*

If `verify_connection` returns 401 or 403, the in-CRM key is not accepted by the API on your account.
That is not a dead end — switch to the developer-portal key in
[connect-lofty-to-claude-code-developer-key.md](connect-lofty-to-claude-code-developer-key.md); the rest of the build is identical.

---

## When you are ready for more

Once the read-only version has earned your trust, ask Claude Code to add write tools **one at a
time**, testing each against a throwaway test lead before a real one: add a note to a lead, update a
lead, add tags. Each new tool is a new key to a new door — add them deliberately, never in a batch.

---

## The honest limits

- This talks to Lofty's **API** (`api.lofty.com`) only. It does not touch your published website,
  the page builder, or anything on `cms.lofty.com` — building your area-authority pages there is a
  separate craft.
- The in-CRM API key is a real credential sitting on your machine. Every rule in the security section
  stays true for as long as the server exists.
- Lofty is reorganizing its developer documentation, but the API itself is stable. If a doc link
  breaks, the API still answers; have Claude Code work from the live reference.

---

## Would rather we build it for you?

Most agents would, and that is a fair choice. Lofty Master Class exists for exactly this — the free
community and the live sessions teach the thinking, and when you want the connection built, the
follow-up automated, and your **area-authority pages** built so your own site becomes the answer
people and AI engines land on for your market, that is the Trust Engine we build.

- **Free:** join and bring your questions to a live session — [loftymasterclass.com](https://loftymasterclass.com).
- **Done for you:** book a call — [experience.mostreferred.expert](https://experience.mostreferred.expert).

The guide above is yours either way. That is the point.

---

*Lofty Master Class · Most Referred Expert. Automate the manufacturing, keep the judgement human.*
