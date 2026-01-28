# OceanBase HNSW Quantization Implementation Plan

> **Status**: Ready for Implementation (HNSW_SQ only)  
> **Date**: January 28, 2026  
> **Blocker**: HNSW_BQ does NOT support `inner_product` distance

---

## 1. Executive Summary

### Key Discovery
```
Docker tag `4.3.5-lts` = Version `4.3.5.5` (BP5)
```
**All quantization features are available. No version upgrade needed.**

### Compatibility Matrix

| Index Type | inner_product | Status |
|------------|:-------------:|--------|
| HNSW (32-bit) | ✅ | Current default |
| **HNSW_SQ (8-bit)** | ✅ | **Ready to implement** |
| HNSW_BQ (1-bit) | ❌ | **BLOCKED** - only supports l2/cosine |

### Recommendation
Implement **HNSW_SQ only** via `VECTOR_VQ_LEVEL=8`.


---

## 2. Validation Commands

### 2.1 Verify OceanBase Version
```bash
docker run --rm oceanbase/oceanbase-ce:4.3.5-lts cat /home/admin/oceanbase/etc/observer.conf.bin 2>/dev/null || \
docker logs $(docker ps -qf "ancestor=oceanbase/oceanbase-ce:4.3.5-lts") 2>&1 | grep "installed"
# Expected: oceanbase-ce-4.3.5.5 already installed
```

### 2.2 Test HNSW_SQ with inner_product
```sql
-- Connect to OceanBase
CREATE TABLE test_sq (id INT PRIMARY KEY, vec VECTOR(128));
INSERT INTO test_sq VALUES (1, '[1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]');

-- Create HNSW_SQ index with inner_product (should succeed)
CREATE VECTOR INDEX idx_sq ON test_sq(vec) 
  WITH (distance=inner_product, type=hnsw_sq, m=16, ef_construction=128);

-- Verify search works
SELECT id, inner_product(vec, '[1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]') AS score 
FROM test_sq ORDER BY score DESC APPROXIMATE LIMIT 10;

-- Clean up
DROP TABLE test_sq;
```

### 2.3 Test HNSW_BQ with inner_product (Expected to FAIL)
```sql
CREATE TABLE test_bq (id INT PRIMARY KEY, vec VECTOR(128));
INSERT INTO test_bq VALUES (1, '[1,0,0,...]');

-- This should fail with OB_NOT_SUPPORTED
CREATE VECTOR INDEX idx_bq ON test_bq(vec) 
  WITH (distance=inner_product, type=hnsw_bq, m=16, ef_construction=128);
-- Expected error: distance parameter only supports l2

DROP TABLE test_bq;
```


---

## 3. Implementation Plan

### 3.1 Code Changes

**File 1**: `packages/service/common/vectorDB/constants.ts`
```typescript
// Add after VectorVQ definition
export const OceanBaseIndexType = (() => {
  if (process.env.VECTOR_VQ_LEVEL === '8') return 'hnsw_sq';
  return 'hnsw';
})();
```

**File 2**: `packages/service/common/vectorDB/oceanbase/index.ts`
```typescript
// Import the constant
import { OceanBaseIndexType } from '../constants';

// Modify index creation (in init function)
// FROM: type=hnsw
// TO:   type=${OceanBaseIndexType}
```

### 3.2 Environment Variable

```bash
# .env
VECTOR_VQ_LEVEL=8  # Enables HNSW_SQ (8-bit) for OceanBase
```

| VECTOR_VQ_LEVEL | PgVector | OceanBase |
|-----------------|----------|-----------|
| 32 (default) | VECTOR | HNSW |
| 16 | HALFVEC | HNSW (no change) |
| **8** | - | **HNSW_SQ** |


---

## 4. BLOCKER: HNSW_BQ + inner_product

### The Problem

FastGPT uses `inner_product` for ALL vector databases:
```typescript
// oceanbase/index.ts
WITH (distance=inner_product, type=hnsw, ...)
SELECT ... inner_product(vector, [...]) AS score ... ORDER BY score desc
```

OceanBase HNSW_BQ **does NOT support inner_product**:

> **Source**: `oceanbase-doc-4.3.5/en-US/640.ob-vector-search/200.ob-vector-index.md` (Line 310)
> 
> "The `distance` parameter of an HNSW_BQ index only supports l2"

### Questions for OceanBase Team

1. **Is there any plan to add `inner_product` support for HNSW_BQ?**
   - Current: Only l2 and cosine (cosine from BP4+)
   - FastGPT requires inner_product for consistency across all vector DBs

2. **Source code shows `is_enable_bp_cosine_and_ip` flag for BP3+**
   - Does this actually enable inner_product for HNSW_BQ in versions [4.3.5.3, 4.4.0.0)?
   - Documentation says no, but code suggests maybe?

### Questions for FastGPT Team

1. **Should HNSW_BQ be implemented with L2 distance as an alternative?**
   - Would require separate env var (e.g., `OCEANBASE_DISTANCE=l2`)
   - Changes `ORDER BY score desc` to `ORDER BY score asc`
   - Breaks consistency with other vector DBs

2. **Or wait for OceanBase to support inner_product for HNSW_BQ?**


---

## 5. Memory Comparison

| Index Type | Memory (1M × 1536 dims) | Compression |
|------------|-------------------------|-------------|
| HNSW | ~6 GB | 1x |
| HNSW_SQ | ~2-3 GB | 2-3x |
| HNSW_BQ | ~405 MB | 15x |

---

## 6. Next Steps

- [x] Verify 4.3.5-lts = 4.3.5.5 (BP5) ✅
- [x] Confirm HNSW_BQ does NOT support inner_product ✅
- [ ] Test HNSW_SQ with inner_product on running OceanBase
- [ ] Implement HNSW_SQ support (Phase 1)
- [ ] Decide on HNSW_BQ approach (blocked pending answers)

---

*Simplified from 1400-line document. Full version: OCEANBASE_HNSW_QUANTIZATION_PLAN_old.md*
