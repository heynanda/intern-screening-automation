# AI-Powered Intern Application Screener
### n8n + Claude + ATS Integration

> **Disclaimer:** This repository is for **educational purposes only**. It is a proof-of-concept demonstration of an AI-augmented recruiting automation workflow. No real candidate data is used. All examples are illustrative. This workflow was inspired by real-world experience screening large volumes of intern applications and is shared to demonstrate AI-native recruiting practices.

---

## The Problem

Imagine receiving **90,000 engineering intern applications** in a single hiring cycle.

Manually reviewing each one — even for 30 seconds — would take **750+ hours** of recruiter time. At 2 minutes per application, that's **3,000 hours**. Weeks of work. Inconsistent evaluation. Recruiter fatigue. Good candidates buried under volume.

This is a real problem. And it is entirely solvable with automation.

---

## The Solution

A three-tool workflow that automatically screens every application against a defined set of criteria — and routes candidates into the right stage before a single human spends any time reviewing.

```
Ashby / ATS  →  n8n  →  Claude AI  →  Decision  →  Back to ATS
(receives        (runs    (reads          (pass /      (stages candidate,
 applications)   logic)   resumes)         reject)      sends email)
```

**Result:** Human review time focuses exclusively on pre-qualified, pre-tagged candidates. The automation handles volume. The recruiter handles judgment.

---

## Tools Used

| Tool | Role | Why |
|------|------|-----|
| **n8n** | Workflow automation engine | Visual, no-code/low-code, connects to any API |
| **Claude (Anthropic)** | Resume parsing and signal extraction | Best-in-class at reading unstructured text and returning structured data |
| **Ashby / Generic ATS** | Applicant Tracking System | Receives applications, stores candidate data, manages stages |

> **Note on ATS:** This workflow is designed for Ashby but the same logic applies to any ATS that supports webhooks and API access — Lever, Greenhouse, Workday, HiringThing, or a simple Google Sheet. The ATS is interchangeable. The logic stays the same.

---

## Screening Criteria

The workflow evaluates every application against three gates, in order. An application stops at the first gate it fails.

### Gate 1 — Graduation Year
- **Pass:** Graduating in the target intern window (e.g. 2025 or 2026)
- **Fail:** Already graduated OR graduating too far out
- **Why first:** Fastest check. Eliminates the largest mismatch group immediately with zero judgment required.

### Gate 2 — Prior Experience
- **Pass (Option A):** 2 or more internships at technology companies
- **Pass (Option B):** Meaningful open-source contribution — active GitHub commits, contributor role on a known OSS project, or publicly visible engineering work
- **Fail:** Zero internship experience AND no OSS activity
- **Why Option B matters:** Some of the best candidates never got traditional internships. They built things publicly instead. A student who contributed to a popular OSS project is showing the same signal as someone who interned at a startup. Excluding them is a fairness problem, not just a sourcing problem.

### Gate 3 — Skill Area Match
The workflow detects which engineering domain the candidate belongs to and tags them accordingly:

| Tag | Keywords detected in resume |
|-----|----------------------------|
| `backend` | Python, Java, Golang, Node.js, REST API, SQL, PostgreSQL, microservices, system design |
| `frontend` | React, Vue, Angular, TypeScript, HTML, CSS, UI components, web performance |
| `full-stack` | Combination of backend + frontend signals |
| `platform` | Kubernetes, Docker, AWS, GCP, Azure, CI/CD, DevOps, Linux, networking, Terraform |
| `ai-ml` | PyTorch, TensorFlow, Hugging Face, LLMs, RAG, model training, data science, NLP |
- **Fail:** No technology keywords detected → auto-reject

### Priority Tiering (Post Gate 3)
Candidates who pass all three gates are further sorted:

| Priority | Criteria | Action |
|----------|----------|--------|
| P1 | 2+ tech internships + skill match | Move to top of review queue |
| P2 | 1 internship + strong OSS + skill match | Second in queue |
| P3 | Skill match only, limited experience | Review if capacity allows |

---

## Workflow Architecture

### Node-by-Node Breakdown

```
[1] WEBHOOK TRIGGER (n8n)
     │
     ▼
[2] RESUME PARSER (Claude API via HTTP node)
     │  Extracts: graduation_year, internship_count,
     │  internship_companies[], github_url, skills[]
     ▼
[3] IF NODE — Graduation Year Check
     │  True: graduation_year IN [2025, 2026]  ──────────────────────┐
     │  False: ──────────────────────────────────────────────────┐   │
     ▼                                                           ▼   ▼
[4] IF NODE — Experience Check                            [REJECT BRANCH]
     │  True: internship_count >= 2 OR github_active == true     │
     │  False: ────────────────────────────────────────────────► │
     ▼                                                           │
[5] SWITCH NODE — Skill Area Tagging                            │
     │  backend / frontend / full-stack / platform / ai-ml       │
     │  none: ──────────────────────────────────────────────── ► │
     ▼                                                           │
[6a] ASHBY API — Move to "Recruiter Review"                     │
      Set stage, add skill tag, set priority flag               │
                                                                │
[6b] ASHBY API — Move to "Auto-Rejected"  ◄──────────────────────┘
      Trigger rejection email template
```

