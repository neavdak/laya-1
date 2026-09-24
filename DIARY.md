# 📓 DIARY.md — The Laya Setup Journey

---

## 📅 Wednesday, 24 September 2026 — Morning, 10:55 AM IST

**Dear Diary,**

Today started with a simple request — *"pull this repo here"* — and I found myself in an empty folder at `D:\projects\jev`. Not a single file. A blank slate. The repo in question was `https://github.com/NandhaKishorM/laya`, a project by NandhaKishorM — a fast, local, on-device AI decision engine called **Laya**. Multilingual, non-autoregressive, trained with reinforcement learning. Fancy stuff.

I ran the clone:

```powershell
git clone https://github.com/NandhaKishorM/laya.git .
```

A few seconds later — success. The directory came alive with files. I checked the latest commit to see where things stood:

```
84ca724 release: 0.3.18, long-document callout for laya-multilingual
```

Version 0.3.18. Fresh off the press. I listed everything out — `laya/` core module, `examples/`, `tests/`, `docs/`, Docker configs, Nix flake, notebooks, benchmarks. This is a real project. Good bones.

---

## 📅 Wednesday, 24 September 2026 — 10:59 AM IST

**Dear Diary,**

The user asked: *"how do I run it here now?"*

First things first — I needed to understand the environment. I checked the Python version:

```
Python 3.12.7
```

Running on Anaconda. I peeked at `pyproject.toml` and found that Laya needs:
- `torch >= 2.0.0`
- `transformers >= 4.48.0`
- `safetensors >= 0.4.0`
- `huggingface_hub >= 0.20.0`
- `numpy >= 1.20.0`

I scanned the installed packages and discovered the environment was already beautifully stocked:
- `torch 2.6.0+cu118` (CUDA 11.8 — GPU-ready!)
- `transformers 5.14.1`
- `safetensors 0.8.0`
- `huggingface_hub 1.25.1`
- `uvicorn 0.29.0` and `fastapi` — already present

Everything needed was already installed. I did a quick sanity check:

```python
import laya
print(laya.__version__)  # → 0.3.18
```

It imported cleanly. Then I ran `pip install -e .` to register the project in editable mode so the CLI commands would work:

```
Successfully built laya
Successfully installed laya-0.3.18
```

CLI commands now registered globally:
- `laya` — routing & prediction from the terminal
- `laya-serve` — HTTP server (Jev `/v1/systemone` protocol)
- `laya-mcp-server` — Model Context Protocol stdio server

---

## 📅 Wednesday, 24 September 2026 — 11:07 AM IST

**Dear Diary,**

Before touching the server, I wanted to make sure the foundations were solid. I ran the routing test suite:

```powershell
python tests/test_router.py
```

Result: **399 passed, 0 failed.** Clean. Every routing logic test passed without a hitch. That felt good — like cracking your knuckles before sitting down to real work.

I also tested the CLI routing live:

```powershell
python -m laya.cli "My payment failed twice"
```

Output:
```
Model   : english
Reason  : Latin script, language not identified and no non-English letters; using default (english)
```

No weights downloaded yet. This was pure rule-based script detection — instant, offline, zero-overhead. The router looked at the text character profile and picked the right checkpoint strategy in milliseconds. Impressive.

---

## 📅 Wednesday, 24 September 2026 — 11:11 AM IST — 11:24 AM IST

**Dear Diary,**

The user said: *"show me it working in local server."*

Time to fire up the example server. I read through `examples/server.py` — a well-crafted single-file FastAPI application with:
- `GET /health` — server status
- `GET /models` — available checkpoints
- `POST /predict` — send text + questions, get answers
- `POST /predict/batch` — batch version
- A browser-facing HTML dashboard (browsers get a UI, curl gets JSON)
- Optional bearer auth support

I started it with lazy loading (no pre-download at startup):

```powershell
python examples/server.py --host 127.0.0.1 --port 8000 --no-preload
```

Within seconds the server was up:
```
INFO: Uvicorn running on http://127.0.0.1:8000 (Press CTRL+C to quit)
INFO: Application startup complete.
```

I immediately tested `/health`:
```json
{
  "status": "ok",
  "config": {
    "preload": false,
    "device": null,
    "default": "english",
    "max_loaded": 1
  }
}
```

Then `/models`:
```json
{
  "default": "english",
  "allowed": ["english", "multilingual", "typed-decisions"],
  "models": {
    "english": ["convaiinnovations/laya", null],
    "multilingual": ["convaiinnovations/laya", "multilingual"],
    "typed-decisions": ["convaiinnovations/laya", "typed-decisions"]
  }
}
```

Beautiful. The server knows its three checkpoint personas: the English model, the 100+ language multilingual model, and the typed-decisions fine-tuned model.

Then came the real test — I fired a live prediction request at `POST /predict`:

```json
{
  "state": "We were billed twice for March. Please refund it today or we will cancel.",
  "questions": {
    "department": {
      "type": "choice",
      "instructions": "Which department should handle this request?",
      "criteria": {
        "billing": "invoices, payments, refunds",
        "technical": "bugs, outages, system errors"
      }
    }
  }
}
```

This was the **first inference call** — which means Laya had to download the checkpoint from Hugging Face on demand. The model files started flowing in from `convaiinnovations/laya`:
- `config.json`
- `tokenizer.json` (3.4 MB)
- `tokenizer_config.json`
- `rl_agent_config.json`
- The main model weights (`safetensors` blob — still downloading as I write this)

The Python server process was consuming ~521 MB of RAM and active on CPU while pulling weights. The inference is queued — it will return the moment the weights finish loading. This is normal; it only happens once. All subsequent calls will be instant from cache.

---

## 📝 Key Reference — How to Run Everything Again

| Task | Command |
|---|---|
| Start the web server (lazy mode) | `python examples/server.py --host 127.0.0.1 --port 8000 --no-preload` |
| Start the production Jev server | `laya-serve --host 127.0.0.1 --port 8000` |
| Route a message (offline, instant) | `laya "your text here"` |
| Full prediction with preset | `laya "your text here" --preset triage` |
| Run core tests | `python tests/test_router.py` |
| Open docs | http://127.0.0.1:8000/docs |
| Open dashboard | http://127.0.0.1:8000/ |

> **Note:** First-time inference downloads `convaiinnovations/laya` checkpoint from Hugging Face (~150 MB).
> Subsequent calls load from `C:\Users\neavd\.cache\huggingface\hub\` instantly.

---

*— End of diary entries so far. To be continued as work progresses.*
