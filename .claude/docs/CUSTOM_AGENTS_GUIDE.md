# Custom Agents & Documentation Guide

This guide explains how to create custom agent configurations and documentation files that Playwright agents will use.

---

## Directory Structure

```
.claude/
├── agents/                          # Agent configurations
│   ├── playwright-test-generator.md           # Default generator
│   ├── playwright-test-generator-custom.md    # ✨ Custom generator (uses POM)
│   ├── playwright-test-planner.md
│   └── playwright-test-healer.md
│
├── docs/                            # Reference documentation
│   ├── page-object-model-pattern.md           # ✨ POM pattern guide
│   ├── test-generation-patterns.md            # Test patterns
│   └── custom-selectors-guide.md              # Selector strategies
│
└── settings.json                    # Permissions & config
```

---

## Part 1: Creating Custom Agent Configurations

### Agent File Structure

Agent configuration files use YAML frontmatter + Markdown content:

```markdown
---
name: your-custom-agent-name
description: 'What this agent does'
tools: Tool1, Tool2, Tool3
model: sonnet
color: blue
---

Agent instructions in markdown...
```

### Required Fields

| Field | Description | Example |
|-------|-------------|---------|
| `name` | Unique agent identifier | `playwright-test-generator-custom` |
| `description` | What the agent does | `'Generates tests using POM pattern'` |
| `tools` | Tools the agent can use | `Glob, Grep, Read, Write, mcp__playwright-test__*` |
| `model` | Claude model to use | `sonnet`, `opus`, `haiku` |
| `color` | UI color indicator | `red`, `green`, `blue`, `purple` |

### Available Tools

Common tools for test agents:

**File Operations:**
- `Glob` - Find files by pattern
- `Grep` - Search file contents
- `Read` - Read files
- `Write` - Create/overwrite files
- `Edit` - Edit existing files
- `LS` - List directory contents

**Playwright MCP Tools:**
- `mcp__playwright-test__browser_*` - Browser interactions
- `mcp__playwright-test__test_*` - Test execution
- `mcp__playwright-test__planner_*` - Test planning
- `mcp__playwright-test__generator_*` - Test generation

---

## Part 2: Creating Reference Documentation

### Documentation Files Location

Store reference documentation in `.claude/docs/`:

```
.claude/docs/
├── page-object-model-pattern.md     # POM pattern
├── test-patterns.md                 # Common test patterns
├── selector-strategy.md             # Selector guidelines
├── fixtures-guide.md                # Custom fixtures
└── constants-guide.md               # Constants usage
```

### What to Document

1. **Architecture Patterns**
   - Page Object Model structure
   - Test file organization
   - Fixture patterns
   - Constants/config patterns

2. **Code Examples**
   - Correct usage examples
   - Anti-patterns to avoid
   - Common scenarios

3. **Project-Specific Rules**
   - Naming conventions
   - Import patterns
   - Test structure
   - Assertion patterns

### Documentation Template

```markdown
# {Topic Name}

Brief description of what this documents.

---

## Overview

Explain the concept/pattern.

## Structure

Show the structure/architecture.

## Examples

### ✅ Correct Usage

\`\`\`typescript
// Good example
\`\`\`

### ❌ Wrong Usage

\`\`\`typescript
// Bad example with explanation
\`\`\`

## Rules for Agents

When generating code, follow these rules:
1. Rule 1
2. Rule 2
3. Rule 3

## Common Patterns

Pattern 1, Pattern 2, etc.

## Summary

Key takeaways.
```

---

## Part 3: Configuring Custom Agents

### Step 1: Create Agent Configuration

Create `.claude/agents/your-agent-name.md`:

```markdown
---
name: playwright-test-generator-custom
description: 'Custom test generator using POM pattern'
tools: Glob, Grep, Read, Write, mcp__playwright-test__*
model: sonnet
color: purple
---

You are a custom test generator that follows the Page Object Model pattern.

# Critical Requirements

**BEFORE GENERATING TESTS**, you MUST:

1. Read('.claude/docs/page-object-model-pattern.md')
2. Read('pages/LoginPage.ts')
3. Read('fixtures/pages.ts')

# Rules

1. **ALWAYS** import from fixtures, not @playwright/test
2. **NEVER** use raw locators in tests
3. **ALWAYS** use page object methods
4. **GROUP** related tests in same file
5. **USE** beforeEach for shared setup

# Example Generated Test

\`\`\`typescript
import { test } from '../../../fixtures/pages';
import { USERS } from '../../../utils/constants';

test.describe('Feature', () => {
  test.beforeEach(async ({ loginPage }) => {
    await loginPage.goto();
    await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
  });

  test('scenario 1', async ({ inventoryPage }) => {
    await inventoryPage.expectToBeVisible();
  });

  test('scenario 2', async ({ inventoryPage }) => {
    await inventoryPage.expectProductCount(6);
  });
});
\`\`\`
```

