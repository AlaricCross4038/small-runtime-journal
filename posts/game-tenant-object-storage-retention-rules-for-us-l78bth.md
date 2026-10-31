# Game Tenant Object Storage: Retention Rules for User-Uploaded Avatars

Short answer: simple S3-compatible object storage is a sensible boundary for storing user-uploaded avatars, but a game tenant's backup policy still decides the expensive part: retained bytes multiplied by retention duration. Set a small, explicit restore window; keep immutable snapshot metadata beside each object; and make deletion a policy decision that can be audited and reversed before it becomes permanent.

For example, a service supporting 2,000 game tenants might produce one 4 GB snapshot per tenant each day. A 30-day window represents 240 TB before compression, replicas, and abandoned uploads are counted. The useful change is not shaving milliseconds from an upload. It is deciding which 30 days remain restorable, then deleting everything outside that boundary predictably.

No magic.

## How should object storage handle storing user-uploaded avatars safely?

A snapshot is useful only if its data and its meaning survive together. The object key should identify a tenant and snapshot, while a small manifest records the snapshot timestamp, schema version, checksum, and the set of objects expected in the restore. A restore request should select a known snapshot ID, never an arbitrary path supplied by a client.

That separation matters during deletion. A lifecycle rule can remove objects after a deadline, but the control plane still needs to know whether the snapshot was complete, selected for legal hold, or currently being restored. Otherwise, a successful object deletion can leave a catalog entry that looks restorable until the first missing read. The trade-off is clear: object storage simplifies durable bytes, while the application must own tenant identity, restore selection, and policy state.

I start with the restore contract, then derive retention from it. A tenant may need the latest seven daily snapshots and four weekly snapshots, for instance. That is a more defensible rule than “keep recent files,” because it names the recovery choices the game actually offers.

## Where does the retention bill come from?

Retention has three distinct byte pools: live snapshots, noncurrent versions or delete markers, and incomplete multipart uploads. Counting only the first pool produces a clean spreadsheet and a dirty incident review. Versioning can preserve an older object after an overwrite, while a failed upload can reserve parts without ever becoming a usable snapshot.

The estimate should therefore be a small model, not a single storage-rate lookup:

```text
retained_bytes =
  complete_snapshot_bytes
  + noncurrent_version_bytes
  + incomplete_upload_bytes

monthly_storage = retained_bytes * replica_count * storage_months
```

The symbols are intentionally boring. They force the team to name replication and retention assumptions rather than hiding them in a vendor quote. Sampling helps elsewhere: retain every restore audit event, but sample high-volume transfer metrics after aggregation. A label such as `tenant_id` on every raw metric can create a cardinality bill that is independent of the backup bytes.

## How should deletion avoid creating a false restore promise?

Use two clocks. The first is the eligibility time derived from policy; the second is the deletion completion time recorded by the control plane. A snapshot becomes eligible after its retention deadline, then a worker checks that no restore is active, no hold applies, and the manifest still matches the object set. Only then should it issue the object deletions and mark the snapshot unavailable.

The worker must be idempotent. A timeout after deleting three of ten objects is not evidence that all ten are gone, so retry logic should tolerate already-absent objects and re-read the manifest before advancing state. Deletion is a workflow with states such as `complete`, `eligible`, `deleting`, and `deleted`; it is not a boolean attached to a row.

Keep a compact audit record after the payload disappears: tenant identifier, snapshot identifier, policy revision, actor or scheduler, eligibility time, deletion result, and correlation ID. Do not keep the deleted game data merely to prove that it was deleted. Keep the evidence, not the payload.

That is the trade.

A focused control-plane request can look like this, with the storage endpoint hidden behind a generic interface:

```bash
curl --request POST \
  --url https://storage.example.test/v1/storage/object/delete/game-backups/tenant-204%2Fsnap-2026-10-21%2Fmanifest.json \
  --header 'Authorization: Bearer REDACTED' \
  --header 'Content-Type: application/json' \
  --data '{"target":"staging","dry_run":true}'
```

The dry run is important, but a delete endpoint is not a restore API. In a real control plane, the manifest is resolved before this operation and the object deletion is issued only after the restore lock check. That boundary keeps the generic storage interface honest: it deletes a key; it does not understand game state.

## What do we deliberately stop keeping?

I would stop retaining intermediate upload parts, failed snapshot manifests, and raw per-object transfer logs beyond the period needed for operations. I would retain completed manifests, restore audit events, checksums, and a bounded set of recent snapshots. That choice makes an incident harder in one specific way: after the window closes, the team cannot reconstruct every discarded intermediate.

That is acceptable only when the restore promise says so plainly. If the game promises a 30-day restore window, an operator should be able to select any valid point in that window and receive a deterministic result. If the promise is seven daily snapshots plus four weekly snapshots, the UI and runbooks must expose those exact boundaries.

Tests should exercise the boundary dates, retries after partial deletion, concurrent restore and deletion, corrupt or incomplete manifests, and a tenant with no eligible snapshots. The most valuable test is not a large upload benchmark. It is proving that the catalog never advertises a snapshot whose required objects have already crossed the deletion boundary.

Retention is therefore a product contract expressed as storage policy. The bill follows the bytes that contract permits us to keep. A precise deletion workflow reduces those bytes while preserving the restore choices that players and operators actually depend on.

## Further reading

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/Versioning.html
- https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- https://www.rfc-editor.org/rfc/rfc9110.html
