---
name: organize-work-records
description: Transform raw work logs into formal IIT 6-part organized records with day-off entries. Supports single-day and week-batch modes.
---

# Organize Work Records Skill

Transform raw work logs from `raw_workrecords/` into formal IIT Industrial Placement Record Book entries following the 6-part format, placed in the correct directory structure. Handles day-off entries automatically.

---

## Invocation

```
/organize-work-records 2026-03-18              # Single day
/organize-work-records 2026-03-15 2026-03-21   # Week range (batch)
```

**Single-day mode:** Read one raw log, produce one organized record.
**Week-batch mode:** Process an entire date range — organize all raw logs found, create day-off entries for gaps, create the week directory if needed.

---

## Execution Flow

### Step 1: Parse Input and Build Calendar

1. Parse the date(s) from the invocation arguments.
2. For week-batch mode, enumerate every date in the range.
3. Classify each date:
   - **Friday / Saturday** → Day off (scheduled)
   - **Sunday through Thursday with a raw log** → Work day (organize from raw)
   - **Sunday through Thursday without a raw log** → Day off (ask user for reason)
4. The work week is **Sunday through Thursday**. Friday and Saturday are always days off unless a raw log exists (extra work).

### Step 2: Resolve Paths

**Raw log location:** `E:/IIT/Placement Year/work-records/raw_workrecords/`
- Files match: `work-record-YYYY-MM-DD.md` (e.g., `work-record-2026-03-18.md`)
- Also check for session variants: `work-record-YYYY-MM-DD-session2.md`

**Output location:** `E:/IIT/Placement Year/work-records/{Month}-{Year}/week-{NN}/DD-mon-YYYY.md`
- Month folder: Full month name, hyphen, four-digit year (e.g., `March-2026`)
- Week folder: `week-{NN}` where NN is the week number within the month
- File name: two-digit day, three-letter lowercase month, four-digit year (e.g., `18-mar-2026.md`)

**Week numbering logic:**
- Week 01 = days 1-7 of the month
- Week 02 = days 8-14
- Week 03 = days 15-21
- Week 04 = days 22-28
- Week 05 = days 29-31 (if applicable)

### Step 3: Ask About Missing Weekdays (Batch Mode Only)

For any **Sunday through Thursday** date with no raw log, prompt the user ONCE with options:
- Ramadan
- Public Holiday (ask which one)
- Annual Leave
- Sick Leave
- Other (ask for description)

If multiple consecutive gaps share the same reason (e.g., all Ramadan), the user can specify once for all.

### Step 4: Create Directories

Run `mkdir -p` for any output directories that don't exist.

### Step 5: Process Work Days

For each work day with a raw log:

1. **Read** the raw work log file in full.
2. **Transform** into the 6-part IIT format (see format spec below).
3. **Write** the organized record to the correct output path.

**For batch mode:** Use parallel agents (up to 3) to process multiple records simultaneously. Each agent receives:
- The raw log content (or path to read)
- The exact output path
- The date and day of week
- The full format specification from this skill
- A gold-standard example to reference

### Step 6: Create Day-Off Entries

For each day off, write a minimal entry:

**Friday/Saturday (scheduled):**
```
[Day of week], [Month] [Day][ordinal], [Year]

Title: Day Off

Scheduled day off.
```

**Weekday gap (with reason):**
```
[Day of week], [Month] [Day][ordinal], [Year]

Title: Day Off — [Reason]

Scheduled day off. [Reason] observance.
```

### Step 7: Verify

- List all files in the output directory to confirm expected count.
- Spot-check one work record for format compliance.

---

## The 6-Part IIT Format (Complete Specification)

Every organized work record MUST follow this exact structure:

### Part 1: Date
```
[Day of week], [Month] [Day][ordinal], [Year]
```
Example: `Wednesday, March 18th, 2026`

### Part 2: Title
```
Title: [Primary Focus]: [Specific Achievement(s)]
```
- Concise, professional, under 120 characters
- Use technical terminology
- Connect 2-3 major themes with commas or ampersands

### Part 3: Objective
```
Objective:
To [action verb] [what] by [how], [context], and [validation].
```
- 2-4 sentences starting with "To [verb]..."
- State the technical problem being solved
- Mention architectural or business context
- Include expected outcomes

### Part 4: Work Carried Out

**THIS IS THE MOST CRITICAL SECTION.**

```
Work Carried Out:

**[Bold Topic Header 1]:** [Detailed narrative paragraph — 3-8 sentences explaining
context, approach, implementation details, and results...]

**[Bold Topic Header 2]:** [Another narrative paragraph...]
```

#### Mandatory Rules:
- Write in **bold-headed paragraphs** with clear topic headers
- Each paragraph: 3-8 sentences with context → approach → implementation → result
- **NO bullet points. NO numbered lists. NEVER.**
- Use professional, enterprise-grade technical language
- Include precise details FROM THE RAW LOG: function names, parameter counts, file names, line counts
- Use causal language: "as a result", "this ensures", "to prevent", "which enables"
- Include quantitative measurements: "5 parameters to 4", "234 records", "48/48 tests passing"
- Explain WHY decisions were made, not just WHAT was done
- Use past tense throughout
- Aim for 5-10 topic paragraphs per entry

