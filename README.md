<div align="center">

# RelayGuard

### Real-time fraud risk scoring on IBM LinuxONE

Catch more fraud without flooding analysts: context-aware scoring, exact explanations and an analyst review workflow.

[![IBM Z Datathon 2026](https://img.shields.io/badge/IBM%20Z%20Datathon-2026-0f62fe)](#hackathon)
[![IBM LinuxONE](https://img.shields.io/badge/IBM%20LinuxONE-s390x-161616)](https://www.ibm.com/linuxone)
[![FastAPI](https://img.shields.io/badge/FastAPI-backend-009688?logo=fastapi&logoColor=white)](https://fastapi.tiangolo.com/)
[![scikit-learn](https://img.shields.io/badge/scikit--learn-HistGradientBoosting-F7931E?logo=scikitlearn&logoColor=white)](https://scikit-learn.org/stable/modules/generated/sklearn.ensemble.HistGradientBoostingClassifier.html)
[![SQLite](https://img.shields.io/badge/SQLite-storage-003B57?logo=sqlite&logoColor=white)](https://www.sqlite.org/)
[![Data: PaySim](https://img.shields.io/badge/data-PaySim%20(synthetic)-6f6f6f)](https://www.kaggle.com/datasets/ealaxi/paysim1)

**Theme: Real-time AI for Critical Decisions**

![RelayGuard dashboard with an alert open](docs/screenshots/dashboard-alert.png)

</div>

> **Prototype only.** RelayGuard runs on simulated PaySim payments. No real funds move, and the results below are on synthetic data, so they don't show real-bank performance.

---

## About

Banks have to approve or flag payments within moments, and two costs pull against each other:

- **Missed fraud** is direct financial loss and harm to the customer.
- **False alerts** burn analyst time and erode trust in the alert queue.

Fraud is also a tiny fraction of traffic (about 0.13% of PaySim rows), so accuracy is a misleading metric. Scoring a payment on its own ignores what the sender, the recipient and the wider payment stream have recently done.

**Research question:** does adding strictly past account and stream context reduce missed fraud at a comparable false-alert rate?

RelayGuard answers it with a bank-facing prototype that scores incoming payments, explains every score, and lets an analyst record a decision. The whole system runs as one Python service on IBM LinuxONE.

## Results

Both models were evaluated on the same fresh holdout: **24,264 payments, 268 frauds**.

| Metric | Previous V2 | Sequence v4 |
|---|---:|---:|
| Frauds caught (TP) | 128 / 268 | **204 / 268** |
| False alerts (FP) | 112 | **8** |
| Missed frauds (FN) | 140 | 64 |
| Legitimate cleared (TN) | 23,884 | 23,988 |
| Recall | 47.8% | **76.1%** |
| Precision | 53.3% | **96.2%** |
| F1 | 0.504 | **0.850** |
| Average precision | 0.475 | **0.878** |
| False-positive rate | 0.467% | **0.033%** |
| Threshold | 0.8977 | 0.9859 |

- Recall rose from 47.8% to 76.1% while precision rose from 53.3% to 96.2%.
- False alerts fell from 112 to 8, a large cut in analyst workload.
- Validation F1 (0.851) is close to test F1 (0.850), which supports consistent model selection.

**Honest caveats:** v4 still misses 64 of 268 frauds. PaySim is synthetic, and stream context may pick up regularities of the generator. No tuning was done after the final test.

## Features

| Feature | What it does |
|---|---|
| **Live payment stream** | Replays held-out PaySim payments one at a time, each scored with trusted past-only stream context |
| **Manual test payment** | Score your own payment: ID, step, type, sender, recipient and amount |
| **CSV training upload** | Upload a labelled CSV (up to 40 MB / 150,000 rows) to retrain a generic context model |
| **Adjustable alert threshold** | Drag the threshold and watch alerts change. The override persists for the same model version |
| **Why this score?** | Exact grouped Shapley breakdown showing which feature groups raised or lowered the score |
| **Analyst review** | Mark an alert *Needs investigation* or *Dismissed*, with an optional note |
| **Review CSV export** | Export active or archived reviews, with formula-injection protection |
| **Health endpoint** | `/api/health` reports status, model version, machine and system as LinuxONE evidence |

### Analyst workflow

1. **Open RelayGuard.** Demo workspace, no login in the prototype.
2. **Choose input.** PaySim replay, manual payment or CSV training upload.
3. **Backend scores.** Validate, build past-only features, run the saved model.
4. **Dashboard updates.** Score, threshold decision, payment details and latency.
5. **Open the alert.** Read the exact grouped Shapley breakdown.
6. **Record a decision.** *Needs investigation* or *Dismissed*, plus an optional note.

Decisions never overwrite dataset labels and never move money.

### Safety rails

- Duplicate transaction IDs and out-of-order steps are rejected.
- Review notes are capped at 1,000 characters.
- CSV export neutralises formula injection (`=`, `+`, `-`, `@`).
- No payment is executed. Records are attempts, not settlements.

## Try the browser demo

`index.html` is a self-contained demo of the analyst dashboard. Open it in any modern browser, or publish it with GitHub Pages.

In the demo, **a small rule-based scorer stands in for the LinuxONE gradient-boosting model**, and the replay uses ten sample PaySim-style payments. The interface, workflow, safety rails and review export match the full service.

| Score payments and open an alert | Export the review log |
|---|---|
| ![Dashboard](docs/screenshots/dashboard-alert.png) | ![Review CSV export](docs/screenshots/review-export.png) |

## Architecture

One portable Python service. The frontend, API, SQLite database and CPU model all share one origin.

```
 Browser UI          FastAPI              Features               Model                    SQLite
 HTML/CSS/JS  ──▶  validate, route,  ──▶  History +       ──▶  HistGradientBoosting ──▶  events, archive,
                   lock                   StreamContext         (uncalibrated score)      policy
```

Each request is validated, then `History` and `StreamContext` build strictly past-only features. The saved model returns a risk score, the threshold turns it into an alert decision, and the record is stored in SQLite with its feature snapshot and model fingerprint.

| Component | Responsibility | Design choice |
|---|---|---|
| `app.py` | API, state, review, export, retrain, health | Threading lock; one worker because history lives in memory |
| `engine.py` | Validation, History, StreamContext, training, threshold policy | Labels and IDs never enter features; whole time steps stay in one split |
| `explanations.py` | Exact grouped Shapley explainer | Computed on demand, cached with the event, refuses mismatched models |
| `train_sequence.py` | Reproducible v4 training and promotion gate | Candidates chosen on validation; test opened once after selection |
| `run.py` / `DEPLOY.md` | Portable launcher and LinuxONE hand-off | `127.0.0.1:8000`, `OMP_NUM_THREADS=2`, SSH tunnel access |
| `test_app.py` | Automated tests | Replay, review, export, leakage, explanations |
| `EVALUATION.json` | Recorded evaluation protocol and results | Reproducible metrics |

### Why LinuxONE fits

- **No x86-specific code.** Pure Python, scikit-learn and SQLite.
- **Native training.** Train on s390x with the PaySim archive. Retrain if scikit-learn versions differ.
- **No external services.** No cloud API, API key or Node build. The UI makes no required external requests.

## Data and leakage prevention

**Dataset:** [PaySim](https://www.kaggle.com/datasets/ealaxi/paysim1), synthetic mobile-money data by Edgar Lopez-Rojas. 6,362,620 records, 8,213 frauds. Each time step is one simulated hour. `isFraud` is used only for training and evaluation.

| Partition | Rule | Rows |
|---|---|---:|
| Training | step < 400; all fraud + every 24th legitimate original row | 244,120 |
| Validation | steps 400–554; original row index mod 8 = 1 | 51,549 (211 fraud) |
| Final test | step ≥ 555; original row index mod 8 = 3; never used for selection | 24,264 (268 fraud) |

Class enrichment applies to training only.

**Leakage controls**

- **Excluded from features:** IDs, `isFraud`, existing fraud flags and all balances.
- **Past-only history:** sender and recipient statistics use events strictly before the current one.
- **Past-only stream:** six stream features computed from earlier original rows only, with no labels and no current or future row.
- **Chronology:** `StreamContext` raises an error on out-of-order events, and manual steps can't go backwards.
- **Injection-proof:** CSV or manual input can't supply stream snapshots, and tests verify this.

## Features and model

23 numeric features in six groups. The explainer uses the same groups.

| Group | Features | Count |
|---|---|---:|
| Payment amount | log of amount | 1 |
| Payment type | one-hot: PAYMENT, TRANSFER, CASH_OUT, CASH_IN, DEBIT | 5 |
| Sender history | prior payment count, mean amount, amount vs mean, new-recipient flag | 4 |
| Simulation timing | hour of day, sine and cosine of hour | 3 |
| Recipient history | incoming count, total received, distinct senders, seen-sender flag | 4 |
| Recent stream | previous and current hour volume, matching transfer and cash-out counts, delay, previous transfer volume | 6 |

**Model:** scikit-learn `HistGradientBoostingClassifier` with 250 iterations, 23 leaf nodes, depth 7 and learning rate 0.06.

**Training and promotion policy**

1. Train three candidates with unweighted, moderate (1:3) and cost-aware (1:10) class weights.
2. Pick the one with the best validation F1, with a false-positive rate of 0.5% or less and recall at least as high as V2. The cost-aware (1:10) candidate won.
3. Promote it only if its final-test F1 beats V2. Otherwise keep the incumbent.

**Model evolution**

- **V1:** transaction and early history features. High precision but low recall (22.8% on the V2 holdout).
- **V2:** account context with class weights. Recall 53.0%, but 125 false alerts on its holdout.
- **V3:** forests and ensembles were tried but didn't reach final evaluation.
- **Sequence v4:** adds past-only stream context. Selected on validation F1 and promoted after one final-test evaluation.

The threshold of **0.9859** was chosen on validation data only. Scores are uncalibrated risk scores, not fraud probabilities.

## Why this score? Exact grouped Shapley

Every scored payment can be opened to see how the six feature groups moved its score relative to a training reference.

- **Game:** the value of a coalition is the mean model score over 32 evenly spaced training reference rows, with those groups set to the payment's values.
- **Exact:** all 2⁶ = 64 coalitions are enumerated, with no sampling.
- **Grouped:** one-hot type and cyclic hour stay together to avoid artificial combinations.
- **Reconciliation:** baseline plus contributions must equal the score within 1e-8, or no explanation is shown.
- **Snapshot:** the original feature vector and model fingerprint are stored with each decision. Old events or changed models get an explicit "unavailable" message.

The summary text is built from the computed signs and the recorded threshold, not generated prose. It explains the model relative to a reference. It isn't a cause or proof of fraud.

## API

| Endpoint | Purpose |
|---|---|
| `POST /api/replay` | Score the next held-out payment |
| `POST /api/score` | Score a manual test payment |
| `GET /api/explain/{id}` | Exact grouped Shapley breakdown |
| `POST /api/review/{id}` | Record *Needs investigation* or *Dismissed*, with a note |
| `POST /api/settings` | Change the alert threshold (persists per model version) |
| `GET /api/export` | Review CSV, active or archived |
| `POST /api/train` | Upload a labelled CSV and retrain |
| `GET /api/health` | Status, model version, machine and system |

## Getting started

### Run on IBM LinuxONE

Install dependencies on the target machine itself. Never copy Windows or x86 binaries to s390x.

```bash
# 1. Copy the source to LinuxONE and install dependencies there
pip install -r requirements.txt

# 2. Train the model natively on s390x with the PaySim archive
OMP_NUM_THREADS=2 python train_sequence.py archive.zip

# 3. Start the service (binds 127.0.0.1:8000, one worker)
python run.py
```

Then, from your own computer, open an SSH tunnel and visit the local URL:

```bash
ssh -L 8000:127.0.0.1:8000 <user>@<linuxone-host>
# open http://127.0.0.1:8000
```

Capture the `/api/health` output as architecture evidence. It reports Linux/s390x, so also capture the organiser dashboard. Full steps are in `DEPLOY.md`.

> **Status:** the LinuxONE deployment is documented but not yet performed or verified.

### Get the data

Download PaySim from [Kaggle (ealaxi/paysim1)](https://www.kaggle.com/datasets/ealaxi/paysim1) and check its terms. Keep the raw data out of public repositories. It is listed in `.gitignore`.

### Run the tests

```bash
python -m pytest test_app.py
```

The tests cover replay, review, export and duplicate rejection; invalid input and out-of-order steps; no label or balance leakage and past-only context; threshold budget, persistent policy and CSV training; and explanation reconciliation, snapshot refusal and ensemble support.

## Demo script (about 3 minutes)

1. Open the dashboard and press **Score next payment** a few times.
2. Drag the **Alert threshold** to show how alerts change.
3. Open an alert to show the **Why this score?** breakdown.
4. Record **Needs investigation** with a note.
5. **Export review CSV.**
6. Show `/api/health`.

## Limits

- Synthetic PaySim data, so there is no real-bank claim. Simulator regularities may inflate results.
- Manual payments use session history only, without full stream context.
- No authentication, tenancy, TLS or banking connector. Keep it behind the SSH tunnel.
- Scores aren't calibrated probabilities.
- Single worker, with history in process memory.

## Roadmap

- [ ] Authenticated streaming API for live payments
- [ ] Probability calibration and cost-based thresholds
- [ ] Drift monitoring and scheduled retraining
- [ ] Validation on real, consented bank data
- [ ] Production hardening on LinuxONE

## Project structure

```
relayguard/
├── index.html              # Self-contained browser demo of the analyst dashboard
├── app.py                  # FastAPI app: API, state, review, export, retrain, health
├── engine.py               # Validation, History, StreamContext, training, threshold policy
├── explanations.py         # Exact grouped Shapley explainer
├── train_sequence.py       # Reproducible v4 training and promotion gate
├── run.py                  # Portable launcher (127.0.0.1:8000, one worker)
├── test_app.py             # Automated tests
├── EVALUATION.json         # Evaluation protocol and results
├── DEPLOY.md               # LinuxONE deployment guide
├── requirements.txt
└── docs/
    ├── RelayGuard-IBM-Z-Datathon.pptx   # Pitch deck
    └── screenshots/
```

## Hackathon

Built for the **IBM Z Datathon 2026**, theme *Real-time AI for Critical Decisions*.

## Team

- **Armaan Kumar**
- **Rishi Chowdhury**

## Acknowledgements

- [PaySim](https://github.com/EdgarLopezPhD/PaySim) by Edgar Lopez-Rojas, via [Kaggle (ealaxi/paysim1)](https://www.kaggle.com/datasets/ealaxi/paysim1).
- IBM LinuxONE, for the hackathon environment.

## License

Released under the [MIT License](LICENSE). The PaySim dataset is not included and has its own terms.
