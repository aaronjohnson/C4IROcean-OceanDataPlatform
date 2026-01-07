# Proposal: SDK Batch Metadata for Dask Integration

**Status:** Discussion
**Author:** Aaron Johnson
**Date:** 2026-01-06
**Related:** `tutorials/drafts/05_dask_distributed_processing.ipynb`, `.claude/skills/dask-odp-patterns/`

## Summary

The ODP SDK's `.batches()` iterator doesn't expose batch count or allow indexed access, making optimal Dask integration difficult. This proposal explores options to enable lazy, parallel loading of ODP data into Dask DataFrames.

## Current Behavior

```python
# Current: must materialize iterator to know batch count
batches = list(dataset.table.select().batches())  # Blocks, loads all batch metadata
n_batches = len(batches)

# Then create delayed objects
delayed_dfs = [delayed(lambda b: b.to_pandas())(batch) for batch in batches]
ddf = dd.from_delayed(delayed_dfs, meta=meta)
```

**Issues:**
1. `list()` call blocks and may load batch metadata for all batches upfront
2. No way to know batch count without iterating
3. Cannot lazily create Dask partition graph without knowing partition count

## Desired Behavior

```python
# Ideal: query batch count without loading batches
select = dataset.table.select(filter_expr)
n_batches = select.batch_count()  # Quick metadata query

# Then create truly lazy delayed objects
@delayed
def load_batch(dataset_id, filter_expr, batch_idx):
    odp = ODPClient()
    ds = odp.dataset(dataset_id)
    return ds.table.select(filter_expr).batch(batch_idx).to_pandas()

partitions = [load_batch(dataset_id, filter_expr, i) for i in range(n_batches)]
ddf = dd.from_delayed(partitions, meta=meta)
```

## Questions for Discussion

1. **Does the backend know batch count?** Is batch count determined by:
   - Server-side chunking (predictable count)?
   - Data size / row count (calculable)?
   - Dynamic streaming (unknown until exhausted)?

2. **Would indexed batch access be feasible?** Could the API support:
   ```python
   batch = select.batch(index)  # Get specific batch by index
   ```

3. **Alternative: batch metadata in first response?**
   ```python
   select = dataset.table.select()
   print(select.estimated_batches)  # Approximate count from first response
   ```

4. **Is there a query API we're missing?** Perhaps:
   ```python
   stats = dataset.table.stats()
   stats.estimated_batch_count  # If row count known, can estimate
   ```

## Proposed Solutions

### Option A: Batch Count Method

Add `.batch_count()` to selection object:

```python
select = dataset.table.select(filter_expr)
n_batches = select.batch_count()  # Returns int

# Implementation could:
# - Query server for partition metadata
# - Calculate from row count / batch size
# - Return estimate with actual count in iteration
```

### Option B: Indexed Batch Access

Add `.batch(index)` method:

```python
select = dataset.table.select()
batch_0 = select.batch(0)  # First batch
batch_5 = select.batch(5)  # Sixth batch (random access)

# Enables parallel loading without iterator
```

### Option C: Iterator with Metadata

Return rich iterator with metadata:

```python
batch_iter = dataset.table.select().batches()
print(batch_iter.total_batches)   # Known if server provides
print(batch_iter.estimated_rows)  # Per batch estimate

for batch in batch_iter:
    print(f"Batch {batch.index} of {batch_iter.total_batches}")
```

### Option D: Parallel Fetch Helper

SDK provides Dask-aware helper:

```python
# SDK handles parallelization internally
ddf = dataset.table.to_dask(
    filter_expr=None,
    npartitions=10  # Hint for parallelism
)
```

### Option E: Document Current Best Practice

If batch metadata isn't feasible, document the recommended pattern:

```python
# Best practice: materialize batch list once, then parallelize
batches = list(dataset.table.select().batches())

from dask import delayed
delayed_dfs = [delayed(lambda b: b.to_pandas())(b) for b in batches]
ddf = dd.from_delayed(delayed_dfs, meta=batches[0].to_pandas().head(0))
```

## Impact

**Without enhancement:**
- Must call `list()` on batches iterator before creating Dask graph
- Memory overhead from holding batch references
- Not fully lazy - batch metadata loaded upfront

**With enhancement:**
- Truly lazy Dask integration
- Better memory efficiency for large datasets
- Cleaner code pattern

## Use Case: Large Dataset Processing

```python
# Processing 100M row dataset with 1000 batches
# Current: list(batches) may be slow/memory-intensive
# Proposed: batch_count() returns quickly, enables lazy graph construction

n_batches = dataset.table.select().batch_count()

@delayed
def process_batch(idx):
    batch = dataset.table.select().batch(idx)
    df = batch.to_pandas()
    return df.groupby('station').mean()

results = [process_batch(i) for i in range(n_batches)]
final = dd.from_delayed(results).compute()
```

