# Overnight audit playbook: functional-ui-kit

Written 2026-09-24 by the planning session on branch `claude/code-audit-overnight-plan-b0omr8`.
Everything in section 2 was checked in a real container against `main` at commit `4070a13`.

## 1. The job

Audit this repo against current best practices and come back with work another agent can verify tomorrow.

The owner's hard rule: absolutely zero regression to the experience. Nothing that users of the npm package, users of apps built with it, or visitors of the Storybook site can see, click, hear through a screen reader, or type-check may change unless the owner opts in.

Fixing a bug changes behavior by definition. So the output is split in two:

- Lane A (merge-safe): the regression harness, the audit report, and only changes proven invisible. Merging Lane A must change nothing for users.
- Lane B (opt-in fixes): real fixes, one per commit, each with proof of exactly what changes. The owner picks which ones to take.

"The experience" covers:

1. npm consumers: the JS in `dist/`, `dist/style.css`, the `.d.ts` types, the subpath entry folders (`fui-button/package.json` and friends), the published file list.
2. End users of apps built on the kit: pixels, layout, focus rings, keyboard behavior, screen reader output, which handlers fire on click or keypress, console warnings.
3. Visitors of the Storybook docs site (GitHub Pages, redeployed from `main` on every push): every story, every docs page, the "Show code" snippets, light and dark themes.
4. The maintainer: `npm run build`, `npm run storybook` and `npm run storybook:build` keep working.

Nothing reaches npm until someone runs `npm publish` by hand (there is no publish workflow). Merging to `main` only redeploys the docs site.

## 2. Verified facts about this repo

- React 18 component library. 16 components in `src/components/*`, global CSS in `src/css/*`, one CSS file per component. Storybook 7.6 (webpack5) for docs. About 6,600 lines in total.
- There are zero tests: no unit tests, no Storybook play functions, no visual tests. The only CI is `.github/workflows/deploy-github-pages.yaml`, which deploys Storybook on push to `main`. Pull requests get no checks.
- `package-lock.json` is gitignored, so every `npm install` can resolve different versions. Install once and reuse that same `node_modules` for every build you compare. Never reinstall between a baseline build and a candidate build.
- `npm install` takes about 1 minute and succeeds with deprecation warnings. It also runs `husky install`, which turns on a pre-commit hook (lint-staged running `eslint --fix` on staged `src/**/*.ts`). After every commit, run `git show --stat HEAD` and confirm the hook changed nothing you didn't intend.
- `npm run build` (Vite library mode, CJS only) takes about 5 seconds and is deterministic: two builds of the same source give a byte-identical `dist/` (38 files, compared by sha256).
- `npm run storybook:build` takes about 17 seconds. `storybook-static/index.json` lists 90 entries: 71 stories and 19 docs pages (3 MDX pages and one autodocs page per component).
- `npm run lint` runs `eslint ./src --fix`, which rewrites files. Never run it. For read-only linting use `npx eslint ./src`. Current baseline: 110 problems (109 errors, 1 warning, 88 auto-fixable).
- `npx tsc --noEmit` reports 9 errors today, all in stories (8 in `fui-button.stories.tsx`, 1 in `fui-notification.stories.tsx`), caused by how `defaultProps` is typed.
- `npm pack --dry-run --json` lists 58 files (59.9 kB packed). It prints a husky line before the JSON, so strip everything before the first `[` when parsing.
- Playwright 1.56.1 is installed globally with Chromium build 1194 in `/opt/pw-browsers`. Pin `@playwright/test` to exactly `1.56.1` so it finds that browser. Do not run `playwright install`. Firefox and WebKit are not available.
- `http-server` and `serve` are installed globally.
- Machine: 4 CPUs, 15 GB RAM, about 29 GB free disk.
- Screenshots of the static Storybook are pixel-identical across repeated runs (checked on `components-button--hierarchy`).
- Dark mode: the preview decorator listens for the storybook-dark-mode channel event. In `iframe.html`, switch it with `window.__STORYBOOK_ADDONS_CHANNEL__.emit('DARK_MODE', true)`, which sets `body[data-theme="dark"]`. Setting the attribute by hand does not stick, because the decorator resets it on every render.
- Fonts: `.storybook/preview-head.html` loads Inter and IBM Plex Mono from Google Fonts, and the shipped `src/css/main.css` `@import`s Atkinson Hyperlegible and Lato from Google Fonts. In this container Chromium rejects the sandbox proxy's certificate (`ERR_CERT_AUTHORITY_INVALID`). With `ignoreHTTPSErrors: true` the fonts load, but some requests fail at random (`ERR_TOO_MANY_RETRIES`). Left alone, this makes screenshots flaky. Section 5, Phase 2 says how to handle it.
- Actions: story `on*` args are wired to the actions addon automatically. To see which handlers fired, listen on the preview channel: `window.__STORYBOOK_ADDONS_CHANNEL__.on('storybook/actions/action-event', e => ...)`, where `e.data.name` is the handler name.
- Playwright treats `aria-disabled="true"` elements as not clickable. Use `click({ force: true })` to simulate what a real user can do.
- `FuiRadio`, `FuiTextInput` and `FuiStepper` build element ids with `Math.random()`. DOM snapshots differ on every render unless these ids are normalized.
- The static Storybook build uses production React, so React development warnings (the `defaultProps` deprecation, for example) never appear there. To collect them, visit every story once on the dev server (`npx storybook dev --ci --no-open -p 6006`).

