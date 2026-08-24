Critical:
 1. Rescearch and find out the files that will to be modified and the files that will be needed to carry out the requirements
 2. Ask clarafying questions 
 3. Make a well laid plan for the above requirements
 4. Create a to-do list of the steps to be taken to implement the requirements
 5. Track your progress by checking off the items in the to-do list as you complete them
 6. After implementing the requirements, provide a brief summary of the implementation plan

---
name: terminal-commands
description: "Guidelines for using terminal commands in the development environment."
metadata:
  author: developer
  version: "0.1.0"
---

# Terminal Commands

## Note on Terminal Commands

When using the terminal, use "cmd" as a prefix for commands. This ensures proper execution in the Windows PowerShell environment.

### Examples

```powershell
# Good - using cmd prefix
cmd /c "npm run build 2>&1 | head -100"

# Good - using cmd prefix
cmd /c "npm install sonner 2>&1"

# Good - using cmd prefix
cmd /c "npm run build"
```

### Why Use "cmd" Prefix?

- Ensures commands run in a consistent shell environment
- Allows for proper output redirection and piping
- Handles command chaining and complex operations reliably
- Works well with PowerShell's output handling

### General Guidelines

1. Always use `cmd /c` prefix for npm and other shell commands
2. Redirect stderr to stdout with `2>&1` for complete output capture
3. Use pipes (`|`) to filter or process command output
4. Test commands before running them in production environments

# NJPST Platform — AI Coding Agent Instructions

## Project Overview

This is the **Nigerian Journal of Polymer Science and Technology (NJPST)** — a Next.js 14+ / React open-access academic journal platform for the Polymer Institute of Nigeria (PIN). The goal is to digitize 30 years of polymer research, achieve global indexing (Scopus, Google Scholar, DOAJ), and become financially self-sustaining via Article Processing Charges (APCs).

**Hosting target:** `journal.polymerinstitute.org.ng`

## Tech Stack

- **Framework:** Next.js 14+ (App Router) with React Server Components
- **Database:** PostgreSQL via Prisma ORM
- **Auth:** NextAuth.js (JWT-based session management)
- **Storage:** AWS S3 (private bucket for manuscripts, presigned URLs for reviewers)
- **Payments:** Paystack / Flutterwave (inline widgets — never handle raw card data)
- **Styling:** Tailwind CSS (assumed from project conventions)

## Directory Structure

```
src/
├── app/
│   ├── layout.tsx              # Root layout: HTML shell, global fonts, base headers
│   ├── page.tsx                # Public homepage (Server Component)
│   ├── archive/page.tsx        # 30-year repository with faceted keyword search
│   ├── article/[id]/page.tsx   # Article landing page — MUST inject Highwire Press + Dublin Core meta tags
│   ├── checkout/[articleId]/page.tsx  # APC payment gateway (Client Component)
│   ├── dashboard/
│   │   ├── layout.tsx          # Protected layout: role-based sidebar navigation
│   │   ├── author/submit/page.tsx    # Multi-step manuscript upload (Client Component)
│   │   ├── reviewer/[assignmentId]/page.tsx  # Double-blind review interface
│   │   └── editor/page.tsx     # Editorial dashboard: dispatch reviews, mint DOIs
│   └── api/                    # Route handlers for backend logic
└── components/
    ├── Navigation.tsx          # Public top-bar menu
    ├── Sidebar.tsx             # Dashboard side nav (permission-aware)
    └── AdWidget.tsx            # Rotational corporate sponsor banners
```

## Core Data Model (5 Entities)

| Entity | Table | Key Fields |
|--------|-------|------------|
| **User** | `User` | `id` (UUID), `email` (unique), `role` (READER/AUTHOR/REVIEWER/EDITOR), `affiliation` |
| **Article** | `Article` | `id` (UUID), `title`, `abstract`, `keywords` (String[]), `pdfUrl`, `status` (SUBMITTED/UNDER_REVIEW/REJECTED/PUBLISHED), `doi` (nullable), `authorId` → User, `issueId` → Issue |
| **ReviewAssignment** | `ReviewAssignment` | `articleId`, `reviewerId`, `status` (PENDING/ACCEPTED/COMPLETED/DECLINED), `editorFeedback`, `authorFeedback`, `recommendation` |
| **Issue** | `Issue` | `volume` (int), `issueNumber` (int), `status` (DRAFT/PUBLISHED), `publishedAt` |
| **ApcToken** | `ApcToken` | `tokenCode` (unique, hashed), `isRedeemed` (boolean) |

