# FastGPT Oceanbase HNSW Quantization Feature - Knowledge Base

## Project Overview

FastGPT is a knowledge base Q&A system built on LLMs, supporting RAG (Retrieval Augmented Generation) with multiple vector database backends.

**Current Version**: v4.14.5.1
**Repository**: https://github.com/labring/FastGPT

## Vector Database Architecture

### Supported Vector Databases (Priority Order)

1. **PostgreSQL with pgvector** (PG_URL)
2. **OceanBase** (OCEANBASE_URL) 
3. **Milvus** (MILVUS_ADDRESS)

The system selects the vector database based on which environment variable is set first (see `packages/service/common/vectorDB/controller.ts`).

### Current Dependency Versions

From `deploy/args.json`:
- **OceanBase**: `4.3.5-lts` (oceanbase/oceanbase-ce)
- **PgVector**: `0.8.0-pg15` (pgvector/pgvector)
- **Milvus**: `v2.4.3` (milvusdb/milvus)

## Vector Quantization Support

### Current Implementation

#### PgVector Quantization (VECTOR_VQ_LEVEL)

**Environment Variable**: `VECTOR_VQ_LEVEL`

**Supported Values** (from `packages/service/common/vectorDB/constants.ts`):
- `32` - Full precision (default, VECTOR type)
- `16` - Half precision (HALFVEC type)
- `8`, `4`, `2` - Defined but not implemented

**Implementation Details**:
```typescript
// packages/service/common/vectorDB/pg/index.ts
const isHalfVec = VectorVQ === 16;

// Table creation uses different types based on precision
vector ${isHalfVec ? 'HALFVEC(1536)' : 'VECTOR(1536)'} NOT NULL

// Index creation uses different operators
USING hnsw (vector ${isHalfVec ? 'halfvec_ip_ops' : 'vector_ip_ops'})
```

**Configuration Location**: `projects/app/.env.template`
```bash
VECTOR_VQ_LEVEL=16 # 向量量化等级（目前支持 PG：32,16， 其他数据库未支持）
```

#### OceanBase Current State

**No quantization support currently implemented**. OceanBase uses fixed full precision:

```typescript
// packages/service/common/vectorDB/oceanbase/index.ts
CREATE TABLE IF NOT EXISTS ${DatasetVectorTableName} (
    vector VECTOR(1536) NOT NULL,  // Always full precision
    ...
);

CREATE VECTOR INDEX IF NOT EXISTS vector_index 
ON ${DatasetVectorTableName}(vector) 
WITH (distance=inner_product, type=hnsw, m=32, ef_construction=128);
```

## HNSW Parameters

### Current Configuration

Both PG and OceanBase support HNSW search parameters via `config.json`:

**File**: `projects/app/data/config.json`
```json
{
  "systemEnv": {
    "hnswEfSearch": 100,  // Search accuracy parameter (PG & OB)
    "hnswMaxScanTuples": 100000  // PG only
  }
}
```

**Documentation** (`document/content/docs/introduction/development/configuration.mdx`):
> `hnswEfSearch`: 向量搜索参数，仅对 PG 和 OB 生效。越大，搜索越精确，但是速度越慢。设置为100，有99%+精度。

### Implementation

**PgVector**:
```sql
SET LOCAL hnsw.ef_search = ${global.systemEnv?.hnswEfSearch || 100};
SET LOCAL hnsw.max_scan_tuples = ${global.systemEnv?.hnswMaxScanTuples || 100000};
```

**OceanBase**:
```sql
SET ob_hnsw_ef_search = ${global.systemEnv?.hnswEfSearch || 100};
```

## Feature Request Analysis

### Requirement
Enable OceanBase HNSW quantization (8-bit and 1-bit) via environment variable, similar to PgVector's half-precision configuration.

### OceanBase Official HNSW Support

**Need to verify via MCP**: 
- Which OceanBase version supports HNSW 8-bit and 1-bit quantization?
- Is version `4.3.5-lts` sufficient, or do we need to upgrade?
- Official documentation: https://www.oceanbase.com/docs/

### Key Questions to Research

