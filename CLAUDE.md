# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

This is an **IIT Industrial Placement Record Book** repository for tracking daily work activities during an Informatics Institute of Technology (IIT) placement year. The work documented here primarily involves the "GX Coin Protocol" - a productivity-based currency built on Hyperledger Fabric blockchain technology, including backend services, frontend wallet development, and Kubernetes infrastructure.

## Primary Function

Claude Code acts as a **technical scribe** that transforms raw daily work logs, git commits, and technical notes into formal, professional entries following IIT's required format and writing style standards.

---

## Repository Structure

The repository follows a strict hierarchical organization:

```
work-records/
├── October-2025/
│   ├── week-01/
│   │   └── whole-week.md
│   ├── week-02/
│   │   ├── 12-oct-2025.md
│   │   ├── 13-oct-2025.md
│   │   └── ...
│   └── week-03/
├── November-2025/
│   ├── week-01/
│   │   ├── 02-nov-2025.md
│   │   └── ...
│   └── week-02/
```

### Organizational Hierarchy
1. **Month folders**: Named as `{Month}-{Year}` (e.g., `October-2025`, `November-2025`)
2. **Week folders**: Named as `week-{NN}` where NN is the week number (e.g., `week-01`, `week-02`)
3. **Daily files**: Named as `DD-mon-YYYY.md` (e.g., `12-oct-2025.md`, `02-nov-2025.md`)
   - Day: Two digits (01-31)
   - Month: Three-letter lowercase abbreviation (jan, feb, mar, apr, may, jun, jul, aug, sep, oct, nov, dec)
   - Year: Four digits
   - Exception: Week summary files may be named `whole-week.md`

---

## Required Entry Structure (6-Part Format)

Every daily entry MUST follow this exact structure:

### 1. Date
**Format:** Day of week, Month Day, Year
**Example:** "Sunday, November 2nd, 2025" or "Monday, October 26, 2025"

### 2. Title
A concise, professional title summarizing the main achievement or focus of the day.

**Guidelines:**
- Use clear, technical terminology
- Include key technologies or systems worked on
- Format: "Primary Focus: Specific Achievement/Task"

**Examples:**
- "Phase 4D Breakthrough: K8s External Chaincode Deployed with Built-in CCAAS"
- "AI Assistant Integration, Production Infrastructure Hardening & Operational Scripting"
- "Foundational Chaincode Refactor: Implementing User/Organization Account Model"

### 3. Objective
A 2-4 sentence statement defining the primary goal of the day's work.

**Guidelines:**
- Start with "To [achieve/execute/implement/develop/resolve]..."
- Clearly state the technical problem being solved
- Mention the architectural or business context
- Include expected outcomes or deliverables

**Structure Pattern:**
```
To [action verb] [technical component/system] by [specific approach/method],
[additional context about why this work matters], and [documentation/validation
activities].
```

**Example:**
```
To resolve the critical 95% completion blocker—the persistent gRPC handshake
failure—for the Kubernetes external chaincode deployment (Phase 4D). The goal
was to pivot from the failing custom "external" builder to Hyperledger Fabric's
built-in "Chaincode-as-a-Service" (CCAAS) builder, achieve 100% network
operationality, and create the final testing documentation.
```

### 4. Work Carried Out
**⚠️ MOST CRITICAL SECTION - This is where quality matters most**

This section should be a **detailed, flowing narrative** that tells the story of your technical work. This is NOT a bullet-point list of commits.

#### Writing Style Requirements:

**✅ DO:**
- Write in **bold-headed paragraphs** with clear topic headers
- Use professional, enterprise-grade technical language
- Create a **narrative flow** that connects different work items
- Explain WHAT was done, WHY it was necessary, and HOW challenges were overcome
- Use precise technical terminology (Kubernetes, Hyperledger Fabric, gRPC, StatefulSets, SHA-256, POSIX, etc.)
- Include specific details: function names, parameter counts, constants, file names
- Explain architectural decisions and their impacts
- Describe validation approaches and error handling
- Mention code organization and refactoring strategies
- Include quantitative details (e.g., "5 parameters to 4 parameters", "577.5B coins", "64-character hex string")

**❌ DON'T:**
- Use bullet points or numbered lists
- Simply list commit messages
- Write in a mechanical, repetitive style
- Skip explaining the "why" behind technical decisions
- Use vague descriptions without context
- Ignore connections between different work items

#### Narrative Structure Pattern:

Each major work item should be a **paragraph with a bold header**. The paragraph should explain:
1. The context/problem
2. The technical approach taken
3. Specific implementation details
4. Results or implications

