---
name: outsystems-agent-session
description: Drive an OutSystems environment through an AI agent over MCP — the hosted ODC server plus Mentor, or the local Service Studio server in O11. Use this before connecting an agent to any OutSystems tenant, and whenever a turn fails or stalls, a quota error appears (OS-AISA-42903, OS-AISA-42902, OS-AISA-42901, OS-AISA-40001, OS-AIWA-50001), a publish hangs or reports indeterminate, the agent starts inventing entities or actions that do not exist, the MCP connection refuses to register, or someone is about to treat "published with 0 errors" as proof that something works. Use it when deciding what to send through the agent and what to paste by hand, and whenever an OutSystems MCP server is being registered in a host.
---

# Driving OutSystems through an agent

A working agreement from roughly a month of driving a tenant this way. Not a tool tour — the habits
that stopped turns being wasted, stopped the quota running out, and stopped wrong results being
believed.

## Know which server you are on

There are two, and they behave nothing alike.

| | **ODC** | **O11** |
|---|---|---|
| Where it runs | hosted, per tenant | **locally, inside Service Studio** (11.55.91+) |
| Endpoint | the tenant's `/mcp` | `http://127.0.0.1:<dynamic port>/mcp` |
| Auth | OAuth against the tenant | **none** — loopback only |
| Scope | the tenant's apps | **the module currently open** in Service Studio |
| How edits happen | the ODC agent (Mentor) edits the app model server-side | the agent writes **C# against the Model API**, run on a **copy** of the module |
| Where you review | the published/unpublished model | **Compare and Merge**, before anything lands |

```
claude mcp add --transport http servicestudio http://127.0.0.1:<PORT>/mcp
```

Two consequences people miss. On O11 the port is dynamic and the server has **no authentication** —
it is loopback-bound, but anything already running on that machine can reach it, so treat it as a
local-dev tool, not something to leave registered on a shared or jump box. And the O11 review step
is a real safety net the ODC path does not have: you see a diff in Compare and Merge before merging,
so writes there are recoverable in a way an applied Mentor edit is not.

On ODC, read operations are generally stable and writes newer — check the current status of the
tooling rather than assuming, and keep write permissions scoped to people who should have them.

## Plan outside, execute inside

This is the single biggest change available, it is free, and everything else here is smaller.

**Use two sessions.** The first is *not* connected: it reads the repository, the platform CSS, the
article, the existing code; it argues about the approach, changes its mind, gets corrected, and
produces a finished **action plan or ready-to-paste prompt**. Only then does that prompt go into the
session that *is* connected.

Why it works: **quota is spent per turn, not per unit of thinking.** All the exploration, the dead
ends and the rewrites happen where they cost nothing, and the expensive turn is paid exactly once,
with a fully formed request.

What the planning session should hand over:

- the elements to create, named, with their types and parameters
- the exact order to create them in
- which parts are payload to be pasted by hand rather than sent through the agent
- what to report back **instead of** publishing, so you can check before anything goes live

It also removes the worst failure mode: a half-specified request the agent half-guesses, which costs
another turn to correct.

## One thing per turn, on the strongest model

Our first real attempt sent a Script and a Block in the same turn. It blew the quota with
`OS-AISA-42903` and published nothing. Split in two, both passed.

The same applies to screens. "Build me the whole data model" or a multi-screen request is the most
reasoning-heavy thing you can ask, and the platform's own recommendation is to run that kind of turn
on **the strongest model tier available**. A weaker model tries, fails, retries, and spends *more*
than a strong model that gets it right the first time.

Measured, not assumed:

- **three quota blowouts in three days**, every one on a turn asking for several screens at once
- a turn on a **mid-tier model with low reasoning effort** finished "clean", 0 errors — with an
  `internal_retry_count` of **14**. The retries are invisible in the result and very visible in the
  quota

Note this is about the model driving *your* side of the conversation, not only the platform agent. A
cheaper model is not cheaper if it retries fourteen times.

