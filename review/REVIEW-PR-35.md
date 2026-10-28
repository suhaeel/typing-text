# Review: PR #35 — Keep the block icon visible when the editor highlights it

- PR: https://github.com/EssentialBlocks/typing-text/pull/35
- Branch reviewed: `fix/block-icon-selected` (copy: `review/typing-text-pr-35`), base `dev`
- Reviewer: Abdullah Aas Suhaeel
- Verdict: **Approve** (2 optional nits below)

## What the PR does
The Typing Text icon is filled `#6C3BFF`. On the blue highlighted row of the `/` inserter, a selected List View row and the drag chip, that purple almost disappears. The PR adds `assets/css/block-icon-selected.css` (recolours those fills to `currentColor`) and enqueues it on `enqueue_block_editor_assets`.

## Scope check
3-way merge into `origin/dev` (`git merge-tree`): conflict-free, 2 files, +56 lines (the CSS file and `typing-text.php`). No change to `dist/`, `controls`, `lib/` or saved block markup, so existing posts cannot be affected.

## How it was tested
Local site, WordPress 7.1.3, PHP 8.2.29, headless Chrome 153, plugin linked in and activated. The PR author states this plugin was not tested in a browser (only accordion-toggle and infobox), so it was tested here. "Before" = same session with the stylesheet request blocked.

| Where | Before | After |
|---|---|---|
| `/` inserter highlighted row (bg `rgb(56,88,233)`) | icon fill `rgb(108,59,255)` | `rgb(239,240,242)` |
| Selected List View row | `rgb(108,59,255)` | `rgb(255,255,255)` |

Screenshots: `review/screenshots/pr35-*.png`. Before, the icon is almost invisible on the blue row; after, the white "T" box is clearly visible.

Other checks
- `php -l typing-text.php`: no syntax errors (PHP 8.2.29).
- This icon differs from the others: `src/icon.js` is a full 60x60 rounded square (`#6C3BFF`) with the "T" cut out, plus a small bar. Recoloured to the row's text colour it stays legible (see screenshot).
- Stylesheet loads once, editor only. No PHP notices.
- Not tested: the drag chip (same rule set, only the colour differs) and browsers other than Chrome.

## Findings
No blocking bugs.
1. (nit) `'1.0.0'` is hard-coded as the style version. The handle is shared across the standalone plugins, so the first plugin to enqueue wins; a later CSS change can be served from cache.
2. (nit) The `#C5B1FF` and white-knockout rules are unused by this icon. Harmless dead rules in a shared file.
3. (info) The PR says the rules match Essential Blocks' `block-icon-selected.css`; that file is not present in the local `essential-blocks-for-gutenberg` `main`, so the claim could not be verified.
4. (info, not caused by this PR) In the editor on Typing Text, the console shows an uncaught `Failed to execute 'observe' on 'MutationObserver': parameter 1 is not of type 'Node'` from `assets/js/eb-animation-load.js` (`waitForElement`). It appears identically with and without this PR. See REVIEW-PR-34.md.