**Example Structure:**
```
**Core Architectural Refactor (User vs. Organization):** A critical architectural
flaw was identified and corrected. The chaincode was refactored to enforce clear
separation between individual human accounts (User) and organizational entities
(Organization). The User struct's AccountType field was completely removed,
establishing the principle that all User accounts are, by definition, individuals
verified through KYC and biometric authentication. This eliminated the architectural
contradiction where business entities could be represented as "users."

**Breaking API Change 1 (CreateUser):** The CreateUser function signature was
changed from 5 parameters to 4, removing the accountType parameter entirely. The
function now exclusively creates individual user identities. Old signature:
`CreateUser(ctx, userID, biometricHash, nationality, age, accountType)`. New
signature: `CreateUser(ctx, userID, biometricHash, nationality, age)`. This forces
proper separation between personal and organizational accounts at the API level.
```

#### Typical Section Headers for Different Work Types:

**For Architectural Refactoring:**
- Core Architectural Refactor
- Breaking API Changes
- Data Model Restructuring
- Constants Reorganization
- Logic Simplification

**For Feature Implementation:**
- Feature Design and Planning
- Core Implementation
- Validation and Error Handling
- Integration with Existing Systems
- Testing and Verification

**For Bug Fixes:**
- Issue Identification
- Root Cause Analysis
- Implementation of Fix
- Validation and Testing
- Regression Prevention

**For DevOps/Infrastructure:**
- Infrastructure Design
- Deployment Configuration
- Automation Script Development
- Testing and Validation
- Monitoring and Alerting Setup

**For Documentation:**
- Documentation Architecture
- Technical Specification Writing
- API Reference Updates
- Migration Guide Development

### 5. Outcome
A 1-2 sentence summary that wraps up the day's accomplishments and states the current project status.

**Guidelines:**
- Summarize the key achievement(s)
- Mention any breaking changes or major impacts
- State what is now possible/improved/fixed
- Can mention next steps if relevant

**Example:**
```
Phase 4D and the entire K8s migration of the Fabric network are 100% complete.
The gxtv3 chaincode is now fully operational as a standalone, K8s-native service
using Fabric's built-in CCAAS builder, and a complete testing guide for all 30
functions has been created.
```

### 6. Activities Covered
Map your work to the official IIT activity categories. List 6-10 relevant activity codes.

**Format:**
```
[Category].[Subcategory] [Brief description of how this applies]
```

**Example:**
```
2.1 Analyze current system
2.2 Identify requirements and deficiencies of the existing system
3.2 Design process outlines (Pivoting from custom builder to CCAAS)
4.2 Program code (Go main.go, YAML manifests)
5.2 Integration (Peer-to-CCAAS integration)
6.4 Installation of software (Deployment of CCAAS chaincode)
9.1 Document and/or update documentation (Testing Guide, Completion Reports)
18.1 Troubleshoot and debug production systems (Root cause analysis)
23.3 Automate deployment processes (K8s manifests)
25.2 Implement smart contracts (Deploying and bootstrapping)
```

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

## Writing Quality Standards

### Professional Language Characteristics:
- **Precise technical terminology**: Use exact names (SHA-256, not "hash function"; Hyperledger Fabric, not "blockchain platform")
- **Quantitative details**: Include specific numbers (5→4 parameters, 64 characters, 577.5B coins)
- **Architectural vocabulary**: Use terms like "refactoring", "breaking changes", "validation", "migration", "backwards compatibility"
- **Causal language**: Explain relationships with "as a result", "this ensures", "to prevent", "which enables"
- **Enterprise tone**: Professional but not overly formal; technical but readable

### Narrative Flow Techniques:
1. **Logical grouping**: Group related commits into coherent themes
2. **Progressive detail**: Start with high-level change, then explain specifics
3. **Cause and effect**: Explain why changes were needed before describing them
4. **Technical depth**: Include function signatures, parameter names, file names
5. **Impact statements**: End each paragraph with the result or benefit

### Example of Good vs. Poor Writing:

**❌ Poor (bullet-point style):**
```
- Removed AccountType from User struct
- Updated CreateUser function
- Added validation
- Wrote documentation
```

**✅ Good (narrative style):**
```
**Core Architectural Refactor (User vs. Organization):** A critical architectural
flaw was identified and corrected. The chaincode was refactored to enforce clear
separation between individual human accounts (User) and organizational entities
(Organization). The User struct's AccountType field was completely removed,
establishing the principle that all User accounts are, by definition, individuals
verified through KYC and biometric authentication. This eliminated the architectural
contradiction where business entities could be represented as "users."
```

---

## Common Work Patterns & Documentation Approach

### Pattern 1: Architectural Refactoring
**Typical commits:** Struct changes, function signature updates, constant renames
**Narrative focus:**
- Explain the architectural problem being fixed
- Describe the old vs. new structure
- Detail breaking changes and migration impact
- Mention backwards compatibility considerations

