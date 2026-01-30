# OceanBase HNSW Quantization Implementation Plan

> **Status**: ✅ Ready for Implementation  
> **Date**: January 30, 2026  
> **Scope**: HNSW_SQ (8-bit) and HNSW_BQ (1-bit) with different distance algorithms

---

## 0. Maintainer Decisions (Confirmed)

| Question | Decision | Implication |
|----------|----------|-------------|
| **Q1: HNSW_BQ distance** | ✅ Build different SQL per quantization level | HNSW_BQ uses `cosine`, HNSW/HNSW_SQ use `inner_product` |
| **Q2: Env var naming** | ✅ Use numbers directly (`VECTOR_VQ_LEVEL`) | `32`=hnsw, `8`=hnsw_sq, `1`=hnsw_bq |
| **Q3: HNSW parameters** | ✅ Use million-level defaults | `m=16, ef_construction=200` (code creates first time only) |
| **Q4: Index migration** | ✅ Document only, no code migration | Users manually `DROP INDEX` + restart if switching types |

---

## 1. Executive Summary

### Key Discovery
```
Docker tag `4.3.5-lts` = Version `4.3.5.5` (BP5)
```
**All quantization features are available. No version upgrade needed.**

### Compatibility Matrix

| Index Type | Distance | Status |
|------------|:--------:|--------|
| HNSW (32-bit) | `inner_product` | Current default |
| **HNSW_SQ (8-bit)** | `inner_product` | **Ready to implement** |
| **HNSW_BQ (1-bit)** | `cosine` | **Ready to implement** (different distance) |

---

## 2. Implementation Details

### 2.1 File Changes Overview

| File | Change |
|------|--------|
| `packages/service/common/vectorDB/constants.ts` | Add `OceanBaseIndexConfig` export |
| `packages/service/common/vectorDB/oceanbase/index.ts` | Use dynamic index/query config |

---

### 2.2 constants.ts Changes

**Location**: `packages/service/common/vectorDB/constants.ts`

```typescript
// After existing VectorVQ definition (line ~27)

/**
 * OceanBase Index Configuration based on VECTOR_VQ_LEVEL
 * 
 * | Level | Index Type | Distance       | Memory Savings |
 * |-------|------------|----------------|----------------|
 * | 32    | hnsw       | inner_product  | 1x (baseline)  |
 * | 8     | hnsw_sq    | inner_product  | 2-3x           |
 * | 1     | hnsw_bq    | cosine         | ~15x           |
 */
export const OceanBaseIndexConfig = (() => {
  const level = process.env.VECTOR_VQ_LEVEL;
  
  if (level === '1') {
    // HNSW_BQ: 1-bit binary quantization
    // Note: HNSW_BQ only supports l2/cosine, NOT inner_product
    return {
      type: 'hnsw_bq',
      distance: 'cosine',
      distanceFunc: 'cosine_distance',
      orderDirection: 'ASC',  // cosine_distance: smaller = more similar
      scoreTransform: (score: number) => 1 - score / 2  // Convert to similarity [0,1]
    };
  }
  
  if (level === '8') {
    // HNSW_SQ: 8-bit scalar quantization
    return {
      type: 'hnsw_sq',
      distance: 'inner_product',
      distanceFunc: 'inner_product',
      orderDirection: 'DESC',  // inner_product: larger = more similar
      scoreTransform: (score: number) => score
    };
  }
  
  // Default: Full precision HNSW
  return {
    type: 'hnsw',
    distance: 'inner_product',
    distanceFunc: 'inner_product',
    orderDirection: 'DESC',
    scoreTransform: (score: number) => score
  };
})();
```

---

### 2.3 oceanbase/index.ts Changes

**Location**: `packages/service/common/vectorDB/oceanbase/index.ts`

#### Change 1: Import the config

```typescript
// Line 2: Update import
import { DatasetVectorTableName, OceanBaseIndexConfig } from '../constants';
```

#### Change 2: Update init() - Index Creation

```typescript
// Line 23-25: Replace the CREATE VECTOR INDEX statement
await ObClient.query(
  `CREATE VECTOR INDEX IF NOT EXISTS vector_index ON ${DatasetVectorTableName}(vector) WITH (distance=${OceanBaseIndexConfig.distance}, type=${OceanBaseIndexConfig.type}, m=16, ef_construction=200);`
);
```

**Key changes from current code**:
- `distance=inner_product` → `distance=${OceanBaseIndexConfig.distance}`
- `type=hnsw` → `type=${OceanBaseIndexConfig.type}`
- `m=32` → `m=16` (OceanBase million-level recommendation)
- `ef_construction=128` → `ef_construction=200` (OceanBase recommendation)

#### Change 3: Update embRecall() - Query

```typescript
// Line 133-143: Replace the SELECT query
const rows = await ObClient.query<
  ({
    id: string;
    collection_id: string;
    score: number;
  } & RowDataPacket)[][]
>(
  `BEGIN;
      SET ob_hnsw_ef_search = ${global.systemEnv?.hnswEfSearch || 100};
      SELECT id, collection_id, ${OceanBaseIndexConfig.distanceFunc}(vector, [${vector}]) AS score
        FROM ${DatasetVectorTableName}
        WHERE team_id='${teamId}'
          AND dataset_id IN (${datasetIds.map((id) => `'${String(id)}'`).join(',')})
          ${filterCollectionIdSql}
          ${forbidCollectionSql}
        ORDER BY score ${OceanBaseIndexConfig.orderDirection} APPROXIMATE LIMIT ${limit};
    COMMIT;`
).then(([rows]) => rows[2]);

return {
  results: rows.map((item) => ({
    id: String(item.id),
    collectionId: item.collection_id,
    score: OceanBaseIndexConfig.scoreTransform(item.score)
  }))
};
```