## 3. Hard rules

Never:

1. Push to `main` or to any branch other than your two lane branches. Never merge a PR. Never force-push or rewrite pushed history.
2. Run `npm publish`, `npm version`, `npm run storybook:deploy` (uploads to Chromatic), or anything else that publishes or deploys.
3. Run `npm run lint`, `eslint --fix`, `prettier --write`, or any other mass reformatting.
4. Add, remove, or upgrade root dependencies, run `npm audit fix`, commit the root `package-lock.json`, or edit the root `package.json`.
5. Edit `.github/workflows/*`, `.storybook/*`, `vite.config.ts`, `tsconfig.json`, `.babelrc.json`, `.eslintrc.js`, `.husky/*`, `CODEOWNERS` or `LICENSE` in either lane. Changes there go in the report as a proposed diff.
6. Add, rename, or remove stories or docs pages.
7. Weaken, skip, or loosen a harness layer, threshold, or normalization rule to make a change pass. A difference is a difference. If you believe the harness itself is flaky, fix the harness, rerun the full self-test, and write down what changed and why.
8. Rotate tokens, change repository settings, or post GitHub comments beyond the two PR descriptions.
9. Stop to ask the owner anything. When unsure, take the conservative option (leave it unchanged), log the decision in `audit/PROGRESS.md`, and move on.
10. Copy a secret value into any file, commit, or PR. Point to the file and line instead.

## 4. Lanes (mechanical rules)

### Lane A

Your designated branch, created from `origin/main`. Draft PR into `main`.

A change belongs in Lane A only if one of these is true:

- It only touches `tools/regression/**`, `audit/**`, a new `CLAUDE.md`, or appends lines to `.gitignore`.
- It touches `src/**`, the built `dist/` is byte-identical to main's (sha256 of every file), and the harness reports zero differences in every layer.

Everything else is not Lane A. Expect Lane A to contain few or no `src/` changes. That is the correct outcome, not a failure.

### Lane B

A second branch named `<your designated branch>-fixes`, created from the Lane A head. Draft PR into the Lane A branch, so its diff shows only the fixes. If Lane A gets new commits after Lane B starts, merge Lane A into Lane B (never rebase pushed history).