### Pattern 2: Feature Development
**Typical commits:** Add new functions, implement business logic, create endpoints
**Narrative focus:**
- Explain what the feature does and why it's needed
- Describe the technical approach (algorithms, data structures)
- Detail integration with existing systems
- Mention testing and validation approaches

### Pattern 3: Bug Fixes
**Typical commits:** Fix edge cases, correct calculations, handle errors
**Narrative focus:**
- Describe how the bug was discovered
- Explain the root cause
- Detail the fix and why it works
- Mention prevention measures (tests, validation)

### Pattern 4: DevOps/Infrastructure
**Typical commits:** Update configs, add scripts, modify deployment
**Narrative focus:**
- Explain the operational challenge
- Describe the automation/tooling solution
- Detail configuration changes
- Mention testing and rollback strategies

### Pattern 5: Documentation
**Typical commits:** Update docs, add guides, create diagrams
**Narrative focus:**
- Explain what was unclear or missing
- Describe the documentation structure created
- Mention target audience and use cases
- Detail completeness (page counts, sections covered)

---

## Common Work Themes

The work records document development across these technical areas:

- **Blockchain/Chaincode**: Hyperledger Fabric smart contracts, identity management, tokenomics, governance
- **Backend Services**: Node.js microservices, CQRS patterns, event-driven architecture, database synchronization
- **Infrastructure**: Kubernetes deployments, Docker configurations, CI/CD automation
- **DevOps**: Deployment scripts, network lifecycle automation, health checks
- **Documentation**: Technical guides, system reports, testing procedures, troubleshooting guides

---

## File Operations

### Creating New Entries

When creating a new daily entry:
1. Determine the correct month and week folder (create if needed using `mkdir -p`)
2. Use the exact naming format: `DD-mon-YYYY.md`
3. Start with the day of week and full date on line 1
4. Follow the 6-part structured format outlined above
5. Write the "Work Carried Out" section in narrative paragraph form with bold headers
6. Ensure activity codes are relevant and accurate (6-10 codes)

### Navigating Entries

- Most recent work is in the latest month/week folders
- Files are named chronologically, making them easy to sort
- Each entry is self-contained but may reference previous days' work
- Use `ls -lt` to see files sorted by modification time

---

## Quick Reference: The 6-Part Structure

```markdown
[Day of week, Month Day, Year]
Title: [Technical Focus: Specific Achievement]
Objective:
To [action] [component] by [method], [context], and [validation].

Work Carried Out:
**[Topic Header 1]:** [Detailed narrative paragraph explaining context,
approach, implementation, and results...]

**[Topic Header 2]:** [Detailed narrative paragraph...]

[Continue with 5-8 major topic paragraphs, each 3-8 sentences long]

Outcome:
[1-2 sentence summary of achievements and current status]

Activities Covered:
[Code].[Subcode] [Description]
[Code].[Subcode] [Description]
[6-10 relevant activity codes]
```

---

## Important Guidelines

### ✅ When Creating Entries:
- Use bold-headed paragraphs for "Work Carried Out" section
- Write in narrative form, not bullet points
- Include precise technical terminology
- Provide quantitative details (parameter counts, line counts, file sizes)
- Explain the "why" behind technical decisions
- Group related work into coherent themes
- Use causal language to show relationships
- Include function signatures, file names, constants
- Mention validation and testing approaches
- State architectural impacts and breaking changes

### ❌ Avoid:
- Bullet-point lists in "Work Carried Out" section
- Vague descriptions without technical context
- Simply listing commit messages
- Skipping the reasoning behind decisions
- Generic statements without specifics
- Forgetting to mention documentation work
- Omitting error handling or validation details
- Missing quantitative measurements

---

## Context for Evaluation

### About IIT Industrial Placement:
This is for an **Industrial Placement Record Book** required by the Informatics Institute of Technology (IIT). The log entries need to be:
- **Formal and professional** (suitable for academic evaluation)
- **Technically detailed** (demonstrates substantial engineering work)
- **Well-structured** (follows the 6-part format consistently)
- **Narrative-driven** (tells the story of your work, not just lists tasks)

### Evaluation Criteria:
Record book entries are evaluated on:
1. **Technical depth**: Demonstrated understanding of complex systems
2. **Problem-solving**: Clear explanation of challenges and solutions
3. **Professional communication**: Enterprise-grade technical writing
4. **Completeness**: All work properly documented with context
5. **Consistency**: Regular, detailed entries throughout placement

### Target Audience:
- IIT academic supervisors
- Industry placement supervisors
- Future employers reviewing technical capabilities
- Personal reference for lessons learned