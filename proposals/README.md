# ODP Enhancement Proposals

Discussion documents capturing observations, questions, and potential improvements discovered while exploring ODP.

## Proposals

| Proposal | Summary |
|----------|---------|
| [data_discoverability_gap.md](data_discoverability_gap.md) | STAC pagination (3,774 items, 10/page default) and access type indicators |
| [sdk_batch_metadata.md](sdk_batch_metadata.md) | Batch count/indexed access for Dask integration |
| [server_side_h3_aggregation.md](server_side_h3_aggregation.md) | H3 aggregation returns indices vs hex strings |
| [my_data_api.md](my_data_api.md) | Programmatic dataset creation workflow |

## Approach

These proposals are written from the perspective of learning through exploration. They:
- Document observed behavior vs expected behavior
- Ask questions rather than make demands
- Propose options for discussion
- Reference industry patterns (CNCF, Kafka, Iceberg) where relevant

## Related

- `tutorials/` - Notebooks that discovered these patterns
- `.claude/skills/` - Codified workarounds and best practices