- Fix commits contain only `src/**` changes plus the specs that prove them in `tools/regression/specs/**`. The one exception is a final docs-only commit that adds the per-fix summaries in `audit/lane-b/`.
- One finding per commit. Each commit must apply cleanly to the Lane A head on its own, so the owner can cherry-pick any subset in any order. If two fixes truly depend on each other, put them in one commit and say why.
- Local fixes only, about 60 changed lines in `src/` at most. Bigger redesigns go in the report.
- Before running the harness on a fix, write down which snapshots you expect to change, for example: "DOM and ARIA of the button stories and of every story that renders a disabled FuiButton: `disabled` attribute added; no pixel changes." Every reported difference must match the prediction. An unexplained difference means you rework or drop the fix.
- Every fix comes with a spec that fails on the parent commit and passes on the fix commit. Use Playwright against the built Storybook by default, and a node script for things a browser can't show (server rendering and hydration, for example).
- Every fix gets a risk grade. B1: the harness shows zero differences (shipped code changed, but nothing observable in the stories did). B2: the harness shows only the predicted differences.
- Commit message: `fix(<component>): <what users will notice> [AUDIT-<id>]`. Body: the user-visible change, semver impact (patch, minor or major), the snapshots that changed, the risk grade, and how to revert.

### Report-only

Everything else: dependency upgrades, packaging changes, CI changes, `.storybook` changes, redesigns, and owner decisions such as rotating a token. Write each one up with a ready-to-apply diff where possible.

## 5. Phases

Keep `audit/PROGRESS.md` as the single source of truth: start time, current phase, what's done, every decision, the next step. Commit and push it at the end of every phase, so a restarted or summarized session can pick up from it. Check `date -u` at every phase boundary.

Target timeline, measured from the start time recorded in PROGRESS.md: harness self-test passed by about 3 h, audit and verification done by about 5.5 h, stop starting new Lane B fixes at 8 h, and keep the last 1.5 h for Phase 7.

### Phase 0: Setup (about 20 min)

1. `git fetch origin main` and create Lane A from `origin/main`. First commit: this playbook, copied to `audit/OVERNIGHT_AUDIT_PLAYBOOK.md`, plus `audit/PROGRESS.md` with the start time.
2. If the `send_later` tool exists, schedule a self check-in with it, 2 hours out, with this message: "Overnight audit check-in: read audit/PROGRESS.md and continue with the next unfinished phase. Re-arm this check-in for 2 hours unless both PRs are open and Phase 7 is done." Re-arm it at every check-in. This keeps the night going if a turn ends early.
3. `npm install` once. Record `npm ls --depth=0` and `node -v` in PROGRESS.md.
4. Quickly re-verify section 2 (build, Storybook build, lint and tsc counts). If anything differs, note it and adapt.
5. Start Phase 1 as a background subagent, then go straight to Phase 2 yourself.

### Phase 1: Best-practice research (background, about 45 min)

Use the `deep-research` skill, or WebSearch and WebFetch directly. Output: `audit/CHECKLIST.md`, a list of concrete, checkable items, each with its source link. Primary sources only. Cover:

