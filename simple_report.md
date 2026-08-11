# Sanskrit OCR Correction — Report

## 1. Problem
OCR tools misread a lot of Sanskrit text because of its complex letter
combinations. This project builds a small AI model that takes OCR text with
mistakes and fixes it back to correct Sanskrit.

## 2. Dataset
- Given: 5 pages, each with an image and its correct text.
- Pages 1–4: used to teach the model.
- Page 5: kept aside, used only to test the model at the end.
- Since 5 pages is too little to train on directly, we created hundreds of
  extra practice examples by adding realistic OCR-style mistakes to the
  clean text from pages 1–4.

## 3. Model Used
**google/byt5-small** — a small model that reads text character by character.
We picked it because OCR mistakes create unusual character combinations, and
this model doesn't rely on a fixed word dictionary, so it isn't thrown off
by them.

## 4. Fine-Tuning Approach
- Trained the model for 10 epochs on our practice examples.
- Used the version of the model that did best on a small validation set
  (not just whichever finished last).

## 5. Hardware Used
- Google Colab, T4 GPU.
- Mixed precision (fp16) used to train faster.

## 6. Results

**Baseline: how bad is raw OCR, before any correction?**
(Measured directly by running OCR on all 5 pages.)

| Page | Character Error Rate |
|---|---|
| Page 1 | 0.16 |
| Page 2 | 0.11 |
| Page 3 | 0.14 |
| Page 4 | 0.12 |
| Page 5 (test) | 0.15 |
| **Average** | **0.14 (14% of characters wrong)** |

**After correction (page 5, unseen during training):**

| | Character Error Rate |
|---|---|
| Before correction | 0.14 |
| After correction | **1.80 (worse than doing nothing)** |

The model made the text *worse*, not better. This is an important, honest
result — not something to hide in the report.

## 7. What Works / What Doesn't

**What went wrong:** the model got stuck repeating the same word or phrase
over and over instead of producing a normal sentence. For example:

- Ground truth: `रोग निवारणम् सम्यक् चिकित्सा द्वारा भवति`
- Model output: `... सम्यक्‌ चिकित्‌ चिकित्‌ चिकित्‌ चिकित्‌ चिकित्‌ चिकि ... भवति भवति भवति भवति भवति भवति भवति भवति भवति भवति भवति भवति भवति भवति`

The word `भवति` alone was repeated 14 times in a row. The same pattern shows
up on the other lines too (`कारणम्` repeated 6 times, `आरोग्य कारणम्‌`
repeated 4 times). This is a well-known failure mode for small generative
models called **repetition looping / degeneration** — the model gets caught
in a cycle where its own most-likely next word is the word it just
produced.

**Why this likely happened here:**
- **Very little real training data (only 5 pages).** With so few examples,
  the model can end up learning shortcuts like "just repeat the last token"
  rather than genuinely learning correction.
- **Training for too many epochs (10) on a tiny dataset** — the model had
  enough passes over the same small set of examples to memorize surface
  patterns rather than generalize.
- Note the raw OCR line itself was actually already fairly close to
  correct for parts of this page (e.g. the OCR for the last line was only
  missing one word, `द्वारा`) — so the "before" baseline had more useful
  signal than the model ended up using.

**What this tells us:** for this data scale, the *fine-tuned model is
currently unsafe to use as-is* — a system that quietly makes OCR output
worse is more dangerous in practice than one that does nothing, since a
user might trust the "corrected" output without checking it.

## 8. Challenges
- Only 5 real labeled pages — had to rely mostly on made-up (synthetic)
  mistakes to get enough training data.
- Sanskrit OCR language support isn't built into Tesseract by default — had
  to download it separately.
- **The trained model degenerated into repeating words instead of
  correcting text** (see Section 7). This is the single biggest problem
  found in this project, and points to the training setup — not the
  overall approach — needing rework.

## 9. What We'd Improve With More Time
- **Fix the repetition problem first**, before anything else. Likely fixes:
  fewer training epochs (e.g. 3–5 instead of 10), a repetition penalty
  during generation (`repetition_penalty` / `no_repeat_ngram_size` in
  `model.generate(...)`), and/or more real training data so the model has
  less reason to fall back to memorized shortcuts.
- More real scanned pages, instead of relying mostly on synthetic mistakes.
- Compare this model against a word-dictionary-based model to see if the
  character-based approach really helps once the repetition issue is
  fixed.
