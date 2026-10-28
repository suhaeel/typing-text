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
2. (info, pre-existing, NOT caused by this PR) In the editor on Typing Text, the console shows an uncaught `Failed to execute 'observe' on 'MutationObserver': parameter 1 is not of type 'Node'` from `waitForElement` (`observer.observe(document.body, ...)`, `document.body` is not a Node in the frame where it runs). It appears identically on `dev` and with this PR (the line number only shifts from 72 to 68). It was not seen on Testimonial or Toggle Content. Worth a separate fix (guard for a missing `document.body`, or wait for it). Cause of why only Typing Text hits it was not investigated.
3. (info) The PR text says the script is registered under the shared handle `essential-blocks-eb-animation` and "first plugin wins". In Typing Text the handle is its own, `typing-text-eb-animation` (`typing-text.php`), so this plugin's copy always loads; the change still has to be shipped in every plugin that carries a copy.
4. (info, pre-existing) `replaceAnimationClasses` queries only the main document, not the block-editor iframe canvas. Unchanged behaviour.
5. (info) `setTimeout(..., 500)` is kept as before. Frontend behaviour is untouched (admin-panel handler only).
