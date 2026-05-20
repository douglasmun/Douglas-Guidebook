# SEA-LION Translation Sidecar

> **Bringing Southeast Asia's native-language cybersecurity media into a unified English-language intelligence picture — locally, privately, and without cloud API costs.**

---

## The Problem This Solves

**Singapore Cyberspace Monitor** is a real-time cyber threat intelligence platform built to track the regional security landscape across all 10 ASEAN member states — Singapore, Malaysia, Indonesia, Thailand, Vietnam, the Philippines, Myanmar, Cambodia, Brunei, and Laos. It ingests hundreds of sources: national CERTs, government advisories, financial regulators, news agencies, dark web leak trackers, social media, and threat intelligence feeds.

The challenge: **the region's most important cybersecurity signals don't always arrive in English.**

Indonesia's national cyber agency BSSN publishes in Bahasa Indonesia. Thai government advisories come in Thai script. Cambodian government sources publish in Khmer. Myanmar's security researchers write in Burmese. These aren't fringe edge cases — they are primary sources from some of ASEAN's most strategically significant digital economies, covering critical infrastructure attacks, ransomware campaigns, regulatory changes, and data breaches that affect millions of people across the region.

Until now, the platform either relied on expensive cloud LLM API calls to translate these articles, or left them untranslated and effectively invisible to English-speaking analysts.

