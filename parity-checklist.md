# 0SFS split: human feature-parity checklist

A person still has to walk this list. Automated smokes of the landing page, `/rc/` and every aircraft passed on 12 October 2026; they are not this sign-off. How the split is published, and what else is still open, is in [repository-split.md](repository-split.md).

Everything a person must check in the new app (the split repos under
`/Users/felg/gh/0sfs`, run from `fly`) to confirm it does everything the old app did.
The old app is the monolith, now the private repository `0SFS/monolith`, at `8bc8af6`, checked out read-only at
`/Users/felg/gh-archive/0sfs-archive`. It ran on the FOSS Earth monolith
`/Users/felg/gh-archive/foss-earth-archive` (`90e23b3e`). Every item names the old source
file it comes from (paths relative to the monolith root unless they start with
`foss-earth:`). The published site is https://0sfs.github.io/ as well as the local preview in the table below.

Scope: FULL parity (every route, tab, conditional section, setting, preset, saved
state, aircraft variant, control, native/remote/audio/effects/render/diagnostics/
lifecycle behaviour). Search and VR are deferred (see "Out of scope" at the end).

## How to run both apps

| | Old app (reference) | New app, development | New site, as published |
|---|---|---|---|
| Start | `node /Users/felg/gh/0sfs/fly/build/split/reference/serve.mjs` | `cd /Users/felg/gh/0sfs/fly && npm run dev` | `cd /Users/felg/gh/0sfs/fly && npm run build && cd ../0sfs.github.io && npm run build && npm run preview` |
| Flight | http://localhost:4180/fly/ | the URL Vite prints + `/fly/`, e.g. http://localhost:5173/fly/ | http://127.0.0.1:4175/fly/ |
| Phone remote | http://localhost:4180/rc/ (or the LAN address, see RC items) | Vite URL + `/rc/` | http://127.0.0.1:4175/rc/ |
| Landing page | http://localhost:4180/ | none: `/` redirects to `/fly/` (the page is the website's) | http://127.0.0.1:4175/ |

The development server is for the flight and the phone remote. Use the site preview
for section 2, for the legacy `?mode=` links (ROUTE-05..07) and for section 15: it
serves `dist/` the way GitHub Pages will, from the website repo's build of fly's build.
The site preview listens on 127.0.0.1 only, so phone items need the development server.

Tips:

- Use two browser profiles (or a normal and a private window) so the two apps do not
  share `localStorage` if they run on the same origin; they run on different ports, so
  normally they do not.
- Saved settings, saved flights and phone pairing live in the browser's
  `localStorage`/`sessionStorage` per origin: clear site data before the "first visit"
  checks and compare old/new from the same starting state.
- Where an item says **[phone]**, a phone on the same LAN is needed (open the dev server
  on the LAN address; camera permission for the QR scanner needs HTTPS or localhost on
  the phone, see RC items). **[gamepad]** needs a connected gamepad. **[WebGPU]** needs
  a WebGPU-capable browser (Chrome/Edge desktop); **[WebGL]** means forcing
  `?renderer=webgl2`. **[tiles]** needs network map tiles (OSM/elevation) and
  **[google]** a Google Photorealistic 3D Tiles key. **[audio]** needs speakers and a
  user gesture to unlock audio.
- "Earth-owned" means the element is provided by FOSS Earth, which has moved on since
  2026-10-08 (theme, toolbar, log, sky, panels with explicit ports; airport/runway
  location presets now host-owned). Earth-owned items may legitimately differ in
  presentation; check the function, not the pixels. Items marked **(Earth-owned)**.

Legend: `- [ ] ID: what. How to check. (source) [conditions]`

## 1. Routes, entry points and URL parameters

Sources: `src/appRoute.ts`, `src/main.tsx`, `index.html`, `vite.config.ts`,
`scripts/pages-html.mjs`, `public/*`, `src/flight/createFlightSimApp.ts`,
`src/flight/engineTestStand.ts`, `src/flight/diagnostics/*Capture.ts|phoneCameraTrace.ts`,
`src/remote/pairing.ts`, `src/remote/createPhoneControllerApp.tsx`;
`foss-earth:src/settings/appSettings.ts`, `foss-earth:src/settings/catalogue/index.ts`,
`foss-earth:src/engine/babylon/createBabylonRuntime.ts`.

### Routes

- [ ] ROUTE-01: `/` shows the landing ("info") page, not the simulator. Open `/`; the landing page appears with no loading screen or game log. (`src/appRoute.ts` appRouteFrom, `src/main.tsx` bootApp)
- [ ] ROUTE-02: `/fly/` boots the flight: loading screen, game log, then the 3D view and HUD. (`src/main.tsx`, `src/flight/createFlightSimApp.ts`)
- [ ] ROUTE-03: `/rc/` boots the phone controller (unpaired screen when opened without an invitation). (`src/main.tsx`, `src/remote/createPhoneControllerApp.tsx`)
- [ ] ROUTE-04: `/fly` (no trailing slash) reaches the flight (old reference server redirects `/fly` -> `/fly/`; the in-page script also accepts `/fly`). Same for `/rc`. (`index.html` inline script, `build/reference/serve.mjs`)
- [ ] ROUTE-05: legacy `/?mode=flight` is rewritten to `/fly/` (address bar changes, other query parameters and the hash kept, `mode` dropped). (`index.html` inline script `canonical()`, `src/appRoute.ts` canonicalAppLocation)
- [ ] ROUTE-06: legacy `/?mode=remote` is rewritten to `/rc/` (hash kept, so an old QR `/?mode=remote#v=1&peer=…&join=…` still pairs). (`index.html`, `src/appRoute.ts`, `src/remote/pairing.ts` parsePairingUrl)
- [ ] ROUTE-07: `?mode=globe` on any route redirects to https://foss-earth.github.io/ . (`index.html`, `src/main.tsx` canonicalizeLocation, `src/appRoute.ts` FOSS_EARTH_URL)
- [ ] ROUTE-08: an unknown path under the base (e.g. `/foo`) shows the landing page (route "info"); on the static build a missing file gets `404.html` if one exists. (`src/appRoute.ts` pathAfterBase, `build/reference/serve.mjs`)
- [ ] ROUTE-09: the static build has real files `fly/index.html` and `rc/index.html` (GitHub Pages has no SPA fallback). Check `dist/fly/index.html` and `dist/rc/index.html` exist after `npm run build`. (`vite.config.ts` copyIndexToPagesPaths)
- [ ] ROUTE-10: page metadata per route: `/` title "0SFS — Open Source Flight Simulator" with canonical `https://0sfs.github.io/` and WebSite JSON-LD; `/fly/` title "Fly — 0SFS", canonical `/fly/`, no JSON-LD; `/rc/` title "0SFS RC — Phone Controller", `<meta name="robots" content="noindex">`, no canonical. View source of each built page. (`scripts/pages-html.mjs`, `index.html`)
- [ ] ROUTE-11: `theme-color` meta is `#080e14` on `/`, `#04060a` on `/fly/`, `#000` on `/rc/`; `html[data-route]` is `info`/`flight`/`remote`. Inspect `<html>` and the meta tag. (`index.html` inline script)
- [ ] ROUTE-12: web app manifests: `/` and `/fly/` link `manifest.webmanifest` (name "0SFS — Open Source Flight Simulator", start_url `./fly/`, display fullscreen); `/rc/` links `rc.webmanifest` (name "0SFS RC — Phone Controller", start_url and scope `./rc/`) and sets `apple-mobile-web-app-title` to "0SFS RC". Check in DevTools > Application > Manifest. (`index.html`, `public/manifest.webmanifest`, `public/rc.webmanifest`)
- [ ] ROUTE-13: static files served: `favicon.svg`, `icons.svg`, `preview.jpg` (og:image), `robots.txt` (Allow all, Sitemap), `sitemap.xml` (lists `/` and `/fly/` only). Fetch each. (`public/`)
- [ ] ROUTE-14: first paint before the bundle loads: on `/` the static "info-first-paint" block (title, two paragraphs, "Start flying" link to `fly/`); on `/fly/` the boot game log ("0SFS · Open Source Flight Simulator", "Downloading the application…"). Throttle the network in DevTools to see it. (`index.html`)
- [ ] ROUTE-15: bundle download failure shows "The application could not download. Check your connection, then reload and try again." (block the main script in DevTools > Network request blocking). On `/` the message box appears in place of the hidden log. (`index.html` error listener and CSS)
- [ ] ROUTE-16: boot failure after the bundle loaded: on `/fly/` the loading screen shows "The application could not initialize. Check your connection and reload to try again."; on other routes "Failed to initialize the application." (`src/main.tsx` startPage catch)
- [ ] ROUTE-17: `<noscript>` text "0SFS needs JavaScript enabled." on `/` and in the log on `/fly/` (disable JavaScript). (`index.html`)
- [ ] ROUTE-18: the HUD is lifted clear of a browser toolbar overlaying the page bottom (Firefox Android URL bar, Chrome dynamic toolbar) on every route. [phone] (`src/main.tsx` trackViewportInsets, Earth-owned helper)

### URL parameters on /fly/

