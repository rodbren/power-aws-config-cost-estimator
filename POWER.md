---
name: "aws-config-cost-estimator"
displayName: "AWS Config Cost Estimator"
description: "Estimate AWS Config recorder costs by analyzing CloudTrail organization trail write events against Config-supported resource types to forecast configuration item counts and monthly costs"
keywords: ["aws config", "config cost", "configuration items", "cloudtrail", "organization trail", "config estimator", "config pricing", "config recorder", "cost estimate", "config forecast"]
---

# Onboarding

## Step 1: Validate prerequisites
Before estimating AWS Config costs, verify:
- **CloudTrail Organization Trail**: An organization trail must exist and be logging management events. Verify with: `aws cloudtrail describe-trails`
- **AWS Credentials**: The user must have permissions to call `cloudtrail:LookupEvents` and `cloudtrail:DescribeTrails`.
- **CRITICAL**: If no organization trail exists, inform the user they need one configured. A regular trail will work for single-account estimates but won't cover the full organization.

## Step 2: Understand the approach
AWS Config creates a **configuration item (CI)** whenever it detects a change to a recorded resource type. By querying CloudTrail for **non-read-only management events** from services that map to AWS Config supported resource types, we can estimate how many CIs would be recorded monthly.

**Default analysis window is 7 days**, extrapolated to a monthly estimate. Customers can request shorter (1–6 days) or longer (8–90 days) windows for less or more accuracy.

**This is an approximation** — not every write API call creates a Config CI, and some Config resource types (like `AWS::Config::ResourceCompliance`) have no CloudTrail equivalent.

## MCP Servers
This power uses the following MCP servers:
- **aws-documentation**: Search and read AWS documentation for Config pricing, supported resource types, and best practices.
- **aws-core**: Execute AWS CLI commands (e.g., `cloudtrail describe-trails`, `cloudtrail lookup-events`) to query CloudTrail data.
- **aws-knowledge**: Retrieve knowledge base content for AWS Config and CloudTrail guidance.

# When to Load Steering Files
- Running a Config cost estimate, querying CloudTrail, or analyzing events → `estimate-workflow.md`
