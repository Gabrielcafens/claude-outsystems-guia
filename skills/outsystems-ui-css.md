---
name: outsystems-ui-css
description: Restyle native OutSystems widgets with CSS only, in ODC or O11 — no Block, no JavaScript. Use this whenever the work touches an OutSystems theme stylesheet, a widget's Style Classes property, the platform tokens (--color-*, --border-radius-*, --font-size-*), or making a Checkbox, Dropdown, Tabs, Table, Button or Popup look different from the platform default. Use it especially when someone says a CSS rule "is being ignored" in OutSystems, when DevTools shows a rule winning but nothing paints, or when a Figma screen has to become CSS. Reach for this before building a Block: if only appearance changes, high-code is overhead.
---

# Writing CSS against OutSystems UI

## Decide this first: CSS or a Block

OutSystems widgets render real HTML. A Button is a real `<button>`, a Table a real `<table>`, a
Checkbox an `<input type="checkbox">` with a styled pseudo-element. None of that needs replacing —
it needs different CSS.

So before any architecture: **is there behaviour to build, or only appearance to change?**

| The ask | The answer |
|---|---|
| Different colours, radii, spacing, focus rings, shadows on a native widget | CSS in the theme. Stop here |
| A control the platform does not have — combobox, command palette, context menu, chart, input OTP, resizable panes | a Block bridging to external code |

Getting this wrong in the expensive direction is common: a component in TypeScript, a Block, a
build pipeline and a Script dependency, to round a corner. If you end up needing the Block, see the
`outsystems-block-component` skill.

## Two groups, very different blast radius

When you write a stylesheet for a theme, separate it into two files, because the risk is not the
same:

| Group | What it is | Effect on the app |
|---|---|---|
| **By class** | your own tokens plus your own `my-*` classes | **none** until someone puts the class on a widget |
| **Platform reskin** | overrides targeting OutSystems UI's own selectors | **global**: every Checkbox, Dropdown, Tabs and Popup in the app changes at once |

Ship the by-class file first — it cannot break anything that already exists. The reskin file is the
one that needs a round of regression looking at real screens.

Where it goes: Studio → **Interface → Themes** → the theme's stylesheet (same path in ODC Studio and
Service Studio). To reuse across apps, put the CSS in a Library's theme and consume that theme. To
apply a by-class rule, the class goes in the widget's **Style Classes** property.

## When the computed style lies

This is the trap worth knowing before you need it, because every debugging instinct points the
wrong way.

A checkbox inside a Table widget, asked for a 4px corner. The corner never appeared — and:

- DevTools showed the rule winning, not struck through
- `getComputedStyle` returned `border-radius: 4px`
- other properties in the **same rule** worked — `border-color: red` painted immediately, corner
  still square
- `!important` changed nothing

The cause: the visible box of an OutSystems checkbox is not the `<input>`, it is its `::before` —
and the platform ships that `::before` with `display: table`. When an element is a table box and the
`border-collapse` in effect is `collapse` (which it is inside OutSystems UI tables), the spec tells
the browser to **ignore** `border-radius`. Defined behaviour, not a bug. The property is discarded
at paint time, after it has already been computed, which is why no specificity fight helps.

```css
.table [data-checkbox]:before {
    border-radius: 4px;
    box-sizing: content-box;
    display: block;
}
```

`display: block` takes the element out of table layout (`border-collapse: separate` on the
`::before` works for the same reason). `box-sizing: content-box` comes along because, leaving
`display: table`, the declared size starts including borders and the box shrinks by 2px.

**The general rule: when a property shows as applied in the computed style but does not paint, stop
fighting the cascade and look at `display`.** The element's layout model is throwing it away.

## Read the platform CSS before writing overrides

Half of what we were about to "create" for the checkbox already existed. The platform's stylesheet
is sectioned and commented — reading the relevant section first is the cheapest step available.

| State | Already in the platform? |
|---|---|
| Normal, checked | yes |
| Hover | yes, with a neutral border |
| Disabled, disabled checked | yes, including the grey check mark |
| Focus | **no** — worse, it removes the `outline` and puts nothing back |
| Error | **no** |

So the override actually worth writing was the one nobody asked for: keyboard users could not see
where they were.

```css
[data-checkbox]:focus-visible:before {
    border-color: var(--color-primary);
}
```

Reading it also saves code: `[data-checkbox][disabled]` already sets `pointer-events: none`, so
hover rules need no `:not(:disabled)`; and inside a table the box is smaller than outside it, with
its own check-mark position per size — change one without the other and the mark goes off-centre.

## Replace the platform class, do not stack it

On widgets that already carry an OutSystems UI class — `btn`, `form-control`, `table` — **swap** the
class instead of adding yours beside it.

With both present your rules usually win in the default state, so it looks like it worked. Then the
platform rules come back on `:hover`, `:focus` and `:disabled`, and you get a widget that is yours
at rest and the platform's the moment anyone touches it.

While you are there: look at the base class before writing a variant. `.btn` is **already** an
outline button — `neutral-0` background, `border: solid currentColor`. Because the border follows
`color`, an error-outline button is one declaration, `color: var(--text-color-error, var(--color-error))`.
Copying four more lines from the filled variant only duplicates what is inherited.

