# TA-oura — Oura Add-on for the Wearables platform

Technology add-on that normalizes raw Oura Ring data into the canonical
**Wearables** data model. Part of the multi-vendor wearables platform (companion
to the `wearables` dashboard app and other per-vendor add-ons such as `TA-garmin`).

## What it does
- **Search-time field normalization** (`props.conf`): maps raw Oura fields into
  canonical model fields — unit conversion (Oura seconds → minutes), field
  renames, derived fields (`sleep_type`), and vendor/dataset tagging. Raw JSON is
  preserved in `_raw`; all normalized fields are additive.
- **Dataset tagging** (`eventtypes.conf` / `tags.conf`): tags Oura events with
  vendor-neutral tags (`wearable`, `wearable_sleep`, …) that the Wearables data
  model constrains on.
- **Lookups** (`transforms.conf` + `lookups/`): the identity/account map and the
  person profile (names + goals).

## Identity & access model
- **Identity is stamped at INGEST**, not here: the Oura fetcher sends `vendor` and
  a canonical **`person_id`** as **HEC indexed fields**. This add-on only reads
  them (and enriches `person_id` → name/goals via the profile lookup).
- **Access isolation is enforced by `authorize.conf` `srchFilter` on the indexed
  `person_id`** (in the deployment's RBAC config), NOT by these lookups. The
  lookups are enrichment only — editing one cannot grant access to another
  person's data.
- All lookups are **write-locked to `admin` / `sc_admin`** (self-managed + Cloud)
  via `metadata/default.meta`; populate them with the Splunk App for Lookup File
  Editing (admin).

## Prerequisites
- An **`index=wearables`** must exist (create via Settings → Indexes on
  self-managed, or the Cloud console / ACS on Splunk Cloud — not shipped in this
  add-on so it stays Cloud-AppInspect-clean).
- The Oura fetcher (`oura_to_hec_with_phi.py`) configured to POST to that index
  with `sourcetype=oura:<data_type>` and `fields:{vendor, person_id}`.

## Sourcetypes
`oura:sleep_detail` (sleep session), `oura:sleep` (daily sleep summary). More
(`oura:heart_rate`, `oura:activity`, `oura:readiness`, …) added incrementally.

## Version
0.1.0 — initial scaffold (Sleep normalization). Apache-2.0.
