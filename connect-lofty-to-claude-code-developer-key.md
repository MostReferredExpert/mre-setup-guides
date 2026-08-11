# Build a Lofty MCP for your own Lofty — the developer-key route

*A do-it-yourself guide from Lofty Master Class. Free. Yours to keep.*

> **Most agents should start with the easy guide:**
> **[Connect Your Lofty to Claude Code](connect-lofty-to-claude-code.md)** uses the API key already
> in your Lofty settings — no developer account. Use *this* page only if your Lofty seat does not
> expose an in-CRM key, or that key does not authenticate. The MCP you build is identical either way.

By the end of this you will be able to open Claude (or Claude Code) and say
*"how many leads came in this week and which ones have no follow-up yet"* and get a real answer
pulled live from **your** Lofty account. No spreadsheet export, no copy-paste. You are building a
small bridge — an MCP server — that lets an AI read and act on your own CRM, safely, on your own
machine.

This is the DIY path. If you would rather we build and run it for you, that is the door at the
bottom. Either way the knowledge here is free.

---

## What you are actually building, in one paragraph

An **MCP server** (Model Context Protocol) is a tiny program that runs on your computer and speaks
two languages: it talks to Claude on one side, and to the Lofty API on the other. You give it your
Lofty API key once, kept in a secret file that never leaves your machine. Then Claude can call it
the way it calls any other tool — *"list this week's leads," "show me this lead's notes,"
"search my active listings under 600k"* — and the MCP fetches the answer from Lofty and hands it
back. You are not giving Claude your password. You are giving a small program you control a scoped
key, and letting Claude ask that program questions.

---

## Before you start — the one thing that gates everything

**You need three things:**

