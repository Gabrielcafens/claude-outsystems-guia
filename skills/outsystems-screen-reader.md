---
name: outsystems-screen-reader
description: Control exactly what a screen reader announces on an OutSystems screen, in ODC or O11, when you do not own the markup — only CSS and a script that runs after the widget paints. Use this whenever the work involves NVDA, JAWS, VoiceOver, TalkBack, aria-label, aria-hidden, role, tabindex, a visually hidden label, WCAG, or a finding from an accessibility audit. Use it especially when aria-label appears to be ignored, when the reader announces only part of what is on screen, when it keeps reading a long list nobody wants to hear, when a card is clickable only on one small element inside it, or when someone is looking for an API that stops the reader mid-sentence. The worked case is a FullCalendar day cell, but the rules hold for any third-party widget rendered inside a Block.
---

# Controlling what a screen reader says in OutSystems

Most accessibility writing assumes you own the markup. Inside OutSystems you usually do not: the
element belongs to a platform widget or to a library rendered inside a Block, and your whole
toolkit is the theme stylesheet plus a script that runs after the thing paints.

The case this came from: a calendar day cell that announced `7` when it should have announced the
full date and how many slots were free — and that then read all ninety slots out loud.

## Decide the sentence before touching the DOM

Write down, in words, the one sentence you want to hear, and the list of things you want silence
for. Two separate problems hide behind "the reader is wrong", and they have opposite fixes:

- **not enough is announced** → add text to the accessibility tree
- **too much is announced** → remove nodes from the accessibility tree

Mixing them is how you end up with a cell that says the right date and then keeps talking. Fix the
first, then the second, and listen between the two.

## The lever is text, not `aria-label`

This is the one that costs people a day. **`aria-label` is ignored on an element the browser gives
no role to.** The attribute sits there in DevTools, the audit panel may even show it, and the
reader says nothing.

Elements that silently drop a label:

- `<a>` with no `href` — it is not a link, it is generic
- `<div>` and `<span>` — generic by default
- any `td` or `tr` inside a table carrying `role="presentation"`, because presentation strips the
  implicit roles of the rows and cells too

Layout tables and anchors-used-as-buttons are exactly what widget libraries emit, so this is the
normal case, not the exotic one.

**Text content, on the other hand, is always announced** — it does not depend on a role. That
asymmetry is the whole technique: when a label will not stick, put real text inside the element and
hide it with CSS.

```js
const sr = document.createElement('span');
sr.textContent = 'Tuesday, 20 October 2026, 3 appointments available';
el.appendChild(sr);
```

Keep the `aria-label` as well when the element does have a role — it costs nothing and wins where
it applies. Just never rely on it alone.

## Hide the text from eyes, not from the reader

`display:none` and `visibility:hidden` remove the node from the accessibility tree. The text has to
stay rendered and merely be impossible to see:

```css
position: absolute;
width: 1px;
height: 1px;
margin: -1px;
padding: 0;
overflow: hidden;
clip-path: inset(50%);
white-space: nowrap;
border: 0;
```

**In OutSystems, inline this instead of relying on a class.** A visually-hidden utility class may
not exist in the theme, and may not survive the next publish of it — and the failure mode is loud:
the hidden sentence renders as visible text in the middle of the screen. The script already runs at
the right moment, so let it carry the style itself:

```js
sr.setAttribute('style', [
  'position:absolute !important', 'width:1px !important', 'height:1px !important',
  'margin:-1px !important', 'padding:0 !important', 'overflow:hidden !important',
  'clip-path:inset(50%) !important', 'white-space:nowrap !important', 'border:0 !important'
].join(';'));
```

The `!important` is not superstition here: theme rules are written against the widget's own
classes, and a single `position: static` from the library is enough to put the span back in the
flow, in view, mid-screen.

## Silencing what must not be read

To take a subtree out of the announcement, hide it **and** make its focusables unreachable.
`aria-hidden="true"` on something that can still receive focus is a conformance violation and
behaves unpredictably — the reader lands on a node it has been told does not exist:

```js
const inner = cell.querySelector('.fc-daygrid-day-events');
if (inner) {
  inner.setAttribute('aria-hidden', 'true');
  inner.setAttribute('role', 'presentation');
  inner.querySelectorAll('a, button, input, select, textarea, [tabindex]')
       .forEach(node => node.setAttribute('tabindex', '-1'));
}
```

And the ordering trap: hide the interior **before** adding your own text and the element goes
completely silent. The sentence has to live outside the hidden subtree — which is why the span goes
on the element you kept, not inside the part you removed.

## There is no API that stops the reader

Nothing in the platform, in ARIA, or in the DOM truncates speech that is already queued.

