---
name: json-file-persistence
description: Enforces standards for using flat JSON files as a lightweight persistence layer (atomic writes, schema validation, concurrency safety, backups) instead of a database. Use this skill only when the user explicitly asks for JSON/file-based storage, "no database", or a flat-file approach — e.g. "just save it to a JSON file", "store settings in a config file", "no database needed for this". Do not use this skill by default: for general persistence needs without an explicit file-storage request, the `sqlite-persistence` skill is this user's default. If an existing project already persists to a `.json` file, this skill applies even without an explicit request.
---

# JSON File Persistence Standards

You are an expert in lightweight, file-based data storage. When assisting with JSON-file persistence, strictly adhere to the following.

## 1. Architectural Philosophy

- JSON files are the right tool for small, low-write, single-process workloads: app configuration, user settings, prototypes, small datasets (roughly low thousands of records / a few MB), or Electron desktop apps storing local preferences. They are the wrong tool for anything with concurrent writers, data that needs querying/filtering at scale, or data integrity guarantees beyond "the file didn't get corrupted."
- This user's default persistence choice is SQLite (`sqlite-persistence` skill). Only use flat JSON files when the user explicitly asked for that approach, or an existing project is already built that way — don't silently choose JSON over SQLite for a new feature.
- Apply "always is cheap and sometimes is expensive": a plain `fs`-based read/write module is enough for most cases. Don't add a database-like query layer on top of a JSON file — if the project needs real querying, that's the signal to migrate to SQLite (§12), not to build one.

## 2. Tech Stack

