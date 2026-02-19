---
name: placement-log
description: Transform git commits and work records into IIT Industrial Placement Record Book entries
---

# Placement Log Skill

Transform daily git commits, work records, and session notes into formal IIT Industrial Placement Record Book entries suitable for both the markdown record book and the Google Sheets condensed format.

---

## Overview

This skill bridges two documentation worlds:
- **Internal work records** (`docs/workrecords/work-record-YYYY-MM-DD.md`) - detailed engineering stories
- **IIT Placement Record Book** - formal, structured academic entries for evaluation

The skill reads git commits and/or internal work records for a given date, then generates placement log entries in the required format.

---

## Privacy & Confidentiality Rules

**CRITICAL:** All output must be generalized to protect proprietary information.

**Replace:**
- "GX Protocol" / "GXCoin" / "GXT" → "the protocol" / "the platform" / "the token"
- Internal service names (`svc-admin`, `svc-kyc`) → "the admin service", "the KYC service"
- Specific domain names / IPs → "the development server", "the cluster"
- Internal URLs → "the admin portal", "the wallet application"
- Credential values → never include
- Team member names → "the team", "the developer"

**Keep:**
- Technology names (Hyperledger Fabric, Next.js, NestJS, Prisma, PostgreSQL, Kubernetes)
- Architecture patterns (Clean Architecture, CQRS, BFF, outbox pattern)
- Generic technical details (function signatures, file counts, line counts)
- Problem-solving methodology and approach descriptions

---

## Output Format 1: Full Markdown Entry (Record Book File)

### File Naming Convention
```
DD-mon-YYYY.md
```
Example: `03-feb-2026.md`

### Directory Structure
```
work-records/
├── {Month}-{Year}/
│   ├── week-{NN}/
│   │   └── DD-mon-YYYY.md
```

### The 6-Part Structure

```markdown
[Day of week], [Month] [Day], [Year]

Title: [Primary Focus]: [Specific Achievement]

Objective:
To [action verb] [technical component/system] by [specific approach/method],
[additional context about why this work matters], and [documentation/validation
activities].

Work Carried Out:

**[Topic Header 1]:** [Detailed narrative paragraph explaining context,
approach, implementation details, and results. Include specific technical
details like function names, parameter counts, file paths. Explain WHY
decisions were made, not just WHAT was done. Each paragraph should be
3-8 sentences with quantitative measurements where possible.]

**[Topic Header 2]:** [Another detailed narrative paragraph for the next
major work item. Continue the narrative flow, connecting work items logically.
Include error handling, validation approaches, and architectural impacts.]

[Continue with 4-8 major topic paragraphs]

Outcome:
[1-2 sentence summary of key achievements and current project status.
Include specific metrics where possible.]

Activities Covered:
[Code].[Subcode] [Brief description of how this applies]
[Code].[Subcode] [Brief description]
[6-10 relevant activity codes from the IIT list]
```

### Writing Rules for "Work Carried Out"

**DO:**
- Write in **bold-headed paragraphs** (NOT bullet points)
- Use professional, enterprise-grade technical language
- Create narrative flow connecting different work items
- Explain WHAT, WHY, and HOW for each work item
- Include specific details: function names, parameter counts, constants, file names
- Use quantitative measurements (e.g., "reduced from 5 parameters to 4", "234 records")
- Explain architectural decisions and their impacts
- Use past tense throughout
- Use causal language: "as a result", "this ensures", "to prevent", "which enables"

**DON'T:**
- Use bullet points or numbered lists in Work Carried Out section
- Simply list commit messages
- Include company-specific identifiers or proprietary names
- Use casual language or first person ("I did...")
- Skip explaining the reasoning behind decisions
- Leave out quantitative details

### Topic Header Patterns by Work Type

**Architecture/Refactoring:**
- Core Architectural Refactor
- Breaking API Change Resolution
- Data Model Restructuring
- Service Decomposition

