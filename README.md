# Apple Silicon LLM Memory Calculator

**[🚀 Open the calculator →](https://meirm.github.io/apple_silicon_llm_memory/)** — one HTML file, no install. *(First-time setup on GitHub: Settings → Pages → Deploy from a branch → `main` / `/` root. Until enabled, you can [view the source of `index.html`](https://github.com/meirm/apple_silicon_llm_memory/blob/main/index.html) and download it.)*

A single-file browser tool that answers one question: **"Will this LLM actually fit on my Mac?"**

It fetches popular Hugging Face text-generation models, estimates total memory need (weights + KV cache + runtime overhead + safety margin), and compares it against a Mac's unified memory configuration. It also gives a rough decode-speed hint derived from the chip's memory bandwidth.

No install, no build, no server — everything runs client-side in one HTML file.

---

## Quick start

**Option 1 — local file** (works offline for the math; fetching the model list needs internet):

```bash
git clone <this repo>
open index.html   # macOS (or just double-click it)
```

**Option 2 — GitHub Pages:** after pushing, enable Pages for the repo (Settings → Pages → deploy from branch) and the calculator is available as a public URL.

---

## How to use it

### 1. Pick your Mac

The **Mac preset** dropdown loads with a best guess for *your* machine:

- On a Mac, the tool reads your GPU's WebGL renderer string (e.g. `ANGLE Metal Renderer: Apple M1 Pro`) and preselects the matching chip preset — so an M1 Pro owner gets the M1 Pro preset, not a generic default.
- **Browsers cannot read your actual RAM size** (macOS unified memory is not exposed to JavaScript), so the tool preselects the *smallest* RAM config for the detected chip. Set **Unified memory GB** to your real configuration (16, 32, 64…).
- Safari hides the exact chip (`Apple GPU`), so it falls back to a reference preset — just pick your chip in the dropdown.
- Chips without a preset (M2, non-Ultra M3) also fall back to the reference entry with a hint.

Choose any preset manually to override auto-detection. Every preset supplies unified memory and the chip's memory bandwidth.

### 2. Pick (or enter) a model

- **Fetch popular models** — lists the 50 most popular Hugging Face text-generation models (data straight from the public Hugging Face API).
- **Search** — type e.g. `qwen`, `llama`, `gemma` and press Search.
- Click any model in the list to select it.
- Or type a parameter count into **Manual model params B** (e.g. `7`) and click **Use manual model** — useful when the API is blocked or a repo is gated.

Parameter counts come from safetensors metadata or the model config when available; otherwise they are estimated from the repo name. The **Detected model architecture** table shows whether the architecture is from the model's config or estimated.

### 3. Tune the assumptions

| Control | What it does | Default |
|---|---|---|
| Unified memory GB | Your Mac's physical RAM | from preset |
| Memory bandwidth GB/s | Chip bandwidth (drives the speed hint) | from preset |
| System reserve GB | Memory kept for macOS + apps | 8 |
| Safety margin % | Padding on top of the estimate | 15 |
| Context tokens | Slider, 4k → 1M. KV cache grows linearly with this | 128k |
| Quantization | Bits per weight (FP16 → Q2) | Q4 |
| KV precision | FP16 / 8-bit / 4-bit KV cache | FP16 |
| Runtime overhead GB | Graph buffers, Metal allocations, misc runtime | 3 |
| Batch / parallel seq | Sequences served in parallel (multiplies KV cache) | 1 |

### 4. Read the verdict

- **Red / "Probably will not fit"** — estimated need exceeds usable memory (>105%).
- **Yellow / "Borderline"** — 85–105% of usable memory; real-world usage will likely swap.
- **Green / "Fits with headroom"** — under 85% of usable memory.
- **Bandwidth hint (`~x.x tok/s*`)** — a rough decode-speed estimate from the chip's bandwidth (`bandwidth × 0.55 ÷ weight size`). It is a heuristic, not a benchmark.

The **Memory breakdown** table shows weights, KV cache, runtime buffers, margin, and total; the **Detected model architecture** table shows layers, hidden size, attention/KV heads, and the GQA ratio used for the KV calculation.

---

## Presets

Bandwidths are Apple's official launch figures where Apple published them:

| Group | Coverage |
|---|---|
| M1 official | M1 (68 GB/s), M1 Pro (200 GB/s), M1 Max (400 GB/s) — 16 → 64 GB |
| M4 / M5 / M6 | Full base/Pro/Max lines; M6 Pro–Ultra are **projected placeholders** (M5-class bandwidth, editable) |
| M5 Ultra | Reported/inferred (2 × M5 Max); treat as unofficial until Apple confirms |
| M3 Ultra | Official ~819 GB/s, 96 → 512 GB |
| Custom | Fully manual memory + bandwidth |

`TBD`/projected presets carry editable bandwidth values — if you have a confirmed figure, just change the **Memory bandwidth** field.

### Adding or editing a preset

Presets live in the `macs` array near the top of the `<script>` block in `index.html`:

```js
{group:'M1 official', name:'M1 Pro — 32GB', chip:'M1 Pro', mem:32, bw:200, source:'official', note:'…'},
```

- `chip` is matched against the WebGL renderer string for auto-detection (use exact tier names like `M1 Pro`, `M4 Max`).
- `source` is one of `official`, `reported/inferred`, `projected`, `reference`, `manual`.
- Keep one entry per chip + RAM step; auto-detection picks the smallest RAM config of the matched chip.

---

## The math (and its limits)

```
weights = params × bits-per-weight / 8 × quant overhead factor
KV cache = 2 × layers × context × hidden size × KV bytes × GQA ratio × batch
total = weights + KV + runtime overhead + safety margin (15% default)
usable = unified memory − system reserve
```

Real llama.cpp / MLX / Ollama memory usage can differ because of:

- graph buffers and Metal allocation granularity,
- actual quant format differences (GGUF/MLX quants vary),
- MoE models (only *active* experts may be resident),
- OS memory pressure and the swap policy.

So treat the result as a planning estimate, not a guarantee.

## Privacy & network use

All calculations run locally in your browser. The only network request goes to `huggingface.co`'s public JSON API to list models. If that request is blocked (CORS restrictions, ad blockers, offline), the error banner explains it and manual model params still work.

## License

MIT.