---

## The Claude Prompt

This is the exact prompt sent to Claude for every resume. Claude reads the raw resume text and returns structured JSON.

```
You are a resume parsing assistant for a technical recruiting team.

Read the resume text below and extract the following information.
Return ONLY valid JSON — no markdown, no explanation, no extra text.

Resume text:
{{resume_text}}

Return this exact JSON structure:
{
  "graduation_year": <4-digit year as integer, or null if not found>,
  "internship_count": <number of internships at technology companies>,
  "internship_companies": [<list of company names>],
  "github_url": <GitHub profile URL as string, or null>,
  "has_oss_contribution": <true if resume mentions open source contribution, false otherwise>,
  "skills": [<list of all technical skills mentioned>],
  "skill_areas": {
    "backend": <true/false>,
    "frontend": <true/false>,
    "full_stack": <true/false>,
    "platform": <true/false>,
    "ai_ml": <true/false>
  },
  "primary_skill_area": <"backend" | "frontend" | "full_stack" | "platform" | "ai_ml" | "none">
}
```

**Why Claude for this?**
- Resumes are unstructured text. Rule-based parsing breaks on formatting variations.
- Claude understands context — it knows "SDE Intern at Razorpay (June–August 2023)" is a technology internship without needing an explicit list of tech companies.
- Claude handles edge cases gracefully — non-English text, unconventional resume formats, PDF-extracted text with noise.
- Returns consistent JSON regardless of resume format.

---

## n8n Workflow: Step-by-Step Setup

