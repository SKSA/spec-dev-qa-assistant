# Spec Dev QA Assistant

> Your AI-powered QA assistant for spec-machine and Claude Code

Automate your entire QA workflow from acceptance criteria to verified tests with JIRA integration and quality tracking.

---

## 🎯 What It Does

**Spec Dev QA Assistant** streamlines your testing workflow by automating:

- ✅ **Multi-source AC Detection** - Automatically fetch ACs from JIRA, Confluence, and Figma
- ✅ **Automated Test Generation** - Generate E2E, API, and unit tests from acceptance criteria
- ✅ **Test User Creation** - Create test users in staging via backend APIs (no browser automation)
- ✅ **X-Ray Test Case Generation** - Create X-Ray test cases directly from JIRA ticket ACs
- ✅ **Squad Risk Analysis** - Calculate quality scores and identify risk areas by squad
- ✅ **Platform Support** - Works with Web (Playwright, Cypress) and Mobile React Native (Maestro, Detox)
- ✅ **Smart Selector Scanning** - Map test selectors across your codebase
- ✅ **Interactive Verification** - Step-by-step AC verification with test execution
- ✅ **JIRA Integration** - Post verification results directly to tickets
- ✅ **Quality Tracking** - Track team performance and identify trends over time
- ✅ **spec-machine Integration** - Works seamlessly with HelloFresh's spec-machine for complete workflow

---

## 🚀 Quick Start (5 minutes)

### 1. Clone the Repository

```bash
git clone git@github.com:mde-hf/spec-dev-qa-assistant.git
cd spec-dev-qa-assistant
```

### 2. Install to Your Project

```bash
cd ~/your-project
cp -r ~/spec-dev-qa-assistant/.claude .
```

### 3. Restart Claude Code

Restart Claude Code to load the new commands.

### 4. Try It Out

```bash
# In Claude Code chat
/collect-ac EPS-1234
```

That's it! 🎉

---

## 📋 Available Commands

| Command | Description |
|---------|-------------|
| `/collect-ac` | Collect acceptance criteria from JIRA, Confluence, and Figma |
| `/generate-e2e-tests` | Generate automated tests (Web: Playwright/Cypress, Mobile: Maestro/Detox) |
| `/create-test-user` | Create US test users in staging environment via backend APIs |
| `/generate-xray-tests` | Generate X-Ray test cases directly from JIRA ticket acceptance criteria |
| `/document-tests` | Generate interactive HTML test coverage dashboard with visual gap analysis |
| `/dev-risk-analysis` | Analyze developer quality metrics and risk scores by squad or JIRA project |
| `/pr-review` | Deep PR code review with Conventional Comments and 2-why hard gate discipline |
| `/verify-ac` | Interactively verify each acceptance criterion |
| `/post-to-jira` | Post verification results to JIRA ticket (auto-prompted after verify) |
| `/figma-ac-extractor` | Extract user flow ACs from Figma designs |
| `/ac-quality-trends` | Track quality metrics across sprints |
| `/setup-qa-assistant` | Setup and validate dependencies (JIRA CLI, Figma MCP, etc.) |

### **Integrates with spec-machine:**

| spec-machine Command | When to Use |
|---------------------|-------------|
| `/analyze-tech-stack` | One-time setup: Analyze project dependencies |
| `/train-context` | One-time setup: Train on repo patterns |
| `/jira-gather-requirements` | Before `/collect-ac`: Fetch ticket requirements |
| `/create-pr` | After `/verify-ac`: Create PR with JIRA link & labels |
| `/a11y-check` | Optional: Check Figma design accessibility |

**Note:** For test ID conventions and patterns, spec-machine's `testing/test-ids-web` and `testing/test-patterns-web` skills provide coding standards that work alongside these commands.

---

## 🔄 Complete Workflow

### **Prerequisites (One-Time Setup)**

```bash
# Setup spec-machine context (if using spec-machine)
/analyze-tech-stack         # Analyze project dependencies
/train-context              # Train on your repo patterns
```

### **Per-Ticket Workflow**

