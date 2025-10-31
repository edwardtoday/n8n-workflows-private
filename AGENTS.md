# Repository Guidelines

## Project Structure & Module Organization
- Root workspace groups individual n8n workflows in dedicated folders (e.g., `Sansi LED Weekly Intel/`, `Academic Research Weekly Intel/`, `comment-webhook/`).  
- Each workflow directory contains a primary `workflow.json`; supporting docs (such as `source-registry.md`) live alongside the JSON.  
- Aggregated API payloads (`workflow_payload.json`, `workflow_payload_min.json`) sit at the repo root for bulk updates.
- 需要参考成熟模板时，可浏览以下本地仓库：`/Users/qingpei/git/n8n/n8n-workflows`、`/Users/qingpei/git/n8n/awesome-n8n-templates`、`/Users/qingpei/git/n8n/n8n-free-templates`，其中按主题收集了多套可复用流程。

## Build, Test, and Development Commands
- There is no traditional build pipeline. Use curl with the n8n API key to update workflows, for example:  
  `curl -X PUT -H "Content-Type: application/json" -H "X-N8N-API-Key: $N8N_API_KEY" --data-binary @workflow_payload.json https://n8n.example.com/api/v1/workflows/<id>`
- Trigger a workflow manually via:  
  `curl -X POST -H "X-N8N-API-Key: $N8N_API_KEY" https://n8n.example.com/webhook/<webhook-path>`

## Coding Style & Naming Conventions
- Workflow JSON should keep script nodes’ `jsCode` and `functionCode` fields identical for clarity.  
- Prefer snake_case for `parserId`/`nodeId` values in custom scripts (e.g., `unilumin_en`).  
- Maintain two-space indentation in embedded JavaScript for readability.

## Testing Guidelines
- Validate changes by triggering the workflow through its webhook and reviewing execution diagnostics:  
  `curl -H "X-N8N-API-Key: $N8N_API_KEY" "https://n8n.example.com/api/v1/executions?limit=1&workflowId=<id>&includeData=true"`  
- For the academic weekly workflow, run the new webhook (`/webhook/academic-weekly-1d5752a1a7a14641a4c63e55`) after deployment and confirm the Dropbox output under `/Apps/n8n-qingpei/academic-weekly-intel/`.  
- Inspect `diagnostics.errors` and resulting artifacts (e.g., Dropbox markdown output) to ensure data quality before shipping.

## Commit & Pull Request Guidelines
- Use concise, Chinese commit messages summarizing the change scope (e.g., “修复周报源解析”).  
- Prefix new merge requests with `Draft:` until all validations pass; include execution IDs or log references in the description for traceability.  
- Reference relevant workflow IDs and highlight any new external dependencies (RSS/API sources) in the PR body.
