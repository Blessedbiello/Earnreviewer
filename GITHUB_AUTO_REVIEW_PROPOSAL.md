# GitHub Auto-Review System - Comprehensive Project Proposal
## Superteam Earn Bounty Submission

**Author**: Senior Staff Engineer | Full-Stack & Blockchain Developer
**Date**: December 19, 2025
**Project Duration**: 2-3 weeks
**Status**: Proposal for Implementation

---

## Table of Contents

1. [Executive Summary](#executive-summary)
2. [Technical Architecture](#technical-architecture)
3. [Current State Analysis](#current-state-analysis)
4. [Proposed Solution](#proposed-solution)
5. [Implementation Roadmap](#implementation-roadmap)
6. [Edge Case Handling](#edge-case-handling)
7. [Scoring Algorithm Design](#scoring-algorithm-design)
8. [Files to Create/Modify](#files-to-createmodify)
9. [Success Criteria](#success-criteria)
10. [Risk Assessment & Mitigation](#risk-assessment--mitigation)
11. [Cost & Resource Estimates](#cost--resource-estimates)
12. [Monitoring & Maintenance](#monitoring--maintenance)
13. [Deliverables](#deliverables)

---

## Executive Summary

### Project Overview

Superteam Earn currently supports AI-powered auto-reviews for text-based submissions (grants, projects, writing bounties), but lacks automated review capabilities for GitHub-based submissions. This proposal outlines a comprehensive solution to build an AI-powered GitHub PR and repository auto-review system that seamlessly integrates with the existing earn-agent infrastructure.

### Key Objectives

1. **Automate GitHub submission reviews** for PRs and repositories
2. **Integrate seamlessly** with existing BullMQ queue system and AI infrastructure
3. **Handle 40+ edge cases** comprehensively (private repos, rate limits, large PRs, etc.)
4. **Provide intelligent scoring** using AI analysis and code quality metrics
5. **Scale to production** handling 1000+ submissions/day

### Technical Approach

- **GitHub GraphQL API** for efficient data fetching (single query vs multiple REST calls)
- **OpenRouter AI SDK** (already integrated) for code analysis and scoring
- **BullMQ + Redis** queue system (existing infrastructure)
- **Token rotation strategy** to avoid GitHub rate limits (5 PATs = 25k requests/hour)
- **Event-driven architecture** following existing patterns for grants/projects/bounties

### Expected Outcomes

- **End-to-end automation**: User submits GitHub link → AI analyzes → Sponsor reviews results
- **80%+ accuracy**: AI reviews align with manual sponsor reviews on test dataset
- **<30 second processing time** for typical PRs (<500 files)
- **Zero downtime**: Comprehensive error handling for all edge cases
- **Scalable solution**: Handle 1000+ submissions/day with current infrastructure

---

## Technical Architecture

### System Components

```
┌─────────────────────────────────────────────────────────────────────────┐
│                          SUPERTEAM EARN PLATFORM                        │
└─────────────────────────────────────────────────────────────────────────┘

┌─────────────────┐                                    ┌─────────────────┐
│   User Submits  │                                    │  Sponsor Views  │
│   GitHub URL    │                                    │  AI Review in   │
│  (PR or Repo)   │                                    │   Dashboard     │
└────────┬────────┘                                    └────────▲────────┘
         │                                                       │
         v                                                       │
┌─────────────────────────────────────────────────────────────────────────┐
│                         EARN SERVICE (Next.js)                          │
├─────────────────────────────────────────────────────────────────────────┤
│  1. URL Validation & Parsing (detect PR vs Repo)                       │
│  2. Store in Database (Submission table)                               │
│  3. Queue Agent Job (BullMQ)                                            │
│  4. Sponsor Dashboard APIs (unreviewed, commit-reviewed)               │
└────────┬────────────────────────────────────────────────────────────────┘
         │
         v
┌─────────────────────────────────────────────────────────────────────────┐
│                    REDIS + BULLMQ QUEUE                                 │
│                   (agentLogicQueue)                                     │
│                                                                          │
│  Job Type: autoReviewGitHubSubmission                                   │
│  Payload: { submissionId, githubUrl, type: 'pr'|'repo' }              │
└────────┬────────────────────────────────────────────────────────────────┘
         │
         v
┌─────────────────────────────────────────────────────────────────────────┐
│              EARN-AGENT SERVICE (Worker - Railway)                      │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐             │
│  │   Fetch PR/  │───>│  AI Analysis │───>│ Store Results│             │
│  │  Repo Data   │    │  & Scoring   │    │  in Database │             │
│  │  (GitHub API)│    │  (OpenRouter)│    │  (Prisma)    │             │
│  └──────────────┘    └──────────────┘    └──────────────┘             │
│         │                    │                                          │
│         v                    v                                          │
│  ┌──────────────┐    ┌──────────────┐                                  │
│  │ Rate Limit   │    │ Edge Case    │                                  │
│  │ Handler      │    │ Handler      │                                  │
│  │ (5 tokens)   │    │ (40+ cases)  │                                  │
│  └──────────────┘    └──────────────┘                                  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
         │
         v
┌─────────────────────────────────────────────────────────────────────────┐
│                         EXTERNAL SERVICES                               │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                          │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐             │
│  │   GitHub     │    │  OpenRouter  │    │   Firecrawl  │             │
│  │  GraphQL API │    │   AI SDK     │    │  (Optional)  │             │
│  │              │    │              │    │              │             │
│  │ • PR data    │    │ • Code       │    │ • Repo       │             │
│  │ • Repo stats │    │   analysis   │    │   scraping   │             │
│  │ • Reviews    │    │ • Scoring    │    │   (fallback) │             │
│  └──────────────┘    └──────────────┘    └──────────────┘             │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

### Integration Points

#### 1. Earn Service (Public Repo)

**Modified Components**:
- Submission creation API: Detect GitHub URLs and queue jobs
- Submission form validation: Add GitHub URL validation
- Sponsor dashboard: Display GitHub-specific stats
- Agent queue utils: Add `autoReviewGitHubSubmission` type

**New Components**:
- GitHub URL parser utility
- GitHub API client wrapper
- GitHub stats API endpoint
- AI Review GitHub modal component

#### 2. Earn-Agent Service (Private Repo - requires access)

**New Components**:
- GitHub data fetcher (GraphQL queries)
- Code analysis engine
- Scoring algorithm implementation
- AI prompt builder for GitHub submissions
- GitHub-specific error handlers

#### 3. External Services

**GitHub API**:
- GraphQL API for efficient data fetching
- REST API fallback for specific operations
- Webhook support (optional future enhancement)

**OpenRouter**:
- Already integrated in earn platform
- Will be used for code analysis via AI

**Firecrawl** (mentioned in bounty):
- Optional: For advanced repo scraping if needed
- Not required for MVP (GitHub API provides sufficient data)

### Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Backend** | Node.js + TypeScript | Consistent with existing earn codebase |
| **API Framework** | Next.js API Routes | Integration with existing earn service |
| **Queue System** | BullMQ + Redis | Existing infrastructure for agent jobs |
| **Database** | MySQL + Prisma | Existing earn database |
| **GitHub API** | Octokit.js (GraphQL) | Official GitHub SDK |
| **AI Analysis** | OpenRouter + Vercel AI SDK | Already integrated |
| **Deployment** | Railway (earn-agent) | Mentioned in bounty description |

---

## Current State Analysis

### Existing Auto-Review Infrastructure

Based on comprehensive codebase analysis of the `earn` repository, the following auto-review capabilities already exist:

#### 1. Grant Applications
- **Trigger**: On application submission
- **Agent Type**: `autoReviewGrantApplication`
- **Analysis**: Reviews application text, external links (Google Docs, GitHub, Twitter), applicant's profile
- **Output**: Predicted label, short note, scores (stored in `submission.ai`)

#### 2. Project Applications
- **Trigger**: On submission creation for project listings
- **Agent Type**: `autoReviewProjectApplication`
- **Analysis**: Context-based evaluation of skills, experience, application quality
- **Output**: Predicted label, notes, scores for skills/experience/application

#### 3. Bounty Submissions (Text/Tweet)
- **Trigger**: After deadline (~12 hours)
- **Agent Types**: `generateContextBounty` (at listing creation), batch review after deadline
- **Analysis**: Tweet analytics, content quality, criteria matching
- **Output**: Quality score, criteria score, total score, notes

### Database Schema

**Submission Table** (relevant fields):
```typescript
{
  id: string;
  link?: string;              // GitHub URL will be stored here
  label: SubmissionLabels;    // Unreviewed, Reviewed, Shortlisted, etc.
  ai?: Json;                  // AI evaluation data (flexible JSON field)
  notes?: string;             // Sponsor notes (AI-generated or manual)
  // ... other fields
}
```

**Submission Labels** (enum):
- `Unreviewed` - Not yet reviewed
- `Reviewed` - Manually reviewed
- `Shortlisted` - Top quality
- `Spam` - Low quality/spam
- `Low_Quality` - Below standards
- `Mid_Quality` - Acceptable
- `High_Quality` - Good quality
- `Pending` - In review
- `Inaccessible` - **Important for GitHub errors**
- `Needs_Review` - **Important for edge cases**

### Agent Queue System

**File**: `src/features/agents/utils/queueAgent.ts`

**Current Implementation**:
```typescript
type AgentActionType =
  | 'autoReviewGrantApplication'
  | 'generateContextProject'
  | 'autoReviewProjectApplication'
  | 'generateContextBounty';

// Queue configuration
const redis = new Redis(process.env.AGENT_REDIS_URL!);
const logicQueue = new Queue('agentLogicQueue', { connection: redis });

// Job options
{
  attempts: 3,
  backoff: { type: 'exponential', delay: 1000 },
  priority: 1,
  removeOnComplete: true,
  removeOnFail: 500
}
```

### Sponsor Dashboard Workflow

**Current UX Flow**:
1. Sponsor views submissions at `/dashboard/listings/[slug]/submissions`
2. Clicks "AI Review" button
3. Modal shows disclaimer and starts review process
4. Progress bar displays (caps at 99% until completion)
5. Results show distribution: Shortlisted, Mid/Low Quality, Inaccessible, Needs Review
6. Sponsor clicks "Proceed" to commit AI labels to database
7. AI notes converted to HTML and stored in `submission.notes`

**API Endpoints**:
- `GET /api/sponsor-dashboard/submissions/ai/unreviewed` - Fetch unreviewed submissions
- `POST /api/sponsor-dashboard/submissions/ai/commit-reviewed` - Apply AI labels

### Gaps & Opportunities

**What's Missing for GitHub**:
1. ❌ GitHub URL detection and parsing
2. ❌ GitHub API integration (no Octokit found)
3. ❌ Code quality analysis logic
4. ❌ PR-specific evaluation criteria
5. ❌ Repository-specific evaluation criteria
6. ❌ GitHub-specific error handling (rate limits, private repos, etc.)
7. ❌ UI components for displaying GitHub stats

**What We Can Leverage**:
1. ✅ BullMQ + Redis queue infrastructure
2. ✅ Agent job retry logic with exponential backoff
3. ✅ OpenRouter AI SDK integration
4. ✅ Submission label system (can use existing labels)
5. ✅ Flexible `submission.ai` JSON field
6. ✅ Sponsor dashboard modal pattern
7. ✅ Prisma ORM and database access patterns

---

## Proposed Solution

### Core Features

#### 1. GitHub URL Detection & Validation

**Supported URL Patterns**:
```
Pull Requests:
- https://github.com/{owner}/{repo}/pull/{number}
- https://github.com/{owner}/{repo}/pulls/{number}

Repositories:
- https://github.com/{owner}/{repo}
- https://github.com/{owner}/{repo}/tree/{branch}
- https://github.com/{owner}/{repo}/commit/{sha}
```

**Validation Logic**:
- Regex pattern matching
- Extract owner, repo, PR number (if applicable)
- Detect type: PR vs Repo
- Validate accessibility (public vs private)
- Store metadata in `submission.ai.githubMeta`

#### 2. GitHub Data Fetching

**For Pull Requests** (GraphQL query):
```graphql
{
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $number) {
      title, body, state, isDraft
      additions, deletions, changedFiles
      commits(first: 100) { nodes { ... } }
      files(first: 100) { nodes { path, additions, deletions } }
      reviews(first: 100) { nodes { state, author } }
      reviewThreads { totalCount }
      mergeable, mergeStateStatus
      headRefName, baseRefName
    }
  }
}
```

**For Repositories**:
```graphql
{
  repository(owner: $owner, name: $repo) {
    stargazerCount, forkCount, watchers { totalCount }
    issues(states: OPEN) { totalCount }
    languages(first: 10) { edges { size, node { name } } }
    defaultBranchRef {
      target { ... on Commit { history { totalCount } } }
    }
    hasIssuesEnabled, hasWikiEnabled
    licenseInfo { name }
    readme: object(expression: "HEAD:README.md") { ... on Blob { text } }
  }
}
```

**Data Stored in Database**:
```typescript
submission.ai = {
  githubType: 'pr' | 'repo',
  githubMeta: {
    owner: string,
    repo: string,
    number?: number,
    url: string,
    detectedAt: timestamp
  },
  githubData: {
    pr?: { ... },    // PR-specific data
    repo?: { ... }   // Repo-specific data
  },
  evaluation: { ... },
  error?: { code, message, type },
  commited: boolean,
  analyzedAt: timestamp
}
```

#### 3. AI-Powered Code Analysis

**Analysis Components**:

**A. Quantitative Metrics** (Direct from GitHub):
- Lines of code changed (additions/deletions)
- Number of files modified
- Commit count and quality
- Review comments and approvals
- CI/CD status
- Repository stars/forks/activity

**B. Qualitative Analysis** (AI-powered via OpenRouter):
- Code quality and complexity assessment
- Alignment with bounty requirements
- Documentation completeness
- Testing adequacy
- Code style and best practices
- Security considerations

**AI Prompt Structure**:
```
System: You are an expert code reviewer evaluating a GitHub submission for a bounty.

Bounty Context:
- Title: {listing.title}
- Description: {listing.description}
- Requirements: {listing.requirements}
- Skills: {listing.skills}

GitHub Data:
- Type: PR/Repo
- Changes: +{additions} -{deletions} in {files} files
- Commits: {commit_count}
- [Code samples/README content]

Evaluate:
1. Code Quality (0-100): Structure, maintainability, complexity
2. Alignment (0-100): Meets bounty requirements?
3. Documentation (0-100): README, comments, explanations
4. Testing (0-100): Test coverage, quality
5. Overall Recommendation: Shortlisted/High/Mid/Low/Needs Review

Provide specific, actionable feedback (2-3 sentences).
```

#### 4. Scoring Algorithm

**For Pull Requests**:

| Component | Weight | Scoring Criteria |
|-----------|--------|------------------|
| **Code Quality** | 30% | Complexity, style, best practices |
| **PR Size & Scope** | 15% | Optimal: 100-500 lines |
| **Commit Quality** | 10% | Messages, count, distribution |
| **Documentation** | 15% | PR description, code comments |
| **Testing** | 15% | Test presence, coverage, quality |
| **Review Feedback** | 5% | Approvals, change requests |
| **CI/CD Status** | 5% | Build/test results |
| **Bounty Alignment** | 5% | Matches requirements |

**For Repositories**:

| Component | Weight | Scoring Criteria |
|-----------|--------|------------------|
| **Code Quality** | 25% | Structure, organization, complexity |
| **Documentation** | 20% | README, API docs, examples |
| **Testing** | 20% | Test files, CI/CD setup |
| **Activity** | 15% | Commit frequency, maintenance |
| **Community** | 10% | Stars, forks, contributors |
| **Tech Stack** | 5% | Language/framework alignment |
| **License** | 5% | Presence, compatibility |

**Label Assignment**:
```
Score >= 85:  Shortlisted
70-84:        High_Quality
50-69:        Mid_Quality
30-49:        Low_Quality
< 30:         Low_Quality (recommend reject)

Special Cases:
- GitHub errors:        Inaccessible
- Edge cases/uncertain: Needs_Review
```

#### 5. Error Handling Strategy

**Comprehensive Coverage of 40+ Edge Cases**:

**Access Errors**:
- Private repository (404/403) → Label: `Inaccessible`
- Deleted PR/repo → Label: `Inaccessible`
- GitHub token expired → Token rotation, retry
- Insufficient permissions → Fallback to public API

**PR State Issues**:
- Draft PR → Label: `Needs_Review`, flag as incomplete
- Closed (not merged) → Lower score, note closure reason
- Force push during review → Use snapshot at submission time
- Branch deleted → Use cached data if available

**Size & Content**:
- Very large PRs (>1000 files) → Sample analysis, flag `isTooLarge`
- Empty repository → Label: `Low_Quality`
- No code changes (docs only) → Context-dependent scoring
- Generated code detected → Filter out, flag if >80%

**Rate Limiting**:
- GitHub API limits → Token rotation (5 tokens = 25k/hour)
- Secondary rate limit → Exponential backoff
- All tokens exhausted → Queue for retry in 1 hour

**Timing Issues**:
- Analysis timeout (>5 min) → Cancel, label `Needs_Review`
- AI service down → Retry 3x, then `Needs_Review`
- GitHub API down → Retry every 15 min for 2 hours

**Workflow**:
- Duplicate submissions → Reject at submission time
- Link edited after review → Disallow if already analyzed
- Submission after deadline → Existing validation handles this

---

## Implementation Roadmap

### Timeline: 2-3 Weeks (Detailed Breakdown)

#### **WEEK 1: Core Infrastructure & GitHub Integration**

##### **Days 1-2: Environment Setup & GitHub API Client**

**Tasks**:
- [ ] Obtain GitHub Personal Access Tokens (5 tokens for rotation)
  - Create GitHub PATs with `repo:public_repo` scope
  - Store securely in environment variables: `GITHUB_TOKEN_1` through `GITHUB_TOKEN_5`
  - Document token rotation strategy

- [ ] Set up Octokit.js GraphQL client
  - Install: `npm install octokit`
  - Create `/src/lib/github.ts` wrapper with:
    - Token rotation logic
    - Rate limit monitoring
    - Automatic retry on 403/429
    - Error handling utilities

- [ ] Create GitHub URL parser utility
  - File: `/src/features/submissions/utils/githubUrlParser.ts`
  - Functions:
    - `isGitHubUrl(url: string): boolean`
    - `parseGitHubUrl(url: string): GitHubMeta | null`
    - `detectGitHubType(url: string): 'pr' | 'repo' | null`
  - Test with various URL patterns

**Deliverables**:
- Working GitHub API client with token rotation
- URL parser with unit tests
- Documentation for token setup

##### **Days 3-4: Submission Flow Integration**

**Tasks**:
- [ ] Extend TypeScript types
  - File: `/src/features/submissions/types/githubSubmission.ts`
  - Define `GitHubSubmissionAi` interface
  - Define `GitHubMeta`, `PRData`, `RepoData` types

- [ ] Extend agent action types
  - File: `/src/features/agents/utils/queueAgent.ts`
  - Add: `type AgentActionType = ... | 'autoReviewGitHubSubmission'`

- [ ] Modify submission creation API
  - File: `/src/pages/api/submission/create.ts`
  - After validation, detect GitHub URLs:
    ```typescript
    const githubMeta = parseGitHubUrl(validatedData.link);
    if (githubMeta) {
      submissionData.ai = {
        githubType: githubMeta.type,
        githubMeta: {...}
      };
    }
    ```
  - After DB insert, queue GitHub review:
    ```typescript
    if (githubMeta) {
      await queueAgent({
        type: 'autoReviewGitHubSubmission',
        id: result.id,
        otherInfo: githubMeta
      });
    }
    ```

- [ ] Update submission form validation
  - File: `/src/features/listings/utils/submissionFormSchema.ts`
  - Add optional GitHub URL validation for specific bounty types
  - Display helper text: "Ensure your repository is public"

**Deliverables**:
- GitHub URL detection in submission flow
- Agent jobs queued for GitHub submissions
- Updated TypeScript interfaces

##### **Days 5-7: GitHub Data Fetcher (earn-agent service)**

**⚠️ Note**: Requires access to private `earn-agent` repository. If access not granted, these tasks will be delayed.

**Tasks**:
- [ ] Create GraphQL query builders
  - File: `/src/workers/github/queries/pr.ts`
  - Comprehensive PR query (title, body, files, commits, reviews, CI status)

  - File: `/src/workers/github/queries/repo.ts`
  - Repository query (metadata, languages, stats, README)

- [ ] Implement data fetcher functions
  - File: `/src/workers/github/fetchPRData.ts`
  - `fetchPRData(owner, repo, number): Promise<PRData>`
  - Handle pagination for large PRs
  - Extract and normalize data

  - File: `/src/workers/github/fetchRepoData.ts`
  - `fetchRepoData(owner, repo): Promise<RepoData>`
  - Fetch repository statistics
  - Parse README content

- [ ] Implement error handling
  - File: `/src/workers/github/errorHandler.ts`
  - Handle 404 (private/deleted) → Return error object
  - Handle 403 (rate limit) → Trigger token rotation
  - Handle timeout → Cancel request, return partial data
  - Handle API downtime → Queue for retry

- [ ] Add rate limit management
  - File: `/src/workers/github/rateLimiter.ts`
  - Monitor `X-RateLimit-Remaining` header
  - Switch tokens when remaining < 100
  - Log token usage statistics
  - Implement exponential backoff on secondary limits

**Deliverables**:
- Complete GitHub data fetching pipeline
- Error handling for all API failures
- Rate limiting strategy implemented
- Integration tests with real GitHub data

**Testing Strategy**:
- Test with public PRs of various sizes
- Test with private repos (expect 404)
- Test with deleted/invalid URLs
- Simulate rate limit scenarios

---

#### **WEEK 2: AI Analysis & Scoring Engine**

##### **Days 8-10: Scoring Algorithm Implementation**

**Tasks**:
- [ ] Implement PR scoring logic
  - File: `/src/workers/github/scorers/prScorer.ts`
  - Functions:
    - `analyzePRSize(additions, deletions): number`
    - `analyzeCommitQuality(commits): number`
    - `analyzeDocumentation(prBody, hasReadme): number`
    - `analyzeTesting(files): number`
    - `calculatePRScore(prData): ScoreBreakdown`

- [ ] Implement Repository scoring logic
  - File: `/src/workers/github/scorers/repoScorer.ts`
  - Functions:
    - `analyzeRepoActivity(commits, lastCommit): number`
    - `analyzeRepoCommunity(stars, forks, contributors): number`
    - `analyzeRepoDocumentation(readme, hasWiki): number`
    - `calculateRepoScore(repoData): ScoreBreakdown`

- [ ] Create label assignment logic
  - File: `/src/workers/github/scorers/labelAssigner.ts`
  - `assignLabel(overallScore, flags): SubmissionLabels`
  - Implement threshold-based assignment
  - Handle special flags (needsManualReview, isTooLarge, etc.)

**Deliverables**:
- Complete scoring algorithm for PRs and repos
- Unit tests for each scoring function
- Documentation of scoring methodology

##### **Days 11-12: AI Integration**

**Tasks**:
- [ ] Build AI prompt templates
  - File: `/src/workers/github/prompts/prPrompt.ts`
  - Create structured prompt for PR analysis
  - Include bounty context, PR data, code samples
  - Define expected JSON output format

  - File: `/src/workers/github/prompts/repoPrompt.ts`
  - Create structured prompt for repository analysis

- [ ] Implement AI analyzer
  - File: `/src/workers/github/analyzer.ts`
  - Integrate with OpenRouter API
  - Functions:
    - `analyzePRWithAI(prData, listingContext): Promise<AIEvaluation>`
    - `analyzeRepoWithAI(repoData, listingContext): Promise<AIEvaluation>`
  - Parse AI response and extract scores
  - Combine AI scores with quantitative metrics

- [ ] Add large PR handling
  - File: `/src/workers/github/samplers/prSampler.ts`
  - For PRs >1000 files:
    - Sample first 50 files
    - Include all test files
    - Include files matching bounty keywords
  - Set flag: `isTooLarge: true`
  - Add note about sampling strategy

**Deliverables**:
- AI-powered code analysis pipeline
- Prompt templates optimized for accuracy
- Large PR sampling strategy
- Integration with OpenRouter

##### **Days 13-14: Edge Cases & Testing**

**Tasks**:
- [ ] Implement edge case handlers
  - File: `/src/workers/github/edgeCases/prEdgeCases.ts`
  - Handle draft PRs
  - Handle closed (not merged) PRs
  - Handle force push scenarios
  - Detect generated code
  - Detect copy-pasted code

  - File: `/src/workers/github/edgeCases/repoEdgeCases.ts`
  - Handle empty repositories
  - Handle monorepos
  - Handle forks vs original repos

- [ ] Add duplicate detection
  - Before queuing job, check for existing submission with same GitHub URL
  - Reject duplicate at submission time
  - Add database query: `findFirst({ where: { listingId, link: githubUrl } })`

- [ ] Comprehensive testing
  - Create test suite with 20+ real GitHub PRs/repos
  - Test edge cases: private repos, deleted PRs, large PRs
  - Validate scoring accuracy against manual reviews
  - Test error scenarios: rate limits, timeouts, API failures

**Deliverables**:
- Edge case handling for 40+ scenarios
- Comprehensive test suite
- Validation report (AI accuracy vs manual reviews)

---

#### **WEEK 3: Dashboard Integration & Production Hardening**

##### **Days 15-17: Frontend Components**

**Tasks**:
- [ ] Create GitHub review modal component
  - File: `/src/features/sponsor-dashboard/components/Submissions/Modals/AiReviewGitHub.tsx`
  - Similar structure to `AiReviewBounties.tsx`
  - States: DISCLAIMER → INIT → PROCESSING → DONE
  - Display GitHub-specific stats:
    - For PRs: additions, deletions, files changed, commits
    - For repos: stars, forks, languages, last commit
  - Show distribution of labels after review
  - Disclaimer: "AI can make mistakes. Verify code quality manually."

- [ ] Update SubmissionPanel component
  - File: `/src/features/sponsor-dashboard/components/Submissions/SubmissionPanel.tsx`
  - Add GitHub badge/icon for GitHub submissions
  - Display `submission.ai.githubData` in panel
  - Show PR state (open/closed/merged) with appropriate styling
  - Link to GitHub URL with external link icon

- [ ] Add GitHub stats display component
  - File: `/src/features/sponsor-dashboard/components/Submissions/GitHubStats.tsx`
  - Reusable component for displaying PR/repo metrics
  - Visual indicators for code quality, testing, documentation
  - Chart/graph for score breakdown (optional)

**Deliverables**:
- Functional AI Review modal for GitHub
- GitHub metadata display in submission panel
- Updated UI matching existing design system

##### **Days 18-19: API Endpoints**

**Tasks**:
- [ ] Create GitHub stats API
  - File: `/src/app/api/sponsor-dashboard/submissions/github-stats/route.ts`
  - `GET /api/sponsor-dashboard/submissions/github-stats?id={submissionId}`
  - Return: `{ type: 'pr'|'repo', pr?: {...}, repo?: {...} }`
  - Fetch from `submission.ai.githubData`
  - Cache response (5 minutes TTL)

- [ ] Modify unreviewed submissions API
  - File: `/src/app/api/sponsor-dashboard/submissions/ai/unreviewed/route.ts`
  - Include GitHub submissions in query
  - Filter: `ai.evaluation exists && ai.commited === false`
  - Return GitHub-specific metadata for UI

- [ ] Modify commit reviewed API
  - File: `/src/app/api/sponsor-dashboard/submissions/ai/commit-reviewed/route.ts`
  - Handle GitHub label conversion:
    - AI prediction → `submission.label`
    - AI notes → `submission.notes` (convert to HTML)
  - Set `ai.commited = true`
  - For bounties: Apply tiered ranking if needed

- [ ] Add validation endpoint (optional)
  - File: `/src/app/api/github/validate/route.ts`
  - `POST /api/github/validate` with `{ url: string }`
  - Return: `{ valid, type, owner, repo, number, accessible }`
  - Use for real-time validation in submission form

**Deliverables**:
- GitHub stats API endpoint
- Updated unreviewed/commit APIs
- Complete API documentation

##### **Days 20-21: Production Hardening & Testing**

**Tasks**:
- [ ] Implement monitoring
  - Track metrics:
    - GitHub API rate limit usage
    - AI service response time & error rate
    - Queue length & processing time
    - Error distribution by type
  - Set up alerts:
    - Rate limit <10% remaining
    - AI error rate >5%
    - Queue backlog >100 jobs

- [ ] Add comprehensive logging
  - Log all GitHub API calls (with rate limit headers)
  - Log AI analysis results (for debugging)
  - Log errors with full context (submission ID, GitHub URL, error type)
  - Use structured logging (JSON format)

- [ ] Performance optimization
  - Implement caching strategy:
    - Cache repository metadata (24 hour TTL)
    - Use ETag headers for conditional requests
  - Batch requests where possible
  - Optimize database queries (add indexes if needed)

- [ ] End-to-end testing
  - Test complete workflow:
    1. User submits GitHub URL
    2. Agent processes submission
    3. Sponsor views AI review
    4. Sponsor commits review
    5. Verify database updates
  - Test with 10+ real submissions
  - Validate against success criteria (80% accuracy, <30s processing)

- [ ] Documentation
  - Create deployment guide for earn-agent updates
  - Document environment variables needed
  - Write troubleshooting guide for common errors
  - Create monitoring dashboard setup guide

**Deliverables**:
- Production-ready monitoring and logging
- Performance-optimized implementation
- Complete end-to-end testing
- Deployment and maintenance documentation

---

## Edge Case Handling

### Comprehensive Coverage of 40+ Scenarios

#### **Category 1: GitHub Access & Permissions (10 cases)**

| # | Edge Case | Detection | Handling | Label |
|---|-----------|-----------|----------|-------|
| 1 | Private repository | 404/403 from API | Store error, add note | `Inaccessible` |
| 2 | Deleted PR/repository | 404 error | Store error, note deletion | `Inaccessible` |
| 3 | GitHub token expired | 401 Unauthorized | Token rotation, retry | `Needs_Review` (if all fail) |
| 4 | Insufficient token permissions | 403 with specific message | Fall back to public API | Continue or `Needs_Review` |
| 5 | DMCA takedown | 451 Unavailable | Store error, note legal issue | `Inaccessible` |
| 6 | Repository fork vs original | `repo.fork === true` | Note in review, analyze fork | Normal scoring |
| 7 | Organization repo with restrictions | 403 error | Same as private repo | `Inaccessible` |
| 8 | GitHub API down | Persistent 500/503 | Retry every 15min for 2h | `Needs_Review` |
| 9 | Network timeout | Request timeout | Retry 3x with backoff | `Needs_Review` |
| 10 | Locked conversation | `pr.locked === true` | No impact, continue | Normal scoring |

#### **Category 2: PR State Issues (8 cases)**

| # | Edge Case | Detection | Handling | Label |
|---|-----------|-----------|----------|-------|
| 11 | Draft PR | `pr.draft === true` | Flag as incomplete, note | `Needs_Review` |
| 12 | PR merged before review | `pr.merged === true` | Positive signal, continue | Boost score +5 |
| 13 | PR closed (not merged) | `pr.state === 'closed' && !merged` | Lower score, note reason | `Mid_Quality` or lower |
| 14 | Force push during review | Monitor `pr.head.sha` | Use snapshot at submission | Normal scoring |
| 15 | Branch deleted after merge | HEAD ref 404 | Use cached commit SHAs | Normal scoring |
| 16 | PR from fork | `pr.head.repo.fork === true` | Note source, analyze | Normal scoring |
| 17 | PR without description | `pr.body === null` | Reduce documentation score | Lower documentation |
| 18 | Very old PR (>1 year) | Compare dates | Note age in review | Context-dependent |

#### **Category 3: Repository Size & Content (10 cases)**

| # | Edge Case | Detection | Handling | Label |
|---|-----------|-----------|----------|-------|
| 19 | Very large PR (>1000 files) | `pr.changed_files > 1000` | Sample 50 files + tests | `Needs_Review` |
| 20 | Empty repository | `repo.size === 0` or no commits | Cannot analyze | `Low_Quality` |
| 21 | Monorepo | Analyze directory structure | Focus on relevant subdirectory | Normal scoring |
| 22 | Generated code (>80%) | Pattern detection in files | Filter out, flag | Lower score |
| 23 | Copy-pasted code | AI detects duplication | Flag for manual review | `Needs_Review` |
| 24 | Very small PR (<5 lines) | `additions + deletions < 5` | Context-dependent scoring | Variable |
| 25 | Dependencies only | Changes only package.json | Check bounty requirements | `Needs_Review` |
| 26 | Documentation only | All markdown/text files | Good if bounty is about docs | Context-dependent |
| 27 | Test files only | All files in test/ | Check if refactoring bounty | Context-dependent |
| 28 | Binary files (images, etc.) | Detect file types | Exclude from analysis | Normal scoring |

#### **Category 4: Rate Limiting & Performance (6 cases)**

| # | Edge Case | Detection | Handling | Label |
|---|-----------|-----------|----------|-------|
| 29 | GitHub API rate limit | `X-RateLimit-Remaining < 10` | Switch token, continue | Normal scoring |
| 30 | All tokens exhausted | All tokens rate limited | Queue for retry in 1h | `Needs_Review` (temp) |
| 31 | Secondary rate limit | 403 with "abuse detection" | Exponential backoff | Retry or `Needs_Review` |
| 32 | Review timeout (>5 min) | Timeout in worker | Cancel, flag for manual | `Needs_Review` |
| 33 | AI service timeout | OpenRouter timeout | Retry 3x, fallback | `Needs_Review` |
| 34 | Very slow API response | Response time >10s | Continue but log warning | Normal scoring |

#### **Category 5: Submission Workflow (6 cases)**

| # | Edge Case | Detection | Handling | Label |
|---|-----------|-----------|----------|-------|
| 35 | Duplicate submission (same PR) | Check existing submissions | Reject at submission time | Error message |
| 36 | Multiple PRs from same repo | Same owner/repo, different PR# | Allow, analyze independently | Normal scoring |
| 37 | User edits link after review | Track `updatedAt` | Disallow if `ai.analyzedAt` exists | Validation error |
| 38 | Link type change (PR → repo) | URL structure change | Re-detect type, re-analyze | Normal scoring |
| 39 | Submission after deadline | Existing validation | Already handled | Validation error |
| 40 | Link to commit vs branch vs PR | Parse URL structure | Analyze appropriate target | Normal scoring |

### Error Handling Strategy

**Three-Tier Approach**:

1. **Tier 1: Retry & Recovery**
   - Transient errors (network issues, timeouts)
   - Strategy: Exponential backoff, retry 3x
   - If successful: Continue normally
   - If failed: Move to Tier 2

2. **Tier 2: Graceful Degradation**
   - Partial data available
   - Strategy: Analyze with available data, flag for review
   - Label: `Needs_Review`
   - Notes: Explain what data was unavailable

3. **Tier 3: Fail with Context**
   - Unrecoverable errors (private repo, deleted PR)
   - Strategy: Store error details, inform sponsor
   - Label: `Inaccessible` or `Needs_Review`
   - Notes: Clear explanation for sponsor

**Error Storage Format**:
```typescript
submission.ai.error = {
  code: 'github_404' | 'rate_limit' | 'timeout' | 'ai_service_down',
  message: 'Human-readable explanation',
  type: 'github_api' | 'rate_limit' | 'timeout' | 'ai_service',
  timestamp: ISO string,
  retryCount: number
}
```

---

## Scoring Algorithm Design

### Pull Request Scoring Rubric (Detailed)

#### **1. Code Quality (30% weight)**

**Sub-components**:

**A. Complexity Analysis (40% of code quality)**
- **Cyclomatic Complexity** (assessed by AI):
  - Simple functions (<10 complexity): 100 points
  - Moderate (10-20): 80 points
  - Complex (20-40): 60 points
  - Very complex (>40): 40 points
- **Function Length**:
  - <50 lines: 100 points
  - 50-100 lines: 80 points
  - 100-200 lines: 60 points
  - >200 lines: 40 points
- **Nesting Depth**:
  - <3 levels: 100 points
  - 3-5 levels: 70 points
  - >5 levels: 40 points

**B. Code Style & Consistency (30% of code quality)**
- Consistent formatting across files
- Proper naming conventions (camelCase, PascalCase)
- Comment quality and relevance
- **AI Assessment**: 0-100 based on style guidelines

**C. Best Practices (30% of code quality)**
- Error handling present
- Security considerations (no hardcoded secrets)
- Performance optimizations
- **AI Checklist Evaluation**: 0-100

**Formula**:
```
codeQualityScore = (
  0.4 × complexityScore +
  0.3 × styleScore +
  0.3 × bestPracticesScore
)
```

#### **2. PR Size & Scope (15% weight)**

**Optimal Range**: 100-500 lines changed

**Scoring Table**:

| Total Lines | Size Category | Score | Reasoning |
|-------------|---------------|-------|-----------|
| <10 | Too small | 40 | Minimal effort or trivial change |
| 10-99 | Small | 70 | Acceptable but limited scope |
| 100-500 | **Optimal** | 100 | Best for review and quality |
| 501-1000 | Large | 85 | Still manageable |
| 1001-2000 | Very large | 60 | Difficult to review thoroughly |
| >2000 | Too large | 30 | Should be split into smaller PRs |

**Formula**:
```python
totalLines = additions + deletions

if totalLines < 10:
    sizeScore = 40
elif 10 <= totalLines < 100:
    sizeScore = 70
elif 100 <= totalLines <= 500:
    sizeScore = 100
elif 500 < totalLines <= 1000:
    sizeScore = 85
elif 1000 < totalLines <= 2000:
    sizeScore = 60
else:
    sizeScore = 30
```

#### **3. Commit Quality (10% weight)**

**Sub-components**:

**A. Commit Message Quality (50%)**
- Conventional commits format (feat:, fix:, docs:, etc.)
- Descriptive and clear (not just "update" or "fix")
- Proper grammar and spelling
- **AI Assessment**: 0-100

**B. Commit Count (30%)**

| Commits | Score | Reasoning |
|---------|-------|-----------|
| 1 | 50 | Might be squashed (acceptable) |
| 2-10 | 100 | Optimal granularity |
| 11-25 | 85 | Acceptable |
| 26-50 | 70 | Possibly too granular |
| >50 | 60 | Messy history |

**C. Commit Size Distribution (20%)**
- Even distribution better than one massive commit
- Calculate standard deviation of commit sizes
- Lower deviation = higher score

**Formula**:
```
commitQualityScore = (
  0.5 × messageQualityScore +
  0.3 × commitCountScore +
  0.2 × distributionScore
)
```

#### **4. Documentation (15% weight)**

**Sub-components**:

**A. PR Description Quality (60%)**
- Has description: +50 base points
- Quality factors:
  - Explains what changed and why
  - Includes context and motivation
  - Links to relevant issues/discussions
  - Screenshots/examples (if applicable)
- **AI Assessment**: 0-100

**B. Code Comments (40%)**
- Appropriate inline comments
- Complex logic explained
- Public API documented
- Avoid obvious/redundant comments
- **AI Assessment**: 0-100

**Formula**:
```
documentationScore = (
  0.6 × descriptionScore +
  0.4 × codeCommentsScore
)
```

#### **5. Testing (15% weight)**

**Sub-components**:

**A. Test Presence (50%)**
- Includes test files: 100 points
- No test files: 0 points
- Detection: Files in `test/`, `__tests__/`, `*.test.*`, `*.spec.*`

**B. Test Coverage (30%)**
- **Estimated coverage**:
  ```
  testLines = sum of additions in test files
  codeLines = sum of additions in non-test files
  coverage = testLines / (testLines + codeLines)
  ```
- Scoring:
  - >50% test changes: 100 points
  - 30-50%: 85 points
  - 20-30%: 70 points
  - 10-20%: 50 points
  - <10%: 30 points

**C. Test Quality (20%)**
- AI assessment of:
  - Test comprehensiveness
  - Edge cases covered
  - Proper assertions
  - Mock/stub usage
- Scoring: 0-100

**Formula**:
```
testingScore = (
  0.5 × testPresenceScore +
  0.3 × testCoverageScore +
  0.2 × testQualityScore
)
```

#### **6. Review Feedback (5% weight)**

**Factors**:
- **Approved reviews**: +20 points each (max 100)
- **Change requests**: -10 points each
- **Comments addressed**: +10 points
- **Reviewer engagement**: Active discussion = positive signal

**Formula**:
```
reviewScore = min(100, (
  (approvedCount × 20) -
  (changesRequestedCount × 10) +
  (commentsAddressedCount × 10)
))
```

**Note**: If PR is from submitter's fork (bounty submission), existing reviews may not be available. In this case, assign neutral score (60 points).

#### **7. CI/CD Status (5% weight)**

**Scoring**:

| CI Status | Score | Notes |
|-----------|-------|-------|
| All checks passing | 100 | Green checkmark |
| Some checks failing | 50 | Partial success |
| All checks failing | 0 | Red X |
| No CI/CD | 60 | Neutral (not penalized) |
| CI in progress | 50 | Pending state |

**Detection**:
- Use GitHub Checks API: `GET /repos/{owner}/{repo}/commits/{sha}/check-runs`
- Status: `success`, `failure`, `neutral`, `cancelled`, `timed_out`, `action_required`, `stale`

#### **8. Alignment with Bounty (5% weight)**

**Evaluation**:
- AI compares PR against bounty context:
  - `listing.title`
  - `listing.description`
  - `listing.requirements`
  - `listing.skills`
- Semantic similarity analysis
- Technical requirement matching

**Scoring Factors**:
- Keywords match: 30%
- Required features implemented: 40%
- Technical approach suitable: 30%

**Output**: 0-100 score

---

### **Overall PR Score Calculation**

```python
def calculate_overall_pr_score(component_scores):
    weights = {
        'codeQuality': 0.30,
        'prSize': 0.15,
        'commitQuality': 0.10,
        'documentation': 0.15,
        'testing': 0.15,
        'reviewFeedback': 0.05,
        'ciStatus': 0.05,
        'alignment': 0.05
    }

    overall = sum(
        component_scores[component] * weights[component]
        for component in component_scores
    )

    return round(overall, 2)
```

**Example Calculation**:
```
Component Scores:
- Code Quality: 85
- PR Size: 100 (350 lines)
- Commit Quality: 90
- Documentation: 80
- Testing: 70
- Review Feedback: 60 (no external reviews)
- CI Status: 100 (all passing)
- Alignment: 85

Overall = (85×0.30) + (100×0.15) + (90×0.10) + (80×0.15) +
          (70×0.15) + (60×0.05) + (100×0.05) + (85×0.05)
        = 25.5 + 15 + 9 + 12 + 10.5 + 3 + 5 + 4.25
        = 84.25

Label: High_Quality (70-84 range)
```

---

### Repository Scoring Rubric (Detailed)

#### **1. Code Quality & Structure (25% weight)**

**Sub-components**:

**A. Project Organization (40%)**
- Proper directory structure (src/, lib/, tests/, docs/)
- Separation of concerns
- Modular architecture
- Config files in root (package.json, tsconfig.json, etc.)
- **AI Assessment**: 0-100

**B. Code Complexity (30%)**
- Average file complexity across repository
- Maintainability index
- **AI Assessment**: 0-100

**C. Code Consistency (30%)**
- Consistent style across all files
- Linting/formatting setup (.eslintrc, .prettierrc)
- **AI Assessment**: 0-100

#### **2. Documentation (20% weight)**

**Sub-components**:

**A. README Quality (60%)**

| Criterion | Points |
|-----------|--------|
| README exists | 50 (base) |
| Has description | +10 |
| Installation instructions | +10 |
| Usage examples | +10 |
| API documentation | +10 |
| Screenshots/demos | +5 |
| Badges (build, coverage, etc.) | +5 |

**Total**: 0-100

**B. Code Documentation (30%)**
- Inline comments
- JSDoc/docstrings for public APIs
- Code examples
- **AI Assessment**: 0-100

**C. Additional Documentation (10%)**

| File | Points |
|------|--------|
| CONTRIBUTING.md | +30 |
| LICENSE | +30 |
| CHANGELOG.md | +20 |
| CODE_OF_CONDUCT.md | +10 |
| .github/ISSUE_TEMPLATE | +10 |

**Total**: 0-100

#### **3. Testing & QA (20% weight)**

**Sub-components**:

**A. Test Coverage (60%)**

| Test Presence | Score |
|---------------|-------|
| Test files present | 40 (base) |
| Test-to-code ratio >30% | +30 |
| Test-to-code ratio >50% | +30 (total 100) |

**B. CI/CD Setup (40%)**

| CI/CD Feature | Score |
|---------------|-------|
| Has CI/CD config (GitHub Actions, CircleCI, etc.) | 70 |
| Runs tests automatically | +15 |
| Code coverage reporting | +15 |
| No CI/CD | 30 (neutral) |

#### **4. Activity & Maintenance (15% weight)**

**Sub-components**:

**A. Commit Frequency (50%)**

| Last Commit | Score | Status |
|-------------|-------|--------|
| <30 days ago | 100 | Very active |
| 30-90 days | 80 | Active |
| 3-6 months | 60 | Moderate |
| 6-12 months | 40 | Low activity |
| >12 months | 20 | Inactive |

**B. Commit Count (30%)**

| Total Commits | Score | Maturity |
|---------------|-------|----------|
| <10 | 40 | New project |
| 10-50 | 70 | Developing |
| 50-200 | 100 | Mature |
| >200 | 90 | Very mature |

**C. Issue/PR Activity (20%)**

| Activity | Score |
|----------|-------|
| Active issues/PRs in last 30 days | 100 |
| Activity in last 90 days | 70 |
| No recent activity | 30 |

#### **5. Community & Adoption (10% weight)**

**Sub-components**:

**A. Stars (40%)**

| Stars | Score | Popularity |
|-------|-------|------------|
| <10 | 30 | Minimal |
| 10-50 | 60 | Growing |
| 50-100 | 80 | Popular |
| 100-500 | 90 | Very popular |
| >500 | 100 | Highly popular |

**B. Forks (30%)**

| Forks | Score |
|-------|-------|
| <5 | 40 |
| 5-20 | 70 |
| 20-50 | 90 |
| >50 | 100 |

**C. Contributors (30%)**

| Contributors | Score | Type |
|--------------|-------|------|
| 1 | 50 | Solo project |
| 2-5 | 80 | Small team |
| >5 | 100 | Community project |

#### **6. Technical Stack (5% weight)**

**Sub-components**:

**A. Language Alignment (60%)**
- Matches bounty required language: 100
- Partially matches: 60
- Different language: 30

**B. Modern Practices (40%)**
- Uses current libraries/frameworks (not deprecated)
- Proper dependency management
- Modern language features
- **AI Assessment**: 0-100

#### **7. License & Legal (5% weight)**

**Sub-components**:

**A. License Presence (70%)**

| License | Score |
|---------|-------|
| OSI-approved license (MIT, Apache, GPL) | 100 |
| Custom permissive license | 70 |
| No license | 30 |

**B. License Compatibility (30%)**
- Matches bounty requirements (if specified)
- Compatible with common use cases
- **Assessment**: 0-100

---

### **Overall Repository Score Calculation**

```python
def calculate_overall_repo_score(component_scores, listing_context):
    weights = {
        'codeQuality': 0.25,
        'documentation': 0.20,
        'testing': 0.20,
        'activity': 0.15,
        'community': 0.10,
        'techStack': 0.05,
        'legal': 0.05
    }

    base_score = sum(
        component_scores[component] * weights[component]
        for component in component_scores
    )

    # Alignment modifier (multiplicative)
    alignment = analyze_alignment(repo_data, listing_context)
    overall = base_score * (0.7 + 0.3 * alignment / 100)

    return round(overall, 2)
```

---

### Label Assignment Logic

```python
def assign_label(overall_score, flags):
    # Priority 1: Error flags
    if flags.get('inaccessible'):
        return 'Inaccessible'

    # Priority 2: Manual review flags
    if (flags.get('needsManualReview') or
        flags.get('isTooLarge') or
        flags.get('isIncomplete')):
        return 'Needs_Review'

    # Priority 3: Score-based assignment
    if overall_score >= 85:
        return 'Shortlisted'
    elif overall_score >= 70:
        return 'High_Quality'
    elif overall_score >= 50:
        return 'Mid_Quality'
    elif overall_score >= 30:
        return 'Low_Quality'
    else:
        return 'Low_Quality'  # Recommend rejection
```

---

## Files to Create/Modify

### **Files in earn Service (Public Repo)**

#### **New Files** (5 files)

1. **`/src/lib/github.ts`** (200 lines)
   - GitHub API client wrapper using Octokit.js
   - Token rotation logic
   - Rate limit monitoring
   - Error handling utilities

2. **`/src/features/submissions/utils/githubUrlParser.ts`** (150 lines)
   - `isGitHubUrl(url: string): boolean`
   - `parseGitHubUrl(url: string): GitHubMeta | null`
   - `detectGitHubType(url: string): 'pr' | 'repo' | null`
   - URL normalization utilities

3. **`/src/features/submissions/types/githubSubmission.ts`** (100 lines)
   - TypeScript interfaces:
     - `GitHubSubmissionAi`
     - `GitHubMeta`
     - `PRData`
     - `RepoData`
     - `AIEvaluation`

4. **`/src/app/api/sponsor-dashboard/submissions/github-stats/route.ts`** (120 lines)
   - `GET /api/sponsor-dashboard/submissions/github-stats?id={submissionId}`
   - Fetch and return GitHub stats from `submission.ai`
   - Cache response (5 min TTL)

5. **`/src/features/sponsor-dashboard/components/Submissions/Modals/AiReviewGitHub.tsx`** (400 lines)
   - React component for GitHub AI review modal
   - States: DISCLAIMER → INIT → PROCESSING → DONE
   - Display GitHub stats and review results
   - Follow existing modal patterns

#### **Modified Files** (5 files)

1. **`/src/features/agents/utils/queueAgent.ts`**
   - Add: `type AgentActionType = ... | 'autoReviewGitHubSubmission'`
   - ~5 lines changed

2. **`/src/pages/api/submission/create.ts`**
   - Add GitHub URL detection after validation
   - Queue GitHub review job if detected
   - Store GitHub metadata in `submission.ai`
   - ~30 lines added

3. **`/src/features/listings/utils/submissionFormSchema.ts`**
   - Add optional GitHub URL validation
   - Display helper text for public repos
   - ~15 lines added

4. **`/src/app/api/sponsor-dashboard/submissions/ai/unreviewed/route.ts`**
   - Include GitHub submissions in query
   - Return GitHub metadata
   - ~10 lines changed

5. **`/src/app/api/sponsor-dashboard/submissions/ai/commit-reviewed/route.ts`**
   - Handle GitHub label conversion
   - AI notes → submission.notes (HTML)
   - Set `ai.commited = true`
   - ~20 lines added

---

### **Files in earn-agent Service (Private Repo - requires access)**

#### **New Files** (6 files)

1. **`/src/workers/github/queries/pr.ts`** (150 lines)
   - GraphQL query builder for PR data
   - Fetch: title, body, files, commits, reviews, CI status
   - Pagination handling

2. **`/src/workers/github/queries/repo.ts`** (120 lines)
   - GraphQL query builder for repository data
   - Fetch: metadata, languages, contributors, README
   - Statistics queries

3. **`/src/workers/github/fetchPRData.ts`** (200 lines)
   - `fetchPRData(owner, repo, number): Promise<PRData>`
   - Execute GraphQL queries
   - Normalize and extract data
   - Error handling

4. **`/src/workers/github/fetchRepoData.ts`** (180 lines)
   - `fetchRepoData(owner, repo): Promise<RepoData>`
   - Fetch repository statistics
   - Parse README content
   - Error handling

5. **`/src/workers/github/analyzer.ts`** (300 lines)
   - Main analysis orchestrator
   - `analyzePRWithAI(prData, listingContext): Promise<AIEvaluation>`
   - `analyzeRepoWithAI(repoData, listingContext): Promise<AIEvaluation>`
   - Integrate scoring + AI analysis
   - Build AI prompts
   - Call OpenRouter API
   - Parse and validate AI response

6. **`/src/workers/handlers/autoReviewGitHubSubmission.ts`** (250 lines)
   - Main worker handler for GitHub review jobs
   - Job flow:
     1. Fetch submission from database
     2. Fetch listing context
     3. Call GitHub data fetcher
     4. Call analyzer
     5. Store results in database
     6. Handle errors and edge cases

#### **Modified Files** (1 file)

1. **`/src/workers/index.ts`** (or wherever worker handlers are registered)
   - Register `autoReviewGitHubSubmission` handler
   - ~10 lines added

---

### **Total Implementation Scope**

- **New files**: 11 files (~2,370 lines of code)
- **Modified files**: 6 files (~90 lines changed)
- **Total**: ~2,500 lines of production code
- **Tests**: ~1,500 lines of test code (not included in count above)

---

## Success Criteria

### Functional Requirements

| # | Requirement | Acceptance Criteria | Measurement |
|---|-------------|---------------------|-------------|
| 1 | End-to-end workflow | User submits GitHub URL → AI reviews → Sponsor sees results | Manual testing: 10+ submissions |
| 2 | PR analysis | System can analyze any public PR | Test with 20+ real PRs |
| 3 | Repository analysis | System can analyze any public repo | Test with 20+ real repos |
| 4 | Error handling | All 40+ edge cases handled gracefully | Unit tests + integration tests |
| 5 | Dashboard integration | GitHub reviews visible in sponsor dashboard | UI testing |
| 6 | Label assignment | AI assigns appropriate labels | Validation against manual reviews |

### Performance Requirements

| Metric | Target | Threshold | Measurement Method |
|--------|--------|-----------|-------------------|
| **Processing Time** | <20 seconds | <30 seconds | Monitor worker processing time |
| **Typical PR (<500 files)** | 15 seconds avg | 25 seconds max | Performance testing |
| **Large PR (500-1000 files)** | 30 seconds avg | 45 seconds max | Performance testing |
| **Repository** | 25 seconds avg | 40 seconds max | Performance testing |
| **GitHub API Rate Limit** | <50% usage | <70% usage | Monitor rate limit headers |
| **Error Rate** | <2% | <5% | Track failed jobs / total jobs |
| **Queue Throughput** | 6 submissions/min | 4 submissions/min | Monitor queue metrics |

### Accuracy Requirements

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| **AI vs Manual Agreement** | >80% | Test against 100 manually reviewed submissions |
| **Label Accuracy** | >75% exact match | Compare AI label vs final sponsor decision |
| **False Positives** (Shortlisted but Low Quality) | <5% | Track sponsor overrides |
| **False Negatives** (Low Quality but actually good) | <10% | Track sponsor overrides |
| **Confidence Threshold** | Uncertain cases → `Needs_Review` | >90% of edge cases flagged |

### Reliability Requirements

| Metric | Target | Measurement Method |
|--------|--------|-------------------|
| **Uptime** | >99% | Monitor worker availability |
| **Job Success Rate** | >95% | Track successful completions |
| **Retry Success Rate** | >90% of retries succeed | Monitor retry outcomes |
| **Error Recovery** | All errors logged with context | Review error logs |

### Testing Requirements

| Test Type | Coverage | Details |
|-----------|----------|---------|
| **Unit Tests** | >80% code coverage | Test individual functions |
| **Integration Tests** | Key workflows | Submission creation → Review → Display |
| **Edge Case Tests** | All 40+ scenarios | One test per edge case |
| **Performance Tests** | Load testing | Simulate 100 concurrent submissions |
| **Manual Testing** | 20+ real submissions | Various PR/repo types |

### Documentation Requirements

| Document | Status | Location |
|----------|--------|----------|
| API Documentation | Complete | `/docs/api/github-review.md` |
| Scoring Methodology | Complete | This document |
| Deployment Guide | Complete | `/docs/deployment/github-agent.md` |
| Troubleshooting Guide | Complete | `/docs/troubleshooting.md` |
| Environment Variables | Complete | `.env.example` updated |

---

## Risk Assessment & Mitigation

### Risk Matrix

| # | Risk | Probability | Impact | Severity | Mitigation Strategy |
|---|------|-------------|--------|----------|---------------------|
| 1 | earn-agent service access not granted | High | High | **CRITICAL** | Contact team immediately; blockers for Week 2 |
| 2 | GitHub API rate limits exceeded | Medium | Medium | **MEDIUM** | 5-token rotation; monitor usage; caching |
| 3 | AI scoring accuracy below target | Medium | High | **HIGH** | Iterative prompt engineering; test data validation |
| 4 | Large PRs cause timeout | Medium | Low | **LOW** | Sampling strategy; timeout handling; flag for manual |
| 5 | Performance degradation at scale | Low | Medium | **MEDIUM** | Load testing; optimization; horizontal scaling |
| 6 | GitHub API downtime | Low | High | **MEDIUM** | Retry logic; queue for delayed processing; monitoring |
| 7 | Security vulnerabilities in code analysis | Low | High | **MEDIUM** | Sanitize inputs; limit AI model access; code review |
| 8 | Database schema changes needed | Low | Medium | **LOW** | Flexible JSON field; avoid migrations |

### Detailed Risk Mitigation Plans

#### **Risk 1: earn-agent Service Access (CRITICAL)**

**Description**: The bounty mentions access to the earn-agent service, but this is a private repository not available on GitHub.

**Mitigation**:
1. **Immediate Action**: Contact Superteam team (Abhishek, Jayesh, Pratik) via:
   - Bounty submission platform
   - Email/Discord if available
   - Request access to earn-agent repository
2. **Contingency Plan**:
   - If access delayed: Implement earn service components first (Week 1)
   - If no access granted: Propose alternative:
     - Implement GitHub review logic in earn service temporarily
     - Migrate to earn-agent later when access granted
3. **Timeline Impact**:
   - With access: On schedule
   - 2-3 day delay: Still achievable in 3 weeks
   - >5 day delay: Request timeline extension

**Status**: Requires immediate action upon bounty acceptance.

#### **Risk 2: GitHub API Rate Limits**

**Description**: GitHub API has strict rate limits (5,000 requests/hour per token). High submission volume could exhaust limits.

**Mitigation**:
1. **Token Rotation Strategy**:
   - Maintain pool of 5 Personal Access Tokens
   - Effective limit: 25,000 requests/hour
   - Rotate when `X-RateLimit-Remaining < 100`
2. **Caching**:
   - Cache repository metadata for 24 hours
   - Use ETag headers for conditional requests (304 responses don't count)
   - Redis cache for frequently accessed data
3. **Request Optimization**:
   - Use GraphQL instead of REST (fewer requests)
   - Batch related queries
   - Avoid polling; use webhooks where possible
4. **Monitoring**:
   - Alert when usage >70% of limit
   - Track requests per submission (optimize high-usage paths)
5. **Fallback**:
   - If all tokens exhausted: Queue submissions for retry in 1 hour
   - Label temporarily as `Needs_Review`
   - Notify sponsor of delay

**Expected Usage**:
- Typical PR: ~5 API requests (with GraphQL)
- Repository: ~8 API requests
- 1000 submissions/day = ~6,000 requests/day (~250/hour)
- Well within limits with token rotation

#### **Risk 3: AI Scoring Accuracy**

**Description**: AI-generated scores may not align with sponsor expectations, leading to low trust in the system.

**Mitigation**:
1. **Iterative Prompt Engineering**:
   - Start with baseline prompt
   - Test against 50 manually reviewed submissions
   - Refine prompt based on discrepancies
   - Iterate 3-5 times to optimize
2. **Validation Dataset**:
   - Request access to historical GitHub bounty submissions
   - If not available: Create synthetic test set (20 PRs with manual scores)
   - Measure accuracy: % agreement within 1 label difference
3. **Safety Nets**:
   - Conservative labeling: When uncertain → `Needs_Review`
   - Flag edge cases for manual review
   - Transparency: Always show score breakdown to sponsor
4. **Continuous Improvement**:
   - Track sponsor overrides (AI said X, sponsor chose Y)
   - Monthly review of override patterns
   - Update prompts and scoring weights based on feedback
5. **Hybrid Approach**:
   - AI provides recommendation
   - Sponsor always has final decision
   - Emphasis: "AI assistant, not replacement for human judgment"

**Acceptance Criteria**:
- >80% agreement with manual reviews
- <5% false positives (Shortlisted but actually low quality)

#### **Risk 4: Large PRs Cause Timeout**

**Description**: PRs with >1000 files may take too long to analyze, causing worker timeout.

**Mitigation**:
1. **Sampling Strategy**:
   - For PRs >1000 files:
     - Analyze first 50 files
     - Include all test files
     - Include files matching bounty keywords
   - Set flag: `isTooLarge: true`
   - Label: `Needs_Review`
   - Note: "Large PR analyzed via sampling. Manual verification recommended."
2. **Timeout Handling**:
   - Set worker timeout: 5 minutes per submission
   - Per-operation timeouts:
     - GitHub API call: 30 seconds
     - AI analysis: 2 minutes
   - If timeout: Cancel, store partial results, flag for manual review
3. **Progressive Processing**:
   - Option: Analyze in chunks, aggregate results
   - More complex but handles very large PRs
4. **Submission Guidelines**:
   - Add to bounty description: "PRs should be <1000 files for automated review"
   - Encourage splitting large contributions

**Expected Frequency**: <5% of submissions based on typical PR sizes.

#### **Risk 5: Performance Degradation at Scale**

**Description**: System may slow down with 1000+ submissions/day or concurrent spikes.

**Mitigation**:
1. **Load Testing**:
   - Simulate 100 concurrent submissions
   - Measure: Queue length, processing time, error rate
   - Identify bottlenecks
2. **Horizontal Scaling**:
   - BullMQ supports multiple workers
   - Deploy 3-5 worker instances (Railway allows horizontal scaling)
   - Each worker processes 6 submissions/min = 18-30 total
3. **Database Optimization**:
   - Add index on `submission.listingId` and `submission.label`
   - Optimize queries (use Prisma query optimization)
4. **Caching**:
   - Cache repository metadata (reduces GitHub API calls)
   - Cache listing context (avoid repeated DB queries)
5. **Priority Queue**:
   - High-value bounties get higher priority
   - Deadline-sensitive submissions processed first

**Capacity Planning**:
- 1 worker: 360 submissions/hour
- 3 workers: 1,080 submissions/hour
- 1000 submissions/day = ~42/hour avg, ~200/hour peak
- **Conclusion**: 2-3 workers sufficient with headroom

#### **Risk 6: GitHub API Downtime**

**Description**: GitHub API may experience outages, preventing submission analysis.

**Mitigation**:
1. **Retry Logic**:
   - On 500/503 errors: Retry every 15 minutes for 2 hours
   - Exponential backoff for transient errors
2. **Graceful Degradation**:
   - Display banner in UI: "GitHub reviews temporarily delayed"
   - Queue all jobs for retry when API recovers
3. **Monitoring**:
   - Subscribe to GitHub status page (https://www.githubstatus.com)
   - Alert on persistent API errors (>10% error rate)
4. **Fallback**:
   - After 2 hours: Label as `Needs_Review`
   - Notify sponsor: "Unable to analyze due to GitHub API issues. Manual review required."

**Historical Uptime**: GitHub API uptime >99.9%, downtime rare and brief.

#### **Risk 7: Security Vulnerabilities**

**Description**: Analyzing user-submitted code could expose system to malicious input.

**Mitigation**:
1. **Input Sanitization**:
   - Validate all GitHub URLs before processing
   - Use URL parser (not regex alone) to prevent injection
2. **Sandboxing**:
   - Never execute submitted code
   - AI analysis is read-only
   - No direct code compilation or running
3. **API Key Security**:
   - Store GitHub tokens in environment variables (encrypted)
   - Never log token values
   - Rotate tokens every 90 days
4. **Rate Limiting**:
   - Prevent abuse: Max 10 submissions per user per bounty
   - Existing earn platform handles this
5. **Code Review**:
   - Security review of all GitHub URL parsing logic
   - Dependency audit (npm audit)

#### **Risk 8: Database Schema Changes**

**Description**: May need to modify Prisma schema if JSON field insufficient.

**Mitigation**:
1. **Flexible JSON Field**:
   - Current `submission.ai` field (Json type) is flexible
   - Can store any structure without migration
   - TypeScript interfaces for type safety
2. **Avoid Migrations**:
   - Use JSON field for all GitHub-specific data
   - No new columns needed
3. **Indexing** (if needed):
   - Can add GIN index on JSON field for queries
   - Not required for MVP

**Conclusion**: Low risk, existing schema is sufficient.

---

## Cost & Resource Estimates

### Infrastructure Costs (Monthly)

| Service | Usage | Cost | Notes |
|---------|-------|------|-------|
| **GitHub API** | 25,000 requests/hour (with 5 tokens) | $0 | Free for public repositories |
| **OpenRouter API** | 1,000 submissions × 2,000 tokens/submission | $50-70 | Depends on model (Gemini Flash: ~$0.05/submission) |
| **Redis (BullMQ)** | Managed Redis (Upstash/Railway) | $20 | Existing infrastructure |
| **Railway (earn-agent workers)** | 3 worker instances | $30 | Based on Railway pricing |
| **Database (MySQL)** | No additional cost | $0 | Existing Prisma/MySQL setup |
| **Monitoring** | Log aggregation, error tracking | $10 | Optional: Sentry, Datadog |
| **Total** | - | **$110-130/month** | For 1,000 submissions/month |

**Scaling**:
- 5,000 submissions/month: ~$450/month (AI costs dominate)
- 10,000 submissions/month: ~$850/month

### Development Resources

| Phase | Duration | Effort | Resource Type |
|-------|----------|--------|---------------|
| Week 1: Infrastructure | 7 days | 40-50 hours | Senior Full-Stack Engineer |
| Week 2: AI Engine | 7 days | 40-50 hours | AI/ML Engineer or Senior Engineer |
| Week 3: Integration | 7 days | 30-40 hours | Frontend + Backend Engineer |
| **Total** | 21 days | **110-140 hours** | 1 Senior Engineer (full-time) |

**Additional Resources**:
- Access to earn-agent repository (Superteam team)
- GitHub PAT tokens (5 tokens, free)
- OpenRouter API key (provided or new account)
- Testing environment (staging instance of earn)

### Budget Breakdown

**Assuming Quote-Based Compensation**:

| Component | Estimated Cost | Notes |
|-----------|---------------|-------|
| Development (110-140 hours) | TBD | Based on hourly rate or fixed project fee |
| Infrastructure (first month) | $130 | Included in project budget |
| Testing & QA | Included | Part of development timeline |
| Documentation | Included | Part of Week 3 deliverables |
| Post-launch support (1 month) | Optional | Bug fixes, prompt optimization |

**Recommended Compensation Model**:
1. **Fixed Project Fee**: Based on 140 hours of senior engineering work
2. **Milestones**:
   - 30% upon acceptance and Week 1 completion
   - 40% upon Week 2 completion (AI engine working)
   - 30% upon final delivery and testing
3. **Post-Launch Support**: Optional hourly rate for 1 month

---

## Monitoring & Maintenance

### Key Metrics to Track

#### **1. Performance Metrics**

| Metric | Target | Alert Threshold | Dashboard |
|--------|--------|----------------|-----------|
| **Average Processing Time** | <20s | >30s | Real-time chart |
| **P95 Processing Time** | <30s | >45s | Histogram |
| **Queue Length** | <10 | >100 | Gauge |
| **Worker Throughput** | 6/min/worker | <4/min | Counter |
| **Error Rate** | <2% | >5% | Percentage chart |

#### **2. GitHub API Metrics**

| Metric | Target | Alert Threshold | Action |
|--------|--------|-----------------|--------|
| **Rate Limit Usage** | <50% | >70% | Switch token |
| **Rate Limit Reset Time** | - | - | Display countdown |
| **Token Rotation Count** | - | - | Track for optimization |
| **API Error Rate** | <1% | >3% | Investigate GitHub status |
| **Secondary Rate Limits** | 0 | >5/hour | Reduce concurrency |

#### **3. AI Analysis Metrics**

| Metric | Target | Alert Threshold | Action |
|--------|--------|-----------------|--------|
| **AI Service Response Time** | <5s | >10s | Check OpenRouter status |
| **AI Service Error Rate** | <1% | >3% | Switch model or retry |
| **Token Usage per Submission** | ~2,000 | >5,000 | Optimize prompts |
| **AI vs Sponsor Agreement** | >80% | <70% | Review prompt quality |

#### **4. Business Metrics**

| Metric | Measurement | Purpose |
|--------|-------------|---------|
| **Total GitHub Submissions** | Count | Track adoption |
| **PR vs Repo Ratio** | Percentage | Understand submission types |
| **Label Distribution** | Breakdown | Validate scoring is balanced |
| **Sponsor Override Rate** | % changed labels | Measure AI accuracy |
| **Edge Case Frequency** | Count by type | Prioritize improvements |

### Monitoring Stack

**Recommended Setup**:

```
┌─────────────────────────────────────────────────────────────┐
│                     MONITORING STACK                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐  │
│  │   Logs       │    │   Metrics    │    │   Alerts     │  │
│  │              │    │              │    │              │  │
│  │ • Winston    │    │ • Prometheus │    │ • Slack      │  │
│  │ • Pino       │    │ • Grafana    │    │ • Email      │  │
│  │ • Datadog    │    │ • StatsD     │    │ • PagerDuty  │  │
│  └──────────────┘    └──────────────┘    └──────────────┘  │
│                                                              │
│  ┌──────────────────────────────────────────────────────┐  │
│  │              ERROR TRACKING                           │  │
│  │                                                        │  │
│  │  • Sentry (errors with context)                       │  │
│  │  • Source maps for stack traces                       │  │
│  │  • User context (submission ID, listing ID)          │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### Alert Configuration

**Critical Alerts** (Immediate Response):
1. GitHub API rate limit >90% on all tokens
2. Worker error rate >10% for 5+ minutes
3. Queue length >500 (system overwhelmed)
4. AI service completely down (all retries failing)

**Warning Alerts** (Review within 1 hour):
1. Rate limit >70% on any token
2. Processing time P95 >45s
3. Error rate >5% for 10+ minutes
4. Sponsor override rate >40% (AI accuracy issue)

**Info Alerts** (Daily Summary):
1. Total submissions processed
2. Label distribution
3. Average scores by component
4. Edge case frequency

### Maintenance Schedule

#### **Daily**
- Review error logs (5 minutes)
- Check GitHub rate limit usage (2 minutes)
- Monitor queue health (2 minutes)

#### **Weekly**
- Review AI accuracy metrics (sponsor overrides)
- Analyze edge case patterns
- Check for GitHub API changes/deprecations
- Review performance trends

#### **Monthly**
- Deep dive on AI accuracy (compare AI vs manual reviews)
- Optimize AI prompts if accuracy <80%
- Review and rotate GitHub PAT tokens
- Update dependencies (npm, Prisma, etc.)
- Capacity planning (is scaling needed?)

#### **Quarterly**
- Comprehensive performance review
- Cost optimization analysis
- Feature enhancement planning
- Security audit (dependency vulnerabilities)

### Post-Launch Optimization Plan

**Month 1: Stability & Accuracy**
- Focus on bug fixes and edge case handling
- Collect sponsor feedback on AI reviews
- Iterate on AI prompts to improve accuracy
- Monitor performance under real-world load

**Month 2: Performance**
- Optimize slow queries and API calls
- Implement advanced caching strategies
- Fine-tune worker concurrency
- Reduce AI token usage (prompt optimization)

**Month 3: Features**
- Add webhook support for real-time PR updates
- Implement code complexity analysis (static analysis)
- Add code similarity detection (plagiarism check)
- Enhance dashboard with charts and graphs

**Ongoing**:
- Monthly review of AI accuracy
- Quarterly security audits
- Continuous monitoring and alerting refinement

---

## Deliverables

### Primary Deliverables

1. **Functional GitHub Auto-Review System**
   - Complete implementation (earn service + earn-agent service)
   - All 40+ edge cases handled
   - Integrated with sponsor dashboard
   - Tested and deployed to production

2. **Scoring Algorithm**
   - PR scoring rubric (8 components)
   - Repository scoring rubric (7 components)
   - AI-powered analysis prompts
   - Label assignment logic

3. **Documentation**
   - API endpoint documentation
   - Scoring methodology explanation
   - Deployment guide for earn-agent updates
   - Troubleshooting guide
   - Environment variables reference

4. **Testing Suite**
   - Unit tests (>80% coverage)
   - Integration tests (key workflows)
   - Edge case tests (40+ scenarios)
   - Performance test results

5. **Monitoring Setup**
   - Metrics dashboard configuration
   - Alert rules and thresholds
   - Error tracking integration
   - Performance monitoring

### Secondary Deliverables

1. **Knowledge Transfer**
   - Code walkthrough session (1 hour)
   - Q&A documentation
   - Video tutorial for maintenance (optional)

2. **Post-Launch Support**
   - 1 month of bug fixes (included)
   - Prompt optimization based on real data
   - Performance tuning

3. **Recommendations Document**
   - Future enhancements (webhooks, advanced analysis)
   - Scaling strategy for >10k submissions/month
   - Cost optimization opportunities

---

## Conclusion

This proposal outlines a comprehensive, production-ready solution for GitHub PR and repository auto-review on the Superteam Earn platform. The approach leverages existing infrastructure, follows established patterns, and prioritizes reliability, accuracy, and scalability.

**Key Strengths**:
1. **Thorough Research**: Deep dive into existing earn codebase and architecture
2. **Comprehensive Edge Case Handling**: 40+ scenarios documented with mitigation strategies
3. **Scalable Design**: Built on proven BullMQ + Redis queue system, can handle 1000+ submissions/day
4. **Intelligent Scoring**: Multi-component scoring algorithm with AI-powered analysis
5. **Production-Ready**: Monitoring, logging, error handling, and documentation included

**Differentiators**:
- **Senior Staff Engineer Approach**: Thinking beyond implementation to production operations
- **Risk Management**: Proactive identification and mitigation of potential issues
- **Iterative Quality**: Testing and validation at every stage
- **Long-Term Maintainability**: Clean code, comprehensive docs, monitoring setup

**Expected Impact**:
- **80%+ time savings** for sponsors reviewing GitHub submissions
- **Consistent evaluation** across all submissions
- **Faster turnaround** for bounty winners (<24 hours from submission to review)
- **Scalable foundation** for future enhancements (code quality tools, webhooks, etc.)

**Timeline Commitment**: 2-3 weeks for full implementation and testing, with ongoing support for optimization.

**Next Steps**:
1. Bounty acceptance and kickoff call
2. Access to earn-agent repository (critical)
3. Setup of GitHub PAT tokens and OpenRouter API
4. Week 1 implementation begins

---

**Thank you for considering this proposal. I'm excited about the opportunity to contribute to Superteam Earn and build a best-in-class GitHub auto-review system.**

---

## Appendix

### A. GitHub API Examples

**GraphQL Query for PR Analysis**:
```graphql
query GetPullRequest($owner: String!, $repo: String!, $number: Int!) {
  repository(owner: $owner, name: $repo) {
    pullRequest(number: $number) {
      title
      body
      state
      isDraft
      additions
      deletions
      changedFiles

      commits(first: 100) {
        nodes {
          commit {
            oid
            message
            additions
            deletions
            author {
              name
              email
              date
            }
          }
        }
      }

      files(first: 100) {
        nodes {
          path
          additions
          deletions
        }
      }

      reviews(first: 100) {
        nodes {
          state
          author {
            login
          }
          body
          createdAt
        }
      }

      mergeable
      mergeStateStatus
    }
  }
}
```

### B. AI Prompt Template

**Example Prompt for PR Analysis**:
```
You are an expert code reviewer evaluating a GitHub Pull Request for a bounty submission.

BOUNTY CONTEXT:
Title: Build authentication system for web app
Description: Implement JWT-based authentication with refresh tokens
Requirements:
- User login/logout endpoints
- Token refresh mechanism
- Password hashing
- Input validation
Required Skills: Node.js, Express, JWT, Security

PULL REQUEST DATA:
Title: Add JWT authentication with refresh tokens
Description: Implemented user authentication using JWT. Includes login, logout, and token refresh endpoints. Passwords are hashed with bcrypt. Added input validation middleware.
State: open
Changes: +487 additions, -23 deletions across 8 files
Commits: 5 commits with descriptive messages
Reviews: 0 external reviews
CI Status: All checks passing

FILES CHANGED:
1. src/auth/authController.js (+120, -5)
2. src/auth/authMiddleware.js (+85, -0) [NEW]
3. src/auth/authRoutes.js (+45, -0) [NEW]
4. tests/auth.test.js (+150, -0) [NEW]
5. src/models/User.js (+35, -8)
6. package.json (+12, -10)
7. .env.example (+5, -0)
8. README.md (+35, -0)

COMMIT MESSAGES:
1. "feat: add user authentication endpoints"
2. "feat: implement JWT token generation and validation"
3. "feat: add refresh token mechanism"
4. "test: add comprehensive auth tests"
5. "docs: update README with auth endpoints"

EVALUATE THE FOLLOWING (0-100 scale):

1. Code Quality: Is the code well-structured, maintainable, and follows best practices?
2. Alignment with Bounty: Does this PR fully address the bounty requirements?
3. Documentation: Is the PR description clear? Are code comments adequate?
4. Testing: Are tests included and comprehensive?

Provide a final recommendation: Shortlisted, High_Quality, Mid_Quality, Low_Quality, or Needs_Review

Include specific, actionable feedback (2-3 sentences) for the sponsor.

Respond in JSON format:
{
  "scores": {
    "codeQuality": 85,
    "alignment": 95,
    "documentation": 80,
    "testing": 90
  },
  "finalLabel": "Shortlisted",
  "notes": "Excellent implementation of JWT authentication with all required features. Code is clean, well-tested, and follows security best practices. Minor suggestion: Consider rate limiting on login endpoint to prevent brute force attacks.",
  "flags": {
    "needsManualReview": false,
    "isIncomplete": false
  }
}
```

### C. Database Schema Reference

**Submission AI Field Structure** (for GitHub):
```typescript
interface GitHubSubmissionAi {
  // GitHub metadata
  githubType: 'pr' | 'repo';
  githubMeta: {
    owner: string;
    repo: string;
    number?: number;
    url: string;
    detectedAt: string; // ISO timestamp
  };

  // Fetched GitHub data
  githubData?: {
    pr?: {
      title: string;
      body: string;
      state: 'open' | 'closed' | 'merged';
      isDraft: boolean;
      additions: number;
      deletions: number;
      changedFiles: number;
      commits: number;
      reviews: number;
      ciStatus: 'success' | 'failure' | 'pending' | 'neutral';
      createdAt: string;
      mergedAt?: string;
      description: string;
    };

    repo?: {
      stars: number;
      forks: number;
      language: string;
      languages: Record<string, number>;
      commits: number;
      contributors: number;
      hasTests: boolean;
      hasReadme: boolean;
      hasLicense: boolean;
      lastCommitDate: string;
    };
  };

  // AI evaluation results
  evaluation?: {
    finalLabel: SubmissionLabels;
    notes: string;
    scores: {
      codeQuality: number;
      alignmentWithBounty: number;
      documentation: number;
      testing: number;
      complexity: number;
      overall: number;
    };
    flags?: {
      isGenerated?: boolean;
      isIncomplete?: boolean;
      needsManualReview?: boolean;
      isTooLarge?: boolean;
    };
  };

  // Error information (if applicable)
  error?: {
    code: string;
    message: string;
    type: 'github_api' | 'rate_limit' | 'timeout' | 'ai_service';
    timestamp: string;
  };

  // Processing metadata
  commited?: boolean;
  analyzedAt?: string;
}
```

---

**End of Document**

**Document Version**: 1.0
**Last Updated**: December 19, 2025
**Author**: Senior Staff Engineer
**Project**: Superteam Earn - GitHub Auto-Review System
**Total Pages**: 60+
**Word Count**: ~18,000 words
