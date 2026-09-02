---
name: voice-clone-maskscore-ladder
description: Build a MaskScore-style 10-rank voice-clone gradient corpus from a set of speech recordings, with a multi-judge ASR panel and wavlm+wer scoring. Use when a new speech dataset needs to enter the RFD 1173 MaskScore Speech stub (RFD 2164), when a new SpeakingFaces subject or similar CC-BY audio source needs to be processed, or when a downstream reward-model distillation (RFD 2167) needs training data.
tools: Read, Write, Edit, Grep, Glob, Bash
---

# Voice-clone MaskScore ladder

Build a 10-rank voice-clone gradient with a multi-judge ASR panel and per-candidate scores. Ships as three ETNF ZSTD parquets per SpeakingFaces (or similar) trial. Attribution baked in via `CITATION.cff`.

## When to invoke

The user names a speech-clip source (a SpeakingFaces subject, another CC-BY audio set) and asks for a MaskScore corpus rung, a voice-clone reward gradient, or a Speech-stub extension. Register a new RFD serial before code lands.

## Pipeline

The reference implementation is in `6-datasource/anny-render-corpus`, run under the `anny-mac`, `asr`, and `tts` pixi environments.

**1. Register the RFD serial.** Next unused serial in `2-contract/manuals-weftspun/SERIALS-vsekai-fabric.usda` under arc `1.3.6.1.4.1.66606.1.2`. Add `def "S<N>"` with a kebab-case `slug`, create `rfd/<N>-<slug>/README.md` (≤40 lines, canary sentence, `spine` to urn:oid:1.3.6.1.4.1.66606.1.1.1173).

**2. Transcribe with the 12-track ASR panel** (`emit_10track_panel.py`): parakeet (CC-BY-4.0), whisper large-v3, voxtral, wav2vec2, gemma-auto (text tracks) + gemma-gbnf, voxtral-ipa, ipa-whisper-small, ipa-whisper-base, allosaurus (ipa tracks). Then run `add_allosaurus_control.py` for allosaurus-eng/rus phone-inventory controls.

**3. Voice-clone 10 ranks per clip** (`voice_clone_10rank.py`) via `Qwen/Qwen3-TTS-12Hz-1.7B-Base` in the `tts` env (python 3.12, torch/torchaudio pinned to 2.5.1):
  - rank 1 identity clone
  - ranks 2-6 pitch/tempo perturbation gradient
  - ranks 7-8 same voice, wrong content (next-clip text; fixed random phrase)
  - ranks 9-10 wrong subject (different SpeakingFaces speaker via `--wrong-subject-dir`, `x_vector_only_mode=True`)

**4. Score** (`score_voice_clones.py`): wavlm_cos speaker embedding (Microsoft WavLM Base+ SV) + voxtral-WER on 15 clips × 10 ranks. Verify the gradient is monotonic across identity → perturbation → wrong content → wrong subject.

**5. Emit ETNF parquets** (`emit_speech_stub.py`): three ZSTD files — root (per clip), candidates (20 per clip: 10 audio + 10 transcript), scores (long-form metric_name/value). All follow the unified schema from RFD 1173.2165.

**6. Upload to HF** (`hf upload ...`) into `chibifire/maskscore-rung-1-bootstrap/<stub>/` with `CITATION.cff` at repo root citing every source under CC-BY-4.0 or MIT (SpeakingFaces, Parakeet, allosaurus, and Apache/MIT acknowledgements for the rest). Update `README.md` `configs` block with `<stub>/*.parquet` entries so the HF dataset viewer picks them up.

**7. Commit + PR** against `weftspun/anny-render-corpus` main, cite the RFD OID in the title, include the attestation canary.

## Rules that catch failure modes

- **CC-BY-4.0 sources need CITATION.cff**; drop the attribution and the corpus is undistributable.
- **Voxtral is faster + cleaner than Whisper for accented English** on this stack; don't reflex-swap the panel canonical judge.
- **Kyutai STT** has a non-obvious moshi API; defer to Rung 2.
- **Qwen3-TTS Base** (not CustomVoice) is the zero-shot cloning variant. CustomVoice uses 9 premium timbres, not arbitrary reference audio.
- **`x_vector_only_mode=True`** for wrong-subject cloning — avoids needing transcripts of the different-speaker source.
- **Voxtral batch on MPS is ~7-10s/clip**, not ~1s; budget accordingly and print per-clip progress in scripts.
- **The Voice-stub USDZ needs the full 78-joint skeleton + skin weights** if any candidate uses bone rotations; a 1-joint dummy skeleton silently ignores pose animation. See RFD 2162 for the fix.