The rule that settled: **one element, or one screen, per turn, publishing between each.** Less per
day, and a blowout in the middle costs at most one screen.

## Model through the agent, content by hand

The most useful distinction we found.

**The agent is good at structure.** Blocks, Input Parameters, Events, Client Actions, lifecycle
handlers, screens, entities — produced exactly as specified, verified afterwards against the
published JavaScript.

**The agent rewrites content.** Two measured cases, both *after* an explicit instruction to copy
literally: one escape sequence inside a CSS comment changed (`\xE7` became `\xE3`), and a
column-configuration JSON came back with labels changed, columns reordered and variant keys
translated from Portuguese to English. The JSON was still valid, so nothing failed loudly — badge
colours just silently stopped matching the data.

Large content does not merely change, it fails: a ~106 KB stylesheet killed a turn with
`OS-AIWA-50001`. Pasting the same file into Studio is instant and consumes **no quota**.

So: let the agent build the shape, paste the payload yourself.

## Validation is not a test

"Published with 0 errors" is static analysis of the model. It does not mean the component renders,
the JSON parses, or the events fire — a far-too-simple `render()` publishes perfectly cleanly.

Our nine persistent warnings, for the record, were all Offline Behavior warnings caused by inline
SVG `data:` URLs in the theme CSS: permanent, harmless, and entirely unrelated to whether anything
worked. Warnings are not a progress bar.

## Verify against the published artifact

Every app lists the versioned files it loads:

```
https://<your-app-url>/moduleservices/moduleinfo
```

Download the theme CSS, the Scripts and the JS bundles and compare them against disk, normalising
two platform behaviours first:

| Asset | What the platform does | How to compare |
|---|---|---|
| Theme CSS | minifies: `0.625rem` → `.625rem`, `::before` → `:before`, colours to hex, identical rules merged | ignore those rewrites |
| Script | does not minify: reindents, CRLF → LF, adds a BOM | ignore whitespace |

Two facts that shape what is even checkable:

- **A Block no screen uses is not in the bundle.** Plan a throwaway screen before trying to audit
  generated JS nodes.
- **Any value that must match data** — JSON, keys, ids — is read from the published app, never from
  the turn's own report. That is the rule the rewritten JSON taught us.

Then open the screen with DevTools: Elements shows whether the component took over its container,
Console shows what it complained about.

## The quota: codes and what to do

On ODC the Mentor has a **daily quota counted per tenant and per organisation — not per user.** On a
shared tenant you are spending a team resource.

| Code | What it is | What to do |
|---|---|---|
| `OS-AISA-42903` | daily usage limit reached, per tenant/organisation | wait for the reset time in the message. The exact quota is not published; it resets once a day |
| `OS-AISA-42902` | too many calls **per minute** on the tenant | if several people share the tenant, coordinate — this is the official recommendation |
| `OS-AISA-42901` | too many calls to the AI provider | wait a few minutes |
| `OS-AISA-40001` | the conversation hit its size limit | resume with fresh context, do not start a new session |
| `OS-AIWA-50001` | the turn failed on a large payload | paste that file by hand in Studio instead |

A free or trial tenant has a tighter quota than a paid one. That is the expected level, not a bug.

### Resume the session, never start a new one

After a failed or cancelled turn the instinct is to start over. Don't. A failed turn still returns
the session id and a new session token, and **resuming continues from where it stopped**. Starting
fresh spends another session slot *and* discards edits that were never published.

When the conversation itself is the problem — size limit, or the agent has started inventing
entities and actions that do not exist, or you are switching to a different task in the same app —
resume with **fresh context** rather than a new session. That resets only the conversation, keeping
the session, the server-side model and any unpublished edits.

### Publish each milestone before asking for the next

Until you publish, everything lives only in the server-side session. If it expires or is collected,
unpublished work is gone and the next turn starts from zero.

