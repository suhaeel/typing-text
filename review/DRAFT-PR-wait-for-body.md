# DRAFT (on hold, not opened): fix: don't throw in waitForElement when document.body is null

Status: **ready but NOT opened upstream.** To be discussed before opening.
Fix branch (on the fork): `fix/animation-wait-for-body`, commit `8951c9e`, based on `origin/dev`.
Target if opened: `EssentialBlocks/typing-text`, base `dev`.

## Suggested PR body

In the block editor, `assets/js/eb-animation-load.js` is also loaded inside the editor canvas iframe (Typing Text's block `script` depends on the animation script, and a block's `script` loads in the canvas). There, `DOMContentLoaded` fires while `document.body` is still `null`, so `waitForElement()` calls `observer.observe(null, ...)`, which throws inside a Promise executor and leaves an unhandled rejection:

`Uncaught (in promise) TypeError: Failed to execute 'observe' on 'MutationObserver': parameter 1 is not of type 'Node'`

The fix observes `document.body || document.documentElement`.

- One line changed in one file (plus a comment).
- Before / after in headless Chrome 153, WordPress 7.1.3: console error present on `dev`, gone with the fix. An animated element added to the canvas iframe is converted (`eb___` to `eb__`) only with the fix.
- Front end unchanged: a published page with the block loads `typed.min.js`, `eb-animation-load.js` and `frontend/index.js` with no errors, before and after.
- Merges cleanly with `dev` and with #34.

## Honest limits (keep these in the PR if opened)
- The error is **intermittent** (timing): reproduced in 3 runs, absent in 1.
- Impact for Typing Text is **console noise**: this block has no entrance-animation control, so nothing user-visible breaks. The animated-element check used an injected element, not a real block.
- Only Typing Text loads the script in the canvas iframe. Toggle Content, Testimonial, Openverse and Notice Block were each tested alone: script only in the main editor page, no errors. The same unguarded line exists in all of them and in Essential Blocks core (`assets/js/eb-animation-load.js`, line 83); core was not tested.

## Question to settle before opening
Is the guard the right fix, or should Typing Text stop loading the animation script at all (drop `typing-text-eb-animation` as a dependency of its front-end script)? The second option removes the cause but needs a check that `frontend/index.js` does not rely on it. The one-line guard is the safer, smaller change.
