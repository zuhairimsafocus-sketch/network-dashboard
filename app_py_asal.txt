from flask import Flask, jsonify, render_template
import pandas as pd

app = Flask(__name__)

df = pd.read_csv("complaints_coverage_by_state_malaysia.csv")

SIGNAL_COL = "signal_strength"  # <-- tukar jika nama kolum lain

# =========================================================
# DISTRICT NAME MAPPING (CSV -> GeoJSON shapeName)
# =========================================================
DISTRICT_NAME_MAP = {
    # City/town in CSV -> admin district in GeoJSON
    "Alor Setar": "Kota Setar",
    "Hulu Langat": "Ulu Langat",
    "Kulai": "Kulaijaya",
    "Ipoh": "Kinta",
    "Taiping": "Larut Matang dan Selama",
    "Sungai Petani": "Kuala Muda",
    "George Town": "Timur Laut",
    # CSV is aggregated; GeoJSON splits into Utara/Tengah/Selatan.
    # Temporary mapping to one district to avoid 0 on map.
    "Seberang Perai": "Seberang Perai Tengah",
}

def normalize_key(s: str) -> str:
    """Light normalization for mapping keys (case/space safe)."""
    return str(s or "").strip()

def apply_district_mapping(df_in: pd.DataFrame) -> pd.DataFrame:
    """Apply district mapping on a copy of df to keep transformations centralized."""
    df_out = df_in.copy()

    if "district" not in df_out.columns:
        return df_out

    # strip spaces
    df_out["district"] = df_out["district"].astype(str).str.strip()

    # map via dictionary
    df_out["district"] = df_out["district"].apply(lambda x: DISTRICT_NAME_MAP.get(normalize_key(x), x))

    return df_out

# Apply mapping once at load time
df = apply_district_mapping(df)


@app.route("/")
def index():
    return render_template("index.html")


@app.route("/api/summary")
def summary():
    status_counts = df["status"].value_counts().to_dict()
    return jsonify({
        "total_complaints": len(df),
        "total_states": df["state"].nunique(),
        "total_districts": df["district"].nunique(),
        "total_submitted": int(status_counts.get("Submitted", 0)),
        "total_in_progress": int(status_counts.get("In Progress", 0)),
        "total_resolved": int(status_counts.get("Resolved", 0))
    })


@app.route("/api/state")
def state():
    return jsonify(df["state"].value_counts().to_dict())


@app.route("/api/district")
def district():
    # district names here are now aligned to GeoJSON shapeName
    return jsonify(df["district"].value_counts().to_dict())


@app.route("/api/issue_type")
def issue_type():
    return jsonify(df["issue_type"].value_counts().to_dict())


@app.route("/api/state_issue_matrix")
def state_issue_matrix():
    ct = pd.crosstab(df["state"], df["issue_type"])
    ct["__total__"] = ct.sum(axis=1)
    ct = ct.sort_values("__total__", ascending=False).drop(columns=["__total__"])

    issue_types = list(ct.columns)
    states = list(ct.index)
    matrix = {issue: ct[issue].astype(int).tolist() for issue in issue_types}

    return jsonify({"states": states, "issue_types": issue_types, "matrix": matrix})


@app.route("/api/state_status_matrix")
def state_status_matrix():
    ct = pd.crosstab(df["state"], df["status"])

    status_order = ["Submitted", "In Progress", "Resolved"]
    for s in status_order:
        if s not in ct.columns:
            ct[s] = 0
    ct = ct[status_order]

    ct["__total__"] = ct.sum(axis=1)
    ct = ct.sort_values("__total__", ascending=False).drop(columns=["__total__"])

    statuses = list(ct.columns)
    states = list(ct.index)
    matrix = {st: ct[st].astype(int).tolist() for st in statuses}

    return jsonify({"states": states, "statuses": statuses, "matrix": matrix})


# NEW: STATE x SIGNAL STRENGTH MATRIX
@app.route("/api/state_signal_strength_matrix")
def state_signal_strength_matrix():
    if SIGNAL_COL not in df.columns:
        return jsonify({
            "states": [],
            "signal_strengths": [],
            "matrix": {},
            "error": f"Column '{SIGNAL_COL}' not found in dataset."
        })

    ct = pd.crosstab(df["state"], df[SIGNAL_COL])

    preferred_order = ["Strong", "Moderate", "Weak", "No Signal"]
    existing = [x for x in preferred_order if x in ct.columns]
    remaining = [x for x in ct.columns if x not in existing]
    signal_order = existing + sorted(remaining)

    ct = ct[signal_order]

    ct["__total__"] = ct.sum(axis=1)
    ct = ct.sort_values("__total__", ascending=False).drop(columns=["__total__"])

    signal_strengths = list(ct.columns)
    states = list(ct.index)
    matrix = {ss: ct[ss].astype(int).tolist() for ss in signal_strengths}

    return jsonify({"states": states, "signal_strengths": signal_strengths, "matrix": matrix})


# REPORTED MONTHS (Total)
@app.route("/api/reported_months")
def reported_months():
    month_order = [
        "January","February","March","April","May","June",
        "July","August","September","October","November","December"
    ]
    s = df["reported_months"].astype(str).str.strip().str.title()
    counts = s.value_counts().to_dict()
    ordered = {m: int(counts.get(m, 0)) for m in month_order}
    return jsonify(ordered)


if __name__ == "__main__":
    app.run(debug=True)
