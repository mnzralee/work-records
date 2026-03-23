---
name: condense-week-sheet
description: Generate a condensed Google Sheets entry file from a week of organized work records, with Activity No codes and Problems/Solutions sections.
---

# Condense Week Sheet Skill

Read all organized work records in a week directory and produce a single `condensed-google-sheet-entry.md` file formatted for pasting into the IIT Placement Record Book Google Sheet.

---

## Invocation

```
/condense-week-sheet February-2026/week-04           # By folder path
/condense-week-sheet 2026-03-08 2026-03-14           # By date range
```

**By folder:** Directly specify the week directory relative to the work-records root.
**By date range:** Resolves to the correct `{Month}-{Year}/week-{NN}/` directory automatically.

---

## Execution Flow

### Step 1: Resolve Week Directory

- If given a folder path, use it directly under `E:/IIT/Placement Year/work-records/`.
- If given a date range, resolve to `{Month}-{Year}/week-{NN}/` using:
  - Week 01 = days 1-7, Week 02 = days 8-14, Week 03 = days 15-21, Week 04 = days 22-28, Week 05 = days 29-31

### Step 2: Read All Records

Read every `.md` file in the week directory (excluding any existing `condensed-google-sheet-entry.md`).
Sort files chronologically by date.

### Step 3: Generate Condensed Entries

For each daily record, produce a condensed entry following the Google Sheets format spec below.

### Step 4: Generate PROBLEMS ENCOUNTERED / SOLUTIONS FOUND

After all daily entries, read back through ALL work day content for the week and write:
- One concise paragraph (3-6 sentences) summarizing the key technical problems/challenges encountered
- One concise paragraph (3-6 sentences) summarizing the solutions applied

These must be **truthful** — only reference issues actually described in the week's records.

### Step 5: Write Output

Write the complete file to: `{week-directory}/condensed-google-sheet-entry.md`

### Step 6: Verify

Confirm the file was written and contains the correct number of daily entries.

---

## Output Format Specification

### File Structure

```
WEEK {NN} — {MONTH} {START}-{END}, {YEAR} — GOOGLE SHEETS CONDENSED ENTRIES
===================================================================

---

{Day 1 entry}

---

{Day 2 entry}

---

[... continue for all 7 days ...]

---

PROBLEMS ENCOUNTERED
{One paragraph summarizing the week's technical challenges}

SOLUTIONS FOUND
{One paragraph summarizing how those challenges were resolved}
```

### Work Day Entry Format

```
[Full Date] — [Title]

Objective: [Single comprehensive sentence starting with an action verb, covering all goals for the day.]

[Section Title 1]
[Dense paragraph with specific technical details. Past tense. No bullets. Include numbers, technology names, file counts, specific outcomes. 3-10 sentences of flowing prose.]

[Section Title 2]
[Another dense paragraph for the next major work item. Formal academic tone throughout.]

[Continue with 4-8 sections]

Outcome: [Dense summary paragraph with specific metrics — files created, commits made, tests passed, features completed, bugs fixed. Quantify everything possible.]

Activity No:
[code].[subcode] [Activity name]
[code].[subcode] [Activity name]
[6-15 relevant codes]
```

### Day-Off Entry Format

**Scheduled day off (Friday/Saturday):**
```
[Full Date] — Day Off

No work carried out. Scheduled day off.
```

**Ramadan / Holiday / Leave:**
```
[Full Date] — Day Off (Ramadan)

No work carried out. Ramadan observance.
```

---

## Critical Formatting Rules

### ABSOLUTELY NO MARKDOWN

This is the most important rule. The output is pasted into Google Sheets cells.

- **NO** `**bold**` markers — section titles are plain text on their own line
- **NO** `# headers` — the file header uses `===` underline only
- **NO** bullet points or numbered lists anywhere
- **NO** backtick code blocks in the daily entries
- **NO** markdown links or formatting

### Writing Style

