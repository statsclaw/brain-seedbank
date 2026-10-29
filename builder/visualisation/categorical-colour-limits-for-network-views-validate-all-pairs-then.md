# Categorical Colour Limits for Network Views: Validate All Pairs, Then Reduce Hues Instead of Adding Them

<!-- brain-entry -->
<!-- domain: builder -->
<!-- subdomain: visualisation -->
<!-- tags: colour, palette, colour-vision-deficiency, delta-e, categorical, accessibility, network-visualisation, dark-mode -->
<!-- contributor: @mariostomazou85 -->
<!-- contributed: 2026-10-29 -->

## Summary
Seven or more categorical hues cannot all be told apart (normal and colour-vision-deficient viewers, light and dark themes). Validate every pair, and when pairs fail, encode categories in other channels or collapse groups rather than adding colours.

## Knowledge
**Pattern.**
- Validate a palette on all pairs, per theme, including a colour-vision-deficiency simulation; a subset passing does not imply the combined set passes. A four-hue set passed on its own, but combined with role colours it failed (violet versus blue deltaE 9.2 normal and 8.3 dark; green versus orange 2.9 under protanopia).
- Resulting decisions: in comparison views, nodes are neutral and only edges carry the context hues (role in tooltips, rings and tables); roles are folded into 3 colour groups plus grey; sign (activation versus inhibition) uses a diverging blue/red pair that passes both themes, while green/red only warns.
- Low-contrast hues (contrast 2.1 and 2.6) are acceptable when legends, tables and panel titles repeat the information.
- Assign colour slots stably: a context keeps its slot when others are removed; new ones take the lowest free slot, so colour follows the context.
- When only one category applies, a single border colour may be all the renderer can draw (one ring per node); put the full membership in a table.

**What worked.** An automated palette validator used before choosing colours.

**What required manual intervention.** The planned role-coloured nodes had to be dropped in the comparison views; a grouping of roles was needed.

**Evidence.** Validator results above; no 4+ subset of the full palette passed all pairs (for example orange versus magenta deltaE 12.9, blue versus violet 10.5, green versus orange CVD 3.2), while blue/orange/aqua passed.

## When to Use
Categorical colouring in dashboards, network diagrams and small multiples where users may have colour-vision differences or use a dark theme.

## Example
```text
palette -> for each theme, for each pair: dE(normal), dE(protan), dE(deutan), dE(tritan) >= threshold
if any pair fails: merge categories, move one to shape/ring/table, or keep nodes neutral
```

## Pitfalls
- Passing a single-theme check does not cover dark mode.
- Contrast warnings on light hues need redundant encoding, not just a thicker line.
