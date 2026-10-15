---
name: outsystems-block-component
description: Build a real front-end component for OutSystems in TypeScript, versioned in Git, driven by a thin Block — the Block-as-bridge technique. Use this whenever the work involves an OutSystems Block with JavaScript, a JS node, $parameters or $actions, a custom widget the OutSystems UI library does not have, high-code inside OutSystems, or wiring TypeScript into Service Studio or ODC Studio — even when the person does not name the technique and only says something like "I need a data table in OutSystems that filters and sorts". Also use it when a Block's events are not firing, when a component publishes cleanly but does not render, or when checking what the platform actually published.
---

# TypeScript components inside OutSystems Blocks

## What this is for

OutSystems does not support Web Components natively. The way to get a real component — one with
behaviour, types, tests and a Git history — is to let the **Block be a thin bridge** and keep the
component outside the platform, in a JavaScript bundle loaded at Layout level.

The Block is the tip of the iceberg: what you see in Studio. The component is the mass below the
waterline, and it evolves as ordinary high-code. Technique credited to António Carvalho,
*AI-Powered Frontend Development in OutSystems O11 and ODC*.

**Use a different approach when there is no behaviour to build.** If the ask is only a different
*look* on a native widget — colours, radii, spacing — that is CSS in the theme, and this whole
architecture is overhead. Check that first; it is the most common way this technique gets misused.

## The bridge

The Block needs exactly five things:

| Element | Why it exists |
|---|---|
| Input Parameters | the data and configuration the component reads |
| An empty Container | the mount point; its runtime id is the handshake |
| A Local Variable of type Object | holds the instance between lifecycle handlers |
| Events | how the component talks back to OutSystems |
| A Script dependency | the bundle, at Layout level or in the Block's Required Scripts |

The class on the other side needs three members: a constructor taking a config object with the
mount id plus the parameters, an `onParametersChanged` method, and a `destroy` that removes every
listener and observer. Publish it on your own namespace — `window.MyApp.MyComponent`.

In O11 Traditional the only difference is initialisation: an Expression that runs as the Block
renders, instead of an `OnReady` handler.

## The three JS nodes

```js
// OnReady
$parameters.Instance = new MyApp.MyComponent({
  runtimeId: $parameters.RuntimeId,
  dataJson: $parameters.DataJson,
  events: {
    OnRowClick: $actions.TriggerRowClick
  }
});

// OnParametersChanged
if ($parameters.Instance) {
  $parameters.Instance.onParametersChanged({ dataJson: $parameters.DataJson });
}

// OnDestroy
if ($parameters.Instance) {
  $parameters.Instance.destroy();
}
```

Keep the `if ($parameters.Instance)` guard in both handlers. `OnParametersChanged` can fire before
`OnReady` has finished, and without the guard you get a null reference at runtime that is painful
to trace back to a lifecycle ordering problem.

## `$actions` cannot see Events

This is the single most common reason the technique "mysteriously does not fire anything", and it
is not in the original article.

```js
events: { OnGreetClicked: $actions.OnGreetClicked }   // fails
```

The compiler rejects it with `Invalid JavaScript — Unknown 'OnGreetClicked' action`. Inside a JS
node, `$actions` resolves **Client Actions only**, never a Block Event.

Bridge it with two extra elements per event:

1. a public Client Action on the Block, e.g. `TriggerRowClick(RowId)`
2. whose only flow step raises the corresponding Event
3. the JS node passes `$actions.TriggerRowClick` into the component
4. the component calls it like any callback: `this.events.OnRowClick(id)`

Budget one Client Action per event from the start, so this never becomes a debugging session.

## Repository layout that works

```
src/index.ts                  registers window.MyApp.MyComponent
src/my-component/types.ts     the contract with the Block: parameters, events
src/my-component/Component.ts the class
src/my-component/styles.ts    component layout, injected once into <head>
build.mjs                     esbuild -> dist/MyComponent.js (IIFE, ES2019, not minified)
preview/index.html            test page loading the real OutSystems UI CSS
```

Keep `types.ts` as the contract: when the Block and the class disagree, that file arbitrates.

Only `dist/MyComponent.js` goes to OutSystems. Edit `src/`, never `dist/`.

## Develop against the real cascade, without publishing

Build a preview page that fetches the OutSystems UI stylesheet **from the app you are targeting**,
plus the third-party libraries its patterns depend on. Then you are developing inside the cascade
you will ship into, instead of discovering conflicts after a publish.

Add a query flag that runs a scripted click-through and writes PASS/FAIL into the page. That
harness is what makes AI-written code safe to accept: you are watching assertions pass, not reading
a diff and hoping.

## Working through the platform agent

When an agent (Mentor in ODC, the Service Studio MCP in O11) is creating these elements for you:

**One element per turn.** Script and Block in the same turn blew the ODC quota with
`OS-AISA-42903` and published nothing; split, both passed. In ODC the quota is counted per tenant
per day, so on a shared tenant you are spending a team resource.

**Model through the agent, content by hand.** The agent builds structure exactly — Blocks,
parameters, events, handlers, screens. It rewrites *content*: a `\xE7` inside a CSS comment came
back as `\xE3`, and a configuration JSON came back with keys translated, which silently broke a
value map. Large content does not merely change, it fails — a ~106 KB stylesheet killed a turn with
`OS-AIWA-50001`. Paste bundles and stylesheets directly in Studio: instant, and it costs no quota.

**Write the prompt somewhere cheaper.** Quota is spent per turn, not per unit of thinking. Draft
the plan in a session that is not connected to the platform, then paste one fully formed request
into the connected one.

## Verify what was published

"Published with 0 errors" is static analysis of the model. It does not mean the component renders,
the JSON parses, or the events fire — a far-too-simple `render()` publishes perfectly cleanly.

Every app lists the versioned files it loads at `https://<app-url>/moduleservices/moduleinfo`.
Download and compare against disk, normalising two platform behaviours first:

| Asset | What the platform does | How to compare |
|---|---|---|
| Theme CSS | minifies: `0.625rem` → `.625rem`, `::before` → `:before`, colours to hex, equal rules merged | ignore those rewrites |
| Script | does not minify: reindents, CRLF → LF, adds a BOM | ignore whitespace |

Two facts that shape what is even checkable:

- **A Block no screen uses is not in the bundle.** Plan a throwaway screen before trying to audit
  the generated JS nodes.
- **Any value that must match data** — JSON, keys, ids — is read from the published app, never from
  the turn's own report.

Then open the screen with DevTools: Elements shows whether the component took over the container,
Console shows what it complained about.

## Checklist

- One bridge Client Action per Event, created up front
- `if (Instance)` guard in `OnParametersChanged` and `OnDestroy`
- A `destroy()` that really removes every listener and observer
- Platform data into the DOM through `textContent`, never `innerHTML` — a value like
  `<img onerror=...>` should render as visible text
- One agent turn per element; bundles and stylesheets pasted by hand
- A screen that uses the Block, so it is auditable
- After publishing: download from `moduleinfo`, compare ignoring whitespace, then open DevTools
- Any data-matching value re-read from the published app
