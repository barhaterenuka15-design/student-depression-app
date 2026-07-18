A small Flask web app that serves a `RandomForestClassifier` trained on the
Kaggle ["Student Depression Dataset"](https://www.kaggle.com/datasets/hopesb/student-depression-dataset).
Given info like academic pressure, sleep duration, financial stress, etc., it
predicts whether a student is at risk of depression.

> ⚠️ **This is an educational/demo project, not a diagnostic tool.** It should
> never be used to make real decisions about someone's mental health.

## Project structure

```
.
├── app.py                  # Flask app (web form + JSON API)
├── model.joblib            # Trained RandomForestClassifier (compressed, ~12MB)
├── model_columns.pkl       # Exact one-hot-encoded column order used at training time
├── dropdown_options.json   # Category values used to populate the form's dropdowns
├── templates/
│   └── index.html          # HTML form + result page
├── requirements.txt        # Pinned Python dependencies
├── Procfile                # Tells Render how to start the app (gunicorn)
└── .gitignore
```

## How it works

- `model.joblib` and `model_columns.pkl` are loaded once when the app starts.
  The model is saved with `joblib.dump(model, "model.joblib", compress=3)`
  instead of raw `pickle` — same model, identical predictions, but compressed
  from ~76MB down to ~12MB. This matters because GitHub warns on files over
  50MB and blocks pushes over 100MB, and smaller files mean faster Render
  builds/deploys.
- `model_columns.pkl` matters because `pd.get_dummies()` only creates columns
  for categories present in whatever row you feed it. To match what the model
  expects, every new input row is one-hot encoded and then **reindexed**
  against the exact column list used during training (any missing columns are
  filled with 0). Skipping this step is a very common source of silent bugs
  ("model loads fine but gives garbage predictions").
- Two endpoints serve predictions:
  - `GET /` — HTML form for humans
  - `POST /predict` — form submission handler, renders the result back into the page
  - `POST /api/predict` — JSON API for programmatic use, e.g.:
    ```bash
    curl -X POST https://your-app.onrender.com/api/predict \
      -H "Content-Type: application/json" \
      -d '{
        "id": 1, "Gender": "Male", "Age": 22, "City": "Delhi",
        "Profession": "Student", "Academic Pressure": 4, "Work Pressure": 0,
        "CGPA": 7.5, "Study Satisfaction": 3, "Job Satisfaction": 0,
        "Sleep Duration": "5-6 hours", "Dietary Habits": "Unhealthy",
        "Degree": "B.Tech", "Have you ever had suicidal thoughts ?": "Yes",
        "Work/Study Hours": 8, "Financial Stress": 4,
        "Family History of Mental Illness": "Yes"
      }'
    ```
  - `GET /health` — simple health check (useful for uptime monitors)

## Run locally

```bash
python3 -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
python3 app.py
```

Then open http://127.0.0.1:5000 in your browser.

---

## Deploying to Render — step by step

### 1. Put the project on GitHub

Render deploys from a Git repository, so this folder needs to be a GitHub repo.

```bash
cd this-project-folder
git init
git add .
git commit -m "Initial commit: student depression predictor"
```

Create a new **empty** repository on GitHub (no README/license, so it stays
empty), then:

```bash
git remote add origin https://github.com/<your-username>/<your-repo>.git
git branch -M main
git push -u origin main
```

> `model.joblib` is ~12 MB (compressed from the original ~76 MB pickle), so a
> plain `git push` will work fine — just don't add it to `.gitignore`.

### 2. Create the Render Web Service

1. Go to https://dashboard.render.com and log in / sign up.
2. Click **New +** → **Web Service**.
3. Connect your GitHub account if you haven't already, then select the repo
   you just pushed.
4. Fill in the settings:
   - **Name**: anything you like, e.g. `student-depression-predictor`
   - **Region**: closest to you
   - **Branch**: `main`
   - **Root Directory**: leave blank *only if* `app.py` sits at the repo root
     (which it does if you followed step 1 exactly). If your repo has this
     project inside a subfolder, put that subfolder name here — this is the
     #1 cause of the `requirements.txt` "No such file or directory" error.
   - **Runtime**: Python 3
   - **Build Command**: `pip install -r requirements.txt`
   - **Start Command**: `gunicorn app:app` (this is already in the `Procfile`,
     so Render should pick it up automatically — but you can also type it
     explicitly here)
   - **Instance Type**: Free is fine for testing
5. Click **Create Web Service**.

### 3. Watch the build & deploy logs

Render will:
- Clone your repo
- Run the build command (`pip install -r requirements.txt`)
- Start the app with the start command (`gunicorn app:app`)

If it succeeds, you'll see `Your service is live 🎉` and a URL like
`https://student-depression-predictor.onrender.com`.

### 4. Test it

- Open the Render URL in your browser → you should see the form.
- Fill it in and click **Predict** → you should see a result box.
- Or test the API directly:
  ```bash
  curl https://<your-app>.onrender.com/health
  ```

### 5. Common issues & fixes

| Symptom | Cause | Fix |
|---|---|---|
| `ERROR: Could not open requirements file` | Render is building from the wrong folder | Set **Root Directory** to the folder that actually contains `requirements.txt` |
| Build succeeds but app crashes on start | Wrong start command | Make sure it's exactly `gunicorn app:app` (matches the Flask variable name `app` inside `app.py`) |
| `ModuleNotFoundError` at runtime | A package used in `app.py` isn't in `requirements.txt` | Add it and redeploy |
| Predictions look wrong / all same class | `model_columns.pkl` missing or not loaded | Confirm both `model.joblib` and `model_columns.pkl` were committed and pushed to GitHub |
| `git push` rejected for large file / repo feels slow | Old ~76MB `model.pkl` still tracked in git history | Make sure you're using the newer `model.joblib` (~12MB) and, if the big file was ever committed before, remove it from history (`git rm --cached model.pkl` then commit, or start a fresh repo) |
| Free tier app "spins down" and is slow to respond after inactivity | Normal Render free-tier behavior | Upgrade to a paid instance if you need always-on, low-latency responses |

### 6. Redeploying after changes

Render auto-deploys on every push to the connected branch by default:

```bash
git add .
git commit -m "update model"
git push
```

That's it — Render picks up the push and rebuilds automatically.

