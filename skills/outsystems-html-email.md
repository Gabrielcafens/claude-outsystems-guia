---
name: outsystems-html-email
description: Build and send HTML email from an OutSystems app — ODC or O11. Use this whenever the work involves an Email element, the Send Email action, an email body that has to carry real HTML or a design, CSS inlining, Foundation for Emails or any email framework, or an SMTP configuration in the Portal. Use it especially when HTML arrives in the inbox as visible text with the tags showing, when someone is looking for an Escape Content property in ODC, when the email layout breaks only in Outlook, when a responsive email does not stack on a phone, or when the send fails saying the SMTP server must be set. Read it before anyone spends a day trying to paste a finished HTML document into an email body.
---

# HTML email in OutSystems

Email is the one place where the platform's normal front-end habits stop working: no cascade you
control, no modern CSS, and a renderer in Outlook desktop that predates most of it. The expensive
mistake is choosing a route before knowing which routes exist.

## Choose the route first

| What is being sent | Route | Cost |
|---|---|---|
| Transactional — confirmation, notification, password reset, simple layout | **body built from widgets** | an hour, no moving parts |
| A real responsive design, or HTML produced outside the platform | **finished HTML through `Body`** | an inlining pipeline and a literal to manage |
| Same, but the app must not hold SMTP credentials or you need full control of the transport | **.NET External Library over SMTP** | a release cycle per code change |

Most asks are the first row. Check that before building anything.

## The blocker: in ODC an Expression always escapes

The obvious plan — one Expression in the body holding the whole HTML, with `Escape Content = No` —
does not work, and the reason is hard to discover from the outside:

**`Escape Content` does not exist in ODC.** It is a Traditional Web property from O11. In Reactive
and in ODC an Expression escapes its content *always*, with no override. Concatenated HTML arrives
in the inbox as visible text, tags and all.

There is no unescaped Expression anywhere in ODC. So inside the email editor, a body is built from
widgets — Containers, Text, Links — which render as `div` and `a` with whatever classes you give
them. If the requirement is finished HTML, it has to enter somewhere other than an Expression.

## The Email element renders; Send Email sends

This is the distinction that opens the second route, and it is easy to miss: in ODC the Email
element does **not** send anything, it renders. Sending is a separate action, and its `Body`
property is of type **Binary Data**, not HTML.

`EmailBodyOf(<Email>)` is only *one* way to produce that binary. Another is:

```
TextToBinaryData(MyFullHtml)
```

from the `BinaryData` extension — a Text input plus an optional `Encoding` that defaults to UTF-8.
Because the HTML never passes through an Expression, nothing is escaped.

**Status, stated precisely:** the action exists with that signature and was confirmed present in the
tenant, but an end-to-end send through it has not been tested. Verify it before designing around it
— the one-minute test is to swap a working `Body` for `TextToBinaryData("<p><b>test</b></p>")` and
see whether the word arrives bold. Do that test first; if it passes, it saves you an external
library, a release cycle and SMTP credentials living inside the app.

## Why email frameworks die in the editor

Foundation for Emails selects its responsive grid on `table.row` and `th.columns`, and one of its
buttons is three nested tables. The ODC email editor produces Containers, which render as `div`. The
selectors never match, so the stylesheet is **inert** — this is not a matter of pasting the CSS in
the right place. The same reasoning applies to any framework whose grid is table-based, which is all
of the serious ones.

And the tables are not legacy fussiness. **Outlook for desktop renders email with the Word engine**,
which ignores `max-width` and `float` on a `div`. Tables are what survives there. Anyone building an
email body out of `div` elements is choosing, usually without realising it, not to support Outlook
desktop. Say that out loud when agreeing the design, rather than discovering it in a stakeholder's
inbox.

## The external library route, if you take it

A small .NET library that takes host, port, user, password, sender, recipient, subject and HTML and
sends over SMTP with STARTTLS works, and it delivered a full ~25 KB document to a capture server.
Two platform facts to know before committing:

- **A published library is not referenceable until a version is released.** The reference is refused
  with *"version tag is required"*, and releasing is a Portal action, not an API one.
- **Released versions are immutable.** Every code change is a new version, and the consuming app's
  reference has to be moved to it.

The cost is not writing the C#. It is that release cycle, on every fix.

### This signature is an open relay if you expose it

