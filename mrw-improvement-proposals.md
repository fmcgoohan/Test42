# mrw: analysis and improvement proposals

| | |
|---|---|
| Repository | `fmcgoohan/crispy-rotary-phone`, `main` at `4462ea5` (2026-07-13, the merge of PR #12 `m5-video`) |
| Date | 2026-11-04 |
| Scope | All of `mrw/`, `tests/`, `schemas/`, the docs and CI, as merged through M5 |
| Method | Static review. Nine reviewers each covered one area, an adversarial verifier re-checked every finding against the code, and a completeness critic looked for gaps. 147 findings were raised; 145 survived verification (2 were rejected as not worth acting on). They are merged here into 64 distinct issues. I re-checked the critical and high-severity ones against the code myself. |
| Not done | No code was executed. Installing the project's dependencies was blocked in the review environment, so behavioural claims rest on reading the code and the pinned library sources, plus small stand-alone re-implementations. None of them came from running mrw. |

Finding IDs such as `VID-1` refer to [mrw-findings-detail.md](mrw-findings-detail.md). That file gives each finding's full evidence, failure scenario, proposal and the verifier's notes. Severity follows the reviewers' rubric: **critical** means silent wrong output or corruption in normal use; **high** means wrong output or a crash on plausible real-world input, or a missing safety net for a hard constraint; **medium** covers edge cases, robustness gaps and test gaps; **low** is polish.

---

## 1. Overall assessment

The project is unusually disciplined for its size. Several things are worth protecting while you fix the rest:

- **Determinism machinery.** There is one canonical write path, with an atomic temp-file-and-rename. Every float written to a document goes through the rounding helper for its unit, and no volatile data appears outside `manifest.json`. Globs are sorted, k-means is seeded and Whisper threads are pinned. Every stage has a double-run byte-comparison test, including the stem FLACs and the JPEG frames.
- **Contract discipline.** The hand-written JSON Schemas (draft 2020-12) state units and conventions for degenerate cases. All five pydantic document models match their schemas field for field and in order, all the worked examples validate, and the smoke tests validate the documents they produce against the schemas without network access.
- **Uncertainty is recorded in the data:** flags, coverage ratios, `untranscribed_regions`, `not_applicable`, a ledger of partial API spend, and a budget check before any API spend.
- **Hygiene.** Heavy imports are deferred until cheap prerequisite checks have run (checklist item T5). The pure logic (`align.py`, the palette and motion helpers, the cost estimator) is split out and has fast unit tests. Test fixtures are generated, never committed media. Code comments cite the review findings that motivated them.

The problems fall into five themes:

1. **Silent wrong-output bugs on ordinary input.** The test fixtures are best cases (solid colours, single-timestamp LRC files, clean speech), so these get through: wrong palettes for almost every real shot, scrambled stills from portrait phone video, lost chorus repeats in compressed `.lrc` files, BOM and non-UTF-8 lyrics files corrupted, and repeated choruses placed at the wrong time.
2. **Up-to-date checks look only at config.** Re-running an upstream stage, changing the code, or replacing the video file at its path never marks dependent documents stale. `mrw status` doesn't even show config changes for the lyrics and video stages.
3. **No CI check that blocks a merge.** The tests only run inside the LLM review agent, and a failing test never turns a check red.
4. **The offline guarantee doesn't hold.** faster-whisper looks the model up on the Hugging Face Hub on every `mrw lyrics` run, and the model revision is not pinned.
5. **Lyrics is the weakest stage.** Parsing misses common LRC and encoding variants, and the aligner matches each line on its own without enforcing order across the song (greedy).

---

## 2. Proposed plan

### P0: small fixes for wrong output today

These are mostly S-sized (under an hour each) and low-risk. Do item 0 first: items 1–4 change document bytes, and today nothing would mark the existing documents stale.

| # | Change | Where | Effort | IDs |
|---|---|---|---|---|
| 0 | Give each stage an integer `ALGORITHM_VERSION`. Record it in the stage's manifest entry and compare it in the up-to-date check and in `mrw status`. Treat a missing value as version 1 so nothing goes stale on upgrade. Add a CHECKLIST rule: bump it whenever a change alters output under an unchanged config. | each `run_*`, `models.py`, `cli.py` | S–M | ORCH-3, DET-8, ARCH-6, LYR-15 |
| 1 | **Palette:** downscale each frame as an H×W image *before* pooling pixels, and make the palette helper take (N,3) samples. Today a shot's pixels are flattened into an N×1 column and then "downscaled" to 1×256, so k-means sees 256 averages of about 4 full image rows each. | `video.py:472-474`, `video.py:109-115` | S | VID-1, SEC-1 |
| 2 | **Rotation:** read the display-matrix rotation and swap width and height, or decode via `-f image2pipe -c:v ppm` and take each frame's size from its header. Assert that the decoded byte count is exact. | `video.py:211-240`, `video.py:249-258` | S | VID-4, ORCH-8, SEC-3 |
| 3 | **LRC parsing:** emit one line per leading time tag, skip ID tags (`[ar:]`, `[ti:]`, `[by:]`…), honour `[offset:]` and accept `mm:ss:xx`. Use one parser for both ingest and lyrics. | `lyrics.py:56`, `lyrics.py:245-268`, `ingest.py` | S | LYR-5, LYR-7, SCH-2, ARCH-14 |
| 4 | **Lyrics encoding:** decode `utf-8-sig` strictly, then UTF-16 by BOM, then cp1252 (or exit 2), and record the encoding used. Never use `errors="replace"`. | `ingest.py:148`, `lyrics.py:514` | S | SEC-7, LYR-6, ORCH-14 |
| 5 | **Whisper offline and pinned:** resolve the model with `local_files_only=True` and a pinned per-model revision, then pass the local path to `WhisperModel`. On a cache miss, exit 2 with "run `mrw models fetch`". `models fetch` must use the same pin. | `lyrics.py:160-162`, `lyrics.py:227-231` | S–M | LYR-3, SEC-2, DET-9, TEST-4 |
| 6 | **`mrw status`:** route lyrics and video through `_display`, as source and features already are, so config changes show as `stale`. | `cli.py:304-305` | S | ORCH-11 |
| 7 | **Strict config:** set `extra="forbid"` and `allow_inf_nan=False` on every config section and make `device` a `Literal`. Turn TOML and validation errors into a one-line message with exit 2. Replace the committed operator `mrw.toml` with `mrw.example.toml`. | `config.py`, `cli.py:28` | S | ARCH-2, ORCH-5, ORCH-10, SEC-10, DET-5 |
| 8 | **Non-finite floats:** pass `allow_nan=False` in `canonical.dumps` as a backstop (after item 7, so an `inf` budget fails at config load, not in `stage_hash`), and turn `-0.0` into `0.0` in the rounders. | `canonical.py` (`dumps`) | S | ARCH-7, DET-7, SCH-7 |
| 9 | **Subprocesses:** read stderr in a background thread or send it to a temp file wherever stdout is streamed, and add timeouts scaled to the media duration. | `ingest.py:111-118`, `video.py` | S | SEC-5, ORCH-9 |
| 10 | **Python range:** commit `.python-version` (3.12) and set `requires-python = ">=3.11,<3.13"`. torch 2.5.1 ships macOS arm64 wheels only for cp311 and cp312, as `uv.lock` shows. | `pyproject.toml:5` | S | TEST-14, ARCH-4 |
| 11 | **CI gate:** add `tests.yml`. On every PR (forks included) and every push to `main`, run `uv sync --locked`, then `pytest -m 'not slow'`; run the slow suite with cached weights. Make it a required check, and have the review agent read `gh pr checks` instead of running the tests itself. | `.github/workflows/` | M | TEST-1, ARCH-3 |

### P1: make the pipeline model trustworthy

1. **Up-to-date checks that follow dependencies** (high). Add a content digest for the stems and an `inputs_sha256` to each manifest `DocumentEntry`, built from what the stage actually read: features from the stems digest, lyrics from the audio_features hash, the stems digest and the lyrics-file sha, video from the verified file sha. Require it to match in the up-to-date check and in `status`. Interim S option: when an upstream stage succeeds, flip its dependents' `ok` entries to `stale` in the same manifest write. This is an additive manifest 1.4.0. *B1 · DET-1, ARCH-1, FEAT-2, LYR-4, ORCH-1, SCH-1, SEC-4, TEST-2*
2. **Rewrite the aligner as a global, in-order (monotone) alignment** (high). Use a line-level dynamic program over (line, word) with skip penalties. A word, once used, can't be reused. Hints become a cost term, with an estimated global offset. Pair words inside a window by content, not position, and flag interpolated timings. Keep unmatched transcript words in an additive array and count them in coverage. Clamp every time to [0, duration], and spread consecutive unanchored lines across the gap instead of stacking them. *D1–D6 · LYR-1, LYR-2, CRIT-1, LYR-8, LYR-11, SCH-3, CRIT-5*
3. **Make the feature values mean what the schema says** (high). Drop onsets in frames that are silent or only leakage from other instruments, ideally judged relative to the mix. Replace the regularity-based `tempo.confidence` with a measure of how strongly the onsets support the beats. Stop extending the beat grid through beatless intros and outros. Define the silent-mix and no-beat conventions, which today emit `bpm_global: 0.0` against `exclusiveMinimum: 0`. *F1–F3 · FEAT-1, FEAT-3, FEAT-8, DET-11, SCH-9*
4. **Put video times on the audio timeline** (high). Use presentation timestamps (scenedetect's `FrameTimecode.seconds` plus an ffprobe packet-timestamp table) instead of `frame_index / avg_frame_rate`. Shift by the video-minus-audio start offset and use the same time origin for motion. Choose the audio stream explicitly with `-map`, and record the offset and a variable-frame-rate (VFR) indicator. *A3–A5 · VID-2, ORCH-13, VID-3, VID-11*
5. **Manifest locking, cost ledger and interruption** (medium). Add `Library.update_manifest(track, mutate)` under an `fcntl.flock`, plus a non-blocking per-stage run lock that exits 2 with "already running". Make the API spend ledger append-only. On Ctrl-C or SIGTERM, record `failed` and any partial spend, then re-raise. Copy the video under a temporary name and rename it into place. *H3–H5 · ORCH-4, ARCH-5, DET-6, SEC-8, ORCH-7, SCH-4, VID-7, SEC-9*
6. **Ingest updates and video identity** (medium). Keep a plain re-ingest a no-op, but list the options it ignored. Add `mrw ingest --update` for new lyrics, title or `copy_video`. Hash-check the video file against `source.json` before analysing it. *B4–B5 · ORCH-6, ARCH-9, DET-4, VID-10, LYR-13, DET-3, ORCH-2, VID-5*
7. **Contract tests** (medium). Add a fast `tests/test_contracts.py`: model↔schema lockstep, validation of every example, a precision walker, golden canonical text, and golden `stage_hash` values. *G2 · TEST-10, SCH-6*
8. **Bring the smoke tests up to PLAN §11**, or record the accepted deviations (CHECKLIST P1/P2). Add a hard-case fixture alongside every fix: a two-colour textured frame, a rotated video, a video with a start offset (`itsoffset`), a VFR video, a dither-only stem, a drone. *I2 · TEST-8, VID-15, TEST-9, LYR-12, FEAT-10*

### P2: hardening, structure and roadmap

- **Correct the determinism contract.** Documents are byte-identical only when the stems are (CPU, `cpu_threads = 1`, same platform and lock file); MPS, multi-threaded and cross-platform runs are best-effort for documents too. Add an OQ-13 resolution note, a macOS arm64 CI job running the fast suite, and a scripted MPS double-run on the dev Mac at each milestone. *E3*
- **Harden the caption backend before the first paid batch.** Add a test against a mocked API endpoint. Use structured output where the model supports it, with code-fence stripping as a fallback. Check `stop_reason`, and validate replies strictly before caching them. Put the prompt template's hash in the cache key, and require an API key only at the first cache miss. Remove "temperature 0" from PLAN. *E2*
- **Stream the video decode.** Keep one frame in memory at a time and take the palette sample straight away. Add a duration guard on audio decodes. *A8, H10*
- **Shared stage helpers:** `require_ingested`, a run clock, and `record_ok`/`record_failed` via the locked update, plus a `STAGES` table for `status` and `export-schemas`, a single `media.py` ffprobe wrapper, and a single LRC parser. Defer a full Stage protocol until M6 needs one. *J1*
- **Harden the review workflow.** Pin actions by commit SHA and keep them updated with Dependabot. Use `uv sync --locked`. Drop `Bash(uv run:*)` from the agent's allowlist once tests run in their own job. Derive cache keys from their inputs and add restore-keys. *I5*
- **Lift the torch < 2.6 pin.** On torch ≥ 2.6, try wrapping `demucs.pretrained.get_model` in `torch.serialization.safe_globals([...])`. Validate it with the M2 CPU double-run, and treat the upgrade as invalidating all stems. *I6*
- **Colour pipeline.** Cluster in true CIELab, not OpenCV's 8-bit Lab. Tone-map HDR, crop letterbox bars, apply the pixel aspect ratio, and record the ffmpeg version. *A6–A7*
- **Docs.** Remove PLAN's "No implementation exists yet" header, correct the module layout and add the missing README commands. Add `docs/reviews/008-m5-video.md` and the OQ resolution notes M5 never got. *J2*
- **M6 readiness.** Before writing structure code, record OQ-17 to OQ-20: conventions for zero cuts and empty beat grids, whether lyrics are required, how much to trust lyric markup, and how staleness propagates. *J3*
- **M7 operability.** Reject or loudly warn on `stems.retain = false` until a combined run exists. Plan `mrw analyze` with `--offline` and `--json`. *J4*

---

## 3. Issue catalogue

Each entry gives severity, the finding IDs, what's wrong, and the proposed fix.

### A. Video stage

**A1 · critical · Palettes are built from averages of whole image rows, not from pixels.** `VID-1`, `SEC-1`
`run_video` flattens a shot's full-resolution frames into an N×1 column (`video.py:472-474`). `palette_from_rgb` then computes `scale = 256 / max(h, w)` with h = N and w = 1, and resizes to 1×256 (`video.py:109-115`). Each k-means sample is therefore the mean of N/256 consecutive pixels: about 8,100 pixels, or 4.2 full rows, for a 1080p frame. A frame that is 70 % red and 30 % blue comes out as five purples. The tests use solid colours, whose row averages are unchanged, so they pass.
→ Downscale per frame as H×W, pool the small frames' pixels, and have the helper take (N,3). Add a stage-path test on a two-colour 1920×1080 frame. Ship it with A6's Lab-scaling fix so palette bytes change only once.

**A2 · high · Portrait and rotated videos produce scrambled frames.** `VID-4`, `ORCH-8`, `SEC-3`
Width and height come from ffprobe's stored dimensions (`video.py:212`), but ffmpeg auto-rotates when decoding. A 1920×1080 iPhone clip tagged as rotated 90° decodes to 1080×1920, and is then reshaped as 1080 rows × 1920 columns (`video.py:233-240`). The byte count matches, so nothing fails, but every still is scrambled and every paid caption describes noise. Motion runs on stretched frames.
→ Swap the dimensions for ±90°/270° rotations, or decode PPM frames and take each frame's size from its header. Assert exact byte totals. Add a `-display_rotation 90` fixture.

**A3 · high · Video times are not on the audio timeline.** `VID-2`, `ORCH-13`
Times are `frame_index / avg_frame_rate`. That ignores stream start offsets (edit lists, AAC priming, MPEG-TS captures) and VFR (phone video), and the motion pass uses a different time origin from the shots. Which audio stream is analysed, and its offset against the video, are left to ffmpeg's defaults and never recorded. M6's planned `on_beat_cut_ratio` (a ±70 ms window) depends on this.
→ See P1-4. A cheaper first step: use scenedetect's timestamp-exact seconds plus an ffprobe packet-timestamp table, and shift everything by the start offset.

**A4 · medium · Motion between the two frames on either side of a cut is counted inside a shot.** `VID-3`
This inflates the mean motion and dominates the 95th percentile (p95) of short shots.
→ Keep frame pair k only if both samples map to the same shot. Land this after A3, because both need the same time origin.

**A5 · medium · The last shot can extend past the real end of the stream.** `VID-11`
When `nb_frames` is missing, the frame count falls back to duration × fps (`video.py:213-214`), which can overshoot and abort frame extraction.
→ Use scenedetect's own decoded end (`start_in_scene=True`), or `-count_packets`.

**A6 · medium · Colour fidelity.** `CRIT-3`, `VID-13`, `CRIT-8`, `CRIT-7`, `VID-12`
- HDR video (HLG or PQ transfer, BT.2020 primaries) is converted to RGB with no tone mapping.
- k-means runs in OpenCV's 8-bit Lab, which stretches lightness by 2.55× relative to the colour axes, so palettes over-split lightness and merge hues.
- Letterbox bars become a black swatch covering about 26 % of the frame.
- The pixel aspect ratio of anamorphic sources is ignored.
- Frame bytes depend on the system ffmpeg's YUV→RGB conversion, whose version is never recorded.

→ First record the colour metadata, the pixel aspect ratio and the ffmpeg version, and flag converted colours. Scale lightness by 100/255 before clustering, shipped with A1. Then tone-map HDR, crop the bars and apply the aspect ratio. Pin the conversion with `-sws_flags accurate_rnd+bitexact+full_chroma_int`.

**A7 · low · Motion precision.** `VID-14`, `DET-14`, `SCH-13`
At 3 dp, motion keeps about one significant digit in the schema's own "static" and "gentle" ranges, while the worked example uses 4 dp. The detector parameters are also written unrounded.
→ Use 4 dp for motion, or restate the scale, and round the detector parameters when config is loaded.

**A8 · medium · Memory use.** `SEC-6`, `VID-8`
All selected frames are decoded at full resolution into one buffer (twice that at peak) and kept for the whole run. Even `--estimate` pays this. A 4K or hour-long video needs many GB.
→ Stream the frames, keeping only the JPEG and a ≤256 px palette sample from each.

### B. Up-to-date checks, provenance and input identity

**B1 · high · Upstream changes never invalidate dependent documents.** `DET-1`, `ARCH-1`, `FEAT-2`, `LYR-4`, `ORCH-1`, `SCH-1`, `SEC-4`, `TEST-2`
Each stage checks whether it is already up to date against its own config hash only (`features.py:298-305`, `lyrics.py:463-470`), and `DocumentEntry` records no input identity. For example, change `[features] vocal_min_gap_seconds` (as the repo's own `mrw.toml` does) and re-run features. `mrw lyrics` then does nothing, leaving a `lyrics.json` whose untranscribed regions and coverage were computed from the old vocal regions. Re-separating the stems leaves features and lyrics marked `ok`. PLAN §5 promises that dependent documents go stale.
→ See P1-1.

**B2 · medium · `mrw status` never shows lyrics or video as stale.** `ORCH-11`
`cli.py:304-305` prints their raw status, although `STAGE_SECTION` maps both stages.
→ See P0-6, and add a status test.

**B3 · medium · Code and toolchain provenance is missing.** `ORCH-3`, `DET-8`, `ARCH-6`, `LYR-15`
Constants documented as changing only with `tool_version` never take effect, because `TOOL_VERSION` is fixed at `mrw 0.1.0` and is never compared. After a code fix, a library silently mixes documents from two algorithms. The versions of ffmpeg, the libraries and the platform are not recorded either.
→ See P0-0. Also record `tools: [{name, version}]` and `platform` in the run metadata via `importlib.metadata`, and keep the version in one place. Don't fold the algorithm version into `stage_hash`: that would force a Demucs re-separation of every track.

**B4 · medium · Ingest only ever runs once per track.** `ORCH-6`, `ARCH-9`, `DET-4`, `VID-10`, `LYR-13`
`mrw ingest song.mp4 --lyrics song.lrc` on an already-ingested track prints "no-op" and ignores the lyrics. A changed ingest config leaves `source=stale` for ever, and the remedy the video stage suggests (re-ingest with `copy_video`) can't work. Unsupported lyrics extensions are silently parsed as `.txt`.
→ See P1-6. Now: fix the video stage's remedy message and accept only `.lrc` and `.txt`.

**B5 · medium · The video stage never checks it is analysing the ingested file.** `DET-3`, `ORCH-2`, `VID-5`
`_resolve_video_path` uses whatever file is at `original_path`, or any `source_video.*`. A re-exported cut saved to the same path is analysed as if it were the original track.
→ After the up-to-date check, verify the file's SHA-256 against `file.sha256`; the hash cost is negligible next to the decode. Exit 2 on a mismatch.

**B6 · medium · Settings that don't change output are hashed.** `ORCH-15` (already known)
Changing the caption budget, `retain` or `copy_video` forces a re-run or leaves a permanent `stale`.
→ Hash each section with these operational fields reset to their defaults. Existing hashes don't change.

### C. Lyrics input parsing

**C1 · high · Compressed LRC lines lose their repeats.** `LYR-5`
For `[00:45.00][01:30.00][02:40.00]Chorus line`, only the first tag is kept as a hint and all tags are stripped (`lyrics.py:249-254`). The repeats at 1:30 and 2:40 become "untranscribed", and M6 loses its evidence that the chorus repeats.
→ Emit one line per leading tag, sharing the same source-line index, and stable-sort by (hint, index). Land this with D2, so that identical lines a few seconds apart don't collapse onto one occurrence.

**C2 · high · Lyrics decoding silently corrupts text.** `SEC-7`, `LYR-6`, `ORCH-14`
The file is read with `read_text(encoding="utf-8", errors="replace")`. A UTF-8 BOM survives `strip()`, because U+FEFF is not whitespace, so the first line's `[mm:ss]` tag doesn't match and ends up in the displayed text. UTF-16 and cp1252 files turn into U+FFFD replacement characters.
→ See P0-4.

**C3 · medium · LRC metadata and format variants.** `LYR-7`, `SCH-2`
- ID tags such as `[ar:]` become section markup labelled "other".
- `[offset:]` is ignored.
- `mm:ss:xx` timestamps stay in the displayed text.
- `♪` and unbracketed headers become sung lines.
- The ingest and lyrics regexes disagree.

→ See P0-3.

**C4 · medium · Token normalisation is too narrow.** `LYR-10`
There is no NFKC normalisation or case folding. Curly apostrophes and combining marks are stripped. Scripts without spaces between words, such as CJK, cannot align.
→ Apply NFKC, case folding and apostrophe folding. Use character-level tokens for scripts without spaces. Ship it as an algorithm-version bump.

**C5 · high · The parser is tested on one hand-picked format.** `TEST-3`
→ Add table-driven parser tests for every variant above: several tags per line, ID tags, offset, `mm:ss:xx`, BOM, CRLF, UTF-16, cp1252.

### D. Lyrics alignment

**D1 · high · Greedy alignment misplaces repeated sections.** `LYR-1`
For each unhinted line, the best-scoring window anywhere in a 400-word lookahead wins (`align.py:120-131`). The cursor then jumps past every word it skipped (`:153`), and `token_sort_ratio` ignores word order. A slightly misheard first chorus therefore loses to a clean second chorus about 60 words later. The rest of the song goes unaligned, while the misplaced lines carry confidence 1.0 and no flag. PLAN §4 promised a Needleman–Wunsch-style alignment.
→ See P1-2. A minimal interim patch with ε ≈ 0.1 would not fix the reported example; go straight to the dynamic program.

**D2 · high · Lines with timestamp hints ignore the cursor.** `LYR-2`
Two identical back-to-back lines with exact hints anchor to the same words, because ties go to the earliest match. When album-synced lyrics are used with a music video whose song starts 25 s later, lines mis-anchor or become 30-second lines. There is no fallback and no estimate of the global offset.
→ Never reuse a word already used by another line. Break ties by distance from the hint. Keep the window inside the hint range, with a cap on gaps (about 8 s). Estimate a global offset from confidently matched unique lines, and fall back to the cursor search.

**D3 · medium · Aligned mode drops transcript words that match no supplied line.** `CRIT-1`
Abbreviated lyric files ("Repeat chorus x2") lose whole sections, which are then reported as untranscribed.
→ Keep the unmatched Whisper words in an additive array that counts toward coverage, and parse repeat instructions as markup with a repeat count.

**D4 · medium · Word timings are assigned by position.** `LYR-8`
When Whisper drops or merges words, the supplied words inherit their neighbours' timings, without the `timing_interpolated` flag.
→ Align tokens by content (Levenshtein opcodes or `SequenceMatcher`). Interpolate and flag the rest.

**D5 · medium · Unanchored lines stack and run past the end of the audio.** `LYR-11`, `SCH-3`
Consecutive unanchored lines share one time span. Hints beyond the audio produce times greater than `duration_seconds`. A document with no anchored lines at all puts every line at 0–4 s. Vocal regions that run to the end can also overrun the duration.
→ Spread the lines across the gap and clamp every time to [0, duration]. Document the zero-anchor convention, and add a bounds assertion across documents to the M3 and M4 tests.

**D6 · medium · Short regions vanish from `untranscribed_regions`.** `CRIT-5`
A region whose only coverage is a zero-length word is dropped.
→ Drop zero-length spans before merging, and judge whether a region is uncovered by its covered length.

**D7 · medium · Aligned lines have no `source_line_index`.** `LYR-9`
M6 needs it to tie section headers to timed lines.
→ Add it as an optional field (lyrics 1.2.0).

**D8 · low · Calibration and example drift.** `CRIT-10`, `LYR-14`
`overlapping_vocals` is an uncalibrated absolute-loudness test that no test exercises. The lyrics example's confidence numbers disagree with the implementation.
→ Describe the flag honestly in the schema, add a test, and correct the example.

### E. Offline guarantee and external services

**E1 · high · Whisper contacts Hugging Face on every run, and the model revision is unpinned.** `LYR-3`, `SEC-2`, `DET-9`, `TEST-4`
`WhisperModel(cfg.model, ...)` (`lyrics.py:160-162`) resolves the model name through `huggingface_hub.snapshot_download(revision="main")`. Whenever the machine is online, that calls the Hub API, and if `main` has moved it silently downloads the new snapshot. A proxy error can fail the stage even when the model is already cached. CLAUDE.md permits only first-use weight downloads, and no test checks offline operation.
→ See P0-5, and run the slow lyrics tests with `HF_HUB_OFFLINE=1`.

**E2 · medium · The caption backend is not ready for paid batches.** `ARCH-12`, `VID-6`, `SEC-11`, `CRIT-12`, `DET-15`
- No test exercises the paid code path.
- A reply wrapped in a code fence, refused or truncated raises an error after the call has been paid for, so every re-run pays again and fails at the same shot.
- Replies are coerced rather than validated (a string `tags` becomes one tag per character), then cached for good.
- The cache key omits the prompt template.
- PLAN still promises temperature 0.
- An API key is required even when every caption comes from the cache.

→ See P2. Don't add `temperature=0` or `max_output_tokens` to the request or the cache key.

**E3 · medium · The determinism contract is overstated for MPS and across devices.** `DET-2`, `FEAT-9`, `TEST-5`, `DET-10`
The manifest schema and OQ-13 promise that documents are byte-identical across devices, because rounding "absorbs" jitter. A 4-minute `audio_features.json` holds roughly 240,000 rounded values, so some always sit within jitter of a rounding boundary, so ±1-sample differences in the stems (from MPS, the CPU fallback, or thread count) do reach the documents. All the determinism evidence comes from one process on Linux x86 CPU.
→ See P2. Also run the double-run tests in separate processes, with different `PYTHONHASHSEED` and thread-count environment variables.

### F. Audio features and stems

**F1 · high · Onset detection ignores signal level.** `FEAT-1`
librosa's onset strength doesn't change with gain: it is a dB ratio followed by min-max normalisation. A vocal stem that contains only dither, or drum leakage at −55 dB, reports hundreds of near-full-strength onsets, while its RMS and spectral centroid say it is silent.
→ Round the RMS first, drop onsets in frames at the published −80 dB floor, and gate the rest relative to the mix (stem minus mix level below a threshold set in config). Embed the parameters next to the data (checklist S4), as audio_features 1.1.0.

**F2 · high · `tempo.confidence` doesn't measure confidence.** `FEAT-3`
It is 1 − 4 × the coefficient of variation of the inter-beat intervals. librosa's beat tracker outputs a near-metronomic grid by construction, so in a re-implementation of the pinned code a drone, pink noise and white noise all score about 0.75–0.8, and a 174 BPM click track is tracked at 87 BPM with confidence 0.988. `trim=False`, which nothing explains, extends the grid through beatless intros.
→ Measure how much onset energy lines up with the beats compared with the envelope's mean, or with grids shifted by a third of a beat. Don't compare against half-beat midpoints, which fails on 8th-note hi-hats. Use `trim=True` or record the span actually tracked, and state in the schema that half- and double-tempo errors aren't detected.

**F3 · medium · Degenerate inputs.** `FEAT-8`, `DET-11`, `SCH-9`
Digital silence writes `bpm_global: 0.0`, which violates `exclusiveMinimum: 0`, and is recorded as `ok`. Inputs shorter than 400 ms fail inside pyloudnorm. Structure's beat indices are undefined for an empty grid.
→ Choose now between failing the stage loudly with a clear error and relaxing the schema. Relaxing it is a consumer-visible change, so justify the version bump.

**F4 · medium · The downbeat fit on the full mix lands on the snare backbeat in common patterns.** `FEAT-4` (plausible)
Nothing in the document says how ambiguous the choice was.
→ Add `beats.downbeat_confidence` (best phase versus second best). Evaluate a kick/bass-based fit.

**F5 · medium · Onset-strength normalisation saturates.** `FEAT-5` (already known)
Normalising by the 98th percentile over all envelope frames pushes channels with sparse onsets to 1.0.
→ Normalise by the 90th percentile of the detected onsets' raw values instead.

**F6 · medium · Stems edge cases.** `CRIT-4`, `CRIT-9`, `CRIT-6`
- 6-source Demucs models are accepted, and their guitar and piano stems are silently left out of "other".
- Samples beyond ±1.0 are clamped without any record.
- Multichannel sources are downmixed with ffmpeg's default matrix: the LFE channel is dropped, the centre channel falls 7.6 dB, and none of it is recorded.

→ Validate the model's sources when config is loaded. Count clipped samples in `stems.warning`. Prefer an existing stereo stream; otherwise downmix explicitly and record how.

**F7 · low · Precision and semantics polish.** `FEAT-6`, `FEAT-7`, `TEST-11`, `FEAT-11`, `FEAT-12`, `FEAT-13`
- bpm is quantised to the tempogram's bins.
- Series hold 1 + ⌊N/441⌋ values, not the documented ⌈duration × 100⌉.
- `mean_rms_db` averages dB values.
- The vocal-activity procedure is undocumented, and config accepts inverted thresholds.
- The centroid's silence test uses the unrounded RMS.

→ Take tempo from a regression of beat times on beat index. Fix the docs and add the series-length assertion. Average in the power domain. Document the procedure and validate the thresholds. Drive silence decisions from the published values.

### G. Schemas and serialization

**G1 · medium · Non-finite floats are written as invalid JSON.** `ARCH-7`, `DET-7`, `SCH-7`
`json.dumps` defaults to `allow_nan=True`. A NaN, such as M6's `on_beat_cut_ratio` on a video with no cuts, is written as the non-JSON token `NaN`. It passes the double-run test but makes Swift's `JSONDecoder` reject the whole document. `-0.0` also survives rounding.
→ See P0-7 and P0-8.

**G2 · medium · No test enforces the contract.** `TEST-10`, `SCH-6`
There is no model↔schema lockstep test, the examples are not validated in CI, the schemas accept undeclared keys, and `export-schemas` covers only 2 of the 5 models.
→ See P1-7. Pin the major version of `schema_version` with a pattern rather than `const`, so documents from older minor versions stay valid.

**G3 · medium · Configurable sample rate and channel count contradict the schema.** `SCH-10`
The schema fixes 44100 Hz and 2 channels, but `ingest.sample_rate` and `ingest.channels` can be set to anything.
→ Type them as `Literal[44100]` and `Literal[2]`. The model dumps the same values, so the config hash doesn't change.

**G4 · low · Manifest versioning on rewrite.** `SCH-5`, `DET-12`
A rewritten manifest keeps its old `schema_version` while new fields are written. Unknown keys are dropped, and an unknown enum value crashes.
→ Re-stamp the version on write and refuse manifests from newer versions. Map validation errors to exit 2, and have `status` skip manifests it can't read.

**G5 · low · Spec drift.** `SCH-11`, `SCH-8`, `SCH-12`
There are a few required/enum mismatches between the models and the schemas. The worked examples are internally inconsistent. Some descriptions misstate the method (`tempo.confidence`, `line_count`, `detector.threshold`).
→ Make targeted fixes and add an example-consistency test.

**G6 · low · `canonical.write` details.** `DET-13`
There is no fsync, documents are created with mode 0600, stale temp files are never removed, and the returned hash is computed over the text rather than the bytes written.
→ Remove stale `.<name>.*.tmp` files when a stage starts, and compare `content_sha256` in the up-to-date check. fsync is optional for documents that can be regenerated.

**G7 · low · `audio_features.json` is about 4.4 MB for a 4-minute track,** roughly 2.5× the documented size. `SCH-14` (already known)
→ Correct the docs. Revisit the format only if decode cost is measured to be a problem.

### H. Config, CLI and operations

**H1 · medium · Config errors are ignored or crash instead of failing cleanly.** `ARCH-2`, `ORCH-5`, `SEC-10`, `DET-5`, `TEST-15`
- A typo such as `devices = "cpu"` is silently ignored, so separation runs on MPS.
- Malformed TOML produces a traceback and exit 1.
- The operator's own `mrw.toml` (`device = "cpu"`, `vocal_min_gap_seconds = 1.0`, `language = "en"`) is committed at the repo root. It applies to anyone running from a checkout, and leaks into module-scoped test fixtures.

→ See P0-7. Add a session-scoped autouse fixture that changes into a temporary directory.

**H2 · medium · The exit-code contract leaks.** `ORCH-10`, `ARCH-8`, `VID-9`, `TEST-6`
- Missing ffmpeg gives a traceback and exit 1.
- One corrupt manifest crashes `status` for the whole library.
- A missing API key or unpriced model is recorded as a stage failure only after the full decode, overwriting an `ok` entry.
- `--estimate` can modify the manifest.

→ Add an error hierarchy that carries exit codes, plus one CLI decorator to map it. Run cheap pre-checks (`shutil.which`, API key, price table) before any work. Add parametrised failure-path tests.

**H3 · medium · Concurrent runs lose manifest updates.** `ORCH-4`, `ARCH-5`, `DET-6`, `SEC-8`
Each stage reads the manifest at the start and rewrites the whole file at the end, so two commands on one track undo each other's entries, including API spend.
→ See P1-5.

**H4 · medium · The cost ledger is overwritten on every re-run.** `ORCH-7`, `SCH-4`, `VID-7`
A run that fails partway and is then retried under-reports what it spent.
→ Use an append-only `api_ledger`, seeded from the existing entry on the first write.

**H5 · medium · Interruption isn't recorded.** `SEC-9`
Ctrl-C or SIGTERM skips failure recording, so paid spend disappears. An aborted `copy_video` leaves a truncated `source_video.*`, which is later preferred over the intact original.
→ See P1-5. Use a copy only if its size matches `source.json`.

**H6 · medium · Subprocesses can hang.** `SEC-5`, `ORCH-9`
The PCM-hash decode reads ffmpeg's stdout and reads stderr only at the end (`ingest.py:111-116`), so it deadlocks once ffmpeg writes more than about 64 KB of errors. Damaged media can do that. No subprocess has a timeout. Decode errors with exit code 0 are discarded, so truncated audio passes silently.
→ See P0-9. Also compare the decoded duration against the container's, and record decode errors.

**H7 · medium · A failed ingest leaves a half-built track directory.** `ORCH-12`
`status` never shows the directory. An `avg_frame_rate` of `"0/0"` crashes after the directory has been created.
→ Build the track in a staging directory and rename it into place, and parse frame rates defensively.

**H8 · medium · Tags are read only from `format.tags`, with lowercase keys.** `CRIT-2`
FLAC (uppercase keys), Ogg/Opus (tags on the stream) and MKV (`ARTIST`) lose their title and artist.
→ Merge tags case-insensitively (stream tags for Ogg only), and add `--artist`.

**H9 · low · Untrusted tags and filenames are echoed raw to the terminal,** which allows escape-sequence injection. `SEC-14`
→ Add one `_safe()` helper for every echo.

**H10 · low · Performance.** `SEC-12`, `CRIT-11`
Whole-file float32 audio decodes make avoidable copies: Demucs peaks near 18 GB on a 1-hour input. Clipped transcription pays a full 30 s encoder pass per vocal clip.
→ Normalise in place (check the result is bit-identical against current outputs), add a duration guard, and measure clip counts before changing clipping.

### I. Tests and CI

**I1 · high · No CI check blocks a merge.** `TEST-1`, `ARCH-3`
The only workflow runs the Claude review action, and the agent is asked to run pytest and report the result. The job's status never depends on the tests, `main` is never re-tested after a merge, and PRs from forks get no tests at all.
→ See P0-11. Making the check required is a branch-protection setting rather than a file, so record the decision in a review note.

**I2 · medium · Smoke tests are weaker than PLAN §11, and the deviations are unrecorded.** `TEST-8`, `VID-15`, `TEST-9`, `LYR-12`, `FEAT-10`
- **M5:** 2 cuts instead of 10, a ±3.6-frame tolerance instead of 1, colour checks that would accept a colour difference (ΔE) of about 83 instead of under 5, and no moving scene.
- **M4:** the aligned path never runs on real audio, the overlap check covers 3 of 5 words, and the music-only check is vacuous.
- **M3:** no test of a failure partway through the stage.

These best-case fixtures are why A1, F1 and F2 went unnoticed.
→ See P1-8.

**I3 · medium · No end-to-end determinism run, and the paid path is never executed.** `TEST-7`
→ Add a slow end-to-end double run into two library roots, via subprocesses, compared with `manifest.json` excluded. Add a fast AnthropicBackend test against a stub endpoint (`ANTHROPIC_BASE_URL` or an injected client).

**I4 · medium · T5 can't be verified.** `TEST-12`
CI installs every dependency, so it can never show that the fast suite runs without the neural ones.
→ Run the fast job with `uv sync --locked --no-install-package torch …` followed by `uv run --no-sync pytest`, or add an import guard in `conftest`.

**I5 · medium · The review workflow needs hardening.** `TEST-13`, `SEC-13`
Actions are pinned to tags that can move. `Bash(uv run:*)` lets the agent run arbitrary commands in a job that holds the OAuth token. `uv sync` runs without `--locked`. Cache keys never refresh.
→ See P2. Also set `persist-credentials: false`.

**I6 · medium · The Python range and torch pin.** `TEST-14`, `ARCH-4`
`requires-python = ">=3.11"` admits 3.13, which has no torch 2.5.1 wheel on macOS arm64. The torch pin exists because of demucs 4.0.1, and torch < 2.6 is also flagged by security scanners (CVE-2025-32434). The practical exposure is narrow, because the checkpoints come from hash-named official URLs.
→ See P0-10 now and the P2 item later.

### J. Architecture, docs and roadmap

**J1 · low · Duplication.** `ARCH-13`, `ARCH-14`
About 250 lines of stage-lifecycle scaffolding are repeated across five `run_*` functions. There are three ffprobe wrappers, and two LRC regexes that disagree.
→ See P2: shared helpers first, and a full Stage protocol only once M6 needs one.

**J2 · low · Docs have drifted from the code.** `ARCH-15`, `SCH-15`, `CRIT-13`
- `PLAN.md:3` still says "No implementation exists yet".
- PLAN §3's module layout (a `video/` package, `timeseries.py`, `structure.py`) differs from the real one.
- The README omits commands.
- OPEN_QUESTIONS still says "nothing is implemented yet".
- M5 has no review record and no OQ resolution notes.

→ One docs PR, plus `docs/reviews/008-m5-video.md`.

**J3 · medium · M6's prerequisites are undecided.** `ARCH-10`
This covers conventions for zero cuts and empty grids, whether lyrics are optional, the quality of lyric markup (ID tags), and staleness.
→ Record them as OQ-17 to OQ-20 before writing structure code. Fix the ID-tag and silent-mix bugs (C3, F3) as separate small PRs now.

**J4 · low · M7 operability gaps.** `ARCH-11`
There is no `analyze` or batch command. `stems.retain = false` makes later stages impossible to run, because no combined run exists. There is no `--offline` or `--json` and no environment check.
→ Reject or warn on `retain = false` now. Plan `mrw analyze [--all] [--offline] [--json]` on the shared stage helpers.

---

## 4. Rejected findings

- `FEAT-14`: whole-file buffers need several GB for multi-hour inputs. True, but outside the project's target material.
- `SEC-15`: passing a path re-hashes the whole file and ingest reads the source three times. True, but the cost is negligible at typical sizes.

## 5. Caveats

- Nothing was executed. Claims about third-party behaviour (librosa, faster-whisper / huggingface_hub, OpenCV, scenedetect) come from reading the pinned source versions in `uv.lock` and from small numpy re-implementations. They are marked "plausible" in the detail file where they could not be fully confirmed: `FEAT-4`, `ORCH-9`, `ORCH-13`, `VID-12`, `SEC-13`, `SEC-11`, `CRIT-9`, `CRIT-11`, `FEAT-9`.
- Verifiers frequently lowered severity (40 findings were filed as high; 23 remain high). Some disagree with the original proposal, and in those cases this report follows the verifier's correction. Examples: don't fold the algorithm version into `stage_hash`; hash-check the video unconditionally rather than caching by mtime; don't merge the two video decodes with a split filter.
