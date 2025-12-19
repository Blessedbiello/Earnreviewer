# GitHub Auto-Review System - Implementation Proposal
**Superteam Earn Bounty Submission**

---

## 📋 Table of Contents

1. [Project Overview](#project-overview)
2. [Solution Architecture](#solution-architecture)
3. [Technical Architecture](#technical-architecture)
4. [Implementation Plan](#implementation-plan)
5. [Edge Case Handling](#edge-case-handling)
6. [Scoring Methodology](#scoring-methodology)
7. [Deliverables](#deliverables)

---

## 🎯 Project Overview

Building an AI-powered auto-review system for GitHub PRs and repositories, integrated with Superteam Earn's existing agent infrastructure.

**Scope**:
- **Timeline**: 2-3 weeks (140-160 hours)
- **Deliverables**: Production-ready system with 40+ edge cases handled
- **Integration**: Seamless fit with existing BullMQ, OpenRouter, and dashboard workflows
- **Technologies**: Node.js, TypeScript, Next.js, Prisma, Octokit.js, OpenRouter

---

## 🏗️ Solution Architecture

### System Components

**1. GitHub URL Detection & Validation**
- Regex-based pattern matching for PR and repository URLs
- Extract: `owner`, `repo`, `PR number` (if applicable)
- Validate accessibility via GitHub API
- Store parsed metadata in `submission.ai.githubMeta`

**2. Data Fetching Layer (GitHub GraphQL API)**

Pull Request Data:
- Metadata: title, body, state, isDraft, mergeable
- Metrics: additions, deletions, changedFiles, commits count
- Reviews: state (APPROVED/CHANGES_REQUESTED), comments
- Files: path, additions, deletions per file
- CI/CD: check runs, status, conclusion

Repository Data:
- Stats: stars, forks, watchers, open issues
- Languages: breakdown by bytes
- Activity: commit count, last commit date, contributors
- Documentation: README content, license
- Structure: test files presence, CI/CD config

**3. AI Analysis Engine**

Quantitative Scoring (GitHub metrics):
- PR size scoring (optimal: 100-500 lines)
- Commit count and distribution
- Test file ratio (test lines / total lines)
- CI/CD status (pass/fail/none)

Qualitative Scoring (OpenRouter AI):
- Code complexity analysis
- Alignment with bounty requirements
- Security and best practices review
- Documentation quality assessment

**4. Scoring & Label Assignment**

8-component weighted algorithm:
- Code Quality: 30%, PR Size: 15%, Commits: 10%
- Documentation: 15%, Testing: 15%
- Reviews: 5%, CI/CD: 5%, Alignment: 5%

Label thresholds:
- `Shortlisted`: ≥85, `High_Quality`: 70-84, `Mid_Quality`: 50-69
- `Low_Quality`: <50, `Inaccessible`: API errors
- `Needs_Review`: Edge cases/uncertainty

**5. Dashboard Integration**
- GitHub stats API endpoint
- AI review modal component (follows existing pattern)
- Metadata display in submission panel
- Commit reviewed workflow integration

## 🔧 Technical Architecture

### Event-Driven Workflow

```
┌─────────────────────────────────────────────────────────────────┐
│ 1. SUBMISSION (Earn Service)                                    │
│    - User submits GitHub URL                                    │
│    - /api/submission/create detects GitHub pattern             │
│    - Parse & validate URL → extract metadata                   │
│    - Store in Submission table (ai.githubMeta)                 │
│    - Queue: queueAgent({ type: 'autoReviewGitHubSubmission' }) │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│ 2. PROCESSING (Earn-Agent Service - Worker)                    │
│    - Consume job from agentLogicQueue (BullMQ + Redis)        │
│    - Fetch submission + listing context (Prisma)              │
│    - GitHub GraphQL API calls (with token rotation)           │
│    - Extract & normalize data                                  │
│    - Run AI analysis (OpenRouter)                             │
│    - Calculate scores (8-component algorithm)                 │
│    - Store results → submission.ai.evaluation                 │
└────────────────────────────┬────────────────────────────────────┘
                             ↓
┌─────────────────────────────────────────────────────────────────┐
│ 3. REVIEW (Sponsor Dashboard)                                   │
│    - GET /api/.../ai/unreviewed → fetch unreviewed GitHub      │
│    - Display in AiReviewGitHub modal                           │
│    - Show scores, notes, GitHub stats                          │
│    - POST /api/.../ai/commit-reviewed → apply labels          │
│    - Update submission.label, submission.notes                 │
└─────────────────────────────────────────────────────────────────┘
```

### Integration Architecture

**Earn Service** (Next.js - Public Repo):
- **Modified**: `submission/create.ts`, `queueAgent.ts`, `submissionFormSchema.ts`
- **New**: `github.ts` (API client), `githubUrlParser.ts`, `AiReviewGitHub.tsx`, `github-stats/route.ts`

**Earn-Agent Service** (Worker - Private Repo):
- **New**: GraphQL queries, data fetchers, analyzers, scorers, error handlers
- **Handler**: `autoReviewGitHubSubmission.ts` (main worker)

**External Services**:
- **GitHub API**: GraphQL (Octokit.js) - 5 PAT tokens for rotation (25k req/hour)
- **OpenRouter**: AI analysis via Vercel AI SDK (already integrated)
- **Redis**: BullMQ queue (existing `AGENT_REDIS_URL`)
- **MySQL**: Prisma ORM (flexible `submission.ai` JSON field)

### Key Technical Decisions

**GraphQL vs REST**: GraphQL chosen for:
- 75% fewer requests (1 query vs 4+ REST calls)
- 2x better rate limits (2,000 vs 900 points/min)
- Precise field selection (67% bandwidth reduction)

**Token Rotation Strategy**:
- 5 GitHub PATs → 25,000 requests/hour capacity
- Auto-switch when `X-RateLimit-Remaining < 100`
- Exponential backoff on secondary rate limits

**Error Handling**:
- 404/403 → `Inaccessible` label
- Timeouts → Retry 3x, then `Needs_Review`
- Rate limits → Token rotation, queue retry

---

## 📅 Implementation Plan (3 Weeks)

### Week 1: Core Infrastructure

**Days 1-2: GitHub API Integration**
- Octokit.js client with 5-token rotation
- URL parser (detect PR vs repo, extract metadata)
- Rate limit monitoring
- Files: `github.ts`, `githubUrlParser.ts`, `githubSubmission.ts` (types)

**Days 3-4: Agent Queue Extension**
- Add `autoReviewGitHubSubmission` to `AgentActionType`
- Modify `submission/create.ts` → detect GitHub URLs, queue job
- Update form validation schema
- Files: `queueAgent.ts`, `submission/create.ts`, `submissionFormSchema.ts`

**Days 5-7: Data Fetchers (earn-agent)**
- GraphQL queries for PR/repo data
- Pagination handling
- Error handling (404, 403, timeouts, rate limits)
- Integration tests
- Files: `queries/pr.ts`, `queries/repo.ts`, `fetchPRData.ts`, `fetchRepoData.ts`

---

### Week 2: AI Analysis Engine

**Days 8-10: Scoring Algorithms**
- PR scorer (8 components: quality 30%, size 15%, commits 10%, docs 15%, tests 15%, reviews 5%, CI 5%, alignment 5%)
- Repo scorer (7 components: quality 25%, docs 20%, tests 20%, activity 15%, community 10%, tech 5%, license 5%)
- Label assignment logic
- Files: `scorers/prScorer.ts`, `scorers/repoScorer.ts`, `scorers/labelAssigner.ts`

**Days 11-12: AI Integration**
- OpenRouter prompt engineering (code analysis, bounty alignment)
- Response parsing & validation
- Large PR sampling (>1000 files → sample 50 + tests)
- Files: `prompts/`, `analyzer.ts`, `samplers/prSampler.ts`

**Days 13-14: Edge Cases & Testing**
- 40+ edge case handlers (private repos, drafts, force push, duplicates, etc.)
- Test suite with 20+ real PRs/repos
- Accuracy validation
- Files: `edgeCases/`, comprehensive test suite

---

### Week 3: Dashboard & Production

**Days 15-17: Frontend**
- `AiReviewGitHub.tsx` modal (DISCLAIMER → INIT → PROCESSING → DONE)
- GitHub stats display in submission panel
- Score breakdown visualization
- Files: `AiReviewGitHub.tsx`, `SubmissionPanel.tsx`, `GitHubStats.tsx`

**Days 18-19: API Endpoints**
- `GET /github-stats` → fetch metadata from `submission.ai`
- Update `/ai/unreviewed` → include GitHub submissions
- Update `/ai/commit-reviewed` → handle GitHub label conversion
- Files: `github-stats/route.ts`, modifications to unreviewed/commit APIs

**Days 20-21: Production Hardening**
- Monitoring (Prometheus/Grafana or similar)
- Structured logging (JSON format)
- Performance optimization (caching, batching)
- E2E testing
- Deployment guide, troubleshooting docs

---

## 🛡️ Edge Case Handling (40+ Scenarios)

### Access & Permissions
| Scenario | Detection | Handling | Result |
|----------|-----------|----------|--------|
| Private repo | 404/403 error | Store error metadata | `Inaccessible` |
| Deleted PR | 404 after submission | Same as private | `Inaccessible` |
| Token expired | 401 Unauthorized | Rotate to next token | Continue |
| Rate limit | `X-RateLimit-Remaining < 100` | Switch token | Continue |
| DMCA takedown | 451 status | Store legal notice | `Inaccessible` |

### PR State
| Scenario | Detection | Handling | Result |
|----------|-----------|----------|--------|
| Draft PR | `pr.draft === true` | Flag `isIncomplete` | `Needs_Review` |
| Merged PR | `pr.merged === true` | Boost score +5 | Normal (positive signal) |
| Closed (not merged) | `state === 'closed'` | Lower score, log reason | `Mid_Quality` ↓ |
| Force push | `pr.head.sha` changed | Use submission snapshot | Normal |
| No description | `pr.body === null` | Reduce docs score | Lower docs component |

### Size & Content
| Scenario | Detection | Handling | Result |
|----------|-----------|----------|--------|
| Large PR | `changedFiles > 1000` | Sample 50 files + tests, flag `isTooLarge` | `Needs_Review` |
| Empty repo | `repo.size === 0` | Skip analysis | `Low_Quality` |
| Generated code | File pattern analysis | Filter, reduce score if >80% | Lower score |
| Tiny PR | `additions + deletions < 5` | Context-aware scoring | Variable |
| Deps only | Only package.json changed | Check bounty requirements | `Needs_Review` |

### Performance & Reliability
| Scenario | Detection | Handling | Result |
|----------|-----------|----------|--------|
| Token exhaustion | All tokens rate-limited | Queue retry in 1 hour | Temp `Needs_Review` |
| Timeout | Processing >5 min | Cancel, log, partial data | `Needs_Review` |
| AI service down | OpenRouter errors | Retry 3x, exponential backoff | `Needs_Review` |
| GitHub API down | Persistent 500/503 | Retry every 15min for 2h | `Needs_Review` |

### Workflow
| Scenario | Detection | Handling | Result |
|----------|-----------|----------|--------|
| Duplicate | Same URL exists for listing | Check on submission | Reject (error) |
| Link edited post-review | `updatedAt` after `analyzedAt` | Disallow edit | Validation error |
| Multiple PRs | Different PR numbers | Allow separate analysis | Independent scoring |

**Full catalog**: See `GITHUB_AUTO_REVIEW_PROPOSAL.md` for complete edge case documentation.

---

## 🎯 Scoring Methodology

### Pull Request Scoring (8 Components)

| Component | Weight | Scoring Criteria | Scale |
|-----------|--------|------------------|-------|
| **Code Quality** | 30% | Complexity (cyclomatic, nesting), Style (formatting, naming), Best practices (error handling, security) | Simple (<10 complexity): 100<br>Complex (>40): 40 |
| **PR Size** | 15% | Optimal: 100-500 lines | <10: 40, 10-99: 70, **100-500: 100**, 501-1000: 85, 1001-2000: 60, >2000: 30 |
| **Commit Quality** | 10% | Message quality (conventional commits), Count (optimal: 2-10), Distribution (even > single massive) | 0-100 |
| **Documentation** | 15% | PR description (60%), Code comments (40%) | 0-100 |
| **Testing** | 15% | Presence (50%), Coverage estimate (30%), Quality via AI (20%) | Test ratio: `test_lines / total_lines` |
| **Review Feedback** | 5% | Approvals (+20 each), Changes requested (-10 each) | Fork submissions: neutral (60) |
| **CI/CD Status** | 5% | All passing: 100, Some failing: 50, All failing: 0, No CI: 60 | From GitHub Checks API |
| **Bounty Alignment** | 5% | AI semantic similarity with listing.requirements | 0-100 |

**Formula**: `overall = Σ(component_score × weight)`

### Repository Scoring (7 Components)

| Component | Weight | Factors | Scoring |
|-----------|--------|---------|---------|
| **Code Quality** | 25% | Structure, complexity, consistency | AI analysis + static metrics |
| **Documentation** | 20% | README (60%), Code docs (30%), Extras (10%) | Has README: +50 base, quality: 0-100 |
| **Testing** | 20% | Test files, coverage ratio, CI/CD | Presence: +40, ratio >30%: +30, >50%: +30 |
| **Activity** | 15% | Last commit (50%), Commit count (30%), Issues/PRs (20%) | Recent (<30d): 100, <90d: 80, <180d: 60, <365d: 40, >365d: 20 |
| **Community** | 10% | Stars (40%), Forks (30%), Contributors (30%) | <10: 30, 10-50: 60, 50-100: 80, 100-500: 90, >500: 100 |
| **Tech Stack** | 5% | Language match (60%), Modern practices (40%) | Matches bounty: 100, Partial: 60, Different: 30 |
| **License** | 5% | OSI-approved (70%), Compatibility (30%) | MIT/Apache/GPL: 100, Custom: 70, None: 30 |

**Formula**: `overall = (Σ(component_score × weight)) × alignment_modifier`
- `alignment_modifier = 0.7 + (0.3 × bounty_alignment / 100)`

### Label Assignment Thresholds

```typescript
if (score >= 85)  return 'Shortlisted';
if (score >= 70)  return 'High_Quality';
if (score >= 50)  return 'Mid_Quality';
if (score >= 30)  return 'Low_Quality';
return 'Low_Quality';  // <30: recommend reject

// Override conditions
if (hasErrors)           return 'Inaccessible';
if (needsManualReview)   return 'Needs_Review';
if (isTooLarge)          return 'Needs_Review';
```

---

## 📊 Success Metrics & Validation

| Metric | Target | Threshold | Validation Method |
|--------|--------|-----------|-------------------|
| **Functional** |
| End-to-end workflow | 100% operational | N/A | 10+ manual test submissions |
| PR/Repo analysis | Any public resource | N/A | 20+ real PRs, 20+ repos |
| Edge case coverage | All 40+ handled | 95%+ | Comprehensive test suite |
| **Performance** |
| Typical PR (<500 files) | 15s avg | 25s max | Load testing |
| Large PR (500-1000 files) | 30s avg | 45s max | Load testing |
| Repository analysis | 25s avg | 40s max | Load testing |
| GitHub API usage | <50% rate limit | <70% | Monitor `X-RateLimit-*` headers |
| Error rate | <2% | <5% | Failed jobs / total jobs |
| **Accuracy** |
| AI vs Manual agreement | >80% | >70% | Test on 100 submissions |
| Label accuracy | >75% exact | >65% | Compare vs sponsor final decision |
| False positives (Shortlisted → Low) | <5% | <10% | Track sponsor overrides |
| False negatives (Low → Shortlisted) | <10% | <15% | Track sponsor overrides |

---

## 📦 Deliverables

### Code Implementation (~4,000 LOC total)
- **Production code**: ~2,500 lines (TypeScript)
- **Test code**: ~1,500 lines (>80% coverage)
- **Files modified**: 5 files in earn service
- **Files created**: 11 files (5 earn, 6 earn-agent)

### Documentation
- API endpoint reference (GraphQL queries, REST routes)
- Scoring algorithm specification
- Deployment guide (Railway setup, env vars)
- Troubleshooting runbook
- Edge case handling catalog

### Infrastructure
- Monitoring dashboards (metrics, alerts)
- Error tracking (Sentry integration)
- Structured logging (JSON format)
- Performance benchmarks

### Support
- 30 days post-launch bug fixes
- Prompt optimization based on feedback
- Performance tuning

### File Breakdown

**Earn Service** (Next.js):
- New: `github.ts`, `githubUrlParser.ts`, `githubSubmission.ts`, `github-stats/route.ts`, `AiReviewGitHub.tsx`
- Modified: `queueAgent.ts`, `submission/create.ts`, `submissionFormSchema.ts`, `ai/unreviewed/route.ts`, `ai/commit-reviewed/route.ts`

**Earn-Agent Service** (Worker):
- New: `queries/pr.ts`, `queries/repo.ts`, `fetchPRData.ts`, `fetchRepoData.ts`, `analyzer.ts`, `autoReviewGitHubSubmission.ts`

---

## 📚 Additional Resources

**Full Technical Proposal**: See `GITHUB_AUTO_REVIEW_PROPOSAL.md` for:
- Detailed technical architecture
- Complete edge case catalog
- In-depth scoring algorithms
- API examples and code samples
- Database schema specifications
- Monitoring and maintenance plans

**Questions?**
- GitHub: [Your GitHub Profile]
- Email: [Your Email]
- Discord: [Your Discord Handle]
- Portfolio: [Your Portfolio Link]

---

*This proposal demonstrates my commitment to thorough planning, technical excellence, and delivering production-grade solutions. I look forward to working with the Superteam team!*

---

**Document Version**: 1.0
**Last Updated**: December 19, 2025
**Project**: Superteam Earn - GitHub Auto-Review System
**Proposed Timeline**: 2-3 weeks

