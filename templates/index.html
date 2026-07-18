import os
import json

import joblib
import pandas as pd
from flask import Flask, request, jsonify, render_template

app = Flask(__name__)

BASE_DIR = os.path.dirname(os.path.abspath(__file__))

# ---------------------------------------------------------------------
# Load the model bundle once, at startup. The bundle is a single
# compressed joblib file containing both the trained model and the
# exact training-time column list, so there is only one artifact to
# manage/commit/deploy (this avoids a real deploy failure where a
# second, separate file was missed when pushing to GitHub).
#
# The column list is required because pd.get_dummies() only creates
# columns for categories present in whatever data you feed it. To match
# the model's expected input shape, we must reindex every new row
# against the same columns used during training (missing ones -> 0).
# ---------------------------------------------------------------------
BUNDLE_PATH = os.path.join(BASE_DIR, "model_bundle.joblib")
if not os.path.exists(BUNDLE_PATH):
    raise FileNotFoundError(
        f"model_bundle.joblib not found at {BUNDLE_PATH}. "
        "Make sure it was committed to git and pushed to your repo "
        "(check 'git status' / 'git ls-files' locally, and confirm "
        "it shows up on GitHub in the repo's file listing)."
    )

_bundle = joblib.load(BUNDLE_PATH)
model = _bundle["model"]
MODEL_COLUMNS = _bundle["columns"]

with open(os.path.join(BASE_DIR, "dropdown_options.json"), "r") as f:
    DROPDOWN_OPTIONS = json.load(f)

RAW_FEATURE_COLUMNS = [
    "id", "Gender", "Age", "City", "Profession", "Academic Pressure",
    "Work Pressure", "CGPA", "Study Satisfaction", "Job Satisfaction",
    "Sleep Duration", "Dietary Habits", "Degree",
    "Have you ever had suicidal thoughts ?", "Work/Study Hours",
    "Financial Stress", "Family History of Mental Illness",
]


def build_feature_row(payload: dict) -> pd.DataFrame:
    """Turn a dict of raw form/JSON inputs into a model-ready one-row DataFrame."""
    row = {
        "id": float(payload.get("id", 0)),
        "Gender": payload["Gender"],
        "Age": float(payload["Age"]),
        "City": payload["City"],
        "Profession": payload["Profession"],
        "Academic Pressure": float(payload["Academic Pressure"]),
        "Work Pressure": float(payload["Work Pressure"]),
        "CGPA": float(payload["CGPA"]),
        "Study Satisfaction": float(payload["Study Satisfaction"]),
        "Job Satisfaction": float(payload["Job Satisfaction"]),
        "Sleep Duration": payload["Sleep Duration"],
        "Dietary Habits": payload["Dietary Habits"],
        "Degree": payload["Degree"],
        "Have you ever had suicidal thoughts ?": payload["Have you ever had suicidal thoughts ?"],
        "Work/Study Hours": float(payload["Work/Study Hours"]),
        "Financial Stress": float(payload["Financial Stress"]),
        "Family History of Mental Illness": payload["Family History of Mental Illness"],
    }
    df = pd.DataFrame([row])
    df = pd.get_dummies(df, drop_first=True)
    # Align to the exact training-time columns; anything unseen becomes 0.
    df = df.reindex(columns=MODEL_COLUMNS, fill_value=0)
    return df


@app.route("/", methods=["GET"])
def home():
    return render_template("index.html", options=DROPDOWN_OPTIONS, result=None)


@app.route("/predict", methods=["POST"])
def predict_form():
    """Handle the HTML form submission and render the result back on the page."""
    try:
        payload = request.form.to_dict()
        x = build_feature_row(payload)
        pred = int(model.predict(x)[0])
        proba = model.predict_proba(x)[0][pred]
        result = {
            "label": "At risk of depression" if pred == 1 else "Not at risk of depression",
            "confidence": f"{proba * 100:.1f}%",
        }
    except Exception as e:
        result = {"error": str(e)}
    return render_template("index.html", options=DROPDOWN_OPTIONS, result=result)


@app.route("/api/predict", methods=["POST"])
def predict_api():
    """JSON API endpoint, e.g. for programmatic / frontend fetch() calls."""
    try:
        payload = request.get_json(force=True)
        x = build_feature_row(payload)
        pred = int(model.predict(x)[0])
        proba = float(model.predict_proba(x)[0][pred])
        return jsonify({
            "prediction": pred,
            "label": "At risk of depression" if pred == 1 else "Not at risk of depression",
            "confidence": round(proba, 4),
        })
    except Exception as e:
        return jsonify({"error": str(e)}), 400


@app.route("/health", methods=["GET"])
def health():
    return jsonify({"status": "ok"})


if __name__ == "__main__":
    # Local dev only; Render uses gunicorn (see Procfile / start command).
    port = int(os.environ.get("PORT", 5000))
    app.run(host="0.0.0.0", port=port, debug=True)
