# SST Guard

## Overview

SST Guard is a Chrome extension that detects server-side tracking (SST) implementations in real time. It operates across three independent detection modalities — network request parameter analysis, first-party cookie inspection, and JavaScript window variable extraction — each backed by a dedicated machine learning model running locally via ONNX Runtime Web. A meta-classifier and a combined classifier aggregate signals across modalities for domain-level SST attribution.

---

## Repository Contents

| File | Description |
|---|---|
| `dist_chrome.zip` | Packaged Chrome extension (Manifest V3), ready to load unpacked |
| `ground-truth.csv` | Ground Truth collected from Google Tag Assistant |
| `ground-truth-labels.csv` | Classification output of SST Guard on ALL ground truth domains in `ground-truth.csv` |
| `adblock-filtered.csv.zip` | sGA classification output from our 150,000-site crawl, with per-domain results across all three detection modalities |
| `sst-domains.txt` | List of domains containing sGA, extracted from adblock-filtered.csv |

---

## Installation

**Requirements:** Google Chrome 116 or later (Manifest V3 and WASM support required).

1. Download and unzip `dist_chrome.zip`.
2. Open Chrome and navigate to `chrome://extensions`.
3. Enable **Developer mode** (toggle in the top-right corner).
4. Click **Load unpacked** and select the unzipped `dist_chrome` folder.
5. The SST Guard icon will appear in the Chrome toolbar.

---

## Extension Overview

### Detection Modalities

#### Network Request Analysis
Every outgoing HTTP request is matched against 23 query parameter patterns associated with Google Analytics 4 and Google Tag Manager (`cid`, `tid`, `en`, `gcd`, `sid`, `_p`, `uafvl`, `ep.user_agent`, and 15 others). A logistic regression model classifies each request individually. In parallel, features are aggregated across all requests on the current page using a logical OR operation and passed to a domain-level network model.

**Default threshold:** 0.7 (per-request), 0.4 (page-level aggregation).

#### Cookie Detection
The extension scans first-party cookies for 5 regex patterns corresponding to known GA/GTM cookie formats, including `GA1.x` session identifiers and `GS2.1` GA4 session cookies. Scans run automatically every 30 seconds and at multiple intervals on page load to capture cookies set by deferred scripts. A logistic regression model classifies the resulting feature vector.

**Default threshold:** 0.4.

#### Window Variable Inspection
A CSP-compliant external script is injected into each page to enumerate `window` properties and serialize them. The content script sends the result to the background worker, which matches values against 10 patterns covering `dataLayer` event structures, `gaGlobal` hit and visitor identifiers, and `google_tag_data` fields (container ID, browser brand strings, chrome version, platform version, architecture, and bitness).

**Default threshold:** 0.4.

### ML Architecture

The extension loads 8 ONNX models into the background service worker and runs inference via ONNX Runtime Web (WASM backend, single-threaded, SIMD enabled). Each model uses a queue to prevent concurrent session conflicts.

| Model | Input | Purpose |
|---|---|---|
| `logistic_regression_model.onnx` | 23 network features | Per-request SST classification |
| `network-request-domain.onnx` | 23 aggregated features | Page-level network SST classification |
| `logistic_regression_model_cookies.onnx` | 5 cookie features | Cookie-based SST classification |
| `logistic_regression_model_window.onnx` | 10 window var features | Window variable SST classification |
| `meta-feature-network.onnx` | 23 features | Network modality embedding for meta-classifier |
| `meta-feature-cookie.onnx` | 5 features | Cookie modality embedding for meta-classifier |
| `meta-feature-window.onnx` | 10 features | Window variable modality embedding for meta-classifier |
| `meta-classifier.onnx` | 3 probabilities | Final domain-level classification from modality embeddings |
| `combined-classifier.onnx` | 38 concatenated features | Alternative end-to-end domain-level classifier |

---

## Verifying Paper Claims

The `adblock-filtered.csv.zip` file contains SST classification results from our large-scale crawl. To verify these results against live sites using the extension:

1. Install the extension following the steps above.
2. Open `adblock-filtered.csv.zip` and identify a domain classified as sGA.
3. Navigate to that domain in Chrome.
4. Click the SST Guard toolbar icon to open the popup.
5. Observe which detection tabs report activity (Network, Cookies, Window Var, Page Network, Meta, Combined) and the associated probability scores.
6. Compare scores against the default thresholds (0.7 for network, 0.4 for all others).


Detection thresholds and an optional auto-block behaviour can be adjusted via the **Options** page, accessible from the gear icon in the popup.