```bash
# 1️⃣ Gather requirements and acceptance criteria
/jira-gather-requirements EPS-1234  # spec-machine: Fetch ticket & requirements
/collect-ac EPS-1234                # Collect ACs from JIRA + Confluence + Figma
→ Fetches ACs from all sources
→ Smart detection with confidence scoring
→ Creates AC checklist

# 2️⃣ (Optional) Add visual ACs from Figma
/figma-ac-extractor https://figma.com/design/xyz
→ Extracts user flows and interactions
→ Merges with existing ACs

# 3️⃣ Create branch (if not already)
git checkout -b feature/EPS-1234-description

# 4️⃣ Implement your feature
# ... write your code ...
# Build the feature according to ACs

# 5️⃣ Generate tests for your implementation
/generate-e2e-tests EPS-1234
→ Choose: Auto (use detected settings) or Manual (choose yourself)
→ Select platform: Web or Mobile React Native
→ Choose framework (Playwright/Maestro/etc.)
→ Generates tests based on your implementation + ACs
→ Follows spec-machine test patterns automatically

# 6️⃣ Run tests
npx playwright test                     # Web
# or
maestro test .maestro/EPS-1234.yaml     # React Native

# 7️⃣ Generate test documentation (optional)
/document-tests EPS-1234
→ Analyzes test coverage across all layers
→ Generates interactive HTML dashboard
→ Shows gaps with priority matrix
→ Updates automatically on future test changes

# 8️⃣ Verify ACs
/verify-ac EPS-1234
→ Interactive verification
→ Test status tracking
→ Asks: "Post to JIRA?" → Auto-posts results

# 9️⃣ Create PR with full context
/create-pr  # spec-machine: Creates PR with JIRA link, labels, template
→ Links JIRA ticket automatically
→ Uses repo's PR template
→ Applies correct labels
→ Follows team conventions

# 🔟 (End of sprint) Track quality
/ac-quality-trends --sprint "Sprint 24"
→ See pass rates, velocity, issues
→ Developer leaderboard
→ Recommendations
```

---

## 🎯 Key Features

### **Multi-Source AC Detection**

Automatically fetches ACs from:
- **JIRA** - Custom fields, description, sub-tasks, comments
- **Confluence** - Linked pages with smart parsing
- **Figma** - User flows, interactions, form behaviors

Smart parsing handles:
- BDD format (Given/When/Then)
- Bullet lists
- Numbered lists
- Tables
- Checkboxes

### **Framework Flexibility**

Generates tests for:
- **E2E:** Playwright, Cypress
- **Unit/Integration:** Jest, Vitest, Mocha
- **Component:** React Testing Library, Vue Test Utils
- **API:** Supertest, REST Assured
- **Performance:** Lighthouse
- **Accessibility:** Axe-core

### **Smart Selector Scanning**

Automatically scans for:
- `data-testid`
- `data-cy`
- `testID` (React Native)
- `aria-label`
- CSS classes (fallback)

Generates quality report with auto-fix suggestions.

### **JIRA Integration**

Posts formatted comments with:
- AC status (passed/failed/needs QA)
- Test results (E2E, API, unit)
- Code coverage metrics
- Test evidence
- Links to reports

Auto-updates ticket status and labels.

### **Quality Tracking**

Tracks across sprints:
- Pass rates
- Test coverage by type
- Time to verify
- Issue patterns
- Source quality
- Developer performance

### **Developer Risk Analysis**

Calculate quality metrics by squad or JIRA project:
- Developer confidence scores (0-100)
- Bug fix tracking (last 6 months)
- Legacy code risk assessment
- Critical bug detection
- Interactive dashboard with drill-down
- Squad/project comparisons
- Real names + GitHub usernames
- Tooltips explaining all metrics

### **X-Ray Test Case Generation**

Automate X-Ray test creation:
- Extract ACs from JIRA tickets
- Generate BDD or Manual format test cases
- Automatic linking to source ticket
- Dry-run mode for preview
- Batch creation from multiple ACs
- Follows JIRA CLI approach (spec-machine standard)

---

## 📦 What's Included

### Commands
- `collect-ac/` - Multi-source AC fetching
- `verify-ac/` - Interactive verification
- `generate-e2e-tests/` - Test generation
- `generate-xray-tests/` - X-Ray test case generation from JIRA ACs
- `document-tests/` - Interactive test coverage dashboard
- `dev-risk-analysis/` - Developer risk analysis and quality metrics
- `pr-review/` - Deep PR code review with Conventional Comments
- `post-to-jira/` - JIRA integration
- `figma-ac-extractor/` - Figma integration
- `ac-quality-trends/` - Quality metrics
- `setup-qa-assistant/` - Dependency setup and configuration

### Documentation
- `README.md` - This file
- `CHANGELOG.md` - Version history and updates
- `docs/QUICK-START.md` - 5-minute guide
- `docs/AC-WORKFLOW-README.md` - Complete workflow guide
- `docs/IMPROVEMENTS-V2.md` - Feature descriptions
- `docs/FIGMA-AC-FOCUS.md` - Figma usage guide
- `docs/INDEX.md` - Navigation guide
- `docs/HOW-TO-SHARE.md` - Sharing guide

---

## 🔧 Prerequisites

### Required
- **Claude Code** - AI coding assistant
- **JIRA CLI** - For JIRA integration
  ```bash
  # Install if needed
  brew install ankitpokhrel/jira-cli/jira-cli
  jira init
  ```

### Optional
- **Figma MCP** - For Figma integration
  ```bash
  # In Claude Code chat
  /add-plugin figma
  ```
- **Playwright** - For E2E test generation
  ```bash
  npm install -D @playwright/test
  ```

---

## 📚 Documentation

