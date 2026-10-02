#!/usr/bin/env python3
"""
BHA jobs board — publish script
===============================
Reads the Jobs table from Airtable, keeps the roles that should be public
(Open, closing today or later), lightly cleans them, and writes jobs.json.

Runs unattended (hourly) from a GitHub Action, and can be run by hand.
Standard library only — no pip install needed.

Environment:
  AIRTABLE_TOKEN   (required)  A read-only Airtable personal access token
                               scoped to the jobs base, data.records:read.

Config below is specific to the BHA jobs base; the field IDs are stable.
"""

import json
import os
import re
import sys
import urllib.request
import urllib.error
import urllib.parse
from datetime import datetime, timezone

# ---------------------------------------------------------------------------
# Config
# ---------------------------------------------------------------------------
BASE_ID  = "apphtvtnT553IHuQv"
TABLE_ID = "tblCeBG5PVYAZF7B9"
OUTPUT   = "jobs.json"

# Field IDs (from the jobs table schema). Using IDs, not names, so the script
# keeps working even if a field is renamed in the Airtable UI.
F = {
    "title":       "fldShdki7ZSww9CXx",
    "hotel":       "fldfIzIr24jzt4jJ5",
    "department":  "fldEOHgj8ibnV9Ugk",
    "type":        "fldKEJtghiJUcPBrL",
    "salary":      "fldl2emOyDHRi9VPW",
    "description": "fldTYLSVnDqc1mVnj",
    "applyUrl":    "fldajKjorP1iXc8g7",
    "email":       "fldSETyhLNZYxezNa",
    "closing":     "fld6noJmgV3ahlg3i",
    "status":      "fldYwYeRm0LA6w09p",
    "posted":      "fldkFaM98nRw2lX3i",
}

# Optional: a "Website description" override field. Leave as "" until the field
# exists; then paste its field ID here. When a role has a value in it, it is
# used verbatim (still whitespace-normalised) instead of the raw Description.
WEBSITE_DESC_FIELD_ID = ""

# Light, safe typo fixes seen in submissions. Extend as needed — these are
# plain substring replacements, so keep them specific.
TYPO_FIXES = {
    "Memeber": "Member",
    "Ideallly": "Ideally",
}

# ---------------------------------------------------------------------------
# Airtable fetch
# ---------------------------------------------------------------------------
def fetch_records(token):
    """Return all records from the table, fields keyed by field ID."""
    base = "https://api.airtable.com/v0/%s/%s" % (BASE_ID, TABLE_ID)
    records, offset = [], None
    while True:
        url = base + "?pageSize=100&returnFieldsByFieldId=true"
        if offset:
            url += "&offset=" + urllib.parse.quote(offset)
        req = urllib.request.Request(url, headers={
            "Authorization": "Bearer " + token,
        })
        with urllib.request.urlopen(req, timeout=30) as resp:
            data = json.loads(resp.read().decode("utf-8"))
        records.extend(data.get("records", []))
        offset = data.get("offset")
        if not offset:
            break
    return records


# ---------------------------------------------------------------------------
# Field helpers (tolerant of REST string shape and object shapes)
# ---------------------------------------------------------------------------
def val(fields, key):
    v = fields.get(F.get(key, ""), "")
    if isinstance(v, dict):          # single-select / linked record object form
        return v.get("name", "")
    if isinstance(v, list):          # multi value — take the first name/string
        if not v:
            return ""
        first = v[0]
        return first.get("name", "") if isinstance(first, dict) else str(first)
    return v if v is not None else ""


# ---------------------------------------------------------------------------
# Cleaning (deliberately light — safe to run unattended)
# ---------------------------------------------------------------------------
def tidy_salary(s):
    s = (s or "").strip()
    if not s:
        return ""
    if s.lower() == "competitive":
        return "Competitive"
    s = re.sub(r"^Pay Rate\s+", "", s)
    if re.match(r"^[0-9]", s):          # bare number -> add £
        s = "£" + s
    s = re.sub(r"\s-\s", " – ", s)  # spaced hyphen -> en dash for ranges
    return s


def clean_description(s):
    s = s or ""
    for a, b in TYPO_FIXES.items():
        s = s.replace(a, b)
    # Repair a few known leading-letter truncations from the source ads.
    if s.startswith("clipse "):
        s = "E" + s
    if s.startswith("he Opportunity"):
        s = "T" + s
    if s.startswith("s a Housekeeping"):
        s = "A" + s
    # Whitespace: normalise line endings, trim each line, collapse blank runs.
    lines = [ln.rstrip() for ln in s.replace("\r\n", "\n").split("\n")]
    out, prev_blank = [], False
    for ln in lines:
        blank = (ln.strip() == "")
        if blank and prev_blank:
            continue
        out.append("" if blank else ln)
        prev_blank = blank
    return "\n".join(out).strip()


# ---------------------------------------------------------------------------
# Main
# ---------------------------------------------------------------------------
def main():
    token = os.environ.get("AIRTABLE_TOKEN", "").strip()
    if not token:
        sys.stderr.write("ERROR: AIRTABLE_TOKEN is not set.\n")
        sys.exit(1)

    try:
        records = fetch_records(token)
    except urllib.error.HTTPError as e:
        sys.stderr.write("ERROR: Airtable returned HTTP %s: %s\n" % (e.code, e.read().decode("utf-8", "ignore")[:300]))
        sys.exit(1)
    except Exception as e:  # noqa
        sys.stderr.write("ERROR: could not reach Airtable: %s\n" % e)
        sys.exit(1)

    today = datetime.now(timezone.utc).strftime("%Y-%m-%d")
    jobs, hidden = [], 0

    for rec in records:
        f = rec.get("fields", {})
        status  = val(f, "status")
        closing = str(val(f, "closing") or "")[:10]
        # Public rule: Open and not past its closing date.
        if status != "Open" or not closing or closing < today:
            hidden += 1
            continue

        description = ""
        if WEBSITE_DESC_FIELD_ID:
            override = f.get(WEBSITE_DESC_FIELD_ID, "")
            if isinstance(override, str) and override.strip():
                description = override
        if not description:
            description = val(f, "description")

        jobs.append({
            "title":       str(val(f, "title")).strip(),
            "hotel":       str(val(f, "hotel")).strip(),
            "department":  val(f, "department"),
            "type":        val(f, "type"),
            "salary":      tidy_salary(val(f, "salary")),
            "description": clean_description(description),
            "applyUrl":    str(val(f, "applyUrl")).strip(),
            # Contact email is deliberately NOT published (data minimisation /
            # spam-harvesting): applicants use the apply link. Keep it out.
            "closing":     closing,
            "posted":      str(val(f, "posted") or "")[:10],
            "status":      status,
        })

    # Newest first (the page re-sorts, but a stable order keeps diffs small).
    jobs.sort(key=lambda j: j["posted"], reverse=True)

    payload = {
        "generatedAt": datetime.now(timezone.utc).isoformat(timespec="seconds"),
        "count": len(jobs),
        "jobs": jobs,
    }
    with open(OUTPUT, "w", encoding="utf-8") as fh:
        json.dump(payload, fh, ensure_ascii=False, indent=2)
        fh.write("\n")

    print("Published %d live role(s); %d hidden (filled/closed/expired)." % (len(jobs), hidden))


if __name__ == "__main__":
    main()
