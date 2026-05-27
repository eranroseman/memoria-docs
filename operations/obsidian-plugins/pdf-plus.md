# pdf-plus (PDF++)

Load-bearing settings:

- `enableHoverHighlight: true` — required for cite-on-hover behavior.
- `copyLinkFormat` — choose the format that includes the citekey and page (`[[{{citekey}}#p={{page}}]]` or similar). Without page-level granularity, the deep-linking value is lost.
- `defaultColorPalette` — keep stable across the vault. Color-coded highlights become semantically meaningful (e.g., red = contradicts, green = supports) only if the palette doesn't shift.
