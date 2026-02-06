# Design Document: ClarityPath AI Mobile Platform

## MVP Overview

ClarityPath AI is a React PWA for mobile-first task execution. Students complete Parsons problems with deterministic validation. Submissions require internet. Static transcripts hosted on S3/CloudFront provide shareable evidence of verified task completion. AI is optional for milestones only. Demo works without AI enabled.

## Architecture

Text Diagram:

```
Mobile Browser (Chrome/Safari)
    ↓
React PWA + Service Worker + IndexedDB (read-only offline)
    ↓ (online submission required)
API Gateway (HTTP API)
    ↓
Single Lambda (TaskService)
    ↓
DynamoDB (users, submissions) + S3/CloudFront (taskpacks, transcripts)
    ↓ (optional, milestone only)
Bedrock Claude Haiku
```

MVP has no offline submission queue; offline is read-only.

## Component Details

| Component        | AWS Service                 | Purpose               | Demo Critical | Failure Fallback                                                                    |
| ---------------- | --------------------------- | --------------------- | ------------- | ----------------------------------------------------------------------------------- |
| Frontend         | CloudFront + S3             | PWA hosting           | Yes           | Local dev server                                                                    |
| Authentication   | Cognito                     | Phone OTP (prod only) | No            | Demo bypass mode                                                                    |
| Task Storage     | S3                          | Task pack JSON        | Yes           | Pre-loaded in IndexedDB                                                             |
| API Layer        | API Gateway (HTTP API)      | HTTP endpoints        | Yes           | Cached tasks continue; show 'Sync unavailable' banner; use pre-generated transcript |
| Validation Logic | Single Lambda (TaskService) | Submission processing | Yes           | Client-side validation only                                                         |
| User Progress    | DynamoDB                    | Completion records    | Yes           | Show cached progress                                                                |
| Transcripts      | S3 + CloudFront             | Static HTML pages     | Yes           | Pre-generated transcript                                                            |
| AI Validation    | Bedrock (Claude Haiku)      | Milestone questions   | No            | Mark "Pending Review"                                                               |

## Data Models

### Users Table (DynamoDB)

```json
{
  "userId": "user_abc123",
  "authId": "cognito_sub_or_demo_user_id",
  "createdAt": "2024-01-15T10:00:00Z",
  "currentTaskIndex": 6,
  "transcriptId": "tr_xyz789"
}
```

Phone numbers are managed by Cognito and are never stored or exposed directly.

### Submissions Table (DynamoDB)

```json
{
  "submissionId": "sub_def456",
  "userId": "user_abc123",
  "taskId": "html-structure-1",
  "answer": [0, 1, 2, 3, 4, 5, 6],
  "isCorrect": true,
  "submittedAt": "2024-01-15T10:30:00Z",
  "attempts": 1
}
```

### TaskPack (S3 JSON file)

```json
{
  "id": "frontend-web-basics-v1",
  "title": "Frontend Web Basics",
  "tasks": [
    {
      "id": "html-structure-1",
      "type": "parsons",
      "title": "HTML Document Structure",
      "blocks": [
        "<!DOCTYPE html>",
        "<html>",
        "<head></head>",
        "<body></body>",
        "</html>"
      ],
      "correctSequence": [0, 1, 2, 3, 4]
    }
  ]
}
```

## API Endpoints

**GET /taskpack**

- Returns: Task pack JSON from S3
- Auth: Optional (works without login for demo)
- Latency varies on 2G/3G; UI shows clear status + retry; demo is pre-warmed

**POST /submit**

- Body: `{ userId, taskId, answer }`
- Validates answer against correct sequence
- Stores submission in DynamoDB
- Returns: `{ isCorrect: boolean, message: string }`
- Latency varies on 2G/3G; UI shows clear status + retry; demo is pre-warmed

**GET /progress**

- Query: `?userId=user_abc123`
- Returns: User's completed tasks from DynamoDB
- Auth: Required (or demo user)
- Latency varies on 2G/3G; UI shows clear status + retry; demo is pre-warmed

**POST /transcript/generate**