## `var()` without a fallback invalidates the whole declaration

If the token does not exist, `var(--token)` does not "do nothing": the declaration becomes invalid
and the property takes its initial value. `border-color: var(--does-not-exist)` resolves to
`currentColor` — the text colour. In theme CSS this is treacherous because nothing appears in the
console.

Check before relying on a token:

```js
getComputedStyle(document.body).getPropertyValue('--border-radius-soft')
```

If that comes back empty, every component depending on that token is unstyled — not just the one in
front of you.

## Sticky only moves inside its parent

Two traps, in this order.

**A sticky element only moves within its parent.** Put it on a container exactly as tall as its
parent and it sticks, then leaves the screen with it — looking precisely as if the rule were
ignored. The class belongs on the element whose parent is the scrolling area.

**`top` is almost never 0.** With a fixed header in the layout, `top: 0` parks the block behind it.
Measure: `document.querySelector('header.header').getBoundingClientRect().height`.

```css
.sticky-content-header {
    background-color: var(--color-neutral-0);
    box-shadow: 0 1px 0 var(--color-neutral-3);
    position: sticky;
    top: 0;    /* the fixed header's height, when there is one */
    z-index: 5;
}
```

The background is not decoration — without it the content scrolls behind and straight through.

## Coming from Figma: write from tokens, not from pixels

Figma hands you hex and px. The theme has a name for almost all of it, and using the name is what
makes the component follow a rebrand instead of needing a second pass.

| Figma | Token |
|---|---|
| `#CED4DA` | `--color-neutral-5` |
| `border-radius: 4px` | `--border-radius-soft` |
| `border: 1px` | `--border-size-s` |
| `font-size: 14px` | `--font-size-s` |

To find the nearest colour token to a hex, compute Euclidean distance across the RGB channels rather
than arguing about which "looks the same": for `#F5F8FA`, `#F3F6FA` sits at 2.8 and `#F8FAFC` at
4.1 — both indistinguishable by eye, so the choice becomes semantic, which surface scale it belongs
to.

Two things Figma exports that should never reach your CSS: `font-feature-settings: 'liga' off`,
which is export noise, and `font-family` inside a component class, which belongs to the theme.

And one real gap to raise with design rather than paper over: **the OutSystems weight scale is 300,
400, 600, 700 — there is no 500.** If the design asks for 500, either it is a hard-coded number or
it is a decision to add the token to the theme. Say which.

## When an agent writes the CSS

**Never ask for a design system's CSS from memory.** Our first attempt produced perfectly plausible,
perfectly outdated CSS: the old shadcn format, the `zinc` preset, colours in HSL — while current
shadcn is Tailwind v4, `neutral`, and `oklch()`. Different values entirely, and no error anywhere.

Point the agent at the source files at a **pinned commit** and have it convert from those. Record
the commit next to the stylesheet, so the next person knows which version it reflects. This
generalises past shadcn: for any fast-moving design system, the agent must read the version you are
on, not recall the version it was trained on.

**Tailwind output is not CSS.** `transition: colors 0.15s ease` appears all over Tailwind-derived
snippets; `colors` is not a property. It fails silently — the declaration is dropped and the
transition simply never happens. Review translated utility classes for this shape.

**Big stylesheets go in by hand.** A ~106 KB stylesheet sent through the ODC agent killed the turn
with `OS-AIWA-50001`. Pasting into Studio is instantaneous and costs no agent quota. See
`outsystems-agent-session`.

## Verify the published stylesheet

The platform minifies theme CSS on publish, so a diff against your source is noisy by default.
Expected and harmless:

- `0.625rem` → `.625rem`
- `::before` → `:before`
- named and `rgb()` colours → hex
- identical rules merged

What survives untouched is more interesting: `oklch()`, `color-mix()` and `:is()` all came through,
because the minifier does not understand them and leaves them alone.

The published file is listed at `https://<app-url>/moduleservices/moduleinfo` — download it from
there and compare, rather than trusting a publish report.

## Develop against the real cascade, without publishing

Publishing to compare pixels is slow. Build a local preview page that loads **the real OutSystems UI
stylesheet from the app you are targeting**, plus the real third-party libraries its patterns depend
on (Flatpickr, noUiSlider, VirtualSelect, Splide). Then the cascade you develop against is the
cascade you ship into, and conflicts show up before a publish instead of after one.

One cosmetic caveat: OutSystems icons render as empty squares outside the app, because the icon font
is not loaded. Not a bug in your CSS.

## Checklist

- Is this appearance only? Then no Block, no JavaScript
- By-class file and platform-reskin file kept separate; reskin reviewed on real screens
- Platform class swapped, not stacked — and `:hover`, `:focus`, `:disabled` checked, not just
  the default state
- Platform CSS for that widget read before writing the override
- Every `var()` has a fallback, or the token was verified to exist
- Rule applied but not painting → look at `display`, not at specificity
- Figma values mapped to tokens; weight 500 flagged rather than hard-coded
- Agent-written CSS came from a pinned source, not from memory
- Published file downloaded from `moduleinfo` and compared, ignoring the minifier's rewrites
