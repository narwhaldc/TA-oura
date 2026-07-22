# TA-oura — Oura Add-on for the Wearables platform

Technology add-on that (1) **ingests** Oura Ring data and (2) **normalizes** it into the
canonical **Wearables** data model. Part of the multi-vendor wearables platform (companion to
the `wearables` dashboard app and other per-vendor add-ons such as `TA-garmin`).

Setup + run instructions: **[INSTALL.md](INSTALL.md)**.

## What it does
- **Ingest** (`tools/oura_to_hec_with_phi.py`, repo-only — NOT shipped in the `.spl`): a **pull
  fetcher** that polls the Oura v2 API (OAuth2) and POSTs to Splunk HEC at `index=wearables`,
  `sourcetype=oura:<data_type>`, stamping HEC **indexed fields** `vendor="oura"` + `person_id`.
  Multi-target fan-out, per-target dedup, checkpoint, lock — see `oura_targets.json` in INSTALL.
- **Search-time normalization** (`props.conf`): maps raw Oura fields → canonical model fields
  (Oura seconds → minutes, renames, derived `sleep_type`, contributor sub-scores). Raw JSON kept
  in `_raw`; all normalized fields additive.
- **Dataset tagging** (`eventtypes.conf` / `tags.conf`): vendor-neutral tags (`wearable`,
  `wearable_sleep`, `wearable_heartrate`, `wearable_activity`, `wearable_workout`, `wearable_daily`,
  `wearable_device`) that the Wearables data model constrains on.
- **Enrichment**: `LOOKUP-` props resolve `person_id`→name/goals and `device_id`→name via the
  shared KV registries in the **wearables** app (cross-vendor, exported global, admin-write-locked).

## Identity & access model
- **Identity is stamped at INGEST** (by the fetcher), not in props: `vendor` + canonical
  **`person_id`** as HEC indexed fields.
- **Access isolation** = `authorize.conf` `srchFilter` on the indexed `person_id` (deployment RBAC
  config), NOT the lookups. Lookups are enrichment only — editing one can't grant access.
- Registries are **write-locked to `admin`/`sc_admin`** (KV Store, in the wearables app).

## Prerequisites
- **`index=wearables`** exists (Settings → Indexes / ACS — not shipped here, so this add-on stays
  Cloud-AppInspect-clean).
- The `wearables` app installed (data model + KV registries).
- Python + an Oura personal-access/OAuth token to run the fetcher (see INSTALL.md).

## Sourcetypes → tags
`oura:sleep_detail`, `oura:sleep` (→ sleep) · `oura:heart_rate` (→ heartrate) · `oura:activity`
(→ activity) · `oura:workouts` (→ workout) · `oura:readiness`, `oura:spo2`, `oura:cardio_age`,
`oura:resilience`, `oura:sleep_time`, `oura:personal_info` (→ daily) · `oura:battery`,
`oura:ring_config` (→ device).

## Version
0.2.7 — normalization for all Oura datasets + the Oura fetcher moved in from the legacy
`oura-health-splunk` repo (2026-07). Apache-2.0. Source: https://github.com/narwhaldc/TA-oura
