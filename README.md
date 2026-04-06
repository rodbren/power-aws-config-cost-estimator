# AWS Config Cost Estimator & Optimizer — Kiro Power

Estimate AWS Config recorder costs **before enabling it** and optimize **existing** Config deployments by analyzing CloudTrail organization trail data, service dependencies, duplicate rules, and recording frequency.

## How It Works

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          Kiro IDE / Power                                │
│                                                                          │
│  Two modes:                                                              │
│                                                                          │
│  ┌─────────────────────────────┐  ┌────────────────────────────────────┐ │
│  │  COST ESTIMATION            │  │  COST OPTIMIZATION                 │ │
│  │  (Config not yet enabled)   │  │  (Config already running)          │ │
│  │                             │  │                                    │ │
│  │  1. Verify org trail        │  │  1. Analyze current Config spend   │ │
│  │  2. Choose data method:     │  │  2. List Config aggregators →      │ │
│  │     Lake / Athena / Lookup  │  │     user picks one (WAIT)          │ │
│  │  3. Query CloudTrail        │  │  3. Top 10 resource types from     │ │
│  │     (7 days default)        │  │     aggregator (confirm with user) │ │
│  │  4. Map eventSource →       │  │  4. CloudTrail deep-dive ALL top   │ │
│  │     AWS:: resource types    │  │     10 → specific AWS:: types      │ │
│  │  5. Continuous estimate:    │  │  5. Per-resource-ID change freq    │ │
│  │     total events × $0.003   │  │  6. 4× rule per resource ID        │ │
│  │  6. Periodic estimate:      │  │  7. Dependencies + exclusions      │ │
│  │     unique resources/day    │  │  8. Control Tower workaround       │ │
│  │     × 30 × $0.012           │  │                                    │ │
│  └─────────────────────────────┘  └────────────────────────────────────┘ │
│                                                                          │
│  Runs from: Management Account, Config Delegated Admin,                  │
│             or CloudTrail Delegated Admin                                 │
└──────────────────────────────────────────────────────────────────────────┘
```

## Key Difference: Continuous vs Periodic

| | Continuous | Periodic |
|---|---|---|
| **What triggers a CI** | Every resource change | 1 per unique resource per day |
| **Price per CI** | $0.003 | $0.012 |
| **Best for** | Resources that rarely change | Resources that change frequently |
| **Break-even** | — | Must produce ≥75% fewer CIs than continuous |
| **Ephemeral resources (<24h)** | Captured | 0 CIs — visibility gap |
| **Example**: EC2 instance modified 50×/day | 50 CIs = $0.15/day | 1 CI = $0.012/day |

## Multi-Account Data Access

| Method | Multi-Account | Speed | Setup Required |
|---|---|---|---|
| CloudTrail Lake | ✅ All accounts | Fast (single SQL) | Event data store must exist |
| Athena on org trail S3 | ✅ All accounts | Medium | Athena table (agent can create) — S3 bucket typically in log archive account |
| `lookup_events` | ❌ Current account only | Slow (paginated) | None |

## Cost Estimation — Example Output

### Continuous Recording
**Analysis period**: 7 days | **Extrapolation**: ×4.29 (30 ÷ 7 — scales partial data to a full month)

| Account ID | Region | Resource Type | Events (7d) | Est. Monthly CIs | Est. Monthly Cost |
|---|---|---|---|---|---|
| 123456789012 | us-east-1 | `AWS::EC2::Instance` | 1,200 | 5,143 | $15.43 |
| 123456789012 | us-east-1 | `AWS::S3::Bucket` | 350 | 1,500 | $4.50 |
| **TOTAL** | | | **1,550** | **6,643** | **$19.93** |

### Periodic Recording
**Sample day**: 2026-03-31 (user-selected)

> ⚠️ Weekly and monthly projections are based on a single day's data. Re-run on different days for higher accuracy.

| Account ID | Region | Resource Type | Unique Resources (1d) | Est. Weekly CIs | Est. Monthly CIs | Est. Monthly Cost |
|---|---|---|---|---|---|---|
| 123456789012 | us-east-1 | `AWS::EC2::Instance` | 45 | 315 | 1,350 | $16.20 |
| 123456789012 | us-east-1 | `AWS::S3::Bucket` | 20 | 140 | 600 | $7.20 |
| **TOTAL** | | | **65** | **455** | **1,950** | **$23.40** |

## Cost Optimization — Example Output

### Top CI Contributors by Account and Region
| Account ID | Region | Resource Type | Monthly CIs | Monthly Cost | % of Total |
|---|---|---|---|---|---|
| 123456789012 | us-east-1 | `AWS::EC2::SecurityGroup` | 15,000 | $45.00 | 25% |
| 123456789012 | us-east-1 | `AWS::EC2::Instance` | 10,000 | $30.00 | 17% |
| 234567890123 | eu-west-1 | `AWS::EC2::NetworkInterface` | 8,000 | $24.00 | 13% |

### Continuous vs Periodic Analysis (75% CI reduction threshold)
Periodic must produce **≥75% fewer CIs** than continuous to offset the 4× price difference.

| Account ID | Region | Resource Type | Continuous CIs/mo | Periodic CIs/mo | CI Reduction | Recommendation | Est. Monthly Savings |
|---|---|---|---|---|---|---|---|
| 123456789012 | us-east-1 | `AWS::EC2::Subnet` | 6,510 | 60 | 99% | ✅ Periodic | $18.78 |
| 123456789012 | us-east-1 | `AWS::EC2::NetworkInterface` | 4,860 | 150 | 97% | ✅ Periodic | $12.78 |
| 123456789012 | us-east-1 | `AWS::EC2::Volume` | 4,469 | 3,179 | 29% | ❌ Continuous | -$24.71 |
| 123456789012 | us-east-1 | `AWS::S3::Bucket` | 2,400 | 900 | 63% | ⚠️ Borderline | -$3.60 |

### Per-Resource-ID Detail (periodic candidates)
| Account ID | Region | Resource Type | Resource ID | Changes/Day | Periodic Saves? |
|---|---|---|---|---|---|
| 123456789012 | us-east-1 | `AWS::EC2::Subnet` | subnet-06e0134b | 108 | ✅ Yes (108× > 4×) |
| 123456789012 | us-east-1 | `AWS::EC2::Subnet` | subnet-007336a4 | 108 | ✅ Yes (108× > 4×) |
| 123456789012 | us-east-1 | `AWS::EC2::NetworkInterface` | eni-0abc123 | 50 | ✅ Yes (50× > 4×) |

### Indirect Relationships — Hidden CI Multiplier
Changes to some EC2/VPC resources generate **extra CIs** for related resources:

| Change to | Generates extra CIs for |
|---|---|
| `AWS::EC2::Instance` | `AWS::EC2::SecurityGroup`, `AWS::EC2::Subnet`, `AWS::EC2::VPC` |
| `AWS::EC2::SecurityGroup` | `AWS::EC2::VPC` |
| `AWS::EC2::RouteTable` | `AWS::EC2::Instance`, `AWS::EC2::NetworkInterface`, `AWS::EC2::Subnet`, `AWS::EC2::VPNGateway`, `AWS::EC2::VPC` |

Customers can disable indirect relationships **per account** via [AWS Support case](https://docs.aws.amazon.com/config/latest/developerguide/faq.html#config-recording).

### Dependency Impact Matrix
| Recommendation | Security Hub CSPM | Control Tower | Firewall Mgr | Backup Audit Mgr | SSM Compliance | Audit Mgr | Trusted Advisor |
|---|---|---|---|---|---|---|---|
| Exclude ResourceCompliance | ⚠️ Stale | ✅ OK | ✅ OK | ✅ OK | ⚠️ No history | ⚠️ Less evidence | ✅ OK |
| Switch EC2 to periodic | ✅ OK | ✅ OK | ❌ Breaks | ✅ OK | ⚠️ Delayed | ✅ OK | ✅ OK |
| Exclude Backup types | ✅ OK | ✅ OK | ✅ OK | ❌ Breaks | ✅ OK | ⚠️ Less evidence | ✅ OK |

## Service Dependencies Checked

| Service | Why It Matters |
|---|---|
| AWS Security Hub CSPM | Config rules power security checks |
| AWS Control Tower | Blocks direct Config recorder changes — requires [workaround solution](https://github.com/aws-samples/aws-control-tower-config-customization) |
| AWS Firewall Manager | Requires **continuous** recording — periodic breaks it |
| AWS Backup Audit Manager | Requires Config resource tracking for compliance frameworks |
| AWS Systems Manager Compliance | Uses Config for patch/association compliance history |
| AWS Audit Manager | Captures Config evaluations as audit evidence |
| AWS Trusted Advisor | Some checks powered by Config managed rules |
| Indirect Relationships | EC2/VPC resources generate extra CIs — can be disabled per account via [Support case](https://docs.aws.amazon.com/config/latest/developerguide/faq.html#config-recording) |

## Installation

In Kiro IDE → Powers panel → **"Add power from GitHub"** → enter:
```
https://github.com/rodbren/power-aws-config-cost-estimator
```

## Prerequisites

- **CloudTrail Organization Trail** with management events enabled
- AWS credentials with `cloudtrail:LookupEvents`, `cloudtrail:DescribeTrails`, `config:DescribeConfigRules` permissions
- For multi-account forecasting (choose one):
  - **CloudTrail Lake**: Event data store with org events (recommended)
  - **Athena**: Read access to org trail S3 bucket (typically in log archive account) — agent can create the Athena table
  - **`lookup_events`**: No extra setup, but single-account only
- For optimization: access to management account or delegated admin account

## MCP Servers Included

| Server | Purpose |
|---|---|
| `aws-documentation` | Search AWS docs for Config pricing, supported resources, best practices |
| `aws-core` | Execute AWS CLI commands to query CloudTrail and Config |
| `aws-knowledge` | Retrieve AWS knowledge base content for guidance |

## Usage

### Cost Estimation (Config not yet enabled)
- "estimate my AWS Config costs"
- "how much would it cost to enable Config recorder"
- "forecast Config costs for 30 days"

### Cost Optimization (Config already running)
- "optimize my AWS Config costs"
- "find duplicate Config rules"
- "which resource types should I switch to periodic"
- "what can I safely exclude from Config recording"
- "analyze Config cost drivers"

### Adjustable Parameters

| Parameter | Default | Options |
|---|---|---|
| Estimation window | 7 days | 1–90 days |
| Periodic sample day | Yesterday | Any specific day |
| Optimization analysis | 7 days of CloudTrail | 1–90 days |
| Account scope | Org-wide (Lake/Athena) or single (lookup_events) | Specific accounts |

## Project Structure

```
power-aws-config-cost-estimator/
├── POWER.md                              # Metadata, onboarding, steering mapping
├── mcp.json                              # MCP server configuration
├── README.md                             # This file
├── CHANGELOG.md                          # Version history
└── steering/
    ├── estimate-workflow.md              # Cost estimation: multi-account access,
    │                                     #   130+ eventSource mappings, pricing
    └── optimization-workflow.md          # Cost optimization: aggregator-first,
                                          #   dependencies, duplicates, change
                                          #   frequency, exclusions, Control Tower
