# ⚖️ CrimeKG Legal QA Assistant

A legal Q&A tool: type a question in plain English, it classifies the legal
category (labour, tenancy, cybercrime, etc.), retrieves the closest matching
question from a curated knowledge base, and returns the stored guidance —
through a proper web chat UI, not just a terminal loop.

This project builds on an earlier prototype's basic idea (classify a legal
question, then look up a canned answer) but has been substantially rewritten:
a fragile, dead-end retrieval pipeline replaced with a resilient one that
recovers gracefully instead of returning "no answer found", a small but
richer knowledge base, and a full web UI added on top of what was
originally a bare terminal script. Every file below is actively used —
no unused legacy code is included.

## Quick start

```bash
pip install -r requirements.txt
python app.py
```

Then open **http://127.0.0.1:5000** in your browser.

Prefer the terminal? `python qa_assistant.py` still works the same way.

## How it works

1. **Classify** — `question_classify.py` loads a trained TF‑IDF +
   LogisticRegression pipeline (`model/question_text.model`) that predicts
   one of 11 legal categories, along with a confidence score. It's trained
   on `data/question_train.csv` (380 example phrasings — formal, casual,
   and typo'd — across the 11 categories).
2. **Retrieve** — `qa_assistant.py` vectorizes your question and the
   knowledge base (`data/qa_pairs.json`, 154 question/answer entries) with
   a stemmed TF‑IDF, then ranks matches by cosine similarity.
   - If the classifier is confident, the search is scoped to that category
     first.
   - If nothing scores highly enough there (or the classifier wasn't
     confident), the search automatically **widens to the whole knowledge
     base** instead of giving up.
   - Widened results still get a small boost toward the classifier's top
     guess (even when it wasn't confident enough to hard-filter on) — this
     stops a coincidental keyword overlap in the wrong category from
     narrowly beating the actually-relevant answer.
   - Widened results are then filtered to stay within one coherent category
     (the best match's own category), so you don't get an unrelated answer
     tacked on just because it also cleared the similarity bar.
   - If still nothing matches closely, you get general guidance for the
     predicted category rather than a dead end — and if the question
     doesn't look like it's about any covered legal topic at all, it says
     so plainly instead of guessing.
3. **Serve** — `app.py` is a small Flask app exposing `POST /api/ask` and a
   chat UI (`templates/index.html`, `static/`).

## Project structure

```
CrimeAssistant/
├── app.py                    # Flask web server
├── qa_assistant.py           # retrieval engine (+ terminal CLI)
├── question_classify.py      # loads and queries the trained classifier
├── question_train.py         # trains/retrains the classifier
├── check_labels.py           # quick label-count sanity check
├── templates/index.html      # chat UI
├── static/style.css          # UI styling
├── static/script.js          # UI logic
├── data/
│   ├── qa_pairs.json         # knowledge base (question → type → answers)
│   └── question_train.csv    # labeled training data for the classifier
├── model/question_text.model # trained classifier (joblib)
└── requirements.txt
```

## Retraining the classifier

If you edit `data/question_train.csv`, retrain with:

```bash
python question_train.py
```

This re-fits the pipeline against your currently installed scikit-learn
version and overwrites `model/question_text.model`, and prints a
cross-validated accuracy report so you can see how well it's generalizing.
Retraining after any Python/library upgrade also avoids scikit-learn
version-mismatch warnings (and occasional bad predictions) from loading a
model pickled by a different version.

## Extending the knowledge base

Add entries to `data/qa_pairs.json` in this shape:

```json
{
  "question": "My landlord is refusing to return my security deposit.",
  "type": "Housing / tenancy",
  "answers": [
    "Review your rental agreement and send a written notice.",
    "Seek legal advice if the deposit is withheld without justification."
  ]
}
```

`type` must be one of the categories the classifier knows about — run
`python check_labels.py` to see the current list and how many training
examples back each one. No retraining needed for knowledge-base changes
only — only changes to `question_train.csv` require re-running
`question_train.py`.

## Notes

- This tool gives general information, not legal advice — the UI says so,
  and that framing should stay in place if you extend it.
- `check_labels.py` is a convenience script, not required for the app to
  run — safe to remove if you don't need it.
