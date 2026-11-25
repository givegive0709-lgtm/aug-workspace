# AUG WorkSpace — macOS-Native WebKit Antidetect Browser

[![Website](https://img.shields.io/badge/Website-aug.jokerdao.win-7c3aed)](https://aug.jokerdao.win/)
[![Platform](https://img.shields.io/badge/Platform-macOS%2026%2B-000000)](https://aug.jokerdao.win/)
[![Engine](https://img.shields.io/badge/Engine-Native%20WebKit%20%2F%20Safari-0ea5e9)](https://aug.jokerdao.win/blog/en-macos-native-webkit-antidetect-browser.html)
[![Automation](https://img.shields.io/badge/Automation-HTTP%20API%20%7C%20SDK%20%7C%20MCP-10b981)](https://aug.jokerdao.win/blog/en-antidetect-browser-mcp-ai-agents.html)

[English](./README.md) | [中文说明](./README_CN.md) | [Official Website](https://aug.jokerdao.win/) | [English Guides](https://aug.jokerdao.win/blog/en.html) | [llms.txt](https://aug.jokerdao.win/llms.txt)

> **AUG WorkSpace** is a **macOS-native WebKit antidetect browser** (not a patched Chromium wrapper). Each browser profile runs in an isolated OS process with its own persistent `WKWebsiteDataStore`, real Apple WebKit/Safari identity, proxy-follow egress alignment, local-first storage, and built-in **HTTP API, Python/Java SDK, and MCP (Model Context Protocol)** automation without exposing CDP or `navigator.webdriver`.

This public repository hosts AUG WorkSpace's **public technical documentation, reproducible Safari/WebKit fingerprinting benchmarks, and API/SDK/MCP integration references**.

---

## Why Native WebKit Instead of Patched Chromium?

| Dimension | AUG WorkSpace (Native WebKit) | Typical Chromium Antidetect Browsers |
|---|---|---|
| **Engine** | Apple system WebKit (Safari identity) | Patched Chromium (Blink) + JS/C++ spoofing layer |
| **Profile Isolation** | 1 OS process + 1 persistent `WKWebsiteDataStore` per profile | Separate Chromium user-data dirs |
| **Audio & Fonts** | Authentic WebKit consistency (no gratuitous audio noise that breaks `identityF32`) | Often injects random noise per render/session |
| **Automation** | Local HTTP control plane + per-profile JSON-RPC bridge + **stdio MCP** (zero CDP) | CDP / Puppeteer / Selenium |
| **Storage Model** | **Local-first** on macOS (optional E2E encrypted team sync) | Cloud-centric or hybrid |
| **Platform** | macOS 26+ (Apple Silicon & Intel) | Windows / macOS / Linux |

---

## Empirical Safari & WebKit Fingerprinting Findings (2026)

From our cross-machine WebKit probe across macOS 14, 15, and 26 (Apple Silicon & Intel, physical & VM):

| Surface | Observed Real Safari Readings | Cross-Machine Entropy | Takeaway |
|---|---|---|---|
| `OfflineAudioContext` sine (`a1..a3`) | `2880258787` across all 7 tested contexts (`identityF32=true`) | **0 (Constant)** | Do **not** inject audio noise; it breaks mathematical self-consistency. |
| `OfflineAudioContext` compressor (`d1..d3`) | `777214086` across all 7 tested contexts | **0 (Constant)** | Bit-for-bit identical across OS versions and CPU architectures. |
| `document.fonts.check` (36 families) | `36/36 true` across all 7 macOS contexts | **0 (Constant)** | WebKit permissive font check carries zero cross-machine entropy. |
| `speechSynthesis.getVoices()` | `187` (macOS 14) · `223` (macOS 15, arm64 == x86) · `68` (macOS 26/27) | **High (OS-versioned)** | Driven by OS voice asset version + locale ordering; synthetic objects fail C++ brand checks on `utterance.voice`. |
| 2D Canvas | Unique per physical Mac, stable across sessions | **High (Per-machine)** | Must stay session-stable per profile. |

Read the full reproducible study: **[Safari & WebKit Cross-Machine Fingerprinting Study (2026)](https://aug.jokerdao.win/blog/en-safari-webkit-fingerprinting-entropy-benchmark.html)**.

---

## Zero-CDP Automation for AI Agents (Python SDK & MCP)

```python
from aug_sdk import Client

client = Client()  # reads AUG_API_KEY; local control plane at http://127.0.0.1:8311
profile = client.control.create_profile(name="agent-us-01", geo="US", no_proxy=True)
pid = profile["created"][0]["id"]
client.control.launch_profile(pid, url="https://example.com")

browser = client.browser(pid)  # auto-resolves per-profile bridge port (zero CDP)
browser.wait_for("h1")
page_map = browser.page_map()  # structured visible interactive elements for AI agents
png = browser.screenshot()     # native WKWebView.takeSnapshot
```

- **Full API Reference**: [docs/API.md](docs/API.md) · [Online HTML](https://aug.jokerdao.win/docs/api.html)
- **Python & Java SDK**: [docs/SDK.md](docs/SDK.md) · [Online HTML](https://aug.jokerdao.win/docs/sdk.html)
- **MCP Automation Guide**: [Antidetect Browser with Built-in MCP & HTTP API](https://aug.jokerdao.win/blog/en-antidetect-browser-mcp-ai-agents.html)

---

## Technical Guides & Benchmarks

- **[Safari & WebKit Cross-Machine Fingerprinting Study (2026): Why Audio & Font Noise Breaks Self-Consistency](https://aug.jokerdao.win/blog/en-safari-webkit-fingerprinting-entropy-benchmark.html)** — Empirical cross-machine measurements across macOS 14, 15, and 26 (Apple Silicon and Intel) show that Safari/WebKit OfflineAudioContext and document.fonts.check have zero cross-machine entropy — while speechSynthesis voices and 2D Canvas carry real distribution entropy. ([local copy](guides/en-safari-webkit-fingerprinting-entropy-benchmark.md))
- **[Best macOS-Native WebKit Antidetect Browser (2026): Native Safari Engine vs. Chromium Spoofing](https://aug.jokerdao.win/blog/en-macos-native-webkit-antidetect-browser.html)** — Most antidetect browsers on macOS are patched Chromium wrappers that spoof Safari or macOS fingerprints in JavaScript. AUG WorkSpace uses Apple's native WebKit engine with per-process WKWebsiteDataStore isolation, proxy-follow egress, and zero-CDP HTTP/SDK/MCP automation. ([local copy](guides/en-macos-native-webkit-antidetect-browser.md))
- **[Antidetect Browser with Built-in MCP & HTTP API (2026): Zero-CDP Automation for AI Agents](https://aug.jokerdao.win/blog/en-antidetect-browser-mcp-ai-agents.html)** — Most browser automation tools rely on Chrome DevTools Protocol (CDP), exposing automation signals on high-security sites. Learn how AUG WorkSpace automates isolated macOS WebKit profiles via a local HTTP control plane, per-profile JSON-RPC bridge, Python SDK, and native stdio MCP server. ([local copy](guides/en-antidetect-browser-mcp-ai-agents.md))
- **[AdsPower & Multilogin Alternatives for macOS (2026): Chromium Forks vs. Native WebKit](https://aug.jokerdao.win/blog/en-adspower-multilogin-alternatives-macos.html)** — Evaluating AdsPower and Multilogin alternatives for macOS in 2026? Most desktop antidetect browsers ship patched Chromium or Firefox forks that spoof Safari. Compare AUG WorkSpace (native macOS WebKit), GoLogin, Dolphin Anty, and Orion across engine authenticity, isolation, and automation. ([local copy](guides/en-adspower-multilogin-alternatives-macos.md))

---

## Official Links & Compliance Boundary

- Website & Download: https://aug.jokerdao.win/
- Pricing (Free tier + Paid / Team plans): https://aug.jokerdao.win/pricing
- English Content Hub: https://aug.jokerdao.win/blog/en.html
- Chinese Content Hub: https://aug.jokerdao.win/blog/
- AI-readable index: https://aug.jokerdao.win/llms.txt · https://aug.jokerdao.win/llms-full.txt
- Contact: `admin@jokerdao.win`

*Compliance note: AUG WorkSpace is a general-purpose multi-account isolation and browser automation tool for macOS. No browser guarantees immunity from platform risk checks; always operate within applicable laws and platform Terms of Service.*