## Critical Business Rules

### Article State Machine
`None → SUBMITTED → UNDER_REVIEW → REJECTED | PUBLISHED`
- Transition to `PUBLISHED` requires: (1) successful APC payment via Paystack/Flutterwave webhook, AND (2) minimum 2 reviewer critiques submitted.
- Once `PUBLISHED`, article is locked into exactly one `Issue` and a DOI is minted via Crossref API.

### APC Pricing Tiers (Financial Engine)
- **PIN Members** (valid token): ₦35,000 NGN
- **Non-member Nigerian authors:** ₦55,000 NGN
- **International authors:** $150 USD
- Tokens are **single-use** — flip `isRedeemed = true` immediately upon successful checkout.

### Double-Blind Review Enforcement
- Reviewers must NEVER see author names, affiliations, or identifying metadata.
- System must programmatically scrub all author-identifying fields before rendering reviewer views.
- Reviewers submit two feedback fields: `editorFeedback` (private to editors) and `authorFeedback` (anonymized to authors).

### RBAC Matrix (Enforced in `middleware.ts`)
- **READERS:** Read-only access to PUBLISHED articles and issues.
- **AUTHORS:** Read/write own submissions; blocked from editing once `UNDER_REVIEW`.
- **REVIEWERS:** Read anonymized assigned manuscripts; write feedback and recommendation scores.
- **EDITORS:** Full access — can modify statuses, assign reviews, generate APC tokens, mint DOIs, publish issues.

## API Endpoints

| Route | Method | Purpose |
|-------|--------|---------|
| `/api/articles/submit` | POST | Create article record; triggers iThenticate plagiarism webhook |
| `/api/apc/calculate` | POST | Calculate APC tier; validate token redemption |
| `/api/webhooks/payment` | POST | Paystack/Flutterwave listener — flips article to PUBLISHED |
| `/api/doi/mint` | POST | Editor-only: register DOI via Crossref REST API |
| `/api/oai-pmh` | GET | OAI-PMH XML feed for AJOL metadata harvesting |

## SEO & Indexing Requirements (Critical)

Every published article landing page (`/article/[id]`) **must** inject these meta tags into `<head>`:
- **Highwire Press tags:** `citation_title`, `citation_author`, `citation_publication_date`, `citation_pdf_url`
- **Dublin Core tags:** `DC.Title`, `DC.Creator`, `DC.Date`, `DC.Identifier` (DOI)
- These are non-negotiable for Google Scholar, Scopus, and DOAJ indexing compliance.

## Security Rules

- **All dashboard routes** (`/dashboard/*`) are protected by NextAuth middleware — unauthenticated users are redirected before any HTML renders.
- **Manuscript PDFs** are stored in a private S3 bucket; reviewers access them via **presigned URLs** (15-min expiry).
- **PCI-DSS compliance:** Never store or transmit raw card data — use Paystack/Flutterwave inline frames exclusively.
- **Database transactions** during checkout must use rollback to prevent corrupted states on gateway handshake failure.

## Key Reference Files

- `.github/instructions/PRD.md` — Full product requirements, data model, state machines, API specs
- `.github/instructions/docs.md` — Problem analysis, functional/non-functional requirements, scope boundaries
- `notes.md` — Miscellaneous notes (e.g., editor board with pictures)

## Conventions

- Use **Server Components** by default; opt into `'use client'` only when interactivity (forms, state) is required.
- All database mutations go through **Prisma** — never write raw SQL.
- API route handlers live under `src/app/api/` following Next.js App Router conventions.
- Role checks should happen in `middleware.ts` for route-level protection AND in API handlers for defense-in-depth.
