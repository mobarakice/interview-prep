# PostgreSQL Masterclass Cheatsheet

> Quick-reference checklist for PostgreSQL transaction isolation, execution planning, indexing, and connection configurations

---

## 1. Transaction Isolation Reference

- **Dirty Read**: Transaction A reads uncommitted modifications from Transaction B.
- **Non-Repeatable Read**: Transaction A reads a row, Transaction B updates the row, Transaction A reads the row again and sees the new value.
- **Phantom Read**: Transaction A queries range, Transaction B inserts a new row, Transaction A queries range again and sees the new row.

| Level | Dirty Reads | Non-Repeatable | Phantom Reads |
|---|---|---|---|
| `Read Uncommitted` | Allowed | Allowed | Allowed |
| `Read Committed` (PG Default) | Prevented | Allowed | Allowed |
| `Repeatable Read` | Prevented | Prevented | Prevented (In PG) |
| `Serializable` | Prevented | Prevented | Prevented |

---

## 2. Query Analysis & Tuning

- Run `EXPLAIN (ANALYZE, BUFFERS)` to inspect query performance details.
- **Checklist**:
  - Eliminate `Seq Scan` (table scans) on high-volume queries by adding indexes.
  - Fix `Filter` conditions by utilizing partial indexes (`CREATE INDEX ... WHERE status = 'PENDING'`).
  - Optimize memory sorting by adjusting `work_mem` settings to prevent temp file creation on disk.

---

## 3. Index Types & Use Cases

- **B-Tree (Default)**: Equality (`=`), range (`<`, `>`), and sorting operations.
- **GIN (Generalized Inverted Index)**: Array columns, JSONB JSON matching (`column @> '{"status": "ok"}'`).
- **GIST (Generalized Search Tree)**: Spatial geocoordinates, PostGIS queries, range bounds.
- **Partial Index**: Reduces index size. Useful when filtering by common status columns:
  ```sql
  CREATE INDEX idx_pending_orders ON orders (created_at) WHERE status = 'PENDING';
  ```