## Related

- `.claude/skills/dask-odp-patterns/` - Dask integration patterns
- `tutorials/drafts/05_dask_distributed_processing.ipynb` - Dask tutorial
- `proposals/data_discoverability_gap.md` - Related SDK observations

## Industry Context

### CNCF Batch System Initiative

The [CNCF TAG Runtime Batch System Initiative](https://tag-runtime.cncf.io/wgs/bsi/charter/) recognizes that batch workload management requires a holistic approach:

> "While existing batch scheduling systems effectively address scheduling concerns, batch workload management requires a holistic approach that considers the entire system, including storage, networking, and cost optimization."

**Relevant principles:**
- Batch metadata (partition count, size estimates) should be queryable upfront
- Parallel processing requires predictable partitioning
- Cloud-native batch systems should enable lazy graph construction

The initiative is working toward specifications and standards for batch processing in cloud-native environments. ODP's batch API could align with emerging patterns.

### Data Lakehouse Standards

Modern data platforms (Iceberg, Delta Lake, Hudi) expose partition metadata:

```python
# Apache Iceberg - partition metadata upfront
table = catalog.load_table("db.table")
snapshot = table.current_snapshot()
manifests = snapshot.manifests  # Partition count known
total_records = snapshot.summary['total-records']

# Each manifest can be processed independently
for manifest in manifests:
    reader = manifest.fetch()
```

This enables distributed frameworks (Spark, Dask, Ray) to construct execution graphs without scanning data.

## Patterns from Other Systems

### Kafka

Kafka provides partition metadata upfront, enabling parallel consumer design:

```python
from kafka import KafkaConsumer

consumer = KafkaConsumer(bootstrap_servers='...')

# Partition count known without consuming
partitions = consumer.partitions_for_topic('my-topic')
n_partitions = len(partitions)

# Can seek to specific partition/offset
consumer.assign([TopicPartition('my-topic', i) for i in range(n_partitions)])
consumer.seek(TopicPartition('my-topic', 5), offset=0)  # Jump to partition 5
```

**Relevant patterns:**
- `partitions_for_topic()` returns count without consuming data
- `seek()` enables random access to specific partitions
- Partition count is stable for a topic (doesn't change during consumption)

### ClickHouse

ClickHouse exposes partition metadata via system tables:

```sql
-- Query partition count and row estimates without scanning data
SELECT
    partition,
    sum(rows) as rows,
    count() as parts
FROM system.parts
WHERE table = 'my_table' AND active
GROUP BY partition;

-- Estimated row count (fast, from metadata)
SELECT count() FROM my_table SETTINGS optimize_count_from_files = 1;
```

**Relevant patterns:**
- `system.parts` exposes partition metadata
- Row count estimates available without full scan
- `LIMIT BY` enables chunked processing with predictable sizing

### Arrow Flight

Arrow Flight (which ODP may use internally) has similar patterns:

```python
# Flight client can get schema without fetching data
flight_info = client.get_flight_info(descriptor)
total_records = flight_info.total_records  # Known upfront
total_bytes = flight_info.total_bytes

# Endpoints describe available partitions
for endpoint in flight_info.endpoints:
    # Each endpoint can be fetched independently
    reader = client.do_get(endpoint.ticket)
```

**Relevant patterns:**
- `FlightInfo` provides `total_records` before fetching
- Multiple endpoints enable parallel fetching
- Schema available without data transfer

### Implications for ODP

These systems share common patterns:
1. **Partition/batch count known upfront** (not discovered during iteration)
2. **Random access** to specific partitions (not just sequential iteration)
3. **Metadata queries are cheap** (don't require data scanning)

If ODP's backend uses Arrow Flight or similar, this metadata may already be available internally - the question is whether to expose it in the SDK.

## References

- [Dask from_delayed](https://docs.dask.org/en/stable/generated/dask.dataframe.from_delayed.html)
- [Dask Best Practices - Load Data](https://docs.dask.org/en/stable/best-practices.html#load-data-with-dask)
- [ODP Python SDK](https://docs.hubocean.earth/python_sdk/intro/)
- [Kafka Consumer API](https://kafka-python.readthedocs.io/en/master/apidoc/KafkaConsumer.html)
- [ClickHouse system.parts](https://clickhouse.com/docs/en/operations/system-tables/parts)
- [Arrow Flight RPC](https://arrow.apache.org/docs/format/Flight.html)
- [CNCF Batch System Initiative](https://tag-runtime.cncf.io/wgs/bsi/charter/)
- [Apache Iceberg Spec](https://iceberg.apache.org/spec/)