### Prerequisites
- [ ] n8n account (cloud: [n8n.io](https://n8n.io) or self-hosted)
- [ ] Anthropic API key ([console.anthropic.com](https://console.anthropic.com))
- [ ] ATS account with webhook + API access enabled
- [ ] 30–60 minutes to set up

---

### Step 1: Create the Webhook Trigger

1. Open n8n → New Workflow → name it `intern-screening-v1`
2. Click **+** → search **Webhook** → drag onto canvas
3. Set **HTTP Method** to `POST`
4. Copy the **Webhook URL** shown (e.g. `https://your-n8n.app.n8n.cloud/webhook/abc123`)
5. In your ATS (Ashby / Lever / Greenhouse):
   - Go to **Settings → Integrations → Webhooks**
   - Click **Add Webhook**
   - Paste the n8n URL
   - Set trigger event: **Application Created**
   - Save

> The webhook fires every time a new application is submitted. No manual action needed.

---

### Step 2: Extract Resume Text

Most ATS webhooks send application metadata but not the full resume text. You need to fetch the resume separately.

1. Connect an **HTTP Request** node to the Webhook
2. Set method: `GET`
3. URL: `{{$json["resume_url"]}}` (the resume URL from the webhook payload)
4. This fetches the raw resume file (PDF or text)

For PDF parsing, add a **PDF parsing service** node (e.g. Affinda, or use n8n's built-in binary file handling) to extract plain text from the PDF before sending to Claude.

---

### Step 3: Send to Claude for Parsing

1. Connect an **HTTP Request** node
2. Set method: `POST`
3. URL: `https://api.anthropic.com/v1/messages`
4. Headers:
   ```
   x-api-key: YOUR_ANTHROPIC_API_KEY
   anthropic-version: 2023-06-01
   Content-Type: application/json
   ```
5. Body (JSON):
   ```json
   {
     "model": "claude-sonnet-4-20250514",
     "max_tokens": 1000,
     "messages": [
       {
         "role": "user",
         "content": "You are a resume parsing assistant...[paste full prompt above]...\n\n{{$json[\"resume_text\"]}}"
       }
     ]
   }
   ```
6. The response will contain the JSON with all extracted fields.

---

### Step 4: Parse the Claude Response

Claude returns a message object. You need to extract the actual JSON from it.

1. Connect a **Code** node
2. Paste this JavaScript:
   ```javascript
   const raw = $input.first().json.content[0].text;
   const parsed = JSON.parse(raw);
   return [{ json: parsed }];
   ```
3. This converts Claude's text response into structured data that subsequent nodes can use.

---

### Step 5: Gate 1 — Graduation Year IF Node

1. Connect an **IF** node
2. Condition: `{{ $json.graduation_year }}` **is equal to** `2025` **OR** `{{ $json.graduation_year }}` **is equal to** `2026`
3. **True branch** → continue to Gate 2
4. **False branch** → route to rejection node (build this in Step 8)

---

### Step 6: Gate 2 — Experience IF Node

1. Connect another **IF** node on the True branch from Step 5
2. Condition:
   - `{{ $json.internship_count }}` **greater than or equal to** `2`
   - **OR** `{{ $json.has_oss_contribution }}` **is equal to** `true`
3. **True branch** → continue to Gate 3
4. **False branch** → rejection node

---

### Step 7: Gate 3 — Skill Area Switch Node

1. Connect a **Switch** node
2. Value to switch on: `{{ $json.primary_skill_area }}`
3. Add cases:
   - `backend` → output 1
   - `frontend` → output 2
   - `full_stack` → output 3
   - `platform` → output 4
   - `ai_ml` → output 5
   - `none` (default) → rejection branch
4. Each passing output route goes to an ATS API node (Step 8)

---

### Step 8: ATS API — Move to Recruiter Review

For each skill area output from the Switch node:

1. Connect an **HTTP Request** node
2. Method: `PATCH` (or `POST` depending on your ATS API)
3. URL: `https://api.ashbyhq.com/applicationStage.change` (or your ATS equivalent)
4. Headers: Include your ATS API key
5. Body:
   ```json
   {
     "applicationId": "{{ $json.applicationId }}",
     "stageId": "RECRUITER_REVIEW_STAGE_ID",
     "tags": ["{{ $json.primary_skill_area }}"],
     "note": "Auto-screened: passed all gates. Priority: {{ $json.internship_count >= 2 ? 'P1' : 'P2' }}"
   }
   ```

---

### Step 9: ATS API — Auto-Reject with Respectful Email

For all rejection branches:

1. Connect an **HTTP Request** node
2. Move candidate to rejected stage via ATS API
3. Trigger the rejection email template you set up in the ATS in advance

**Sample rejection email template:**
```
Subject: Your application for [Role] at [Company]

Hi {{first_name}},

Thank you for applying for our engineering intern programme.

We reviewed your application and while your profile is impressive,
we are moving forward with candidates whose experience more closely
matches what we need for this cycle.

We encourage you to apply again in a future cycle and wish you
the best in your search.

[Company] Recruiting Team
```

> **Candidate experience principle:** Every applicant gets a response — automatically, within hours, not weeks. Even a rejection handled with speed and respect is better than silence. This is HackerRank's founding thesis applied to our own workflow.

---

## Testing Before Going Live

1. Submit a test application yourself through the ATS job posting
2. In n8n, go to **Executions** → watch the workflow run in real time
3. Click each node to see the data passing through
4. Verify the candidate appeared in the correct ATS stage
5. Check the email was sent correctly
6. Test edge cases:
   - Wrong graduation year → should auto-reject at Gate 1
   - Zero internships + no GitHub → should reject at Gate 2
   - Non-tech skills only → should reject at Gate 3
   - Perfect candidate → should appear in Recruiter Review with correct tag

---

## What This Achieves

| Metric | Without Automation | With Automation |
|--------|--------------------|-----------------|
| Time to screen 90,000 applications | 750–3,000 hours | < 1 hour (compute time) |
| Consistency of criteria | Variable (human fatigue) | 100% identical for every application |
| Time to candidate response | Days to weeks | Minutes to hours |
| Recruiter hours on unqualified candidates | ~70% of total | 0% |
| Recruiter hours on qualified candidates | ~30% of total | 100% of total |

---

## Extending This Workflow

Once the basic workflow runs, here are ways to extend it:

**Add GitHub enrichment**
If a GitHub URL is detected, use the GitHub API to fetch the candidate's public repos, star count, commit activity, and language breakdown. Add this as additional signal for prioritisation.

**Add email sequence**
For Priority 1 candidates, trigger a personalised "we saw your application" email via SendGrid or Postmark — letting strong candidates know they'll hear from a recruiter soon. Reduces dropoff from top candidates who get competing offers.

**Add Slack notification**
When a P1 candidate passes all gates, send an n8n Slack node message to the relevant hiring manager: "New P1 backend engineer candidate just passed screening — review when you can."

**Add a scoring model**
Instead of binary pass/fail, assign a numerical score to each candidate based on weighted criteria. Sort the Recruiter Review queue by score, not just priority tier.

---

## Folder Structure

```
intern-screening/
├── README.md                    ← This file
├── workflow/
│   └── intern-screening.json    ← Exportable n8n workflow file
├── prompts/
│   └── resume-parser.txt        ← Claude prompt for resume parsing
├── templates/
│   ├── rejection-email.txt      ← Rejection email template
│   └── p1-notification.txt      ← Slack/email for P1 candidates
└── docs/
    └── ats-api-setup.md         ← Notes on configuring ATS webhooks
```

---

## Why This Matters

The skills needed to build this are not advanced engineering skills. They are:

- **Process thinking** — defining clear, consistent screening criteria
- **Tool awareness** — knowing which tools exist and what each one does well
- **Prompt design** — knowing how to ask an AI to extract the right information
- **Workflow logic** — connecting steps in the right order with the right conditions

This is what AI-native recruiting looks like. Not replacing human judgment — **directing it to where it actually matters.**

The recruiter who built this workflow spent their time designing the criteria, testing the logic, and improving the quality signal. Not sorting through 90,000 applications manually.

---

## About

Built to demonstrate AI-native recruiting automation in practice.

Inspired by real experience handling high-volume intern hiring at a fast-growing technology company.

Connect: [LinkedIn](https://www.linkedin.com/in/nandakishoresubba/) · [GitHub](https://github.com/heynanda) · [X](https://twitter.com/nksubba)

---

*For questions, suggestions, or collaboration — open an issue or connect on LinkedIn.*
