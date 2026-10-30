# Custom Canvas Layer on a WebGL Network Renderer: Detach With a "Dead" Flag Set on the Kill Event, Sort Once, and Cap Per Frame

<!-- brain-entry -->
<!-- domain: builder -->
<!-- subdomain: frontend-patterns -->
<!-- tags: sigma-js, webgl, canvas-layer, overlay, react-cleanup, kill-event, lifecycle, performance, per-frame-cost -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-30 -->

## Summary
A 2D canvas layer attached to a sigma.js renderer (for directed arrows, annotations) crashes the page on graph or profile switches if its cleanup runs after the renderer has been killed. Track renderer death with a flag set on the renderer's kill event, and keep per-frame work cheap by sorting edges once when data are set and capping the drawn count.

## Knowledge
**Lifecycle.** In React the order of cleanups is not guaranteed: when the user switches graph or performance profile, the renderer's `kill()` can run before the effect cleanup of the component that attached the layer. After `kill()` every layer is gone and calls such as `refresh()` or `killLayer()` throw. Fix:
- On attach, `let dead = false; renderer.once('kill', () => { dead = true })`.
- Every public method of the layer (`set`, `draw`, `detach`) returns immediately when `dead`. `detach` also unsubscribes from `afterRender`, and calls `killLayer` only if not dead and the layer still exists.
- Draw in the renderer's `afterRender` event so the layer follows camera moves.

The defect was found by the final reviewer as critical: a graph or profile switch crashed the 2D views and also broke an end-to-end smoke step at an unrelated action (a colour selector). Evidence is two tests: a unit test "ignores `set()` once the renderer has been killed" (red then green) and an end-to-end step that switches the performance tier.

**Per-frame cost.** Sorting overlay edges by strength inside the draw callback ran thousands of comparisons on every frame. Sort once, in `set()`, and only when the edge array reference changes (`if (e !== edges) sorted = byStrength(e)`). A unit test asserts the sort happens once (red then green). Keep the drawing order strongest-first so that a cap removes the weakest edges: for example a cap of 3,000 arrows for a high-quality profile and 800 for a fast profile, applied after filtering to edges whose endpoints are both visible.

**Canvas sizing.** Size the overlay canvas to the renderer dimensions times the device pixel ratio, set its CSS size explicitly, and apply `setTransform(dpr, 0, 0, dpr, 0, 0)` before drawing.

## When to Use
Single-page applications that add their own drawing layer (arrows, regions, labels) to sigma.js or similar WebGL graph renderers inside a component framework.

## Example
```ts
function attachLayer(s: Sigma) {
  const ctx = s.createCanvas('overlay', { afterLayer: 'edges' }).getContext('2d')!
  let edges: Edge[] = [], sorted: Edge[] = [], dead = false
  s.once('kill', () => { dead = true })
  const draw = () => { if (dead) return; /* clear, then draw sorted.slice(0, cap) */ }
  s.on('afterRender', draw)
  return {
    set(e: Edge[]) { if (dead) return; if (e !== edges) sorted = byStrength(e); edges = e; s.refresh() },
    detach() { s.off('afterRender', draw); if (!dead && s.getCanvases()['overlay']) s.killLayer('overlay') },
  }
}
```

## Pitfalls
- A `dead` check inside `draw` alone is not enough: `set()` and `detach()` must also guard.
- Sorting a new array each time `set` is called with an equal but not identical array defeats the reference check; memoise upstream.
- The same defect class (listeners, contexts released late) applies to other lifecycle code; test profile and graph switching, not only first mount.
