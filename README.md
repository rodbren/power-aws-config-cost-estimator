# AWS Config Cost Estimator — Kiro Power

Estimate AWS Config recorder costs **before enabling it** by analyzing CloudTrail organization trail write events against AWS Config's supported resource types.

## How It Works

```
┌──────────────────────────────────────────────────────────────────────────┐
│                          Kiro IDE / Power                                │
│                                                                          │
│  1. Activate power with keywords                                         │
│     ("estimate config costs", "config pricing", etc.)                    │
│                                                                          │
│  2. Onboarding: Verify CloudTrail Organization Trail exists              │
│                                                                          │
│  3. Query CloudTrail Organization Trail                                  │
│     ┌───────────────────────────────────┐                                │
│     │  Filter:                          │                                │
│     │   • readOnly = false              │                                │
│     │   • managementEvent = true        │                                │
│     └──────────────┬────────────────────┘                                │
│                    │                                                     │
│                    ▼                                                     │
│  4. Map eventSource → AWS Config Supported Resource Types                │
│     ┌───────────────────────────────────────────────────┐                │
│     │ ec2.amazonaws.com        → EC2, EBS, VPC, ...     │                │
│     │ s3.amazonaws.com         → S3 Buckets, ...        │                │
│     │ lambda.amazonaws.com     → Lambda Functions        │                │
│     │ iam.amazonaws.com        → IAM Users, Roles, ...  │                │
│     │ rds.amazonaws.com        → RDS Instances, ...     │                │
│     │ ... 130+ eventSource mappings                     │                │
│     └──────────────┬────────────────────────────────────┘                │
│                    │                                                     │
│                    ▼                                                     │
│  5. Two estimation modes                                                 │
│                                                                          │
│     ┌─────────────────────────┐    ┌──────────────────────────────────┐  │
│     │  CONTINUOUS RECORDING   │    │  PERIODIC RECORDING              │  │
│     │                         │    │                                  │  │
│     │  Query: 7 days default  │    │  Query: 1 day (user picks day)  │  │
│     │  Count: total events    │    │  Count: unique resource IDs     │  │
│     │  Extrapolate: ×30/7     │    │  Project: ×7 weekly, ×30 monthly│  │
│     │  Price: $0.003/CI       │    │  Price: $0.012/CI               │  │
│     └─────────────────────────┘    └──────────────────────────────────┘  │
│                                                                          │
│  6. Output: Side-by-side cost tables + top cost drivers                  │
└──────────────────────────────────────────────────────────────────────────┘
```

## Example Output

### Continuous Recording Estimate
**Analysis period**: 7 days (2026-03-25 to 2026-03-31) | **Extrapolation**: ×4.29

| Account ID | Region | Event Source | Events (7d) | Est. Monthly CIs | Est. Monthly Cost |
|---|---|---|---|---|---|
| 123456789012 | us-east-1 | ec2.amazonaws.com | 1,200 | 5,143 | $15.43 |
| 123456789012 | us-east-1 | s3.amazonaws.com | 350 | 1,500 | $4.50 |
| 123456789012 | us-east-1 | iam.amazonaws.com | 200 | 857 | $2.57 |
| **TOTAL** | | | **1,750** | **7,500** | **$22.50** |

### Periodic Recording Estimate
**Sample day**: 2026-03-31

> ⚠️ Weekly and monthly projections are based on a single day's data. Re-run on different days for higher accuracy.

| Account ID | Region | Event Source | Unique Resources (1d) | Est. Weekly CIs | Est. Monthly CIs | Est. Weekly Cost | Est. Monthly Cost |
|---|---|---|---|---|---|---|---|
| 123456789012 | us-east-1 | ec2.amazonaws.com | 45 | 315 | 1,350 | $3.78 | $16.20 |
| 123456789012 | us-east-1 | s3.amazonaws.com | 20 | 140 | 600 | $1.68 | $7.20 |
| 123456789012 | us-east-1 | iam.amazonaws.com | 12 | 84 | 360 | $1.01 | $4.32 |
| **TOTAL** | | | **77** | **539** | **2,310** | **$6.47** | **$27.72** |

## Key Difference: Continuous vs Periodic

| | Continuous | Periodic |
|---|---|---|
| **What triggers a CI** | Every resource change | 1 per unique resource per day |
| **Price per CI** | $0.003 | $0.012 |
| **Best for** | Resources that rarely change | Resources that change frequently |
| **Data source** | 7 days of events (configurable) | 1 day of unique resource IDs |
| **Example**: EC2 instance modified 50×/day | 50 CIs = $0.15/day | 1 CI = $0.012/day |

## Installation

In Kiro IDE:
1. Open the **Powers panel**
2. Click **"Add power from GitHub"**
3. Enter: `https://github.com/rodbren/power-aws-config-cost-estimator`

## Prerequisites

- **CloudTrail Organization Trail** with management events enabled
- AWS credentials with `cloudtrail:LookupEvents` and `cloudtrail:DescribeTrails` permissions

## MCP Servers Included

| Server | Purpose |
|---|---|
| `aws-documentation` | Search AWS docs for Config pricing and supported resource types |
| `aws-core` | Execute AWS CLI commands to query CloudTrail data |
| `aws-knowledge` | Retrieve AWS knowledge base content for guidance |

## Usage

Activate the power by mentioning any of these keywords in a conversation:
- "aws config", "config cost", "config estimator", "config pricing"
- "configuration items", "config recorder", "cloudtrail"

### Continuous: Adjustable Time Range

| Command | Days | Accuracy |
|---|---|---|
| "quick estimate" | 7 (default) | Good |
| "use 1 day" | 1 | Low — snapshot only |
| "use 30 days" | 30 | High |
| "use 90 days" | 90 | Highest (max CloudTrail LookupEvents supports) |

### Periodic: Selectable Sample Day

| Command | Behavior |
|---|---|
| (default) | Uses yesterday |
| "use March 28" | Samples that specific day |
| "use last Monday" | Samples a typical weekday |
| "use last Saturday" | Samples a weekend day for comparison |

## Caveats

- **Approximation**: Not every CloudTrail write event creates a Config CI
- **Missing types**: Some Config resource types (e.g., `AWS::Config::ResourceCompliance`) have no CloudTrail equivalent
- **Periodic forecasting**: Weekly/monthly periodic projections are extrapolated from a single day — pick a representative day for best results
- **CI recording only**: Config rules and conformance pack evaluations are billed separately
- **Regional pricing**: Verify current pricing at [aws.amazon.com/config/pricing](https://aws.amazon.com/config/pricing/)

## Project Structure

```
power-aws-config-cost-estimator/
├── POWER.md                          # Power metadata, onboarding, steering mapping
├── mcp.json                          # MCP server configuration
├── README.md                         # This file
└── steering/
    └── estimate-workflow.md          # Full estimation workflow + eventSource mapping
```

## References

- [AWS Config Pricing](https://aws.amazon.com/config/pricing/)
- [AWS Config Supported Resource Types](https://docs.aws.amazon.com/config/latest/developerguide/resource-config-reference.html)
- [Estimating AWS Config Costs Using CloudTrail (AWS Blog)](https://aws.amazon.com/blogs/mt/estimating-aws-config-recorder-costs-and-usage-using-aws-cloudtrail/)
- [Kiro Powers Documentation](https://kiro.dev/docs/powers/create/)

## License

MIT