- **Default:** Node's built-in `fs/promises` — no dependency needed for read/write/rename.
- Acceptable lightweight helper if the project wants a nicer API: [`lowdb`](https://github.com/typicode/lowdb) (thin wrapper over a JSON file, still just a file underneath). Don't reach for it by default — plain `fs` is usually simpler and it's one less dependency.
- Do NOT use a JSON file as a stand-in for a real database in disguise (e.g. hand-rolling indexes, joins, or transactions across multiple JSON files) — that complexity is exactly what SQLite is for.

## 3. Project Structure

```
data/
  settings.json        # one file per logical collection/entity
  users.json
  .backups/             # timestamped snapshots before risky writes
src/
  config/
    dataDir.js          # resolves the data directory (see §11 for Electron)
  models/
    settingsModel.js     # all reads/writes for settings.json live here
    userModel.js
  utils/
    jsonStore.js          # shared atomic read/write helpers
```

- Gitignore `data/*.json` and `data/.backups/` for anything containing real or user-generated data; commit a checked-in file only for static seed/config data meant to ship with the app.
- One JSON file per logical collection, not one giant file for the whole app — keeps reads/writes smaller and reduces the blast radius of a corrupted file.

## 4. File I/O: Always Write Atomically

A crash or concurrent write mid-write must never leave a half-written, corrupt JSON file. Never write directly to the target path — write to a temp file, then rename:

```js
// utils/jsonStore.js
import { promises as fs } from "fs";
import path from "path";

export async function readJson(filePath, fallback = {}) {
  try {
    const raw = await fs.readFile(filePath, "utf-8");
    return JSON.parse(raw);
  } catch (err) {
    if (err.code === "ENOENT") return fallback; // first run — no file yet
    throw err; // don't silently swallow a corrupt-file parse error
  }
}

export async function writeJson(filePath, data) {
  const tmpPath = `${filePath}.tmp`;
  await fs.writeFile(tmpPath, JSON.stringify(data, null, 2), "utf-8");
  await fs.rename(tmpPath, filePath); // atomic on the same filesystem
}
```

- `fs.rename` on the same filesystem is atomic — the file at `filePath` is either the old complete version or the new complete version, never a partial write. Never `fs.writeFile` straight to the real path for data that matters.
- Pretty-print (`JSON.stringify(data, null, 2)`) for anything a human might open to debug; skip the indentation only for files large enough that size matters.

## 5. Concurrency & Locking

- Node is single-threaded but `await`s interleave — two overlapping requests that both read-modify-write the same file can race and one write can silently clobber the other. Serialize writes to each file through an in-process queue:
  ```js
  // simple per-file write queue — one write to a given path finishes before the next starts
  const queues = new Map();
  export function enqueueWrite(filePath, task) {
    const prev = queues.get(filePath) || Promise.resolve();
    const next = prev.then(task, task);
    queues.set(filePath, next);
    return next;
  }
  ```
- This only protects against races *within one Node process*. JSON files are not safe for multiple server instances/processes writing the same file — if the project needs that, it needs a real database, not a lock file hack.
- Keep the read-modify-write cycle short: read, mutate in memory, write — don't hold a file "open" across an unrelated `await` (a network call, a timer) between reading and writing it.

## 6. Schema & Validation

- A JSON file enforces nothing — validate data at the model boundary with a schema library (`zod`), same as the `node-express-backend` skill requires for request input. Don't let unvalidated data get written to disk.
- Include a `_version` (or top-level `schemaVersion`) field in each file so a future format change can be detected and migrated in code, rather than breaking silently on old data.
- Define a TypeScript type or JSDoc typedef for each file's shape and import it everywhere that file is read/written, so a shape change is caught at edit time, not at runtime.

## 7. Query Patterns

- There is no query engine — load the whole file into memory and filter/map/sort with plain JS. This is fine at small scale; it is the reason this approach doesn't scale past roughly low-thousands of records or a few MB.
- Don't build ad-hoc indexing (e.g. maintaining a separate `users-by-email.json` lookup file) to work around this — that complexity is a clear signal to migrate to SQLite (§12) instead.
- Cache the parsed contents in memory for read-heavy files if reading the file per-request becomes a bottleneck, but always write through to disk immediately on any mutation — never let the in-memory copy and the on-disk copy diverge.

## 8. Backups

- Before any destructive or bulk write (a migration, a "reset" operation, a schema version bump), copy the current file into `data/.backups/<name>-<timestamp>.json` first.
- For long-running apps, keep a small rolling number of periodic snapshots (e.g. last 5) rather than one single overwritten backup, so a bad write has more than one recovery point.

## 9. Security

- Never store plaintext passwords, API keys, or tokens in a JSON data file — hash passwords the same as the `node-express-backend` skill requires, and keep secrets in environment variables, never in `data/`.
- The `data/` directory must not be inside anything served statically by the web server or reachable by a direct URL — treat it the same as a database file, not a public asset.
- If a filename or path is ever built from user input (e.g. a per-user file), sanitize it and resolve against a known base directory (`path.join` + a check that the result stays inside `data/`) to prevent path traversal.

## 10. Testing

- Point tests at a temp directory (`os.tmpdir()` or a `data/test/` folder cleaned up after each run) — never let tests read or write the real `data/` files.
- Seed test files directly (write the JSON fixture, then run the code under test) rather than driving the whole app through its API just to get data into place.

## 11. Electron / Desktop Apps

- Never store the JSON data file inside the app's install/source directory — it's read-only after packaging on most platforms and gets wiped on reinstall. Use `app.getPath("userData")` to resolve a writable, per-user location, and build all file paths from that.
- Same atomic-write and per-file-queue rules apply — a desktop app crashing mid-write is at least as likely as a server request racing another.

## 12. Migration Path to SQLite

- Keep all reads/writes behind model functions (`settingsModel.get()`, `userModel.findByEmail()`) exactly as the `node-express-backend` skill structures the data layer — this is what makes swapping the storage backend later a contained change instead of a rewrite.
- Signs it's time to migrate to `sqlite-persistence`: the file is being fully rewritten often enough that write latency is noticeable, the data needs relational queries (filtering/joining across collections), multiple processes need to write it, or the file has grown past a few MB.
- When migrating, write a one-time script that reads the JSON file(s) and inserts the rows into SQLite via the model layer already defined in `sqlite-persistence` — don't hand-write a bespoke import path.

## 13. Instructions for the Agent

1. Acknowledge that you are applying the `json-file-persistence` skill.
2. Confirm this is the right call: if the feature sounds like it needs querying, relational data, or multiple writers, say so and suggest `sqlite-persistence` instead before building on JSON files.
3. Always write through the atomic write helper (§4) — never a direct `fs.writeFile` to the real data path.
4. Always validate data with a schema at the model boundary before writing it to disk.
5. Keep all file reads/writes behind model functions, never scattered `fs` calls in routes/controllers or UI code.
