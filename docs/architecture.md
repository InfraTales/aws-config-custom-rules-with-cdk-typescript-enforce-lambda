# Architecture Notes

## Overview

AWS Config sits at the center, recording all resource changes and routing configuration-item events to two Python Lambda evaluators: one that checks every Lambda function's timeout against a 300-second ceiling, and one that calls IAM APIs to detect users with active access keys. Evaluation results flow back to Config, and a separate remediation Lambda — triggered by Config compliance events via EventBridge — writes an audit entry to both CloudWatch Logs and S3 before touching anything. Compliance snapshots land in a dedicated S3 bucket with a three-tier lifecycle (Standard → IA at 90 days → Glacier at 365 → Deep Archive at 730) and hard expiration at 2,555 days. A LocalStack detection flag swaps Config triggers for EventBridge alternatives during local development, which is the non-obvious part most teams skip entirely.

## Key Decisions

- allSupported: true on the Config recorder captures every resource type, including ones you don't care about — Config pricing is per configuration item recorded, so in accounts with frequent EC2 or ECS churn this can spike costs unexpectedly [from-code]
- Deep Archive transition at 730 days cuts storage cost to ~$0.00099/GB/month but retrieval takes 12 hours — any compliance audit that needs data older than 2 years will block until restore completes [inferred]
- Running evaluators as Lambda functions means cold starts add latency to the evaluation pipeline; if Config batches many items simultaneously you can hit Lambda concurrency limits and push evaluations past the 15-minute SLA window [inferred]
- includeGlobalResourceTypes: true forces Config to record IAM resources but is only supported in the home region — deploying this stack to multiple regions without a guard will cause a CloudFormation error in secondary regions [inferred]
- Remediation Lambda writes to S3 and CloudWatch Logs before making changes, which is correct for auditability but means a failed S3 write silently blocks remediation unless the error path is explicitly handled [editorial]