- **Dense, flowing prose paragraphs** — every section is a continuous paragraph
- **Past tense throughout** — "Identified...", "Implemented...", "Resolved..."
- **Formal academic/professional tone** — suitable for university evaluation
- **Specific numbers everywhere** — file counts, pod counts, line counts, percentages, commit counts, test counts
- **Technical terminology used freely** — exact technology names, architecture patterns, protocol names
- Each section flows as a continuous paragraph, not fragmented sentences

### Activity No Section

- Listed at the end of each WORK DAY entry (not day-off entries)
- Format: `[code].[subcode] [Full activity name from IIT list]`
- Extract codes from the source record's "Activities Covered" section
- Simplify descriptions to just the official activity name (no parenthetical details)
- Include 6-12 codes per work day

### PROBLEMS ENCOUNTERED / SOLUTIONS FOUND

- Appears ONCE at the very end of the file, after all daily entries
- Separated from the last entry by `---`
- **PROBLEMS ENCOUNTERED**: One paragraph (3-6 sentences) summarizing the week's most significant technical challenges, blockers, bugs, architectural issues
- **SOLUTIONS FOUND**: One paragraph (3-6 sentences) summarizing specific technical approaches, tools, patterns used to resolve those problems
- Both paragraphs must be truthful — only reference issues documented in the week's records
- Plain text, past tense, formal tone, no markdown

---

## Condensing Strategy

The condensed entry is NOT a simple copy of the organized record. It is a **dense re-synthesis**:

1. **Merge related paragraphs** — if the organized record has separate paragraphs for "Bug Discovery" and "Bug Fix", combine into one dense section
2. **Preserve all quantitative details** — file counts, line counts, commit counts, test results
3. **Preserve all technology names** — exact framework/library/tool names
4. **Remove narrative connectors** — cut "The day commenced with..." style openings; get straight to the technical content
5. **Compress 8-10 organized paragraphs into 4-8 condensed sections** — each section covers more ground
6. **Keep section titles descriptive** — not generic ("Implementation") but specific ("Transfer Pipeline Architecture and Unit Standardisation")

---

## Privacy & Confidentiality

Same rules as organized records:
- Replace company/product names with generic terms
- Replace internal service names with descriptive names
- Never include credentials, IPs, or internal URLs
- Keep technology names and architecture patterns

---

## Truthfulness Principle

Every condensed entry must be strictly derived from the organized records. No fabrication, no invented metrics, no embellished details. If the source record is brief, the condensed entry is proportionally brief.

---

## Official IIT Activities List Reference

### 1. Involvement in Monitoring Production Activities
- 1.1 Conduct preliminary investigations
- 1.2 Carry out feasibility study

### 2. Systems Analysis
- 2.1 Analyze current system
- 2.2 Identify requirements and deficiencies of the existing system
- 2.3 Specify requirements of the proposed system

### 3. Systems Design
- 3.1 Design data
- 3.2 Design process outlines
- 3.3 User Interfaces/User experience/HCI/Screen formats and layouts
- 3.4 Prepare database/file specifications
- 3.5 Prepare program specifications
- 3.6 Prepare implementation plan including test plan and test data
- 3.7 Prepare user/operator manuals
- 3.8 Use complementary design techniques
- 3.9 UI Planning
- 3.10 Design/Develop Interactive UI Elements
- 3.12 User Experience and HCI

### 4. Programming
- 4.1 Program design
- 4.2 Program code
- 4.3 Test programs
- 4.4 Customization of package and software

### 5. Testing
- 5.1 Testing module
- 5.2 Integration
- 5.3 System testing
- 5.4 User acceptance

### 6. Implementation
- 6.1 Educate and train users
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

### 10-11. Networking / Network Management
- 10.1-10.4 Design, Implementation, Testing, Maintenance
- 11.1-11.3 Monitor functions, Monitor resources, Identify improvements

### 12. Project Management
- 12.1 Project planning
- 12.2 Document project performance
- 12.3 Review project with reference to organizational goals