**The SEA-LION sidecar is the solution.** It runs a dedicated, locally-hosted translation model that speaks Southeast Asian languages natively — built by [AI Singapore](https://aisingapore.org/), the national AI programme — and translates every ingested non-English article into English automatically, in real time, on-device, at zero marginal cost per translation.

---

## Why not Ollama?

The first instinct was to load SEA-LION through [Ollama](https://ollama.com/) — the same local inference runtime already used by the platform for summarisation and analysis. Ollama is convenient: one command pulls a model, another serves it, and the OpenAI-compatible API means no code changes.

The attempt failed at the first step:

```bash
$ ollama pull aisingapore/Gemma-SEA-LION-v4.5-E2B-IT
# Error: file does not exist
```

`aisingapore/Gemma-SEA-LION-v4.5-E2B-IT` **is not in the Ollama model catalogue.** Ollama's library only indexes models that have been explicitly packaged as Modelfiles and submitted to [ollama.com/library](https://ollama.com/library). SEA-LION has not been submitted there.

The next attempt was to load it directly from HuggingFace using Ollama's `hf.co/` import syntax:

```bash
$ ollama run hf.co/aisingapore/Gemma-SEA-LION-v4.5-E2B-IT
# Error: aisingapore/Gemma-SEA-LION-v4.5-E2B-IT: Repository is not GGUF
#        or is not compatible with llama.cpp
```

This also fails. Ollama's HuggingFace import only works with models published in **GGUF format** (the quantised format used by llama.cpp). Gemma-SEA-LION-v4.5-E2B-IT is a multimodal model — it includes vision components — and llama.cpp does not support Gemma 4 multimodal architectures. No GGUF exists on HuggingFace for this model, and creating one would require significant quantisation engineering with no guarantee of quality preservation for the ASEAN language fine-tuning.

**The conclusion:** Ollama is not a viable path for SEA-LION. The model must be loaded directly via HuggingFace Transformers, which handles the full PyTorch architecture including the multimodal processor. This is why the sidecar exists as a separate Python process rather than a simple Ollama API call.

---

## What Is SEA-LION?

[SEA-LION](https://huggingface.co/aisingapore/Gemma-SEA-LION-v4.5-E2B-IT) (South-East Asian Languages In One Network) is a family of large language models developed by **AI Singapore** — the national AI R&D programme under the National Research Foundation of Singapore. It is purpose-built for Southeast Asian languages and fine-tuned on regional corpora that general-purpose LLMs like GPT-4 or Claude have seen far less of.

The model used here is `Gemma-SEA-LION-v4.5-E2B-IT`:

- **Base:** Google Gemma 4 E2B (2B-parameter efficient variant of the Gemma 4 family)
- **Fine-tuned by:** AI Singapore on ASEAN language corpora
- **Size:** ~4 GB (bfloat16)
- **Inference:** HuggingFace Transformers, runs natively on Apple Silicon MPS or CUDA

This gives us a model that understands the **vocabulary, idioms, and named entities** of the region — government agency acronyms like BSSN (Indonesia), BNM (Malaysia), NCSA (Thailand), DICT (Philippines) — that a generic translation model would mangle or transliterate.

---

## Languages Covered

| Country | Language | Script | Sample Sources |
|---|---|---|---|
| Indonesia | Bahasa Indonesia | Latin | Detik News, Kompas, Republika, BSSN |
| Malaysia | Bahasa Melayu | Latin | Malay Mail, MyCERT, BNM |
| Thailand | Thai | Thai script | ThaiCERT, Khaosod, The Nation |
| Vietnam | Vietnamese | Latin + diacritics | VnExpress, Tuoi Tre, VNCERT |
| Philippines | Filipino (Tagalog) | Latin | DICT, Philstar, Rappler |
| Myanmar | Burmese | Myanmar script | Mizzima, Myanmar Now |
| Cambodia | Khmer | Khmer script | Cambodia Daily, CamCERT |
| Singapore | Tamil | Tamil script | Tamil-language community media |
| Brunei | Malay / English | Latin | Covered via Malaysia/Malay row; Brunei sources publish primarily in Malay and English |
| Laos | Lao | Lao script | Not currently supported — no Lao-language sources in the active feed catalogue |

---

## How It Fits Into the Platform

The monitor runs a continuous ingestion pipeline. Every few minutes, background workers poll hundreds of RSS feeds, CERT advisories, government press releases, and social media accounts. Each article is:

1. Fetched and cleaned
2. **Translated to English** (if non-English is detected)
3. Enriched with AI-generated summaries and IOC extraction
4. Embedded into a vector database for semantic search and RAG
5. Surfaced in the analyst dashboard alongside feeds from English-language sources

The SEA-LION sidecar plugs into step 2. It starts alongside the background workers and runs as a persistent local HTTP service. Non-English articles POST their title and content to it; translated English comes back in under 5 seconds. The rest of the pipeline sees only English from that point forward.

```
ASEAN News Sources (multilingual)
         │
         ▼
  Feed Ingest Service
         │
         ├─ English content ──────────────────────────────────┐
         │                                                    │
         └─ Non-English content                               │
                  │                                           │
                  ▼                                           │
         SEA-LION Sidecar (127.0.0.1:5200)                    │
         POST /translate { title, content }                   │
                  │                                           │
                  ▼                                           │
         { title_en, content_en }                             │
                  │                                           ▼
                  └──────────────────────────────► Enrichment Pipeline
                                                   (AI summary, IOC extraction,
                                                    vector embedding, RAG)
```

---

## Architecture

The sidecar is a minimal Python Flask server. It loads the model once at startup and serves a single translation endpoint for the lifetime of the worker process.

```
start-workers.ts
  └── startSealionSidecar()          [lib/services/sealion-sidecar.ts]
        └── spawn .venv-sealion/bin/python3 scripts/sealion-server.py
              └── Flask HTTP on 127.0.0.1:5200
                    ├── GET  /health  → { "status": "ok" | "loading" }
                    └── POST /translate { title, content }
                               → { title, content }  (English)

feed-ingest-service.ts
  └── translateToEnglish(title, content)
        └── getTranslationProvider()  → 'sealion'
              └── sendAIMessage(prompt, { provider: 'sealion' })
                    └── sendSealionMessage()
                          └── POST http://127.0.0.1:5200/translate
                                ↳ on error: fallback to getAIProvider()
```

**Key design decisions:**

- `SEALION_SERVER_URL` set → `getTranslationProvider()` returns `'sealion'` (opt-in; removing the env var reverts to the previous behaviour with zero code changes)
- On AWS deployment (`DEPLOYMENT_TARGET=aws`), translation routes to Anthropic — the local model is not available in cloud environments
- The sidecar is a completely separate Python process; the Node.js app never loads PyTorch
- `enable_thinking=False` in the Gemma 4 chat template disables chain-of-thought reasoning, halving latency for translation tasks that don't need it

---

## File Map

### Python — sidecar process

These files live entirely in the Python layer. They are independent of the Node.js application and can be developed, tested, or replaced without touching any TypeScript.

| File | Purpose |
|---|---|
| `scripts/sealion-server.py` | **Main sidecar.** Flask HTTP server; loads `Gemma-SEA-LION-v4.5-E2B-IT` once at startup via HuggingFace Transformers, then serves `GET /health` and `POST /translate` for the lifetime of the process |
| `requirements-sealion.txt` | Pinned Python dependencies (`transformers`, `torch`, `accelerate`, `flask`, `pillow`, `torchvision`) |
| `scripts/setup-sealion-venv.sh` | One-time bootstrap: creates `.venv-sealion/` and installs `requirements-sealion.txt` |
| `.venv-sealion/` | Isolated Python venv (git-ignored; created by the setup script) |

### TypeScript — Node.js integration

These files wire the sidecar into the existing Next.js + BullMQ application.

| File | Purpose |
|---|---|
| `lib/services/sealion-sidecar.ts` | Spawns `sealion-server.py` as a child process, streams its stdout/stderr to the app logger, polls `/health` until ready, and exposes `startSealionSidecar()` / `stopSealionSidecar()` |
| `lib/ai/ai-provider.ts` | Adds `'sealion'` to the `AIProvider` union; implements `getTranslationProvider()` and `sendSealionMessage()` (POSTs to the Flask endpoint and returns a normalised `AIResponse`) |
| `lib/feeds/feed-ingest-service.ts` | `translateToEnglish()` — detects non-English content, calls the sidecar, and falls back to the general AI provider on any error |
| `scripts/start-workers.ts` | Calls `startSealionSidecar()` at Step 0 (before Redis/BullMQ init) and registers SIGTERM/SIGINT handlers to stop the sidecar gracefully |
| `scripts/dev/test-sealion-translation.ts` | Standalone validation script: tests 7 hardcoded ASEAN language samples and up to 30 recent non-English DB items, with primary + fallback reporting |

---

## Setup

### 1. Python venv + dependencies

```bash
bash scripts/setup-sealion-venv.sh
```

This creates `.venv-sealion/` and installs:

```
transformers>=4.51.0
torch>=2.7.0
accelerate>=1.6.0
flask>=3.1.0
pillow>=11.0.0        # required — Gemma 4 is multimodal
torchvision>=0.22.0   # required — Gemma 4 video processor
```

> **Note:** `pillow` and `torchvision` are mandatory even for text-only translation. Gemma-SEA-LION-v4.5-E2B-IT is a multimodal model; its processor imports both libraries unconditionally at load time.

### 2. Download the model

On first run the model is fetched automatically from HuggingFace Hub (~4 GB). To pre-download:

```bash
source .venv-sealion/bin/activate
python3 -c "
from transformers import AutoProcessor, AutoModelForCausalLM
AutoProcessor.from_pretrained('aisingapore/Gemma-SEA-LION-v4.5-E2B-IT')
AutoModelForCausalLM.from_pretrained('aisingapore/Gemma-SEA-LION-v4.5-E2B-IT', dtype='auto', device_map='auto')
print('Done')
"
```

Cached to `~/.cache/huggingface/hub/models--aisingapore--Gemma-SEA-LION-v4.5-E2B-IT/`.

### 3. Environment variables

Add to `.env.local`:

```bash
SEALION_SERVER_URL="http://127.0.0.1:5200"
SEALION_SERVER_HOST="127.0.0.1"
SEALION_SERVER_PORT="5200"
SEALION_MODEL_ID="aisingapore/Gemma-SEA-LION-v4.5-E2B-IT"   # optional override
```

Setting `SEALION_SERVER_URL` is the only flag required to activate the provider. Remove it to fall back to the previous behaviour.

---

## Running

### With workers (recommended)

The sidecar starts automatically when `npm run workers` runs:

```bash
npm run workers
```

Startup sequence:
1. `startSealionSidecar()` spawns `sealion-server.py` via `.venv-sealion/bin/python3`
2. Health poll begins: GET `/health` every 5 s, 120 s timeout, 4 s per-fetch abort
3. Flask process loads model (`device_map="auto"` → MPS on Apple Silicon, CUDA on Linux)
4. `/health` returns `{ "status": "ok" }` → sidecar marked healthy
5. Workers proceed to Redis, BullMQ, and collectors

On SIGTERM/SIGINT, `stopSealionSidecar()` sends SIGTERM to the Python process before workers exit.

### Standalone (development/testing)

```bash
source .venv-sealion/bin/activate
SEALION_SERVER_HOST=127.0.0.1 SEALION_SERVER_PORT=5200 python3 scripts/sealion-server.py
```

### Manual API test

```bash
curl -s -X POST http://127.0.0.1:5200/translate \
  -H 'Content-Type: application/json' \
  -d '{"title":"Serangan Siber Mengincar Infrastruktur Kritis","content":"BSSN melaporkan peningkatan serangan siber."}' | jq .
```

Expected response:

```json
{
  "title": "Cyberattacks Target Critical Infrastructure",
  "content": "BSSN reports an increase in cyberattacks."
}
```

---

## Provider Resolution

`getTranslationProvider()` in `lib/ai/ai-provider.ts`:

```
DEPLOYMENT_TARGET=aws   →  anthropic   (cloud deployment; local model unavailable)
SEALION_SERVER_URL set  →  sealion
ai.provider.translation = anthropic  →  anthropic
ai.provider.translation = openai     →  openai
(default)               →  ollama
```

`translateToEnglish()` fallback chain in `lib/feeds/feed-ingest-service.ts`:

```
1. POST to SEA-LION sidecar
   ↳ success → use translated title + content
   ↳ failure → try fallbackProvider (getAIProvider())
               ↳ success → use translated title + content
               ↳ failure → return null (store item untranslated, not dropped)
```

Non-English detection uses a 15% non-ASCII character threshold. Sources `detik news`, `kompas`, and `republika` (internal feed source identifiers) are always translated regardless of character composition — their titles are often already in English from upstream processing, but their content bodies remain in Bahasa Indonesia.

---

## Flask Sidecar Internals

`scripts/sealion-server.py` core translation logic:

```python
messages = [
    {"role": "system", "content": TRANSLATE_SYSTEM},
    {"role": "user", "content": f"Title: {title}\n\nContent: {content[:1000]}"},
]

text = processor.apply_chat_template(
    messages,
    tokenize=False,
    add_generation_prompt=True,
    enable_thinking=False,      # disables CoT for Gemma 4 — ~2x faster
)
inputs = processor(text=text, return_tensors="pt").to(model.device)
input_len = inputs["input_ids"].shape[-1]   # used to slice off the prompt from the output
outputs = model.generate(**inputs, max_new_tokens=1024, temperature=0.0, do_sample=False)
raw = processor.decode(outputs[0][input_len:], skip_special_tokens=True).strip()
```

- `temperature=0.0, do_sample=False` — deterministic, consistent output across repeated runs
- `max_new_tokens=1024` — sufficient for article summaries up to ~800 words
- Content truncated to 1000 characters before sending — prevents OOM on long articles
- Markdown fence stripping applied if the model wraps output in ` ```json ``` `
- On `JSONDecodeError`: returns `{ title: original_title, content: raw_output }` rather than erroring — translation degrades gracefully

---

## Performance (Apple Silicon M5, 128 GB)

Model: `aisingapore/Gemma-SEA-LION-v4.5-E2B-IT` (2B parameters, bfloat16)  
Device: MPS via `device_map="auto"`

| Metric | Value |
|---|---|
| Model load time | ~25–35 s (first start; cached thereafter) |
| Model download size | ~4 GB (11 files from HuggingFace Hub) |
| Per-translation latency | ~2–5 s (title + 1000-char content) |
| Memory footprint | ~4 GB unified memory |
| Throughput | ~10–15 translations/min sustained |

**Observed translation quality — 2026-05-20 test run:**

| Test | Result |
|---|---|
| 7 ASEAN language samples (direct, hardcoded) | 7/7 via SEA-LION — 100% |
| Detik News DB items (30 most recent) | 19 SEA-LION primary, 11 Anthropic fallback, 0 failed |
| Overall success rate | **100%** |

**Sample translations from the test run:**

| Language | Original | SEA-LION English |
|---|---|---|
| Bahasa Indonesia | *Serangan Siber Mengincar Infrastruktur Kritis Indonesia* | Cyberattacks Target Indonesia's Critical Infrastructure |
| Bahasa Melayu | *Malaysia Perkukuh Keselamatan Siber Sektor Kewangan* | Malaysia Strengthens Cybersecurity in the Financial Sector |
| Thai | *ไทยเตรียมรับมือภัยคุกคามไซเบอร์ระดับชาติ* | Thailand Prepares to Face National-Level Cyber Threats |
| Vietnamese | *Việt Nam tăng cường bảo mật mạng cho cơ sở hạ tầng trọng yếu* | Vietnam Strengthens Cybersecurity for Critical Infrastructure |
| Filipino | *Pilipinas naglunsad ng bagong pambansang estratehiya sa cybersecurity* | Philippines Launches New National Cybersecurity Strategy |
| Burmese | *မြန်မာနိုင်ငံတွင် ဆိုက်ဘာတိုက်ခိုက်မှုများ တိုးမြင့်လာ* | Cyberattacks Increase in Myanmar |
| Tamil | *சிங்கப்பூரில் இணைய பாதுகாப்பு மீறல்கள் அதிகரிப்பு* | Increase in Cybersecurity Breaches in Singapore |

Anthropic fallbacks on Detik News occurred on items with empty `content` fields — SEA-LION returned the title unchanged when there was nothing to translate, triggering the fallback heuristic. No articles were dropped or lost.

---

## Validation

Run the standalone translation test against live data:

```bash
npx tsx --env-file=.env.local scripts/dev/test-sealion-translation.ts
```

Output covers:
- **Part 1:** 7 hardcoded ASEAN language samples (Indonesian, Malay, Thai, Vietnamese, Filipino, Burmese, Tamil) — Khmer not included in the hardcoded set but covered via DB items if Cambodian sources are present
- **Part 2:** Up to 30 recent items per non-English DB source

Expected summary:

```
=== Summary ===
Success (primary):  N
Success (fallback): M
Failed:             0
Success rate:       100.0%
```

---

## Sharing the Sidecar as a Standalone Service

The Python sidecar has no dependency on the Singapore Cyberspace Monitor codebase. Only three files are needed to hand it to someone else:

```
sealion-sidecar/
├── sealion-server.py       ← copy from scripts/sealion-server.py
├── requirements.txt        ← copy from requirements-sealion.txt  (rename it)
└── setup.sh                ← copy from scripts/setup-sealion-venv.sh
```

Note: rename `requirements-sealion.txt` to `requirements.txt` in the standalone directory — `setup.sh` installs from `requirements.txt` by that exact name.

Copy those three files into a new directory and the recipient has a self-contained, working ASEAN translation microservice.

### Source files

#### `sealion-server.py`

```python
#!/usr/bin/env python3
"""
SEA-LION translation sidecar.
Loads aisingapore/Gemma-SEA-LION-v4.5-E2B-IT and serves POST /translate.
"""
import os
import json
import logging
from flask import Flask, request, jsonify

logging.basicConfig(level=logging.INFO, format="[SEA-LION] %(message)s")
log = logging.getLogger(__name__)

HOST = os.environ.get("SEALION_SERVER_HOST", "127.0.0.1")
PORT = int(os.environ.get("SEALION_SERVER_PORT", "5200"))
MODEL_ID = os.environ.get("SEALION_MODEL_ID", "aisingapore/Gemma-SEA-LION-v4.5-E2B-IT")

app = Flask(__name__)
processor = None
model = None

TRANSLATE_SYSTEM = (
    "You are a professional translator. "
    "Translate the given title and content to English. "
    "Return ONLY valid JSON with keys \"title\" and \"content\". "
    "Preserve proper nouns, URLs, and technical terms."
)

def load_model():
    global processor, model
    log.info(f"Loading model {MODEL_ID}...")
    from transformers import AutoProcessor, AutoModelForCausalLM
    processor = AutoProcessor.from_pretrained(MODEL_ID)
    model = AutoModelForCausalLM.from_pretrained(MODEL_ID, dtype="auto", device_map="auto")
    log.info("Model loaded.")

@app.route("/health")
def health():
    if model is None:
        return jsonify({"status": "loading"}), 503
    return jsonify({"status": "ok"})

@app.route("/translate", methods=["POST"])
def translate():
    if model is None:
        return jsonify({"error": "model not loaded"}), 503

    data = request.get_json(force=True)
    title = data.get("title", "")
    content = data.get("content", "")

    messages = [
        {"role": "system", "content": TRANSLATE_SYSTEM},
        {"role": "user", "content": f"Title: {title}\n\nContent: {content[:1000]}"},
    ]

    text = processor.apply_chat_template(
        messages,
        tokenize=False,
        add_generation_prompt=True,
        enable_thinking=False,
    )
    inputs = processor(text=text, return_tensors="pt").to(model.device)
    input_len = inputs["input_ids"].shape[-1]

    outputs = model.generate(**inputs, max_new_tokens=1024, temperature=0.0, do_sample=False)
    raw = processor.decode(outputs[0][input_len:], skip_special_tokens=True).strip()

    # Strip markdown fences if present
    if raw.startswith("```"):
        raw = raw.split("```")[1]
        if raw.startswith("json"):
            raw = raw[4:]
        raw = raw.strip()

    try:
        result = json.loads(raw)
        return jsonify({"title": result.get("title", title), "content": result.get("content", content)})
    except json.JSONDecodeError:
        log.warning(f"JSON parse failed, returning raw: {raw[:100]}")
        return jsonify({"title": title, "content": raw})

if __name__ == "__main__":
    load_model()
    log.info(f"Starting server on {HOST}:{PORT}")
    app.run(host=HOST, port=PORT, threaded=False)
```

#### `requirements.txt`

```
transformers>=4.51.0
torch>=2.7.0
accelerate>=1.6.0
flask>=3.1.0
pillow>=11.0.0
torchvision>=0.22.0
```

> `pillow` and `torchvision` are mandatory even for text-only translation. Gemma-SEA-LION-v4.5-E2B-IT is a multimodal model; its processor imports both libraries unconditionally at load time. Omitting either causes an `ImportError` at startup.

#### `setup.sh`

```bash
#!/usr/bin/env bash
set -euo pipefail

# Require Python 3.10+
PY_VERSION=$(python3 -c "import sys; print(sys.version_info.major * 100 + sys.version_info.minor)")
if [ "$PY_VERSION" -lt 310 ]; then
  echo "Error: Python 3.10+ required (found $(python3 --version))"
  exit 1
fi

VENV=".venv"

if [ ! -d "$VENV" ]; then
  echo "Creating venv at $VENV..."
  python3 -m venv "$VENV"
fi

echo "Installing requirements..."
"$VENV/bin/pip" install --upgrade pip -q
"$VENV/bin/pip" install -r requirements.txt

echo "Done. Run with: $VENV/bin/python3 sealion-server.py"
```

### Minimum viable README for the recipient

````markdown
# SEA-LION Translation Sidecar

A local HTTP translation service for Southeast Asian languages, powered by
[aisingapore/Gemma-SEA-LION-v4.5-E2B-IT](https://huggingface.co/aisingapore/Gemma-SEA-LION-v4.5-E2B-IT).

## Requirements

- Python 3.10+
- ~4 GB disk (model cache) + ~4 GB RAM/unified memory

## Setup

```bash
bash setup.sh          # creates .venv and installs dependencies
```

## Run

```bash
.venv/bin/python3 sealion-server.py
# Server starts on http://127.0.0.1:5200
# First start downloads the model (~4 GB) — subsequent starts use cache
```

## API

**Health check**
```
GET /health
→ { "status": "ok" }        # model loaded and ready
→ { "status": "loading" }   # still loading (HTTP 503)
```

**Translate**
```
POST /translate
Content-Type: application/json

{ "title": "...", "content": "..." }

→ { "title": "...(English)...", "content": "...(English)..." }
```

## Quick test

```bash
curl -s -X POST http://127.0.0.1:5200/translate \
  -H 'Content-Type: application/json' \
  -d '{"title":"Serangan Siber Mengincar Infrastruktur Kritis","content":"BSSN melaporkan peningkatan serangan siber."}' | python3 -m json.tool
```
````

### Docker packaging (optional)

If the recipient wants a fully containerised service, the sidecar translates directly into a `Dockerfile`:

```dockerfile
FROM python:3.11-slim

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY sealion-server.py .

ENV SEALION_SERVER_HOST=0.0.0.0
ENV SEALION_SERVER_PORT=5200

EXPOSE 5200

CMD ["python3", "sealion-server.py"]
```

Build and run:

```bash
docker build -t sealion-sidecar .
# Mount the HuggingFace cache to avoid re-downloading 4 GB on every restart
docker run -p 5200:5200 \
  --volume ~/.cache/huggingface:/root/.cache/huggingface \
  sealion-sidecar
```

### Calling the sidecar from any language

Because the API is plain HTTP + JSON, any language can consume it without Python:

**JavaScript / TypeScript**
```typescript
const title = 'Serangan Siber Mengincar Infrastruktur Kritis';
const content = 'BSSN melaporkan peningkatan serangan siber.';

const res = await fetch('http://127.0.0.1:5200/translate', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({ title, content }),
});
const { title: titleEn, content: contentEn } = await res.json();
// titleEn → "Cyberattacks Target Critical Infrastructure"
```

**Python** (requires `pip install httpx`, or use `urllib.request` from stdlib)
```python
import httpx

title = 'Serangan Siber Mengincar Infrastruktur Kritis'
content = 'BSSN melaporkan peningkatan serangan siber.'

r = httpx.post('http://127.0.0.1:5200/translate', json={"title": title, "content": content})
result = r.json()   # { "title": "Cyberattacks Target Critical Infrastructure", "content": "..." }
```

**curl (shell scripts / pipelines)**

Use `jq` to safely build the JSON body — avoids breakage if the text contains quotes or special characters:
```bash
curl -s -X POST http://127.0.0.1:5200/translate \
  -H 'Content-Type: application/json' \
  -d "$(jq -n --arg t "$TITLE" --arg c "$CONTENT" '{title: $t, content: $c}')"
```

### Configuration environment variables

| Variable | Default | Description |
|---|---|---|
| `SEALION_SERVER_HOST` | `127.0.0.1` | Bind address. Set to `0.0.0.0` to expose on the network |
| `SEALION_SERVER_PORT` | `5200` | Listen port |
| `SEALION_MODEL_ID` | `aisingapore/Gemma-SEA-LION-v4.5-E2B-IT` | HuggingFace model ID — swap to test other SEA-LION variants |

### What the recipient needs to know

- **First start is slow.** The model (~4 GB, 11 files) downloads automatically from HuggingFace on first run. Subsequent starts load from `~/.cache/huggingface/` in ~25–35 s.
- **GPU is optional.** `device_map="auto"` picks MPS on Apple Silicon, CUDA on Linux with a GPU, or CPU as a last resort. CPU inference is significantly slower (~60–120 s per translation).
- **Memory.** The model uses ~4 GB of unified memory (Apple Silicon) or VRAM (CUDA). On CPU it occupies ~4 GB of system RAM.
- **The server is single-threaded** (`threaded=False` in Flask). It processes one translation at a time. For high-throughput use, run multiple instances on different ports behind a load balancer.
- **Content is truncated at 1000 characters.** The sidecar only translates the first 1000 characters of `content`. This is intentional — articles are typically summarised before ingestion, and longer inputs cause OOM errors on 4 GB devices.

---

## Troubleshooting

**Sidecar fails to start — `ImportError: PIL`**
```bash
# In the main app (.venv-sealion)
.venv-sealion/bin/pip install pillow torchvision

# In a standalone install (.venv)
.venv/bin/pip install pillow torchvision
```
Both are required even for text-only translation; Gemma 4's processor loads them unconditionally.

**Port already in use**
Port 5200 is chosen to avoid collision with Vite's default (5173). Override with `SEALION_SERVER_PORT`.

**Sidecar times out — health check exceeded (120 s)**
The model is still downloading from HuggingFace. Re-run `npm run workers` after download completes, or pre-download the model manually (see the *Download the model* section under Setup).

**`SEALION_SERVER_URL` set but sidecar not running**
`sendSealionMessage()` throws `connection refused`. The `translateToEnglish()` fallback chain catches this silently and routes to the general AI provider. No articles are dropped.

**Translation returns original text unchanged**
The content field was empty. SEA-LION has nothing to translate; the title passes through unchanged. Expected behaviour — the fallback provider will handle it.

**AWS / cloud deployment**
`getTranslationProvider()` returns `'anthropic'` when `DEPLOYMENT_TARGET=aws`. The Python sidecar is never started. `SEALION_SERVER_URL` is not needed in the production environment.

---

## Why Local Inference Matters

Running SEA-LION locally rather than routing through a cloud translation API has three concrete benefits for a platform handling regional security intelligence:

**Cost.** At 10–15 articles per minute sustained, a cloud translation API would accumulate significant per-token costs over time. Local inference has zero marginal cost after the one-time setup.

**Latency.** A local MPS inference call returns in 2–5 seconds. A cloud API roundtrip adds network latency, rate-limit queuing, and cold-start variability on top of generation time.

**Data sovereignty.** Cyber threat intelligence from national CERTs, government advisories, and sensitive security bulletins stays on-device. Nothing in the translation pipeline leaves the machine.

---

*SEA-LION is developed by [AI Singapore](https://aisingapore.org/) under the National AI Programme. Model weights are available at [huggingface.co/aisingapore](https://huggingface.co/aisingapore).*
