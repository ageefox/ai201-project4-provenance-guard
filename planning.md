# planning.md — Provenance Guard

## Architecture

### Submission Flow

```
POST /submit (text, creator_id)
        |
        v
  [Input Validation]
        |
        v
  [Signal 1: LLM Classifier (Groq)]   ← semantic/stylistic coherence
        |
        v
  [Signal 2: Stylometric Heuristics]  ← structural uniformity (sentence variance, TTR, punctuation)
        |
        v
  [Signal 3: Burstiness Score]        ← vocabulary distribution patterns
        |
        v
  [Ensemble Confidence Scorer]        ← weighted combination → single score 0.0–1.0
        |
        v
  [Transparency Label Generator]      ← maps score to plain-language label text
        |
        v
  [Audit Logger]                      ← writes structured JSON entry
        |
        v
  JSON Response: {content_id, attribution, confidence, signal_scores, label}
```

### Appeal Flow

```
POST /appeal (content_id, creator_reasoning)
        |
        v
  [Look up original entry in audit log / storage]
        |
        v
  [Update status: "classified" → "under_review"]
        |
        v
  [Append appeal entry to audit log]
        |
        v
  JSON Response: {message: "Appeal received", content_id, status: "under_review"}
```

### Additional Endpoints

```
GET  /log              → returns most recent N audit log entries (JSON)
GET  /dashboard        → returns analytics summary (stretch)
POST /verify           → initiates provenance certificate flow (stretch)
GET  /content/{id}/certificate  → returns certificate status (stretch)
```

---

## Detection Signals

### Signal 1: LLM Classifier (Groq — llama-3.3-70b-versatile)

**What it measures:** Holistic semantic and stylistic coherence. The model evaluates whether the text reads as AI-generated based on patterns it has learned — things like overly balanced sentence structure, hedged phrasing ("it is important to note"), generic transitions ("furthermore," "in conclusion"), and the kind of smooth, uniform flow that comes from a language model predicting the next token rather than a human expressing a specific thought.

**Output format:** A float between 0.0 and 1.0, where 1.0 = high confidence the text is AI-generated. Extracted from a structured JSON response prompted from the model.

**Prompt strategy:** Ask the model to return `{"ai_probability": <float>, "reasoning": "<string>"}` only. Temperature set to 0 for consistency.

**What it misses:** Lightly edited AI text (a human who pastes AI output and tweaks a few words will often fool this). Very short texts give the model too little signal. Non-native English speakers writing formally can trigger false positives.

---

### Signal 2: Stylometric Heuristics (pure Python)

**What it measures:** Statistical structural properties that differ between human and AI writing. AI text tends to be more uniform — consistent sentence lengths, moderate vocabulary diversity, steady punctuation rhythm. Human writing is messier and more variable.

**Metrics computed:**
- **Sentence length variance:** Standard deviation of word-count per sentence. Low variance → more AI-like.
- **Type-token ratio (TTR):** Unique words / total words. AI text clusters around 0.55–0.70; human text varies more widely.
- **Punctuation density:** Non-alphanumeric characters as a fraction of total characters. AI text tends to be punctuation-moderate; human casual writing is more variable.
- **Average sentence length:** AI text averages ~18–25 words/sentence; human writing spans more widely.

**Output format:** A float between 0.0 and 1.0 computed by normalizing and averaging the four metric scores, where 1.0 = high confidence AI-generated.

**What it misses:** Formal human writing (academic papers, legal briefs) scores high on uniformity metrics and gets false-positive AI signals. Very short texts (< 3 sentences) don't have enough data for variance to be meaningful.

---

### Signal 3: Burstiness Score (pure Python)

**What it measures:** Vocabulary burstiness — the tendency of words to appear in local clusters. In human writing, if you use the word "grief" once, you're likely to use it again nearby; then it drops away. AI models distribute vocabulary more evenly across a text because they're predicting each token in context without the kind of thematic "fixation" humans have.

**How computed:** For each content word, compute the intervals between its occurrences in the text. Low variance in those intervals (evenly spaced) = AI-like. High variance (bursty) = human-like. Aggregate across the top-N most frequent content words. Invert so that 1.0 = AI-like.