#### Topic Header Patterns:

**For Architecture/Refactoring:** Core Architectural Refactor, Breaking API Change Resolution, Data Model Restructuring, Service Decomposition

**For Feature Development:** Feature Design and Implementation, Core Business Logic, API Endpoint Development, Frontend Component Architecture

**For Bug Fixes:** Root Cause Analysis and Resolution, State Management Fix, Data Integrity Correction, Error Handling Enhancement

**For DevOps/Infrastructure:** Infrastructure Configuration, Deployment Pipeline Enhancement, Container Orchestration, Network Security Hardening

**For Testing:** End-to-End Testing with Browser Automation, Integration Test Coverage, Manual Testing and Bug Discovery

**For Documentation:** Technical Documentation Suite, Architecture Decision Records

### Part 5: Outcome
```
Outcome:
[1-2 sentence summary with specific metrics — commits, files, tests, features.]
```

### Part 6: Activities Covered
```
Activities Covered:
[code].[subcode] [Description of how this applies to the day's work]
```
- Select 6-12 codes that accurately reflect the documented work
- Include the description showing relevance

---

## Privacy & Confidentiality Rules

**CRITICAL:** All output must protect proprietary information.

**Replace:**
- Company/product names → "the protocol", "the platform", "the token"
- Internal service names (`svc-admin`, `svc-kyc`) → "the admin service", "the KYC service"
- Specific domain names / IPs → "the development server", "the cluster"
- Internal URLs → "the admin portal", "the wallet application"
- Credential values → never include
- Team member names → "the team", "the developer"

**Keep:**
- Technology names (Hyperledger Fabric, Next.js, Prisma, PostgreSQL, Kubernetes, Go, TypeScript)
- Architecture patterns (Clean Architecture, CQRS, BFF, outbox pattern)
- Generic technical details (function signatures, file counts, line counts)
- Problem-solving methodology and approach descriptions

---

## Truthfulness Principle

**Every work record must be strictly derived from the raw work log.**

- No fabrication, no invented details, no embellished technical content.
- The raw log is the single source of truth.
- If a raw log is light on detail, the organized record reflects that scope honestly.
- Never invent functions, metrics, or technical details not present in the source material.
- Activity codes must accurately reflect documented work, not aspirational coverage.

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

## Gold-Standard Writing Reference

When transforming raw logs, match this calibre of writing:

```
**Core Architectural Refactor (User vs. Organization):** A critical architectural
flaw was identified and corrected. The chaincode was refactored to enforce clear
separation between individual human accounts (User) and organizational entities
(Organization). The User struct's AccountType field was completely removed,
establishing the principle that all User accounts are, by definition, individuals
verified through KYC and biometric authentication. This eliminated the architectural
contradiction where business entities could be represented as "users."
```

Key qualities to match:
- Opens with context/problem
- Explains the technical approach
- Includes specific implementation details (struct names, field names)
- Closes with the result/impact
- Professional but readable enterprise tone
- Causal language connecting ideas

---

## Agent Prompt Template (For Batch Processing)

When dispatching parallel agents, use this prompt structure:

```
You are a technical scribe for an IIT Industrial Placement Record Book.

CRITICAL: Only document work ACTUALLY described in the raw log. Never fabricate.

Step 1: Read the raw work log at [PATH]
Step 2: Read a style reference at E:/IIT/Placement Year/work-records/February-2026/week-02/11-feb-2026.md
Step 3: Transform into the 6-part IIT format
Step 4: Write to [OUTPUT_PATH]

Date: [Day of week], [Month] [Day][ordinal], [Year]

Rules:
- Bold-headed narrative paragraphs — NO bullet points, NO numbered lists
- Include precise technical details FROM the raw log
- Professional enterprise-grade technical language
- Maintain CONFIDENTIALITY: generalize company-specific details
- Map to 6-10 IIT activity codes
- Past tense throughout
```

---

## Quality Checklist

Before finalizing any record:
- [ ] Date format correct (Day of week, Month Day, Year)
- [ ] Title is concise and technical
- [ ] Objective starts with "To [verb]..." and is 2-4 sentences
- [ ] Work Carried Out uses bold-headed paragraphs (NOT bullets)
- [ ] Each paragraph has context, approach, implementation, result
- [ ] Quantitative details included (file counts, line counts, metrics)
- [ ] Technical terminology is precise
- [ ] No proprietary information exposed
- [ ] Outcome summarizes with metrics in 1-2 sentences
- [ ] 6-12 activity codes mapped accurately
- [ ] Professional, enterprise-grade tone throughout
- [ ] Past tense used consistently
- [ ] No bullet points in narrative sections
- [ ] Content is truthful — derived only from the raw log
