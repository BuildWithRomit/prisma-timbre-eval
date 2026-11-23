# Prisma (ASR) & Timbre (TTS) Evaluation

Hands-on evaluation of Gnani's **Prisma** speech-to-text and **Timbre** text-to-speech APIs, approached the way a Solutions Engineer would before recommending them to a customer: define what "good" means, build test cases for it, and measure.

- **Notebook:** `eval.ipynb` (both parts, runs top to bottom)
- **Report:** `report.pdf` (1–2 page summary of design, results and recommendations)

---

## Key findings

**Prisma (ASR), 100 clips**
- Strong on Indian languages: clean Hindi 7.4% WER / 2.0% CER, Bengali 1.5% CER; no loss on 8 kHz telephony audio.
- English degrades sharply in heavy background noise (4.8% → 24.8% WER at 5 dB SNR); Hindi stays at 5–7%.
- Silent failures: very quiet or noisy clips returned HTTP 200 with an empty transcript.
- Hinglish is recognised accurately (7.1% WER against an all-Devanagari reference), but English words are output in Devanagari, so WER against a mixed-script reference is 53.8%.

**Timbre (TTS), 25 sentences**
- English and Hindi intelligibility at or above a real-human baseline; strong Indian localisation (lakh grouping, DD/MM dates, abbreviations, Hinglish).
- One serious failure: "8.5% per annum" was read as "eight point five" (words dropped).
- All requests faster than real time (RTF 0.21–0.57).

Details, charts and recommendations are in the notebook and report.

---

## Repository structure

```
prisma-timbre-eval/
├── eval.ipynb               # the full evaluation (Parts 1 and 2
├── report.pdf               # written summary
├── requirements.txt
├── .env            
├── data/
│   ├── asr/                 # FLEURS clips + generated noisy / 8 kHz versions
│   ├── my_recordings/       # own recordings (numbers, Hinglish, short replies, domain terms)
│   ├── tts/                 # Timbre output audio (regenerated on run)
│   └── asr_manifest.csv     # every ASR clip: path, reference text, category, condition
└── results/
    ├── asr_results.csv      # Prisma output per clip
    ├── tts_results.csv      # Timbre latency / RTF per sentence
    ├── whisper_results.csv  # Whisper transcripts used for TTS intelligibility
    ├── noise_curve.png
    └── tts_latency_vs_length.png
```

---

## Setup

**Requirements:** Python 3.11 (3.10–3.12 should work), a Gnani API key from [app.gnani.ai](https://app.gnani.ai), and about 3 GB free disk space (mostly the Whisper model).

```bash
git clone https://github.com/BuildWithRomit/prisma-timbre-eval.git
cd prisma-timbre-eval

python -m venv .venv
# Windows:      .venv\Scripts\Activate.ps1
# macOS/Linux:  source .venv/bin/activate

pip install -r requirements.txt
```

Create a `.env` file in the project folder (see `.env`):

```
GNANI_API_KEY=your-key-here
```

**ffmpeg** is only needed to convert new phone recordings (M4A/MP3). The included recordings are already converted to WAV.
- Windows: `winget install ffmpeg`
- macOS: `brew install ffmpeg`

---

## How to run

1. Open `eval.ipynb` in VS Code or Jupyter and select the `.venv` kernel.
2. **Restart the kernel, then Run All.**

**What to expect on the first run:**
- FLEURS clips are downloaded from HuggingFace (a few MB).
- The Whisper `medium` model is downloaded (~1.5 GB) and runs on CPU (several minutes). A GPU is optional.
- API calls use Gnani credits: about 100 Prisma calls and 25 Timbre calls.

**Reruns are fast and free:** every API and Whisper result is cached in `results/`, and cached items are skipped. To re-evaluate from scratch, delete the CSV files in `results/`.

---

## Reproducibility notes

- Noise mixing uses a fixed random seed (42), so noisy clips are identical on every run.
- WER/CER are implemented from scratch (`normalize`, `align_counts`, `score`) and verified with unit tests in the notebook.
- **Prisma model:** the API default `prisma-v2.5` was used. **Timbre model:** `timbre-v3.0`.
- Timbre's REST endpoint returns raw 16-bit mono PCM at 44.1 kHz (no WAV header); the notebook wraps it into WAV.
- Timbre calls go through a retry helper (exponential backoff on 429/5xx and timeouts). Prisma calls are not retried: each clip's failure is logged and skipped, and reruns resume from the cached CSV.

---

## Data and licences

- **FLEURS** (Conneau et al., 2022), test split, CC-BY 4.0: clean read speech in English, Hindi and Bengali, used as the ASR baseline and the Whisper calibration set.
- **Own recordings:** single speaker (the author), phone-recorded, converted to 16 kHz mono WAV.
- **Whisper** (OpenAI, via `faster-whisper`): independent ASR used to measure TTS intelligibility.

---

## Limitations

Small samples (≈100 words per noise condition, 3–5 own recordings per category, 25 TTS sentences), a single speaker for own recordings, one listener for manual TTS checks, and Whisper being unreliable for Bengali. Results show clear patterns but are not precise benchmarks.