### Step 2: Create Reference Documentation

Create `.claude/docs/page-object-model-pattern.md`:

```markdown
# Page Object Model Pattern

## Structure

\`\`\`typescript
export class PageName {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  // Locators
  get element(): Locator {
    return this.page.locator('[data-test="element"]');
  }

  // Actions
  async performAction(): Promise<void> {
    await this.element.click();
  }

  // Assertions
  async expectToBeVisible(): Promise<void> {
    await expect(this.element).toBeVisible();
  }
}
\`\`\`

## For Agents

When generating tests:
1. Import from fixtures
2. Use page object methods
3. No raw locators
```

### Step 3: Reference Documentation in Agent

In your agent configuration, instruct the agent to read documentation:

```markdown
# Critical Requirements

**BEFORE GENERATING**, you MUST:

\`\`\`typescript
// Read pattern documentation
await Read('.claude/docs/page-object-model-pattern.md');

// Read existing implementations
await Read('pages/LoginPage.ts');
await Read('fixtures/pages.ts');

// Read example tests
await Read('tests/e2e/authentication/login.spec.ts');
\`\`\`
```

---

## Part 4: Using Custom Agents

### Option 1: Invoke by Name

When talking to Claude Code:

```
"Use the playwright-test-generator-custom agent to generate tests for login flow"
```

Claude will automatically invoke the custom agent.

### Option 2: Let Claude Choose

Describe what you want:

```
"Generate tests using the Page Object Model pattern for cart management"
```

If your agent description matches, Claude will use it.

### Option 3: Default Behavior

To make your custom agent the default, rename it to replace the original:

```bash
# Backup original
mv .claude/agents/playwright-test-generator.md .claude/agents/playwright-test-generator-original.md

# Use custom as default
mv .claude/agents/playwright-test-generator-custom.md .claude/agents/playwright-test-generator.md
```

---

## Part 5: Advanced Patterns

### Pattern 1: Multiple Documentation Files

```markdown
# In agent config
**BEFORE GENERATING**, read these:

1. Read('.claude/docs/page-object-model-pattern.md')
2. Read('.claude/docs/test-patterns.md')
3. Read('.claude/docs/selector-strategy.md')
4. Read('.claude/docs/fixtures-guide.md')
```

### Pattern 2: Conditional Documentation

```markdown
# In agent config
**Based on the test type**, read appropriate docs:

- For login tests: Read('.claude/docs/authentication-patterns.md')
- For cart tests: Read('.claude/docs/cart-patterns.md')
- For checkout tests: Read('.claude/docs/checkout-patterns.md')
```

### Pattern 3: Code Examples Library

Create `.claude/docs/examples/` with full example files:

```
.claude/docs/examples/
├── login-test-example.spec.ts
├── cart-test-example.spec.ts
└── checkout-test-example.spec.ts
```

Reference in agent:
```markdown
Read example test files:
- Read('.claude/docs/examples/login-test-example.spec.ts')
- Read('.claude/docs/examples/cart-test-example.spec.ts')
```

### Pattern 4: Templates

Create `.claude/templates/` with code templates:

```
.claude/templates/
├── test-with-beforeeach.template.ts
├── test-multiple-scenarios.template.ts
└── page-object.template.ts
```

### Pattern 5: Validation Rules

Include validation in agent config:

```markdown
# After Generating Test

Validate the generated code:

- [ ] Imports from fixtures, not @playwright/test
- [ ] No raw page.locator() calls
- [ ] Uses page object methods
- [ ] Has beforeEach for shared setup
- [ ] Uses constants from utils/constants
- [ ] Multiple related tests in one file
- [ ] Follows naming: tests/e2e/{feature}/{functionality}.spec.ts

If any check fails, regenerate following the rules.
```

---

## Part 6: Real-World Examples

### Example 1: Custom Test Generator for Your Project

**File:** `.claude/agents/saucedemo-test-generator.md`

