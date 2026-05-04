# GitNexus Embeddings WAL Corruption Fix

## Problem Summary

When using `gitnexus analyze --embeddings` on Windows, the SQLite/LadybugDB WAL (Write-Ahead Log) file becomes corrupted, causing all subsequent database operations to fail with:

```
Runtime exception: Corrupted wal file. Read out invalid WAL record type.
```

## Root Cause

The issue was identified in `gitnexus/src/core/lbug/lbug-adapter.ts`. The `closeLbug()` function and related database cleanup paths were closing the database connection **without** issuing a `CHECKPOINT` command first.

### Why This Matters

LadybugDB 0.16.0 uses a non-blocking checkpoint thread that can outlive the `close()` call. When embeddings are generated:

1. Large amounts of data are written to the WAL file
2. The connection is closed without checkpointing
3. The background checkpoint thread may still be writing
4. WAL pages remain pending on disk
5. The next database open either:
   - Races with WAL replay, or
   - Trips the database-id check on orphaned `.wal`/`.shadow` sidecars

This is **especially critical** for embeddings because:
- Embedding writes generate significantly more WAL data than normal indexing
- The issue document shows 151.7s for embeddings vs 10.1s without (15x slower)
- More WAL data = higher chance of incomplete checkpoint on close

### Evidence from Codebase

The `bridge-db.ts` file (used for cross-repo group sync) **already had this fix**:

```typescript
// bridge-db.ts:236-238
// CHECKPOINT before close so the WAL/.shadow contents are flushed into
// the main database file. Without this, LadybugDB 0.16.0's non-blocking
// checkpoint thread can outlive the close call and leave sidecar pages
// pending on disk...
try {
  await (handle._conn as lbug.Connection).query('CHECKPOINT');
} catch {
  /* ignore — older LadybugDB or schemaless DB may not accept it */
}
```

But the main `lbug-adapter.ts` file did **not** have this protection.

## The Fix

Added `CHECKPOINT` calls before closing the database in three locations in `lbug-adapter.ts`:

### 1. `closeLbug()` - Main cleanup function

```typescript
export const closeLbug = async (): Promise<void> => {
  // CHECKPOINT before close so the WAL/.shadow contents are flushed into
  // the main database file. Without this, LadybugDB 0.16.0's non-blocking
  // checkpoint thread can outlive the close call and leave sidecar pages
  // pending on disk, which makes a subsequent read-side open either race
  // with the WAL replay or trip the database-id check on the sidecars.
  // This is especially critical after embedding writes, which generate
  // large amounts of WAL data. CHECKPOINT is a no-op when there's nothing
  // pending, so it's cheap on the happy path.
  if (conn) {
    try {
      await conn.query('CHECKPOINT');
    } catch {
      /* ignore — older LadybugDB or schemaless DB may not accept it */
    }
  }
  // ... rest of close logic
};
```

### 2. `doInitLbug()` - Database switch path

```typescript
const doInitLbug = async (dbPath: string) => {
  // Different database requested — close the old one first
  if (conn || db) {
    // CHECKPOINT before close to flush WAL contents (same rationale as closeLbug)
    if (conn) {
      try {
        await conn.query('CHECKPOINT');
      } catch {
        /* ignore — older LadybugDB or schemaless DB may not accept it */
      }
    }
    // ... rest of close logic
  }
  // ... rest of init logic
};
```

### 3. `withLbugDb()` - Retry cleanup path

```typescript
// Inside the retry loop cleanup:
await runWithSessionLock(async () => {
  // CHECKPOINT before close to flush WAL contents (same rationale as closeLbug)
  if (conn) {
    try {
      await conn.query('CHECKPOINT');
    } catch {
      /* best-effort */
    }
  }
  // ... rest of cleanup logic
});
```

## Why This Fix Works

1. **Explicit WAL flush**: `CHECKPOINT` forces LadybugDB to flush all pending WAL pages into the main database file before the connection closes
2. **No-op when clean**: If there's nothing pending, `CHECKPOINT` is cheap (no performance penalty)
3. **Graceful degradation**: The `try/catch` ensures compatibility with older LadybugDB versions that may not support `CHECKPOINT`
4. **Consistent with bridge-db**: Applies the same proven pattern already used in the group sync code

## Testing Recommendations

After applying this fix, test the following scenarios:

### 1. Basic Embeddings Test
```bash
# Clean start
npx gitnexus clean --force

# Index with embeddings
npx gitnexus analyze --embeddings --skills --verbose

# Verify query works (should NOT return empty results)
npx gitnexus query "orchestrator" -r <repo-name> --limit 5

# Verify cypher works (should NOT show WAL corruption)
npx gitnexus cypher "MATCH (n:Symbol) RETURN count(n) as total" -r <repo-name>
```

### 2. Incremental Embeddings Test
```bash
# First run
npx gitnexus analyze --embeddings --skills

# Make a small code change
# Second run (incremental)
npx gitnexus analyze --embeddings --skills

# Verify no corruption
npx gitnexus query "test query" -r <repo-name>
```

### 3. Database Switch Test
```bash
# Index repo A with embeddings
cd /path/to/repo-a
npx gitnexus analyze --embeddings

# Switch to repo B with embeddings
cd /path/to/repo-b
npx gitnexus analyze --embeddings

# Query both repos (tests database switching)
npx gitnexus query "test" -r repo-a
npx gitnexus query "test" -r repo-b
```

## Performance Impact

**Negligible to none**:
- `CHECKPOINT` is a no-op when there's nothing pending
- When there is pending data, the checkpoint would have happened anyway (just later, in a background thread)
- Making it explicit and synchronous prevents corruption at the cost of a few milliseconds during close

## Files Modified

- `gitnexus/src/core/lbug/lbug-adapter.ts` - Added `CHECKPOINT` calls in 3 locations

## Related Issues

- Original issue: `fork/issues/gitnexus-embeddings-wal-corruption-issue.md`
- LadybugDB 0.16.0 non-blocking checkpoint behavior
- Windows file locking and WAL mode interaction

## References

- SQLite WAL mode: https://www.sqlite.org/wal.html
- LadybugDB checkpoint behavior (0.16.0+)
- GitNexus `bridge-db.ts` implementation (lines 236-241)

---

**Fix Applied**: 2026-05-04  
**Status**: Ready for testing