**Key changes**:
- `inner_product(vector, ...)` → `${OceanBaseIndexConfig.distanceFunc}(vector, ...)`
- `ORDER BY score desc` → `ORDER BY score ${OceanBaseIndexConfig.orderDirection}`
- Add score transformation for cosine_distance compatibility

---

## 3. Configuration Summary

### Environment Variable Mapping

| `VECTOR_VQ_LEVEL` | PGVector | OceanBase Index | Distance | Memory |
|-------------------|----------|-----------------|----------|--------|
| `32` (default) | `VECTOR` | `hnsw` | `inner_product` | 100% |
| `16` | `HALFVEC` | `hnsw` | `inner_product` | 100% |
| `8` | — | `hnsw_sq` | `inner_product` | ~40% |
| `1` | — | `hnsw_bq` | `cosine` | ~7% |

### Index Parameters (Million-level defaults)

| Parameter | Value | Rationale |
|-----------|-------|-----------|
| `m` | 16 | OceanBase recommendation for <1M vectors |
| `ef_construction` | 200 | OceanBase recommendation for <1M vectors |
| `ef_search` | 100 | Configurable via `global.systemEnv?.hnswEfSearch` |

---

## 4. Migration Guide (Documentation Only)

### For Existing OceanBase Deployments

Per maintainer decision, migration is **not handled in code**. Users switching index types must:

1. **Connect to OceanBase directly**:
   ```bash
   mysql -h <host> -P <port> -u <user> -p<password> -D fastgpt
   ```

2. **Drop the existing index**:
   ```sql
   DROP INDEX vector_index ON modeldata;
   ```

3. **Set the environment variable** and restart FastGPT:
   ```bash
   VECTOR_VQ_LEVEL=8  # For HNSW_SQ
   # or
   VECTOR_VQ_LEVEL=1  # For HNSW_BQ
   ```

4. **FastGPT will recreate the index** on startup with the new type.

### Important Notes

- **hnsw ↔ hnsw_sq**: Supported via `DBMS_VECTOR.REBUILD_INDEX` (but we use DROP+CREATE for simplicity)
- **hnsw ↔ hnsw_bq**: **NOT** supported via REBUILD_INDEX; requires DROP+CREATE
- Code uses `IF NOT EXISTS`, so it won't recreate if index already exists

---

## 5. Memory Comparison

| Index Type | Memory (1M × 1536 dims) | Compression | Use Case |
|------------|-------------------------|-------------|----------|
| HNSW | ~6 GB | 1x | Maximum recall |
| HNSW_SQ | ~2-3 GB | 2-3x | Balanced |
| HNSW_BQ | ~405 MB | 15x | Memory-constrained |

---

## 6. Testing Checklist

### Unit Tests

- [ ] `VECTOR_VQ_LEVEL=32` → Creates `type=hnsw, distance=inner_product`
- [ ] `VECTOR_VQ_LEVEL=8` → Creates `type=hnsw_sq, distance=inner_product`
- [ ] `VECTOR_VQ_LEVEL=1` → Creates `type=hnsw_bq, distance=cosine`
- [ ] Query with HNSW_BQ uses `cosine_distance` and `ORDER BY ASC`
- [ ] Score transformation for HNSW_BQ returns values in expected range

### Integration Tests

- [ ] HNSW_SQ index creation succeeds on OceanBase 4.3.5 BP5
- [ ] HNSW_BQ index creation succeeds on OceanBase 4.3.5 BP5
- [ ] Vector search returns correct results with HNSW_SQ
- [ ] Vector search returns correct results with HNSW_BQ
- [ ] Recall rate is acceptable (>90%) for both index types

---

## 7. SQL Reference

### HNSW (Full Precision)
```sql
-- Index
CREATE VECTOR INDEX IF NOT EXISTS vector_index ON modeldata(vector) 
  WITH (distance=inner_product, type=hnsw, m=16, ef_construction=200);

-- Query
SELECT id, collection_id, inner_product(vector, [...]) AS score
  FROM modeldata WHERE ...
  ORDER BY score DESC APPROXIMATE LIMIT 10;
```

### HNSW_SQ (8-bit)
```sql
-- Index
CREATE VECTOR INDEX IF NOT EXISTS vector_index ON modeldata(vector) 
  WITH (distance=inner_product, type=hnsw_sq, m=16, ef_construction=200);

-- Query (same as HNSW)
SELECT id, collection_id, inner_product(vector, [...]) AS score
  FROM modeldata WHERE ...
  ORDER BY score DESC APPROXIMATE LIMIT 10;
```

### HNSW_BQ (1-bit)
```sql
-- Index (note: cosine distance, NOT inner_product)
CREATE VECTOR INDEX IF NOT EXISTS vector_index ON modeldata(vector) 
  WITH (distance=cosine, type=hnsw_bq, m=16, ef_construction=200);

-- Query (note: cosine_distance and ASC order)
SELECT id, collection_id, cosine_distance(vector, [...]) AS score
  FROM modeldata WHERE ...
  ORDER BY score ASC APPROXIMATE LIMIT 10;
```

---

## 8. Next Steps

- [x] Verify 4.3.5-lts = 4.3.5.5 (BP5) ✅
- [x] Confirm HNSW_BQ distance limitations ✅
- [x] Get maintainer decisions ✅
- [ ] Implement `OceanBaseIndexConfig` in constants.ts
- [ ] Update oceanbase/index.ts init() and embRecall()
- [ ] Add documentation to FastGPT docs
- [ ] Test on running OceanBase instance
- [ ] Submit PR

---

*Updated: January 30, 2026 - Added maintainer decisions and complete implementation details*
