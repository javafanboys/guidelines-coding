# Web UI — verifying what you built

Read this whenever you produce HTML/CSS for a human to look at: a static site generator, a server-rendered page, a JS app, an emailed HTML body, a slide exported to a page. It is not tied to any framework or language.

## The principle

A typed markup DSL (luvml/luvs), a template engine, a linter and a green test suite all guarantee the same one thing: **the markup is exactly what you wrote.** None of them can tell you that what you wrote is wrong. Every visual defect ever shipped compiled clean, validated, and passed its tests.

So the check is not "did it build". The check is **walk the links, then look at the page.** Both, every time, after every generation run.

## 1. Walk the links

For each `.html`, resolve every relative `href`/`src` against its own directory and assert the target exists.

```bash
cd <out-dir>
for f in $(find . -name "*.html"); do d=$(dirname "$f")
  grep -o 'href="[^"#]*"' "$f" | sed 's/href="//;s/"$//' \
    | grep -vE '^(https?:|mailto:|//)' \
    | while read -r l; do [ -e "$d/$l" ] || echo "BROKEN $f -> $l"; done
done
```

This is not theoretical. It caught a live 404 on **all 21 record pages of a site** — a helper that builds a link prefixed the collection folder, so a page already *inside* `/treaties/` needed `rel = "../"` and was passed `""`, producing `/treaties/treaties/x.html`. Nothing else in the build could see it.

## 2. Look at the page

Taking the screenshot is not the check. **Reading it is.**

```
"C:/Program Files/Google/Chrome/Application/chrome.exe" --headless=new --disable-gpu \
  --hide-scrollbars --virtual-time-budget=4000 --window-size=1440,1500 \
  --screenshot="shot.png" "file:///abs/path/index.html"
```

`--virtual-time-budget` is required, or the shot fires before web fonts load and you review a page in the wrong typeface. `--window-size` height IS the capture height — there is no full-page flag in this mode — so set it taller than the page or you never see the footer.

**Say what you expect to see before you look.** An assumption you never stated is not tested by a screenshot you glanced at.

### Trap: `--window-size` cannot go narrow

Chrome clamps its minimum window width (~489px) and then crops the PNG to whatever width you asked for. Request 420px and you get a 420px **crop of a 489px render**, which looks exactly like catastrophic horizontal overflow and is not. This has produced a confidently wrong diagnosis.

To test narrow layouts, render at the real width in an iframe inside a large window:

```html
<div style="display:flex;gap:14px">
  <iframe src="index.html" width="390" height="1500"></iframe>
  <iframe src="index.html" width="768" height="1500"></iframe>
</div>
```

### Measure overflow, do not infer it

```js
document.documentElement.scrollWidth === document.documentElement.clientWidth  // no overflow
```

Equal means no overflow whatever the picture looks like. `--dump-dom` runs after load, so write the result into `document.title` and grep the dump.

## The defect classes that reach production

Every one of these passed the compiler and the link check, and was caught only by looking. They are listed as a checklist to run your eye down, not as a history:

- **Inline vs block** — `margin-top` on a `<span>` is silently dropped, and two elements that should stack run onto one line.
- **Colour inheritance into a wrapper `<a>`** — a whole card wrapped in an anchor takes the global link colour through its entire body.
- **Layout semantics that are valid but wrong** — `align-items: start` plus `min-height: 100%` means a sidebar column never stretches. Valid CSS, broken column.
- **Hover leakage** — a global `a:hover { text-decoration: underline }` underlines every wrapper anchor until it is overridden.
- **Missing padding wrapper** — a list dropped straight into a panel with no inner body element, so every row sits flush on the panel border.
- **A component used on the wrong surface** — a pale ghost button built for a dark hero is invisible when reused on a white page. Light surfaces need the outline variant.
- **Two surfaces sharing a colour** — page and sidebar both the same tone, so the content box's edge vanishes. Keep adjacent surfaces distinct.
- **Missing margin between sections** — a card grid with no bottom margin collides with the panel below it.
- **Unbalanced auto-fit grid** — four cards break 3 + 1 at mid widths until the min track shrinks and an explicit column rule is added at the awkward breakpoint.
- **Doubled rules at a junction** — a nav and a hero that both end in a rule, plus a content-area top border directly beneath, read as a stray bar.
- **A text heuristic that misfires on one input in thirty** — a heading detector turned a court order's cover sheet, party list and footnote fragments into a 40-entry table of contents of pure noise. Nothing in the build could see it; the page compiled, linked and validated. **Two lessons: screenshot the input that is *least* like the others, and give any heuristic a sanity gate that withdraws its own output when it fires too often.**
- **Stale output** — a generator that writes into the publish directory without emptying it first leaves every renamed page live at its old URL. `ls` the output and count; do not trust the run log.