1. **OceanBase Version Compatibility**
   - Does OceanBase 4.3.5-lts support HNSW quantization?
   - What syntax is used for quantized vectors? (e.g., `VECTOR(1536, 8BIT)`, `VECTOR(1536, 1BIT)`)
   - Are there specific index creation parameters for quantization?

2. **Dependency Updates**
   - If quantization requires a newer OceanBase version, what's the upgrade path?
   - Are there breaking changes in newer versions?
   - Docker image availability for newer versions

3. **Index Type Support**
   - Does OceanBase support other index types besides HNSW? (IVF, FLAT)
   - Current implementation only uses HNSW (`type=hnsw`)
   - No evidence of IVF or FLAT support in FastGPT codebase

4. **Performance Implications**
   - Memory usage differences between full/8-bit/1-bit precision
   - Search accuracy trade-offs
   - Index build time differences

## Implementation Considerations

### Environment Variable Design

Following PgVector's pattern, could use:
- Extend existing `VECTOR_VQ_LEVEL` to support OceanBase
- Or create new `OCEANBASE_VECTOR_PRECISION` variable

**Proposed values**:
- `32` - Full precision (default, current behavior)
- `8` - 8-bit quantization
- `1` - 1-bit quantization

### Code Changes Required

1. **Environment Variable Handling**
   - Update `packages/service/type/env.d.ts`
   - Update `packages/service/common/vectorDB/constants.ts`

2. **OceanBase Vector Controller**
   - Modify `packages/service/common/vectorDB/oceanbase/index.ts`
   - Update table creation SQL based on precision
   - Update index creation parameters if needed

3. **Configuration Documentation**
   - Update `projects/app/.env.template`
   - Update `document/content/docs/introduction/development/configuration.mdx`

4. **Docker Compose Files**
   - Potentially update OceanBase version in:
     - `deploy/docker/global/docker-compose.oceanbase.yml`
     - `deploy/docker/cn/docker-compose.oceanbase.yml`
     - `deploy/args.json`

### Backward Compatibility

- Default to full precision (32-bit) to maintain current behavior
- Existing deployments should work without changes
- Migration path for existing vector data if precision changes

## Related Files

### Core Implementation
- `packages/service/common/vectorDB/oceanbase/index.ts` - OceanBase vector operations
- `packages/service/common/vectorDB/oceanbase/controller.ts` - OceanBase client
- `packages/service/common/vectorDB/pg/index.ts` - PgVector reference implementation
- `packages/service/common/vectorDB/constants.ts` - Vector quantization constants
- `packages/service/common/vectorDB/controller.ts` - Vector DB selection logic

### Configuration
- `projects/app/.env.template` - Environment variables template
- `projects/app/data/config.json` - System configuration
- `packages/service/type/env.d.ts` - Environment variable types

### Deployment
- `deploy/docker/global/docker-compose.oceanbase.yml` - Global OceanBase deployment
- `deploy/docker/cn/docker-compose.oceanbase.yml` - China region OceanBase deployment
- `deploy/args.json` - Version tags and image references

### Documentation
- `document/content/docs/introduction/development/configuration.mdx` - Config docs
- `document/content/docs/introduction/development/intro.mdx` - Development guide
- `document/content/docs/introduction/development/docker.mdx` - Docker deployment guide

## Next Steps

1. **Research OceanBase Documentation** (via MCP)
   - Verify HNSW quantization support in version 4.3.5-lts
   - Find SQL syntax for quantized vectors
   - Check for any required system parameters

2. **Test Current OceanBase Setup**
   - Deploy local OceanBase 4.3.5-lts
   - Test vector operations
   - Verify HNSW index behavior

3. **Design Implementation**
   - Define environment variable schema
   - Plan SQL changes for different precision levels
   - Consider migration strategy

4. **Validate No Side Effects**
   - Confirm IVF/FLAT indexes are not used (they aren't in current code)
   - Ensure PgVector quantization remains unaffected
   - Test Milvus compatibility (no changes expected)

## Notes

- FastGPT uses a monorepo structure with pnpm workspaces
- Vector database selection is mutually exclusive (only one can be active)
- HNSW is the only index type currently used in FastGPT
- No evidence of IVF or FLAT index support in the codebase
- PgVector already has working half-precision (16-bit) implementation as reference