An action accepting host, port, user, password, recipient and body will send mail from any server to
anyone. Inside a Server Action it is a utility; behind an exposed REST endpoint it is an open relay
with your infrastructure's name on it.

- credentials in settings marked as **secret**, never in code, never in a `.md`
- never let them reach a Client Action, a log or a trace
- before publishing, check that no leftover test endpoint exposes the action — test harnesses have a
  way of getting published and forgotten

## Inlining: CSS has to become `style` attributes

Email clients drop `<style>` blocks often enough that the CSS has to be inlined. That is a tool's
job, and the tool is [juice](https://www.npmjs.com/package/juice).

One catch cost real time: with `inlineContent`, juice **discards every `@media` block — even with
`preserveMediaQueries: true`.** In our document that was 9 blocks, and they are precisely what stacks
the columns on a phone. So the email inlines "successfully" and then does not respond, with nothing
reported anywhere.

It is less a bug than a logical limit: a media query cannot become a `style` attribute, so it has to
travel in a `<style>` block in the `<head>`. Extract the blocks from the source CSS by **counting
braces** — a regex breaks on nested rules — and reinsert them after inlining:

```js
function extractMedia(source) {
  const blocks = [];
  let i = 0;
  while ((i = source.indexOf("@media", i)) !== -1) {
    const open = source.indexOf("{", i);
    let depth = 1, j = open + 1;
    while (j < source.length && depth > 0) {
      if (source[j] === "{") depth++;
      else if (source[j] === "}") depth--;
      j++;
    }
    blocks.push(source.slice(i, j));
    i = j;
  }
  return blocks;
}
```

After any inlining run, grep the output for `@media` before shipping. Its absence is silent.

## Getting a large HTML document into an OutSystems literal

If you take the finished-HTML route, the document becomes a literal in an Assign. Three rules, in
order of how much pain they save:

**Use `Replace()` on markers instead of cutting the string.** The tempting approach is to slice the
HTML at each parameter and concatenate with `+ Name +`. Leave markers in the HTML and resolve them
afterwards:

```
Replace(Replace(Html, "{{Name}}", Name), "{{Link}}", Link)
```

Now the HTML is **one** literal, pasted without being edited — and regenerating the file is a fresh
copy-paste instead of a fresh set of cuts. This is the difference between a five-minute update and
an afternoon.

**Double the quotes.** An OutSystems text literal escapes `"` as `""`. Our document had 456 quotes,
all in attributes. That is a script's job, not a person's.

**Flatten to one line.** Do not depend on how the expression editor treats newlines inside a
literal. HTML and CSS do not need them, and flattening removed 3.5 KB of indentation for free.

## ODC does not give you a mail server

Without SMTP configured in the Portal, the send fails with *"The 'SMTP server' must be set in
Portal's Email Configuration"*. It lives under **Portal → Email**, per environment — so configuring
Development does nothing for Production.

Two frictions worth knowing in advance:

- **Gmail refuses** with `5.7.8 BadCredentials` unless you use an app password rather than the
  account password.
- **Portal changes take one to two minutes to take effect.** An immediate retry can fail for the
  previous reason and send you chasing a problem you already fixed. Wait, then retry once.

## Test against a capture server and read the raw source

For testing, a capture server (Ethereal and similar) is the right tool: nothing reaches a real
inbox, and it shows the **raw source** of what arrived — which is the only thing that answers the
actual question, whether your tables and your `style` attributes survived.

"It looks right in my client" means very little here. One client is not a test, and the client most
likely to disagree is the one you do not have installed. [Can I email](https://www.caniemail.com/)
is the per-client support table.

## Checklist

- Is this transactional with a simple layout? Then widgets, and stop
- Nobody is looking for `Escape Content` in ODC — it does not exist there
- If finished HTML is required, the `TextToBinaryData` route was tested **before** an external
  library was chosen
- Is the layout table-based, or has Outlook desktop been explicitly dropped with the stakeholder's
  knowledge?
- After inlining: output grepped for `@media`, media blocks reinserted in `<style>`
- The HTML is one literal with `{{markers}}`, not a concatenation of slices
- SMTP credentials in secret settings, unreachable from a Client Action, log or trace; no test
  endpoint left exposing the send
- SMTP configured in **every** environment that sends, not just Development
- Delivered to a capture server and the **raw source** read, not just the rendered preview
