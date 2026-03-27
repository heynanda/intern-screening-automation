[setup-instructions.md](https://github.com/user-attachments/files/26302469/setup-instructions.md)
# Workflow Setup Instructions

## Importing into n8n

1. Open your n8n instance (cloud or self-hosted)
2. Click + New Workflow → name it "intern-screening"
3. Build each node manually following the README steps, OR
   import the JSON file if provided:
   - Three-dot menu (top right) → Import from JSON

## Before activating — update these values

| Node | Field to update | What to put |
|------|----------------|-------------|
| Claude HTTP node | x-api-key header | Your Anthropic API key |
| ATS API nodes | Authorization header | Your ATS API key |
| ATS API node (pass) | stageId | Your "Recruiter Review" stage ID |
| ATS API node (reject) | stageId | Your "Auto-rejected" stage ID |
| Slack node (optional) | Webhook URL | Your Slack incoming webhook URL |

## Finding your Anthropic API key
console.anthropic.com → API Keys → Create Key

## Finding your ATS stage IDs
See docs/ats-api-setup.md for instructions per ATS.

## Testing the workflow
1. Submit a test application through your ATS job posting
2. In n8n → Executions → watch the run in real time
3. Click each node to inspect the data passing through
4. Verify the candidate appeared in the correct ATS stage
5. Run edge case tests (see README for test scenarios)
