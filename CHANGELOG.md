# Changelog

## v1.1.0 (2026-04-02)
- **Periodic recording**: Now counts unique resource IDs (not total events) for accurate per-resource-per-day estimation
- **Periodic forecasting**: Daily → weekly (×7) → monthly (×30) projections
- **User-selectable sample day**: Users can pick which day to use for periodic analysis (default: yesterday)
- **Forecasting disclaimer**: Always shown for periodic projections
- **Separate output tables**: Continuous and periodic results displayed independently

## v1.0.1 (2026-03-31)
- Fixed MCP server package names (`awslabs.core-mcp-server`, `awslabs.bedrock-kb-retrieval-mcp-server`)
- Removed `AWS_PROFILE`/`AWS_REGION` env var dependency from `mcp.json`

## v1.0.0 (2026-03-31)
- Initial release
- 130+ CloudTrail eventSource → AWS Config resource type mappings
- Continuous recording cost estimation ($0.003/CI)
- Periodic recording cost estimation ($0.012/CI)
- Default 7-day analysis window with configurable range (1–90 days)
- MCP servers: aws-documentation, aws-core, aws-knowledge
