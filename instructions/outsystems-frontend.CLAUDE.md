# Working agreement — AI-assisted front-end on OutSystems

Rules that apply to every turn in this repository. Each carries its reason, because a rule without
one gets dropped the first time it is inconvenient.

## Working through the platform agent

- **Write the prompt in a session that is not connected to the platform**, then paste it into the
  connected one. Quota is spent per turn, not per unit of thinking.
- **One element, or one screen, per turn.** Publish between each. A Script and a Block in the same
  turn blew the quota (`OS-AISA-42903`) and published nothing.
- **Run heavy turns on the strongest model tier available.** A mid-tier model with low reasoning
  effort finished "clean" with an `internal_retry_count` of 14.
- **Model through the agent, content by hand.** The agent builds structure exactly and rewrites
  content silently. Paste stylesheets and bundles directly in Studio — instant, and no quota.
  A ~106 KB stylesheet killed a turn with `OS-AIWA-50001`.
- **After a failed turn, resume the session.** Starting a new one spends another session slot and
  discards unpublished edits.
- **Do not poll a running turn** more often than about every 30 seconds. Each check costs a full
  model turn.

## Verifying

- **"Published with 0 errors" is not a test.** It is static analysis of the model.
- **Verify against the published artifact:** `https://<app-url>/moduleservices/moduleinfo` lists
  the versioned files the app loads. Compare against disk, ignoring the platform's rewrites — it
  minifies theme CSS, and reindents Scripts with LF and a BOM.
- **Any value that must match data** — JSON, keys, ids — is read from the published app, never from
  the turn's own report.
- **A Block no screen uses is not in the bundle.** Plan a throwaway screen before auditing JS nodes.

## Safety

- **Confirm before publishing.** The agent reports what it is about to do; a human approves.
- **Never point an agent at a customer environment** for experiments.
- **Secrets never enter a versioned file**, not even inside a `.md`. Name the setting, never the
  value.
- **Never start a second publish of the same app while the first is running.**

## Front-end specifics

- **Appearance only means CSS in the theme, not a Block.**
- **Edit `src/`, never `dist/`.**
- **Swap the platform class, do not stack beside it** — and check `:hover`, `:focus` and
  `:disabled`, not just the default state.
- **Platform data into the DOM through `textContent`, never `innerHTML`.**
- **Branch → PR → merge**, even for a small change.
