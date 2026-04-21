# iA Writer Template Modification Learnings

## How iA Writer manages template files

iA Writer stores installed templates in its SQLite database (`~/Library/Containers/pro.writer.mac/Data/Library/Application Support/iA Writer/index.db`). The `.iatemplate` bundle files in `~/Library/Containers/.../Templates/` are a cache derived from that database.

**Do not edit the cache directly.** iA Writer watches those files and will revert or delete them when it detects external changes. Always edit the source files in your repo and reinstall via double-click.

## Correct workflow

1. Edit `style.css` or `document.js` in the repo
2. Double-click `letter.iatemplate` in Finder — iA Writer prompts to install/replace
3. Open (or re-open) a document using the Letter template
4. Open print preview to verify changes

## JavaScript injection does not work in the print renderer

`window.addEventListener('load', ...)` runs in the normal editing view but not in iA Writer's print renderer. DOM elements injected via JS do not appear in print preview or printed output.

**Use CSS instead.** The print renderer fully processes `@media print` rules including `position: absolute`, `background-color`, and pseudo-elements.

## CSS pseudo-elements are the correct approach for print-only marks

`body::before` and `body::after` work reliably in the print renderer:

```css
@media print {
    body {
        position: relative;
        overflow: visible;
    }

    body::before,
    body::after {
        content: '';
        display: block;
        position: absolute;
        left: -2.7cm;   /* places mark in the left margin */
        width: 0.4cm;
        height: 1px;
        background: #aaa;
        -webkit-print-color-adjust: exact;
        print-color-adjust: exact;
        margin: 0;
        padding: 0;
        box-sizing: content-box;
    }
}
```

- `position: relative` on `body` is required so that `position: absolute` children are anchored to the body
- `overflow: visible` is required because the screen CSS sets `overflow: hidden` on body; without overriding this in print, marks positioned outside the body box (negative `left`) are clipped
- `print-color-adjust: exact` (and the `-webkit-` prefix) prevents the print renderer from stripping background colors

## Coordinate system for position: absolute in print

The `top` value for `position: absolute` children of `body` is measured from the top edge of the body element, which in the print layout sits approximately 26mm below the top of the physical A4 page (with the default print margins iA Writer applies).

For DIN 5008 fold marks at 99mm and 198mm from the physical page top:
- `top: 73mm` → renders at ~99mm from physical page top
- `top: 172mm` → renders at ~198mm from physical page top

## The *:before, *:after print rule

The template's print block contains:

```css
*, *:before, *:after {
    color: var(--text-color) !important;
    box-shadow: none !important;
    text-shadow: none !important;
}
```

This sets text `color` to black but does **not** affect `background-color` or `border-color`. Fold marks using `background` are unaffected. If using `border-top` for marks, set the color explicitly in the shorthand (e.g. `border-top: 1px solid #aaa`) rather than relying on `currentColor`.