1. **A Lofty seat.** You already have this if you log into Lofty.
2. **A Lofty developer account + an API key.** This is self-serve at
   [developer.lofty.com](https://developer.lofty.com) — create a developer account, register an
   application, and generate a **personal access token (API key)** or set up **OAuth 2.0**. The key
   is scoped to *your own team's data*, resolved from the token. Nobody else's leads, only yours.
3. **Claude Code or the Claude desktop app**, and about 20 minutes.

⚠ **If you cannot get an API key** — some brokerage or team configurations do not expose developer
access to individual agents — do not force it. Skip to *"When the DIY path is not open to you"* at
the bottom. There is a supported route.

You do not need to be a developer. Claude Code writes the program. Your job is to get the key,
keep it safe, and check the result.

---

## The security rules — read these before anything else

Your Lofty API key can read your entire book of business. Treat it like your online-banking login.
These four rules are not optional.

1. **Never paste your API key into a chat prompt, a message, or a public place.** Not to Claude in
   the browser, not into this file, not into a GitHub commit. The key goes in **one** place: a local
   secret file (a `.env`) that stays on your computer.
2. **Never commit the key.** If Claude Code sets up a project folder, make sure `.env` is listed in
   `.gitignore` before you ever run `git add`. Ask Claude Code to confirm this for you.
3. **Start read-only.** Build the tools that *read* first — list leads, read notes, search listings.
   Live with it for a week. Only add tools that *change* data (update a lead, add a note) once you
   trust it and understand exactly what each one does. A read tool that is wrong shows you bad
   information. A write tool that is wrong changes your CRM.
4. **If a key ever leaks, rotate it immediately.** In your Lofty developer account, revoke the old
   key and generate a new one. A rotated key is a closed door; a leaked key you leave active is an
   open one.

---

## Step 1 — Get your Lofty API key

1. Go to [developer.lofty.com](https://developer.lofty.com) and create a developer account (use the
   same email as your Lofty login).
2. Register an application. Give it a name like `my-lofty-mcp`.
3. Generate a **personal access token / API key**. Copy it once — you may not see it again.
4. Put it somewhere safe for the next five minutes (a password manager, not a sticky note, not a
   text message to yourself).

That token is the only credential the rest of this needs. The API answers to
`https://api.lofty.com` and every call carries your token in an `Authorization: Bearer <token>`
header. Claude Code handles that plumbing; you just supply the token once.

---

## Step 2 — Hand Claude Code the build

Open Claude Code in an empty folder (`mkdir my-lofty-mcp && cd my-lofty-mcp && claude`), and paste
the prompt below. It tells Claude Code exactly what to build and, just as importantly, what **not**
to do.

> **Prompt to paste into Claude Code:**
>
> Build me a local MCP server that wraps my own Lofty CRM's public Developer API so I can query my
> Lofty from Claude. Requirements:
>
> - Language: Python, using the official MCP SDK (`mcp` / FastMCP) and `httpx`. Keep it to a single
>   `server.py` plus a `requirements.txt` and a `.env.example`.
> - Auth: read my API key from an environment variable `LOFTY_API_KEY` loaded from a local `.env`
>   file. Every request goes to `https://api.lofty.com` with the header
>   `Authorization: Bearer ${LOFTY_API_KEY}`. Never hard-code the key. Create a `.gitignore` that
>   ignores `.env` before anything else, and tell me you have done it.
> - Start **read-only**. Expose exactly these tools, each calling the real Lofty endpoint and
>   returning clean JSON:
>   - `list_leads` — GET `/v1.0/leads` (support optional query params like page/size and any
>     documented date or status filters; read the spec, do not guess parameter names).
>   - `get_lead` — GET `/v1.0/leads/{leadId}`.
>   - `list_lead_notes` — GET `/v1.0/notes` filtered to a lead, or the lead-notes endpoint if the
>     spec exposes one; check the spec.
>   - `list_team_members` — GET `/v1.0/members`.
>   - `search_listings` — POST `/v2.0/listings/search` with a JSON body I can pass through.
> - Do NOT add any tool that writes, updates, or deletes yet. Do NOT call anything on
>   `cms.lofty.com` — that is a different, internal surface and is out of scope.
> - Before you write the tools, fetch and read the current Lofty API reference at
>   developer.lofty.com so the paths, parameters, and response shapes are correct as of today. If a
>   parameter name is uncertain, make the tool accept a generic `params` dict and tell me which
>   parameters the endpoint documents.
> - Handle errors gracefully: on a non-200 response, return the status code and the Lofty error body
>   rather than crashing, so I can see what went wrong.
> - After building, show me the exact command to register this server with Claude Code, and one
>   example question I can ask to test it end to end.
>
> Do not put my API key in any file except `.env`. Confirm `.env` is gitignored before you finish.

Claude Code will read the live Lofty docs, write the server, set up the secret file, and tell you
how to run it. Read what it produces. If it tries to add a write tool, or reference `cms.lofty.com`,
stop it — those are outside what this first version should do.

---

## Step 3 — Put your key in, and register the server

1. Claude Code will have created a `.env.example`. Copy it to `.env` and paste your real key in:
   ```
   LOFTY_API_KEY=your-token-here
   ```
2. Confirm `.gitignore` contains `.env`. Ask Claude Code to double-check if you are unsure.
3. Register the server with Claude Code. The command Claude Code gives you will look roughly like:
   ```bash
   claude mcp add lofty -- python /full/path/to/my-lofty-mcp/server.py
   ```
   (Claude Code will fill in the exact path and interpreter for your machine.)

---

## Step 4 — Try it

Start a fresh Claude Code session and ask, in plain English:

- *"List my leads from the last 7 days and tell me which have no notes yet."*
- *"Show me everything on the lead named ⟨name⟩."*
- *"Search my active listings between 400k and 600k with 3+ beds."*
- *"Who is on my team, and which of them can take new lead assignments?"*

If the answers match what you see in Lofty, your bridge works. You now have an AI that reads your
CRM in your own words.

---

## What good looks like — the reference shape

You do not need to write this; Claude Code does. It is here so you can sanity-check what you get.
A correct minimal server reads the key from the environment, sets the Bearer header once, and each
tool is a thin, honest wrapper around one real endpoint:

```python
# server.py  (reference shape — Claude Code will write the real one against the live spec)
import os, httpx
from mcp.server.fastmcp import FastMCP

API = "https://api.lofty.com"
KEY = os.environ["LOFTY_API_KEY"]          # from .env, never hard-coded
HEADERS = {"Authorization": f"Bearer {KEY}", "Content-Type": "application/json"}

mcp = FastMCP("lofty")

@mcp.tool()
def list_leads(params: dict | None = None) -> dict:
    """List leads from your own Lofty team. `params` = documented query filters."""
    r = httpx.get(f"{API}/v1.0/leads", headers=HEADERS, params=params or {}, timeout=30)
    return {"status": r.status_code, "data": r.json() if r.is_success else r.text}

@mcp.tool()
def get_lead(lead_id: str) -> dict:
    """Get one lead by id."""
    r = httpx.get(f"{API}/v1.0/leads/{lead_id}", headers=HEADERS, timeout=30)
    return {"status": r.status_code, "data": r.json() if r.is_success else r.text}

@mcp.tool()
def search_listings(body: dict) -> dict:
    """Search listings. `body` = the documented v2 search payload."""
    r = httpx.post(f"{API}/v2.0/listings/search", headers=HEADERS, json=body, timeout=30)
    return {"status": r.status_code, "data": r.json() if r.is_success else r.text}

if __name__ == "__main__":
    mcp.run()
```

The pattern is deliberately boring: one tool, one endpoint, the error body returned instead of
swallowed. Boring is what you want holding a key to your book of business.

---

## Growing it later — only when you are ready

Once the read-only version has earned your trust, you can ask Claude Code to add, one at a time:

- **Write tools** — add a note to a lead (`/v1.0/leads/{leadId}/notes`), update a lead
  (`PUT /v1.0/leads/{leadId}`), add tags (`/v1.0/agent/{agentId}/tag/add`). Add these **one at a
  time**, and test each against a throwaway test lead before you point it at a real one.
- **Saved questions** — teach it the five questions you ask every Monday, so a single prompt runs
  your whole weekly review.
- **A follow-up finder** — leads with no contact in N days, ranked by source or price band.

Each new tool is a new key handed to a new door. Add them deliberately.

---

## Troubleshooting

- **`400 · "User in token does not exist"`** — the key is dead or was revoked. Generate a fresh one
  in your developer account.
- **`401 / 403`** — the header is wrong or the token is not `Authorization: Bearer <token>`. Ask
  Claude Code to print the exact request it is sending (with the key redacted).
- **Empty results but no error** — the endpoint is fine, your filter is too narrow. Loosen the
  params.
- **Claude cannot see the server** — the MCP is not registered, or the path in `claude mcp add` is
  wrong. Re-run it with the absolute path to `server.py`.

---

## The honest limits

- This talks to Lofty's **public Developer API** only (`api.lofty.com`). It does not touch the CMS
  website builder, your published site's pages, or anything on `cms.lofty.com` — those are a
  separate surface with their own rules, and building your area-authority pages there is its own
  craft.
- The Lofty developer **documentation** is being reorganized, but the endpoints themselves are
  stable. If a doc link 404s, the API still answers; ask Claude Code to work from the live
  reference.
- An MCP that reads your CRM is a real credential living on your machine. Everything in the security
  section stays true for as long as the server exists.

---

## When the DIY path is not open to you

If you cannot get a developer key, or you would simply rather not run a program on your own machine,
that is a reasonable choice — most agents make it. Lofty Master Class exists for exactly this:
the free community and the live sessions teach the thinking, and when you want it built and run for
you — the CRM connected, the follow-up automated, and your **area-authority pages** built so your
own site becomes the answer people and AI engines land on for your market — that is the Trust Engine
we build.

- **Free:** join the community and bring your questions to a live session —
  [loftymasterclass.com](https://loftymasterclass.com).
- **Done for you:** book a call and we build the connection, the pages, and the system —
  [experience.mostreferred.expert](https://experience.mostreferred.expert).

Either way, the guide above is yours. That is the point.

---

*Lofty Master Class · Most Referred Expert. Automate the manufacturing, keep the judgement human.*
