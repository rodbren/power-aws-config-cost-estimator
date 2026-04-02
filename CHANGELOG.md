# Changelog

## v2.2.0 (2026-04-02)
- **Optimization starts from Config data, not CloudTrail**: Lists existing Config aggregators, asks user to pick one, pulls top contributors from there
- **Top 10 resource types**: Only the top 10 CI-contributing resource types get CloudTrail deep-dive (not all 130+)
- **Per-resource-ID detail for periodic candidates**: Shows individual resource IDs with daily change counts
- **Account/region breakdown mandatory**: Every report must show per-account, per-region cost breakdown with % of total
- **Security Hub CSPM**: Renamed service references for clarity

## v2.1.1 (2026-04-02)
- Renamed service references from "AWS Security Hub" to "AWS Security Hub CSPM" for clarity per official naming
- Config rule prefixes (`securityhub-*`) unchanged — only service name references updated

## v2.1.0 (2026-04-02)
- **Account/region breakdown**: Optimization report now shows top CI contributors per account and region with % of total spend
- **Full AWS:: resource type names**: All outputs use `AWS::EC2::Instance`, `AWS::S3::Bucket`, etc. instead of generic service names
- **Indirect relationships**: Full table of all 10 indirect relationship pairs, extra CIs generated, dependent services, and step-by-step AWS Support case process to disable per account
- **Support case guidance**: How to disable indirect relationships via Technical support case (individual or bulk from management account with CSV)

## v2.0.0 (2026-04-02)
- **NEW: Cost Optimization workflow** — analyze existing Config deployments for savings
- **CloudTrail change frequency analysis**: Query org trail per resource type, calculate daily change ratio, apply 4× rule for continuous-vs-periodic recommendations with actual dollar savings
- **Service dependency matrix**: Security Hub, Control Tower, Firewall Manager, Backup Audit Manager, SSM Compliance, Audit Manager, Trusted Advisor — checked before every recommendation
- **Duplicate rule detection**: Identify overlapping rules from conformance packs, Security Hub, Control Tower
- **Control Tower workaround**: Reference to aws-control-tower-config-customization solution with deployment parameters
- **ResourceCompliance stale risk**: Document impact of excluding ResourceCompliance on detective rules, with CloudTrail query mitigation
- **Delegated admin support**: Can run from management account, Config delegated admin, or CloudTrail delegated admin with capability matrix
- **Resource exclusion guidance**: Safe vs risky exclusions with pre-exclusion checklist
- Power renamed to "AWS Config Cost Estimator & Optimizer"
- Added optimization-related keywords for activation

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
