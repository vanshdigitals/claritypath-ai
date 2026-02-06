# Requirements Document

## Scope Definition

**IN SCOPE (MVP)**:

- React PWA with Service Worker + IndexedDB
- Single learning path: Frontend Web Basics (HTML/CSS/JS)
- Parsons problems with deterministic validation
- MVP supports Parsons problems only. Other task types are future.
- Read-only offline task viewing; submission requires internet
- Phone OTP authentication (production); demo bypass allowed
- Static transcript generation (S3 + CloudFront)
- Pre-staged demo data for 5-minute presentation

**OUT OF SCOPE (Future)**:

- Multiple career paths
- Video hosting or content ingestion
- Certificates or badges
- Offline submission capability
- Real-time collaboration
- Mobile app store deployment
- Advanced analytics

## Glossary

- **ClarityPath_System**: React PWA + AWS serverless backend for task execution enforcement
- **Task_Pack**: Pre-generated Frontend Web tasks stored as JSON in S3
- **Parsons_Problem**: Drag-and-drop code blocks into correct sequence (mobile-optimized)
- **Deterministic_Validator**: Client-side validation using exact sequence matching
- **Transcript_Page**: Static HTML showing verified task completions, hosted on S3/CloudFront
- **Share_ID**: Random unguessable identifier for public transcript URLs
- **Demo_User**: Pre-configured user with existing progress for live demonstration

## Requirements

### Requirement 1: Authentication

**User Story:** As a student, I want to authenticate with my phone number, so I can access my learning progress securely across sessions.

Demo bypass uses a fixed demo user; Cognito is not exercised live.

#### Acceptance Criteria

1. THE ClarityPath_System SHALL support phone OTP authentication via Amazon Cognito for production accounts
2. THE ClarityPath_System SHALL allow demo bypass mode using pre-authenticated user for presentation purposes
3. THE ClarityPath_System SHALL maintain user sessions for 7 days after successful login
4. THE ClarityPath_System SHALL store user progress in DynamoDB linked to authId (Cognito sub in production, fixed demo user id in demo)
5. THE ClarityPath_System SHALL function without Cognito in demo mode (bypass authentication)

### Requirement 2: Task Pack and Offline Reading

**User Story:** As a student with limited connectivity, I want to view learning tasks offline, so I can study when internet is unavailable.

Offline mode is read-only by design to avoid sync conflicts.

#### Acceptance Criteria

1. THE ClarityPath_System SHALL cache task packs in IndexedDB via Service Worker
2. THE ClarityPath_System SHALL allow read-only task viewing without internet connection
3. THE ClarityPath_System SHALL be designed for intermittent 2G/3G; "first load < 10s on 3G (tested). Cached tasks open instantly"
4. THE ClarityPath_System SHALL display clear "offline mode" indicator when disconnected
5. THE ClarityPath_System SHALL target text-first bundle, best-effort (<1–2MB gzipped for core PWA shell)

### Requirement 3: Structured Tasks

**User Story:** As a phone-only learner, I want mobile-friendly coding tasks, so I can practice without typing complex code.

#### Acceptance Criteria

1. THE ClarityPath_System SHALL provide Parsons problems with touch-friendly drag-and-drop
2. THE ClarityPath_System SHALL validate task answers using deterministic sequence matching
3. THE ClarityPath_System SHALL target client-side feedback within 500ms (best-effort)
4. THE ClarityPath_System SHALL use zero AI for daily task validation
5. THE ClarityPath_System SHALL display tasks in mobile-first responsive layout

### Requirement 4: Online Submission and Verification

**User Story:** As a learner completing tasks, I want my work verified and recorded, so I can build evidence of verified task completion.

#### Acceptance Criteria

1. THE ClarityPath_System SHALL require internet connection for task submission
2. THE ClarityPath_System SHALL send submissions to Lambda via API Gateway
3. THE ClarityPath_System SHALL store verified completions in DynamoDB
4. Submission may take up to 5–8 seconds on 3G during demo; UI SHALL show clear status and allow retry
5. THE ClarityPath_System SHALL show clear error message if submission fails

### Requirement 5: Public Transcript

**User Story:** As a student building evidence of verified task completion, I want a shareable transcript link, so recruiters can view my verified work without app access.

#### Acceptance Criteria

1. THE ClarityPath_System SHALL generate static HTML transcript pages via Lambda
2. THE ClarityPath_System SHALL host transcripts on S3 with CloudFront distribution
3. THE ClarityPath_System SHALL create public URLs using unguessable Share_ID
4. THE ClarityPath_System SHALL exclude phone numbers from public transcript pages
5. THE ClarityPath_System SHALL display completed and verified tasks with timestamps (no skill mastery claims)

### Requirement 6: Demo Pre-Staging and Failure Fallback

**User Story:** As a hackathon presenter, I want reliable demo flow with fallback options, so technical failures don't derail the presentation.

#### Acceptance Criteria

1. THE ClarityPath_System SHALL pre-seed Demo_User with 5 verified tasks and existing transcript
2. THE ClarityPath_System SHALL pre-load task pack in S3 before demo starts
3. THE ClarityPath_System SHALL function completely without Bedrock AI enabled
4. THE ClarityPath_System SHALL provide backup screenshots for each demo state
5. THE ClarityPath_System SHALL target rehearsed live demo flow (task → submit → transcript) under ~3 minutes
