# OceanBase HNSW Quantization Implementation Plan

> **Status**: Ready for Implementation (HNSW_SQ only)  
> **Date**: January 28, 2026  
> **Blocker**: HNSW_BQ does NOT support `inner_product` distance

---

## 0. Draft Scope Check Questions for Maintainers

您好 @c121914yu，有几个技术细节想向您确认：

---

### Q1: HNSW_BQ（1-bit 量化）的距离算法限制

**问题背景**：OceanBase 的 HNSW_BQ 索引仅支持 `l2` 和 `cosine` 距离算法，**不支持 `inner_product`**。

**官方文档参考**（[创建向量索引](https://www.oceanbase.com/docs/common-oceanbase-database-cn-1000000004920603)）：
> "HNSW_BQ 索引 `distance` 参数支持 l2 和 cosine。cosine 从 V4.3.5 BP4 版本开始支持。"

**当前 FastGPT 实现**：
- 索引创建：使用 `distance=inner_product`（[index.ts#L24](packages/service/common/vectorDB/oceanbase/index.ts#L24)）
- 查询排序：使用 `ORDER BY score DESC`（[index.ts#L138-L144](packages/service/common/vectorDB/oceanbase/index.ts#L138-L144)）

**问题**：如需支持 HNSW_BQ，需要改用 `cosine` 或 `l2` 距离，且需调整排序方向：
- `cosine_distance`：值越小越相似，需 `ORDER BY score ASC`
- 或使用 `cosine_similarity` 函数，保持 `ORDER BY score DESC`

**确认选项**：
- **(A)** 本期仅实现 HNSW_SQ（8-bit），暂不支持 HNSW_BQ（1-bit）？ ← **推荐**
- **(B)** 实现 HNSW_BQ 时改用 `cosine` 距离（需同步修改查询逻辑）？
- **(C)** 等待 OceanBase 支持 HNSW_BQ + `inner_product`？

---

### Q2: 环境变量命名规范

**当前 PGVector 实现**（[constants.ts#L9-L25](packages/service/common/vectorDB/constants.ts#L9-L25)）：
```typescript
export const VectorVQ = (() => {
  if (process.env.VECTOR_VQ_LEVEL === '32') return 32;  // 全精度
  if (process.env.VECTOR_VQ_LEVEL === '16') return 16;  // 半精度 (HALFVEC)
  // ...
})();
```

**建议 OceanBase 复用 `VECTOR_VQ_LEVEL`**，映射关系如下：

| `VECTOR_VQ_LEVEL` | PGVector 行为 | OceanBase 行为 |
|-------------------|--------------|----------------|
| `32`（默认） | `VECTOR`（全精度） | `type=hnsw`（全精度） |
| `16` | `HALFVEC`（半精度） | `type=hnsw`（无变化，OB 无半精度） |
| `8` | — | `type=hnsw_sq`（8-bit 标量量化） |
| `1` | — | `type=hnsw_bq`（1-bit，**受 Q1 阻塞**） |

**确认**：是否接受此命名方式？或需要单独的 OceanBase 环境变量（如 `OCEANBASE_INDEX_TYPE`）？

---

### Q3: HNSW 参数配置

**OceanBase 官方最佳实践**（[索引类型选择](https://www.oceanbase.com/docs/common-oceanbase-database-cn-1000000004920602)）针对不同数据规模给出推荐参数：

| 数据规模 | 推荐参数 |
|---------|---------|
| 百万级 | `m=16, ef_construction=200, ef_search=100` |
| 千万级 | `m=32, ef_construction=400, ef_search=350`（HNSW_SQ） |
| 亿级 | 使用分区表，`m=32, ef_construction=400, ef_search=1000, refine_k=10` |

**当前 FastGPT 默认值**：`m=32, ef_construction=128`（与 PGVector 保持一致，[index.ts#L24](packages/service/common/vectorDB/oceanbase/index.ts#L24)）

**确认**：
- **(A)** 保持当前默认值（`m=32, ef_construction=128`），与 PGVector 一致？
- **(B)** 提供额外环境变量供用户配置（如 `OCEANBASE_HNSW_M`、`OCEANBASE_HNSW_EF_CONSTRUCTION`）？

**备注**：`ef_search` 已可通过 `global.systemEnv?.hnswEfSearch` 配置。

---

### Q4: 现有数据库迁移方案

**技术限制**：根据 OceanBase 文档（[创建向量索引 - 维护](https://www.oceanbase.com/docs/common-oceanbase-database-cn-1000000004920603)）：
- `DBMS_VECTOR.REBUILD_INDEX` **支持** `hnsw` ↔ `hnsw_sq` 转换
- `DBMS_VECTOR.REBUILD_INDEX` **不支持** `hnsw` ↔ `hnsw_bq` 转换（需 `DROP INDEX` + `CREATE INDEX`）

**当前代码**使用 `CREATE VECTOR INDEX IF NOT EXISTS`（[index.ts#L24](packages/service/common/vectorDB/oceanbase/index.ts#L24)），如果索引已存在则不会重建。

**确认**：对于已有 OceanBase 部署的用户切换索引类型时：
- **(A)** 仅提供文档说明，用户需手动执行 `DROP INDEX vector_index` 后重启服务？
- **(B)** 提供迁移脚本或 API 接口？
- **(C)** 此功能仅面向新部署，不提供迁移支持？

---

## Original Scope Questions (English Version)

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