- WAI-ARIA Authoring Practices Guide patterns for every component: Button, Checkbox (including the mixed state), Radio Group, Switch, Listbox and the select-only Combobox (for FuiSelect), Dialog (Modal), Tooltip, Alert and status messages, Spinbutton (for FuiStepper), and a radio-group style segmented control (for FuiToggle). https://www.w3.org/WAI/ARIA/apg/patterns/
- WCAG 2.2 AA criteria that apply to components: 1.3.1, 1.4.3, 1.4.11, 1.4.13, 2.1.1, 2.1.2, 2.4.3, 2.4.7, 2.4.11, 2.5.8, 3.3.2, 4.1.2, 4.1.3. https://www.w3.org/TR/WCAG22/
- React: the Rules of React, "You Might Not Need an Effect", `useId` for element ids, and the React 19 upgrade guide (`defaultProps` removed for function components, `element.ref` removed). https://react.dev
- Library packaging: publint rules (https://publint.dev/rules) and Are The Types Wrong (https://arethetypeswrong.github.io).
- Testing: Playwright visual comparisons and ARIA snapshots (https://playwright.dev/docs/test-snapshots, https://playwright.dev/docs/aria-snapshots), the axe-core rules and the limits of automated accessibility testing, the Storybook 7 testing docs.
- Supply chain and CI: GitHub's security hardening guide for Actions (pin third-party actions to a full commit SHA, least-privilege permissions), npm guidance on lockfiles, Chromatic's guidance on keeping project tokens secret.
- CSS: `prefers-reduced-motion`, `forced-colors`, `:focus-visible`, the performance cost of CSS `@import` chains, and the privacy problem with loading Google Fonts from Google's servers.
- Process: separating structural changes from behavior changes (Kent Beck, "Tidy First?") and characterization, or golden-master, tests before changing legacy code (Michael Feathers).

### Phase 2: Build the regression harness and prove it works (about 2.5 h)

This is the most important phase. No audit changes happen until its self-test passes.

Location: `tools/regression/`, with its own `package.json` (exact pinned versions, `@playwright/test` at `1.56.1`) and its own lockfile. The root `.gitignore` ignores every `package-lock.json`, so append `!tools/regression/package-lock.json`, and also ignore the harness output and cache folders. Document usage in `tools/regression/README.md`.

Interface: `node tools/regression/run.mjs --base <ref> --head <ref> [--mode fast|full] [--filter <story-id-prefix>]`. It compares two git refs from scratch every time. No baselines are stored in git. The harness code always runs from your working tree; only the product sources come from the two refs.

Building each ref:

- Export the ref into a temp directory outside the repo (`git worktree add --detach`, or `git archive`). Give it the same `node_modules` (a symlink, or `cp -al` hardlinks if symlinks break module resolution).
- Build the two refs one after the other, never in parallel, and clear `node_modules/.cache` before each Storybook build, so one ref's build cannot leak into the other's.
- Run `npm run build` and `npx storybook build` there, and keep both outputs outside the repo.
- Serve each `storybook-static` on its own local port.

Layers. Each one captures base and head the same way and diffs them:

1. Package: sha256 of every file in `dist/`; the full text of every `.d.ts`; the `npm pack --dry-run --json` file list; the export names and `typeof` of each `require('dist/<entry>/index.js')`; raw and gzip sizes; the list of Storybook entries in `index.json` (id, title, name, type).
2. Pixels: every `index.json` entry (71 stories and 19 docs pages), in light and dark, at 1280x800 and 375x812. Full-page screenshots with animations disabled and the caret hidden. Wait for network idle and `document.fonts.ready`, then keep taking screenshots until two in a row match. Compare with pixelmatch at threshold 0: any differing pixel is a difference. On docs pages, click every "Show code" button first.
3. DOM: `document.body.outerHTML` after render. floating-ui and react-modal render portals into `body`, so capture all of `body`. Normalize only what is random by construction: the `Math.random()` ids (`fui-(radio|text-input|stepper)-(input|label)-<random>`) and React `useId` ids. Replace each with a stable token in order of first appearance, and rewrite every reference to it (`for`, `aria-labelledby`, `aria-describedby`, `aria-controls`) the same way. List every normalization rule in the README.
4. Accessibility tree: `page.locator('body').ariaSnapshot()`.
5. axe: `@axe-core/playwright` violations per entry, sorted by rule id and target.
6. Console: errors, warnings and page errors per entry, from the static build and from one pass over the dev server (the dev pass is what catches React warnings).
7. Interactions (story entries only, light theme, 1280 wide, at most about 30 elements per story):
   - Tab walk: press Tab until focus cycles or 25 presses. After each press, record the focused element (tag, role, accessible name, classes), a viewport screenshot and the ARIA snapshot.
   - Activation walk: for each focusable or `.fui-interactable` element, reload the story and click it with `force: true`. Separately, focus it and press each of Space, Enter, ArrowDown, ArrowUp and Escape, reloading between presses. After each, record the DOM, a screenshot, and which action handlers fired (from the actions channel) with their arguments.
   - Hover walk: hover each such element and take a screenshot. This covers hover states and tooltips.

Determinism requirements:

- Fonts: serve every `fonts.googleapis.com` and `fonts.gstatic.com` request from a local cache (`tools/regression/.cache/fonts`, gitignored). On a cache miss, fetch through the proxy with retries and store the response. Google Fonts returns different CSS for different browsers, so send the same User-Agent the browser sends. If a font cannot be fetched after retries, abort the run with a clear error. A font must never fall back silently in one run and not in the other.
- Base and head always use the same browser, machine, `node_modules` and fonts, and are always captured in the same run.
- Use 3 to 4 Playwright workers.
- Fast mode: layer 1, plus layers 2 to 6 at 1280 only. Full mode: everything. `--filter` limits layers 2 to 7 to matching story ids, for quick iteration only; it never counts as a gate.

Self-test. It must pass before Phase 3. Record the results in `audit/HARNESS_SELF_TEST.md`.

1. Run full mode with base and head both set to `origin/main`, twice. Both runs must report zero differences. If not, fix the harness and repeat.
2. Canaries. On a scratch local branch that is never pushed, make one throwaway commit per canary, run the harness against `origin/main` for each, and confirm the named layer flags it:
   - one light-theme color token changed by one unit (for example `#ffffff` to `#fffffe`): pixels, light only
   - one dark-theme token changed by one unit: pixels, dark only
   - a harmless extra class on one component's root element: DOM
   - one changed `aria-label`: ARIA
   - one component's click handler made to do nothing: interactions (actions fired)
   - one component made to ignore the Space key: interactions
   - one prop type changed: package (`.d.ts`)
   - a `console.warn` added inside a component: console (dev pass)
   - `margin-top: 1px` added in a component's CSS: pixels and package (`style.css` hash)

   Every canary must be caught. A missed canary is a hole in the net: fix it before continuing. Delete the scratch branch afterwards.
3. Commit the harness and the self-test results. Push.

Optional (Phase 2b, only if on schedule): a consumer fixture app in `tools/regression/fixture/` that installs the packed tarball the way a real user would, imports `functional-ui-kit/fui-button`, `functional-ui-kit/style` and every other entry, type-checks with `tsc`, and renders prop combinations the stories don't cover. Wire it into the harness as another layer.

### Phase 3: Audit fan-out (read-only, about 1.5 h)

Run parallel read-only reviewers (subagents or a workflow). Give each one `audit/CHECKLIST.md`, the harness outputs from the main-vs-main run (axe results, console logs, Tab-walk and action logs), section 8 of this playbook, and one lens:

- L1 Correctness and React: hook dependencies, stale closures, timers, effect cleanup, keys, controlled vs uncontrolled state, event handling, `defaultProps`, React 19 readiness, server-side rendering safety.
- L2 Accessibility: APG pattern conformance per component, keyboard support, focus management, names, roles and states, contrast of the color tokens in both themes, reduced motion, forced colors, target size.
- L3 Public API, types and packaging: `.d.ts` quality, exported types, the `enum` in FuiStepper, prop naming consistency, publint and attw results (run read-only on the packed tarball), CJS-only output, the missing `exports` map, `sideEffects`, the peer dependency range.
- L4 CSS and theming: token consistency between themes, unused CSS, `[class*="fui"]` substring selectors that can match consumer class names, `!important`, z-index, the Google Fonts `@import` in shipped CSS, RTL.
- L5 Security, supply chain and CI: secrets in the repo, the missing lockfile, action pinning, workflow permissions, `npm audit` (read-only), XSS surfaces, end-of-life tooling.
- L6 Performance and bundle: chunk sizes, the react-modal weight in fui-modal, code duplicated across chunks, re-render hot spots, misused memoization, layout reads during render.
- L7 Docs, stories and developer experience: stories vs the real props and states, MDX accuracy, README accuracy, the Storybook config, deprecated imports, the lint and type baselines, the `npm run lint` trap, the lint-staged glob.
- L8 Coverage: states that have no story, and what the harness cannot see.

Also run the `/code-review` skill at `max` effort on each component folder as one more lens.

Every finding uses this shape: id, lens, severity (critical, high, medium, low), file:line, what is wrong, evidence (a quote from the code, a harness log line, or a repro), the best-practice source it breaks, the proposed fix, the predicted user-visible effect of that fix, and a lane guess.

### Phase 4: Verify (about 1 h)

For every finding, a separate verifier tries to prove it wrong. Where possible it writes a repro: a Playwright spec against the static Storybook or a small node script. Mark each finding "confirmed by repro", "confirmed by reading" or "not confirmed". Move "not confirmed" findings to an appendix and keep them out of the fix lists. Merge duplicates. Assign lanes using section 4. Write `audit/REPORT.md` (format in section 6). Commit and push.

### Phase 5: Lane A changes (about 30 min)

The harness, checklist and report are already on Lane A at this point. Only add `src/` changes that pass the Lane A test in section 4. Optionally add a short `CLAUDE.md` with the verified commands, the traps from section 2, and how to run the harness.

Gate: full-mode harness with base `origin/main` and head the Lane A branch reports zero differences in every layer, and `dist/` is byte-identical. Save the harness summary to `audit/LANE_A_PROOF.md`.

### Phase 6: Lane B fixes (until the 8 h mark)

Work in severity order, highest first. For each confirmed finding that qualifies for Lane B:

1. Write the spec first and watch it fail on the current Lane B head.
2. Make the smallest fix that makes it pass. Write down your prediction of what the harness will show.
3. Run the harness with base = parent commit and head = fix commit: layers 1 to 6 on all entries (both themes, both viewports), and the interaction layer on every story whose component imports the changed file directly or indirectly (work it out from the imports; any CSS change means all stories). Compare against your prediction. An unexplained difference means rework or drop.
4. Run the spec: it must pass.
5. Cherry-pick the commit alone onto a scratch branch made from the Lane A head to prove it applies on its own, then delete the scratch branch.
6. Commit with the message format from section 4 and push after every commit.

When done, run full mode once with base = Lane A head and head = Lane B head, and confirm that every difference is covered by one of the fix predictions. Save a per-fix summary under `audit/lane-b/AUDIT-<id>.md` on the Lane B branch.

### Phase 7: Final gates and handoff (last 1.5 h)

1. Run `/code-review max` and `/security-review` on each lane's diff. Fix real problems only within each lane's rules, then rerun the affected gates.
2. Rerun the Lane A gate: full mode, zero differences, byte-identical `dist/`.
3. Write `audit/REVIEWER.md`: exact commands to reproduce every claim, what the harness cannot see, and the reviewer prompt from section 7 with the real PR numbers filled in.
4. Open two draft PRs with the GitHub tools: Lane A into `main`, Lane B into the Lane A branch. Use the descriptions from section 6.
5. Set PROGRESS.md to "done", push, and stop re-arming the check-in.
6. Final message to the owner: both PR links, finding counts by severity and lane, the 5 most important findings in one line each, and anything that needs their decision.
7. Optional: publish a private Artifact page with before and after screenshots for each Lane B fix, and link it from the Lane B PR description.

## 6. Output formats

`audit/REPORT.md`:

- Summary: counts by severity and by lane, the harness self-test result, the Lane A proof, and what was not checked.
- Findings table: id, severity, lens, component, file:line, one-line issue, confidence, lane (A, B with commit sha, or report-only), user-visible effect of the fix, semver impact.
- One section per finding: full evidence, the source it breaks, the proposed fix or the Lane B commit, risk.
- Report-only proposals, with ready-to-apply diffs where possible.
- Blind spots: Firefox and Safari rendering, real screen readers (VoiceOver, NVDA), server-side rendering, React 19 consumers, other bundlers (Create React App, Next.js), touch devices.
- Sources used.

Lane A PR description: what it adds, the zero-regression proof (harness summary and `dist/` hashes), how to rerun the proof, and a link to the report. Say plainly that merging it changes nothing for users.

Lane B PR description: a table of fix commits (id, commit, component, what users will notice, risk grade, semver impact), how to verify each one, and a warning that it must be taken commit by commit, not merged whole.

## 7. Reviewer prompt (copy into REVIEWER.md with the PR numbers filled in)

> You are reviewing an overnight audit of alexyakir/functional-ui-kit. Lane A is PR #<A>; it claims that merging it changes nothing for users. Lane B is PR #<B>; it claims that each commit is one fix whose effects are fully listed. Treat every claim as unverified. Don't push or merge anything.
>
> 1. Read `audit/REPORT.md`, `audit/REVIEWER.md` and `audit/HARNESS_SELF_TEST.md` on the Lane A branch.
> 2. Rebuild the proof yourself: `npm install` at the root, `npm ci` in `tools/regression`, then run the harness self-test and a full-mode run of `origin/main` against the Lane A head. Expect zero differences in every layer and a byte-identical `dist/`.
> 3. Try to break the harness. Plant three canaries the self-test didn't use (for example a one-unit change to a hover color, a changed ARIA attribute on another component, a handler that fires twice) and confirm each is caught. If one slips through, the zero-regression claim is void; say which.
> 4. Read the Lane A diff line by line. Any change outside `tools/regression/`, `audit/`, `CLAUDE.md` and `.gitignore` needs an explanation and proof of a byte-identical `dist/`.
> 5. For each Lane B commit: run the harness between it and its parent and check that only the declared snapshots changed; run its spec on the parent (must fail) and on the commit (must pass); cherry-pick it alone onto the Lane A head; read the code. Give each one a verdict (take, take with changes, drop) with one line of reasoning.
> 6. Spot-check five report-only findings for accuracy.
>
> Report back with the Lane A verdict, a Lane B verdict table, any holes found in the harness, and the decisions the owner needs to make.

## 8. Leads to verify

Starting points from the planning session. Only item 1 was confirmed in a browser. Treat the rest as unverified and run each one through Phase 4 like any other finding.

1. Confirmed in a browser: `FuiButton` with `disabled` still fires `onClick`, both on mouse click and on Enter. It only sets `aria-disabled`, the handler isn't guarded, and `.fui-disabled` only changes the cursor. (`src/components/fui-button/fui-button.tsx:41-42`, `src/css/interactable.css:7-9`)
2. `FuiPopover`'s wrapper class is inverted: it adds `fui-popover-is-open` when the popover is closed. (`fui-popover.tsx:81`)
3. `FuiPopover` memoizes `content` on `props.body` only, so changes to header, footer or className don't render. (`fui-popover.tsx:70-78`)
4. `FuiTooltip`: `timeout` is a plain variable recreated on every render, so `clearTimeout` can't clear a pending timer; `openDelay` only applies when `closeDelay` is set; it opens on hover only, with no focus trigger, no `role="tooltip"` and no `aria-describedby`. (`fui-tooltip.tsx:21-42, 52-56, 70`)
5. Single-value `FuiSelect` marks options with `props.value?.includes(option.value)` on a string, which is a substring match: a value of `apple-pie` also marks `apple`. (`fui-select.tsx:195`)
6. `FuiSelect` reads `ref.current?.offsetWidth` during render for the menu width and in memo dependencies. (`fui-select.tsx:205, 220, 349, 364`)
7. `FuiSelect`'s own key handlers only handle Space (Escape may come from floating-ui's `useDismiss`; check). There are no listbox or option roles, no `aria-expanded` or `aria-selected`, and the `<label>` isn't tied to a control. (`fui-select.tsx:103, 136-141, 195, 234`)
8. `FuiCheckbox`: `role="checkbox"` without `aria-checked`; the focusable element is the wrapper, which has no role; `aria-label` sits on the wrapper; indeterminate isn't exposed as `aria-checked="mixed"`; the `<label>` isn't tied to anything. (`fui-checkbox.tsx:42-46`)
9. `FuiRadio`: two nested `role="radio"` elements plus a native radio input, and the native input never receives `checked`, so its state can disagree with the visual state. (`fui-radio.tsx:52-54`)
10. `FuiSwitch`: `role="checkbox"` with no `aria-checked`, no `tabIndex` and no key handler, so keyboard users can't reach or operate it. (`fui-switch.tsx:30`)
11. `FuiToggle`: every instance uses `name="fui-toggle"` and `id={option.value}`, so two toggles on one page share a radio group and duplicate ids. The option class list can contain the literal strings `false` or `undefined`. (`fui-toggle.tsx:63, 67-68`)
12. `FuiTextInput`: the wrapper class contains the literal string `undefined` when no `className` is passed, and `aria-labelledby` points at a label id that doesn't exist when there is no label. (`fui-text-input.tsx:47, 55`)
13. `Math.random()` ids in FuiRadio, FuiTextInput and FuiStepper differ between server and client render (hydration mismatch). `useId` is the standard fix. (`fui-radio.tsx:20-21`, `fui-text-input.tsx:28-29`, `fui-stepper.tsx:28-29`)
14. `FuiStepper`: the minus and plus buttons are icon-only with no accessible name, and the public API exports a TypeScript `enum`. (`fui-stepper.tsx:10-14, 82-90`)
15. `defaultProps` on function components in 7 places (FuiBadge, FuiButton, FuiModal, FuiNotification, the multi-value FuiSelect, FuiToggle, FuiTooltip). React 18.3 warns about it and React 19 ignores it, so defaults such as the tooltip's `placement: 'top'` would silently change. `FuiModal.defaultProps` also sets `layout`, which isn't a prop.
16. `PopoverTrigger` reads `(children as any).ref`, which React 19 removed. (`popover.tsx:133`)
17. `FuiModal`: the dialog has no accessible name; react-modal's app element is never set, so background content isn't hidden from screen readers; the footer condition `(footerContent && layout === 'horizontal' || !layout)` renders an empty span when no layout is passed. (`fui-modal.tsx:63, 80-91`)
18. The shipped CSS `@import`s two Google Fonts: Atkinson Hyperlegible, which looks unused (`--fui-font-family` only references Lato), and Lato. Every consumer's app makes requests to Google's servers through a render-blocking `@import` chain. (`src/css/main.css:1-2`, `src/css/text-styles.css:2`)
19. A Chromatic project token is committed in `package.json` (the `storybook:deploy` script). Treat it as leaked. Rotating it is the owner's call. Don't copy the value anywhere.
20. No lockfile is committed, so the docs deploy builds with whatever versions resolve that day.
21. The deploy workflow pins `bitovi/github-actions-storybook-to-github-pages` by tag, not by commit SHA.
22. `.storybook/main.js` lists `"@storybook/manager-api, @storybook/theming, @storybook/addon-links"` as a single string, so those addons are probably not registered as intended.
23. `.storybook/docs-container.js` imports from `@storybook/addon-docs/blocks`, which logs a deprecation warning on every docs page.
24. `.storybook/theme.js` looks unused (`manager.js` uses `YourTheme.js`) and sets `colorPrimary` twice.
25. lint-staged only matches `src/**/*.ts`, so the pre-commit hook never lints `.tsx` files, which is nearly all of the code.
26. Tooling at or near end of life: Storybook 7, ESLint 8, Vite 4. Upgrades are report-only.

## Appendix: kickoff prompt used to start the overnight session

```
Overnight zero-regression audit of alexyakir/functional-ui-kit. I'm asleep until morning, so don't ask me anything and don't wait for me. Keep working until everything is done.

First read the full playbook and follow it exactly:
git fetch origin claude/code-audit-overnight-plan-b0omr8
git show origin/claude/code-audit-overnight-plan-b0omr8:audit/OVERNIGHT_AUDIT_PLAYBOOK.md

The rules that matter most:
1. Zero regression means zero. Lane A (your designated branch, created from main) must build a byte-identical dist/ and show zero differences in the regression harness against main. Every change to shipped code goes to Lane B (your branch name plus -fixes), one fix per commit, each with proof. You have my permission to push both branches.
2. Build the regression harness and pass its self-test before changing any code.
3. Never push to main, merge, publish, deploy, run npm run lint (it rewrites files), change dependencies, or touch CI or .storybook.
4. Research current best practices online and use the /code-review and /security-review skills as the playbook says. I opt in to using a workflow (multi-agent orchestration) for the audit and verification phases, up to about 20 agents.
5. Finish with two draft PRs (Lane A into main, Lane B into Lane A), the report files in audit/, and a final message with both PR links and the top findings.
```
