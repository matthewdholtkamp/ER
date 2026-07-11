# ER Commander's Report Dashboard

Single-page, de-identified ER analytics dashboard. GitHub is the code source of truth; Firestore is the operational data source. `data.json` is a timestamped, hash-verified offline fallback and never overwrites reachable Firestore data.

## Verified data model

The application supports a staged v2 model with one document per calendar date:

- `dccs_data/er_day_v2_{YYYY-MM-DD}` — daily metrics plus de-identified, hourly grouped patient records. The prefix keeps v2 inside the collection already authorized by the deployed Firestore rules.
- `dccs_data/er_v2_meta` — migration status, counts, totals, source timestamp, and aggregate SHA-256 hash.
- `dccs_data/er_data` — untouched legacy rollback source.

Dates are stored as `YYYY-MM-DD` calendar keys and converted with numeric year/month/day components. Bare ISO date strings are never passed to `new Date(string)`.

## Safe migration

Migration functions only run when the app is served from `localhost` or `127.0.0.1`:

```js
const backup = await ERMigration.createLegacyBackup();
const verified = await ERMigration.migrateLegacyToV2(backup.payloadHash);
await ERMigration.activateVerifiedV2(verified.aggregateHash);
```

Rollback changes only the metadata reader mode and retains both datasets:

```js
await ERMigration.rollbackToLegacy();
```

The Firestore security rules must authorize the operator to read and write the prefixed v2 documents and `dccs_data/er_v2_meta`. If permissions are insufficient, migration stops before activation and the application continues reading the legacy document. Do not delete the legacy document or the downloaded migration backup.

## Import behavior

Workbook batches are fully parsed before writes. Invalid or duplicate dates abort the batch. Existing dates show metric/patient differences and require confirmation before replacement. When v2 is active, only affected day documents are written and verified.
