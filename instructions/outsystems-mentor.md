# One element per turn — driving Mentor from Claude

**Use when** Claude Code is connected to an OutSystems environment through the MCP server and is
driving **Mentor**, the ODC agent that actually edits the app.

Two agents are involved and it helps to keep them apart:

| | Claude Code | Mentor |
|---|---|---|
| What it is | the session you talk to | the ODC-side agent that edits the app model |
| Where it runs | your machine | the OutSystems environment |
| What it costs | your model usage | **the environment's quota**, counted per tenant and per organisation |
| What it is good at | reading the repo, planning, checking the result | building platform elements exactly as specified |

So the request travels: you → Claude → the MCP server → Mentor → the app's model on the server.
Everything below is about the shape of what Claude sends down that chain.

## Mentor is better at a narrow request than a broad one

This is not a budget rule dressed up as advice — the output is genuinely better.

Given one element with every parameter named, Mentor builds it **exactly**: Blocks, Input
Parameters, Events, Client Actions, lifecycle handlers, screens and entities all came back as
specified, verified afterwards against the published JavaScript.

Given a broad request — "build the data model", "create these four screens" — it has to decide what
you meant. It guesses, retries internally, and produces something plausible you then have to audit.
Our first real attempt sent a Script and a Block in the same turn: it exhausted the daily quota with
`OS-AISA-42903` and published **nothing**, neither of them. Split into two turns, both passed.

Three quota exhaustions in three days, every one on a turn that asked for several things at once.
None on a turn that asked for one.

And the cost is not proportional to the size of the ask. A turn run on a mid-tier model with low
reasoning effort reported success — 0 errors — with an `internal_retry_count` of **14**. The retries
are invisible in the result and fully visible in the quota. For heavy turns, use the strongest model
tier available: a cheaper model is not cheaper if it retries fourteen times.

## The prompt

```
Create exactly one element in <app / module>, and nothing else.

Element: <type — Block, Client Action, Screen, Entity, Structure>
Name: <name>
<Input Parameters / Attributes / Events, each with its type and default>

Do not create anything that is not listed above. Do not publish.

When you are done, report back:
- the element you created and every parameter, with its type
- anything you changed, renamed or added that I did not ask for
- what you would need next

I will review and tell you whether to publish.
```

## Why each clause is there

**"Nothing that is not listed above."** Helpfulness is the failure mode. Ask for a Block and Mentor
will often add the screen that uses it, wire the two together and mention it in passing. Now two
things changed in a turn you wanted to be able to undo, and the report reads as success.

**"Do not publish."** Publishing is the irreversible part of the loop; everything before it can be
thrown away. Ending with a report keeps the decision with a person.

**"Anything you changed that I did not ask for."** This is the clause that catches the real problem,
below.

## Structure is exact; content gets rewritten

The most useful distinction we found, and the reason the report matters.

**Structure comes back exactly as specified.** Verified against the published output, not against
Mentor's own account of it.

**Content comes back rewritten.** Two measured cases, both *after* an explicit instruction to copy
literally: an escape sequence inside a CSS comment changed character, and a column-configuration
JSON came back with labels changed, columns reordered and keys translated from Portuguese to
English. The JSON was still valid, so nothing failed loudly — the badge colours just silently
stopped matching the data.

Large content does not merely change, it fails: a ~106 KB stylesheet killed a turn with
`OS-AIWA-50001`. Pasting the same file into ODC Studio is instant and costs **no quota**.

So: **model through Mentor, payload by hand** — and any value that has to match data is read from
the published app, never from the turn's own report.

## When a turn fails, resume it — never start a new one

The instinct after a failed or cancelled turn is to start over. Don't.

A failed turn still returns its session id and a new session token, and **resuming continues from
where it stopped**. Starting a new session spends another session slot on the environment *and*
discards edits that were never published.

When the conversation itself is the problem — it hit the size limit (`OS-AISA-40001`), Mentor has
started inventing entities and actions that do not exist, or you are switching to a different task
in the same app — resume with **fresh context** rather than a new session. That resets only the
conversation and keeps the session, the server-side model and any unpublished edits.

And do not poll a running turn more often than about every 30 seconds: each status check costs a
full model turn. It is consumption that does not look like consumption.

## Publish each milestone before asking for the next

Until you publish, everything lives only in the server-side session. If it expires or is collected,
unpublished work is gone and the next turn starts from zero.

The order that worked: publish the **data model**, then the **backend**, then screens **one at a
time**, publishing between each. A failure in the middle then costs one turn — not the batch, where
you usually cannot even tell how much of it landed.

## When this is overkill

A one-line change, a rename, anything where you already know exactly what will come out: just ask.
The shape is for work that is large, exact, or metered — and on a shared ODC environment the quota
you spend is the team's, so it is metered even when it is not billed to you.
