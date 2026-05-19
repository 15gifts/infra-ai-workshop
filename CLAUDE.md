# infra-ai-workshop

When the user asks to run the project (or any variation such as "start", "launch", "go"), respond with exactly this question before doing anything else:

> What would you like to run?
> 1. **Cost Report** — generates an AWS tag-based cost report as HTML
> 2. **Terraform Security Review** — reviews a Terraform codebase for HIGH and CRITICAL security issues and generates an HTML report

Wait for the user to choose. Do not proceed until they have selected one.

Once the user selects or indicates which function they want, read the corresponding prompt file in full and follow it exactly:

- Cost Report → read and follow `cost-report-prompt.md`
- Terraform Security Review → read and follow `terraform-security-prompt.md`

If the user asks a question or provides additional context mid-flow, answer it and then continue from where you left off in the prompt. Do not restart the prompt from the beginning.

## Reports output

- Cost reports: `~/development/infra-ai-workshop/reports/cost-report/`
- Terraform security reports: `~/development/infra-ai-workshop/reports/terraform-security/`

## Hard rules (apply to both functions)

- **Read-only.** No write, create, update, or delete operations on AWS or the local filesystem, except writing the HTML report output.
- Do not deviate from the prompt files without explicit user instruction.
- If a required tool or credential is unavailable, explain the issue clearly and stop rather than proceeding with incomplete data.