```markdown
---
name: saucedemo-test-generator
description: 'Generates tests for Saucedemo using POM pattern with beforeEach hooks and grouped scenarios'
tools: Glob, Grep, Read, Write, mcp__playwright-test__*
model: sonnet
color: purple
---

You generate tests specifically for the Saucedemo project.

# MANDATORY STEPS BEFORE GENERATING

1. Read('.claude/docs/page-object-model-pattern.md')
2. Read('pages/LoginPage.ts')
3. Read('pages/InventoryPage.ts')
4. Read('fixtures/pages.ts')
5. Read('utils/constants.ts')
6. Read('tests/e2e/authentication/login.spec.ts')

# PROJECT RULES

1. Import: `import { test } from '../../../fixtures/pages';`
2. Constants: `import { USERS, ERROR_MESSAGES } from '../../../utils/constants';`
3. Multiple tests per file (3-5 related scenarios)
4. Use beforeEach for authentication
5. File naming: `tests/e2e/{feature}/{functionality}.spec.ts`

# TEMPLATE

\`\`\`typescript
import { test } from '../../../fixtures/pages';
import { USERS } from '../../../utils/constants';

test.describe('{Feature}', () => {
  test.beforeEach(async ({ loginPage }) => {
    await loginPage.goto();
    await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
  });

  test('scenario 1', async ({ inventoryPage }) => {
    // Page object methods only
  });

  test('scenario 2', async ({ inventoryPage }) => {
    // Another scenario
  });

  test('scenario 3', async ({ inventoryPage }) => {
    // Yet another
  });
});
\`\`\`
```

### Example 2: Custom Planner with POM Awareness

**File:** `.claude/agents/saucedemo-test-planner.md`

```markdown
---
name: saucedemo-test-planner
description: 'Plans tests for Saucedemo with awareness of existing page objects'
tools: Glob, Grep, Read, LS, mcp__playwright-test__browser_*, mcp__playwright-test__planner_*
model: sonnet
color: green
---

You are a test planner that creates plans compatible with existing page objects.

# Before Planning

1. Read existing page objects: `Glob('pages/*.ts')`
2. List available page objects and methods
3. Plan scenarios using those methods

# Plan Format

\`\`\`markdown
### 1. {Feature Name}
**Seed:** tests/seeds/{appropriate-seed}.spec.ts
**Page Objects:** LoginPage, InventoryPage

#### 1.1 {Scenario Name}
**Methods to use:**
- loginPage.goto()
- loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password)
- inventoryPage.expectToBeVisible()

**Steps:**
1. Navigate to login page (loginPage.goto())
2. Login as standard user (loginPage.login())
3. Verify products page (inventoryPage.expectToBeVisible())
\`\`\`
```

---

## Part 7: Testing Your Custom Agent

### Test the Agent Configuration

1. **Create test prompt file** `.claude/test-prompts/test-generator.md`:

```markdown
Generate a test for adding items to cart with these scenarios:
1. Add single item
2. Add multiple items
3. Remove item

Use the custom test generator with Page Object Model pattern.
```

2. **Invoke agent** via Claude Code:

```
"Read .claude/test-prompts/test-generator.md and generate tests using the custom agent"
```

3. **Verify output**:
   - ✅ Uses fixtures imports
   - ✅ Uses page object methods
   - ✅ Has beforeEach hook
   - ✅ Multiple tests in one file
   - ✅ Uses constants

### Iterate and Refine

If output doesn't match expectations:

1. **Add more examples** to documentation
2. **Be more explicit** in agent rules
3. **Add validation checklist** to agent config
4. **Include anti-patterns** (wrong examples)

---

## Part 8: Maintenance

### Keep Documentation Updated

When you:
- Add new page objects → Update POM documentation
- Change patterns → Update pattern docs
- Add new fixtures → Update fixtures guide
- Change constants → Update constants docs

### Version Documentation

For major pattern changes:

```
.claude/docs/
├── page-object-model-pattern.md      # Current
├── page-object-model-pattern-v1.md   # Old version
└── CHANGELOG.md                       # Document changes
```

### Review Generated Tests

Periodically check if agents follow patterns:

```bash
# Run all tests
npx playwright test

# Review generated test files
# Check they follow patterns
```

---

## Quick Reference

### Create New Agent

```bash
# 1. Create agent config
code .claude/agents/my-custom-agent.md

# 2. Add frontmatter
name: my-custom-agent
description: 'What it does'
tools: Glob, Grep, Read, Write
model: sonnet
color: blue

# 3. Add instructions
# 4. Test with Claude Code
```

### Create New Documentation

```bash
# 1. Create doc file
code .claude/docs/my-pattern.md

# 2. Document pattern with examples
# 3. Reference in agent config:
#    Read('.claude/docs/my-pattern.md')
```

### Use Custom Agent

```
"Use my-custom-agent to generate tests for {feature}"
```

or

```
"Generate tests for {feature} using {pattern}"
```

---

## Summary

1. **Agent configs** go in `.claude/agents/*.md`
2. **Documentation** goes in `.claude/docs/*.md`
3. **Agents read docs** via `Read()` instructions
4. **Reference existing code** (page objects, tests, fixtures)
5. **Include examples** (good and bad)
6. **Add validation** rules
7. **Test and iterate**

Your agents will now follow your project's patterns consistently!
