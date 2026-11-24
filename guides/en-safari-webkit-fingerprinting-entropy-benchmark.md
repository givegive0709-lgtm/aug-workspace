# Safari & WebKit Cross-Machine Fingerprinting Study (2026): Why Audio & Font Noise Breaks Self-Consistency

> Empirical cross-machine measurements across macOS 14, 15, and 26 (Apple Silicon and Intel) show that Safari/WebKit OfflineAudioContext and document.fonts.check have zero cross-machine entropy — while speechSynthesis voices and 2D Canvas carry real distribution entropy.

*Updated 2026-11-24 · Category: Comparisons & Benchmarks · Canonical: https://aug.jokerdao.win/blog/en-safari-webkit-fingerprinting-entropy-benchmark.html · GitHub: https://github.com/givegive0709-lgtm/aug-workspace*

---

## Direct Answer

In cross-machine Safari/WebKit testing across macOS 14, 15, and 26 (Apple Silicon and Intel), **WebAudio (`OfflineAudioContext`) and `document.fonts.check` produce zero cross-machine entropy**: every real machine yielded identical bit-for-bit audio hashes (`2880258787` for pure sine, `777214086` for a FingerprintJS-style DynamicsCompressor chain) and `36/36 true` font checks. Adding random "fingerprint noise" to audio does not make a browser look like another Mac—it breaks mathematical identity checks and immediately marks the session as synthetic.

The surfaces that actually vary across real Macs running Safari/WebKit are **OS-versioned `speechSynthesis.getVoices()` tables** (187 voices on macOS 14, 223 on macOS 15, 68 on macOS 26/27) and **2D Canvas hashes** (distinct per physical machine, stable across repeated sessions on the same machine).

## Test Methodology & Measured Environments

To separate real cross-machine entropy from folklore, the AUG WorkSpace engineering team ran a unified same-origin three-leg probe (`surface`, `audio`, and `env`) across physical Macs, macOS CI runners, and iOS real-device baselines:

- **Audio leg (`a1..a3` and `d1..d3`)**: Three consecutive same-page renders of a 4096-frame `sin(i*0.05)*0.5` `OfflineAudioContext` compared against pure JavaScript `Math.fround` (`identityF32`), plus three renders of a 10 kHz triangle oscillator through `DynamicsCompressorNode`.
- **Environment leg (`env`)**: Full `speechSynthesis.getVoices()` enumeration (polled until two consecutive reads match, capturing cold-start timeline and `voiceschanged` count) plus `document.fonts.check` across 36 font families.
- **Surface leg (`surface`)**: 2D Canvas hash, WebGL `readPixels`, static GPU vendor/renderer strings, screen/DPR, and hardware concurrency.

Measured real Safari/WebKit baselines included:
1. Physical Apple Silicon Mac (M4, macOS 26.7.1, Safari 27.0 — 2 independent sessions)
2. macOS 26.6.2 (Safari 26.6.2, Apple Silicon VM)
3. macOS 15.7.9 (Safari 26.6.1, Apple Silicon VM)
4. macOS 15.7.9 (Safari 26.6, Intel Macmini6,2 x86_64)
5. macOS 14.8.9 (Safari 17.x, sampled for OS-version voice asset comparison)
6. Physical iPhone & iPad (iOS 18.7 baseline for voices/fonts/measureText)

## Cross-Machine WebKit Entropy Table (Empirical Readings)

| Fingerprint Surface | Observed Readings Across Real Safari/WebKit Machines | Cross-Machine Entropy | Within-Machine Stability | Engineering Implication |
|---|---|---|---|---|
| **Audio sine chain (`a1..a3`)** | `2880258787` — **identical across all 7 groups** (macOS 15/26, arm64/x86, physical/VM); `identityF32=true` | **0 (Constant)** | Identical 3/3 renders | **Never add noise.** Noise breaks `identityF32` and makes a profile corpus-out. |
| **Audio compressor chain (`d1..d3`)** | `777214086` — **identical across all 7 groups** | **0 (Constant)** | Identical 3/3 renders | Same conclusion: real Safari machines agree bit-for-bit. |
| **`document.fonts.check` (36 families)** | `36/36 true` across all 7 macOS groups (and `37/37 true` on iOS 18.7) | **0 (Constant)** | Stable | WebKit's permissive check exposes zero cross-machine differentiation here. |
| **`speechSynthesis.getVoices()` count** | **187** (macOS 14.8.9) · **223** (macOS 15.7.9, identical on arm64 & x86) · **68** (macOS 26.6.2 & 26.7.1 / iOS 18.7) | **High (OS-version driven)** | Stable after warm-up | Entropy comes from **OS voice asset version**, not CPU architecture. Cold-start returns `[]` → full list + **exactly 1** `voiceschanged` event (~83–86 ms on real Safari). |
| **`speechSynthesis` voice set diff** | Safari 27.0 vs. 26.6.2: both 68 entries, **18-entry symmetric set diff**; ordering follows locale (`en-US` starts with `Majed`, `zh-CN` starts with `婷婷`) | **High (Version + Locale)** | Stable | Fixed-index probes break across locales; match by voice URI multiset and version. |
| **2D Canvas hash** | Physical M4: `4169334980` (identical across 2 sessions); CI Macs: `3076071988`, `3352399334`, `1740524260` | **High (Per-machine)** | Identical across sessions | Genuine machine-level discriminator; must remain session-stable per profile. |
| **WebGL `staticGPU` & `readPixels`** | `Apple Inc. / Apple GPU` + `WebKit WebGL`, `readPixels=1676974080` across all GPU-backed Macs; `null`/`0` on headless Intel VM without GPU | **~0 (GPU-backed Macs)** | Stable | Missing WebGL signals a headless VM rather than a consumer Mac. |

