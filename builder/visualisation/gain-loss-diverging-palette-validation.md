# Gain/Loss Diverging Colour Pair With a Neutral Midpoint: Validate the Two Poles as a Pair in Both Themes, Then Cross-Check Against Colours Already in Use

<!-- brain-entry -->
<!-- domain: builder -->
<!-- subdomain: visualisation -->
<!-- tags: diverging-palette, colour-vision-deficiency, dark-mode, neutral-midpoint, delta-e, oklch, design-tokens, accessibility, signed-effect -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-30 -->

## Summary
For a signed effect (gained versus lost) use a diverging pair that is not red/green, interpolate from a neutral midpoint, and run the validator on the two poles for each theme. Then compute pairwise distances to every other encoding on the same screen, because the pair can pass alone and still clash with existing role or sign colours.

## Knowledge
**Choice of poles.** Red/green fails colour-vision-deficiency checks. Blue/orange or blue/red pairs were already in use for other encodings. A purple (gain) and rust (loss) pair, found by an OKLCH search, passed the validator in both themes. Light theme poles `#6d3871` and `#902201`; dark theme poles `#9d3eb1` and `#bd3d1b`. The neutral midpoint is a theme token too (a light grey on light surfaces, a dark mid-grey on dark surfaces), not black or white.

**Validation steps.**
1. Run the categorical-palette validator on the two poles per theme (two slots): lightness band, chroma floor, CVD separation and normal-vision floor, contrast of each pole against the surface (at least 3:1). Result: all checks passed in both themes; worst adjacent pair delta E 15.0 (deutan) and 11.9 (tritan) in light, 22.1 and 12.6 in dark.
2. Cross pairs: for every pole against every other colour in use (here five role and sign colours), compute delta E in OKLab times 100 under normal vision, protanopia, deuteranopia and tritanopia, in both themes. Target at least 8. Worst was 10.8 (a loss pole against an existing orange role colour in the dark theme).
3. Record the transcript next to the design spec so palette changes are justified.

**Encoding.** Colour = mid blended toward the pole by `0.25 + 0.75 * min(1, |effect| / s)` where `s` is the saturating effect size (here 4 on the log fold-change scale), so any non-zero change leaves the midpoint by a visible step; effect exactly 0 or non-finite = midpoint. Width encodes magnitude, 0.6 to 2.4 px with the same saturation, so magnitude is not carried by colour alone.

**Limit of the validator.** The categorical validator checks the two poles as categories. The ramp between the pole and the midpoint was not validated (lightness monotonicity along the ramp is a separate check for sequential or diverging ramps).

**What required manual intervention.** The two existing colour sets (roles and signs) already failed among themselves (an orange against a red); they were left unchanged because the new pair needed only to be safe against them, and the swap is confined to the theme token file if the pair has to change.

## When to Use
Any visualisation that shows direction of change (gain/loss, up/down, increased/decreased) alongside other colour encodings, in light and dark themes, for audiences that include colour-vision deficiency.

## Example
```ts
function signedColour(effect: number, t: { gain: string; loss: string; mid: string }, S = 4) {
  if (effect === 0 || !Number.isFinite(effect)) return t.mid
  const f = Math.min(1, Math.abs(effect) / S)
  const pole = effect > 0 ? t.gain : t.loss
  return f >= 1 ? pole : mix(t.mid, pole, 0.25 + 0.75 * f)
}
```

## Pitfalls
- A pair passing alone is not enough; the cross-check against existing colours found the weakest pair.
- Light colours need redundant encoding (width, tooltip, table); the validator itself warns that a delta E in the 6-8 band is legal only with secondary encoding.
- The dark theme midpoint must stay distinguishable from both the surface and the poles; check it as a third colour if edges near zero matter.