- [ ] URL-01: `?report` (any value) on `/fly/` shows only the bug-report form, without starting the simulator. (`src/main.tsx` wantsReportOnly/showReportOnly; Earth-owned form)
- [ ] URL-02: `?set.<parameter id>=<value>` sets any registered parameter for this visit, flight ones included, e.g. `?set.osfs.aircraft.id=f-35b` (short form `?aircraft=f-35b`), `?set.osfs.start.latitude=…&set.osfs.start.longitude=…`, `?set.osfs.start.heading=90`. The parameter shows its URL provenance in Settings. (`foss-earth:src/settings/appSettings.ts`, URL prefix `set.` in `@foss-earth/ui/settings`)
- [ ] URL-03: an addressed start (`set.osfs.start.latitude` or `set.osfs.start.longitude` from the URL) overrides "Resume last flight": the flight starts at the addressed position, not the saved flight. (`src/flight/createFlightSimApp.ts` addressedStart)
- [ ] URL-04: `?engineTest=1` opens the engine test stand (the selected aircraft held down at MSP runway 35 threshold, heading 350, standard atmosphere, zero wind, no assists; cold and stopped). `&engineStart=running` starts it with the engine running. Neither is saved; leaving the stand removes both parameters. (`src/flight/engineTestStand.ts`, `createFlightSimApp.ts` ~L1235)
- [ ] URL-05: `?flightPerf=1` enables the stutter trace (Debug tab "Stutter trace" metrics, Copy performance trace, `window.osfsFlightPerformance` in DevTools). Without it Debug shows "Open the simulator with flightPerf=1 in the URL to capture stutters." (`src/flight/diagnostics/flightPerformanceCapture.ts`, `FlightControlPanel.tsx` FlightPerformancePanel)
- [ ] URL-06: `?phoneCameraTrace=1` enables the phone-camera trace (see RC section, camera tuning diagnostics). [phone] (`src/flight/diagnostics/phoneCameraTrace.ts`)
- [ ] URL-07: map URL aliases still work on `/fly/` (Earth-owned): `mapSource`/`tiles` (`google`, `google-3d-tiles`, `fallback`, raster basemap ids), `elevationSource`/`terrainSource`, `key`/`googleKey` (moved to this device's storage and stripped from the address), `renderer=webgpu|webgl2|webgl`, `rasterImagery=atlas|legacy`, retired `terrainQuality=low|balanced|high|auto`. Each changes the map/renderer as in the old app. [tiles] [google] (`foss-earth:src/settings/catalogue/index.ts` fossEarthUrlAliases)
- [ ] URL-08: `?mapDebug=1` and `?terrainCapture=1` enable the globe runtime's map debug and terrain capture in the flight (Earth runtime). (`foss-earth:src/engine/babylon/createBabylonRuntime.ts`)
- [ ] URL-09: the chosen basemap and elevation provider are written back to the address bar (`mapSource=`, `elevationSource=`) so a reload or a shared link opens the same map. (`foss-earth:src/settings/appSettings.ts` LINKED_PARAMETER_IDS, Earth-owned)
- [ ] URL-10: `?dev=1`: in the OLD flight app this had no effect (the FOSS Earth Dev host was only created by the globe app, `foss-earth:src/app/createGlobeApp.ts`; 0SFS registered no dev contributions). If the new app shows a Dev tab with `?dev=1`, that is an addition, not a regression; without `?dev=1` nothing may be hidden that the old app showed (Debug tab stays). (`docs/proposals/tab-inventory.md` "Dev is proposed")
- [ ] URL-11: `/fly/` query parameters survive the `mode=` canonicalisation and the landing "Start flying" link: `/?mapSource=google&key=…` -> "Start flying" goes to `/fly/?mapSource=google&key=…`. (`src/appRoute.ts` appHref/flightHref, `src/info/createInfoPage.ts`)

### URL parameters on /rc/

- [ ] URL-12: `/rc/#v=1&peer=<id>&join=<secret>` pairs immediately and the fragment is removed from the address bar before networking starts. [phone] (`src/remote/createPhoneControllerApp.tsx` consumeInvitation, `src/remote/pairing.ts`)
- [ ] URL-13: invalid invitations are rejected (more than one `mode`, a `mode` other than `remote`, a path other than `/rc`, fragment with extra keys or `v`≠1, bad peer id or secret, URL over 2048 chars, credentials in the URL, plain HTTP on a public host): the unpaired screen shows instead. (`src/remote/pairing.ts` parsePairingUrl)
- [ ] URL-14: changing the hash on an open `/rc/` page (new invitation) re-pairs without a reload. (`createPhoneControllerApp.tsx` onHashChange)
- [ ] URL-15: the pairing URL never copies the desktop's query (map keys etc.) to the phone: QR URL is `<origin><base>rc/#v=1&peer=…&join=…`. (`src/remote/pairing.ts` createPairingUrl)
- [ ] URL-16: on a dev server bound to the LAN (`npm run dev:lan` or `npm run dev -- --host`) the page gets `window.__OSFS_LAN_ORIGIN__` and the QR points at the LAN address; on loopback without it the Remote Control tab says "The phone cannot reach this computer at … run `npm run dev:lan`, or `npm run dev -- --host`." `VITE_PHONE_CONTROLLER_URL` overrides the QR base. [phone] (`vite.config.ts` lanOriginPlugin, `scripts/dev-lan.mjs`, `src/remote/pairing.ts` defaultControllerUrl)

## 2. Landing page (/)

Source: `src/info/createInfoPage.ts`, `src/info/aircraft.ts`, `src/info/info.css`. In the
split the landing is the website repo's static `0sfs.github.io/index.html` and
`landing.css` (moved from `info.css` unchanged); check it on the site preview. In headless
Chrome its desktop and phone screenshots are byte-identical to the old page's
(`0sfs.github.io/build/validation/landing/`); only link targets changed (LAND-07/08).

- [ ] LAND-01: hero: "0SFS Open Source Flight Simulator" heading, lede ("A real-world browser flight simulator built to be explored, extended, and shared…"), "Skip to content" link (visible on keyboard focus), "Start flying" button and "Source on GitHub" link. (`createInfoPage.ts`)
- [ ] LAND-02: "Start flying" goes to `/fly/` keeping the current query and hash (minus `mode`). (`createInfoPage.ts` flightHref)
- [ ] LAND-03: "What it is" section with its five feature bullets (3D world, JSBSim at 120 Hz, cockpit/chase camera, set up next flight, fly with your phone). (`createInfoPage.ts`)
- [ ] LAND-04: "Aircraft" section: three cards with thumbnails `aircraft/thumbnails/cessna-172.png`, `cirrus-vision-jet.png`, `f-35b.png`, labels "Cessna 172 Skyhawk", "Cirrus Vision Jet", "Lockheed Martin F-35B Lightning II", summaries, and the development notes for the Vision Jet (G2+ mapped to G2 runtime…) and F-35B (experimental F-16/Aeromatic-derived model…); closing paragraph about the default aircraft. Images load (no broken image). (`src/info/aircraft.ts`, `createInfoPage.ts`)
- [ ] LAND-05: "Controls" table, caption "Default keyboard": W/S Pitch, A/D Roll, Q/E Yaw/rudder, Shift/Control throttle, F/R flaps, G gear, B brake, P pause, V camera; plus the paragraph about throttle/trim sliders and the G button. (`createInfoPage.ts` CONTROLS)
- [ ] LAND-06: "Phone controller" section text and link "How pairing works, and what to do when it fails" to `docs/phone-controller.md`. (`createInfoPage.ts`)
- [ ] LAND-07: "Documentation" links: development, deploying, jsbsim, sound, foss-earth-relationship, phone-controller (all to `https://github.com/<repo slug>/blob/main/docs/…`). Decide where these point after the split (docs moved to the `docs` repo) and that none 404. (`createInfoPage.ts` blobBase, `__REPOSITORY_SLUG__` from `vite.config.ts`)
- [ ] LAND-08: footer: AGPL-3.0-only LICENSE link, Repository link, NOTICE, THIRD_PARTY_LICENSES.md, ASSET_LICENSES.md links resolve. (`createInfoPage.ts`)
- [ ] LAND-09: the landing page has no game log and no loading screen; the first-paint block is replaced by the real page. (`createInfoPage.ts`, `src/main.tsx` clearBootShell)
- [ ] LAND-10: layout at phone width and desktop matches (styles from `info.css`), dark scheme. (`src/info/info.css`)

## 3. Flight shell: toolbar, window, tabs, log, loading, overlays

Sources: `src/flight/hud/createFlightHudBar.ts`, `src/flight/hud/FlightControlPanel.tsx`,
`src/flight/hud/createFlightControlPanel.tsx`, `src/flight/createFlightSimApp.ts`
(~L2087-2230), `src/loading/createFlightLoadingScreen.ts`,
`src/flight/hud/FlightStatusOverlay.tsx`, `src/fullscreen/*`;
`foss-earth:src/shell/WindowOverlay.tsx`, `foss-earth:src/windowing/**`,
`foss-earth:src/shell/hudBar.ts`, `foss-earth:src/shell/savedWorkspace.ts`.
The toolbar component, the window/tab chrome, the log and the theme are **Earth-owned**
and may look different; the flight's items in them must all be present and do the same.

### Toolbar (HUD bar, bottom of the screen)

- [ ] SHELL-01: toolbar items present, in this order: Pause (Ⅱ / ▶), FPS chip ("FPS —" then "FPS n"), GPU renderer chip (WEBGPU green, otherwise red), ⚙ settings, ⛶ fullscreen, ☰ game log history, 🐞 bug report, position readout chip, then at the right end the World detail rail, download speed + basemap button and its credit link. (`createFlightHudBar.ts`) (Earth-owned chrome)
- [ ] SHELL-02: Pause button toggles pause; label/title switch between "Pause simulation" and "Resume simulation", `aria-pressed` follows; it works while the world is still loading (the flight then starts paused). (`createFlightHudBar.ts` LOADING_LIVE_ATTRIBUTE)
- [ ] SHELL-03: FPS chip click shows/hides the Debug tab. (`createFlightHudBar.ts` onDebugClick)
- [ ] SHELL-04: renderer chip click shows/hides the Renderer tab; chip shows renderer activity. (`createFlightHudBar.ts`, Earth `attachRendererActivity`)
- [ ] SHELL-05: ⚙ opens the right panel (the window). (`createFlightSimApp.ts` onSettingsClick)
- [ ] SHELL-06: ⛶ enters/leaves fullscreen. (`createFlightHudBar.ts` attachFullscreenButton)
- [ ] SHELL-07: ☰ opens/closes the game log history; `aria-pressed` follows. (`createFlightSimApp.ts` onLogToggle)
- [ ] SHELL-08: 🐞 shows/hides the Bug report tab; it is hidden when Interface → Toolbar "bug report" visibility is Off, and its fit priority follows the shared toolbar priority setting when "edit priorities" is on. (`createFlightHudBar.ts` `interface.toolbar.bugReport`)
- [ ] SHELL-09: position chip shows latitude, longitude, altitude (feet by default: host default `interface.position.altitudeUnit=ft`), altitude above ground, and heading `h###°`; over Google 3D Tiles altitude is measured above FOSS Earth's EGM2008 geoid (`interface.position.seaLevelGrid`), not shown as "—", in both the Classic readout and the Compact ALT chip. Click shows/hides the Location tab. (`createFlightHudBar.ts`, `hud/seaLevel.ts`, `registerFlightSettings.ts`)
- [ ] SHELL-10: basemap button (name + download speed) shows/hides the Map tab; the World detail rail beside it adjusts detail and shows the red "Flight minimum" marker while a low spawn holds detail. [tiles] (`createFlightHudBar.ts` createMapSourceHud, `createFlightSimApp.ts` showFlightMinimum)
- [ ] SHELL-11: input-method control in the toolbar (Mouse / Trackpad / Touch as the device allows) with gesture hints ("Right-click drag to orbit", "Two-finger drag to orbit", "Two-finger swipe to orbit", "Mouse wheel to zoom", "Pinch to zoom"); clicking it shows/hides the Controls tab. (`createFlightHudBar.ts` createInputModeHud)
- [ ] SHELL-12: toolbar fits a narrow window: lower-priority items collapse, Pause always stays. Resize to phone width. (`createFlightHudBar.ts` fitHudBar)

### Window and tabs

- [ ] SHELL-13: the window offers 21 tabs: flight tabs Weather, Aircraft, Fuel, Autopilot, Controls, Remote Control, Sound, Exhaust, Engine, G-forces, Logging, Debug (each with its icon: CloudSun, Plane, Fuel, Navigation, Gauge, Smartphone, Volume2, Flame, Fan, Weight, ScrollText, Bug), plus shared Location, Map, Renderer, Sky, Date and time, Interface, Settings, About, Bug report. Open the tab menu ("Open new tab") and count. (`FlightControlPanel.tsx` TAB_DEFINITIONS, `docs/proposals/tab-inventory.md`)
- [ ] SHELL-14: no Scenes, 360 or 360-settings tabs in the flight (globe-only tabs are not registered). (`docs/proposals/tab-inventory.md`)
- [ ] SHELL-15: every flight tab except Controls shows a title row with its icon and label. (`FlightControlPanel.tsx` renderAdditionalTab)
- [ ] SHELL-16: open/close/select tabs, dock, float, resize and reorder windows as in the old app; layout and open tabs persist across reloads (`foss-earth.workspace` in localStorage). (Earth-owned `savedWorkspace.ts`, windowing)
- [ ] SHELL-17: Escape closes the focused popup/menu as before. (Earth-owned `useEscapeToClose.ts`)
- [ ] SHELL-18: closing the Logging tab while recording or with unsaved samples asks "Closing Logging will end the recording. The tab must stay open to log data. Unsaved samples will be discarded."; Cancel keeps the tab open; OK stops and clears. (`FlightControlPanel.tsx` onBeforeCloseTab, `hud/loggingTab.ts`)
- [ ] SHELL-19: flight keys keep working with the window open, except while a text field has focus (typing releases all held keys); a focused slider keeps its arrow keys but W/A/S/D still fly. (`input/flightInputManager.ts` attach, gamepad-tools `controlTakesKey`)
- [ ] SHELL-20: aircraft draft choices in the Aircraft tab survive switching to another tab and back (state held above the tabs). (`FlightControlPanel.tsx` FlightControlPanel)

### Game log, loading screen, status overlay

- [ ] SHELL-21: the game log (top left) shows the boot lines and loading phases: Application, World renderer, Flight simulation, Aircraft, Terrain, each "… ready (n s)", with progress and terrain detail. (`loading/createFlightLoadingScreen.ts` PHASE_LABELS) (log presentation Earth-owned)
- [ ] SHELL-22: loading title "Preparing your flight"; "Copy loading report" action copies a JSON report (or downloads it when the clipboard is unavailable: "Clipboard unavailable. Loading report downloaded."). (`createFlightLoadingScreen.ts`)
- [ ] SHELL-23: aircraft mesh download shows "Aircraft: Loading <LOD label>, x of y MB" with progress; placeholder aircraft reports "Using the flight-ready fallback aircraft"; a failed mesh "Aircraft mesh unavailable; using the flight-ready fallback". Throttle the network to see it. (`createFlightSimApp.ts` onModelState)
- [ ] SHELL-24: load failure: "Flight could not load. Check your connection and try again." / "Flight could not start safely. Check your connection and map access, then retry." with "Reload and try again" and "Copy loading report"; each unfinished phase says "stopped". Block a JSBSim or tile request to test. (`createFlightSimApp.ts`, `createFlightLoadingScreen.ts` fail)
- [ ] SHELL-25: resume message in the log: "Resuming the flight saved <date>[, paused]"; startup pause line "The flight will start paused." / "The flight will start running." when P is pressed during loading. (`createFlightSimApp.ts`, `input/startupPause.ts`)
- [ ] SHELL-26: waiting overlay: when no terrain height is under the aircraft for 0.4 s, "Waiting for terrain height data" card; when World detail is coarser than the Flight minimum, "World detail is too coarse for flight" with the 2^n requirement text; after 5 s adds "Still waiting after several seconds — climb away or reposition if it does not clear." [tiles] (`createFlightSimApp.ts` updateStatusOverlay, `FlightStatusOverlay.tsx`)
- [ ] SHELL-27: fault overlay: a physics fault pauses the flight ("Pausing: physics reported a fault" in the Debug event log) and shows "Simulation paused", the fault message, the failed reasons list and a Resume button; Resume resets physics to the last valid state. (`FlightStatusOverlay.tsx`, `createFlightSimApp.ts` syncSimulationPaused)
- [ ] SHELL-28: flight event log messages also reach the visit's diagnostics trail (info as steps, warnings/errors as trouble) and appear in a bug report. (`createFlightSimApp.ts` flightLog.subscribe)

### Fullscreen offer

- [ ] SHELL-29: on a browser that can go fullscreen, the log offers "Hide the browser bars for a bigger view." with Fullscreen and "Every visit"; in fullscreen it says "Fullscreen. Use ⛶ in the toolbar or Esc to leave."; with Every visit on, the next visit says "Fullscreen starts with your first tap or key press." and enters on the first gesture (not Escape, not a click in the log). A refusal logs "The browser declined fullscreen. Try ⛶ in the toolbar." (`src/fullscreen/fullscreen.ts`)
- [ ] SHELL-30: on iPhone Safari (no fullscreen API) the log says "To hide the browser bars, add 0SFS to your Home Screen (Share → Add to Home Screen)." with "Why?", which opens the controller's fullscreen notice (asked once per device). Not offered when launched from the Home Screen (standalone). [phone: iPhone] (`src/fullscreen/fullscreen.ts`, `FlightFullscreenNotice.tsx`, `remote/PhoneFullscreenPrompt.tsx`)

## 4. Flight lifecycle: start, pause, saved flight, location, reset

Sources: `src/flight/createFlightSimApp.ts`, `src/flight/jsbsim/savedFlight.ts`,
`src/flight/jsbsim/resetFlightLocation.ts`, `src/flight/input/startupPause.ts`,
`src/flight/worldDetail.ts`, `src/flight/jsbsim/fdmProfiles.ts` (runwayPresets);
`foss-earth:src/windowing/react/LocationPanel.tsx`, `foss-earth:src/airports/*`.

- [ ] LIFE-01: first visit (cleared storage) starts at the Start position: 44.977753, -93.265011 (Minneapolis), 1,524 m above ground, heading 300°, the aircraft profile's airspeed and throttle (C172: 120 kt default; host defaults per profile), running (not paused), chase view. (`flightParameters.ts` osfs.start.*, `createFlightSimApp.ts` setHostDefault)
- [ ] LIFE-02: the flight is saved every `osfs.start.saveInterval` s (5 s) of unpaused flight, on pause, after a placement, on fuel changes while paused, on tank jettison, when the page is hidden and on pagehide. Not saved while loading, faulted, on the test stand or with Resume off. Check `osfs.saved-flight.v1` in localStorage updates. (`createFlightSimApp.ts` saveFlight)
- [ ] LIFE-03: reload resumes the saved flight: same position, altitude, heading, controls (throttle, trim, flaps lever, gear), fuel, external tanks, engine state, and paused state if it was paused. (`savedFlight.ts` captureSavedFlight/restoreSavedFlight/applySavedControls)
- [ ] LIFE-04: Aircraft → Start shows "Last saved <time>, at <lat>°N <lon>°W." or "No flight saved. The next session starts at the start position."; "Start a new flight" discards the save and reloads at the start position. (`FlightControlPanel.tsx` describeSavedFlight, `createFlightSimApp.ts` startNewFlight)
- [ ] LIFE-05: turning "Resume last flight" off clears the saved flight immediately and the next load starts at the start position; turning it on saves now. (`createFlightSimApp.ts` osfs.start.resume watcher)
- [ ] LIFE-06: a saved flight for a different aircraft or one that cannot be applied does not block start: log "The saved flight's controls could not be applied" / "The saved flight could not be restored; it starts level where it was". (`createFlightSimApp.ts`)
- [ ] LIFE-07: storage refused (e.g. Safari private with quota 0): log "The flight could not be saved: the browser refused its storage. The next session starts at the start position." once. (`createFlightSimApp.ts` saveFlight)
- [ ] LIFE-08: pause/resume from P key, toolbar button, Aircraft tab "Pause simulation"/"Resume simulation", gamepad Pause binding and the phone; all stay in step. Paused: physics, tyre audio and engine sound hold, the render loop idles (renders on request), engine monitor freezes. Resume does not integrate the paused time. (`createFlightSimApp.ts` syncSimulationPaused, `input/flightInputManager.ts`)
- [ ] LIFE-09: pausing is a safe boundary: pending Aircraft → Ground handling wheel/force/contact/backend choices apply on pause or reset, not mid-flight. (`createFlightSimApp.ts` applyGroundBoundary)
- [ ] LIFE-10: Location tab (Earth-owned panel): typed lat/lon/alt + "Go to location" moves the aircraft; a "Preparing your destination" loading pass runs (terrain phase), the aircraft is hidden meanwhile, then placed 1,000 m clear of the ground at the entered position. The draft (query, coordinates, airport) persists. [tiles] (`createFlightSimApp.ts` teleportToLocation, `foss-earth:src/windowing/react/LocationPanel.tsx`)
- [ ] LIFE-11: a placement resets: wheel spin, ground boundary, external tank debris, G vision, phone camera/link state, auto-trim, autopilot holds; weather is re-applied; engine smoke reset; audio starts a new epoch; flight saved afterwards. (`createFlightSimApp.ts` teleportToLocation)
- [ ] LIFE-12: destination terrain failure: "The destination terrain is not ready. Check your connection and map access, then retry." with "Try again" that retries the same placement. (`createFlightSimApp.ts`)
- [ ] LIFE-13: airport presets (host-owned now; were Earth's `enableAirportPresets`): search an airport, choose Position Departure or Arrival and a Runway. Departure places the aircraft on the runway threshold aligned with the runway, gear down, throttle at the profile's departure value, flaps at the profile's departure setting (C172 0, SF50 0.5, F-35B 0), and pauses. Arrival places it on a 3° approach at the profile's arrival airspeed/throttle/flaps (C172 75 kt, 0.35; SF50 85 kt, 0.35, full flaps; F-35B 160 kt, 0.5), gear down, running. "Runway elevation unavailable." + "Enter coordinates instead" when data is missing. Attribution "Airport data © OpenStreetMap contributors; FreeAirportDB". [tiles] (`fdmProfiles.ts` runwayPresets, `foss-earth:src/airports/geometry.ts` airportSpawn, `createFlightSimApp.ts` location.flightPreset)
- [ ] LIFE-14: place search in the Location tab returns places (OpenStreetMap) and airports; without a provider it says "Place search is unavailable. Enter coordinates below." (Earth-owned; the separate Search tab is deferred, but the old Location tab search existed and must still work or be explicitly accepted as deferred). [tiles] (`foss-earth:src/windowing/react/LocationPanel.tsx`)
- [ ] LIFE-15: a map source or elevation-provider switch (Google <-> 2D basemap, or another elevation provider) re-prepares terrain and re-places the aircraft at its current position; one 2D basemap for another changes imagery only. [tiles] (`createFlightSimApp.ts` mapGround/preparedGround)
- [ ] LIFE-16: low spawn detail hold: with Google 3D Tiles, spawning lower than "Hold detail below" (100 m) with detail coarser than the Flight minimum holds detail at the minimum until 1 s ("Release the hold after") of flight at least 100 m above the ground; the Map → Detail Flight minimum marker is draggable and "Allow coarser terrain for this session" hollows it and releases the hold. [google] (`createFlightSimApp.ts` prepareFlightTerrain/showFlightMinimum, `worldDetail.ts`)
- [ ] LIFE-17: a legacy World detail value (`osfs.world-detail-target`) is imported once into the shared detail controller; failure logs "World detail settings could not be saved; they apply until reload". (`worldDetail.ts` importLegacyWorldDetail)
- [ ] LIFE-18: aircraft change (Aircraft tab Apply with a different aircraft) saves the whole selection then reloads the page; the new aircraft boots. If storage is refused: "Could not save your aircraft choice. The active aircraft is unchanged. Allow browser storage, then apply again." If reload fails: "Your aircraft choice was saved, but the app could not reload. Reload the page to activate it." (`createFlightSimApp.ts` writeAircraftSelection/onAircraftApply)
- [ ] LIFE-19: generation and model detail changes for the same aircraft apply live without reload. (`createFlightSimApp.ts` followAircraftPresentation)
- [ ] LIFE-20: host defaults the flight sets in the shared registry, visible in Settings as host defaults: `sky.model=dome`, `sky.groundLight.mode=rendered`, `map.focus.refineFrom=focus`, `map.focus.point=aircraft`, `map.focus.mode=both`, `interface.position.altitudeUnit=ft`, `osfs.start.throttle`/`osfs.start.airspeed` from the aircraft profile; on a very slow device the slow-device sky defaults; viewpoint surface lighting offered. (`createFlightSimApp.ts`, `registerFlightSettings.ts`, `settings/skyDefaults.ts`)
- [ ] LIFE-21: hidden tab: haptics cancelled, phone told the page is hidden (control can pass to the phone), flight saved. (`createFlightSimApp.ts` onVisibilityChange)
- [ ] LIFE-22: legacy local settings from older versions migrate on first load: `osfs.aircraft`, `flight-sim.aircraft`, `osfs.aircraft-generation`, `osfs.aircraft-lod`, `flight-sim.aircraft-lod`, `osfs.auto-trim`, `osfs.auto-roll-trim`, `osfs.arcade-ground-launches`, `osfs.gamepad-polling-rate`, `osfs.gamepad-response`, `osfs.keyboard-stick`, `osfs.attitude-renderer`, `osfs.flight-terrain-requirement`, `osfs.terrain-detail-anchor`, `osfs.ground-interaction.v1` (+ profiles), `osfs.autopilot.v1`, `osfs.phone-camera-tuning`, `osfs.audio.settings.v1`, `osfs.engineMonitor.v1`. Seed one in localStorage on a clean origin and check the parameter. (`settings/flightMigrations.ts`, `groundInteractionSettings.ts` groundProfilesMigration)

## 5. Aircraft, variants, models and visuals

Sources: `src/flight/aircraft/aircraftIds.ts`, `aircraftCatalog.ts`, `sf50Variants.ts`,
`aircraftSelectionSetting.ts`, `createAircraftModel.ts`, `createPlaceholderAircraft.ts`,
`aircraftAnimation.ts`, `aircraftLights.ts`, `createAircraftLights.ts`,
`externalTankDefinitions.ts`, `createExternalTankVisuals.ts`, `createAircraftEngineVisuals.ts`,
`src/flight/jsbsim/fdmProfiles.ts`, `public/aircraft/**`, `public/jsbsim-data/**`.
In the split: `aircraft` (common), `C172`, `SF-50`, `F-35B` (packs), `IO320`, `FJ33`, `F135` (engines).

### Saved ids, families and variants

- [ ] AC-01: saved aircraft ids accepted by `osfs.aircraft.id`: `cessna-172` (default), `cirrus-vision-jet` (G1, legacy id), `cirrus-vision-jet-g2`, `cirrus-vision-jet-g3`, `f-35b`. Each loads via `?set.osfs.aircraft.id=<id>` and via the Aircraft tab. An unknown id falls back to the C172. (`aircraftIds.ts`, `aircraftCatalog.ts` getAircraftDefinition)
- [ ] AC-02: families in the gallery: Cessna 172 Skyhawk, Cirrus Vision Jet, Lockheed Martin F-35B Lightning II, each with its thumbnail (`aircraft/thumbnails/*.png`), "Selected" / "Select aircraft" caption. (`aircraftCatalog.ts` AIRCRAFT_FAMILIES, `AircraftSelectionPanel.tsx`)
- [ ] AC-03: `osfs.aircraft.generation` choices: `cessna-172`, `g1`, `g2`, `g3`, `g2+`, `f-35b`. Vision Jet "Generation" select offers G1, G2, G3, G2+; G2+ flies the G2 runtime (`cirrus-vision-jet-g2`) but is remembered as G2+. (`aircraftCatalog.ts` SF50_FAMILY_VARIANTS, `sf50Variants.ts`)
- [ ] AC-04: C172 flies JSBSim `c172p` with the IO-320 piston engine (RPM gauge, max 2,700 RPM, 2-blade propeller); SF50 G1/G2/G3 fly `sf50`, `sf50-g2`, `sf50-g3` with the FJ33-5A turbofan (N1 %/N2 gauges, 28,000 ft G1 / 31,000 ft G2-G3 operating ceilings); F-35B flies `F-35B-jsbsim` (coupled F135 plant) or `F-35B-jsbsim-empirical` per Engine → Simulation. Check Engine tab gauges and "Flying: …" line. (`fdmProfiles.ts`, `public/jsbsim-data/aircraft/*`)
- [ ] AC-05: each aircraft's start values: C172 throttle 0.65 / 120 kt (parameter default), SF50 throttle 0.35, F-35B throttle 0.48 / 300 kt. Check Aircraft → Start "Show all parameters" host defaults after loading each. (`fdmProfiles.ts` initialThrottleNorm/initialAirspeedKts, `createFlightSimApp.ts`)
- [ ] AC-06: the Vision Jet and F-35B development notes appear in the aircraft details (?), as on the landing page. (`aircraftCatalog.ts` developmentNote)

### Model detail (LOD), thumbnails, credits

- [ ] AC-07: Model detail choices `osfs.aircraft.lod`: Auto (by chase distance), HD, LOD3, LOD2, LOD1, LOD0 where the aircraft has them, each shown as "<label> — n tris". C172: LOD3 1,016 / LOD2 778 / LOD1 288 / LOD0 130 tris, auto from 0/60/160/320 m. Vision Jet: HD (hilos run textured, 7,294 tris, opt-in only, never chosen by Auto), LOD3 1,654 / LOD2 926 / LOD1 438, auto from 0/65/170 m. F-35B: HD "reconstructed engine" only. (`aircraftCatalog.ts` *_LODS)
- [ ] AC-08: Auto switches LOD as the chase camera zooms out/in; details show "Auto selected <LOD>." and the triangle count. Zoom the chase camera. (`createAircraftModel.ts` refreshAutoLod, `AircraftSelectionPanel.tsx`)
- [ ] AC-09: aircraft details (?) show: pending-changes status, Selected generation, Currently flying, summary, development note, Current model status and Triangles, model credits (artist, note, licence, source link) for the offered meshes, "Gallery image:" credit, and "Applying reloads the simulation to activate the complete aircraft." for another aircraft. (`AircraftSelectionPanel.tsx`)
- [ ] AC-10: Apply button text: "Apply & fly <aircraft>" (other aircraft, reloads), "Apply model settings" (same aircraft, live), "Aircraft active" (disabled, no changes); "Return to active aircraft settings" undoes a draft. (`AircraftSelectionPanel.tsx`)
- [ ] AC-11: a family thumbnail that fails to load shows "Image unavailable". (`AircraftSelectionPanel.tsx` FamilyThumbnail)
- [ ] AC-12: placeholder block aircraft appears only when no mesh exists or the mesh failed; while a chosen mesh loads nothing stands in. (`createFlightSimApp.ts` onModelState, `createPlaceholderAircraft.ts`)

### Animation

- [ ] AC-13: control surfaces move with the controls and FDM positions: ailerons, elevator, rudder (C172, SF50), ruddervators (SF50 V-tail), flaps (C172 0-30°, SF50, F-35B trailing-edge flaps), F-35B differential ailerons. Move the stick in chase view. (`aircraftAnimation.ts` readControlSurfaceState, HingedPart)
- [ ] AC-14: C172 propeller spins with engine RPM (blur at speed); wheels roll at ground speed (not airspeed) when on the ground. (`aircraftAnimation.ts` propeller, RollingWheel)
- [ ] AC-15: retractable gear (SF50, F-35B) animates up and down following the FDM gear position, including the nose and wing bay doors on their hinge lines; the C172's fixed gear does not move. Press G in flight. (`aircraftAnimation.ts` RetractingGear)
- [ ] AC-16: stowed gear is hidden at full retraction unless Debug → Aircraft visuals → "Render stowed landing gear" is on (bay doors visible either way). (`createAircraftModel.ts` renderStowedGear, `osfs.debug.renderStowedGear`)
- [ ] AC-17: F-35B STOVL conversion animates the lift-fan doors and the swivelling 3-bearing nozzle (pitch/yaw) from the FDM's physical actuator positions; nozzle exit area (petals) follows the native nozzle position. Move the VTOL lever. (`aircraftAnimation.ts` conversion doors/nozzle, `createAircraftEngineVisuals.ts`, `engineNozzleRig.ts`)
- [ ] AC-18: animation holds still while the simulation is held (paused, loading, terrain-blocked). (`createFlightSimApp.ts` applyAircraftRig simulationHeld)

### Lights

- [ ] AC-19: each aircraft carries its lights: red/green/white position lights by sector, white strobes (1.2 s period, 0.1 s flash), red beacons (45/min, between strobes; C172 one, F-35B two, SF50 none), landing and taxi lamps (F-35B's on the gear, lit only with gear down). Fly at dusk/night (Date and time tab) to see them. (`aircraftLights.ts`)
- [ ] AC-20: landing/taxi beams light the ground ("Beams on the ground"), and the aircraft is lit from the ground below it; flashes hold while paused or loading. (`createAircraftLights.ts`, `createFlightSimApp.ts`)
- [ ] AC-21: Lights parameters (Aircraft/Lights section): `osfs.lights.navigation` (on), `osfs.lights.antiCollision` (on), `osfs.lights.landing` (on), `osfs.lights.groundLight` (on), and at "all" level `osfs.lights.sizePx` (32 px), `osfs.lights.liftMeters` (0.3 m), `osfs.lights.referenceNits` (1000 cd/m²). Each applies live. NOTE: in the old app the Aircraft tab did not draw an "aircraft/lights" section; these were reachable through the registry (Settings, saved record import, `?set.`). Check the new app exposes them at least as reachably. (`flightParameters.ts` LIGHTS, `FlightControlPanel.tsx` AircraftPanel)

### External tanks (F-35B)

- [ ] AC-22: F-35B carries two external tanks (Right external = JSBSim tank 2, Left external = tank 3; 2,991 lb each, 300 lb dry with pylon) drawn under the wings. (`externalTankDefinitions.ts`, `createExternalTankVisuals.ts`, `public/aircraft/f-35b/ExternalTank_FlightGear.glb`)
- [ ] AC-23: Fuel tab attach/jettison: jettisoning a tank detaches it in JSBSim, drops the tank mesh with the aircraft's ground-relative momentum, saves the flight; the debris disappears after "Released tank lifetime" (15 s) and at most "Released tank limit" (2) are kept (Renderer → Released fuel tanks). "Respawn empty" re-attaches an empty tank. (`createFlightSimApp.ts` onFuelAttachmentChange, `flightParameters.ts` EXTERNAL_TANKS)
- [ ] AC-24: detached tanks are cleared on placement and on Start a new flight; a resumed flight restores attach state. (`createFlightSimApp.ts` resetDetached)
- [ ] AC-25: tank models failing to load logs "External tank models could not be prepared". (`createFlightSimApp.ts`)

### Cockpit view per aircraft

- [ ] AC-26: cockpit view: C172 and SF50 from their cockpit offsets with no own mesh; F-35B from inside AF267's canopy with the airframe mesh drawn around the camera (`cockpitMesh`). (`aircraftCatalog.ts` cockpitOffset/cockpitMesh, `createPlaceholderAircraft.ts`)

## 6. Flight tabs

Each tab section below says which parameters it draws. A section's controls come from
the registry ("main" level parameters are drawn; "all" level ones appear only under
"Show all parameters", Earth-owned control). Check for every section: the controls are
present with the default shown here, changing one applies (live unless noted
`live=false`, which applies on the next start/reload), the value persists across a
reload (`foss-earth.settings.v1`), "Show all parameters" lists the section's "all"
parameters, each parameter's help (?) shows its description, default reason and source
link (`https://github.com/0SFS/0SFS.github.io/blob/main/<source>`: decide the new link
base, see Ambiguities). Source of every parameter: `src/flight/settings/flightParameters.ts`
unless noted; `params.txt` in `build/parity-work/` lists them all with bounds.

### Weather tab (`weather`)

Source: `FlightControlPanel.tsx` WeatherPanel, `createFlightSimApp.ts` applyWeather.

- [ ] TAB-WEATHER-01: heading "Conditions / Custom wind" with the CloudSun icon; "Wind direction" slider 0-359° (shows n°), "Wind speed" slider 0-50 kt (shows n kt), "Clear wind" button sets both to 0. Defaults 0°/0 kt every session (not persisted). (`FlightControlPanel.tsx` WeatherPanel)
- [ ] TAB-WEATHER-02: wind acts on the aircraft: set 270° 30 kt and watch drift/ground speed; it is the direction the wind comes FROM (JSBSim `atmosphere/wind-north-fps = -cos·v`, `wind-east-fps = -sin·v`). Wind is re-applied after a placement. (`createFlightSimApp.ts` applyWeather)
- [ ] TAB-WEATHER-03: no FOSS Earth weather section in this tab (the shared `foss-earth/weather` package was proposed, not present at 8bc8af6). If the new app shows extra Earth weather controls, that is an addition; the custom wind must still drive JSBSim. (`docs/proposals/tab-inventory.md`)

### Aircraft tab (`aircraft`)

Source: `FlightControlPanel.tsx` AircraftPanel, `AircraftSelectionPanel.tsx`,
`GroundInteractionSettingsPanel.tsx`, Earth `MeshInspectorPanel`.

- [ ] TAB-AIRCRAFT-01: metrics row: Airspeed (kt), Altitude (ft), Heading (°), Throttle (%), updating ~10 Hz. (`AircraftPanel`)
- [ ] TAB-AIRCRAFT-02: section "Aircraft" (model): the gallery/generation/model-detail chooser (AC-02..AC-11) covering `osfs.aircraft.id` (default `cessna-172`, live=false), `osfs.aircraft.generation` (default `cessna-172`, live=false), `osfs.aircraft.lod` (default `auto`, live). (`AIRCRAFT_SELECTION_PARAMETER_IDS`)
- [ ] TAB-AIRCRAFT-03: section "Mesh inspector": Earth's mesh tree of the loaded aircraft, and `osfs.aircraft.wireframe` "Show polygon edges" (off) drawing orange triangle edges over the parts selected in the tree; off releases the edge resources. (Earth-owned panel; `createFlightSimApp.ts` createMeshInspector)
- [ ] TAB-AIRCRAFT-04: section "Camera": Cockpit / Chase segmented buttons (pressed state follows V key and phone), plus `osfs.camera.chaseFrame` "Chase camera turns with" (Roll, pitch and heading / Pitch and heading (wings level) / Heading only (horizon level); default attitude). "Show all" adds `osfs.camera.orbitPitchLimits` (-60..81°), `osfs.camera.orbitReturnTime` (0.45 s), `osfs.camera.orbitRestoreYaw` (0°), `osfs.camera.chaseDistance` (14 m, live=false), `osfs.camera.chaseHeight` (2.2 m, live=false), `osfs.camera.chaseZoomLimits` (8..500 m), `osfs.camera.nearClipMeters` (0.05 m), `osfs.camera.fieldOfView` (60.2°), `osfs.camera.gamepadOrbitYawRate` (85.9°/s), `osfs.camera.gamepadOrbitPitchRate` (65.9°/s). Check each takes effect (FOV and near clip live; zoom limits clamp the wheel; return time when orbit recentres). (`flightParameters.ts` CAMERA)
- [ ] TAB-AIRCRAFT-05: "Pause simulation"/"Resume simulation" button and the keymap hint "W/S Pitch, A/D Roll, Q/E Rudder, ⇧/⌃ Throttle". (`AircraftPanel`)
- [ ] TAB-AIRCRAFT-06: section "Flight controls" shown ONLY for an aircraft with a control law (F-35B): hint text about Auto/Fly-by-wire/Manual; `osfs.aircraft.controlLaw` (Auto/Manual/Fly-by-wire, default auto) with the live reading "Active: Fly-by-wire|Manual (aircraft default)"; `osfs.aircraft.fullStickRollRate` (30 °/s); `osfs.aircraft.rollTrimRange` (1). Not shown for C172/SF50. (`AircraftPanel`, `createFlightSimApp.ts` followControlLaw)
- [ ] TAB-AIRCRAFT-07: section "Assists": `osfs.assist.autoFlaps` "Auto flaps" (on), `osfs.assist.autoTrim` "Auto pitch trim" (on), `osfs.assist.autoRollTrim` "Auto roll trim" (on); "Show all" adds `osfs.assist.trimMaxRate` (0.35/s), `trimDeadband` (0.04 rad/s²), `trimFilter` (0.08 s), `trimGain` (1.25/s), `trimAuthority` (20 psf), `trimMinAirspeed` (80 ft/s). The HUD TRIM squares and AUTO flaps button edit the same values. (`flightParameters.ts` ASSISTS)
- [ ] TAB-AIRCRAFT-08: section "Ground handling": `osfs.ground.arcadeLaunches` "Arcade ground launches" (off; bouncier body collision, restitution 1.35 vs 0.25), plus the Ground interaction panel: selection Auto within my choices / Manual (default manual); rows for Wheel response (Off/Instant rolling/Finite inertia), Ground forces (Existing JSBSim/Coupled rigid wheel/Combined-slip tire/Compliant tire and soil), Ground contact (Shared terrain/Per-wheel support/Wheel footprint/Swept wheel shape), Compute backend (Auto (CPU today)/CPU · JavaScript/CPU · WASM/Worker/GPU), Tire audio (Off/Slip cue/Contact cues/Geometry rolling/Detailed), Haptics (Off/Landing cues/Roughness), Wheel visuals (Off/Asset wheel rotation), each with a Lock checkbox; Tire audio volume (0.7) and Haptic strength (0.6) sliders; Haptic devices status (gamepad description; phone "Uses the phone's Haptics switch where its browser supports vibration" or "Not paired"); Wheel experiment override note with keep/discard. Defaults are the Minimal profile (rotation off, jsbsim, shared, auto, tire audio off, haptics off, visuals off). (`GroundInteractionSettingsPanel.tsx`, `groundInteractionSettings.ts`)
- [ ] TAB-AIRCRAFT-09: unavailable ground choices show their reason (e.g. Coupled rigid wheel "…: the coupled solver is not yet integrated with the native bridge", audio "Web Audio is unavailable in this browser"); in Auto selection an unavailable choice falls back with "Auto: …"; boundary choices pending mid-flight show "Applies when the simulation is paused or reset". (`groundInteractionSettings.ts` resolveGroundInteraction)
- [ ] TAB-AIRCRAFT-10: ground parameter ids all present: `osfs.ground.selection`, `osfs.ground.rotation`, `osfs.ground.forceModel`, `osfs.ground.contactModel`, `osfs.ground.backend`, `osfs.ground.tireAudio`, `osfs.ground.haptics`, `osfs.ground.wheelVisuals`, `osfs.ground.tireAudioVolume`, `osfs.ground.hapticStrength`, `osfs.ground.lock.{rotation,forceModel,contactModel,backend,tireAudio,haptics,wheelVisuals}` (all false), `osfs.ground.arcadeLaunches`. (`groundInteractionSettings.ts` GROUND_PARAMETER_IDS)
- [ ] TAB-AIRCRAFT-11: section "Start": saved-flight line and "Start a new flight" (LIFE-04); `osfs.start.resume` "Resume last flight" (on, live=false); "Show all" adds `osfs.start.latitude` (44.977753), `osfs.start.longitude` (-93.265011), `osfs.start.heightAboveGround` (1,524 m), `osfs.start.heading` (300°), `osfs.start.airspeed` (120 kt; host default per aircraft), `osfs.start.throttle` (0.65; host default per aircraft), `osfs.start.saveInterval` (5 s, live). (`flightParameters.ts` START)
- [ ] TAB-AIRCRAFT-12: Aircraft/Lights parameters (AC-21) and the presets section names: Aircraft, Mesh inspector, Camera, Start, Assists, Flight controls, Ground handling, Lights. (`FLIGHT_SECTION_TITLES`)

### Fuel tab (`fuel`)

Source: `src/flight/hud/FuelPanel.tsx`, `src/flight/aircraft/fuelTankLayouts.ts`,
`src/flight/jsbsim/fuelTanks.ts`, `externalFuelTanks.ts`.

- [ ] TAB-FUEL-01: "Total fuel" gauge slider with "<lb> of <capacity> lb · <gal> gal"; lowering takes the same share from every attached tank, raising fills the same share of each tank's empty space; dragging back returns the pilot's own split. (`FuelPanel.tsx` setTotal/distributeFuel)
- [ ] TAB-FUEL-02: top-down aircraft outline per family (C172, Vision Jet, F-35B) with each JSBSim tank drawn where the model puts it, each with its own gauge slider (vertical when the tank is long); legend lists tanks left to right with contents; Tab moves between gauges. (`FuelPanel.tsx`, `fuelTankLayouts.ts`)
- [ ] TAB-FUEL-03: changing fuel writes JSBSim tank contents immediately; while paused it also saves the flight. Fuel burns during flight and the gauges follow. (`createFlightSimApp.ts` onFuelChange)
- [ ] TAB-FUEL-04: F-35B: "Capacity: internal … · attached external …", Jettison / "Respawn empty" per external tank, hint "Jettison releases a tank and its fuel. Respawn adds an empty tank."; detached tanks show "Detached" and a disabled gauge. (`FuelPanel.tsx`)
- [ ] TAB-FUEL-05: layout note, credit and equipment credit with "Source and licence" link (F-35B external tank notice under `aircraft/f-35b/`) open correctly. (`FuelPanel.tsx`, `fuelTankLayouts.ts`)
- [ ] TAB-FUEL-06: an aircraft with no tanks says "The flight model reports no fuel tanks for this aircraft." (`FuelPanel.tsx`)
- [ ] TAB-FUEL-07: fuel transfer: internal tank feed and any JSBSim-defined transfer between tanks continues as the flight model defines (watch tank levels change unevenly in a long F-35B/SF50 flight). Native behaviour; check it matches the old app over the same 2 minutes. (`public/jsbsim-data/aircraft/*`, `fuelTanks.ts`)

### Autopilot tab (`autopilot`)

Source: `src/flight/hud/AutopilotPanel.tsx`, `src/flight/autopilot/*`, `createFlightSimApp.ts`.

- [ ] TAB-AUTOPILOT-01: "Backend" Built in / ArduPilot (`osfs.autopilot.backend`, default `ours`) with hint text. (`AutopilotPanel.tsx`)
- [ ] TAB-AUTOPILOT-02: "ArduPilot" status: Link "Not connected", Identity, Mode, Actuators "None"; status detail (shown as an error when ArduPilot is selected); "Connect SITL" button present but disabled, with the Circle/LOITER note. Selecting ArduPilot blocks engage with the detail as the reason. (`AutopilotPanel.tsx`, `ardupilotStatus.ts`, `controlArbiter.ts` autopilotEngageBlockReason)
- [ ] TAB-AUTOPILOT-03: "Automated axes" checkboxes Roll, Pitch, Yaw / rudder, Throttle, Landing gear, Flaps (`osfs.autopilot.axes.{roll,pitch,yaw,throttle,gear,flaps}`, all on) and "Control everything our AP can own" (disabled when all on). (`AutopilotPanel.tsx`, `flightParameters.ts` autopilotAxis)
- [ ] TAB-AUTOPILOT-04: "Throttle" Behaviour: Hold airspeed (capture on engage) / Hold current lever (`osfs.autopilot.throttleMode`, default airspeed), disabled when the throttle axis is off; hint about takeover. (`AutopilotPanel.tsx`)
- [ ] TAB-AUTOPILOT-05: "Master engage": Master On/Off, Can engage Yes/No, "Engage autopilot"/"Disengage autopilot" button; the HUD AP button next to the gear lever toggles the same; blocked reason or read-only (storage) reason shown in red. On the engine test stand the reason is "The engine test stand uses manual throttle." (`AutopilotPanel.tsx`, `createFlightSimApp.ts` autopilotBlockReason)
- [ ] TAB-AUTOPILOT-06: "Who is flying" shows each axis owner Pilot / Our AP / ArduPilot, live. (`AutopilotPanel.tsx` OWNER_LABEL)
- [ ] TAB-AUTOPILOT-07: engaged AP holds attitude/heading and captures airspeed (or holds the lever), holds gear and flaps; deflecting the stick beyond `osfs.autopilot.stickOverride` (0.05, "all" level) yields that attitude axis while deflected or on the ground; moving the throttle lever takes the throttle axis back. Engaging with Auto flaps on adopts the actual flap position. (`autopilot/controlArbiter.ts`, `ourAutopilot.ts`, `createFlightSimApp.ts` setAutopilotEngaged)
- [ ] TAB-AUTOPILOT-08: AP state is not saved across reload (master off after reload) while the axes/backend/throttle mode persist. (`autopilotSettings.ts`)

### Controls tab (`controls`)

Source: `FlightControlPanel.tsx` (controls branch, KeyboardStickSettingsPanel),
`createFlightHudBar.ts` mountInputMethod, `GamepadBindingsPanel.tsx`,
`input/gamepadToolsAdapter.ts`, `input/keyboardStickResponse.ts`, `keyboardStickSettings.ts`,
`input/gamepadPolling.ts`, `gamepadResponseSetting.ts`, `feedback/haptics.ts`;
`@felipegalind0/gamepad-tools` (binding editor, Felipegalind0-owned);
`foss-earth:src/settings/catalogue/controls.ts` (orbit parameters).

- [ ] TAB-CONTROLS-01: no title row (unlike other flight tabs). (`FlightControlPanel.tsx`)
- [ ] TAB-CONTROLS-02: section "F-35B STOVL" shown ONLY for the F-35B: hint "Use the VTOL lever beside THR: 0% conventional flight; 100% lift fan and downward nozzle. Conversion takes time. Experimental STOVL handling." and `osfs.aircraft.stovlConversion` "STOVL conversion" (0, "all" level), which the HUD VTOL lever edits. (`FlightControlPanel.tsx`, `flightParameters.ts` STOVL)
- [ ] TAB-CONTROLS-03: "Input method" fieldset: Earth's input-method section (Touch on a touchscreen; Mouse/Trackpad choice where there is a pointer; both on a touch laptop) with pointer sensitivity; the toolbar's input-method button opens this tab. (Earth-owned `createInputModeHud`, `FlightControlPanel.tsx` InputMethodSettings)
- [ ] TAB-CONTROLS-04: controller bindings editor (gamepad-tools): built-in profiles "Classic" (previously "Legacy 0sfs compatibility") and "Xbox" (previously "Standard Xbox / PlayStation flight"); custom profiles saved; rebinding by capture (flight keys suspended while capturing); actions listed by category: Flight controls (Pitch, Roll, Rudder, Absolute throttle, Throttle increase / decrease), Camera (Camera left/right, up/down), Trim (Pitch trim, Roll trim), Configuration (Absolute flaps, Flaps increase / decrease, Brake), Commands (Toggle landing gear, Pause simulation, Toggle camera view). If unavailable: "Controller bindings are unavailable in this session." [gamepad] (`createFlightSimApp.ts` gamepadBindings, `gamepadToolsAdapter.ts` FLIGHT_GAMEPAD_ACTIONS)
- [ ] TAB-CONTROLS-05: default keyboard bindings in both profiles: W/S pitch, D/A roll, E/Q rudder, ShiftLeft/ControlLeft and ShiftRight/ControlRight throttle rate, F/R flaps rate, B brake, G gear toggle, P pause, V view. (`gamepadToolsAdapter.ts` keyboardDefaults)
- [ ] TAB-CONTROLS-06: "Classic" gamepad: axis 0 roll, axis 1 pitch (inverted), axis 2 rudder, axis 3 absolute throttle (inverted), button 0 brake. "Xbox": left stick roll/pitch, triggers (buttons 7/6) rudder right/left, Y/B (3/1) throttle up/down, right stick camera yaw/pitch (inverted), A (0) brake, X (2) gear, Start (9) pause, Back/View (8) camera view. Switching profiles takes effect on the next movement without waiting. [gamepad] (`gamepadToolsAdapter.ts` createLegacyFlightProfile/createStandardFlightProfile, `createFlightSimApp.ts` onProfileChange)
- [ ] TAB-CONTROLS-07: section "Gamepad" with hint ("Polling and response apply across binding profiles…"): `osfs.input.gamepadPollingRate` (Every frame | 10-240 Hz), `osfs.input.gamepadResponse` (Smooth/Direct, smooth), `osfs.input.gamepadResponseTime` (0.375 s), `osfs.input.gamepadDeadzoneMode` (Cutoff/Rescaled, cutoff); "Show all": `osfs.input.stickDeadzone` (0.08, live=false), `osfs.input.gamepadSmoothing` (8/s), `osfs.input.takeoverDeadband` (0.12), `osfs.input.throttleRate` (0.5/s; also the keyboard throttle rate). [gamepad] (`flightParameters.ts` GAMEPAD)
- [ ] TAB-CONTROLS-08: gamepad polling: with the flight running, input is read before each flight frame (or at the set rate); while paused or loading, an idle poll still lets controller Pause and binding capture work. [gamepad] (`input/gamepadPolling.ts`, `createFlightSimApp.ts` flightGamepadPolling)
- [ ] TAB-CONTROLS-09: section "Keyboard response": "Response mode" select Direct / Smooth target / Rate ramp / Flight assist (`osfs.input.keyboard.mode`, default smooth) with its description; mode-specific sliders: Smooth → Response time, Return time (0.375 s each); Rate → Time to full (0.6 s), Time to center (0.25 s), Accel after (0.35 s), Accel multiplier (2.5×), Max deflection (100%); Flight assist → Roll rate command (45°/s), Pitch rate command (20°/s), Yaw rate command (20°/s), PID Kp (0.8), Ki (0.15), Kd (0.02), Max deflection (100%); Direct → "No ramp — held keys are full deflection…"; always Expo (0%) and "Reset keyboard stick defaults". (`FlightControlPanel.tsx` KeyboardStickSettingsPanel)
- [ ] TAB-CONTROLS-10: keyboard parameter ids (all level): `osfs.input.keyboard.expo`, `.smoothResponseSec`, `.smoothReturnSec`, `.rateTimeToFull`, `.rateTimeToCenter`, `.rateAccelAfterSec`, `.rateAccelMultiplier`, `.rateMaxDeflection`, `.assistRollRateDeg`, `.assistPitchRateDeg`, `.assistYawRateDeg`, `.assistKp`, `.assistKi`, `.assistKd`, `.assistMaxDeflection`. Note the in-tab sliders have narrower ranges than the parameter bounds (e.g. smooth 0.05-2 s vs 0.05-3 s). (`flightParameters.ts` keyboardNumber, `FlightControlPanel.tsx`)
- [ ] TAB-CONTROLS-11: keyboard stick behaviour per mode: Direct snaps to full; Smooth eases (95% in the response time); Rate ramps and accelerates after the hold time; Flight assist commands body rates and holds near zero rate on release. Expo softens small deflections. Only keyboard bindings are affected (HUD stick, gamepad and phone unchanged). (`input/keyboardStickResponse.ts`)
- [ ] TAB-CONTROLS-12: section "Orbit": Earth's `input.orbit.invertYaw` (off), `input.orbit.invertPitch` (off), `input.orbit.recenterMode` Stay / Spring back (Earth default `hold`) and flight `osfs.input.touchWheelCooldown` (180 ms, "all"). Spring back returns the chase orbit to its resting heading/pitch over the orbit return time when no orbit input is active; inversion applies to mouse, gamepad and phone orbit. (`createFlightSimApp.ts` orbitInvert/recenterOrbit)
- [ ] TAB-CONTROLS-13: section "Feedback" (all level): `osfs.feedback.hapticInterval` (50 ms), `osfs.feedback.hapticMaxDuration` (60 ms), `osfs.feedback.minMagnitude` (0.03), `osfs.feedback.touchdownReference` (300 N·s), `osfs.feedback.slipReference` (1,200 J). With Haptics = Landing cues and a wheel response on, touchdown vibrates the selected gamepad and the paired phone. [gamepad] [phone] (`feedback/haptics.ts`, `createFlightSimApp.ts` haptics)
- [ ] TAB-CONTROLS-14: section titles: "F-35B STOVL", "Gamepad", "Keyboard response", "Feedback", and Earth's "Orbit"; no nested sub-tabs. (`FLIGHT_SECTION_TITLES`)

### Remote Control tab (`remote`)

Source: `src/flight/hud/RemoteControlTab.tsx`, `createPhonePairingPanel.ts`,
`PhoneCameraTuningPanel.tsx`, `src/flight/remote/*`, `src/remote/pairing.ts`,
`src/remote/peerTransport.ts`, `src/remote/iceConfig.ts`. [phone] for most items.

- [ ] TAB-REMOTE-01: order of fieldsets: on a desktop (fine pointer) "Pair a phone", "Who flies", "Phone controller", "Phone camera trackpad", "Use this device as the remote"; on a phone (coarse pointer) "Use this device as the remote" first and no QR is made until asked. (`RemoteControlTab.tsx`)
- [ ] TAB-REMOTE-02: pairing code (PeerJS + QR encoder) loads only when the tab first opens ("Loading…"); failure shows "Pairing could not load. Check the connection and try again." with "Try again". The pairing session outlives the tab (switching tabs keeps the phone). (`RemoteControlTab.tsx`, `createFlightSimApp.ts` loadPhonePairing)
- [ ] TAB-REMOTE-03: "Pair a phone": status line ("No phone paired.", "Preparing connection…", "Scan with your phone camera", "Connecting phone controls…", "Phone paired · Desktop controls", "Phone controls", "Phone and computer blended", "QR expired. Create a new QR.", "Phone disconnected. Create a new QR to pair."), QR canvas, countdown "QR expires in n seconds" (120 s), instructions, "Create new QR" / "Replace phone / New QR", "Copy link" (messages "Link copied. Share only with the phone you want to control this aircraft." / clipboard failures), "Take control" / "Keep control here", "Disconnect". QR unavailable → "QR unavailable. Use Copy link." (`createPhonePairingPanel.ts`, `createPhoneControlSession.ts`, `pairing.ts` INVITATION_TTL_MS)
- [ ] TAB-REMOTE-04: "Connection details" (collapsed): failure code — detail, the connection timeline text, "Copy diagnostics" ("Diagnostics copied. Addresses are masked to their subnet."). "Pairing service disconnected. An established flight connection can keep working." when the PeerJS server drops. (`createPhonePairingPanel.ts`)
- [ ] TAB-REMOTE-05: the QR admits one phone; a second scan of a used/expired QR fails on the phone. Replace phone makes a new invitation. (`createPhoneControlSession.ts`, `pairing.ts`)
- [ ] TAB-REMOTE-06: "Who flies" section: `osfs.remote.sharing` One device at a time / Blend both (exclusive), `osfs.remote.blendPriority` Phone / This computer (phone), `osfs.remote.handover` Automatically / Only when taken / Latched to the phone / Latched to this computer (auto), `osfs.remote.returnIdle` "Rest before returning" (1 s), `osfs.remote.holdLast` "Hold the phone's last command" (2 s), `osfs.remote.hostFlightUi` "Flight UI on this computer while a phone is connected" (off: the flight HUD, toolbar instruments too, hides while a phone is connected; it comes back while the phone has not been heard for 2.5 s, and on a key pressed on the computer until the phone is touched again; not for typing in a text field, keys in an open dialog, a modifier alone or a ⌘/Ctrl shortcut). (`flightParameters.ts` REMOTE_CONTROL, `remote/controlSharing.ts`, `createFlightSimApp.ts`)
- [ ] TAB-REMOTE-07: exclusive sharing: the phone's Fly takes control after the desktop's controls are centred ("Center controls and tap Fly again." otherwise); local flight input takes control back ("Desktop controls · Back to the phone when these controls rest" in auto mode), and control returns to the phone after `returnIdle` s of rest while the tab is visible and the phone is heard; "Only when taken" keeps it until explicitly taken; "Latched to this computer" revokes the phone ("Control latched to the computer"). (`createPhoneControlSession.ts`)
- [ ] TAB-REMOTE-08: blend sharing: both fly at once; where both move a control, the Blend priority device wins and the other fills in; levers (throttle, trims, flaps) continue from where both left them; changing sharing mode while the phone flies revokes ("Sharing changed"). (`remote/controlBlend.ts`, `createFlightSimApp.ts` controlBlend)
- [ ] TAB-REMOTE-09: lost link: the phone's last command is held for `holdLast` s ("Phone signal lost · Holding its last command"), then control returns to the desktop ("Phone signal lost · Desktop controls[ · Back to the phone when it is heard]"). Turn the phone's Wi-Fi off to test. (`createPhoneControlSession.ts`)
- [ ] TAB-REMOTE-10: desktop tab hidden while the phone flies pauses: "Desktop hidden · Simulation paused"; phone releasing control pauses: "Phone released control · Simulation paused"; a pending handoff is cancelled by a settings/pause change ("Flight settings changed. Tap Fly again." / "Control transfer canceled. Tap Fly again."). (`createPhoneControlSession.ts`)
- [ ] TAB-REMOTE-11: protocol mismatch between desktop and phone builds shows the protocol mismatch message on both. (`createPhoneControlSession.ts` PROTOCOL_MISMATCH_MESSAGE, `src/remote/protocol.ts`)
- [ ] TAB-REMOTE-12: "Phone controller" section (sent to the paired phone and changeable from it): `osfs.phone.gridPosition` Under the controls / Over the controls (bottom), `osfs.phone.yawRelease` Return to centre / Keep value (center), `osfs.phone.yawReturnMs` (0 ms), `osfs.phone.haptics` (off), `osfs.phone.attitude` "Attitude indicator on the phone" (on), `osfs.phone.sliders` "Phone sliders" Jump to the finger / Grab the handle (jump), `osfs.phone.twoFingers` "Two fingers on the phone camera pad" Orbit and zoom / Zoom only (both). Changing on either side updates the other. (`flightParameters.ts` PHONE_CONTROLLER, `remote/phoneControllerSettings.ts`)
- [ ] TAB-REMOTE-13: "Phone camera trackpad" section: explanatory hint, "Now: <current tuning>" line, `osfs.camera.phone.send` (On each touch and a 60 Hz timer / Once per touch frame; batch), `osfs.camera.phone.source` (Lose their movement / Recover it from the running total; total), `osfs.camera.phone.present` (As soon as it arrives / Paced on the phone's timeline; playout), `osfs.camera.phone.bufferMs` (12 ms), `osfs.camera.phone.catchUp` (2×, named value "Jump"), `osfs.camera.phone.predictMs` (0 ms). Each applies on the next swipe without re-pairing. (`PhoneCameraTuningPanel.tsx`, `flightParameters.ts` PHONE_CAMERA, `remote/phoneCameraTuning.ts`, `phoneCameraPlayout.ts`)
- [ ] TAB-REMOTE-14: "Use this device as the remote" → "Switch to RC mode" opens a clean `/rc/` on this device (the simulator closes). (`RemoteControlTab.tsx`, `createFlightSimApp.ts` onUseAsRemote)
- [ ] TAB-REMOTE-15: ICE servers: default STUN config; `osfs.ice-servers` in localStorage overrides per device (for TURN). Check a pairing works across a NAT if the old app's did. (`src/remote/iceConfig.ts`)
- [ ] TAB-REMOTE-16: section titles "Who flies", "Phone controller", "Phone camera trackpad". (`FLIGHT_SECTION_TITLES`)

### Sound tab (`sound`)

Source: `src/flight/hud/SoundSettingsPanel.tsx`, `src/flight/audio/*` (createFlightAudio,
audioSettings, audioQuality, createTireAudio, jsbsimAudioAdapter, aircraftAudioProfiles,
engineAcousticDefinitions, dsp/*.wasm, worklet/dspProcessor.js). [audio]

- [ ] TAB-SOUND-01: section "Sound": "Enable sound" (when off) / "Turn sound off" (when on); `osfs.sound.enabled` default on. (`SoundSettingsPanel.tsx`)
- [ ] TAB-SOUND-02: "Quality" select Off / Low / Med / High / Auto (`osfs.sound.quality`, default med); unavailable tiers are disabled with their reason (e.g. "This browser has no AudioWorklet.", "This browser cannot run WebAssembly.", "The selected engine renderer does not implement procedural High."); status "Requested X · Effective Y[ (tire cue only)][ · held while paused or hidden]". (`SoundSettingsPanel.tsx`, `audio/audioQuality.ts`)
- [ ] TAB-SOUND-03: overload downgrade: under sustained overload the tier drops ("Sound dropped to …: p95 … exceeded the … budget twice.") or sheds detail ("Sound shed detail: …"), `osfs.sound.downgradedFrom` records it, and the tab says "Quality was lowered from X after an overload and stays lowered until you re-test." with "Re-test X". (`audio/audioQuality.ts`, `createFlightAudio.ts`)
- [ ] TAB-SOUND-04: sliders: "Master volume · n%" 0-800% (`osfs.sound.masterVolume`, default 2 = 200%), "Engine volume · n%" 0-800% (`osfs.sound.engineVolume`, 0.8), "Sound position" Camera ↔ Cockpit (`osfs.sound.listenerCockpitBlend`, 1 = cockpit), "Airframe wind volume" 0-1 (`osfs.sound.airframeVolume`, 0.6); checkboxes "Mute engine" (`osfs.sound.engineMuted`, off) and "Reduced dynamic range" (`osfs.sound.reducedDynamicRange`, off). (`SoundSettingsPanel.tsx`, `flightParameters.ts`)
- [ ] TAB-SOUND-05: "Show all" adds tier limits (each default = its cap): `osfs.sound.low.partials` (4), `osfs.sound.low.noiseBands` (5), `osfs.sound.med.partials` (12), `osfs.sound.med.noiseBands` (6), `osfs.sound.med.irMs` (20 ms), `osfs.sound.high.partials` (12), `osfs.sound.high.noiseBands` (8), `osfs.sound.high.irMs` (40 ms), each with a live reading "Running n[, shed k stages under load]" for the running tier; and `osfs.sound.downgradedFrom` (none). (`flightParameters.ts` soundLimit, `createFlightSimApp.ts` setReadingSource)
- [ ] TAB-SOUND-06: telemetry and diagnostics lines: aircraft profile label, "Engine sound follows the flight model. Combustion is read from …" (or "This aircraft has no engine sound model; only the tire cue plays."), "Not published by this model, so left silent rather than guessed: …", engine telemetry line, "Timeline: epoch n · resyncs · stale fades · dropped snapshots", stats line. (`SoundSettingsPanel.tsx`)
- [ ] TAB-SOUND-07: audio starts only from a user gesture: with sound enabled and saved, nothing plays until the first click/key ("Tire sound starts after your next click or key press." when locked); the controls' own clicks unlock it synchronously. (`createFlightAudio.ts` onUnlockGesture/sync)
- [ ] TAB-SOUND-08: engine sound per aircraft: C172 IO-320 piston (propeller/RPM driven), SF50 FJ33 turbofan (N1/N2 spectra), F-35B F135 with afterburner roar driven by reheat combustion; High tier adds procedural spectra and directionality; Med adds the cabin impulse response. Compare by ear at idle, cruise, full power in both views. (`audio/aircraftAudioProfiles.ts`, `engineAcousticDefinitions.ts`, `dsp/core.cpp`, `turbofan.h`)
- [ ] TAB-SOUND-09: engine start/stop sounds follow the engine (silent when stopped; spool up on start). (`audio/startupAudio.test.ts` behaviour, `jsbsimAudioAdapter.ts`)
- [ ] TAB-SOUND-10: airframe wind and gear/flap turbulence grow with airspeed, independent of engine volume/mute. (`createFlightAudio.ts`)
- [ ] TAB-SOUND-11: listener position: chase vs cockpit view changes the mix (exterior 1/0), the Sound position slider blends; camera zoom changes distance attenuation; ground reflection off when terrain height is unknown. (`createFlightSimApp.ts` flightAudio.updateView, `cameraZoomAudio.integration.test.ts` behaviour)
- [ ] TAB-SOUND-12: sound holds (fades) while paused, loading or the page is hidden, resumes after; a placement starts a new epoch with a fade. (`createFlightSimApp.ts` setHeld/beginEpoch)
- [ ] TAB-SOUND-13: audio faults: "Sound stopped after an audio processor fault. Switch it off and on to retry."; "Sound could not start: …"; non-finite telemetry: "Some telemetry was rejected as non-finite; sound is holding its last valid state."; an aircraft without telemetry logs "Engine sound telemetry is unavailable; the tire cue still plays". (`createFlightAudio.ts`, `createFlightSimApp.ts`)
- [ ] TAB-SOUND-14: tyre audio (Aircraft → Ground handling Tire audio = Slip cue, or Debug → Enable tire sound with a wheel experiment): touchdown/spin-up squeal fading as the tread matches ground speed; volume by Tire audio volume. (`audio/createTireAudio.ts`, `feedback/wheelCueBus.ts`)
- [ ] TAB-SOUND-15: the audio WASM and worklet load from the built site (`audio-dsp.wasm`, `dspProcessor.js`); `npm run build` verifies the artifact (`verify:audio`). (`scripts/verify-audio-artifact.mjs`, `src/flight/audio/dsp/audio-dsp.provenance.json`)

### Exhaust tab (`exhaust`)

Source: `FlightControlPanel.tsx` exhaust branch, `src/flight/aircraft/createAircraftEngineVisuals.ts`,
`createEngineExhaust.ts`, `createEngineHotSurfaceGlow.ts`, `createEngineSmoke.ts`,
`engineGasOptics.ts`, `engineGasSupport*.ts`, `engineExhaustDepth.ts`,
`engineReactionEmission.ts`, `engineExhaustProfiles.ts`, `engineSmokeProfiles.ts`.
Only the F-35B has an exhaust installation at 8bc8af6; on the C172/SF50 the tab's
settings exist but draw nothing.

- [ ] TAB-EXHAUST-01: fieldset "Gas and luminance" (main): `osfs.exhaust.enabled` "Aircraft exhaust" (on), `contributionView` "Engine light contribution" Combined / Solid emission / Gas emission / Reflected light / Nearby scene light (combined), `sampleCount` (32 samples/pixel), `depthOcclusionEnabled` (on), `depthResolutionScale` (1), `axialFieldSamples` (64), `radialFieldSamples` (32), `maxDistanceMeters` (2,000 m), `intensity` "Exhaust display gain" (1), `gasReferenceNits` (1,000 cd/m²), `dryIntensity` (1), `surfaceReferenceNits` (1,000 cd/m²), `osfs.exhaust.light.enabled` (on), `osfs.exhaust.light.gain` (1), `osfs.exhaust.light.rangeMeters` (12 m). All live. (`flightParameters.ts` EXHAUST)
- [ ] TAB-EXHAUST-02: under a sky model the two reference-nits parameters show the note "Not used while a sky model is on: the sky's exposure shows N cd/m² as white…" and follow the sky's exposure; with the sky model off they apply. (`createFlightSimApp.ts` noteExhaustReferences)
- [ ] TAB-EXHAUST-03: fieldset "Smoke" (main): `osfs.exhaust.smoke.enabled` (on), `maxParticles` (64/engine), `emissionPerSecond` (8/s), `lifetimeSeconds` (2 s), `maxDistanceMeters` (1,000 m), `opacity` (0.025). (`flightParameters.ts` EXHAUST_SMOKE)
- [ ] TAB-EXHAUST-04: fieldset "Afterburner" shown ONLY for an aircraft with an afterburner (F-35B): `osfs.sound.afterburnerVolume` "Afterburner volume" (0.5). Absent for C172/SF50. (`FlightControlPanel.tsx`, `flightParameters.ts` soundVolume AFTERBURNER)
- [ ] TAB-EXHAUST-05: F-35B exhaust gas plume renders from the F135 nozzle exit, brightening with power and changing colour/length with afterburner (reheat fuel burning), occluded by opaque geometry when depth occlusion is on, culled beyond the draw distance; each contribution view isolates its term. [WebGPU and WebGL] (`createEngineExhaust.ts`, `engineGasOptics.ts`, `engineExhaustDepth.ts`)
- [ ] TAB-EXHAUST-06: hot surfaces: nozzle liner and core-facing metal glow by their native metal temperatures (only once the thermal model is initialized/valid). (`createEngineHotSurfaceGlow.ts`, `aircraftCatalog.ts` hotSurfaceRegions)
- [ ] TAB-EXHAUST-07: the exhaust's scene light illuminates nearby aircraft/ground within its range. (`engineReactionEmission.ts`, `osfs.exhaust.light.*`)
- [ ] TAB-EXHAUST-08: smoke particles ("faint aircraft aerosol") trail the engine, reset on placement. (`createEngineSmoke.ts`, `engineSmokeProfiles.ts`, `createFlightSimApp.ts` resetSmoke)
- [ ] TAB-EXHAUST-09: the exhaust optics data is generated and checked at build (`verify:exhaust`, `build:exhaust`). (`scripts/build-exhaust-optics.mjs`)
- [ ] TAB-EXHAUST-10: exhaust visual errors are logged under "aircraft" in the Debug event log rather than breaking the flight. (`createFlightSimApp.ts` engineVisuals onError)

### Engine tab (`engine`)

Source: `FlightControlPanel.tsx` engine branch (EngineDetailsHost, EngineModelStatus),
`src/flight/hud/engineMonitor.ts`, `engineLiveData.ts`, `engineVariables.ts`,
`engineHistory.ts`, `enginePlotRangeControl.ts`, `engineMonitorLayout.ts`,
`engineMonitorModel.ts`, `engineSpool*.ts`, `src/flight/jsbsim/enginePlantSettings.ts`,
`src/flight/engineTestStand.ts`, `createFlightSimApp.ts` (engineMonitor options).

- [ ] TAB-ENGINE-01: the tab hosts the engine monitor's two persisted sections "Settings" and "Live data" (open/closed state remembered in `osfs.engineMonitor.v2`, migrated from v1); "Waiting for the flight model…" before data. (`engineMonitor.ts`, `engineMonitorLayout.ts`)
- [ ] TAB-ENGINE-02: fieldset "Simulation" shown ONLY for an aircraft offering engine models (F-35B): "Flying: <model> · JSBSim <version>" with help (?) showing the model summary; `osfs.engine.model` Coupled engine plant / Empirical tables (plant, live=false); after choosing the other model "<model> flies after the flight reloads. Reload now" (saves the flight and reloads). (`FlightControlPanel.tsx` EngineModelStatus, `fdmProfiles.ts` F35B_ENGINE_MODELS)
- [ ] TAB-ENGINE-03: Simulation plant parameters, visible only while the plant model is chosen (`visibleWhen` plant): `osfs.enginePlant.algorithm` Reduced / Component (reduced), `osfs.enginePlant.iterationCap` (24), `osfs.enginePlant.subdivisionCap` (4), `osfs.enginePlant.tolerance` (1e-6, log scale), `osfs.enginePlant.closureBudgetKiB` (1,024 KiB). Applied live to the native plant. (`flightParameters.ts` ENGINE_SIMULATION, `jsbsim/enginePlantSettings.ts`)
- [ ] TAB-ENGINE-04: section "Engine" (in Settings): `osfs.engineMonitor.fuelFlowUnit` lb/h / gal/h (lb/h), `osfs.engineMonitor.orbFps` (1,000 fps ceiling), `orbTurnsPerSecond` (2 rev/s), `orbMaxPatternStep` (0.45), `orbPixelRatio` (2). (`flightParameters.ts` ENGINE)
- [ ] TAB-ENGINE-05: section "History sampling": `osfs.engineMonitor.historySeconds` (60 s), `historyHz` (5 Hz), `historyMetrics` (32), `historyMemoryKiB` (4,096 KiB), `hiddenHistory` "Continue hidden history capture" (on), `thermalHistory` "Record solid heat balances" (off). (`flightParameters.ts` ENGINE_HISTORY)
- [ ] TAB-ENGINE-06: section "Test stand conditions": `osfs.engineTest.zoomLimits` (1..100 m) and `osfs.engineTest.viewDistanceMeters` (6 m, live=false). (`flightParameters.ts` ENGINE_TEST)
- [ ] TAB-ENGINE-07: "Engine test stand" block with help text per aircraft (F-35B: complete engine reconstruction…; aircraft with test nodes: nozzle assembly isolated; others: readings without a mesh); "Initial engine state" Cold-soaked and stopped / Already-running idle; "Open engine test stand" (or "Reinitialize engine" on the stand, title "Reload the stand with the selected initial state…"); "Return to flight" on the stand. Opening/leaving saves the flight first and navigates with `engineTest`/`engineStart`. (`engineMonitor.ts` build testStand, `createFlightSimApp.ts`)
- [ ] TAB-ENGINE-08: on the test stand: the aircraft (or F-35B full engine mesh only) is held down at MSP runway 35 with zero wind, standard atmosphere, no assists, autopilot blocked, no saved-flight writes, the chase camera framed on the engine at the test distance and zoom limits; the Engine tab opens automatically; Location still moves the stand; V does not toggle the view. (`createFlightSimApp.ts` engineTest branches, `engineTestStand.ts`)
- [ ] TAB-ENGINE-09: "Live data": engine phase line with help, "Shaft indicator" with help; variables discovered from the flight model grouped by family (Rotation/propulsion, thermal, fuel…), each as a value, compact or plot; "Group compatible traces", "Collapse plot"; plot range control; "Find native property" search jumping to a variable or "→ Settings"; "This flight model publishes no engine properties." when none. (`engineLiveData.ts`, `engineVariables.ts`, `enginePlotRangeControl.ts`)
- [ ] TAB-ENGINE-10: history: "Clear history", "Release compact histories", "Download CSV" (`engine-history.csv`, each quantity with unit, gaps empty; disabled with no samples); "History recording help" shows the Hz, seconds, buffer KiB used of budget and slots. Pausing holds history without repeated samples. (`engineLiveData.ts`, `engineHistory.ts`)
- [ ] TAB-ENGINE-11: "Audio-source diagnostics" (collapsible): Mode, Quality, Held, Autoplay lock, Engine as heard (N1/N2/burning/running), Combustion rule, Not published, Timeline. (`engineMonitor.ts` soundRows)
- [ ] TAB-ENGINE-12: "Transitions, newest first" log of engine phase changes with "Clear". (`engineMonitor.ts` logSection)
- [ ] TAB-ENGINE-13: per-aircraft engine content: C172 IO-320 (RPM, piston values), SF50 FJ33 (N1/N2, 53.4% N2 light-off), F-35B F135 plant (N1/N2, nozzle, lift fan, metal temperatures, afterburner state and burned fuel flow; empirical model variant has its own set). Compare the variable lists with the old app for each aircraft. (`engineVariables.ts`, `fdmProfiles.ts`)
- [ ] TAB-ENGINE-14: shaft indicator renderer per Renderer → Instruments → Engine shaft indicators (`osfs.renderer.engineOrbs`), with the note "Drawing with <backend>. …" / "Preparing shaft indicators…". (`createFlightSimApp.ts` onOrbStatus, `engineSpoolRenderer.ts`)
- [ ] TAB-ENGINE-15: section titles "Simulation", "Engine", "History sampling", "Test stand conditions". (`FLIGHT_SECTION_TITLES`)

### G-forces tab (`gforces`)

Source: `FlightControlPanel.tsx` gforces branch, `src/flight/physics/gForce.ts`,
`src/flight/hud/gMeter.ts`, `gVisionOverlay.ts`, `createFlightSimApp.ts` (~L1137-1210).

- [ ] TAB-GFORCES-01: section "Indicator": `osfs.gForce.indicator` "G indicator" (on) shows/hides the HUD G chip; the chip ("Load at the pilot's seat, in g: 1 in level flight. Click to show or hide the G-forces tab.") toggles this tab. (`gMeter.ts`)
- [ ] TAB-GFORCES-02: section "Vision" with its hint: `osfs.gForce.vision` "Dim vision under load" (on), `blackoutRange` (5..9 g), `redoutRange` (-3..-2 g), `onsetTime` (5 s), `recoveryTime` (2 s), `visionStrength` "Vignette strength" (1), `visionViews` "Dim vision in" Every view / Cockpit only (all), `visionLayer` "Vignette covers" Everything / 3D view only (everything). (`flightParameters.ts` G_VISION)
- [ ] TAB-GFORCES-03: pulling sustained high g closes a black vignette from the edges; pushing negative g a red one; follows onset and recovery; "Cockpit only" keeps chase view clear; "3D view only" leaves the HUD, panels and toolbar clear. Turning vision off forgets the load (clear sight on re-enable). The aircraft flies the same. (`gVisionOverlay.ts`, `physics/gForce.ts` stepPilot)
- [ ] TAB-GFORCES-04: section "Passing out" with its hint: `osfs.gForce.passOut` "Pass out" (off), `unconsciousTime` "Mean time out" (11.9 s), `unconsciousSpread` "Time out spread" (4 s). Holding a load at or above the top of the blackout range for about three onset times → the screen goes black in every view, the stick, pedals and brakes are released; the pilot comes to after a random time; the Debug event log records "Pilot passed out" / "Pilot came to"; turning Pass out off wakes the pilot ("Pilot came to: passing out was turned off"). (`createFlightSimApp.ts` pilotOut, `physics/gForce.ts`)
- [ ] TAB-GFORCES-05: G state is cleared on placement. (`createFlightSimApp.ts` clearGVision)

### Logging tab (`logging`)

Source: `src/flight/hud/LoggingPanel.tsx`, `loggingTab.ts`, `src/flight/diagnostics/flightRecorder.ts`.

- [ ] TAB-LOGGING-01: "Flight log" fieldset with hint; metrics Status (Idle/Recording/Stopped), Duration (mm:ss or h:mm:ss), Samples, Marks; buttons Record/Stop, Mark (disabled without samples), Save CSV (disabled without samples), Clear; hint "M marks the current sample while recording, even if another tab is selected." (`LoggingPanel.tsx`)
- [ ] TAB-LOGGING-02: recording samples at 20 Hz of simulation time, up to 30 minutes (ring buffer); off until started; Clear stops and clears. (`flightRecorder.ts` DEFAULT_SAMPLE_HZ/DEFAULT_CAPACITY_SECONDS)
- [ ] TAB-LOGGING-03: M hotkey marks while recording (ignored with Ctrl/Cmd/Alt, in text fields, on repeat, or when not recording). (`loggingTab.ts` bindRecorderMarkHotkey)
- [ ] TAB-LOGGING-04: Save CSV downloads `0sfs-<aircraft>-<build>-<ISO time>.csv` with metadata comment lines (build, built, aircraft, generation, jsbsim identity, `# sample_hz: 20`) and columns lat_deg, lon_deg, alt_msl_ft, alt_agl_ft, pressure_alt_ft, oat_degR, kcas, ktas, mach, vs_fpm, pitch_deg, roll_deg, heading_deg, alpha_deg, beta_deg, gamma_deg, p/q/r_deg_s, nz_g, weight_lb, fuel_lb, n1_pct, n2_pct, rpm, thrust_lbf, fuel_flow_gph, throttle/elevator/aileron/rudder_cmd, pitch/roll/yaw_trim_cmd, flap_cmd, flap_pos, gear_cmd, gear_pos, brake_left/right, wow, stall_warning, stick_pusher, pusher_cmd, plus the leading row columns and marks. Diff the header with an old-app file. (`flightRecorder.ts`, `createFlightSimApp.ts` recordingFileName)
- [ ] TAB-LOGGING-05: closing the tab while recording/with samples asks for confirmation (SHELL-18). (`loggingTab.ts` allowCloseLoggingTab)

### Debug tab (`debug`)

Source: `FlightControlPanel.tsx` DebugPanel/FlightPerformancePanel/EventLog,
`FrameBudgetPanel.tsx`, `src/flight/diagnostics/*`.

- [ ] TAB-DEBUG-01: metrics: Frame rate (fps), Renderer (WEBGPU/WEBGL2…), Map runtime (mode), Camera (first/third). (`DebugPanel`)
- [ ] TAB-DEBUG-02: section "Aircraft visuals": `osfs.debug.renderStowedGear` "Render stowed landing gear" (off; AC-16). (`flightParameters.ts` AIRCRAFT_VISUALS)
- [ ] TAB-DEBUG-03: section "Forces" with the colour legend hint: `osfs.forces.enabled` "Show aircraft forces" (off), `newtonsPerMeter` (20,000 N/m), `maxArrowMeters` (20 m), `labels` (on), `labelRefreshHz` (5 Hz), `controlSurfaces` (on), `newtonMetersPerDegree` (2,000 N·m/°), `arcRadiusMeters` (3 m), `maxArcDegrees` (300°). Arrows: green lift, red drag, purple side, amber propulsion, blue weight, white applied total, yellow net; labels in kN; control-surface moment arcs; F-35B lists lift fan, roll posts, inlet ram drags, stores and pushback (plant) or the four engine force carriers (empirical). Unavailable data logs under "forces". (`diagnostics/createForcesDebugOverlay.ts`, `aircraftForces.ts`, `fdmProfiles.ts` force*)
- [ ] TAB-DEBUG-04: "Collision geometry" checkbox (session-only, off on reload): cyan 5 swept body probes, amber 3 wheel contacts, purple 4 structural contacts, white centre of gravity, legend and hints, visible through the aircraft in both views; "Shows the current C172 physics geometry…" note. (`DebugPanel`, `diagnostics/createCollisionDebugOverlay.ts`)
- [ ] TAB-DEBUG-05: "Wheel spin experiment": Wheel response Off / A · Instant rolling / B · Gradual spin-up (session-only override of Ground handling), "Enable tire sound" (disabled when Off), tire audio status, per-wheel telemetry "n rpm · x m/s slip · contact|air"; with Collision geometry on, tire outlines and rotating spokes. Keep/discard the experiment in Aircraft → Ground handling ("Experiment choices saved to Ground interaction."). (`DebugPanel`, `diagnostics/createWheelSpinDebugOverlay.ts`, `physics/wheelSpin.ts`, `createFlightSimApp.ts` handleGroundAction)
- [ ] TAB-DEBUG-06: runtime status line (last error or message). (`DebugPanel`)
- [ ] TAB-DEBUG-07: "Frame budget" (Earth's panel with flight wording): Measure frame time (`renderer.profiling.enabled`, off; session), Frames summarised (600), Table refresh (500 ms), trace frames; sections flight/camera, flight/terrain, flight/input, flight/physics (+ terrain contact, collision), flight/feedback, flight/view, flight/hud, flight/panels, flight/instruments, attitude indicator; unmeasured note; trace download `0sfs-frame-trace`. `flightPerf=1` turns it on. (`FrameBudgetPanel.tsx`, `registerFlightSettings.ts` frameProfilingParameters, Earth-owned panel)
- [ ] TAB-DEBUG-08: "Stutter trace" (with `?flightPerf=1`): Samples, Frame p95/p99, Worst frame, Physics p95, Terrain p95, Collision p95, Tick p95; slow-frame counts and streamed-tile frames; "Copy performance trace"; `window.osfsFlightPerformance`. (`FlightPerformancePanel`, `diagnostics/flightPerformanceCapture.ts`)
- [ ] TAB-DEBUG-09: "Event log": newest 40 flight log entries with time (s), source, message and detail; "No events yet. Automatic pauses and faults appear here." when empty; "Clear log". Sources seen: sim, terrain, physics, jsbsim, hud, aircraft, forces, pilot. (`EventLog`, `diagnostics/flightLog.ts`)
- [ ] TAB-DEBUG-10: section title "Frame budget", "Forces", "Aircraft visuals". (`FLIGHT_SECTION_TITLES`)

## 7. Shared tabs as the flight shows them (Earth-owned)

Sources: `src/flight/createFlightSimApp.ts` (~L2066-2140: createRendererPanel,
createSkyPanel, createDateTimePanel, createMapSourcePanel, createPresetsSection,
createSavedSettingsSection, createDiagnosticsSection, createAboutPanel,
createBugReportPanel, the Interface and Settings sections), `src/flight/settings/registerFlightSettings.ts`,
`src/flight/settings/skyDefaults.ts`, `src/flight/presets/*.json`, `src/flight/worldDetail.ts`;
`foss-earth:src/shell/{mapSourcePanel,mapCacheSection,rendererPanel,skyPanel,dateTimePanel,aboutPanel,bugReportPanel,diagnosticsSection}.ts`,
`foss-earth:src/shell/settings/{presetsSection,savedSettings,parameterSection}.ts`,
`foss-earth:src/settings/catalogue/{map,renderer,sky,interface,controls,profiling}.ts`,
`foss-earth:src/windowing/react/LocationPanel.tsx`.
Every tab here is an Earth panel; check the function and the flight's additions and host
defaults, not the presentation. A host default shows in the parameter's help as the
default with the flight's reason (LIFE-20). Each Earth tab appends, after its own
sections, a collapsed section for any other section of that tab a parameter is homed in
(`appendHostSections`), which is how the flight's Renderer sections appear.

### Location tab

- [ ] EARTH-LOC-01: fields Lat / Lon / Alt (metres; placeholders show the current position to 8 decimals and altitude) and the ▶ "Go to location" button; invalid input says "Enter latitude from −90 to 90 and longitude from −180 to 180." / "Enter a valid altitude in meters."; success "Location applied." (placement as LIFE-10/LIFE-11). (`foss-earth:src/windowing/react/LocationPanel.tsx`) **(Earth-owned)**
- [ ] EARTH-LOC-02: "location search" box + Search button: "Searching…", "Select a result, then apply.", "No places found.", "Search failed. You can still enter coordinates."; results attribution "Search © OpenStreetMap contributors · FreeAirportDB". Selecting a place fills the coordinates. [tiles] (`LocationPanel.tsx`, `foss-earth:src/search/locationSearch.ts`) **(Earth-owned)**
- [ ] EARTH-LOC-03: selecting an airport result: "Loading runway details…", then "Choose departure or arrival and a runway, then Go." or "No usable runway data found. Enter coordinates manually instead."; Position select Departure / Arrival with the hint "On runway, paused. Resume when ready for takeoff." / "5 NM final · 3° approach."; Runway select; "Choose a runway first." if none; Go is disabled while resolving or with no runway elevation. Behaviour and per-aircraft values: LIFE-13. [tiles] (`LocationPanel.tsx`, `foss-earth:src/airports/*`, `fdmProfiles.ts` runwayPresets) **(Earth-owned panel; presets host-owned now)**

### Map tab

- [ ] EARTH-MAP-01: section "Source" (open): basemap choice USGS Imagery / USGS Imagery Topo (default `usgs-imagery-topo`) / USGS Topo / OpenStreetMap / CARTO Positron / CARTO Dark Matter / OpenTopoMap, plus Google Photorealistic 3D Tiles when a key is set; "Elevation provider" Mapterhorn Terrain (default) / AWS Terrarium ("Terrain is included. No elevation provider is used." under Google). `map.source.googleKey` (live=false) and `map.source.cartoKey` are this device's secrets and never exported. Switching re-places the aircraft (LIFE-15) and is written to the address (URL-09). The toolbar basemap chip toggles this tab (SHELL-10). [tiles] (`foss-earth:src/shell/mapSourcePanel.ts`, `foss-earth:src/settings/catalogue/map.ts`, `createFlightSimApp.ts` RASTER_BASE_MAP_SOURCES/TERRAIN_SOURCES) **(Earth-owned)**
- [ ] EARTH-MAP-02: section "Detail" (open): imagery range and default (default "normal"), Google 3D Tiles range 4..64 px with default "renderer-default", terrain range 2..16 with default 4, `map.detail.linkTerrainToImagery` (on), "all" `map.detail.imageryPath` (atlas, live=false); `map.focus.mode` (host default **both**; Earth `view`), `map.focus.point` (host default **aircraft**; Earth `orbit-target`), `map.focus.radius` (10,000 m), `map.focus.horizonCull` (on), `map.focus.detailBelow.imagery` / `.google` ("detail"), `map.focus.refineFrom` (host default **focus**; Earth `camera`). The same detail controller drives the toolbar World detail rail. (`foss-earth:src/settings/catalogue/map.ts`, `registerFlightSettings.ts`) **(Earth-owned)**
- [ ] EARTH-MAP-03: flight parameters in Map → Detail: `osfs.flight.allowCoarserThisSession` "Allow coarser terrain for this session" (main, off, session-only: off again after reload); "Show all" adds `osfs.flight.minimum` "Flight minimum" (4,096 px, log scale, Google error-target bounds), `osfs.flight.holdBelow` "Hold detail below" (100 m, 0..2,000), `osfs.flight.holdReleaseAfter` "Release the hold after" (1 s, 0..60). The Flight minimum is a draggable red marker on the Google track, hollow while Allow coarser is on (LIFE-16, SHELL-26). [google] (`flightParameters.ts` MAP_DETAIL, `createFlightSimApp.ts` showFlightMinimum)
- [ ] EARTH-MAP-04: section "Automatic adjustment" (collapsed) with its hint: `map.auto.terrainDetail` (off), `map.auto.imageryDetail` (off), `.frameTimeGoal` (display), `.coarsenAbove` (1.2), `.refineBelow` (1.05), `.window` (1,000 ms), `.coarsenWindows` (2), `.refineWindows` (10), `.step` (0.25), `.interval` (5,000 ms). (`foss-earth:src/settings/catalogue/auto.ts`) **(Earth-owned)**
- [ ] EARTH-MAP-05: section "Loading and memory" (collapsed): `map.imagery.gpuBudget` (device-derived default), `map.retryDelay`, `map.google.cacheTiles` (6,000-8,000 by device), `map.google.cacheBytes` (307.2-409.6 MiB by device), `map.imagery.pageTableDepth` (6); footer "Map cache": "Saved tiles" usage list ("Showing the first 40 tiles."), "Clear saved tiles" ("Saved tile cache cleared. Tiles already in the scene remain visible."), "Browser cache" explanation and "Provider requests this visit, browser cache hits included:". [tiles] (`foss-earth:src/settings/catalogue/loading.ts`, `foss-earth:src/shell/mapCacheSection.ts`) **(Earth-owned)**
- [ ] EARTH-MAP-06: sections "Imagery selection" and "Terrain selection" (collapsed) with their explanatory notes and selector calibration parameters. (`foss-earth:src/shell/mapSourcePanel.ts`, `foss-earth:src/settings/catalogue/map.ts`) **(Earth-owned)**

### Renderer tab

- [ ] EARTH-RENDER-01: section "Renderer" (open): Auto-detect / WebGPU / WebGL2 / WebGL with the note "Running on X[, chosen automatically]" or "Running on X. Y was chosen but did not start."; "Changing the renderer reloads the page." (Earth's `applyRendererChoice`; the flight is saved by its pagehide handler and resumes); diagnostics block (Secure context ✓/✗ "WebGPU requires HTTPS", IsSupportedAsync, Error: fallback reason). `renderer.backend` default auto (live=false). The toolbar renderer chip toggles this tab (SHELL-04). (`foss-earth:src/shell/rendererPanel.ts`, `createFlightSimApp.ts` applyRendererChoice) **(Earth-owned)**
- [ ] EARTH-RENDER-02: Earth sections in the flight's Renderer tab, collapsed: "Resolution and frame rate" (`renderer.resolutionScale` 0.25..2 log2, default 1 or 1/devicePixelRatio on the safe WebGPU fallback; `renderer.antialias` on, live=false; `renderer.frameRateCap` Off or 10..240 fps), "Depth range" (`renderer.clipping` From the camera's height / Fixed; `renderer.clipping.fixed` 1..1e7 m when fixed), "Work-saving experiments" (All work-saving experiments plus the panorama experiments, all off; inert in the flight), "Performance debug" (POI sprite size tuner, Compass scale tuner toggles, compass sizes under "Show all"; globe-only panels, inert in the flight). Note which of these the new app shows; the inert ones may legitimately be dropped. (`foss-earth:src/settings/catalogue/renderer.ts`, `interface.ts` PERFORMANCE, `parameterSection.ts` appendHostSections) **(Earth-owned)**
- [ ] EARTH-RENDER-03: flight section "Instruments": `osfs.renderer.attitudeIndicator` Auto ("The GPU when the globe runs on WebGPU, Canvas 2D otherwise.") / WebGPU / Canvas 2D (auto); `osfs.renderer.attitudeView` Camera ("The ball turns with the 3D view…") / Aircraft ("The nose at the centre and the wings level…") (camera); `osfs.renderer.engineOrbs` "Engine shaft indicators" Auto ("Shared WebGPU device, then WebGL2, then WebGL1.") / WebGPU / WebGL2 / WebGL1 (auto). Each shows its live note: "Drawing with WebGPU, on the globe's GPU device." / "Drawing with Canvas 2D." plus a fallback reason; "Drawing with <backend>." / "Preparing shaft indicators…". (`flightParameters.ts` INSTRUMENTS, `hud/attitudeRenderer.ts`, `createFlightSimApp.ts` onOrbStatus) [WebGPU and WebGL]
- [ ] EARTH-RENDER-04: flight section "Released fuel tanks": `osfs.externalTanks.debrisLifetimeSeconds` "Released tank lifetime" (15 s, 0..120), `osfs.externalTanks.maxDetachedTanks` "Released tank limit" (2, 0..2). Behaviour: AC-23. (`flightParameters.ts` EXTERNAL_TANKS)

### Sky tab and Date and time tab

- [ ] EARTH-SKY-01: Sky tab heading line "Now, <time at the place>" or "At the set time, …" with a "Date and time" button that opens/selects the Date and time tab ("Opens the Date and time tab, whose dials set the time the Sun is placed for."). The place is the flight camera's observer, or the aircraft's position before the first frame. (`foss-earth:src/shell/skyPanel.ts`, `createFlightSimApp.ts` skyPlace) **(Earth-owned)**
- [ ] EARTH-SKY-02: section "Atmosphere" (open): `sky.model` (host default **dome**: "a flight is lit by the Sun and the sky; the dome draws on any renderer"), `renderer.ambientFillMultiplier` (1, only with the sky model off), `sky.atmosphere.aerosolOpticalDepth` (0.1); "Show all" groundAlbedo (0.2), dome rows/columns (32/32), steps (16), sunAngle (0.05), densityChange (0.02). Fly at noon and dusk and compare the sky and aircraft lighting. (`foss-earth:src/settings/catalogue/sky.ts`) **(Earth-owned)**
- [ ] EARTH-SKY-03: section "Night" (open): `sky.night.luminance` (0.0002), `sky.moon.mode` (sky), `sky.stars.mode` (catalogue), limiting magnitude (6.5), star size (3 px), daylight altitude range (0..80 km). (`sky.ts`) **(Earth-owned)**
- [ ] EARTH-SKY-04: section "Surface" (collapsed): `sky.surface.lighting` Earth choices plus the flight's extra choice "As at the viewpoint" (default daylight; host default `viewpoint` only on a very slow device: WebGL1 renderer, ≤2 GiB memory or ≤2 threads), `sky.surface.albedoScale` (1.5), `sky.groundLight.mode` (host default **rendered**: "a flight lights the aircraft from the ground below it as it is drawn"; `uniform` on a very slow device), probe size 16 / interval 0.5 s / field of view 120°. (`sky.ts`, `settings/skyDefaults.ts`, `createFlightSimApp.ts` L300-304) **(Earth-owned, flight additions)**
- [ ] EARTH-SKY-05: section "Night lights" (collapsed): `sky.nightLights.mode` (light), date 2025-09-21, colour temperature 3,000 K, satellite NOAA20, floor 0.5, radiance per emittance 431, ground reflectance 0.15, sun below 0°, max zoom 8, window tiles 8, memory 32 MiB. Fly at night over a city. [tiles] (`sky.ts`) **(Earth-owned)**
- [ ] EARTH-SKY-06: section "Exposure" (open): `sky.exposure.mode` (metered), meter range 1..16 EV, adaptation 0.75, `ev100` 9.7 (fixed mode), `renderer.exposureEV` (0). The exhaust's reference nits follow this exposure (TAB-EXHAUST-02). (`sky.ts`) **(Earth-owned)**
- [ ] EARTH-TIME-01: Date and time tab: `sky.time.mode` Now (follow this device's clock) / set time, default a set time: `sky.time.date` (2026-04-20) and `sky.time.utcHours` (22.53, 16:20 solar time over Minneapolis, when the Sun lines up with the shadows in Google's 3D Tiles). Once the flight shows live data (weather, ADS-B), its default becomes Now (`settings/skyDefaults.ts`). Dials for day of the year and solar time with Today / Now, shortcuts Mar equinox / Jun solstice / Sep equinox / Dec solstice and Sunrise / Noon / Sunset / Midnight; lines "Now, … solar time on … at the place shown, which is <UTC>.", "Sunrise … and sunset …: n h mm min of daylight." (or "The Sun stays above/below the horizon all day."), the time-zone paragraph. Setting a time moves the Sun in the flight live. (`foss-earth:src/shell/dateTimePanel.ts`, `sky.ts` TIME_TAB) **(Earth-owned)**

### Interface tab

- [ ] EARTH-IFACE-01: section "Toolbar" shows ONLY the bug-report visibility (`interface.toolbar.bugReport` On / Auto / Off) and "edit priorities" (with the bug report's priority when on); no theme choice and no globe toolbar items (the flight keeps its own toolbar, SHELL-01/08). (`createFlightSimApp.ts` toolbarParameters covers) **(Earth-owned)**
- [ ] EARTH-IFACE-02: section "Position readout": `interface.position.coordinateLabels` None / Words / Icons (none), `interface.position.altitude` ASL / AGL (asl), `interface.position.seaLevelGrid` 1° / 30′ / 15′ (15′), `interface.position.altitudeUnit` Metres / Feet (host default **ft**; Earth m). The toolbar position chip follows each (SHELL-09). (`foss-earth:src/settings/catalogue/interface.ts`, `registerFlightSettings.ts`) **(Earth-owned)**
- [ ] EARTH-IFACE-03: section "Log": `interface.log.lineDuration` "Log line time" (8 s), `interface.log.maxLines` "Log lines kept" (40); the game log follows live. (`interface.ts`, `foss-earth:src/log/gameLogDefaults.ts`) **(Earth-owned)**
- [ ] EARTH-IFACE-04: section "Search": `search.cacheDuration` (24 h), `search.cacheEntries` (64), `search.timeout` (20 s), used by the Location tab search (EARTH-LOC-02). (`interface.ts`, `foss-earth:src/search/searchDefaults.ts`) **(Earth-owned)**

### Settings tab

- [ ] EARTH-SET-01: section order Presets, Saved settings, Diagnostics (collapsed); NO "App files" section (`app.keepFiles` etc. are globe-only: the flight has no service worker). (`createFlightSimApp.ts` settingsSections)
- [ ] EARTH-SET-02: Presets: Earth's "This device", "Save battery and data", "Sharpest", "Smooth motion", then the flight's six in this order: "Low and slow sightseeing", "Fast jet", "Ground: Minimal", "Ground: Landing feedback", "Phone camera: original", "Phone camera: recommended". A line "Matches <preset>" / "Custom"; "Apply…" lists "Applying X changes n values:" with "Apply these changes"; "Applied X: n values copied. Editing any of them makes them Custom."; "Every value already matches."; "Save as preset" (name prompt), Rename, Delete ("Delete X?"), Export, "Import a preset…" ("Imported “X” with n values; it is listed above." / "Not imported: …"). (`foss-earth:src/shell/settings/presetsSection.ts`, `src/flight/presets/*.json`, `registerFlightSettings.ts` FLIGHT_PRESETS) **(Earth-owned section)**
- [ ] EARTH-SET-03: flight preset contents: Low and slow = focus around, radius 20,000 m, imagery -3..1 default 1, Google 2..64 default 2, terrain 1..16 default 1, GPU budget 512, start height 305 m, airspeed 80 kt; Fast jet = focus view, imagery -3..0 default -1, Google 4..64 default 16, terrain 4..32 default 8, Flight minimum 16,384; Ground: Minimal = rotation off, backend jsbsim, shared, auto, tire audio off, haptics off, visuals off; Ground: Landing feedback = rotation inertia, tire audio slip, rest as Minimal; Phone camera: original = send timer, source delta, present arrival, buffer 12, catch-up 2, predict 0; Phone camera: recommended = batch, total, playout, 12, 2, 0. Apply each and diff the values against the JSON. (`src/flight/presets/osfs-*.json`)
- [ ] EARTH-SET-04: Saved settings: "From the URL for this visit" lists `?set.` values with "Keep these values" ("Saved n values."); "Every saved value" lists everything that differs from defaults; "Only values that differ from their defaults are saved, on this device. Keys and secrets are never exported."; a storage error is shown. (`foss-earth:src/shell/settings/savedSettings.ts`) **(Earth-owned)**
- [ ] EARTH-SET-05: Diagnostics section: `diagnostics.trail` "Remember how a visit ended" (on), `diagnostics.trailSteps` "Steps kept" (80); footer "Copy report" and the previous visit's trail ("Reading the last visit's trail…" / "No trail of an earlier visit on this device."). The report includes the flight's state lines: "Aircraft: …; generation …; LOD …; model …; active LOD …; n triangles.", "Flight: loading|paused|running; view …; physics fault …", "JSBSim: <package> <version>; native <commit>; SDK <commit>." (or "JSBSim: build identity unavailable."), and all settings with secrets redacted. (`createFlightSimApp.ts` diagnostics.addState/reportSecrets, `foss-earth:src/shell/diagnosticsSection.ts`) **(Earth-owned section)**

### About and Bug report tabs

- [ ] EARTH-ABOUT-01: About: "This app" with version (`__SOURCE_VERSION__`: git short hash, `-dirty` when uncommitted; "unknown" in the reference build), Served/Built times, Source link (repository slug; none in the reference build), "Site's latest deploy", "Built from" package tree with the JSBSim fork's commit described by `jsbsimBuild`, "Commits, the one it was built from first", warnings "Built with changes that were not committed…" / "No remote branch held this commit…" / duplicate-package notes. (`foss-earth:src/shell/aboutPanel.ts`, `vite.config.ts` builtFrom) **(Earth-owned)**
- [ ] EARTH-BUG-01: Bug report tab (also 🐞 and `?report`): Issue title, "What happened?", "Activity report to attach" (reviewable, secrets redacted), buttons Download report, Open GitHub issue, Copy reviewed report, Refresh report; with no repository slug "The GitHub issue repository is not configured." (the reference build). Check the new app files issues against the intended repository. (`foss-earth:src/shell/bugReportPanel.ts`, `createFlightSimApp.ts` issueReporterFromBuild) **(Earth-owned)**
- [ ] EARTH-HELP-01: every shared parameter's help (?) shows description, default with its reason (the flight's host-default reason where set), bounds, source link (Earth's for Earth parameters, `https://github.com/0SFS/0SFS.github.io/blob/main/…` for flight ones: see Ambiguities) and "applies on reload" for live=false. (`registerFlightSettings.ts` source base, Earth parameter help) **(Earth-owned)**

## 8. HUD and cockpit instruments

Sources: `src/flight/hud/flightHud.ts`, `throttleLever.ts`, `attitudeIndicator.ts`,
`attitudeCanvas.ts`, `attitudeWebGpu.ts`, `attitudeRenderer.ts`, `cameraAttitudeView.ts`,
`evaluationInstruments.ts`, `gMeter.ts`, `engineMonitor.ts`, `engineSummary.ts`,
`engineSpoolRenderer.ts`, `src/flight/createFlightSimApp.ts` (~L1130-1240).
The flight HUD is drawn by 0SFS (not Earth); compare per aircraft: C172, SF50 (G1/G2/G3),
F-35B, in chase and cockpit views, at desktop and phone width.

- [ ] HUD-01: layout: attitude cluster (trim assists, ROLL trim slider, PITCH trim lever, attitude ball, FLAPS lever), tapes column (IAS kt, HDG °, VS fpm, AoA slot, G slot, gear "G" and "AP" buttons), yaw/throttle column (CAS slot, YAW slider, engine summary, THR lever, VTOL lever on the F-35B). (`flightHud.ts` markup)
- [ ] HUD-02: tapes: IAS 3 digits (kt), HDG 3 digits (°), VS signed 4 digits (fpm, "+0000"); values update while flying and hold while paused. (`flightHud.ts`)
- [ ] HUD-03: attitude ball: a KSP-style navball flattened onto a square (nose at the centre, edges 90° from the nose, corners ~165°), pitch rings, heading meridians, sky/ground; surface-mode velocity markers prograde/retrograde, normal/anti-normal, radial out/in from the ground velocity (hidden below 3 ft/s ground speed); markers past 150° pinned to the edge. Fly inverted, straight up and backwards (tail slide) and check it stays right. (`attitudeIndicator.ts`)
- [ ] HUD-04: Attitude indicator view (Renderer → Instruments): Camera (default) turns the ball with the 3D view and moves the aircraft symbol to where the nose points; Aircraft keeps the nose centred and the wings level. Orbit the chase camera to compare. (`cameraAttitudeView.ts`, `osfs.renderer.attitudeView`)
- [ ] HUD-05: attitude renderer: WebGPU on the globe's device when the globe runs WebGPU (Auto), else Canvas 2D; forcing WebGPU on a WebGL globe falls back with "The globe is running on WebGL, so WebGPU is not available to the instrument."; a pipeline failure falls back with "WebGPU could not build the instrument's pipelines here." [WebGPU and WebGL] (`attitudeRenderer.ts`, `osfs.renderer.attitudeIndicator`)
- [ ] HUD-06: attitude pad as a control: dragging on the ball steers pitch/roll (stick deadzone 0.035); arrow keys steer while it has focus; released on pointer up, blur, resize and when the page is hidden; no context menu on long press. (`flightHud.ts` attitude pad)
- [ ] HUD-07: TRIM AUTO buttons for roll and pitch (`osfs.assist.autoRollTrim` / `autoTrim`): titles "Trim assist on — the <axis> wheel cancels leftover moment in real time, including while you fly. Click to turn off." / "Trim assist off — click to cancel leftover <axis> moment on the trim wheel." / "<axis> is owned by Autopilot. TRIM is a wheel assist; disengage AP to use it."; the manual ROLL slider and PITCH lever (with % outputs) are disabled while auto or AP-owned. (`flightHud.ts`)
- [ ] HUD-08: FLAPS lever with % output (actual position when the model reports it, commanded while dragging) and, only where flap automation exists, an AUTO button: "Automatic flaps on. Move the slider to take over manually." / "Automatic flaps off. Click to resume automatic control." / "Flaps are owned by Autopilot. …"; moving the lever turns auto flaps off; keys arrows/Home/End/PageUp/PageDown work on the focused lever. (`flightHud.ts`)
- [ ] HUD-09: gear button "G" (pressed = down): "Landing gear down (G) — click to raise" / "Landing gear up (G) — click to lower" / "Landing gear is held by Autopilot. Click or press G to take it back."; the G key, this button and the phone stay in step; the C172 (fixed gear) ignores it as before. (`flightHud.ts`, `createFlightSimApp.ts` setGearDown)
- [ ] HUD-10: AP button: "Autopilot on — flying the axes selected on the Autopilot tab. Click to disengage." / "Autopilot off — click to engage the package from the Autopilot tab." / the blocked reason ("Autopilot cannot engage."), shown blocked. (`flightHud.ts`, TAB-AUTOPILOT-05)
- [ ] HUD-11: YAW slider with % output; releases to centre on pointer up/blur; Q/E move it. (`flightHud.ts`)
- [ ] HUD-12: THR lever with % (🔥 instead of % and the afterburner accent colour with afterburner on the F-35B; "OFF" when the engine is stopped). Engine off: "Engine off. Press and hold the throttle to start the engine, and keep holding until it runs; drag up as you hold to choose the power it starts to." — the ring fills with start progress; "Starting. Keep holding until the engine runs; letting go now abandons the start."; at idle: "Idle. Press and hold the handle until the ring closes to shut the engine down." (1.5 s hold, 6 px slop); blocked reasons appended. Keys on the focused lever: arrows ±1%, PageUp/PageDown ±10%, holding Space/Enter acts as holding the pointer. The handle is a hollow ring on a bare track (no fill); coloured I-beams show engine speed on the lever's scale: blue N1 and green N2 (turbine, against their maxima) or blue RPM (piston, against rated RPM). Every HUD slider handle is a hollow ring. Shared with the phone (RC-12). (`throttleLever.ts` throttleLeverGauges, `flightControls.css`)
- [ ] HUD-13: VTOL lever (F-35B only) with title "0% conventional flight; 100% vertical lift. n% commanded; m% actual conversion." (TAB-CONTROLS-02). (`flightHud.ts`)
- [ ] HUD-14: AoA readout appears only where the model publishes `aero/alpha-deg` (one decimal); CAS slot shows "STICK PUSHER" (outranks) or "STALL WARNING" from `fcs/stick-pusher` / `fcs/stall-warning` (SF50), announced assertively to screen readers. No warning sound exists at 8bc8af6. (`evaluationInstruments.ts`)
- [ ] HUD-15: G chip (when `osfs.gForce.indicator` is on): load at the pilot's seat in g; click toggles the G-forces tab (TAB-GFORCES-01). (`gMeter.ts`)
- [ ] HUD-16: engine summary: fuel flow (in the Engine tab's unit), turbine N1/N2 and thrust or one piston RPM ring (C172), and the engine phase; the spool rings are drawn as "orbs" per Engine shaft indicators; "Open engine details" opens the Engine tab. (`engineMonitor.ts`, `engineSummary.ts`, `engineSpoolRenderer.ts`)
- [ ] HUD-17: cockpit view per aircraft (AC-26) with the HUD unchanged; chase view framing per Aircraft → Camera. (`aircraftCatalog.ts`, `createFlightSimApp.ts`)
- [ ] HUD-18: HUD fits a phone-width window without overlapping the toolbar (and the browser's bottom bar, ROUTE-18). (`src/styles/flight.css`, `src/flight/hud/*.css`)

## 9. Input: keyboard, gamepad, mouse, trackpad, touch

Sources: `src/flight/input/flightInputManager.ts`, `flightCameraInput.ts`,
`gamepadToolsAdapter.ts`, `gamepadPolling.ts`, `applyFlightControls.ts`, `autoFlaps.ts`,
`autoTrim.ts`, `src/flight/createFlightSimApp.ts` (~L1600-1700: V key, gamepad runtime,
binding editor), `src/flight/hud/createFlightHudBar.ts` (input method);
`@felipegalind0/gamepad-tools` (`file:../Felipegalind0/gamepad-tools`; the reference build
used commit `8427c1a`: `src/ui/editor.ts`, `src/core/persistence.ts`, `src/core/transform.ts`);
`foss-earth:src/settings/catalogue/controls.ts` (input method, sensitivity, orbit).
The Controls tab items (TAB-CONTROLS-*) cover the settings; these items cover behaviour.

### Keyboard

- [ ] INPUT-01: flight keys (TAB-CONTROLS-05) work with physical key codes (layout-independent: `KeyW` on AZERTY is the Z key). Hold W/S/A/D/Q/E for continuous deflection shaped by the keyboard response mode; Shift/Control ramp the throttle at `osfs.input.throttleRate`; F/R ramp flaps; B holds the brakes. (`flightInputManager.ts` KEY_BINDINGS)
- [ ] INPUT-02: P (pause) and G (gear) are latching commands: one toggle per press, auto-repeat ignored. (`flightInputManager.ts`)
- [ ] INPUT-03: V toggles cockpit/chase view (ignored on the engine test stand); M marks while logging (TAB-LOGGING-03). Both are ignored in text fields. (`createFlightSimApp.ts` ~L1625, `hud/loggingTab.ts`)
- [ ] INPUT-04: flight keys are read in the capture phase, so they still fly with focus on a panel button or slider (a focused slider keeps its own arrow keys; HUD controls declare the keys they take, `data-takes-keys`); typing in a text field releases every held key (SHELL-19). (`flightInputManager.ts`)
- [ ] INPUT-05: while the binding editor captures an input ("Listening for …"), flight keys and controller commands do not act. (`flightInputManager.ts` isBindingCaptureActive, gamepad-tools capture)
- [ ] INPUT-06: losing window focus (blur) or hiding the page releases held keys, the HUD stick and transient controls (no stuck deflection after Alt-Tab). (`flightInputManager.ts`, `flightHud.ts`)

### Gamepad (gamepad-tools)

- [ ] INPUT-07: controller selection: "Controller" select with "Automatic selection" and each connected pad ("Slot n", "Unknown controller" when unnamed); "No controller selected" / "Connect a controller to select numeric axes or buttons." with none. Unplug and re-plug a pad mid-flight: input resumes. [gamepad] (gamepad-tools `src/ui/editor.ts`, `src/browser/source.ts`)
- [ ] INPUT-08: controller preview ("Loading controller preview...") highlights pressed buttons and moved sticks with standard names (Left stick left/right, Right stick up/down, D-pad up…, View, Menu, stick presses); "Input details" shows raw axes/buttons. [gamepad] (gamepad-tools `src/ui/editor.ts`, `src/viewer/index.ts`)
- [ ] INPUT-09: binding: "Select an input, then choose the action to assign."; "Bind" → "Listening for <action>…" → "Use candidate" / Cancel ("Binding cancelled."); actions can be left "Unassigned"; unavailable actions say "This action is not currently available." Keyboard rows show "Keyboard: <key>". [gamepad] (gamepad-tools editor, `gamepadToolsAdapter.ts` FLIGHT_GAMEPAD_ACTIONS)
- [ ] INPUT-10: "Profile actions": Duplicate, Edit name ("Profile name"), Export profile (JSON download), Import profile ("Import binding profile": "Profile imported.", "The selected file is not valid JSON.", "The selected profile is too large.", a name clash offers "Keep both" / "Replace existing"), Delete for copies only ("Built-in profiles stay available. Your copies can be deleted."); errors "This profile could not be saved/deleted.", "Saved profiles could not be loaded.". [gamepad] (gamepad-tools editor)
- [ ] INPUT-11: profiles persist in `localStorage` under `gamepad-tools:profiles:0sfs:index` and `gamepad-tools:profiles:0sfs:profile:<id>`; the selected profile survives a reload; built-ins are rebuilt with the current Stick deadzone (`osfs.input.stickDeadzone`, live=false) and the selected pad's slot. Analog response/polling come from the osfs parameters, not gamepad-tools' own response key. [gamepad] (gamepad-tools `src/core/persistence.ts`, `gamepadToolsAdapter.ts` namespace "0sfs", `createFlightSimApp.ts`)
- [ ] INPUT-12: gamepad camera: Camera yaw/pitch actions orbit the chase camera at `osfs.camera.gamepadOrbitYawRate` (~86°/s) / `PitchRate` (~66°/s), inverted per Controls → Orbit, and count as orbit activity for Spring back; "Toggle camera view" switches views on press. [gamepad] (`createFlightSimApp.ts` onCameraOrbit, `gamepadToolsAdapter.ts`)
- [ ] INPUT-13: gamepad arming and takeover: a bound axis that is off-centre when the controller connects or the profile changes (e.g. an absolute throttle lever at 70%) does nothing until it moves more than `osfs.input.takeoverDeadband` (0.12) from where it rested, so the throttle does not jump; controller input beyond the deadband also counts as local input that takes control back from the phone (TAB-REMOTE-07). [gamepad] (`flightInputManager.ts` armedBindingSources, hasActiveFlightInput)
- [ ] INPUT-14: calibration: gamepad-tools at `8427c1a` has no calibration screen; axes are shaped only by deadzone (Cutoff / Rescaled), response curve and inversion in the profile's transforms. If the new app adds calibration it is an addition. [gamepad] (gamepad-tools `src/core/transform.ts`)

### Mouse, trackpad and touch (camera)

- [ ] INPUT-15: Mouse mode: right-button drag orbits the chase camera (no context menu on the canvas), wheel zooms (chase zoom limits). (`flightCameraInput.ts`)
- [ ] INPUT-16: Trackpad mode: two-finger swipe orbits, pinch (ctrl+wheel; Safari's GestureEvents instead, never both) zooms. (`flightCameraInput.ts`)
- [ ] INPUT-17: Touch (any mode): two fingers orbit and pinch-zoom; finger right gives positive yaw (same direction as a mouse drag); late synthesized wheel pans after a touch are ignored for `osfs.input.touchWheelCooldown` (180 ms; Android Firefox). [phone] (`flightCameraInput.ts`)
- [ ] INPUT-18: sensitivity per input method and movement from Earth's `input.sensitivity.<device>.<movement>` (default 1, "Show all" in Input method); changing it changes orbit/zoom speed live. (`foss-earth:src/settings/catalogue/controls.ts`, `flightCameraInput.ts`) **(Earth-owned parameters)**
- [ ] INPUT-19: input-method toolbar control (SHELL-11): Mouse / Trackpad (default `input.mode` trackpad) / Touch as the device allows, with the gesture hints; choosing one switches the canvas gestures at once and persists. **(Earth-owned control)** (`createFlightHudBar.ts`, Earth `createInputModeHud`)
- [ ] INPUT-20: on-screen HUD controls (stick pad, trims, flaps, yaw, throttle, gear, AP) work with mouse, pen and touch, and with the keyboard when focused (HUD-06..HUD-13). [phone] (`flightHud.ts`, `throttleLever.ts`)

## 10. Phone remote (/rc/)

Sources: `src/remote/createPhoneControllerApp.tsx`, `PhoneController.tsx`, `PhoneUnpaired.tsx`,
`PhoneQrScanner.tsx`, `qrScanner.ts`, `PhoneStick.tsx`, `PhoneYaw.tsx`, `PhoneThrottle.tsx`,
`PhoneEngine.tsx`, `PhoneViewSwitch.tsx`, `PhoneSettings.tsx`, `phoneSettingsStore.ts`,
`PhoneFullscreenPrompt.tsx`, `useFullscreenOffer.ts`, `fullscreenMessage.tsx`,
`ConnectionDiagnosticsPanel.tsx`, `connectionDiagnostics.ts`, `phoneControllerClient.ts`,
`peerTransport.ts`, `pairing.ts`, `protocol.ts`, `iceConfig.ts`, `phone.css`;
`src/flight/remote/createPhoneControlSession.ts`, `src/flight/diagnostics/phoneCameraTrace.ts`.
Desktop side: TAB-REMOTE-*. Pairing URL rules: URL-12..URL-16. All items [phone]; the
phone and computer on the same non-guest Wi-Fi. Over plain HTTP on the LAN the phone has
no in-page QR camera, wake lock or clipboard (secure-context APIs): pair with the phone's
own camera app, or serve HTTPS (`DEV_TLS_CERT`/`DEV_TLS_KEY`, `npm run dev:lan`).

### Pairing and connection

- [ ] RC-01: `/rc/` with no invitation: "Scan a new QR to connect", "Each QR pairs once…" text, "Scan QR code" button and the hint "Keep both devices on the same non-guest Wi-Fi…". (`PhoneUnpaired.tsx`)
- [ ] RC-02: in-page QR scanner (jsQR): "Starting the camera…", "Point the camera at the QR on the computer.", "Found it. Connecting…"; failures: not secure ("…secure (https) page…"), unsupported browser, "Camera access was not allowed…", "No camera was found.", camera busy, generic failure; "That QR is not a 0SFS controller link…" for a foreign QR; Cancel / Try again. A 0SFS QR for another origin navigates there. (`PhoneQrScanner.tsx`, `qrScanner.ts`)
- [ ] RC-03: connection progress messages in order: "Connecting to pairing service…", "Connecting to your computer…", "Opening direct control channel…", "Synchronizing aircraft controls…", then "Connected · Desktop controls". Failures: "Could not connect directly. …" (after 15 s), "Pairing service unavailable…", "This browser could not start phone control. Try a current Safari or Chrome browser.", "Controller session ended. Scan a new QR.", "Unsupported phone protocol. Reload both devices." (`phoneControllerClient.ts`, `protocol.ts`)
- [ ] RC-04: one phone per QR: re-scanning a used or expired QR fails; after the desktop chooses Replace phone the old phone ends ("Phone connection closed. Scan a new QR to reconnect."). (`pairing.ts`, `peerTransport.ts`, TAB-REMOTE-05)
- [ ] RC-05: pairing-service reconnect: if the PeerJS signalling server drops, up to 3 attempts with 2 s / 4 s / 8 s backoff ("Reconnect attempt n of 3") while an open flight link keeps working. (`peerTransport.ts`)
- [ ] RC-06: link loss: "Connection lost · Waiting for the computer" on the phone; the desktop holds the last command then takes over (TAB-REMOTE-09); when the link returns the phone resumes without re-pairing; a closed channel says "Phone control channel closed…". Toggle the phone's Wi-Fi briefly. (`phoneControllerClient.ts`, `peerTransport.ts`)
- [ ] RC-07: ICE servers: `osfs.ice-servers` (this device, JSON) overrides `VITE_ICE_SERVERS` (build), which overrides the default STUN list (stun.l.google.com:19302, stun1.l.google.com:19302, stun.cloudflare.com:3478). Connection details show Direct vs "Relayed through TURN". (`iceConfig.ts`)
- [ ] RC-08: protocol v1 limits: messages ≤ 2,048 bytes, control frames at 60 Hz plus heartbeats every 50 ms, host status stale after 250 ms (link chip shows not live), handoff 2 s. Additive v1 fields (gear, engine, hand-back, blend, camera aim, haptic feedback, auto flaps, auto trim, the touch mark on frames and actions) keep an old phone and new desktop (and vice versa) working. Pair an old-app phone page with a new-app desktop if the protocols are meant to stay compatible. (`protocol.ts`)

### Controller screen

- [ ] RC-09: layout: the HUD's AUTO TRIM corner (TRIM AUTO above, AUTO TRIM below; each half turns its own trim assist on or off, lit only while on, and disables its trim slider while on; a computer without trim assists gets the centre-trim corner ROLL / PITCH instead), PITCH trim lever, ROLL trim slider, PITCH/ROLL stick pad, FLAPS lever; YAW, engine widget, camera pad with the view switch, THR lever; chip grid under the controls (or over them per settings). Portrait and landscape. (`PhoneController.tsx`, `phone.css`)
- [ ] RC-10: stick pad: the knob sits under the finger, full travel at the ring, returns to centre on release; separate pads work at the same time (multi-touch). Once the computer sends an attitude, the pitch/roll pad is the desktop's attitude indicator instead (same renderer and camera view, full travel at the instrument's edge per axis, at rest showing the aircraft's controls); Plain in settings keeps the crosshair pad. Its corners carry the Compact HUD's readings, tinted the same: IAS top left, HDG top centre, VS top right, AOA bottom left, ALT (above sea level) bottom right. (`PhoneStick.tsx`, `phoneAttitudeReadings.ts`, `fly/src/flight/hud/phoneAttitude.ts`)
- [ ] RC-11: YAW: returns to centre on release, wherever the finger lifts, by touch or mouse (Settings → Yaw "Return to centre", return time 0-1,500 ms in 50 ms steps, 0 = "Instant") or keeps its value ("Keep value"). Every slider: a press on the track jumps the handle there (Settings → Sliders "Jump to finger") or, with "Grab the handle", only a press on the handle moves it, by as much as the finger moves. (`PhoneYaw.tsx`, `sliderPointer.ts`, `PhoneSettings.tsx`)
- [ ] RC-12: THR lever is the desktop's lever (HUD-12): hold to start with the ring, hold at idle to shut down, afterburner 🔥 on the F-35B, hollow handle, no fill, N1/N2 or RPM I-beams. With Sliders "Grab the handle" a press off its handle does nothing, except holding to start a stopped engine. FLAPS has an AUTO button (lit while automatic flaps fly them) on a computer that offers it. Engine widget: tapping fuel flow toggles lb/h ↔ gal/h (saved in `osfs.phone-fuel-flow-unit`). (`PhoneThrottle.tsx`, `PhoneEngine.tsx`, `throttleLever.ts`)
- [ ] RC-13: camera pad: one-finger swipe orbits (same angle per pixel as a mouse drag, 0.005 rad/px), pinch zooms (each frame limited to ×0.5..×2) and, by default, two fingers also orbit with the point between them (⚙ Camera pad → Zoom only turns that off); lifting one of two glides the zoom on while the finger left keeps orbiting; arrow keys when focused; the view switch toggles cockpit/chase and follows the desktop's view. Behaviour is tuned by the desktop's Phone camera trackpad section (TAB-REMOTE-13). (`PhoneController.tsx` PhoneCameraPad, `PhoneViewSwitch.tsx`, `protocol.ts`)
- [ ] RC-14: telemetry chips: IAS kt (3 digits) and ALT ft (5 digits) only while the pad has no corner readings (RC-10), HDG ° (3 digits), "---" before data; G gear chip ("Landing gear down — tap to raise" / "Landing gear up — tap to lower"; disabled with "This simulator does not report gear position" when the host does not report gear). (`PhoneController.tsx`)
- [ ] RC-15: link chip PAIRING / DESKTOP / PHONE / BLEND / OFFLINE, lit while host status is fresh; screen-reader status carries the message line. (`PhoneController.tsx`)
- [ ] RC-16: while flying: Pause (Ⅱ / ▶, "Pause simulation"/"Resume simulation", lit while paused) and Release; otherwise "Take control" ("Taking control…"; disabled with the reason as its title, e.g. "Center controls to take over."; "Request timed out…", "Connection busy. Try again."); after the session ended "Scan QR code". Messages "Phone controls[ · Simulation paused]". Control offered back by the computer is asked for automatically (hand-back now/idle, asked every 500 ms). (`PhoneController.tsx`, `phoneControllerClient.ts`)
- [ ] RC-17: Haptics chip ("Touchdown pulses while you fly" / "Unavailable on this device" where `navigator.vibrate` is missing, e.g. iPhone): touchdown pulses up to 60 ms. Synced with the desktop's `osfs.phone.haptics`. (`PhoneController.tsx`, `protocol.ts` MAX_HAPTIC_PULSE_MS)
- [ ] RC-18: send-rate chip "nHz" ("Control frames this phone sent in the last second"; "—" when not sending). (`PhoneController.tsx`)
- [ ] RC-19: connection details sheet: path Direct / "Relayed through TURN", pairing service registered, offered candidates, ICE servers, channels, frames sent, buffered, round trip, receive-to-physics time, latest applied frame, timeline; "Copy diagnostics" ("Diagnostics copied."; addresses masked); after the session ended the recovery text "On the computer, open the Remote Control tab and create a new QR, then scan it." (`ConnectionDiagnosticsPanel.tsx`, `connectionDiagnostics.ts`)
- [ ] RC-20: ⚙ settings sheet: Button grid Bottom / Top; Pitch/roll pad Attitude indicator / Plain; Yaw Return to centre / Keep value + Return time; Sliders Jump to finger / Grab the handle; Camera pad Orbit and zoom / Zoom only. Saved on the phone (`osfs.phone-grid-position`, `osfs.phone-attitude`, `osfs.phone-yaw-release`, `osfs.phone-yaw-return-ms`, `osfs.phone-haptics`, `osfs.phone-sliders`, `osfs.phone-two-fingers`) and synced both ways with the desktop's Phone controller section (TAB-REMOTE-12). (`PhoneSettings.tsx`, `phoneSettingsStore.ts`)
- [ ] RC-21: B brake chip, held rather than tapped (brakes while pressed). (`PhoneController.tsx` PhoneBrake)
- [ ] RC-22: controls are disabled ("locked") while not flying; a pending take-control asks the desktop to centre its controls first (exclusive mode). (`PhoneController.tsx`, `createPhoneControlSession.ts`)

### Phone lifecycle and fullscreen

- [ ] RC-23: phone page hidden or pagehide: transient controls (stick, yaw, brake) are centred and sent twice more, the wake lock is released; on return the wake lock is re-acquired and control is asked back. Blur and orientation change cancel transient controls. (`phoneControllerClient.ts`, `createPhoneControllerApp.tsx`)
- [ ] RC-24: screen wake lock keeps the phone awake while connected (secure context only). (`createPhoneControllerApp.tsx`)
- [ ] RC-25: fullscreen offer: popup "Hide the browser bars for a bigger controller?" with Dismiss / ⛶ Fullscreen ("The browser declined fullscreen." on refusal); dismissal remembered in `osfs.fullscreen-prompt-dismissed`; with "every visit" stored (`osfs.fullscreen-every-visit`) the first tap enters fullscreen; ⛶ chip toggles fullscreen. iPhone: Home Screen notice (Share → Add to Home Screen) with "Got it". (`PhoneFullscreenPrompt.tsx`, `useFullscreenOffer.ts`, `fullscreenMessage.tsx`)
- [ ] RC-26: the desktop's tab hidden while the phone flies pauses (TAB-REMOTE-10) and the phone shows "Simulation paused". (`createPhoneControlSession.ts`)

### Camera trace

- [ ] RC-27: `/fly/?phoneCameraTrace=1` records the phone camera pipeline (touch, send, receive, playout, apply) for up to 7,200 records (~2 min); `window.osfsPhoneCameraTrace.download()` saves `phone-camera-trace-<ISO>.json`. Without the flag nothing is recorded. (`src/flight/diagnostics/phoneCameraTrace.ts`, `protocol.ts` MAX_TRACE_EVENTS)

## 11. Rendering, audio and effects (cross-cutting)

Sources: `foss-earth:src/engine/babylon/rendererPreference.ts`, `createRendererMode.ts`,
`foss-earth:src/settings/catalogue/index.ts` (URL aliases); `src/flight/settings/skyDefaults.ts`,
`src/flight/hud/attitudeRenderer.ts`, `engineSpoolRenderer.ts`, `src/flight/aircraft/*`,
`src/flight/diagnostics/create*Overlay.ts`, `src/flight/hud/gVisionOverlay.ts`,
`src/flight/audio/*` (`audioRendererRegistry.ts`, `worklet/dspProcessor.js`, `dsp/*`),
`vite.config.ts` (`babylonShaderChunks`, `assetsInclude: **/*.wasm`).
Per-tab behaviour is in TAB-SOUND-*, TAB-EXHAUST-*, TAB-GFORCES-*, TAB-DEBUG-*, AC-*; these
items check that it all holds on each renderer and in a production build.

- [ ] RENDER-01: Auto-detect starts WebGPU where it works, else WebGL2, else WebGL; the renderer that last started is remembered (`foss-earth:renderer-preference`) so a device where WebGPU failed does not probe it every visit; choosing Auto-detect again forgets it; `?renderer=webgpu|webgl2|webgl` forces a renderer for the visit (URL-07); a choice in the Renderer tab saves `renderer.backend` and reloads with `renderer` and `set.renderer.backend` removed from the address. **(Earth-owned)** (`rendererPreference.ts` applyRendererChoice)
- [ ] RENDER-02: on WebGPU and on WebGL2 (`?renderer=webgl2`), compare: aircraft meshes, materials and LODs; navigation/strobe/beacon/landing lights and their ground beams (AC-19/20); exhaust plume, hot-surface glow, smoke (TAB-EXHAUST-05..08); G vision vignette (TAB-GFORCES-03); forces, collision and wheel-spin overlays (TAB-DEBUG-03..05); wireframe (TAB-AIRCRAFT-03); sky, night lights and exposure (EARTH-SKY-*). No console shader/pipeline errors. [WebGPU and WebGL] (`src/flight/aircraft/*`, `src/flight/diagnostics/*`)
- [ ] RENDER-03: on a WebGL1-only or very slow device the flight still starts: slow-device sky defaults apply (EARTH-SKY-04), the attitude indicator uses Canvas 2D, shaft indicators fall back to WebGL1. Force `?renderer=webgl`. [WebGL] (`skyDefaults.ts`, `attitudeRenderer.ts`, `engineSpoolRenderer.ts`)
- [ ] RENDER-04: GPU sharing: on WebGPU the attitude indicator and the shaft indicators draw on the globe's GPU device (notes in Renderer → Instruments). [WebGPU] (`createFlightSimApp.ts` gpuDevice, `attitudeWebGpu.ts`)
- [ ] RENDER-05: a paused flight renders only on request: the frame rate drops while paused and nothing moves, yet orbiting (mouse, gamepad, phone) and panel changes still redraw. (`createFlightSimApp.ts` requestRender)
- [ ] RENDER-06: the production build loads Babylon's shader chunks (no runtime shader fetch failures on a static host) and its WASM files (JSBSim, audio DSP) from the built site. Run the built site from a static server, not only `npm run dev`. (`vite.config.ts` babylonShaderChunks/assetsInclude)
- [ ] AUDIO-01: the audio engine runs in an AudioWorklet (`osfs-dsp` processor, ABI `osfs-audio-dynamics-v4`) on WebAssembly; a mismatched worklet/WASM build is refused rather than played; without AudioWorklet or WebAssembly the Quality tiers are disabled with their reasons and only the tire cue plays (TAB-SOUND-02). [audio] (`audioRendererRegistry.ts`, `worklet/dspProcessor.js`)
- [ ] AUDIO-02: no sampled engine recordings ship: High is procedural; the experimental IndexedDB bank (`osfs-audio-bank`) is not used in production, so the browser's IndexedDB for the site stays empty after a flight. [audio] (`audioBank.ts`)
- [ ] AUDIO-03: no warning sounds exist at 8bc8af6 (stall warning, stick pusher and gear are visual only: HUD-14); if the new app adds any, that is an addition. [audio] (`src/flight/audio/*`, `hud/evaluationInstruments.ts`)

## 12. Persistence: saved state, migrations, session-only settings

Sources: `src/flight/jsbsim/savedFlight.ts`, `src/flight/settings/flightMigrations.ts`,
`src/flight/settings/registerFlightSettings.ts`, `src/flight/createFlightSimApp.ts`
(preferenceStorage, saveFlight), `src/flight/hud/engineMonitorLayout.ts`,
`src/remote/phoneSettingsStore.ts`, `src/remote/PhoneEngine.tsx`, `src/remote/iceConfig.ts`,
`src/fullscreen/*`; `foss-earth:src/settings/appSettings.ts`, `foss-earth:src/settings/catalogue/index.ts`
(legacy keys), `foss-earth:src/shell/savedWorkspace.ts`, `foss-earth:src/shell/panelSectionsOpen.ts`,
`foss-earth:src/windowing/react/locationDraft.ts`, `foss-earth:src/search/lookupCache.ts`,
`foss-earth:src/diagnostics/sessionTrail.ts`, `foss-earth:src/terrain/mapCache.ts`,
`foss-earth:src/engine/babylon/rendererPreference.ts`; gamepad-tools `src/core/persistence.ts`.
Everything is per origin (per scheme + host + port): the old and new apps on different
ports never share it, and a move to a new domain starts every user from defaults unless
something migrates it (see Known differences).

- [ ] PERSIST-01: localStorage key inventory after a session that touches everything (fly, change settings, pair a phone, open the Engine tab, rebind a control, search a place): 0SFS `osfs.saved-flight.v1`, `osfs.engineMonitor.v2`, `osfs.world-detail-import`, `osfs.ice-servers` (only if set by hand), `osfs.fullscreen-every-visit`, `osfs.fullscreen-prompt-dismissed`; Earth `foss-earth.settings.v1`, `foss-earth.workspace`, `foss-earth.panelSectionsOpen`, `foss-earth.location-draft.v1.airport`, `foss-earth.location-lookups.v1`, `foss-earth:renderer-preference`, `foss-earth.visit <id>` trail records and `foss-earth.visit-ping`; gamepad-tools `gamepad-tools:profiles:0sfs:index` and `…:profile:<id>`; on the phone origin `osfs.phone-grid-position`, `osfs.phone-yaw-release`, `osfs.phone-yaw-return-ms`, `osfs.phone-haptics`, `osfs.phone-two-fingers`, `osfs.phone-fuel-flow-unit`. Compare the key lists of both apps in DevTools → Application; any renamed key needs a migration or a Known-differences entry. (sources above)
- [ ] PERSIST-02: other storage: Cache Storage `foss-earth-map-tiles-v1` / `foss-earth-map-index-v1` (Map → Loading and memory → Map cache, EARTH-MAP-05); no IndexedDB in production (AUDIO-02); no sessionStorage written by 0SFS code; no service worker (NATIVE-01). (`foss-earth:src/terrain/mapCache.ts`)
- [ ] PERSIST-03: `foss-earth.settings.v1` holds only values that differ from their defaults, flight parameters included; secrets (`map.source.googleKey`, `map.source.cartoKey`) are kept on this device but never exported or put in reports/presets. Change a flight setting, reload, check it stays; reset it to default and check the entry disappears. (`foss-earth:src/settings/appSettings.ts`, EARTH-SET-04)
- [ ] PERSIST-04: session-only (lost on reload, never saved): `osfs.flight.allowCoarserThisSession`, the frame-budget profiling parameters (`renderer.profiling.*`: Measure frame time etc.), Debug → Collision geometry and the Wheel spin experiment (until kept), the custom wind (TAB-WEATHER-01), autopilot master engage (TAB-AUTOPILOT-08). (`flightParameters.ts` session, `registerFlightSettings.ts` frameProfilingParameters, `FlightControlPanel.tsx`)
- [ ] PERSIST-05: `?set.<id>=` values apply for this visit only, are listed under Settings → Saved settings "From the URL for this visit", and are saved only by "Keep these values" (URL-02). (`foss-earth:src/shell/settings/savedSettings.ts`)
- [ ] PERSIST-06: two open flight tabs on the same origin: a setting saved in one is followed by the other (storage event → registry reload); the saved flight is written by whichever tab saves last. (`foss-earth:src/settings/appSettings.ts` onStorage)
- [ ] PERSIST-07: 0SFS legacy keys migrate into parameters once (LIFE-22) and the World detail import runs once (LIFE-17); Earth migrates its own legacy keys the flight inherits: `foss-earth.map-detail.v1`, `foss-earth.theme`, `foss-earth.hudButtons`, `foss-earth.performanceMetricVisibility`, `foss-earth.inputMode`, `foss-earth.inputSensitivity` (+ version), `osfs.orbit-invert` (into `input.orbit.*`), tuner keys. Seed `osfs.orbit-invert` = `{"invertYaw":true}` on a clean origin and check Controls → Orbit. **(Earth-owned migrations)** (`foss-earth:src/settings/catalogue/index.ts` legacy keys)
- [ ] PERSIST-08: the saved flight (`osfs.saved-flight.v1`) restores everything in LIFE-03 and is ignored, not fatal, if corrupt (LIFE-06); a test-stand visit never writes it (TAB-ENGINE-08). Paste a corrupt value and reload. (`savedFlight.ts`)
- [ ] PERSIST-09: the window layout (open tabs, docked/floating windows, sizes) and each panel section's open/closed state persist (`foss-earth.workspace`, `foss-earth.panelSectionsOpen`); the Engine tab's own sections persist per aircraft/engine scope in `osfs.engineMonitor.v2`. **(Earth-owned for the first two)** (`savedWorkspace.ts`, `panelSectionsOpen.ts`, `engineMonitorLayout.ts`)
- [ ] PERSIST-10: the Location tab draft (query, coordinates, airport, Departure/Arrival, runway, last results) persists in `foss-earth.location-draft.v1.airport` (the flight uses the airport variant); search answers are cached per `search.cacheDuration`/`cacheEntries` in `foss-earth.location-lookups.v1`. **(Earth-owned)** (`locationDraft.ts`, `lookupCache.ts`)
- [ ] PERSIST-11: storage refused (private mode with zero quota, or blocked site data): the flight still starts and flies with defaults; saving shows the LIFE-07 / LIFE-18 messages; settings changes apply until reload. (`createFlightSimApp.ts`, `foss-earth:src/settings/appSettings.ts`)
- [ ] PERSIST-12: there is no "saved locations" or bookmarks feature at 8bc8af6 (only the Location draft and the Start position, `osfs.start.latitude/longitude`). If the new app adds one, it is an addition. (`LocationPanel.tsx`, `flightParameters.ts` START)

## 13. Native, PWA and page lifecycle

Sources: `index.html`, `public/manifest.webmanifest`, `public/rc.webmanifest`, `src/main.tsx`
(offerFullscreen, trackViewportInsets), `src/fullscreen/*`, `src/flight/createFlightSimApp.ts`
(onVisibilityChange, pagehide), `src/flight/audio/createFlightAudio.ts`, `src/flight/hud/flightHud.ts`,
`throttleLever.ts`, `src/remote/phoneControllerClient.ts`; `foss-earth:src/app/appFiles.ts`
(globe-only). Manifests and metadata: ROUTE-10..ROUTE-13; fullscreen offers: SHELL-29/30, RC-25.

- [ ] NATIVE-01: no service worker at 8bc8af6: DevTools → Application → Service workers is empty on `/`, `/fly/` and `/rc/`; the app does not work offline (a reload with the network off fails the same way in both apps). Earth's "App files" caching is globe-only (EARTH-SET-01). If the new app registers a worker, that is an addition to review. (`src/main.tsx`, `foss-earth:src/app/appFiles.ts`)
- [ ] NATIVE-02: install: "Install app" / Add to Home Screen from `/fly/` installs "0SFS" opening `/fly/` fullscreen; from `/rc/` installs "0SFS RC" scoped to `/rc/` (iOS title "0SFS RC" via `apple-mobile-web-app-title`); launched from the Home Screen there are no browser bars (`apple-mobile-web-app-capable`, black-translucent status bar). [phone] (`index.html`, `public/*.webmanifest`)
- [ ] NATIVE-03: notched phones: `viewport-fit=cover`; HUD, toolbar and phone controls stay inside the safe area in portrait and landscape. [phone] (`index.html`, `src/styles/flight.css`, `src/remote/phone.css`)
- [ ] NATIVE-04: fullscreen: ⛶ enters/leaves; Esc or the browser leaving fullscreen keeps the toolbar button in step; "Every visit" (`osfs.fullscreen-every-visit`) enters on the first interaction of later visits; the flight and the phone share the dismissal key `osfs.fullscreen-prompt-dismissed` per origin. (`src/fullscreen/fullscreen.ts`, `src/remote/useFullscreenOffer.ts`, `createFlightHudBar.ts` attachFullscreenButton; the storage helpers, keys included, are Earth's `foss-earth/shell` fullscreen module) **(Earth-owned helpers)**
- [ ] NATIVE-05: wake lock: only the phone controller holds a screen wake lock while connected (RC-24); the desktop flight requests none. (`phoneControllerClient.ts`)
- [ ] NATIVE-06: page hidden (switch tab, minimise, lock phone): the frame loop stops with the browser's animation frames, so the flight does not advance; sound is held (TAB-SOUND-12); the HUD stick and throttle hold are released; haptics cancelled, the phone told and the flight saved (LIFE-21); engine history keeps sampling only if "Continue hidden history capture" is on (TAB-ENGINE-05). On return the flight continues from where it was; compare with the old app that it does not jump ahead after a long absence. (`createFlightSimApp.ts`, `createFlightAudio.ts`, `flightHud.ts`, `throttleLever.ts`)
- [ ] NATIVE-07: leaving or reloading the page saves the flight on `pagehide` even where no visibilitychange fires first (iOS Safari); a reload right after a manoeuvre resumes there. (`createFlightSimApp.ts`)
- [ ] NATIVE-08: touch on the canvas never scrolls or zooms the page (two-finger gestures go to the camera, INPUT-17); long-press on HUD controls and the attitude pad does not open a context menu or select text. [phone] (`flightCameraInput.ts`, `flightHud.ts`)

## 14. Diagnostics captures, globals and URL flags

Sources: `src/main.tsx` (startAppDiagnostics, report-only route),
`src/loading/createFlightLoadingScreen.ts`, `src/flight/diagnostics/{flightPerformanceCapture,frameProfile,phoneCameraTrace,flightRecorder,flightLog}.ts`,
`src/flight/jsbsim/createJsbsimRuntime.ts`, `src/remote/connectionDiagnostics.ts`;
`foss-earth:src/diagnostics/*`, `foss-earth:src/engine/babylon/createBabylonRuntime.ts`.
URL flags themselves: URL-01..URL-08; this section checks the captures they produce.

- [ ] DIAG-01: every capture and its file/global, in both apps: loading report (Copy loading report or download `osfs-loading-report.json`; `window.osfsLoadingDiagnostics()` while loading, SHELL-22); flight log CSV (TAB-LOGGING-04); engine history `engine-history.csv` (TAB-ENGINE-10); frame trace `0sfs-frame-trace…` and `window.osfsFrameProfiler.summary()` / `window.osfsFrameProfile` while frame measurement is on (TAB-DEBUG-07); stutter trace "Copy performance trace" and `window.osfsFlightPerformance` with `?flightPerf=1` (TAB-DEBUG-08); phone camera trace with `?phoneCameraTrace=1` (RC-27); phone connection diagnostics (TAB-REMOTE-04, RC-19); bug report / activity report (EARTH-BUG-01, EARTH-SET-05). Diff each output's structure (keys/columns) old vs new. (sources above)
- [ ] DIAG-02: `window.osfsJsbsimBuild` holds the running JSBSim build identity (package name/version, native and SDK commits, dirty flags) and matches About → Built from and the Settings → Diagnostics "JSBSim: …" line. (`createJsbsimRuntime.ts`)
- [ ] DIAG-03: the visit trail: with "Remember how a visit ended" on, a visit that ended badly (kill the tab mid-load) is reported on the next visit's Diagnostics/bug report as an earlier visit's trail; flight event-log messages appear in the trail (SHELL-28). **(Earth-owned)** (`foss-earth:src/diagnostics/sessionTrail.ts`, `createFlightSimApp.ts` flightLog.subscribe)
- [ ] DIAG-04: `?report` on `/fly/` (URL-01) shows only the bug-report form and still reads the last visit's trail, without loading the simulator; this is the way to report a flight that crashes on start. (`src/main.tsx` showReportOnly)
- [ ] DIAG-05: reports never carry secrets: Google/CARTO keys are redacted in the diagnostics report and bug report; phone diagnostics mask addresses to their subnet. Set a fake `?key=` and search the copied report for it. (`createFlightSimApp.ts` reportSecrets, `connectionDiagnostics.ts`)
- [ ] DIAG-06: console: the only expected warning on a reference start without a Google key is "No Google Maps API key found…"; JSBSim stderr lines are forwarded as `[jsbsim] …` warnings. Compare the console of both apps on a clean start for new errors. (`createFlightSimApp.ts` onLog, `build/reference/README.md` smoke run)
- [ ] DIAG-07: Earth runtime debug flags in the flight: `?mapDebug=1`, `?terrainCapture=1` (URL-08) and the profiling session (TAB-DEBUG-07) behave as in the old app. **(Earth-owned)** (`foss-earth:src/engine/babylon/createBabylonRuntime.ts`)

## 15. Build and deploy outputs

Sources: `package.json` (scripts), `vite.config.ts` (base, defines, copyIndexToPagesPaths,
lanOriginPlugin, builtFrom/jsbsimBuild, babylonShaderChunks, server https), `scripts/pages-html.mjs`,
`scripts/verify-jsbsim-artifact.mjs`, `scripts/verify-audio-artifact.mjs`,
`scripts/build-exhaust-optics.mjs`, `scripts/build-f135-engine.mjs`, `scripts/dev-lan.mjs`,
`scripts/check-phone-layout.mjs`, `public/*`; the reference build in
`build/reference/0sfs/dist` and `build/reference/README.md`.
In the split, the landing page is built by the website repo and the flight by `fly`; each
item applies to whichever repo now produces that output.

- [ ] BUILD-01: the production build runs and passes, in order: verify JSBSim artifact, verify audio artifact, check exhaust optics, check F135 engine assets, typecheck, `vite build`, then verify the JSBSim and audio artifacts inside `dist`. A missing or stale generated asset fails the build rather than shipping. (`package.json` build)
- [ ] BUILD-02: `dist` holds what the reference holds: `index.html`, `fly/index.html`, `rc/index.html`, `assets/` (hashed JS/CSS, `jsbsim_wasm-*.wasm`/`.mjs`, `audio-dsp-*.wasm`, `dspProcessor-*.js`), `jsbsim-data/` (`aircraft/`, `engine/`, `manifest.json`), `jsbsim-artifact.json`, `aircraft/` (`cessna-172`, `cirrus-vision-jet`, `f-35b`, `thumbnails`), `favicon.svg`, `icons.svg`, `preview.jpg`, `manifest.webmanifest`, `rc.webmanifest`, `robots.txt`, `sitemap.xml`; no `404.html`. Total about 24 MB. Compare listings. (`build/reference/0sfs/dist`, `vite.config.ts`, `public/`)
  - Split: compare the website's `0sfs.github.io/dist`. It has the reference's files plus `landing.css`, and lacks only the F-35B data's `.gitattributes`, a dotfile the old `gh-pages` deploy did not publish either. 24.4 MB. fly's own `dist` has no `robots.txt`, `sitemap.xml` or `preview.jpg`, which are the website's.
- [ ] BUILD-03: `fly/index.html` and `rc/index.html` are the home page retagged by `pageHtmlFor`: title, `og:title`, `og:url` per route, the WebSite JSON-LD removed, canonical `https://0sfs.github.io/fly/` on `/fly/` and `noindex` instead of a canonical on `/rc/` (ROUTE-10); the build fails if `index.html` lacks any tag it retags. (`scripts/pages-html.mjs`)
- [ ] BUILD-04: base path: `/` locally and for a `<owner>.github.io` repository; `/<repository>/` when built in GitHub Actions for any other repository. Build with `GITHUB_ACTIONS=1 GITHUB_REPOSITORY=owner/name` and check assets, manifests, `fly/`, `rc/`, aircraft and JSBSim data all load under `/name/`. (`vite.config.ts` base)
- [ ] BUILD-05: version label: `__SOURCE_VERSION__` is the 12-character git commit, `-dirty` with uncommitted changes, `unknown` outside a git checkout (as in the reference build); `__BUILD_TIME__` the build's ISO time. Both show in About, in the flight log CSV metadata and file name (`0sfs-<aircraft>-<version>-<time>.csv`), and the version in the bug report identity. (`vite.config.ts`, `createFlightSimApp.ts` recordingFileName)
- [ ] BUILD-06: repository slug: `GITHUB_REPOSITORY`, else the git `origin` remote; drives About's source link, the bug report's GitHub issue target (EARTH-BUG-01) and the landing page's documentation/licence links, which fall back to `0SFS/0SFS.github.io` when there is no slug (LAND-07/08). Empty in the reference build. (`vite.config.ts` getRepositorySlug, `src/info/createInfoPage.ts` blobBase)
  - Split: fly's builds record `https://github.com/0SFS/fly` with the commit, for About and the issue target (KD-11). The landing's links are fixed (AMB-02).
- [ ] BUILD-07: About → Built from lists the JSBSim fork with its identity from `dist/build-metadata.json` (package `@felipegalind0/jsbsim` `1.2.4-fork.21`, native commit `34eeae66e5c8…` in the reference) and the other packages and checkouts (FOSS Earth, gamepad-tools). (`vite.config.ts` builtFrom/jsbsimBuild, `jsbsim-artifact.json`)
- [ ] BUILD-08: dev servers: `npm run dev` (HTTP, loopback, no LAN origin); `npm run dev:lan` finds the LAN address, serves HTTPS with `DEV_TLS_CERT`/`DEV_TLS_KEY`, and injects `window.__OSFS_LAN_ORIGIN__` so the QR points at the LAN (URL-16); the dev server may read `../foss-earth` (`server.fs.allow`). (`scripts/dev-lan.mjs`, `vite.config.ts` lanOriginPlugin)
- [ ] BUILD-09: deploy: `npm run deploy` = build then `gh-pages -d dist` (pushes `dist` to the `gh-pages` branch). Decide and check the split's equivalent for both the website and the flight. (`package.json` deploy:gh-pages)
  - Decided: the website publishes the whole site, flight included (KD-12; `0sfs.github.io/docs/deploying.md`). fly has no deploy script. Not run: deploying is the release step after sign-off.
- [ ] BUILD-10: maintenance scripts still exist where needed: `build:audio` (rebuild the DSP WASM), `build:exhaust`, `build:engine-assets`, `check:phone-layout` (phone layout screenshots), `lint`, `test`, `ci`. (`package.json`)
- [ ] BUILD-11: the 0SFS package's library exports (`.` re-exporting FOSS Earth, its runtime and `createFlightSimApp`; `./layers`, `./cameraMath`, `./smoothElevation`, `./style.css`) — no consumer found in the checked repos; decide whether the split keeps them. (`package.json` exports, `src/index.ts`, `src/compat/*`)

## Ambiguities (decide before signing off)

Points where the old app does not say what the new app should do. Each needs a decision
recorded next to it; they are not parity failures by themselves.

- [ ] AMB-01: link base for every flight parameter's help "source" link: the old app points at `https://github.com/0SFS/0SFS.github.io/blob/main/<path>` (`registerFlightSettings.ts`). After the split the flight sources live in `fly` (and other repos), so decide the new base and check a few links resolve. (Section 6 intro, EARTH-HELP-01)
  - Decided: each link points at the repository that has the file, `https://github.com/0SFS/<repo>/blob/main/<path>`; for example `osfs.assist.autoTrim` goes to controls' `src/autoTrim.ts`. The link text is that path (KD-09). `flightParameters.test.ts` checks every link names a file in the checkouts. The links 404 until the repositories are pushed.
- [ ] AMB-02: landing page documentation and licence links (LAND-07/08) are built from the repository slug with the fallback `0SFS/0SFS.github.io`; decide where docs such as `docs/phone-controller.md`, `docs/jsbsim.md`, `docs/sound.md` live after the split and where the website's links point. (`src/info/createInfoPage.ts` blobBase)
  - Decided: the page links each document in its repository: phone controller in rc, sound in sound, development, JSBSim and the FOSS Earth relationship in fly, deploying in the website, and the licence files in fly. "Source on GitHub" and "Repository" go to https://github.com/0SFS. The website's tests check every link names a file in the checkouts. The links 404 until the repositories are pushed.
- [ ] AMB-03: search vs airport presets: search is deferred (Out of scope), but at 8bc8af6 the only way to reach the airport Departure/Arrival presets (LIFE-13, EARTH-LOC-03) is choosing an airport from the Location tab's search results. Decide whether the new app keeps the Location search box (with its Interface → Search settings and lookup cache) or offers another way to pick an airport; otherwise airport presets are unreachable.
- [ ] AMB-04: the Earth sections that are inert in the flight (Renderer → Work-saving experiments and Performance debug, EARTH-RENDER-02) appeared only because Earth appended every registered section; decide whether the new app shows them.
- [ ] AMB-05: phone protocol compatibility across versions (RC-08): decide whether an old-app phone page must pair with a new-app desktop and the reverse, or whether "Unsupported phone protocol. Reload both devices." is acceptable after a deploy.
- [ ] AMB-06: old-app wording inconsistency: the desktop says "Center controls and tap Fly again." / "Tap Fly again." (TAB-REMOTE-07/10) while the phone's button reads "Take control" (RC-16). Keep or fix; either is parity in function.
- [ ] AMB-07: gamepad-tools version: the reference build used `8427c1a`; the local checkout is newer (`5260c9e` at the time of writing). Editor wording and layout from a newer gamepad-tools are not 0SFS regressions; profile storage keys (INPUT-11) and the built-in profiles (TAB-CONTROLS-06) must still match or migrate.
- [ ] AMB-08: library exports of the old package (BUILD-11): keep, move or drop.
- [ ] AMB-09: entries in "How to run both apps" marked "to be confirmed" (the new app's flight URL and where the landing page is served) stay open until the new repos' dev servers are run.
  - Confirmed 2026-10-12: the development server serves `/fly/` and `/rc/`, and the site preview serves `/` too. Both were smoked headless with every aircraft.

## Known differences (accepted, not failures)

- [ ] KD-01: Earth-owned presentation: theme, toolbar component, game log, window/tab chrome, panel layout and the shared tabs (Location, Map, Renderer, Sky, Date and time, Interface, Settings, About, Bug report) come from FOSS Earth, which has moved on since `90e23b3e` (2026-10-08) and now uses panels with explicit ports. Compare what they do (items marked **(Earth-owned)**), not how they look. New Earth parameters, sections or presets that appear in the flight are additions to note, not failures.
- [ ] KD-02: airport/runway location presets are host-owned in `fly` now; at 8bc8af6 they were Earth's Location panel feature switched on by `enableAirportPresets`, with the per-aircraft values from `fdmProfiles.ts` runwayPresets. Positions, headings, paused/running state, throttle, flaps, gear and airspeed must still match LIFE-13; the panel's wording and attribution may differ.
- [ ] KD-03: the reference build's version label is `unknown` and its repository slug is empty (built outside a git checkout): About has no source link and the bug report says "The GitHub issue repository is not configured." A new build from git shows a commit and may file issues. Compare structure, not these values. (`build/reference/README.md`)
- [ ] KD-04: settings and saved state are per origin: the reference (port 4180) and the new dev server never share settings, saved flights, gamepad profiles, workspace or phone preferences; set both up from the same cleared state. In production, users keep their state only if the new app is served from the same origin (`https://0sfs.github.io`) with the same storage keys (PERSIST-01) or migrations for renamed ones.
- [ ] KD-05: no Google key in the reference build, so Google 3D Tiles items ([google]) need a key supplied to both apps (`?key=` once, stored per device), and the console warning "No Google Maps API key found…" is expected without one.
- [ ] KD-06: third-party endpoints (map tiles, elevation, OpenStreetMap place search and FreeAirportDB airport data, the PeerJS pairing server, STUN) are live services in both apps, so failures or rate limits there affect both equally; retry before calling a difference a regression.
- [ ] KD-07: the reference build's test baseline has 4 pre-existing timeouts (`build/reference/README.md`); the same tests timing out in the new repos is not a regression.
- [ ] KD-08: the landing page is static HTML from the website repo, so `/` runs no JavaScript bundle. It has nothing to show before the page itself (ROUTE-14), no bundle that can fail to download (ROUTE-15), and no `html[data-route]` (ROUTE-11). Its title, canonical, JSON-LD, theme-color, manifest and `<noscript>` text are unchanged. Old `?mode=flight|remote|globe` links are forwarded by a script in its head (ROUTE-05..07).
- [ ] KD-09: a flight parameter's help shows its source as a path in its own repository, e.g. `src/autoTrim.ts` (controls), where the old app showed the monolith path, e.g. `src/flight/controls/autoTrim.ts`. The link goes to that file in that repository (AMB-01). FOSS Earth parameters are unchanged.
- [ ] KD-10: fly's development server sends `/` and unknown paths such as `/foo` to `/fly/`, where the old development server showed the landing page (ROUTE-01, ROUTE-08). The site preview, like GitHub Pages, answers an unknown path with 404, as the old static build did.
- [ ] KD-11: the bug report's GitHub issue target is `0SFS/fly`, which the build records with its commit. The reference build had none (KD-03).
- [ ] KD-12: publishing changed. The website's `npm run deploy` assembles fly's build with the landing page and refuses a fly build with uncommitted or unpushed changes. The monolith's `npm run deploy` published `dist/` with `gh-pages`, which fly no longer depends on. Nothing has been deployed.

## 360° drone mod

Not in the old app. The drones are mods in the default modpack (since 29 October; before that they were absent until added).

- [ ] Chrome WebGPU: open `/fly/`. The gallery has the DJI family with Mini 2 and Air 2S, and About lists them under `0sfs/default-modpack`. The network does not request the drone feature chunk, drone JSBSim XML or HD asset.
- [x] Chrome WebGPU: Apply Mini 2. The network then requests the feature chunk and `dji-mini2.xml`. Physics runs at 120 Hz. Auto draws the local HD model nearby and the procedural silhouette beyond 15 m.
- [x] Chrome WebGPU: Mini 2 and Air 2S retain visible geometry through HD/procedural/HD zoom transitions. Their camera minimums are 0.25 m and 0.35 m respectively; repeated Auto refreshes do not remount the same procedural mesh.
- [ ] Shutter and 360 require WebGPU. A WebGL renderer shows "360° capture requires WebGPU." and does not read the canvas.
- [ ] Remove the mod. The flight returns to the Cessna 172 and the gallery loses the family.

## Out of scope

- [ ] OOS-01: Search: Earth's place search and anything beyond what the flight needs to place the aircraft (the Location tab search box EARTH-LOC-02, Interface → Search EARTH-IFACE-04, the lookup cache PERSIST-10) is deferred. There was no separate Search tab in the old flight (SHELL-13). See AMB-03 for the airport presets that depend on it.
- [ ] OOS-02: VR: there is no WebXR/VR code in 0SFS at 8bc8af6 (the only "VR" match is the SF50 rotation speed in validation code), so nothing to check; VR work in other repos (e.g. UMN-VR) is not part of this parity check.
- [ ] OOS-03: globe-only FOSS Earth features never shown in the flight: Scenes, 360 and 360-settings tabs (SHELL-14), Earth's own Controls tab and camera parameters, Interface → Theme and the globe toolbar items, Settings → App files and its service worker, the Dev host (`?dev=1`, URL-10). `?mode=globe` still redirects to FOSS Earth (ROUTE-07), which is in scope.