**Feature Development:**
- Feature Design and Implementation
- Core Business Logic Implementation
- API Endpoint Development
- Frontend Component Development

**Bug Fixes:**
- Root Cause Analysis and Resolution
- State Management Fix
- Data Integrity Correction
- Error Handling Enhancement

**DevOps/Infrastructure:**
- Infrastructure Configuration
- Deployment Pipeline Enhancement
- Container Orchestration
- Network Security Hardening

**Testing:**
- End-to-End Testing with Browser Automation
- Integration Test Coverage
- Manual Testing and Bug Discovery

**Documentation:**
- Technical Documentation Enhancement
- Architecture Decision Records
- Handover Documentation

---

## Output Format 2: Google Sheet Condensed Entry

For pasting into the placement record book Google Sheet (single cell, column B).

### Structure

```
[Full Date] — [Title]: [Subtitle]

Objective: [Single comprehensive sentence describing all goals for the day, starting with an action verb.]

[Section Title 1]
[Dense paragraph with specific technical details. Past tense. No bullets. Include numbers, technology names, file counts, specific outcomes. Each section is 3-10 sentences of flowing prose.]

[Section Title 2]
[Another dense paragraph for the next major work item. Maintain formal academic tone throughout.]

[Continue with 4-8 sections]

Outcome: [Dense summary paragraph with specific metrics - files created, commits made, tests passed, features completed, bugs fixed. Quantify everything possible.]
```

### Style Rules for Sheet Format

- **No markdown formatting** (no `**bold**`, no `#headers`, no bullets)
- Section titles are plain text on their own line (they appear bold in the sheet due to cell formatting)
- Dense, flowing prose paragraphs
- Past tense throughout ("Identified...", "Implemented...", "Created...")
- Formal academic/professional tone
- Specific numbers everywhere (file counts, pod counts, line counts, percentages)
- Technical terminology used freely
- Each section flows as a continuous paragraph
- The Outcome paragraph quantifies everything

### Example Condensed Entry

```
Monday, February 3, 2026 — Multi-Service Bug Resolution: Supply Display Precision, Landing Page Styling & Blockchain Registration Pipeline

Objective: Resolve critical display precision errors in the admin dashboard supply visualization, fix Tailwind CSS v4 breaking changes on the wallet landing page, and repair the blockchain user registration pipeline ensuring approved users appear correctly in the administrative queue.

Supply Display Precision Architecture Correction
Investigation revealed a 1000x magnitude error in the admin dashboard's token supply visualization, displaying values in quintillions instead of trillions. Root cause traced to the backend returning raw 9-decimal precision values while the frontend applied additional scaling. Implemented precision-aware formatting that correctly interprets the backend's nano-token units. Additionally resolved runtime TypeError caused by null pool access when system pools had not yet been initialized, adding defensive null checks throughout the supply statistics components. Updated the phase distribution visualization to support 8 phases including the three Phase 1 sub-phases (1A Early Adopters, 1B Growth, 1C Maturity) reflecting the tiered early-adopter reward structure defined in the tokenomics specification.

[More sections...]

Outcome: Three critical production issues resolved across four repositories with 47 commits. Supply display now correctly shows trillion-scale values with proper precision handling. Landing page restored to full visual fidelity under Tailwind CSS v4. Blockchain registration pipeline repaired by adding onchain status tracking during KYC approval, with data migration applied to existing approved users. All fixes verified through manual testing and browser automation.
```

---

## Workflow: Generating a Placement Log Entry

### Step 1: Gather Source Data

Read git commits for the target date:
```bash
# From git-logs directory
cat git-logs/git-logs-YYYY-MM-DD.txt
```

And/or read internal work records:
```bash
# From work records
cat docs/workrecords/work-record-YYYY-MM-DD.md
```

### Step 2: Identify Major Themes

Group commits and work into 4-8 major themes. Common groupings:
- By service/repository (frontend, backend, chaincode, infra)
- By feature area (authentication, KYC, supply management)
- By activity type (bug fixes, new features, testing, documentation)
- By problem-solution narrative (investigation → fix → verification)

