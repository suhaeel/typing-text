# Review: PR #34 — Replace deprecated `DOMNodeInserted` listener in `eb-animation-load.js`

- PR: https://github.com/EssentialBlocks/typing-text/pull/34
- Branch reviewed: `fix/animation-mutation-event` (copy: `review/typing-text-pr-34`), base `dev`
- Reviewer: Abdullah Aas Suhaeel
- Verdict: **Approve**, with a suggestion to rebase so the PR diff shows only the real change

## What the PR does
`assets/js/eb-animation-load.js` attached a `DOMNodeInserted` listener for the admin panel. The PR replaces it with one capture-phase `change` listener on `document` that checks `event.target.id === 'eb-animation-style'`.

## "19 files" is a display artifact
The branch was cut from `fix/audit-followups-1.3.1`, squash-merged into `dev` as #33, so GitHub still lists that PR's commits. A 3-way merge into `dev` (`git merge-tree origin/dev origin/fix/animation-mutation-event`) is **conflict-free** and changes **1 file**: `assets/js/eb-animation-load.js`, +7 / -11.

Suggestion: `git rebase --onto origin/dev d31615d` so the PR shows just that one commit.

## Does it work? (tested in a real browser)
Headless Chrome 153, WordPress 7.1.3, Local site, plugin active.

1. `DOMNodeInserted` listeners registered while the editor loads: **2 before, 0 after** (the test site also runs another plugin with its own copy of the script; the route override replaced every copy).
2. Functional test of the admin handler: add `<div class="eb___animated eb___fadeIn">` in view, add `<select id="eb-animation-style">` after page load (like the inspector), fire `change` on the select, wait 900 ms.

| JS | Div classes after `change` |
|---|---|
| old (`dev`) | `eb___animated eb___fadeIn` (nothing happened) |
| PR #34 | `eb__animated eb__fadeIn` (handler ran) |

So this is not only a console-warning cleanup: Chrome removed mutation events in Chrome 127, so on current Chrome the old admin handler **never ran**. The PR fixes a real functional bug. Caveat: the select was injected by script, not driven through the real inspector UI.

## Findings
No blocking bugs introduced by this PR.
1. (suggestion) Rebase onto `dev` as above.
2. (info, pre-existing, NOT caused by this PR) **Uncaught `MutationObserver` error in the editor canvas iframe.** See the section below.
3. (info) The PR text says the script is registered under the shared handle `essential-blocks-eb-animation` and "first plugin wins". In Typing Text the handle is its own, `typing-text-eb-animation` (`typing-text.php`), so this plugin's copy always loads; the change still has to be shipped in every plugin that carries a copy.
4. (info, pre-existing) `replaceAnimationClasses` queries only the main document, not the block-editor iframe canvas. Unchanged behaviour.
5. (info) `setTimeout(..., 500)` is kept as before. Frontend behaviour is untouched (admin-panel handler only).

## Separate bug found while testing (pre-existing, not part of this PR)

**Symptom.** Opening the block editor with Typing Text active logs: `Uncaught (in promise) TypeError: Failed to execute 'observe' on 'MutationObserver': parameter 1 is not of type 'Node'` from `assets/js/eb-animation-load.js` (`waitForElement`, line 72 on `dev`, 68 with this PR).

**Root cause (instrumented in the browser).**
- The script is loaded not only in the main editor page but also inside the **editor canvas iframe** (a `blob:` document). Typing Text registers it as a dependency of the block's front-end script (`eb-typing-text-frontend` depends on `typing-text-eb-animation`, and that script is attached to the block via `'script' => 'eb-typing-text-frontend'` in `typing-text.php`), and a block's `script` also loads in the editor canvas.
- In that iframe, `DOMContentLoaded` fires while `document.readyState` is `interactive` but `document.body` is still `null`.
- `waitForElement()` does `observer.observe(document.body, ...)` with no guard, so `observe(null)` throws. The throw happens inside a `new Promise(...)` executor, so it becomes an unhandled promise rejection, and the `.then(...)` that does the initial `replaceAnimationClasses(...)` and registers the `scroll` listener never runs in that iframe.

**Impact.**
- Front end (visitor) is **not affected**: a published page with the Typing Text block loaded `typed.min.js`, `eb-animation-load.js` and `frontend/index.js` with **no page errors**, because `document.body` exists there.
- Editor: an uncaught console error (intermittent: reproduced in 3 runs, absent in one, it depends on timing), and the animation initialisation is skipped in the canvas iframe.
- **User-visible impact for Typing Text itself: none found.** The Typing Text block has no entrance-animation control: the built bundle (`dist/index.js`, `dist/frontend/index.js`) contains no `eb___` / `eb__` animation classes and no `eb-animation-style` select. The animation script is only loaded because the block's front-end script lists it as a dependency. So today this is console noise, not a broken feature.
- Same on `dev`: reproduced on a branch that only adds the CSS of #35 (JS identical to `dev`), so it is not introduced by #34 or #35.
- Not reproduced on Testimonial or Toggle Content in the same site. Whether their copy loads in the iframe was not checked, so the reason only Typing Text shows it is **not confirmed**.

**Suggested fix (separate small PR).** Make `waitForElement` safe when `document.body` is missing, e.g. observe `document.documentElement`, or wait for `body` before observing:

```js
observer.observe(document.body || document.documentElement, { childList: true, subtree: true });
```

Also add a `.catch(...)` or otherwise handle the rejection so one frame cannot leave an unhandled rejection.

## Other problems / notes found in this plugin
- PR #34's description says the script uses the shared handle `essential-blocks-eb-animation` ("first plugin wins"). In Typing Text the handle is its own, `typing-text-eb-animation`, so this plugin's copy always loads and the fix must ship in every plugin that carries a copy.
- `replaceAnimationClasses` queries only the main document, so it never reaches blocks inside the canvas iframe even when the script runs there (pre-existing, unchanged).