**Output format:** A float between 0.0 and 1.0.

**What it misses:** Short texts don't have enough word repetitions to compute meaningful intervals. Repetition-heavy genres (poetry, chants, certain literary styles) will appear "bursty" but may still be AI-generated.

---

## Ensemble Confidence Scoring

### Weighting

| Signal | Weight | Rationale |
|--------|--------|-----------|
| LLM Classifier | 50% | Most holistic signal; captures semantic patterns no heuristic can |
| Stylometric Heuristics | 30% | Robust on longer texts; independent of LLM API availability |
| Burstiness Score | 20% | Useful supplement; less reliable on short texts |

**Formula:**
```
confidence = (0.50 × llm_score) + (0.30 × stylometric_score) + (0.20 × burstiness_score)
```

### Uncertainty Representation

A score of 0.60 means: the signals collectively lean toward AI-generated but with meaningful uncertainty — the system is not confident enough to label this definitively. This score should produce an "uncertain" label, not an AI label.

**Thresholds (designed with false-positive asymmetry in mind):**

| Score Range | Attribution | Rationale |
|-------------|-------------|-----------|
| ≥ 0.70 | `likely_ai` | Strong multi-signal agreement needed before labeling human work as AI |
| 0.40 – 0.69 | `uncertain` | Any ambiguity defaults to uncertain |
| < 0.40 | `likely_human` | Must be clearly human-like across signals |

The thresholds are intentionally asymmetric: it takes a higher signal consensus to call something AI-generated (≥ 0.70) than to call it human (< 0.40). This reflects the fact that false positives (accusing a human creator of using AI) cause more harm than false negatives.

### How Scores Are Validated

- Test with 4+ known inputs: one clearly AI, one clearly human, two borderline
- Print individual signal scores alongside combined score to identify which signal misbehaves on edge cases
- Check that the full 0.0–1.0 range is reachable (not collapsed to 0.45–0.65)
- Adjust signal weights if one signal dominates unexpectedly

---

## Transparency Label Variants

All three variants are written for a non-technical reader. No jargon ("logit," "classifier," "probability") appears in label text.

**High-confidence AI (confidence ≥ 0.70):**
```
"Our system's analysis suggests this content was likely AI-generated. This label is
based on automated analysis and may not be accurate. If you are the creator and
believe this is wrong, you can submit an appeal."
```

**Uncertain (confidence 0.40–0.69):**
```
"Our system could not confidently determine whether this content was written by a
human or generated by AI. The analysis is inconclusive. If you are the creator,
you may submit an appeal to provide additional context."
```

**High-confidence human (confidence < 0.40):**
```
"Our system's analysis suggests this content was likely written by a human.
Attribution labels help readers understand the origin of creative work."
```

---

## Appeals Workflow