The order that worked: publish the **data model**, then the **backend**, then screens **one at a
time**, publishing between each. Then a failure in the middle costs one turn, not the whole build.

### Do not poll a running turn

Each status check of a running turn **costs a full model turn**. Checking every few seconds burns
quota while producing nothing — consumption that does not look like consumption.

| What you are polling | Sensible interval |
|---|---|
| an agent turn's status | at least ~30 seconds |
| a publish or deploy status | 5 to 15 seconds is fine |

## Publishing discipline

- **Confirm before publishing.** Have the agent report what it is about to do and approve it
  explicitly. Publishing is the irreversible part of the loop.
- **Never start a second publish of the same app while the first is running.** Two revisions
  building at once can wedge the build engine, and a wedged app keeps failing with configuration and
  timeout errors until someone recovers it in Studio.
- **A failed publish carrying an `OS-*` platform code is the build's own verdict.** Surface the
  code; do not blindly retry.
- **A failure marked `indeterminate` is different**: the outcome was never observed, so the publish
  may still be building and may yet succeed. Re-poll, or verify the deployed inventory. Do not
  republish.

## Connection trouble: walk the OAuth chain

The first ODC connection can fail with `Dynamic Client Registration rejected (HTTP 404)`. The
message names the right step with the wrong code, so searching for "404" leads nowhere.

**Restart the client first** — closing and reopening the terminal made it redo discovery and
connect. It is the cheap move. When it does not help, walk the discovery chain the client walks;
each step tells you whether the problem is yours or the server's configuration:

| Step | Expected |
|---|---|
| `/mcp` with no credential | 401 with a `WWW-Authenticate` header |
| the URL that header points at, `/.well-known/oauth-protected-resource/...` | metadata naming the identity provider |
| the provider's registration endpoint | should accept the registration |
| the server's own `/mcp/register` | 400 to an empty POST — alive and validating |

In our case the provider's endpoint answered **403** while the server's own answered **400**: a
tenant-side configuration mismatch, with five lines of evidence ready for a support ticket.

Also: register the server where you will actually work. A terminal session and an IDE extension are
**separate sessions with separate MCP state** — authenticating one does not authenticate the other,
and a resumable agent session belongs to the session that created it. Pick one place and stay there.

## Environments and data

**Never point an agent at a customer environment for experiments.** Treat anything in a company
environment as customer code: do not publish it elsewhere and do not document internal apps outside
the work environment. Learn on a personal or throwaway environment.

If that personal environment is a free tier, know its lifecycle before relying on it: it hibernates
after roughly 72 hours of inactivity (recoverable), and after longer inactivity it is **deleted**,
with an email warning about three days ahead and a download link valid for seven days. There is no
backup and no recycle bin — the downloaded `.oml` is the only way back.

And what comes back is not identical. After reimporting, **revision history restarts at 1**, and any
file pasted by hand after the last export comes back as the **older version** — silently, because
the app publishes with 0 errors either way. Our hand-pasted 75 KB theme CSS survived intact; a
JavaScript bundle came back one revision behind, and only a file comparison revealed it.

## Checklist before you spend a turn

- Was this prompt written in a session that is **not** connected?
- Is it one element, or one screen — not a batch?
- Is the payload (stylesheet, bundle, long JSON) going in by hand instead of through the agent?
- Is the previous milestone already published?
- If the last turn failed, am I **resuming** that session rather than starting a new one?
- Am I on the strongest model tier available for a heavy turn?
- Does the prompt end by asking for a report, so I approve before anything is published?
- Is this a personal or throwaway environment, and not a customer one?

## A sane first session

1. Connect, then **search before editing**: list the apps, search elements by name. Real tenant data
   coming back proves the connection end to end.
2. Read an app's revisions and metadata. Still read-only.
3. Make one small structural change through the agent, on a throwaway app.
4. Publish it, with explicit approval.
5. Verify through `moduleinfo` and DevTools.

Step 5 is the one people skip, and it is the one that catches the rewritten JSON.
