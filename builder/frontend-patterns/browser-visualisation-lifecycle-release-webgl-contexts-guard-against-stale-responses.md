# Browser Visualisation Lifecycle: Release WebGL Contexts, Guard Against Stale Responses, Write View State From Latest State, and Do Not Mistake Software Rendering for a Regression

<!-- brain-entry -->
<!-- domain: builder -->
<!-- subdomain: frontend-patterns -->
<!-- tags: webgl, three-js, sigma-js, react, context-release, event-listener-leak, race-condition, end-to-end-testing, performance-tier, headless-chromium -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-29 -->

## Summary
Interactive network views that mount and unmount 2D (sigma-style) and 3D (three.js-style) renderers leak WebGL contexts and listeners unless torn down on unmount. Several UI races also appear when async fetches and multi-parameter state writes interleave.

## Knowledge
**Pattern.**
- Release the renderer and its WebGL context when a component unmounts, and remove every listener it added. Test it by switching between 2D and 3D twenty times and failing on any browser console warning about too many contexts (9 warnings before the fix, 0 after). A separate test that dragging in the 3D view raises no errors caught a listener leak.
- Stale responses: when a user switches the selected graph while a previous request is in flight, ignore responses that no longer match the current selection (live guard). An end-to-end test "switching graph while the previous one loads" went red then green.
- State writes: when a tab sets two view parameters back to back, write each through a setter that merges into the latest state, not a spread of state captured in props; the stale merge drops the first write.
- Leaving a heavy tab mid-computation (a force-layout settle) must not blank the app: test "leave the tab during the settle".
- Camera: a selection change must not reset a shared camera (assert the camera ratio is unchanged after selecting).
- Run a short startup GPU probe only as long as needed (stop after 3 draws if all exceed 33 ms) so a software-rendered page does not block for about a second before first paint. Pick a Fast profile when WebGL is missing, the renderer is software (SwiftShader, llvmpipe, Microsoft Basic Render), the core count is 2 or fewer, or the probe median is slow.
- A canvas the application sizes itself can cap the device pixel ratio in Fast mode; a canvas the library owns keeps the native ratio (a limitation to document).
- Headless Chromium rasterises WebGL in software, with GPU tasks of 7-9 s on large graphs; raise the smoke timeout and use smaller graphs for the smoke step instead of treating it as an application regression.
- Split the bundle: lazy-load the 3D chunk (main chunk 1,924.91 kB to 526.33 kB; gzip 525.39 to 151.54 kB; the 3D chunk 1,403 kB on demand).
- Replace a library worker supervisor with a time-sliced own worker if it re-runs expensive reducers on every iteration message.

**What worked.** Each defect got an end-to-end or unit test that was observed red first.

**What required manual intervention.** The context leak and the blank app were found by an independent reviewer and re-graded from Minor to Important/Critical; the stale-response race had no automated test (no component-test harness) and the ledger recorded that explicitly.

**Evidence.** Counts above; full end-to-end suite passing after the fixes (32/32).

## When to Use
Single-page applications using WebGL network renderers in React or similar frameworks.

## Example
```ts
useEffect(() => {
  const r = createRenderer(container);
  return () => { r.removeListeners(); r.dispose(); r.forceContextLoss?.(); };   // release on unmount
}, []);

let latest = 0;
async function load(id) { const my = ++latest; const g = await fetchGraph(id); if (my === latest) setGraph(g); }
```

## Pitfalls
- A stale-response guard needs a counter or key compare, not a flag that is never reset.
- Software rendering is also what many users with older hardware get; keep the Fast profile usable.
