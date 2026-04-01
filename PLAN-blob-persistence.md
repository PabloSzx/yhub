# Plan: Generic Blob Storage Adapter for Azure/GCS Support

## Context

y/hub currently has a single storage plugin (`S3PersistenceV1` in `src/plugins/s3.js`) hardcoded to the MinIO SDK. We need to support Azure Blob Storage and GCS without adding cloud SDK dependencies to yhub. The caller brings their own SDK and provides simple blob operations via an adapter interface.

## Approach

Create a generic `BlobPersistence` class that accepts a simple adapter (`put`, `get`, `del`, `init?`). This class handles all yhub-specific logic. No new dependencies. `S3PersistenceV1` stays unchanged.

---

## 1. Create `src/plugins/blob.js`

New file (~80 lines) implementing `PersistencePlugin` via a caller-provided adapter.

**Adapter interface (caller provides):**

```javascript
/**
 * @typedef {{
 *   put: (path: string, data: Buffer) => Promise<void>,
 *   get: (path: string) => Promise<Buffer|null>,
 *   del: (path: string) => Promise<void>,
 *   init?: () => Promise<void>
 * }} BlobAdapter
 */
```

- `put(path, data)` — store a blob. Caller handles retries.
- `get(path)` — retrieve a blob. Return `null` if not found (caller handles 404 mapping).
- `del(path)` — delete a blob. Should not throw if missing.
- `init()` — optional setup (e.g. create container/bucket).

**Class: `BlobPersistence` (implements `t.PersistencePlugin`):**

```javascript
import * as t from '../types.js'
import * as buffer from 'lib0/buffer'
import { logger } from '../logger.js'

const log = logger.child({ module: 'blob' })

/**
 * @typedef {{
 *   put: (path: string, data: Buffer) => Promise<void>,
 *   get: (path: string) => Promise<Buffer|null>,
 *   del: (path: string) => Promise<void>,
 *   init?: () => Promise<void>
 * }} BlobAdapter
 */

/**
 * @implements {t.PersistencePlugin}
 */
export class BlobPersistence {
  /**
   * @param {string} pluginId - Unique identifier (e.g. 'AzureBlob:v1', 'GCS:v1')
   * @param {BlobAdapter} adapter
   */
  constructor (pluginId, adapter) {
    this._pluginId = pluginId
    this._adapter = adapter
  }

  get pluginid () {
    return this._pluginId
  }

  async init () {
    await this._adapter.init?.()
  }

  /**
   * @param {t.AssetId} assetId
   * @param {t.Asset} asset
   * @return {Promise<t.RetrievableAsset?>}
   */
  async store (assetId, asset) {
    if (assetId.branch === 'main') {
      const path = t.assetIdToString(assetId)
      const data = Buffer.from(buffer.encodeAny(asset))
      await this._adapter.put(path, data)
      return { type: 'asset:retrievable:v1', plugin: this._pluginId }
    }
    return null
  }

  /**
   * @param {t.AssetId} assetId
   * @param {t.Asset} assetInfo
   * @return {Promise<t.Asset?>}
   */
  async retrieve (assetId, assetInfo) {
    if (assetInfo.type === 'asset:retrievable:v1' && assetInfo.plugin === this._pluginId) {
      const path = t.assetIdToString(assetId)
      const data = await this._adapter.get(path)
      return data && t.$asset.expect(buffer.decodeAny(data))
    }
    return null
  }

  /**
   * @param {t.AssetId} assetId
   * @param {t.Asset} assetInfo
   * @return {Promise<boolean>}
   */
  async delete (assetId, assetInfo) {
    if (assetInfo.type !== 'asset:retrievable:v1' || assetInfo.plugin !== this._pluginId) {
      return false
    }
    const path = t.assetIdToString(assetId)
    setTimeout(() => {
      this._adapter.del(path).catch(err => log.error({ err, path }, 'error deleting object'))
    }, 10_000)
    return true
  }
}
```

**Key behaviors (mirroring `S3PersistenceV1`):**
- Only stores `main` branch assets — others stay in PostgreSQL
- Encodes with `buffer.encodeAny()`, decodes with `buffer.decodeAny()` + `t.$asset.expect()`
- Object key from `t.assetIdToString(assetId)` (format: `id:ydoc:v1/{org}/{docid}/{branch}/{gc}/{t}`)
- Delayed deletion (10s `setTimeout`) to avoid stale reads
- Returns `{ type: 'asset:retrievable:v1', plugin }` on store

