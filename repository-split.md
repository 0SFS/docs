# Repository split

Published on 12 October 2026. This is where the monolith went, how the live site is built, and what is still open. The proposal that led here is [proposals/repository-split.md](proposals/repository-split.md). The checks a person still has to walk are [parity-checklist.md](parity-checklist.md). Open items are also listed in [TODO.md](TODO.md).

## Where things are

| | |
|---|---|
| Monolith | Private [0SFS/monolith](https://github.com/0SFS/monolith). `main` is the app from before the split. Its `gh-pages` branch is the old published history, not the live site. |
| Flight | [0SFS/fly](https://github.com/0SFS/fly). It composes the packages and serves `/fly/` and `/rc/`. The package repositories created for the split (`ci`, `controls`, `exhaust`, `fuel`, `ground-contact`, `logging`, `rc`, and the others) sit beside it. |
| Site | Public [0SFS/0SFS.github.io](https://github.com/0SFS/0SFS.github.io). `main` is the landing page and the scripts that assemble a fly build. GitHub Pages serves `gh-pages`, which is the built site: landing page, flight and phone controller. |
| Live | https://0sfs.github.io/ , https://0sfs.github.io/fly/ and https://0sfs.github.io/rc/ |

The repository that GitHub serves at `https://0sfs.github.io/` has to be named `0SFS.github.io`. Renaming it takes the site down until a repository with that name exists again and Pages is pointed at its `gh-pages` branch.

The first publish of the split was site source `099d192`, flight `1ea3103`, and pages commit `0ca2c3a`. That pages commit is an orphan: it has the built site and none of the monolith's deploy history. The next deploy commits on top of it.

`jpeg-js` in the org folder is an upstream clone. It is not part of the split and was not pushed.

[0SFS/about](https://github.com/0SFS/about) is public. `main` is `3c6f4af`. The empty repository had a ruleset, "Migration publication hold", that blocked every new branch until the check "attribution / Attribution policy" passed. That check cannot run with no branch, so the ruleset was removed and `main` was pushed.

## How to run it

| | Old app | Development | Whole site, as published |
|---|---|---|---|
| Start | `node /Users/felg/gh/0sfs/fly/build/split/reference/serve.mjs` | `cd /Users/felg/gh/0sfs/fly && npm run dev` | `cd fly && npm run build && cd ../0sfs.github.io && npm run build && npm run preview` |
| Flight | http://localhost:4180/fly/ | the URL Vite prints, plus `/fly/` | http://127.0.0.1:4175/fly/ , or https://0sfs.github.io/fly/ |
| Phone | http://localhost:4180/rc/ | Vite's URL plus `/rc/` | http://127.0.0.1:4175/rc/ |
| Landing | http://localhost:4180/ | none: `/` redirects to `/fly/` | http://127.0.0.1:4175/ |

The old-app build is the monolith before the split, kept in fly's gitignored `build/split/reference/`. The preview listens on this computer only, so a phone needs the development server. Restart the development server after `vite.config.ts` or the lockfile changes. A server started earlier keeps the old module graph, which is how the flight once failed with `familyId` of undefined.

Only fly has `node_modules`. The packages are linked with `file:../<repo>`. FOSS Earth and gamepad-tools are linked from the checkouts beside the org folder. `npm install` belongs in fly, with `--ignore-scripts` so a lifecycle script does not write into the FOSS Earth checkout.

## How a change reaches the site

Pushing `main`, in fly or in the website, does not publish. GitHub Pages is set to the `gh-pages` branch of `0SFS/0SFS.github.io`.

```sh
cd fly && npm run build
cd ../0sfs.github.io && npm run build && npm run check && npm run deploy
```

`npm run deploy` replaces `gh-pages` with `dist/`, commits `Deploy the site at <website sha> with the flight at <fly sha>`, and pushes. It refuses to run when the website checkout is dirty, or when the flight was built from a fly commit that is dirty or not on a remote. fly has no deploy script and no longer depends on `gh-pages`.

The website build copies fly's `dist/` without fly's own root `index.html` and without dotfiles, then adds the landing page and `public/`. It fails if the landing page's aircraft cards do not match the flight build's families, order, names, summaries and notes. `public/.gitignore`, which lists linked package files, is removed from fly's `dist/` so it cannot keep those files out of the deploy commit.

## What differs from the monolith

Accepted differences, not regressions. The checklist's KD-08 through KD-12 are these.

- The landing page is static HTML. It loads no bundle, so it has no first-paint shell and no script-download failure of its own. Old `?mode=flight`, `?mode=remote` and `?mode=globe` links are forwarded by a script in the page head.
- fly's development server sends `/` and unknown paths to `/fly/`. The published site, like GitHub Pages, answers an unknown path with 404.
- A flight parameter's help links to the repository that contains the file, for example `osfs.assist.autoTrim` to controls' `src/autoTrim.ts`. The label is the path inside that repository. FOSS Earth parameters are unchanged.
- Bug reports go to `0SFS/fly`, which the build records with its commit.
- Publishing is the website deploy above, and it will not publish a fly build that is not a pushed commit.

Search and VR were deferred before the split and stay deferred.

## Facts the next change has to keep

- Nothing imports fly or the website. A repository's `scripts/` and `benchmarks/` may reach other repositories; those are developer tools. `check-graph` in fly's scratch `build/split/split-tools/` enforces the split layout.
- Two fly tests used to import other repositories by relative path. They now import `@0sfs/f-35b/validation/gearGeometry` and `@0sfs/sound/validation/engineProfileReference`. F-35B declares `@babylonjs/core` as an optional peer because of that export. The sound export is the render used by `check-engine-profile-reference.mjs`, whose command-line comparison still works.
- In development, Vite stamps `?v=` on some imports of a linked file and not on others, so the file can load twice and split a registry. fly's `oneUrlPerSourceFile()` in `vite.config.ts` strips that stamp. A crawl of the dev server found each of 907 modules under one URL. The workaround belongs in FOSS Earth's `localSource()` eventually.
- The landing page's `images/hero.png` is not published. Nothing on the page references it.
- Relative Markdown links that the split broke were rewritten: same repository, a new relative path; another repository, `https://github.com/0SFS/<repo>/blob/main/<path>` (or `tree/main` for a directory). 654 links were repaired. 94 were already broken in the monolith and were left. The repair tools are in fly's gitignored `build/split/split-tools/` (`broken-links.mjs`, `fix-doc-links.mjs`).
- Package versions stay `0.1.0`. The org names commits `YY.MM.DD#N`, which does not fit npm's version field.
- `tsconfig.test.json` is not what `npm test` runs. Under it, the same 40 errors exist as before the landing-page move. Some rc phone tests are excluded there because of other uncommitted work; the note in that file points here.

## Left to do

**A person signs off the checklist.** Headless Chrome has loaded the landing page, `/rc/`, and every aircraft (Cessna 172, Vision Jet G1, G2 and G3, F-35B, and forces on for the Cessna and the F-35B) from the assembled site, and the Cessna and F-35B from the development server. Every flight reached ready with no exceptions. The failed requests that also happen in the old app are Mapterhorn elevation tiles returning 404, and NASA night-lights imagery cancelled when the page closes. That is not the human pass in [parity-checklist.md](parity-checklist.md).

**Rename the local `browser-testing` branch to `main`.** Its contents are on `origin/main`. The local branch is still called `migration/repository-split-20261010`.

**Leave `engine-sound` and `wind-noise` empty until sound can take them as sources.** Giving them code is a redesign of sound, not a file move. Both repositories exist and their `main` branches are pushed.

**Asks for FOSS Earth.** None of these were changed in the FOSS Earth checkout.

- Export `createBrowserPositionSeaLevelPort` from `foss-earth/shell`. Altitude above sea level needs it. Today it is only in `@foss-earth/engine/browser`.
- Replace `fly/src/flight/location/lookup/` with an Earth export. It is a copy of Earth's lookup from engine `1573d79`.
- When `map.focus.refineBasemapFrom` lands, its default is `"focus"`. fly could set the host default to `"camera"`.
- Toolbar order, bug report before settings, is Earth's.
- `localSource()` should serve each linked file under one URL, so fly can drop `oneUrlPerSourceFile()`.
- The settings registry only gives `@foss-earth/<pkg>/…` sources a repository link of their own. Every other source is a base prefix plus the path, so the label cannot name the repository. Support `@<scope>/<pkg>/…` for any scope, or a host resolver.
- Should the generic parameter catalogue (`engines/src/parameterCatalogue.ts`) move to `@foss-earth/ui/settings`?
- Should the location contribution (`fly/src/flight/location/`) move to FOSS Earth?

**Tests already known to fail, and not to be weakened.**

- `createFlightSimApp.test.ts`, "logs what limits 2D imagery…", fails because of uncommitted FOSS Earth work in `engine/src/map/mapDetailLog.ts`.
- `hud/createFlightHudBar.test.ts`, "toggles Controls…", expects "Google 3D Tiles" and gets "3D Tiles", from FOSS Earth's `mapSourceHud.ts`.
- `F135/scripts/build-f135-engine.test.mjs` can time out at 5 seconds on the installed-asset test. The same timeout fails in the old app. It did not fail on the last full run (2140 passed, 1 expected failure, the two failures above).
- The expected failure is the F-35B main-leg clearance defect in `aircraftAnimation.f35b.test.ts`.

**The 94 Markdown links** that were already broken in the monolith are still broken. They name files the monolith never had, including evidence frames. They were not invented by the split.