### Step 3: Generate Title

Format: `[Primary Focus]: [Specific Achievement(s)]`

Combine the major themes into a concise title. Use ampersands (&) to connect 2-3 themes.

### Step 4: Write Objective

Start with "To [action verb]..." and include:
- The primary technical goal
- The approach or methodology
- The expected outcome or impact

### Step 5: Write Work Carried Out / Sections

For each theme, write a bold-headed paragraph (markdown) or section (sheet):
1. **Context**: What was the situation/problem?
2. **Approach**: What technical solution was chosen?
3. **Implementation**: What specific changes were made?
4. **Result**: What was the outcome?

### Step 6: Write Outcome

Summarize with specific metrics:
- Number of commits
- Files created/modified
- Bugs fixed / features added
- Tests passed
- Services affected

### Step 7: Map Activity Codes (markdown only)

Select 6-10 IIT activity codes that best represent the work done.

---

## Official IIT Activities List (26 Categories)

### 1. Involvement in Monitoring Production Activities
- 1.1 Conduct preliminary investigations
- 1.2 Carry out feasibility study

### 2. Systems Analysis
- 2.1 Analyze current system (investigate, model, document)
- 2.2 Identify requirements and deficiencies of the existing system
- 2.3 Specify requirements of the proposed system

### 3. Systems Design
- 3.1 Design data (entity descriptions/models, DFD, RDA, data dictionary)
- 3.2 Design process outlines
- 3.3 User Interfaces/User experience/HCI/Screen formats and layouts
- 3.4 Prepare database/file specifications
- 3.5 Prepare program specifications (pseudocode, flow charts, NS diagrams)
- 3.6 Prepare implementation plan including test plan & test data
- 3.7 Prepare user/operator manuals
- 3.8 Use complementary design techniques (prototyping, top-down design, modularization, review & walkthrough, etc.)
- 3.9 UI Planning
- 3.10 Design/Develop Interactive UI Elements
- 3.11 Model/Texture/Animate 2D/3D environments
- 3.12 User Experience and HCI

### 4. Programming
- 4.1 Program design
- 4.2 Program code
- 4.3 Test programs
- 4.4 Customization of package & software

### 5. Testing
- 5.1 Testing module
- 5.2 Integration
- 5.3 System testing
- 5.4 User acceptance

### 6. Implementation
- 6.1 Educate and train users
- 6.2 Site preparation
- 6.3 Create database/convert files
- 6.4 Installation of software (Deployment)
- 6.5 Knowledge transfer

### 7. Quality Assurance
- 7.1 Analysis stage
- 7.2 Design stage
- 7.3 Implementation stage
- 7.4 Maintenance stage

### 8. Operations
- 8.1 Managing operations of systems
- 8.2 Disaster recovery
- 8.3 Backups
- 8.4 Security
- 8.5 Knowledge management

### 9. Maintenance
- 9.1 Document and/or update documentation
- 9.2 Conduct maintenance review and enhancement

### 10. Networking
- 10.1 Design
- 10.2 Implementation
- 10.3 Testing
- 10.4 Maintenance

### 11. Network Management
- 11.1 Monitor user functions
- 11.2 Monitor resource usage
- 11.3 Identify possible improvements (hardware, software, network)

### 12. Project Management
- 12.1 Project planning
- 12.2 Document project performance
- 12.3 Review project with reference to organizational goals

### 13. Multimedia
- 13.1 Multimedia systems design and development
- 13.2 Integration of hypertext, graphics, sound, animation, and video
- 13.3 Multimedia application

### 14. Marketing
- 14.1 Organizing product/brand expansion or launching campaigns
- 14.2 Preparing advertising/marketing schedules
- 14.3 Coordinating with advertising agencies

### 15. Accounting
- 15.1 Preparing income and expenditure statements
- 15.2 Preparing budgets and forecasting costs & revenues relating to projects
- 15.3 Preparing bank reconciliation statements