**Who can appeal:** Any creator who submitted the content (identified by `creator_id` matching the original submission's `creator_id`). For simplicity in this implementation, the endpoint accepts any appeal for a valid `content_id` without authentication — noted as a known limitation.

**What they provide:**
- `content_id` (required) — the ID from the original `/submit` response
- `creator_reasoning` (required) — free-text explanation from the creator

**What the system does on appeal:**
1. Looks up the original audit log entry by `content_id`
2. Updates its status from `"classified"` to `"under_review"`
3. Appends a new audit log entry of type `"appeal"` with: `content_id`, `creator_reasoning`, `timestamp`, `original_attribution`, `original_confidence`
4. Returns `{"message": "Appeal received", "content_id": "...", "status": "under_review"}`

**Automated re-classification:** Not implemented. A human reviewer would use `GET /log` to see all entries with `status: "under_review"`.

**What a reviewer sees in the log:**
- The original entry with `attribution`, `confidence`, all signal scores
- The appeal entry with the creator's reasoning
- The status field updated to `"under_review"`

---

## Anticipated Edge Cases

**1. Formal academic human writing**
A human writing in academic register — complex vocabulary, long sentences, hedged claims — will score high on both the stylometric and LLM signals because it structurally resembles AI output. Example: a PhD student submitting a passage from their thesis. This will likely land in the "uncertain" band or even trigger a false "likely_ai" label. Mitigation: the appeals workflow is the primary recourse.

**2. Short texts (< 50 words)**
A haiku, a tweet-length post, or a brief caption gives the stylometric and burstiness signals almost nothing to work with. Sentence length variance from 3 sentences is not statistically meaningful. The system should flag short texts in the response (`"short_text_warning": true`) and default toward "uncertain" regardless of LLM score, to avoid confident mislabeling.

**3. Lightly edited AI output**
A creator who uses AI to generate a draft and then edits it substantially will produce text that the LLM may partially recognize but the stylometric signal might score as human (due to the edits introducing variance). This will likely land in the uncertain band, which is arguably the correct behavior — the content is genuinely hybrid.

**4. Non-native English speakers writing formally**
Formal, grammatically careful writing from a non-native speaker can look stylometrically uniform (careful word choice, consistent sentence structure from following grammar rules consciously). This is a known false-positive risk documented in the label text itself.

---

## Stretch Features

### Ensemble Detection
Three signals (LLM, stylometric, burstiness) with documented weights (50/30/20). Individual signal scores returned in every API response alongside the combined score.

### Provenance Certificate
A creator earns a "Verified Human" certificate by completing a manual review step. Implementation: `POST /verify` submits their `creator_id` and a brief statement; an admin endpoint `POST /admin/approve_certificate` marks them as verified. Verified status is stored per `creator_id`. When verified creators submit content, the response and label include a `certificate` field and a distinct label variant.

**Certificate label variant:**
```
"This content was created by a Verified Human creator. The creator has completed
an additional verification step confirming human authorship."
```

### Analytics Dashboard
`GET /dashboard` returns:
- `verdict_distribution`: count of `likely_ai`, `uncertain`, `likely_human` verdicts
- `appeal_rate`: appeals / total submissions
- `avg_confidence`: mean confidence score across all submissions
- `signal_agreement_rate`: fraction of submissions where all 3 signals agreed on verdict direction

### Multi-Modal Support
A second content type: **image description metadata** (structured JSON describing an image — alt text, EXIF data presence, filename patterns). Signal approach: heuristic analysis of whether the description reads as auto-generated alt text vs. human-written. Endpoint: `POST /submit` accepts `content_type: "image_metadata"` and a `metadata` field instead of `text`.

---

## AI Tool Plan

### Milestone 3 — Submission endpoint + Signal 1 (LLM)
**Spec sections to provide:** Detection Signals (Signal 1), Architecture diagram (submission flow)
**What to ask for:** Flask app skeleton with `POST /submit` route stub; `classify_with_llm(text)` function that calls Groq and returns a float
**How to verify:** Call the function directly with the 4 test inputs from the spec; confirm output is a float in [0, 1]; check the route returns `content_id`, `attribution`, `confidence`, `label` fields

### Milestone 4 — Signals 2 & 3 + confidence scoring
**Spec sections to provide:** Detection Signals (Signals 2 & 3), Ensemble Confidence Scoring, Uncertainty Representation
**What to ask for:** `compute_stylometric_score(text)` function; `compute_burstiness_score(text)` function; `combine_signals(llm, stylo, burst)` function using the 50/30/20 weights
**How to verify:** Run all three functions on the 4 test inputs independently; confirm combined scores span the full threshold range; check that clearly AI text scores ≥ 0.70 and clearly human text scores < 0.40

### Milestone 5 — Production layer (labels, appeals, rate limiting, audit log)
**Spec sections to provide:** Transparency Label Variants, Appeals Workflow, Architecture diagram (appeal flow)
**What to ask for:** `generate_label(confidence, certificate_status)` function; `POST /appeal` endpoint; Flask-Limiter setup on `/submit`; structured audit log write/read helpers
**How to verify:** Submit inputs that hit all three label variants; test `POST /appeal` with a real `content_id`; confirm `GET /log` shows the appeal entry with `status: "under_review"`; run the rate-limit bash loop and confirm 429s after limit is hit