- **[Quick Start](docs/QUICK-START.md)** - Get started in 5 minutes
- **[Complete Workflow Guide](docs/AC-WORKFLOW-README.md)** - Detailed workflow documentation
- **[Latest Updates](CHANGELOG.md)** - What's new in v2.1
- **[Feature Descriptions](docs/IMPROVEMENTS-V2.md)** - Deep dive into features
- **[Figma Integration](docs/FIGMA-AC-FOCUS.md)** - How Figma extraction works
- **[Full Index](docs/INDEX.md)** - Navigate all documentation

---

## 💡 Examples

### Example 1: Simple Feature

```bash
/collect-ac EPS-5678
→ Found 3 ACs from JIRA

# Implement the feature
# ... write code to satisfy the ACs ...

/generate-e2e-tests EPS-5678
→ Scans your implementation
→ Select framework: 1 (Playwright)
→ Generated 3 E2E tests based on your code

# Run tests
npx playwright test

/verify-ac EPS-5678
→ All passed ✅
→ Post to JIRA? Yes
→ Posted to EPS-5678
```

### Example 2: Complex Flow with Figma

```bash
/collect-ac EPS-9012
→ Found 5 ACs from JIRA + Confluence

/figma-ac-extractor https://figma.com/design/abc123
→ Added 4 interaction ACs from Figma
→ Total: 9 ACs

# Implement the feature
# ... write code for all 9 ACs ...

/generate-e2e-tests EPS-9012
→ Scanned selectors: 87% coverage in your code
→ Generated 9 tests (Playwright + Jest)

# Run tests
npx playwright test

/verify-ac EPS-9012
→ 8 passed, 1 needs QA
→ Posted to JIRA
```

### Example 3: Create Test Users for E2E

```bash
# Create new US test user (no subscription)
/create-test-user

→ 🎯 Creating test user...
→ ✅ User account created (ID: 12345678)
→ ✅ Test user created successfully!

📧 Email: test-user-1735564800@hellofresh.com
🔑 Password: qwerty123

# Create active US user with subscription
/create-test-user --state active

→ ✅ Subscription created (ID: 87654321)
📦 Plan: 2 meals for 2 people
📊 State: active
```

### Example 4: Sprint Retrospective

```bash
/ac-quality-trends --sprint "Sprint 24"

→ 18 tickets completed
→ 94% pass rate ⬆️ (+5%)
→ 2.3h avg verification time ⬇️
→ Top issues: Missing selectors (4)
→ Recommendations: Increase visual testing
```

### Example 4: Generate X-Ray Test Cases

```bash
/generate-xray-tests LMISSIONS-1358 --dry-run

→ Fetching ticket from JIRA...
→ Extracted 4 acceptance criteria
→ Generating X-Ray test cases (BDD format)
→ Preview: Would create 4 Test issues linked to LMISSIONS-1358
→ Run without --dry-run to create actual test cases
```

### Example 5: Developer Risk Analysis

```bash
/dev-risk-analysis

→ Enter squad name or JIRA project key: SSX
→ Analyzing last 6 months of PRs...
→ Found 15 developers, 87 PRs, 12 bug fixes
→ Generated dashboard: ~/hellofresh-web/.dev-risk-analysis/index.html
→ Squad Average Score: 82/100
→ Top Risk: Legacy code changes (3 critical areas)
```

---

## 🎯 Benefits

### For Developers
- ⚡ **2+ hours saved** per ticket
- 🤖 **Automated test generation** - No more manual test writing
- 🎯 **Clear acceptance criteria** - Know exactly what to build
- ✅ **Confidence** - Tests written before implementation

### For QA
- 📋 **Structured verification** - Step-by-step checklist
- 🔍 **Complete coverage** - E2E, API, unit, visual tests
- 📊 **Quality metrics** - Track team performance
- 🔄 **Automated updates** - Results post to JIRA automatically

### For Teams
- 📈 **Visibility** - Quality trends and velocity tracking
- 🏆 **Accountability** - Developer leaderboards
- 🎯 **Process improvement** - Identify bottlenecks
- 🤝 **Consistency** - Standardized workflow

---

## 🔄 Updates

### v2.2 (April 6, 2026)
- ✨ **NEW:** `/generate-xray-tests` - Create X-Ray test cases from JIRA ACs
- ✨ **NEW:** `/dev-risk-analysis` - Developer quality metrics and risk scores
- ✅ Developer risk dashboard with tooltips and squad selector
- ✅ Test coverage dashboard with comprehensive tooltips
- ✅ JIRA CLI integration following spec-machine standards
- ✅ Interactive dashboards with HelloFresh brand colors

### v2.1 (March 22, 2026)
- ✅ JIRA post now auto-prompted after verify-ac
- ✅ Framework selection simplified (shows detected only)
- ✅ Smart selector scan integrated into generate-e2e-tests
- ✅ Figma extractor focuses on user flows (not visual styling)

See [CHANGELOG.md](CHANGELOG.md) for full history.

---

## 🤝 Contributing

Found a bug? Have a suggestion? Want to add a feature?

1. Create an issue
2. Submit a PR
3. Share feedback in #spec-dev-qa-assistant
