# infra-ai-workshop

When the user asks to run the project (or any variation such as "start", "launch", "go"), respond with exactly this question before doing anything else:

> What would you like to run?
> 1. **Cost Report** — generates an AWS tag-based cost report as HTML
> 2. **Terraform Security Review** — reviews a Terraform codebase for HIGH and CRITICAL security issues and generates an HTML report

Wait for the user to choose. Do not proceed until they have selected one.

Once the user selects or indicates which function they want, follow the corresponding prompt file exactly:

- Cost Report → `cost-report-prompt.md`
- Terraform Security Review → `terraform-security-prompt.md`

## Reports output

- Cost reports: `~/development/infra-ai-workshop/reports/cost-report/`
- Terraform security reports: `~/development/infra-ai-workshop/reports/terraform-security/`

## Hard rules (apply to both functions)

- **Read-only only.** No write, create, update, or delete operations on AWS or the local filesystem, except writing the HTML report output.
- Do not deviate from the prompt files without explicit user instruction.