### 16. General Administration / Management
- 16.1 Coordinating & liaising with various internal & external parties
- 16.2 Preparing reports
- 16.3 Conducting presentations & informing (in writing/orally) of activities conducted

### 17. HRM/Personnel Management
- 17.1 Organizing activities for staff development
- 17.2 Preparing reports/presentations for staff development
- 17.3 Recording/updating employee details
- 17.4 Involvement in activities for recruitment of new staff and appraisal

### 18. Operations / Logistics Management
- 18.1 Managing activities relating to inbound and outbound logistics
- 18.2 Warehousing & stock control activities
- 18.3 Involvement in monitoring production activities

### 19. Cybersecurity
- 19.1 Conduct security assessments
- 19.2 Implement security protocols
- 19.3 Monitor security systems
- 19.4 Respond to security incidents
- 19.5 Develop security policies and procedures

### 20. Data Analysis and Management
- 20.1 Collect and clean data
- 20.2 Perform data analysis using statistical tools
- 20.3 Create data visualizations
- 20.4 Manage databases
- 20.5 Develop data models

### 21. Artificial Intelligence and Machine Learning
- 21.1 Develop machine learning models
- 21.2 Implement AI algorithms
- 21.3 Perform model training and validation
- 21.4 Evaluate model performance
- 21.5 Deploy AI solutions
- 21.6 Monitor AI systems

### 22. Cloud Computing
- 22.1 Design cloud architecture
- 22.2 Implement cloud solutions
- 22.3 Manage cloud services
- 22.4 Optimize cloud resources
- 22.5 Migrate applications to the cloud

### 23. Software Development and DevOps
- 23.1 Develop software applications
- 23.2 Implement DevOps practices
- 23.3 Automate deployment processes
- 23.4 Monitor application performance
- 23.5 Conduct code reviews
- 23.6 Maintain version control

### 24. E-commerce
- 24.1 Manage online store operations
- 24.2 Monitor sales and performance
- 24.3 Optimize user experience

### 25. Blockchain and Cryptocurrency
- 25.1 Develop blockchain applications
- 25.2 Implement smart contracts
- 25.3 Monitor blockchain networks
- 25.4 Perform cryptocurrency analysis
- 25.5 Ensure blockchain security

### 26. Robotics and Automation
- 26.1 Design robotic systems
- 26.2 Develop automation scripts
- 26.3 Implement robotic process automation (RPA)
- 26.4 Monitor robotic performance

---

## Invocation Examples

### Generate from git commits
```
/placement-log 2026-02-03
```
Reads git commits from `git-logs/git-logs-2026-02-03.txt` and generates both formats.

### Generate from work records
```
/placement-log 2026-02-02 --from-workrecord
```
Reads `docs/workrecords/work-record-2026-02-02.md` and condenses into placement format.

### Generate sheet-only format
```
/placement-log 2026-02-03 --sheet-only
```
Outputs only the Google Sheet condensed format (single text block ready to paste).

### Generate markdown-only format
```
/placement-log 2026-02-03 --markdown-only
```
Outputs only the full 6-part markdown format with activity codes.

---

## Quality Checklist

Before finalizing any placement log entry:

- [ ] Date format correct (Day of week, Month Day, Year)
- [ ] Title is concise and technical (under 100 chars)
- [ ] Objective starts with "To [verb]..." and is 2-4 sentences
- [ ] Work Carried Out uses bold-headed paragraphs (NOT bullets)
- [ ] Each paragraph has context → approach → implementation → result
- [ ] Quantitative details included (file counts, line counts, metrics)
- [ ] Technical terminology is precise (exact technology names)
- [ ] No proprietary information exposed (company names, internal URLs, credentials)
- [ ] Outcome summarizes achievements in 1-2 sentences with metrics
- [ ] 6-10 activity codes mapped (markdown format)
- [ ] Professional, enterprise-grade tone throughout
- [ ] Past tense used consistently
- [ ] No bullet points in narrative sections
