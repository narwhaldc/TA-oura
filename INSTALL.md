# TA-oura → Splunk — Installation Guide

Setup for the Oura data pipeline into the Wearables platform.
**Add-on:** TA-oura · **Ingest:** `tools/oura_to_hec_with_phi.py` (Oura v2 API → HEC, OAuth2 pull)

The Oura fetcher moved here from the legacy `oura-health-splunk` repo (2026-07) so each vendor's
normalization + ingest live together. It's **repo-only tooling — never shipped in the `.spl`**
(it holds credentials).

---

## Architecture
```
Oura Ring → Oura Cloud API (v2)
                ↓  OAuth2 (auto-refresh; PAT fallback)
        tools/oura_to_hec_with_phi.py        (cron; pull)
        per-type field filtering · per-target dedup · checkpoint · fcntl lock
                ↓  multi-target fan-out (oura_targets.json)
        Splunk HEC → index=wearables, sourcetype=oura:<data_type>,
                     indexed fields vendor=oura + person_id
                ↓
        TA-oura (search-time normalization → canonical Wearables model)
        wearables app (data model + dashboards)
```

## Prerequisites
- **`index=wearables`** exists (Settings → Indexes / ACS).
- **`wearables` app** installed (data model + KV registries), and **TA-oura** installed.
- A Splunk **HEC token** with access to `index=wearables`.
- Python 3.9+ on the ingest host.
- An **Oura developer app** (OAuth2 client id/secret from developer.ouraring.com) — or a legacy
  Personal Access Token as fallback.

## 1. Install the Splunk apps
Install + restart Splunk: **`TA-oura`** (this add-on) and **`wearables`** (model + dashboards).
Optionally the viz add-ons `hypnogram_viz` + `charge_ring_viz` for the custom visualizations.

## 2. Install the fetcher + deps
Copy `tools/oura_to_hec_with_phi.py` to your ingest host:
```bash
python3 -m pip install requests
```

## 3. Oura OAuth2 authorization (one-time)
```bash
export OURA_CLIENT_ID='...'        # from developer.ouraring.com
export OURA_CLIENT_SECRET='...'
python3 oura_to_hec_with_phi.py --auth     # opens a browser; approve
```
Tokens persist to `oura_tokens.json` (`OURA_TOKEN_FILE`) and auto-refresh thereafter.
(Fallback: set `OURA_PAT` to a Personal Access Token instead of OAuth2.)
> Keep client secret / tokens in env or the token file — **never commit them** (gitignored).

## 4. HEC targets (`oura_targets.json`)
Multi-target fan-out; same format as `garmin_targets.json`:
```json
{
  "targets": {
    "personal": {
      "hec_url":   "https://splunk:8088/services/collector/event",
      "hec_token": "YOUR-HEC-TOKEN",
      "index":     "wearables",
      "person_id": "P001",
      "verify_ssl": false
    }
  }
}
```
`vendor=oura` + `person_id` are stamped as indexed HEC fields (RBAC key). Sourcetype is derived
per data type (`oura:<data_type>`) — do not set a per-target sourcetype. This file holds a token →
**gitignored, never commit.** (Single-target alternative: `SPLUNK_HEC_URL` + `SPLUNK_HEC_TOKEN`
env vars.)

## 5. Populate the registries (admin, KV Store in the wearables app)
```
| makeresults | eval person_id="P001", person_name="Tony", step_goal=10000
| table person_id person_name step_goal | outputlookup wearable_person_profile
```
Device name is auto-derived after first ingest (Oura `ring_config` supplies model/firmware).

## 6. First run & backfill
```bash
python3 oura_to_hec_with_phi.py --dry-run          # print payloads, send nothing
python3 oura_to_hec_with_phi.py --backfill 2024-01-01
python3 oura_to_hec_with_phi.py                     # incremental (checkpoint - overlap .. now)
python3 oura_to_hec_with_phi.py --status            # per-target coverage
```

## 7. Cron
```cron
*/10 * * * * cd /opt/oura && /usr/bin/python3 oura_to_hec_with_phi.py >> oura_to_hec.log 2>&1
```
The `fcntl` lock (`oura_sync.lock`) makes overlapping runs safe; overlap dupes are cleaned by the
wearables app's "Wearables Dedup" saved searches.

## 8. Verify
```
index=wearables vendor=oura | stats count by sourcetype
index=wearables tag=wearable_sleep vendor=oura | table _time total_sleep_min sleep_score
```
Then open the wearables dashboards (Today / Sleep / Heart / Activity / Wellness / Device).

## State files (all gitignored — never commit)
`oura_tokens.json` (OAuth tokens) · `oura_targets.json` (HEC tokens) · `oura_checkpoint.json` ·
`oura_dedup_store.json` · `oura_sync.lock`. See the script header for the full env-var list
(`--help`, `--show-filters`).

## Troubleshooting
- **Auth errors** → re-run `--auth`; confirm `OURA_CLIENT_ID/SECRET` (or `OURA_PAT`).
- **Nothing in Splunk** → check `oura_targets.json` hec_url/token/index; `--dry-run` to inspect payloads.
- **Re-send a date** → `--reset-dedup` (all) or `--reset-dedup --target NAME` (one), then `--backfill`.
- **Missing history** → Oura retains limited history for some types; backfill only recovers what the API still serves.