### 16. General Administration / Management
- 16.1 Coordinating and liaising with various internal and external parties
- 16.2 Preparing reports
- 16.3 Conducting presentations

### 19. Cybersecurity
- 19.1 Conduct security assessments
- 19.2 Implement security protocols
- 19.3 Monitor security systems
- 19.4 Respond to security incidents
- 19.5 Develop security policies and procedures

### 20. Data Analysis and Management
- 20.1-20.5 Collect/clean data, Perform analysis, Create visualizations, Manage databases, Develop data models

### 22. Cloud Computing
- 22.1-22.5 Design cloud architecture, Implement solutions, Manage services, Optimize resources, Migrate applications

### 23. Software Development and DevOps
- 23.1 Develop software applications
- 23.2 Implement DevOps practices
- 23.3 Automate deployment processes
- 23.4 Monitor application performance
- 23.5 Conduct code reviews
- 23.6 Maintain version control

### 25. Blockchain and Cryptocurrency
- 25.1 Develop blockchain applications
- 25.2 Implement smart contracts
- 25.3 Monitor blockchain networks
- 25.4 Perform cryptocurrency analysis
- 25.5 Ensure blockchain security

---

## Example Condensed Entry (Reference)

```
Monday, February 3, 2026 — Multi-Service Bug Resolution: Supply Display Precision, Landing Page Styling and Blockchain Registration Pipeline

Objective: Resolve critical display precision errors in the admin dashboard supply visualization, fix Tailwind CSS v4 breaking changes on the wallet landing page, and repair the blockchain user registration pipeline ensuring approved users appear correctly in the administrative queue.

Supply Display Precision Architecture Correction
Investigation revealed a 1000x magnitude error in the admin dashboard's token supply visualization, displaying values in quintillions instead of trillions. Root cause traced to the backend returning raw 9-decimal precision values while the frontend applied additional scaling. Implemented precision-aware formatting that correctly interprets the backend's nano-token units. Additionally resolved runtime TypeError caused by null pool access when system pools had not yet been initialized, adding defensive null checks throughout the supply statistics components. Updated the phase distribution visualization to support 8 phases including the three Phase 1 sub-phases (1A Early Adopters, 1B Growth, 1C Maturity) reflecting the tiered early-adopter reward structure defined in the tokenomics specification.

Outcome: Three critical production issues resolved across four repositories with 47 commits. Supply display now correctly shows trillion-scale values with proper precision handling. Landing page restored to full visual fidelity under Tailwind CSS v4. Blockchain registration pipeline repaired by adding onchain status tracking during KYC approval, with data migration applied to existing approved users. All fixes verified through manual testing and browser automation.

Activity No:
2.1 Analyze current system
2.2 Identify requirements and deficiencies of the existing system
3.3 User Interfaces/User experience/HCI/Screen formats and layouts
4.2 Program code
5.2 Integration
8.4 Security
9.1 Document and/or update documentation
23.2 Implement DevOps practices
23.5 Conduct code reviews
25.1 Develop blockchain applications
```

---

## Quality Checklist

Before finalizing the condensed file:
- [ ] Header line matches `WEEK {NN} — {MONTH} {START}-{END}, {YEAR} — GOOGLE SHEETS CONDENSED ENTRIES`
- [ ] Each day separated by `---`
- [ ] Zero markdown formatting (no `**`, no `#`, no bullets, no backticks)
- [ ] Section titles are plain text on their own line
- [ ] All paragraphs are dense flowing prose in past tense
- [ ] Activity No section present for every work day
- [ ] Activity codes use official IIT names
- [ ] Day-off entries are minimal one-liners
- [ ] PROBLEMS ENCOUNTERED paragraph present at end
- [ ] SOLUTIONS FOUND paragraph present at end
- [ ] Both paragraphs are truthful and reference actual documented issues
- [ ] All quantitative details preserved from source records
- [ ] No proprietary information exposed
- [ ] Professional academic tone throughout