```

## References

- [Best Practices for Analyzing AWS Config Recording Frequencies](https://aws.amazon.com/blogs/mt/best-practices-for-analyzing-aws-config-recording-frequencies/)
- [How to Retrieve AWS Config Items Per Month](https://repost.aws/knowledge-center/retrieve-aws-config-items-per-month)
- [Optimize AWS Config Costs](https://repost.aws/knowledge-center/optimize-aws-config)
- [AWS Config Pricing](https://aws.amazon.com/config/pricing/)
- [AWS Config Supported Resource Types](https://docs.aws.amazon.com/config/latest/developerguide/resource-config-reference.html)
- [AWS Config Cost Optimization Best Practices](https://aws-samples.github.io/cloud-operations-best-practices/docs/guides/AWS%20Config/AWS%20Config%20Cost%20Optimization/)
- [Cost Optimization Recommendations for AWS Config](https://aws.amazon.com/blogs/mt/cost-optimization-recommendations-for-aws-config/)
- [Discover Duplicate AWS Config Rules](https://aws.amazon.com/blogs/security/discover-duplicate-aws-config-rules-for-streamlined-compliance/)
- [Customize AWS Config in Control Tower](https://aws.amazon.com/blogs/mt/customize-aws-config-resource-tracking-in-aws-control-tower-environment/)
- [AWS Config Service Integrations](https://docs.aws.amazon.com/config/latest/developerguide/service-integrations.html)
- [AWS Backup Audit Manager](https://docs.aws.amazon.com/aws-backup/latest/devguide/aws-backup-audit-manager.html)
- [AWS Systems Manager Compliance](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-compliance.html)
- [Indirect Relationships in AWS Config](https://docs.aws.amazon.com/config/latest/developerguide/faq.html#config-recording)
- [Kiro Powers Documentation](https://kiro.dev/docs/powers/create/)

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for version history and updates.

**To update**: Remove and re-add the power from the Kiro IDE Powers panel.

## License

MIT