- Body: `{ userId }`
- User requests transcript generation
- Lambda fetches submissions from DynamoDB
- Renders HTML string with task list and timestamps
- Writes to S3: `transcripts/{transcriptId}.html`
- Returns: `{ transcriptUrl: "https://cdn.claritypath.ai/t/xyz789" }`
- Latency varies on 2G/3G; UI shows clear status + retry; demo is pre-warmed

## Deterministic Validation

Parsons Problem Sequence Matching:

```javascript
function validateParsons(userAnswer, correctSequence) {
  if (userAnswer.length !== correctSequence.length) {
    return false;
  }
  for (let i = 0; i < userAnswer.length; i++) {
    if (userAnswer[i] !== correctSequence[i]) {
      return false;
    }
  }
  return true;
}
```

Example:

- Correct: `[0, 1, 2, 3, 4]`
- User submits: `[0, 1, 2, 3, 4]` → Pass
- User submits: `[1, 0, 2, 3, 4]` → Fail (wrong order)

No AI involved. Client validates first for instant feedback. Server validates on submission for record-keeping. Server repeats the same sequence check to prevent client tampering.

## Transcript Generation Flow

1. User completes task → POST /submit → DynamoDB stores submission
2. User requests transcript → POST /transcript/generate
3. Lambda queries DynamoDB for all user submissions
4. Lambda renders HTML string:
   ```html
   <html>
     <body>
       <h1>Learning Transcript</h1>
       <ul>
         <li>HTML Structure - Completed 2024-01-15</li>
         <li>CSS Basics - Completed 2024-01-16</li>
       </ul>
     </body>
   </html>
   ```
5. Lambda writes HTML to S3: `s3://transcripts/tr_xyz789.html`
6. CloudFront serves at: `https://cdn.claritypath.ai/t/xyz789`
7. User gets shareable link in PWA

## Demo Script with Timestamps

**0:00 - 0:30 | Pre-Demo Setup**

- Task pack already in S3
- Demo user exists with 5 completed tasks
- Transcript pre-generated at `/t/demo123`
- PWA loaded on mobile device

**0:30 - 1:00 | Show Existing Progress**

- Open PWA on phone
- Show 5 completed tasks
- Display current transcript link
- Explain: "Student has evidence of verified task completion, not certificate"

**1:00 - 3:00 | Complete New Task**

- Start Parsons problem: "CSS Flexbox Layout"
- Drag 5 code blocks into correct order on mobile
- Submit (requires internet)
- Show instant validation: "Correct!"
- Task marked verified

**3:00 - 4:00 | Show Updated Transcript**

- Click "Share Transcript" button
- Copy public URL
- Open in new browser tab
- Show 6 tasks now (5 old + 1 new)
- Timestamp shows today's date

**4:00 - 5:00 | Explain Architecture**

- Show AWS console (optional)
- Explain: "PWA + Lambda + S3, no servers"
- Mention: "AI optional for milestones, not used in demo"
- Show backup screenshots if live demo fails

## MVP Tradeoffs

**We do NOT do in MVP:**

- Multiple career paths (only Frontend Web Basics)
- Offline submission (read-only offline, submit requires internet)
- Video hosting or content ingestion
- Certificates or badges
- Real-time collaboration or chat
- Complex analytics or dashboards
- Mobile app store deployment (PWA only)
- AI for daily tasks (deterministic only)

**Why these limits:**

- Single path keeps demo focused and buildable in hackathon timeframe
- Online submission avoids complex conflict resolution
- No video hosting avoids storage costs and copyright issues
- Transcripts replace certificates with actual evidence of verified task completion
- PWA avoids app store approval delays

## Cost Note

**Variable Costs:**

- Cognito OTP: SMS cost varies by provider/route; main variable cost
- Bedrock AI: ~$0.01-0.05 per milestone (optional, not required for demo)

**Fixed/Free Tier:**

- Lambda: 1M requests/month free
- DynamoDB: 25GB storage + 25 RCU/WCU free
- S3: 5GB storage + 20K GET requests free
- API Gateway: 1M requests/month free
- CloudFront: low cost at pilot scale; free tier may apply

At demo scale, running cost is near-zero (mostly within free tier). OTP is the main variable cost. AI is optional and can be kept off.

**Production Cost (100 users/month):** Single-digit USD per month; OTP is main variable; AI optional and capped.

OTP cost is the only non-linear variable; demo bypass avoids SMS entirely.