## Three Pitfalls Most Antidetect Browsers Get Wrong

### 1. Adding Random Audio Noise ("More Noise = Safer" Myth)
Many Chromium-based and spoofed browsers inject per-render or per-session random jitter into `OfflineAudioContext`. In our within-page three-render stability check, real Safari returns the exact same hash (`2880258787` / `777214086`) every time and matches pure IEEE-754 `fround` math. Per-render jitter fails same-page stability immediately; even seeded audio noise breaks mathematical identity against real WebKit.

### 2. Ignoring `SpeechSynthesisVoice` C++ Brand Checks
If a browser overrides `speechSynthesis.getVoices()` by returning synthetic JavaScript objects created with `Object.create(SpeechSynthesisVoice.prototype)`, `instanceof SpeechSynthesisVoice` passes—but assigning `utterance.voice = v` throws a **TypeError** inside WebKit's C++ brand check! Passing through native host voice objects whenever a matching `voiceURI` exists preserves both real version tables and native `SpeechSynthesisUtterance` compatibility.

### 3. Treating Voice Order as Fixed Across Languages
Real macOS WebKit re-orders `speechSynthesis.getVoices()` according to the active locale context. Assuming a hardcoded array index works across English and Chinese profiles leads to false positives and unnatural spoofing artifacts.

## How AUG WorkSpace Applies These Findings

AUG WorkSpace applies a strict **no-gratuitous-noise** architecture:
- **Zero audio noise by default**: Preserves exact WebKit `OfflineAudioContext` mathematical identity and within-page stability.
- **OS-version-matched voice tables**: Loads real-machine sampled voice tables matching the profile's claimed macOS/Safari family (14 → 187, 15 → 223, 26/27 → 68), reproduces Safari's authentic cold-start (`[]` → populated + single `voiceschanged`), and passes through native `SpeechSynthesisVoice` instances where available on host.
- **Session-stable per-profile identity**: Each profile maintains a persistent, isolated `WKWebsiteDataStore` on native macOS WebKit.

## Frequently Asked Questions

### Does adding audio fingerprint noise improve anti-detection on macOS Safari?

No. Empirical testing across macOS 15 and macOS 26 (both Apple Silicon and Intel) shows real Safari produces identical bit-for-bit OfflineAudioContext hashes (`2880258787` / `777214086`). Adding audio noise breaks within-page stability and mathematical identity checks, making the browser stand out from real Macs.

### Why does speechSynthesis.getVoices() differ between Macs?

Our measurements show arm64 and x86_64 Macs on the same macOS 15.7.9 image return the exact same 223 voices (zero set difference), whereas macOS 14.8.9 returns 187 and macOS 26 returns 68. Voice table differences are driven by macOS system voice assets and locale ordering, not hardware.

### Are WebKit font checks useful for distinguishing Macs?

Testing `document.fonts.check` across 36 standard font families returned `36/36 true` on every tested macOS Safari context (and `37/37 true` on iOS 18.7). On WebKit, this check carries zero cross-machine entropy.

## References

- W3C Web Audio API — [OfflineAudioContext Specification](https://www.w3.org/TR/webaudio/#OfflineAudioContext)
- W3C Web Speech API — [SpeechSynthesis & SpeechSynthesisVoice](https://wicg.github.io/speech-api/#tts-section)
- WebKit Fingerprinting Defense — [WebKit Tracking Prevention Policy](https://webkit.org/tracking-prevention-policy/)
- AUG WorkSpace Public Research & Docs — [github.com/givegive0709-lgtm/aug-workspace](https://github.com/givegive0709-lgtm/aug-workspace)
