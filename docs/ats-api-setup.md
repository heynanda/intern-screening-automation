[ats-api-setup.md](https://github.com/user-attachments/files/26302367/ats-api-setup.md)
# ATS Webhook and API Setup

## How to connect any ATS to this workflow

### Step 1 — Get your n8n webhook URL
Open the workflow in n8n → click the Webhook trigger node →
copy the Production URL shown.
Format: https://your-instance.app.n8n.cloud/webhook/abc123

---

### Step 2 — Register the webhook in your ATS

#### Ashby
Settings → Integrations → Webhooks → Add Webhook
- URL: paste your n8n webhook URL
- Event: Application Created
- API base URL: https://api.ashbyhq.com
- Auth: Basic (API key as username, blank password)
- Key endpoints:
  - Move stage: POST /applicationStage.change
  - Add tag:    POST /tag.createAndApply
  - List stages: GET /applicationStage.list (use this to find stage IDs)

#### Lever
Settings → Integrations → Webhooks
- Event: candidateCreated
- API base URL: https://api.lever.co/v1/
- Auth: Bearer token

#### Greenhouse
Dev Center → Web Hooks → New Web Hook
- Event: Application Created
- API base URL: https://harvest.greenhouse.io/v1/
- Auth: Basic (API key as username)

#### Any other ATS
Requirements to swap in a different ATS:
1. Outbound webhook on new application → triggers n8n
2. REST API to update candidate stage → n8n moves them
3. REST API to add tags/labels → n8n tags skill area

Only the URLs, authentication headers, and request body format
change between ATS tools. The n8n logic stays identical.

---

### Step 3 — Find your stage IDs

You need the internal IDs of two stages in your ATS:
1. The "Recruiter Review" stage (where passing candidates go)
2. The "Auto-rejected" stage (where failing candidates go)

In Ashby, call:
  GET https://api.ashbyhq.com/applicationStage.list
  with your API key. The response lists all stages with their IDs.

Copy the IDs and paste them into the relevant HTTP nodes in n8n.

---

### Troubleshooting

**Webhook not firing**
- Check the ATS webhook is pointing to the correct n8n URL
- Make sure you activated the workflow in n8n (toggle top right)
- Use n8n's "Test workflow" button to trigger manually

**Claude returning invalid JSON**
- Check your API key is correct in the HTTP headers
- Check the resume text is being passed correctly to the prompt
- Add a Code node after Claude to strip any leading/trailing whitespace

**Candidates going to wrong stage**
- Double-check stage IDs are correct
- Check the IF node conditions are using the right field names
  from Claude's JSON output
