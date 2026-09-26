# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project

"Tiny LLM Lab" — an educational, single-file web demo (`llm-demo.html`) that teaches how language models work. It is plain HTML + inline CSS + inline vanilla JS (`"use strict"`), with no build step, no dependencies, no package manager and no tests. To run it, open `llm-demo.html` in a browser. Quick JS checks can be done with `node -e` against extracted snippets. To syntax-check the whole inline script (the only `<script>` block):

```bash
node -e "const s=require('fs').readFileSync('llm-demo.html','utf8');new Function(s.split('<script>')[1].split('</script>')[0])"
```

## Architecture

The page has four tabbed `<section>`s (only one visible at a time, chosen tab persisted in `localStorage` key `llmTab`). They share a single model trained live in the browser:

1. **Tokens** — a toy byte-pair encoder (`trainBPE`, `bpeEncode`) learned from the corpus textarea; a slider controls how many merges are applied. Independent of the neural net.
2. **Training** — a word-level MLP language model (not a transformer): embedding table `E` (V×D) → concat of `CTX` context embeddings → tanh hidden layer (`W1`,`b1`) → logits (`W2`,`b2`) → softmax. Forward pass in `forward`, hand-written backprop + SGD in `trainStep`, one shuffled pass in `trainEpoch`. Hyperparameters are the constants `CTX`, `D`, `H`; `"."` (`PAD`) doubles as sentence boundary and start padding. Also draws a loss chart and a 3D PCA view of embeddings (`pca3`, `drawEmb`, draggable/zoomable) plus nearest-neighbour lookup.
3. **Attention** — heatmap of scaled dot-product attention computed directly from the trained embeddings `net.E` (no learned Q/K/V), with optional causal mask and a sharpness/temperature slider.
4. **Sampling** — next-token distribution from the model with temperature / top-k / top-p filtering (`filteredDist`), step-by-step or batch generation; clicking a probability bar forces that token.

Key flow and conventions:
- `retrain()` is the single entry point that rebuilds everything from the corpus (vocab, training data, fresh weights, BPE merges, UI ranges). Global mutable state (`vocab`, `w2i`, `data`, `net`, `losses`, `epoch`, `training`, …) is declared near `CTX`.
- `loop()` runs via `requestAnimationFrame`: redraws the 3D embedding every frame and, while training, runs epochs within a ~12 ms per-frame budget (also breaking every 5 epochs so charts update steadily), then calls `drawTraining()` and `onDistChange()`. Training auto-pauses at `maxEpoch` (400; "Train more" adds 200).
- `drawTraining()` redraws all canvases; draw functions bail out when their canvas is hidden (`!c.offsetParent`), so switching tabs must trigger a redraw (`showTab` does this).
- Randomness uses unseeded `Math.random` (weight init, epoch shuffling, sampling), so training runs and generated text differ between reloads. Don't expect reproducible numbers.
- All canvases go through `fitCanvas(c)` for high-DPI handling; it pins the CSS height from the `height` attribute on first use — keep that pattern when adding canvases.
- Each section ends with explanatory prose (`<h3>` subsections); keep the explanations consistent with the code when changing behaviour.
- **Two languages (English/Polish)**, switched by the EN/PL buttons in the header (choice persisted in `localStorage` key `llmLang`). See the i18n block near the top of the script. Static text: tag the element with `data-i18n="key"` (or `data-i18n-title` for tooltips); the English stays in the HTML (captured into `EN` at start-up) and the Polish goes in `PL[key]`. Dynamic strings built in JS live in `STR.en` / `STR.pl` and are read with `msg(key)` (entries with numbers are functions); use `fmt(x, d)` for numbers (Polish decimal comma). `setLang()` swaps everything and re-runs the renderers. When you change English prose or add a control, update the Polish too. The corpus/example words intentionally stay English. Don't name a callback parameter `msg`/`tr`/`fmt` — `t` is fine, it is not a global.
- The code is heavily commented for learners; match that comment style. Theme colors are CSS variables in `:root` (dark theme), but canvas drawing uses hard-coded hex values matching them.
