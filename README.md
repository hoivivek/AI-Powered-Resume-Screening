# AI Powered Resume Screening
The purpose of this automation is to to streamline hiring processes with intelligent candidate evaluation and robust error monitoring.

---

## 📋 Table of Contents

- [Workflows Overview](#workflows-overview)
- [Workflow 1: AI-Powered Resume Screening](#workflow-1-ai-powered-resume-screening)
- [Workflow 2: Error Logger](#workflow-2-error-logger)
- [Tech Stack](#tech-stack)
- [Prerequisites](#prerequisites)
- [Setup & Installation](#setup--installation)
- [Configuration](#configuration)
- [How It Works](#how-it-works)
- [Scoring Methodology](#scoring-methodology)
- [Data Schema](#data-schema)


---

## Workflow 1: AI-Powered Resume Screening

### What it does

Automatically screens candidate resumes against a job description using GPT-4o-mini. When triggered via Slack, it extracts structured job requirements, retrieves resumes from Google Drive, and scores each candidate across three dimensions — outputting results back to Slack and logging them in a data table.

### Workflow Design


<img width="809" height="379" alt="image" src="https://github.com/user-attachments/assets/acf791c4-a022-4f05-a4f9-65eeae03d4b2" />



### Nodes

| Node | Type | Purpose |
|---|---|---|
| **Slack Trigger** | `slackTrigger` | Listens for messages or `@app_mention` events across the workspace |
| **Extract Job Requirements** | `langchain.agent` | Parses job description and extracts structured fields |
| **OpenAI Chat Model** | `lmChatOpenAi` | GPT-4o-mini (temp: 0.2) powering the extraction agent |
| **Job Requirements Schema** | `outputParserStructured` | Enforces typed JSON output for job data |
| **Get Resumes from Drive** | `googleDrive` | Downloads PDF resumes from Google Drive |
| **Extract Resume Text** | `extractFromFile` | Converts PDF binary to plain text |
| **Calculate Match Scores** | `langchain.agent` | Scores the resume against job requirements |
| **OpenAI Scoring Model** | `lmChatOpenAi` | GPT-4o-mini (temp: 0.1) powering the scoring agent |
| **Match Scores Schema** | `outputParserStructured` | Enforces typed JSON output for scores |
| **Prepare Results for Logging** | `set` | Maps and formats all fields with a timestamp |
| **Log Screening Results** | `dataTable` | Persists results to the `screening_results` data table |
| **Post to Slack** | `slack` | Sends a formatted screening summary to `#general` |

### Slack Output Format

```
🎯 Resume Screening Results

Candidate: [Name]
Company Name: [Company]
Job Role: [Role]
Overall Match: [Score]%

📊 Score Breakdown:
• Skills Match: [Score]%
• Experience Match: [Score]%
• Education Match: [Score]%

Resume: [filename]
Screened: [ISO timestamp]
```

---

## Workflow 2: Error Logger

### What it does

Acts as a global error handler for the Resume Screening workflow (and any other workflow that references it). When any execution fails, it simultaneously logs the error details to a data table and sends an alert to Slack.

### Pipeline Flow


<img width="809" height="402" alt="image" src="https://github.com/user-attachments/assets/886e9f3b-6a72-4969-b808-1e1bb7dea7d7" />



### Nodes

| Node | Type | Purpose |
|---|---|---|
| **When Error Occurs** | `errorTrigger` | Fires automatically on any workflow execution failure |
| **Log Error to Data Table** | `dataTable` | Writes error details to the `error log` data table |
| **Send Slack Notification** | `slack` | Posts an alert with the error message and timestamp to `#general` |

### Slack Alert Format

```
🚨 Error Alert

Error: [error message]
Time: [current timestamp]
```

---

## Tech Stack

- **Automation Platform:** [n8n](https://n8n.io/)
- **AI Model:** OpenAI GPT-4o-mini (via LangChain nodes)
- **File Storage:** Google Drive
- **Notifications:** Slack
- **Data Persistence:** n8n Data Tables

---

## Prerequisites

Before importing these workflows, make sure you have:

- An n8n instance (self-hosted or n8n Cloud)
- An **OpenAI API key** with access to `gpt-4o-mini`
- A **Slack App** with OAuth2 configured and bot permissions:
  - `channels:read`
  - `chat:write`
  - `app_mentions:read`
  - `messages.channels` (for trigger)
- A **Google Drive OAuth2** credential with read access to the folder containing resumes
- Two n8n **Data Tables** created with the schemas below

---

## Setup & Installation

1. **Clone or download** this repository.

2. **Import workflows** into your n8n instance:
   - Go to **Workflows → Import from File**
   - Import `AI-Powered_Resume_Screening.json`
   - Import `Error_Logger.json`

3. **Configure credentials** (see [Configuration](#configuration) below).

4. **Create the required Data Tables** in your n8n project (see [Data Schema](#data-schema)).

5. **Link the Error Workflow:** The Resume Screening workflow is pre-configured to use the Error Logger (`errorWorkflow: "E8DkCziQU2TO1bc4"`). Ensure the Error Logger workflow ID matches after import, and update the setting if needed under **Workflow Settings → Error Workflow**.

6. **Activate both workflows.**

---

## Configuration

### Credentials to Replace

| Credential | Used In | Notes |
|---|---|---|
| `openAiApi` | Extract Job Requirements, Calculate Match Scores | Replace with your own OpenAI API credential |
| `slackOAuth2Api` | Post to Slack, Send Slack Notification | OAuth2 Slack App credential |
| `slackApi` | Slack Trigger | Bot token credential for the trigger node |
| `googleDriveOAuth2Api` | Get Resumes from Drive | OAuth2 Google credential |


---

## How It Works

### Resume Screening (Step by Step)

1. A user sends a message to the Slack bot containing a job description (or mentions the app).
2. The **Extract Job Requirements** agent parses the description and returns a structured JSON object with fields like `requiredSkills`, `minYearsExperience`, `educationLevel`, etc.
3. The workflow fetches a resume PDF from Google Drive and converts it to plain text.
4. The **Calculate Match Scores** agent compares the resume text against the extracted job requirements and generates numerical scores.
5. Results are formatted, timestamped, and written to the `screening_results` data table.
6. A summary card is posted back to Slack.

### Error Handling

Any failure in the Resume Screening workflow automatically triggers the Error Logger, which records the `Workflow_ID`, `Workflow_Name`, `ErrorMessage`, and `ErrorNode` to the `error log` table, and simultaneously fires a Slack alert.

---

## Scoring Methodology

The AI scoring agent uses a weighted average across three dimensions:

| Dimension | Weight | What's Evaluated |
|---|---|---|
| **Skills Match** | 40% | Required and preferred skills present in the resume |
| **Experience Match** | 35% | Years of experience and relevance to the role |
| **Education Match** | 25% | Education level and field of study alignment |

All scores are on a **0–100** scale. The `overallMatchScore` is the final weighted composite.

---

## Data Schema

### `screening_results` Table

| Column | Type | Description |
|---|---|---|
| `candidateName` | string | Extracted from the resume |
| `companyName` | string | Extracted from the job description |
| `jobRole` | string | Job title extracted from the posting |
| `overallMatchScore` | number | Weighted composite score (0–100) |
| `skillsMatch` | number | Skills alignment score (0–100) |
| `experienceMatch` | number | Experience alignment score (0–100) |
| `educationMatch` | number | Education alignment score (0–100) |
| `resumeFileName` | string | Name of the screened resume file |
| `screeningTimestamp` | string | ISO 8601 timestamp of when screening ran |

### `error log` Table

| Column | Type | Description |
|---|---|---|
| `Workflow_ID` | string | Execution ID of the failed run |
| `Workflow_Name` | string | Name of the workflow that errored |
| `ErrorMessage` | string | The error message thrown |
| `ErrorNode` | string | The last node that executed before failure |