**What the caller handles in their adapter:**
- SDK initialization and authentication
- Transient error detection and retries (cloud-specific error codes differ)
- 404/not-found → return `null` from `get`

---

## 2. Update `package.json` exports

Add new export and fix existing S3 types path typo:

```diff
 "./plugins/s3": {
   "default": "./src/plugins/s3.js",
-  "types": "./dist/src/storage/s3.d.ts"
+  "types": "./dist/src/plugins/s3.d.ts"
 },
+"./plugins/blob": {
+  "default": "./src/plugins/blob.js",
+  "types": "./dist/src/plugins/blob.d.ts"
+}
```

---

## 3. Update `.env.template`

Add commented guidance for Azure and GCS after the existing S3 block:

```
## Azure Blob Storage (use with BlobPersistence from @y/hub/plugins/blob)
# AZURE_STORAGE_CONNECTION_STRING=DefaultEndpointsProtocol=https;AccountName=...;AccountKey=...;EndpointSuffix=core.windows.net
# AZURE_STORAGE_CONTAINER=yhub

## Google Cloud Storage (use with BlobPersistence from @y/hub/plugins/blob)
# GCS_BUCKET=yhub
# GCS_PROJECT_ID=my-project
# GCS_KEY_FILENAME=/path/to/service-account.json
# Or set GOOGLE_APPLICATION_CREDENTIALS env var
```

---

## 4. No changes to existing files

- `src/plugins/s3.js` — unchanged, no breaking changes
- `src/persistence.js` — unchanged, already supports plugin arrays
- `src/types.js` — unchanged, `BlobPersistence` implements existing `PersistencePlugin`

---

## Usage Examples

### Azure Blob Storage

```javascript
import { BlobPersistence } from '@y/hub/plugins/blob'
import { BlobServiceClient } from '@azure/storage-blob' // caller's dependency

const client = BlobServiceClient.fromConnectionString(process.env.AZURE_STORAGE_CONNECTION_STRING)
const container = client.getContainerClient('yhub')

const plugin = new BlobPersistence('AzureBlob:v1', {
  init: () => container.createIfNotExists(),
  put: (path, data) => container.getBlockBlobClient(path).upload(data, data.length),
  get: async (path) => {
    try {
      const resp = await container.getBlockBlobClient(path).download()
      const chunks = []
      for await (const chunk of resp.readableStreamBody) chunks.push(chunk)
      return Buffer.concat(chunks)
    } catch (e) {
      if (e.statusCode === 404) return null
      throw e
    }
  },
  del: (path) => container.getBlockBlobClient(path).deleteIfExists()
})

createYHub({ persistence: [plugin], ... })
```

### Google Cloud Storage

```javascript
import { BlobPersistence } from '@y/hub/plugins/blob'
import { Storage } from '@google-cloud/storage' // caller's dependency

const storage = new Storage({ projectId: process.env.GCS_PROJECT_ID })
const bucket = storage.bucket(process.env.GCS_BUCKET)

const plugin = new BlobPersistence('GCS:v1', {
  init: async () => {
    const [exists] = await bucket.exists()
    if (!exists) await bucket.create()
  },
  put: (path, data) => bucket.file(path).save(data),
  get: async (path) => {
    try {
      const [data] = await bucket.file(path).download()
      return data
    } catch (e) {
      if (e.code === 404) return null
      throw e
    }
  },
  del: (path) => bucket.file(path).delete({ ignoreNotFound: true })
})

createYHub({ persistence: [plugin], ... })
```

---

## Files Summary

| File | Action |
|------|--------|
| `src/plugins/blob.js` | **Create** — ~80 lines, generic BlobPersistence class |
| `package.json` | **Edit** — add `./plugins/blob` export, fix S3 types path |
| `.env.template` | **Edit** — add Azure/GCS env var comments |

---

## Verification

1. `npm run lint` — standard + tsc passes
2. `npm test` — existing S3 tests still pass (nothing changed)
3. Manual smoke test: instantiate `BlobPersistence` with an in-memory adapter, call store/retrieve/delete