- a **live region** *queues* — it adds to what will be said, it never cuts it short
- **moving focus** is the only interrupt, and it interrupts by starting something else

So "make it stop reading after the count" is not something you implement. It is a sign that the
content you do not want is still in the accessibility tree, and the fix is to take it out. Hunting
for an interrupt instead is a day lost.

## The text being read may not be in the element you clicked

When the reader announces something you cannot find, walk upward. A container above the element
often carries text of its own, and hiding the child changes nothing:

```js
let node = document.activeElement;
while (node && node !== document.body) {
  console.log(node.className || node.tagName,
              '| aria-hidden:', node.getAttribute('aria-hidden'),
              '| text:', (node.textContent || '').trim().slice(0, 60));
  node = node.parentElement;
}
```

Run that from the console with the element focused. In the worked case the slot list was not inside
the cell at all — it was a separate panel elsewhere on the screen, which is why every change made
inside the cell did nothing.

## Making the whole card clickable, without double-firing

Two different problems get confused here.

**Clicks are being swallowed.** Libraries stack an absolutely positioned background layer over the
cell for highlighting. It has `inset: 0` and a `z-index`, so it eats every click that does not land
on the one element drawn above it:

```css
.fc-daygrid-day-bg,
.fc-daygrid-bg-harness { pointer-events: none; }
```

**The library may already handle the whole cell.** FullCalendar's `dateClick` fires for the entire
day cell, not just the number. A delegated handler of your own on top of it means the Block action
runs twice per click. Before writing a listener, read the widget's own options — the behaviour you
are about to build may already be configured.

For the same reason, `event.stopPropagation()` inside the cell tends to break the library rather
than help it: its handler is bound above the widget root, so stopping propagation stops the feature.

## What stretching an element over its parent costs you

The clean way to let one inner element own the whole card is to stretch it:

```css
.fc-daygrid-day-frame  { position: relative !important; height: 100%; min-height: 3.5em; }
.fc-daygrid-day-top    { position: static !important; min-height: 1.75em; }
.fc-daygrid-day-number {
  position: absolute !important; top: 0; left: 0;
  width: 100% !important; height: 100% !important; z-index: 5;
  display: flex !important; justify-content: flex-end; align-items: flex-start;
  padding: 6px 8px; box-sizing: border-box; cursor: pointer;
  text-decoration: none !important;
}
```

Both `min-height` values are load-bearing. Once the number leaves the flow, the row that held it
has no content left, and the cells at the edges of the grid — the ones with no events — collapse to
nothing. The symptom reads as "the first and last days came out the wrong size", which does not
sound like a positioning bug.

Two more that look unrelated until they bite:

- **`border-radius` is discarded at paint time inside `border-collapse: collapse`.** Calendar grids
  are collapsed tables. Put the radius on an inner `div`, never on the `td`.
- **`toLocaleDateString` throws `RangeError` on an invalid `<html lang>`.** OutSystems writes that
  attribute from the app's locale, and one malformed value kills the rest of your function at the
  exact line where you were building the sentence. Wrap the call, or build the string by hand from
  an array of month names.

## Where this lives in an OutSystems app

- **the CSS** → the theme stylesheet, or the Block's own style sheet
- **the script** → a Script element referenced at Layout level, or the Block's `OnReady` JS node

The widget re-renders — every month change, every filter — and takes your work with it. Make the
function idempotent, mark what you touched, and re-run on mutation:

```js
function enhance() {
  document.querySelectorAll('.fc-daygrid-day:not([data-sr-done])').forEach(cell => {
    cell.setAttribute('data-sr-done', '1');
    // add the sentence, then silence the interior
  });
}
enhance();
new MutationObserver(enhance).observe(document.body, { childList: true, subtree: true });
```

The `:not([data-sr-done])` is what keeps the observer from fighting itself: without it, the span you
append is a mutation, which wakes the observer, which appends another span.

## Verify with the reader, not with the panel

DevTools' accessibility tree gives you the computed name of one node. It does not tell you the
**order** things are announced in, or where the reader carries on after the part you fixed — and
order is the whole complaint. Turn on a real reader and listen to the sentence end to end. NVDA is
free on Windows; VoiceOver ships with macOS and iOS.

One habit worth keeping: read the element, then press the key that moves to the next one. If what
you silenced turns up there, it is still in the tree and you hid the wrong node.

## Honest limits

The announcement behaviour described here was verified by listening, with a reader running, in
Chrome on Windows. The `aria-hidden` plus `tabindex="-1"` pairing is the spec requirement and was
applied for conformance, not measured against an audit tool. Nothing here was tried with JAWS or
TalkBack.
