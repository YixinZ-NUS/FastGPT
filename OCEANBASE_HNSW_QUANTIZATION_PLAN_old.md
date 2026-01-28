# OceanBase HNSW Quantization Feature Implementation Plan

> **Feature Request**: Enable OceanBase HNSW quantization (8-bit via HNSW_SQ and 1-bit via HNSW_BQ) through environment variables, similar to PgVector's half-precision configuration.
>
> **Date**: January 27, 2026  
> **Status**: Planning Phase  
> **Reference**: [AGENTS.md](AGENTS.md)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Current State Analysis](#2-current-state-analysis)
3. [OceanBase Version & BP Requirements](#3-oceanbase-version--bp-requirements)
4. [Critical Compatibility Issue: Distance Function](#4-critical-compatibility-issue-distance-function)
5. [Technical Implementation Options](#5-technical-implementation-options)
6. [Proposed Solution](#6-proposed-solution)
7. [Implementation Details](#7-implementation-details)
8. [Migration & Backward Compatibility](#8-migration--backward-compatibility)
9. [Questions for Official Maintainers](#9-questions-for-official-maintainers)
10. [Next Steps](#10-next-steps)

---

## 1. Executive Summary

### 1.1 Feature Request (Original Chinese)
> 目前 Oceanbase 是固定全精度配置，OB 官方 HNSW 已经支持 8bit 和 1bit，希望可以通过环境变量指定量化等级。可参考 PGVector 半精度配置。

### 1.2 Key Findings

| Finding | Impact | Severity |
|---------|--------|----------|
| **HNSW_BQ + `inner_product`** | Docs say unsupported, but source code shows BP3+ may support | 🟡 **VERIFY** |
| Current OceanBase version `4.3.5-lts` has no BP suffix | Likely base version (BP0), missing quantization features | �� **BLOCKER** |
| HNSW_SQ requires BP1+, HNSW_BQ requires BP2+ | Version upgrade required | 🟡 High |
| HNSW_BQ cosine support only from BP4+ | Limited distance options even after upgrade | 🟡 High |
| HNSW_SQ supports all distance types (l2/inner_product/cosine) | Viable path for 8-bit quantization | 🟢 Good |

### 1.3 Recommendation

**Phase 1 (Immediate)**: Implement HNSW_SQ (8-bit) support only, which supports `inner_product`.

**Phase 2 (Future)**: HNSW_BQ (1-bit) requires either:
- Changing FastGPT's distance metric to `l2` or `cosine` (breaking change)
- Waiting for OceanBase to add `inner_product` support to HNSW_BQ

---

## 2. Current State Analysis

### 2.1 FastGPT OceanBase Implementation

**File**: `packages/service/common/vectorDB/oceanbase/index.ts`

```typescript
// Current table creation - FIXED full precision
CREATE TABLE IF NOT EXISTS ${DatasetVectorTableName} (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    vector VECTOR(1536) NOT NULL,  // Always full precision
    team_id VARCHAR(50) NOT NULL,
    dataset_id VARCHAR(50) NOT NULL,
    collection_id VARCHAR(50) NOT NULL,
    createtime TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

// Current index creation - FIXED HNSW type with inner_product
CREATE VECTOR INDEX IF NOT EXISTS vector_index 
ON ${DatasetVectorTableName}(vector) 
WITH (distance=inner_product, type=hnsw, m=32, ef_construction=128);
```

**File**: `packages/service/common/vectorDB/oceanbase/index.ts` (search query)

```typescript
// embRecall function uses inner_product
SELECT id, collection_id, inner_product(vector, [${vector}]) AS score
  FROM ${DatasetVectorTableName}
  WHERE team_id='${teamId}'
    AND dataset_id IN (...)
  ORDER BY score desc APPROXIMATE LIMIT ${limit};
```

### 2.2 PgVector Reference Implementation

**File**: `packages/service/common/vectorDB/pg/index.ts`

```typescript
const isHalfVec = VectorVQ === 16;

// Table creation adapts to precision
vector ${isHalfVec ? 'HALFVEC(1536)' : 'VECTOR(1536)'} NOT NULL

// Index creation uses different operators
USING hnsw (vector ${isHalfVec ? 'halfvec_ip_ops' : 'vector_ip_ops'})
```

### 2.3 Current Environment Variable

**File**: `packages/service/common/vectorDB/constants.ts`

```typescript
export const VectorVQ = (() => {
  if (process.env.VECTOR_VQ_LEVEL === '32') return 32;
  if (process.env.VECTOR_VQ_LEVEL === '16') return 16;
  if (process.env.VECTOR_VQ_LEVEL === '8') return 8;
  if (process.env.VECTOR_VQ_LEVEL === '4') return 4;
  if (process.env.VECTOR_VQ_LEVEL === '2') return 2;
  return 32;
})();
```

**Note**: Values 8, 4, 2 are defined but NOT implemented for any database.

---

## 3. OceanBase Version & BP Requirements

### 3.1 Current FastGPT Configuration

**File**: `deploy/args.json`
```json
{
  "tags": {
    "oceanbase": "4.3.5-lts"
  }
}
```

**Critical Issue**: The tag `4.3.5-lts` has NO BP (Build Patch) suffix, indicating it's the base release (effectively BP0).

### 3.2 Feature Availability by BP Version

| Feature | BP Version Required | Distance Support | Notes |
|---------|---------------------|------------------|-------|
| HNSW (full precision) | Base (BP0) | l2, inner_product, cosine | ✅ Current FastGPT |
| **HNSW_SQ (8-bit)** | **BP1+** | l2, inner_product, cosine | ✅ Compatible with FastGPT |
| **HNSW_BQ (1-bit/RaBitQ)** | **BP2+** | l2 only | ❌ NOT compatible |
| HNSW_BQ + cosine | BP4+ | l2, cosine | ❌ Still no inner_product |
| IVF/IVF_PQ | BP1+ | l2, inner_product, cosine | Alternative option |
| `refine_k` parameter | BP3+ | - | For HNSW_BQ tuning |
| `refine_type` parameter | BP3+ | sq8/fp32 | For HNSW_BQ precision |

### 3.3 BP Version Release Timeline (from docs)

| Version | Release Date | Key Vector Features |
|---------|--------------|---------------------|
| V4.3.5 BP1 | 2025-03-18 | HNSW_SQ, IVF indexes |
| V4.3.5 BP2 | 2025-05-15 | HNSW_BQ indexes |
| V4.3.5 BP3 | 2025-07-21 | refine_k, refine_type, adaptive memory |
| V4.3.5 BP4 | 2025-09-10 | HNSW_BQ cosine support |
| V4.3.5 BP5 | 2025-11-17 | IVF improvements, similarity functions |

### 3.4 Docker Image Availability

Need to verify availability of BP-versioned images:
- `oceanbase/oceanbase-ce:4.3.5-bp1`
- `oceanbase/oceanbase-ce:4.3.5-bp2`
- etc.

**Action Required**: Check Docker Hub for available tags.


---

## 4. Critical Compatibility Issue: Distance Function

### 4.1 The Core Problem

FastGPT uses `inner_product` (内积) as its distance metric for ALL vector databases:

```typescript
// OceanBase index creation
WITH (distance=inner_product, type=hnsw, ...)

// OceanBase search query  
SELECT ... inner_product(vector, [...]) AS score ... ORDER BY score desc
```

However, **HNSW_BQ (1-bit quantization) does NOT support `inner_product`**:

| Index Type | l2 | inner_product | cosine |
|------------|:--:|:-------------:|:------:|
| HNSW | ✅ | ✅ | ✅ |
| HNSW_SQ | ✅ | ✅ | ✅ |
| HNSW_BQ | ✅ | ❌ | ✅ (BP4+) |
| IVF/IVF_PQ | ✅ | ✅ | ✅ |

**Source**: OceanBase Documentation
> "HNSW_BQ 索引 `distance` 参数支持 l2 和 cosine。cosine 从 V4.3.5 BP4 版本开始支持。"
> 
> Translation: "HNSW_BQ index `distance` parameter supports l2 and cosine. Cosine is supported starting from V4.3.5 BP4."

### 4.2 Why This Matters

1. **Cannot simply add HNSW_BQ support** - The existing `inner_product` queries would fail
2. **Changing to l2/cosine is a breaking change** - Affects search semantics and score interpretation
3. **PgVector doesn't have this limitation** - Its HALFVEC type works with all distance operators

### 4.3 Is 1-bit Quantization the Same as HNSW_BQ/RaBitQ?

**Yes**, OceanBase's HNSW_BQ uses the **RaBitQ** algorithm:

> "HNSW_BQ 索引的召回率略低于 HNSW 索引，但显著减少了内存占用。BQ 量化压缩算法（Rabitq）能将向量压缩至原大小的 1/32"
>
> Translation: "HNSW_BQ index has slightly lower recall than HNSW, but significantly reduces memory usage. The BQ quantization algorithm (RaBitQ) can compress vectors to 1/32 of original size."

**Technical Details**:
- RaBitQ = Randomized Binary Quantization
- Compresses each float32 dimension to 1 bit
- 32x compression ratio (1536 dims × 4 bytes → 1536 bits = 192 bytes)
- Requires reranking (`refine_k`) for good recall

### 4.4 Distance Function Technical Details

| Distance | Formula | Score Interpretation | OB Function |
|----------|---------|---------------------|-------------|
| L2 (Euclidean) | √Σ(aᵢ-bᵢ)² | Lower = More Similar | `l2_distance()` |
| Inner Product | Σ(aᵢ×bᵢ) | Higher = More Similar | `inner_product()` |
| Cosine | 1 - (a·b)/(‖a‖‖b‖) | Lower = More Similar | `cosine_distance()` |
| Negative IP | -Σ(aᵢ×bᵢ) | Lower = More Similar | `negative_inner_product()` |

**FastGPT Current Behavior**:
```sql
ORDER BY score desc  -- Because inner_product: higher = better
```

If changed to L2 or Cosine:
```sql
ORDER BY score asc   -- Because: lower = better
```

---

## 5. Technical Implementation Options

### 5.1 Option A: HNSW_SQ Only (Recommended)

**Scope**: Implement 8-bit quantization only using HNSW_SQ

| Pros | Cons |
|------|------|
| ✅ Supports `inner_product` | ❌ Only ~2-3x memory reduction |
| ✅ Minimal code changes | ❌ Doesn't fulfill full feature request |
| ✅ Same recall as HNSW | |
| ✅ BP1+ requirement (available) | |

**Memory Comparison** (1M vectors × 1536 dims):
- HNSW: ~6 GB
- HNSW_SQ: ~2-3 GB (1/2 to 1/3)
- HNSW_BQ: ~405 MB (1/15)

### 5.2 Option B: HNSW_BQ with Distance Change

**Scope**: Implement 1-bit quantization by changing distance metric

| Pros | Cons |
|------|------|
| ✅ Maximum memory savings | ❌ **Breaking change** for existing data |
| ✅ Full feature request | ❌ Requires migration strategy |
| | ❌ Changes search semantics |
| | ❌ Affects PgVector/Milvus consistency |

**Required Changes**:
1. Change index from `inner_product` to `l2` or `cosine`
2. Change search query from `inner_product()` to `l2_distance()` or `cosine_distance()`
3. Change `ORDER BY score desc` to `ORDER BY score asc`
4. Update score normalization logic

### 5.3 Option C: Dual-Index Strategy

**Scope**: Allow different distance metrics per vector database

| Pros | Cons |
|------|------|
| ✅ Flexible per-database | ❌ Complex configuration |
| ✅ No breaking change for existing | ❌ Inconsistent search behavior |
| | ❌ Hard to compare results across DBs |

### 5.4 Option D: Wait for OceanBase Update

**Scope**: Request OceanBase team to add `inner_product` support for HNSW_BQ

| Pros | Cons |
|------|------|
| ✅ No code changes | ❌ Unknown timeline |
| ✅ Maintains compatibility | ❌ May never happen |

---

## 6. Proposed Solution

### 6.1 Phased Approach

#### Phase 1: HNSW_SQ Support (8-bit) ✅ Recommended First

**Environment Variable**: Extend existing `VECTOR_VQ_LEVEL`

```bash
# .env
VECTOR_VQ_LEVEL=8  # Enable HNSW_SQ for OceanBase (requires BP1+)
```

**Behavior**:
| VECTOR_VQ_LEVEL | PgVector | OceanBase |
|-----------------|----------|-----------|
| 32 (default) | VECTOR + vector_ip_ops | HNSW |
| 16 | HALFVEC + halfvec_ip_ops | HNSW (no change) |
| 8 | (not implemented) | **HNSW_SQ** |

#### Phase 2: Consider HNSW_BQ Later

Only if:
1. OceanBase adds `inner_product` support to HNSW_BQ, OR
2. FastGPT decides to change its distance metric globally, OR
3. A separate env var allows opting into `l2`/`cosine` distance

### 6.2 Minimum Version Requirement

```json
// deploy/args.json
{
  "tags": {
    "oceanbase": "4.3.5-bp1"  // Minimum for HNSW_SQ
  }
}
```


---

## 7. Implementation Details

### 7.1 Files to Modify

| File | Changes |
|------|---------|
| `packages/service/common/vectorDB/constants.ts` | Add OceanBase-specific quantization constant |
| `packages/service/common/vectorDB/oceanbase/index.ts` | Modify index creation SQL |
| `packages/service/type/env.d.ts` | (Optional) Add new env var type |
| `projects/app/.env.template` | Document new OceanBase quantization option |
| `deploy/args.json` | Update OceanBase version to BP1+ |
| `deploy/docker/*/docker-compose.oceanbase.yml` | Update image tag |
| `document/content/docs/introduction/development/configuration.mdx` | Document feature |

### 7.2 Code Changes

#### 7.2.1 Constants Update

```typescript
// packages/service/common/vectorDB/constants.ts

export const VectorVQ = (() => {
  // ... existing code ...
})();

// NEW: OceanBase-specific index type
export const OceanBaseIndexType = (() => {
  if (process.env.VECTOR_VQ_LEVEL === '8') {
    return 'hnsw_sq';  // 8-bit quantization
  }
  // Future: HNSW_BQ support would go here (requires distance change)
  return 'hnsw';  // Default full precision
})();
```

#### 7.2.2 OceanBase Index Creation

```typescript
// packages/service/common/vectorDB/oceanbase/index.ts

import { OceanBaseIndexType } from '../constants';

// In init() function:
await ObClient.query(
  `CREATE VECTOR INDEX IF NOT EXISTS vector_index 
   ON ${DatasetVectorTableName}(vector) 
   WITH (distance=inner_product, type=${OceanBaseIndexType}, m=32, ef_construction=128);`
);
```

#### 7.2.3 Environment Template

```bash
# projects/app/.env.template

# 向量库优先级: pg > oceanbase > milvus
# PG 向量库连接参数
VECTOR_VQ_LEVEL=32 # 向量量化等级
# - PG: 32 (全精度), 16 (半精度 HALFVEC)
# - OceanBase: 32 (HNSW 全精度), 8 (HNSW_SQ 8位量化, 需要 BP1+ 版本)
# - 注意: OceanBase HNSW_BQ (1位量化) 暂不支持，因为不兼容 inner_product 距离函数
```

### 7.3 SQL Syntax Reference

#### HNSW (Current - Full Precision)
```sql
CREATE VECTOR INDEX vector_index ON modeldata(vector) 
WITH (distance=inner_product, type=hnsw, m=32, ef_construction=128);
```

#### HNSW_SQ (8-bit Quantization)
```sql
CREATE VECTOR INDEX vector_index ON modeldata(vector) 
WITH (distance=inner_product, type=hnsw_sq, m=32, ef_construction=128);
```

#### HNSW_BQ (1-bit - NOT COMPATIBLE)
```sql
-- ❌ This would FAIL with inner_product
CREATE VECTOR INDEX vector_index ON modeldata(vector) 
WITH (distance=inner_product, type=hnsw_bq, m=32, ef_construction=128);

-- ✅ This works but requires changing FastGPT's distance metric
CREATE VECTOR INDEX vector_index ON modeldata(vector) 
WITH (distance=l2, type=hnsw_bq, m=32, ef_construction=128, refine_k=4);
```

### 7.4 Search Query Considerations

The search query does NOT need to change for HNSW_SQ:

```sql
-- Works with both HNSW and HNSW_SQ
SET ob_hnsw_ef_search = 100;
SELECT id, collection_id, inner_product(vector, [...]) AS score
FROM modeldata
WHERE team_id='...' AND dataset_id IN (...)
ORDER BY score desc APPROXIMATE LIMIT 10;
```

For HNSW_BQ (if implemented in future with l2):
```sql
-- Would require different function and sort order
SET ob_hnsw_ef_search = 100;
SELECT id, collection_id, l2_distance(vector, [...]) AS score
FROM modeldata
WHERE team_id='...' AND dataset_id IN (...)
ORDER BY score asc APPROXIMATE LIMIT 10;  -- Note: asc, not desc
```

### 7.5 HNSW_BQ Specific Parameters (For Future Reference)

If HNSW_BQ is ever implemented, these parameters are important:

| Parameter | Default | Range | Description |
|-----------|---------|-------|-------------|
| `refine_k` | 4.0 | [1.0, 1000.0] | Reranking factor for better recall |
| `refine_type` | sq8 | sq8/fp32 | Precision for reranking vectors |
| `bq_bits_query` | 32 | 0/4/32 | Search precision in bits |
| `bq_use_fht` | true | true/false | Fast Hadamard Transform |

**Recommended Values for Different Scales**:

| Data Scale | TopN | ef_search | refine_k |
|------------|------|-----------|----------|
| Million | Top10 | 64 | 4 |
| Million | Top100 | 240 | 4 |
| 10 Million | Top10 | 100 | 4 |
| 10 Million | Top100 | 1000 | 10 |

---

## 8. Migration & Backward Compatibility

### 8.1 New Deployments

No migration needed. Simply:
1. Use OceanBase 4.3.5 BP1+ image
2. Set `VECTOR_VQ_LEVEL=8` in environment
3. Start FastGPT

### 8.2 Existing Deployments

#### Scenario A: Keep Full Precision
No action needed. Default `VECTOR_VQ_LEVEL=32` maintains current behavior.

#### Scenario B: Migrate to HNSW_SQ

**Option 1: Drop and Recreate Index** (Recommended for smaller datasets)
```sql
-- Connect to OceanBase
DROP INDEX vector_index ON modeldata;
-- Restart FastGPT with VECTOR_VQ_LEVEL=8
-- Index will be recreated automatically
```

**Option 2: Use REBUILD_INDEX** (For larger datasets)
```sql
-- This is a full refresh, runs in background
CALL dbms_vector.rebuild_index('vector_index');
```

**Note**: Vector data in the table does NOT change. Only the index structure changes.

### 8.3 Rollback Strategy

If issues occur after enabling HNSW_SQ:
1. Set `VECTOR_VQ_LEVEL=32`
2. Restart FastGPT
3. Drop and recreate index (will use HNSW)

### 8.4 Version Compatibility Matrix

| FastGPT Version | OceanBase Version | VECTOR_VQ_LEVEL | Index Type |
|-----------------|-------------------|-----------------|------------|
| Current | 4.3.5-lts | 32 (only) | HNSW |
| After This PR | 4.3.5-bp1+ | 32 | HNSW |
| After This PR | 4.3.5-bp1+ | 8 | HNSW_SQ |
| After This PR | 4.3.5-lts | 8 | ❌ Error on startup |

---

## 9. Questions for Official Maintainers

### 9.1 For FastGPT Maintainers

1. **Single vs Multiple Environment Variables**
   - Current proposal extends `VECTOR_VQ_LEVEL` for OceanBase
   - Alternative: Separate `OCEANBASE_INDEX_TYPE` variable
   - **Question**: Which approach is preferred for consistency?

2. **Distance Function Strategy**
   - FastGPT uses `inner_product` across all vector DBs
   - This limits HNSW_BQ support
   - **Question**: Is there any plan to support configurable distance metrics?

3. **Minimum Version Enforcement**
   - Should FastGPT check OceanBase version at startup?
   - **Question**: How to handle version mismatch gracefully?

4. **Documentation Location**
   - Should HNSW_BQ limitations be documented?
   - **Question**: Where should users be warned about this?

### 9.2 For OceanBase Team (Feature Request)

1. **Inner Product Support for HNSW_BQ**
   - Current: HNSW_BQ only supports `l2` and `cosine`
   - **Request**: Add `inner_product` support for HNSW_BQ
   - **Rationale**: Many vector embedding models (OpenAI, etc.) are optimized for inner product similarity

2. **Docker Image Tags**
   - Current FastGPT uses `oceanbase/oceanbase-ce:4.3.5-lts`
   - **Question**: Are BP-versioned tags available? (e.g., `4.3.5-bp1`, `4.3.5-bp2`)
   - **Question**: What is the recommended tag for latest stable with vector features?

### 9.3 Technical Clarifications Needed

1. **HNSW_SQ Distance Support**
   - Documentation examples show `distance=l2` for HNSW_SQ
   - **Question**: Is `distance=inner_product` confirmed working with HNSW_SQ?
   - Need to test before implementation

2. **Index Rebuild Behavior**
   - When changing from HNSW to HNSW_SQ, does `DROP INDEX` + `CREATE INDEX` work?
   - Or is `REBUILD_INDEX` required?

3. **Memory Calculation**
   - What's the exact memory formula for HNSW_SQ?
   - Is it always 1/2 to 1/3 of HNSW regardless of vector dimensions?


---

## 10. Next Steps

### 10.1 Immediate Actions

- [ ] **Verify Docker Image Availability**
  ```bash
  docker pull oceanbase/oceanbase-ce:4.3.5-bp1
  docker pull oceanbase/oceanbase-ce:4.3.5-bp2
  ```

- [ ] **Test HNSW_SQ with inner_product**
  ```sql
  -- In OceanBase 4.3.5 BP1+
  CREATE TABLE test_vec (id INT PRIMARY KEY, vec VECTOR(1536));
  INSERT INTO test_vec VALUES (1, '[...]');
  
  CREATE VECTOR INDEX test_idx ON test_vec(vec) 
  WITH (distance=inner_product, type=hnsw_sq, m=16, ef_construction=128);
  
  -- Verify search works
  SELECT id, inner_product(vec, '[...]') AS score 
  FROM test_vec 
  ORDER BY score DESC APPROXIMATE LIMIT 10;
  ```

- [ ] **Benchmark Memory Usage**
  - Compare HNSW vs HNSW_SQ memory for 100K, 1M vectors
  - Document actual savings

### 10.2 Implementation Checklist

- [ ] Update `deploy/args.json` with BP1+ version
- [ ] Add `OceanBaseIndexType` constant
- [ ] Modify `oceanbase/index.ts` init function
- [ ] Update `.env.template` documentation
- [ ] Add version check/warning on startup
- [ ] Write unit tests for new behavior
- [ ] Update configuration documentation

### 10.3 Future Considerations

- [ ] Monitor OceanBase releases for HNSW_BQ + inner_product support
- [ ] Consider adding configurable distance metric (long-term)
- [ ] Evaluate IVF_PQ as alternative for extreme memory savings
- [ ] Add `hnswRefineK` to config.json for HNSW_BQ (when supported)

---

## Appendix A: OceanBase Documentation References

### Local Documentation (oceanbase-doc-4.3.5/)

| Topic | Path |
|-------|------|
| Vector Index Creation | `en-US/640.ob-vector-search/200.ob-vector-index.md` |
| Vector Functions | `en-US/640.ob-vector-search/250.ob-vector-function.md` |
| Core Features | `en-US/640.ob-vector-search/100.ob-vector-search-overview/200.ob-vector-search-core-feature.md` |
| BP Version Changes | `zh-CN/0.bp-version-change-record.md` |

### Online Documentation

- [Vector Index Types](https://www.oceanbase.com/docs/common-oceanbase-database-cn-1000000004920599)
- [Index Selection Guide](https://www.oceanbase.com/docs/common-oceanbase-database-cn-1000000004920602)
- [Recall Tuning](https://www.oceanbase.com/docs/common-oceanbase-database-cn-1000000004920603)
- [HNSW Parameters](https://www.oceanbase.com/docs/common-oceanbase-database-cn-1000000003376604)

---

## Appendix B: Glossary

| Term | Definition |
|------|------------|
| **HNSW** | Hierarchical Navigable Small World - graph-based ANN index |
| **HNSW_SQ** | HNSW with Scalar Quantization (8-bit) |
| **HNSW_BQ** | HNSW with Binary Quantization (1-bit, RaBitQ) |
| **RaBitQ** | Randomized Binary Quantization algorithm |
| **IVF** | Inverted File Index - clustering-based ANN index |
| **IVF_PQ** | IVF with Product Quantization |
| **ANN** | Approximate Nearest Neighbor |
| **BP** | Build Patch (OceanBase version suffix) |
| **ef_search** | Size of candidate set during HNSW search |
| **ef_construction** | Size of candidate set during HNSW index building |
| **refine_k** | Reranking factor for quantized indexes |
| **inner_product** | Dot product similarity (higher = more similar) |
| **l2_distance** | Euclidean distance (lower = more similar) |
| **cosine_distance** | Angular distance (lower = more similar) |

---

## Appendix C: Test SQL Scripts

### C.1 Check OceanBase Version
```sql
SELECT version();
-- Example output: 4.3.5.1-101000012025031821
-- The format is: major.minor.patch.bp-buildnumber
```

### C.2 Check Vector Index Support
```sql
SHOW VARIABLES LIKE '%vector%';
```

### C.3 Check Index Type After Creation
```sql
SHOW INDEX FROM modeldata;
-- Look for Index_type = VECTOR
```

### C.4 Estimate Memory Usage
```sql
CALL dbms_vector.index_vector_memory_advisor(
  'HNSW_SQ',      -- index type
  1000000,        -- row count
  1536,           -- dimensions
  'FLOAT32',      -- data type
  'M=32,DISTANCE=INNER_PRODUCT'
);
```

### C.5 Monitor Index Memory
```sql
SELECT * FROM V$OB_VECTOR_MEMORY;
```

---

## Appendix D: Decision Log

- [Appendix E: NEW FINDINGS](#appendix-e-new-findings-updated-2026-01-27)
- [Appendix F: Scope Check Draft for Maintainers](#appendix-f-scope-check-draft-for-maintainers)

| Date | Decision | Rationale |
|------|----------|-----------|
| 2026-01-27 | Recommend HNSW_SQ only for Phase 1 | HNSW_BQ incompatible with inner_product |
| 2026-01-27 | Extend VECTOR_VQ_LEVEL instead of new var | Consistency with existing PgVector pattern |
| 2026-01-27 | Require BP1+ minimum version | HNSW_SQ not available in base 4.3.5-lts |

---

*Document generated: January 27, 2026*  
*Based on: OceanBase 4.3.5 documentation and FastGPT v4.14.5.1 codebase*

---

## Appendix E: NEW FINDINGS (Updated 2026-01-27)

### E.1 Critical Discovery: BP3+ May Support inner_product for HNSW_BQ

**Source**: OceanBase source code analysis (`src/share/vector_index/ob_vector_index_util.cpp`)

```cpp
// From OceanBase source code
if (type_hnsw_bq_is_set && !is_enable_bp_cosine_and_ip && distance_name != "L2") {
  ret = OB_NOT_SUPPORTED;
}
```

The `is_enable_bp_cosine_and_ip` flag controls inner_product/cosine support for HNSW_BQ:

| Version Range | `is_enable_bp_cosine_and_ip` | HNSW_BQ + inner_product |
|---------------|------------------------------|-------------------------|
| < 4.3.5.3 | `false` | ❌ NOT supported |
| [4.3.5.3, 4.4.0.0) | `true` | ✅ **MAY BE supported** |
| [4.4.0.0, 4.4.1.0) | `false` | ❌ NOT supported |
| >= 4.4.1.0 | `true` | ✅ **MAY BE supported** |

**Implication**: This contradicts the documentation! **Version 4.3.5.3 (BP3+) may actually support `inner_product` for HNSW_BQ.**

⚠️ **Action Required**: Verify this finding by testing with OceanBase 4.3.5.3+ before implementation.


### E.2 Docker Image Tag Mapping (Verified)

**Finding**: OceanBase Docker Hub does NOT use explicit BP tags like `4.3.5-bp1`.

**Actual Format**: `4.3.5.X-BUILDNUMBER` where X corresponds to BP version.

| Docker Tag Example | BP Version | Release Date (approx) |
|--------------------|------------|----------------------|
| `4.3.5.1-...` | BP1 | 2025-03-18 |
| `4.3.5.2-...` | BP2 | 2025-05-15 |
| `4.3.5.3-...` | BP3 | 2025-07-21 |
| `4.3.5.4-...` | BP4 | 2025-09-10 |
| `4.3.5.5-105000012025111711` | BP5 | 2025-11-17 |

**Current FastGPT**: Uses `4.3.5-lts` which is the base version (effectively BP0).

**Recommended Update**: 
```json
// deploy/args.json
{
  "tags": {
    "oceanbase": "4.3.5.3"  // For HNSW_SQ + potentially HNSW_BQ with inner_product
  }
}
```


### E.3 Index Rebuild Methods (Confirmed)

**Method 1: DROP + CREATE** (Recommended for index type changes)
```sql
-- Step 1: Drop existing index
DROP INDEX vector_index ON modeldata;

-- Step 2: Create new index with different type
CREATE VECTOR INDEX vector_index ON modeldata(vector) 
WITH (distance=inner_product, type=hnsw_sq, m=32, ef_construction=128);
```

**Method 2: REBUILD_INDEX Procedure** (For index refresh, same type)
```sql
-- Full refresh of existing index
CALL dbms_vector.rebuild_index('vector_index');
```

**Note**: `REBUILD_INDEX` does a full refresh but keeps the same index type. For changing HNSW → HNSW_SQ, use DROP + CREATE.

**Memory Optimization**: Enable memory saving mode during rebuild:
```sql
-- Reduce memory usage during index operations
ALTER SYSTEM SET vector_index_memory_saving_mode = 'ON';
```


---

## Appendix F: Scope Check Draft for Maintainers

### F.1 Summary for Review

**Feature**: Add OceanBase HNSW quantization support (8-bit HNSW_SQ, 1-bit HNSW_BQ) via `VECTOR_VQ_LEVEL` environment variable.

**Current Status**: 
- Neither HNSW_SQ nor HNSW_BQ is implemented in FastGPT for OceanBase
- OceanBase version in deploy/args.json is `4.3.5-lts` (base version, no BP features)
- PgVector already has half-precision (16-bit) support as reference

### F.2 Questions for FastGPT Maintainers

#### Q1: Environment Variable Approach
> **Current Proposal**: Extend `VECTOR_VQ_LEVEL` to support value `8` for OceanBase HNSW_SQ.
> 
> **Alternative**: Create separate `OCEANBASE_INDEX_TYPE` variable.
> 
> **Question**: Which approach aligns better with the project's configuration philosophy?

#### Q2: Minimum Version Enforcement
> **Issue**: HNSW_SQ requires OceanBase 4.3.5.1+ (BP1), HNSW_BQ requires 4.3.5.2+ (BP2).
> 
> **Question**: Should FastGPT:
> - (a) Check OceanBase version at startup and fail if incompatible?
> - (b) Log a warning but continue?
> - (c) Simply document the requirement?

#### Q3: HNSW_BQ Compatibility Concern
> **Issue**: Documentation says HNSW_BQ does NOT support `inner_product`, but source code analysis suggests BP3+ (4.3.5.3+) may support it via `is_enable_bp_cosine_and_ip` flag.
> 
> **Question**: Should we:
> - (a) Only implement HNSW_SQ (8-bit) for now (safe path)?
> - (b) Implement both after verifying BP3+ supports inner_product?
> - (c) Implement HNSW_BQ with L2 distance as optional configuration?

#### Q4: Docker Image Version Update
> **Current**: `oceanbase: "4.3.5-lts"`
> 
> **Required**: At least `4.3.5.1` for HNSW_SQ, `4.3.5.3` recommended for potential HNSW_BQ support.
> 
> **Question**: Is updating the default OceanBase version acceptable?


### F.3 Questions for OceanBase Team

#### Q1: inner_product Support for HNSW_BQ (Clarification Request)
> **Documentation States**: HNSW_BQ only supports `l2` and `cosine` (cosine from BP4+).
> 
> **Source Code Shows**: `is_enable_bp_cosine_and_ip` flag enables inner_product for HNSW_BQ in versions `[4.3.5.3, 4.4.0.0)` and `[4.4.1.0, )`.
> 
> **Question**: Can you confirm whether HNSW_BQ supports `inner_product` distance in version 4.3.5.3+?
> 
> **Context**: FastGPT uses `inner_product` as its distance metric for all vector databases. If HNSW_BQ supports inner_product, we can implement 1-bit quantization. Otherwise, it's a blocker.

#### Q2: Docker Image Tag Convention
> **Observation**: Docker Hub shows tags like `4.3.5.5-105000012025111711` but no explicit `-bp1`, `-bp2` tags.
> 
> **Question**: Is the convention `4.3.5.X` where X = BP version number?
> 
> **Question**: What tag should we use for production deployments requiring BP1+ features?

#### Q3: HNSW_SQ + inner_product Confirmation
> **Question**: Can you confirm that HNSW_SQ fully supports `inner_product` distance metric?
> 
> **Use Case**: We plan to use `CREATE VECTOR INDEX ... WITH (distance=inner_product, type=hnsw_sq, ...)` and `SELECT inner_product(vector, ...) ... ORDER BY score DESC`.

### F.4 Verification Test Plan

Before implementation, run these tests on OceanBase 4.3.5.3+:

```sql
-- Test 1: Verify HNSW_SQ with inner_product
CREATE TABLE test_sq (id INT PRIMARY KEY, vec VECTOR(128));
INSERT INTO test_sq VALUES (1, ARRAY_TO_VECTOR('[1.0, 2.0, ...]'));
CREATE VECTOR INDEX idx_sq ON test_sq(vec) 
  WITH (distance=inner_product, type=hnsw_sq);
SELECT id, inner_product(vec, '[...]') AS score FROM test_sq ORDER BY score DESC;
-- Expected: SUCCESS

-- Test 2: Verify HNSW_BQ with inner_product (the key question)
CREATE TABLE test_bq (id INT PRIMARY KEY, vec VECTOR(128));
INSERT INTO test_bq VALUES (1, ARRAY_TO_VECTOR('[1.0, 2.0, ...]'));
CREATE VECTOR INDEX idx_bq ON test_bq(vec) 
  WITH (distance=inner_product, type=hnsw_bq);
-- Expected: SUCCESS if BP3+ supports it, OB_NOT_SUPPORTED if not
```


### F.5 Proposed Implementation Scope

Based on the analysis, here is the proposed scope:

| Scope Item | Phase 1 (Safe) | Phase 2 (If Verified) |
|------------|----------------|----------------------|
| HNSW_SQ (8-bit) | ✅ Implement | ✅ Maintain |
| HNSW_BQ (1-bit) | ❌ Skip | ✅ Implement if BP3+ confirmed |
| VECTOR_VQ_LEVEL=8 | ✅ Enable HNSW_SQ | ✅ Keep for HNSW_SQ |
| VECTOR_VQ_LEVEL=1 | ❌ Not implemented | ✅ Enable HNSW_BQ |
| Min OceanBase version | 4.3.5.1 | 4.3.5.3 |
| Distance metric | inner_product (no change) | inner_product (no change) |

### F.6 Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| HNSW_SQ doesn't work with inner_product | Low | High | Test before implementation |
| BP3+ inner_product support is wrong | Medium | Medium | Only implement HNSW_SQ in Phase 1 |
| Docker image not available | Low | Low | Use specific build tag |
| Performance regression | Low | Medium | Benchmark before release |
| Index migration issues | Medium | Medium | Provide clear migration docs |

---

*Appendices E and F added: January 27, 2026*
*Based on: OceanBase source code analysis and additional research*

---

## Appendix G: Experimental Verification Guide

This guide provides step-by-step instructions to verify HNSW quantization support on OceanBase, closely following official documentation.

### G.1 Environment Setup

#### Option 1: Using Docker (Recommended for Testing)

```bash
# Step 1: Pull OceanBase images for different BP versions
# Base version (current FastGPT default)
docker pull oceanbase/oceanbase-ce:4.3.5-lts

# BP3 version (for inner_product + HNSW_BQ testing)
docker pull oceanbase/oceanbase-ce:4.3.5.3

# Latest BP5 version
docker pull oceanbase/oceanbase-ce:4.3.5.5

# Step 2: Start OceanBase container (choose one version)
# For 4.3.5-lts (base):
docker run -d --name ob-test-lts \
  -p 2881:2881 \
  -e MODE=mini \
  -e OB_MEMORY_LIMIT=6G \
  oceanbase/oceanbase-ce:4.3.5-lts

# For 4.3.5.3 (BP3):
docker run -d --name ob-test-bp3 \
  -p 2882:2881 \
  -e MODE=mini \
  -e OB_MEMORY_LIMIT=6G \
  oceanbase/oceanbase-ce:4.3.5.3

# Step 3: Wait for initialization (2-3 minutes)
docker logs -f ob-test-lts  # or ob-test-bp3

# Step 4: Connect to OceanBase
# For 4.3.5-lts:
mysql -h127.0.0.1 -P2881 -uroot -p  # password is empty by default

# For 4.3.5.3:
mysql -h127.0.0.1 -P2882 -uroot -p
```

#### Option 2: Using OBClient

```bash
# Install obclient if not available
# On Ubuntu/Debian:
wget https://github.com/oceanbase/obclient/releases/download/v2.2.4/obclient_2.2.4-1_amd64.deb
sudo dpkg -i obclient_2.2.4-1_amd64.deb

# Connect
obclient -h127.0.0.1 -P2881 -uroot@sys -Doceanbase
```

### G.2 Version Verification

```sql
-- Check OceanBase version
SELECT version();
-- Expected format: 4.3.5.X-BUILDNUMBER
-- X = 0 means base (no BP), X = 1 means BP1, etc.

-- Check cluster status
SELECT * FROM oceanbase.DBA_OB_SERVERS;

-- Verify vector capability is enabled
SHOW VARIABLES LIKE '%vector%';
```

### G.3 Test Database Setup

```sql
-- Create test database
CREATE DATABASE IF NOT EXISTS vector_test;
USE vector_test;

-- Create test table with 1536-dimension vectors (OpenAI embedding size)
CREATE TABLE test_vectors (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    content VARCHAR(500),
    vector VECTOR(1536) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample vectors (use Python script below to generate)
-- For quick test, use smaller dimension:
CREATE TABLE test_vectors_128 (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    vector VECTOR(128) NOT NULL
);
```

### G.4 Generate Test Vectors (Python Script)

```python
#!/usr/bin/env python3
"""
Generate test vectors and insert into OceanBase.
Save as: generate_test_vectors.py
"""

import mysql.connector
import numpy as np
import random

def generate_normalized_vector(dim):
    """Generate a random normalized vector."""
    vec = np.random.randn(dim)
    vec = vec / np.linalg.norm(vec)  # Normalize for inner_product
    return vec.tolist()

def format_vector(vec):
    """Format vector as OceanBase string."""
    return '[' + ','.join(f'{v:.6f}' for v in vec) + ']'

def main():
    # Connect to OceanBase
    conn = mysql.connector.connect(
        host='127.0.0.1',
        port=2881,  # Change to 2882 for BP3 container
        user='root',
        password='',
        database='vector_test'
    )
    cursor = conn.cursor()
    
    # Generate and insert 1000 test vectors
    dim = 128  # Use 128 for quick testing, 1536 for production testing
    batch_size = 100
    total = 1000
    
    print(f"Inserting {total} vectors of dimension {dim}...")
    
    for batch_start in range(0, total, batch_size):
        values = []
        for i in range(batch_start, min(batch_start + batch_size, total)):
            vec = generate_normalized_vector(dim)
            vec_str = format_vector(vec)
            values.append(f"({vec_str})")
        
        sql = f"INSERT INTO test_vectors_128 (vector) VALUES {','.join(values)}"
        cursor.execute(sql)
        conn.commit()
        print(f"Inserted {min(batch_start + batch_size, total)}/{total}")
    
    print("Done!")
    cursor.close()
    conn.close()

if __name__ == '__main__':
    main()
```

**Run the script:**
```bash
pip install mysql-connector-python numpy
python generate_test_vectors.py
```

### G.5 Experiment 1: Test HNSW (Base - Should Work on All Versions)

```sql
USE vector_test;

-- Test 1.1: Create HNSW index with inner_product
CREATE VECTOR INDEX idx_hnsw_ip ON test_vectors_128(vector) 
WITH (distance=inner_product, type=hnsw, m=16, ef_construction=128);
-- Expected: SUCCESS on all versions

-- Verify index creation
SHOW INDEX FROM test_vectors_128;

-- Test 1.2: Search using inner_product
SET ob_hnsw_ef_search = 64;

SELECT id, inner_product(vector, '[0.1,0.2,...]') AS score  -- Use actual 128-dim vector
FROM test_vectors_128
ORDER BY score DESC
APPROXIMATE LIMIT 10;
-- Expected: SUCCESS, returns top 10 similar vectors

-- Test 1.3: Check memory usage
SELECT * FROM V$OB_VECTOR_MEMORY WHERE index_name LIKE '%idx_hnsw%';

-- Clean up
DROP INDEX idx_hnsw_ip ON test_vectors_128;
```

### G.6 Experiment 2: Test HNSW_SQ (Requires BP1+)

```sql
USE vector_test;

-- Test 2.1: Create HNSW_SQ index with inner_product
CREATE VECTOR INDEX idx_hnsw_sq_ip ON test_vectors_128(vector) 
WITH (distance=inner_product, type=hnsw_sq, m=16, ef_construction=128);
-- Expected on 4.3.5-lts: ERROR (feature not available)
-- Expected on 4.3.5.1+: SUCCESS

-- If successful, verify and test search
SHOW INDEX FROM test_vectors_128;

SET ob_hnsw_ef_search = 64;
SELECT id, inner_product(vector, '[0.1,0.2,...]') AS score
FROM test_vectors_128
ORDER BY score DESC
APPROXIMATE LIMIT 10;

-- Compare memory usage with HNSW
SELECT * FROM V$OB_VECTOR_MEMORY WHERE index_name LIKE '%idx_hnsw_sq%';

-- Clean up
DROP INDEX idx_hnsw_sq_ip ON test_vectors_128;
```

### G.7 Experiment 3: Test HNSW_BQ (Requires BP2+)

```sql
USE vector_test;

-- Test 3.1: Create HNSW_BQ with L2 (should work on BP2+)
CREATE VECTOR INDEX idx_hnsw_bq_l2 ON test_vectors_128(vector) 
WITH (distance=l2, type=hnsw_bq, m=16, ef_construction=128);
-- Expected on 4.3.5-lts: ERROR
-- Expected on 4.3.5.2+: SUCCESS

-- Test 3.2: Create HNSW_BQ with inner_product (THE KEY TEST)
CREATE VECTOR INDEX idx_hnsw_bq_ip ON test_vectors_128(vector) 
WITH (distance=inner_product, type=hnsw_bq, m=16, ef_construction=128);
-- Expected on 4.3.5-lts: ERROR
-- Expected on 4.3.5.2: ERROR (OB_NOT_SUPPORTED)
-- Expected on 4.3.5.3+: ???  <-- This is what we need to verify!

-- If 3.2 succeeds, test search
SET ob_hnsw_ef_search = 64;
SELECT id, inner_product(vector, '[0.1,0.2,...]') AS score
FROM test_vectors_128
ORDER BY score DESC
APPROXIMATE LIMIT 10;

-- Check memory savings
SELECT * FROM V$OB_VECTOR_MEMORY;

-- Clean up
DROP INDEX idx_hnsw_bq_l2 ON test_vectors_128;
DROP INDEX idx_hnsw_bq_ip ON test_vectors_128;  -- If created
```

### G.8 Experiment 4: Test HNSW_BQ with refine_k (Requires BP3+)

```sql
USE vector_test;

-- Test with refine_k for better recall
CREATE VECTOR INDEX idx_hnsw_bq_refine ON test_vectors_128(vector) 
WITH (
    distance=l2,  -- or inner_product if BP3+ supports it
    type=hnsw_bq, 
    m=16, 
    ef_construction=128,
    refine_k=4,
    refine_type=sq8
);
-- Expected on < BP3: ERROR (refine_k not recognized)
-- Expected on BP3+: SUCCESS

-- Compare recall with and without refine_k
-- Insert a known vector, then search for it
INSERT INTO test_vectors_128 (vector) VALUES ('[1.0,0,0,0,...]');  -- 128-dim unit vector
SET @query_vec = '[1.0,0,0,0,...]';

-- Search and check if the exact match appears in top results
SELECT id, l2_distance(vector, @query_vec) AS dist
FROM test_vectors_128
ORDER BY dist ASC
APPROXIMATE LIMIT 10;
```

### G.9 Complete Test Script

Save this as `test_oceanbase_hnsw.sql`:

```sql
-- ===========================================
-- OceanBase HNSW Quantization Test Script
-- Run against different OceanBase versions
-- ===========================================

-- Setup
CREATE DATABASE IF NOT EXISTS vector_test;
USE vector_test;

DROP TABLE IF EXISTS test_results;
CREATE TABLE test_results (
    test_name VARCHAR(100),
    ob_version VARCHAR(50),
    result VARCHAR(20),
    error_msg TEXT,
    tested_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

DROP TABLE IF EXISTS test_vec;
CREATE TABLE test_vec (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    vector VECTOR(128) NOT NULL
);

-- Insert test data (simple vectors for quick test)
INSERT INTO test_vec (vector) VALUES 
    ('[1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]'),
    ('[0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]'),
    ('[0,0,1,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]');

-- Get version
SELECT @ob_version := version();
SELECT CONCAT('Testing on OceanBase version: ', @ob_version) AS info;

-- ===========================================
-- TEST 1: HNSW with inner_product (baseline)
-- ===========================================
SELECT 'TEST 1: HNSW + inner_product' AS test;

DROP INDEX IF EXISTS idx_test ON test_vec;
CREATE VECTOR INDEX idx_test ON test_vec(vector) 
WITH (distance=inner_product, type=hnsw, m=16, ef_construction=64);

INSERT INTO test_results (test_name, ob_version, result) 
VALUES ('HNSW + inner_product', @ob_version, 'SUCCESS');

DROP INDEX idx_test ON test_vec;

-- ===========================================
-- TEST 2: HNSW_SQ with inner_product
-- ===========================================
SELECT 'TEST 2: HNSW_SQ + inner_product' AS test;

-- This may fail on base 4.3.5-lts
CREATE VECTOR INDEX idx_test ON test_vec(vector) 
WITH (distance=inner_product, type=hnsw_sq, m=16, ef_construction=64);

-- If we reach here, it succeeded
INSERT INTO test_results (test_name, ob_version, result) 
VALUES ('HNSW_SQ + inner_product', @ob_version, 'SUCCESS');

DROP INDEX idx_test ON test_vec;

-- ===========================================
-- TEST 3: HNSW_BQ with L2
-- ===========================================
SELECT 'TEST 3: HNSW_BQ + L2' AS test;

CREATE VECTOR INDEX idx_test ON test_vec(vector) 
WITH (distance=l2, type=hnsw_bq, m=16, ef_construction=64);

INSERT INTO test_results (test_name, ob_version, result) 
VALUES ('HNSW_BQ + L2', @ob_version, 'SUCCESS');

DROP INDEX idx_test ON test_vec;

-- ===========================================
-- TEST 4: HNSW_BQ with inner_product (KEY TEST)
-- ===========================================
SELECT 'TEST 4: HNSW_BQ + inner_product (KEY TEST)' AS test;

CREATE VECTOR INDEX idx_test ON test_vec(vector) 
WITH (distance=inner_product, type=hnsw_bq, m=16, ef_construction=64);

-- If we reach here, BP3+ supports inner_product for HNSW_BQ!
INSERT INTO test_results (test_name, ob_version, result) 
VALUES ('HNSW_BQ + inner_product', @ob_version, 'SUCCESS');

DROP INDEX idx_test ON test_vec;

-- ===========================================
-- TEST 5: HNSW_BQ with refine_k
-- ===========================================
SELECT 'TEST 5: HNSW_BQ + refine_k' AS test;

CREATE VECTOR INDEX idx_test ON test_vec(vector) 
WITH (distance=l2, type=hnsw_bq, m=16, ef_construction=64, refine_k=4);

INSERT INTO test_results (test_name, ob_version, result) 
VALUES ('HNSW_BQ + refine_k', @ob_version, 'SUCCESS');

DROP INDEX idx_test ON test_vec;

-- ===========================================
-- Show Results
-- ===========================================
SELECT '========== TEST RESULTS ==========' AS separator;
SELECT * FROM test_results ORDER BY tested_at;

-- Clean up
DROP TABLE test_vec;
```

### G.10 Running the Experiments

```bash
# Step 1: Start both containers
docker run -d --name ob-lts -p 2881:2881 -e MODE=mini -e OB_MEMORY_LIMIT=6G oceanbase/oceanbase-ce:4.3.5-lts
docker run -d --name ob-bp3 -p 2882:2881 -e MODE=mini -e OB_MEMORY_LIMIT=6G oceanbase/oceanbase-ce:4.3.5.3

# Step 2: Wait for initialization
sleep 180  # 3 minutes

# Step 3: Run tests on base version
mysql -h127.0.0.1 -P2881 -uroot < test_oceanbase_hnsw.sql 2>&1 | tee results_lts.txt

# Step 4: Run tests on BP3 version  
mysql -h127.0.0.1 -P2882 -uroot < test_oceanbase_hnsw.sql 2>&1 | tee results_bp3.txt

# Step 5: Compare results
echo "=== 4.3.5-lts Results ===" && cat results_lts.txt
echo ""
echo "=== 4.3.5.3 (BP3) Results ===" && cat results_bp3.txt

# Step 6: Clean up containers when done
docker stop ob-lts ob-bp3
docker rm ob-lts ob-bp3
```

### G.11 Expected Results Matrix

| Test | 4.3.5-lts (BP0) | 4.3.5.1 (BP1) | 4.3.5.2 (BP2) | 4.3.5.3 (BP3) |
|------|-----------------|---------------|---------------|---------------|
| HNSW + inner_product | ✅ SUCCESS | ✅ SUCCESS | ✅ SUCCESS | ✅ SUCCESS |
| HNSW_SQ + inner_product | ❌ ERROR | ✅ SUCCESS | ✅ SUCCESS | ✅ SUCCESS |
| HNSW_BQ + L2 | ❌ ERROR | ❌ ERROR | ✅ SUCCESS | ✅ SUCCESS |
| HNSW_BQ + inner_product | ❌ ERROR | ❌ ERROR | ❌ ERROR | ❓ **VERIFY** |
| HNSW_BQ + refine_k | ❌ ERROR | ❌ ERROR | ❌ ERROR | ✅ SUCCESS |

### G.12 Evidence Collection Template

After running experiments, document results using this template:

```markdown
## Experiment Report

**Date**: YYYY-MM-DD
**Tester**: [Name]
**OceanBase Version**: [e.g., 4.3.5.3-xxx]

### Environment
- Docker image: oceanbase/oceanbase-ce:X.X.X
- Host OS: [e.g., Ubuntu 22.04]
- Memory allocated: 6GB

### Results

| Test Case | Expected | Actual | Notes |
|-----------|----------|--------|-------|
| HNSW + IP | SUCCESS | | |
| HNSW_SQ + IP | SUCCESS | | |
| HNSW_BQ + L2 | SUCCESS | | |
| HNSW_BQ + IP | ??? | | **KEY FINDING** |
| HNSW_BQ + refine_k | SUCCESS | | |

### Error Messages (if any)
```
[Paste error messages here]
```

### Memory Usage Comparison
| Index Type | Memory (MB) | Compression Ratio |
|------------|-------------|-------------------|
| HNSW | | 1.0x |
| HNSW_SQ | | |
| HNSW_BQ | | |

### Conclusions
- [ ] HNSW_SQ supports inner_product: YES / NO
- [ ] HNSW_BQ supports inner_product on BP3+: YES / NO
- [ ] Recommended OceanBase version for FastGPT: X.X.X
```

---

*Appendix G added: January 27, 2026*
*Based on OceanBase official documentation and Docker Hub*
