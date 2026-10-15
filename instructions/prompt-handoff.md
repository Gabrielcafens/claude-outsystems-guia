# Prompt handoff: think in the cheap session, act in the expensive one

Use two AI sessions instead of one.

The **planning session** runs on a free or low-cost model. You talk to it the way you actually
think: half-formed, changing your mind, pasting things in, being corrected. It does not touch your
codebase, your environment or anything that costs money to run. Its only output is a **finished
prompt**.

The **execution session** runs on the paid, metered or rate-limited agent — the one wired to your
repository, your platform, your tenant. It receives that finished prompt, once.

## Why this saves so much

**You are charged per turn, not per unit of thinking.** The expensive agent costs the same whether
your request is precise or vague — but a vague request costs several turns, because it guesses, you
correct, it guesses again. The thinking is not the expensive part. The *round trips* are.

So move every round trip to where they are free. Exploration, dead ends, rewrites, arguments about
the approach, "actually no, do it the other way" — all of that happens in the planning session. The
expensive turn is paid exactly once, with a fully formed request.

The secondary saving is just as real: a half-specified request is the worst failure mode there is.
The agent half-guesses, produces something plausible, and you now pay again to undo it — sometimes
without noticing you need to.

## What each session is for

| | Planning session | Execution session |
|---|---|---|
| Model | free or cheap tier | the strongest tier you have |
| Connected to | nothing | repository, platform, environment |
| You give it | context, constraints, your half-formed idea | one finished prompt |
| It gives you | a prompt, a plan, a checklist | the actual change |
| Number of turns | as many as you like | as few as possible |

## What the planning session must hand over

A plan is not "here's roughly what to do". Ask for all of it:

1. **The units of work, named**, with their types, parameters and anything else that must be exact.
2. **The order** to execute them in, one unit per turn.
3. **What goes by hand instead of through the agent** — large files, long configuration, anything
   the agent would retype and therefore rewrite.
4. **The exact prompt for each turn**, ready to paste, ending with a request to report back rather
   than to commit, publish or deploy.
5. **What to check after each turn** before moving to the next.
6. **Its own assumptions, marked as assumptions** — this is how you catch a plan built on a file or
   an element that does not exist.

Point 6 is the one people leave out, and it is the cheapest insurance on the list.

## How to ask for it

Open the planning session with the handoff stated explicitly, or it will try to do the work itself:

```
I am going to run this through <the paid agent> in a separate session.
Do not try to do it here.

What I want: <the outcome, in plain language>
What already exists: <so it reuses rather than recreates>
Constraints: <the things that are not negotiable>

Turn this into a plan I can execute one turn at a time, with the exact prompt
for each turn. Mark anything you are assuming. Ask me whatever would change the
plan before you write it.
```

## What must not go into a free session

A free tier is free for a reason, and on many providers the reason is that your input is retained,
reviewed or used for training. Check the provider's terms before this becomes a habit, and in the
meantime work on the **shape** of the problem rather than its contents:

- describe the structure, do not paste customer code
- rename real identifiers to placeholders — `<ClientName>`, `<app-url>`, `<table>`
- never paste credentials, tokens, keys or connection strings, in any session
- never paste personal data, production records or anything under an NDA

A plan built from a described shape is just as good. The executing session is the one that has the
real context, and that is where the real context should stay.

## Failure modes, and what they look like

| What happens | Why | Fix |
|---|---|---|
| The plan references something that does not exist | the planner has no access, so it fills gaps plausibly | require assumptions to be marked; verify the named things before pasting |
| The expensive agent does more than you asked | helpfulness — you asked for one thing and it built the thing around it too | end every prompt with "do nothing that is not listed above" |
| The plan is right but the execution drifts | too much in one turn | one unit per turn, verify between each |
| You fix the error in the expensive session | that is a round trip at full price | go back to the planning session with the error message and get a corrected prompt |

That last row is where most of the saving is lost. When an expensive turn fails, the instinct is to
reply to it. Don't — take the failure back to the free session, work it out there, and return with
one corrected prompt.

## When not to bother

The handoff has a cost of its own: you explain the context twice, and the planner can only reason
about what you told it. For a one-line change, a rename, or anything where you already know exactly
what to type, skip it and just ask.

Use it when the work is **large, exact, or metered** — a multi-step build, something that must match
an existing structure precisely, or any agent with a quota, a bill or a rate limit attached.

## Measured

From an OutSystems lab, where the platform agent has a hard daily quota per tenant:

- Three quota exhaustions in three days, every one on a turn that asked for several things at once.
  After splitting to one unit per turn, none.
- A turn run on a mid-tier model with low reasoning effort reported success with an internal retry
  count of **14**. The retries are invisible in the result and very visible in the bill. A cheaper
  model is not cheaper if it retries fourteen times — which is exactly why the cheap model belongs
  in the planning seat and not in the execution seat.
