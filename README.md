# Gnani speech evaluation: Prisma (ASR) and Timbre (TTS)

| Notebook | What it evaluates | Findings |
|---|---|---|
| `part1_prisma_asr.ipynb` | Prisma v3.0 and v2.5 speech-to-text on 336 clips (57 minutes) across 11 test categories | section 7 |
| `part2_timbre_tts.ipynb` | Timbre v3.0 and v2.5 text-to-speech on 117 test items, 286 synthesis jobs and 60 streaming runs | section 8 |

Both notebooks run end to end from the shipped data and response caches, with no download and no API credits.

## Setup

```bash
python -m venv .venv && source .venv/bin/activate      # Python 3.10+
pip install torch --index-url https://download.pytorch.org/whl/cpu   # CPU build is enough (Part 2 only)
pip install -r requirements.txt
cp .env.example .env                                    # then paste your keys into .env
jupyter lab                                             # open a notebook and Run All
```

System dependency: `ffmpeg` on the PATH (Part 1 phone-channel simulation, Part 2 μ-law decoding).

`.env` keys:

| Key | Needed for | When |
|---|---|---|
| `GNANI_API_KEY` | Prisma and Timbre calls | only for requests not already in the caches |
| `SARVAM_API_KEY` | Part 1: regenerating the 46 synthetic probe clips. Part 2: the independent judge ASR | only for clips or transcripts not already cached |

## Part 1 · Prisma (ASR)

Categories: clean read speech, simulated and real 8 kHz telephony, babble and stationary noise, Hindi-English code-switching,
Indian-accented English, banking entities (amounts, dates, IDs), short replies, long turns and no-speech probes.
WER and CER are implemented from scratch (edit distance with a backtrace for substitutions, deletions and insertions) and checked
against `jiwer`. Two extra runs isolate one variable each: v3.0 with domain `keyterms` on the entity set, and v3.0 with
`format=verbatim` on the FLEURS-based sets (like-for-like script with v2.5).

* The shipped test set lives in `data/asr/` (manifest plus 336 clips). Set `REBUILD = True` in section 2 to download the public
  sources (about 1.8 GB) and rebuild every clip with the same seed.
* Responses are cached in `results/part1/responses.jsonl`. Per-clip scores, summaries and charts are written to `results/part1/`.

## Part 2 · Timbre (TTS)

"Good" is defined on six dimensions, each measured objectively and repeatably:

| Dimension | Metric |
|---|---|
| Correct reading | 98 per-item checks of the expected spoken form (amounts, dates, IDs, abbreviations, names), taken from Gnani's Text Normalization Guide |
| Intelligibility | Round-trip CER: synthesize, transcribe with an independent judge (Sarvam `saaras:v4`, verbatim mode), compare. A second judge (Prisma) tests for vendor bias, and both judges are scored on studio human speech for their own floor |
| Reliability | First-try success, HTTP 5xx and empty-audio responses, truncation, run-to-run consistency, clipping and loudness |
| Speed | Time to first audio (client and server side) over SSE and WebSocket, REST time, real-time factor, p50 and p95 |
| Naturalness | UTMOS22 predicted MOS, anchored on IndicTTS and LJ Speech studio recordings |
| Prosody and expression | Final pitch rise on question vs statement pairs, and measured effect of v3.0 audio tags |

* Synthesized audio is in `data/tts/audio/<model>/<format>/`, studio anchors in `data/tts/anchors/`.
* Every synthesis attempt (including failed ones) is in `results/part2/attempts.jsonl`, streaming timings in `latency.jsonl`,
  judge transcripts in `judge.jsonl` and UTMOS scores in `utmos.csv`. Delete a file to re-query that step.
* UTMOS22 weights (392 MB) download once from the SpeechMOS GitHub release on first use.

## Run times

The account used allows one active request at a time, so calls are sequential with backoff on HTTP 429.
From scratch, Part 1 takes about 25 minutes and Part 2 about 60 minutes, mostly API calls. From the shipped caches, each runs in under a minute.

## Data sources and licences

| Source | Used for | Licence |
|---|---|---|
| [FLEURS](https://huggingface.co/datasets/google/fleurs) (Google) | Part 1: clean read Hindi and Tamil, base clips for noise, telephony and long-form | CC BY 4.0 |
| [GramVaani](https://www.openslr.org/118/) (OpenSLR 118) | Part 1: real 8 kHz rural Hindi telephony, short replies | CC BY 4.0 |
| [MUCS 2021](https://www.openslr.org/104/) (OpenSLR 104) | Part 1: Hindi-English code-switching | CC BY 4.0 |
| [EdAcc](https://huggingface.co/datasets/edinburghcstr/edacc) (University of Edinburgh) | Part 1: Indian English and a US English control | CC BY-SA 4.0 |
| Sarvam Bulbul v3 TTS | Part 1: synthetic banking-entity sentences and short replies | generated for this test |
| [IndicTTS](https://www.iitm.ac.in/donlab/indictts/) Hindi and Tamil (IIT Madras), via Hugging Face | Part 2: studio human anchors | IndicTTS licence (research use) |
| [LJ Speech](https://keithito.com/LJ-Speech-Dataset/) | Part 2: studio human anchor (English) | Public domain |
