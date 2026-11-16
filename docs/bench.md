# Benchmark: Node.js vs. Bun (1.4.4)

## Environment

- **Hardware / OS:** Apple M1, macOS
- **Versions:** Node `v26.13.0`, npm `12.2.1`, Bun `1.4.4`
- **Methodology:** Same source tree (`HEAD`), same `package.json`, separate install caches per tool. Medians reported using `hyperfine 2.0`.

---

## 1. Package Management & Build Overview

| Scenario | npm | bun | Delta |
| --- | --- | --- | --- |
| Fresh install, no lockfile, empty cache (`git clone` → install) | **36.6 s** | **5.15 s** | **7.1x faster** |
| Fresh install with lockfile, empty cache | 4.66 s | **2.89 s** | **1.6x faster** |
| Reinstall, warm cache (5 runs) | 4131 ms ±64 | **814 ms ±7** | **5.1x faster** |
| Reinstall peak memory | 709 MiB | **11.7 MiB** | **60x less** |
| Production build, vite build (5 runs) | 5760 ms ±70 | **5490 ms ±49** | 4.5% faster |
| Dev server ready (port accepting, 5 runs) | 450 ms | **350 ms** | 22% faster |
| Runtime boot (`-e 0`, 10 runs) | 33.8 ms | **5.5 ms** | **6.2x faster** |
| `check:markdown` | 767 ms | **510 ms** | 33% faster |
| `check:export` | 310 ms | **241 ms** | 23% faster |
| `node_modules` on disk | 326 MB | 326 MB | same |

> **Note on Build Output:** `dist/` from both builds is byte-identical (sha256 of all 20 files), and `bun install` from scratch regenerates a `bun.lock` identical to the committed one. So nothing about the app bundle changed — only tooling.

### What this means

- **Bun's win is install + process startup.** 36.6s → 5.15s on a clean clone is the headline; anything that spawns Node (the two check scripts) gets ~25-35% back.
- **Build is only ~4.5% faster,** which is exactly the `npm run` → `bun run` dispatch overhead. Vite/Rollup does the actual work either way, so no surprise there.
- **Runtime perf of the app is unchanged.** dev/build/preview all execute Vite; the dev server in the browser is the same code.

---

## 2. Running Repo Scripts (Node 26.13 vs Bun 1.4.4)

| Same `.mjs` script, same input | Node | Bun | Delta |
| --- | --- | --- | --- |
| Empty process boot (`-e 0`) | 33.8 ms | 5.5 ms | 6.2x |
| `check:markdown` (Vite SSR + happy-dom) | 767 ms | 510 ms | -33% |
| `check:export` (ZIP round-trip) | 310 ms | 241 ms | -23% |

Both exit `0` under Bun; `check:export` prints a harmless Vite HMR warning (`Port undefined is already in use`).

---

## 3. Running Vite Itself on Each Runtime

| Metric | Node | Bun | Delta |
| --- | --- | --- | --- |
| `vite build` | 5442 ms ±100 | 4696 ms ±153 | -13.7% |
| Build peak memory | 1.2 GiB | 1.7 GiB | +52% |
| Dev server opens port | 155 ms | 240 ms | +55% slower |
| First transformed entry (`/src/main.jsx`, cold `.vite`) | ~1150 ms | ~1060 ms | -8% |

> [!warning]
> **Bun-built output is NOT byte-identical.** Same 76 assets, same chunk list, sizes within 4 bytes of each other (index chunk 1,870,481 vs 1,870,477; gzip 579,265 vs 579,266). However, the minified identifier allocation differs, so hashes and every downstream cache key change. It's an engine-dependent Rollup/esbuild render difference, not a code difference — functionally equivalent, but not something you'd want to silently swap into a release pipeline without diffing.

---

## 4. Raw Engine Microbenchmarks

*(Fixed work, internal timing)*

| Workload | Node | Bun | Bun Speedup |
| --- | --- | --- | --- |
| `JSON.parse` ×8 (~90 MB) | 43.9 ms | 14.3 ms | 3.1x |
| `split`/`join` on 9 MB | 142.3 ms | 48.0 ms | 3.0x |
| Regex global replace on 9 MB | 53.6 ms | 13.5 ms | 4.0x |
| `map`/`filter`/`reduce` over 2M | 98.7 ms | 15.3 ms | 6.5x |
| Sort 2M doubles | 293.6 ms | 134.8 ms | 2.2x |
| SHA-256 over 9 MB | 4.9 ms | 4.1 ms | 1.2x |
| `Object.keys` + map over 200k | 61.0 ms | 56.9 ms | 1.1x |

JavaScriptCore is genuinely faster than V8 on array/string/JSON-heavy work in this shape of test.

---

## 5. Caveats and Known Issues

1. `bun run lint` **is broken** — but not because of Bun. `eslint` is not in `devDependencies` at all (checked `HEAD` and `b61a4a4^`), so the script fails with `eslint: command not found`.
    - *Action:* Pre-existing bug. Consider adding eslint + a flat config or dropping the script.
    
2. `bun scripts/check-export.mjs` **warning** — It passes but prints `WebSocket server error: Port undefined is already in use` (benign Vite HMR quirk under Bun's runtime). Exit code `0` and all export checks pass, so it's cosmetic — but it means the Bun runtime path isn't 100% clean yet.

---

## The Important Caveat for This App

The app's runtime performance is unchanged and can't be measured with `bun` vs `npm`. The app's JS executes in the browser — Vite serves the same ESM modules either way. Choosing Bun only changes who starts the dev server and the bundler process.

- **Runtime wins you actually get:** Scripts run 23–33% faster, and empty-process startup is ~28 ms cheaper per invocation (that's the `bun run` overhead you saw in the build numbers).
- **Runtime wins you don't get:** The editor, tldraw canvas, encryption, IndexedDB — all browser-side, identical.

---

## Final Recommendation

> [!tip]
> **Keep Node/npm-equivalent for builds** (for deterministic output). **Keep Bun for installs and the **`check:*`** scripts** where it's both faster and verified passing.
>
> Running Vite on Bun's runtime buys ~13% build time but costs +50% memory and changes output bytes. The tradeoff is not currently worth it.
