# Custom Playwright Agents Configuration

This directory contains custom agent configurations and documentation for Playwright test generation that follows your project's Page Object Model pattern.

---

## Quick Start

### Use Custom Test Generator

Talk to Claude Code with your requirements:

```
"Generate tests for cart management using the custom test generator with Page Object Model"
```

or

```
"Use playwright-test-generator-custom to create tests for checkout flow"
```

The agent will automatically:
- ✅ Use your Page Object Model pattern
- ✅ Import from `fixtures/pages`
- ✅ Generate multiple tests per file
- ✅ Use `beforeEach` for authentication
- ✅ Reuse selectors via page objects
- ✅ Use constants from `utils/constants`

---

## What's Available

### Custom Agents

Located in `.claude/agents/`:

| Agent | Purpose | When to Use |
|-------|---------|-------------|
| **playwright-test-generator-custom.md** | Generates tests using POM pattern | When generating new tests |
| playwright-test-generator.md | Default generator (original) | Fallback/comparison |
| playwright-test-planner.md | Creates test plans | Planning test scenarios |
| playwright-test-healer.md | Fixes failing tests | Debugging test failures |

### Documentation

Located in `.claude/docs/`:

| Document | Purpose |
|----------|---------|
| **page-object-model-pattern.md** | Your POM pattern reference |
| **test-generation-patterns.md** | Test structure patterns |
| **CUSTOM_AGENTS_GUIDE.md** | How to create custom agents |

---

## Key Differences: Default vs Custom Generator

### Default Generator

```typescript
// Generates raw Playwright code
import { test, expect } from '@playwright/test';

test('login', async ({ page }) => {
  await page.goto('https://www.saucedemo.com/');
  await page.locator('[data-test="username"]').fill('standard_user');
  await page.locator('[data-test="password"]').fill('secret_sauce');
  await page.locator('[data-test="login-button"]').click();
});
```

### Custom Generator (POM)

```typescript
// Uses your Page Object Model
import { test } from '../../../fixtures/pages';
import { USERS } from '../../../utils/constants';

test.describe('Authentication - Login', () => {
  test('valid login', async ({ loginPage, inventoryPage }) => {
    await loginPage.goto();
    await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
    await inventoryPage.expectToBeVisible();
  });

  test('invalid login', async ({ loginPage }) => {
    await loginPage.goto();
    await loginPage.fillUsername('invalid');
    await loginPage.fillPassword('wrong');
    await loginPage.clickLogin();
    await loginPage.expectErrorMessage(ERROR_MESSAGES.INVALID_CREDENTIALS);
  });
});
```

---

## Features of Custom Generator

### 1. Uses Your Page Objects

The custom generator reads your existing page objects and uses their methods instead of generating raw locators.

**Your Page Objects:**
- `pages/LoginPage.ts`
- `pages/InventoryPage.ts`
- `pages/MenuComponent.ts`

**Generated tests use them:**
```typescript
await loginPage.fillUsername(USERS.STANDARD.username);
await inventoryPage.expectToBeVisible();
await menuComponent.logout();
```

### 2. Multiple Tests Per File

Groups related test scenarios in one file:

```typescript
test.describe('Cart - Manage Items', () => {
  test('add single item', async ({ inventoryPage }) => { /* ... */ });
  test('add multiple items', async ({ inventoryPage }) => { /* ... */ });
  test('remove item', async ({ inventoryPage, cartPage }) => { /* ... */ });
});
```

### 3. Reuses Authentication

Uses `beforeEach` to avoid repeating login:

```typescript
test.describe('Authenticated Tests', () => {
  test.beforeEach(async ({ loginPage }) => {
    await loginPage.goto();
    await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
  });

  test('test 1', async ({ inventoryPage }) => {
    // Already logged in
  });

  test('test 2', async ({ inventoryPage }) => {
    // Also logged in
  });
});
```

### 4. Uses Your Constants

Imports and uses constants from `utils/constants.ts`:

```typescript
import { USERS, ERROR_MESSAGES } from '../../../utils/constants';

await loginPage.fillUsername(USERS.STANDARD.username);
await loginPage.expectErrorMessage(ERROR_MESSAGES.INVALID_CREDENTIALS);
```

### 5. Selector Reuse

Selectors defined once in page objects, reused everywhere:

```typescript
// In LoginPage.ts (defined once)
get usernameInput(): Locator {
  return this.page.locator('[data-test="username"]');
}

// In tests (reused)
await loginPage.fillUsername('user');  // Uses selector from page object
```

---

## Making Custom Generator Default

To make the custom generator the default:

```bash
cd .claude/agents

# Backup original
mv playwright-test-generator.md playwright-test-generator-original.md

# Make custom the default
mv playwright-test-generator-custom.md playwright-test-generator.md
```

Now when you ask for test generation, it will use your POM pattern automatically.

---

## How It Works

### 1. Agent Configuration

The custom agent configuration in `.claude/agents/playwright-test-generator-custom.md` includes:

```markdown
# Critical Requirements

**BEFORE GENERATING TESTS**, you MUST:

1. Read('.claude/docs/page-object-model-pattern.md')
2. Read('pages/LoginPage.ts')
3. Read('fixtures/pages.ts')
4. Read('utils/constants.ts')

# Rules

1. ALWAYS import from fixtures
2. NEVER use raw locators
3. ALWAYS use page object methods
4. GROUP related tests in same file
5. USE beforeEach for shared setup
```

### 2. Documentation Reference

The agent reads your POM documentation which includes:

- ✅ Your page object structure
- ✅ Import patterns
- ✅ Fixture usage
- ✅ Test file structure
- ✅ Examples (good and bad)

### 3. Pattern Enforcement

The agent validates generated code against rules:

- [ ] Imports from `fixtures/pages`
- [ ] No raw `page.locator()` calls
- [ ] Uses page object methods
- [ ] Has `beforeEach` for setup
- [ ] Multiple tests per file
- [ ] Uses constants

---

## Example Usage

### Scenario: Generate Cart Tests

**You ask:**
```
"Generate tests for cart management - adding items, removing items, and updating quantities"
```

**Custom agent does:**

1. Reads your POM documentation
2. Reads existing page objects (`InventoryPage`, `CartPage`)
3. Reads your fixtures configuration
4. Reads your constants file

5. Generates:

```typescript
import { test } from '../../../fixtures/pages';
import { USERS } from '../../../utils/constants';

test.describe('Cart - Manage Items', () => {
  test.beforeEach(async ({ loginPage }) => {
    await loginPage.goto();
    await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
  });

  test('Add single item to cart', async ({ inventoryPage }) => {
    await inventoryPage.expectToBeVisible();
    await inventoryPage.clickAddToCart('Sauce Labs Backpack');
    await inventoryPage.expectCartBadge('1');
  });

  test('Add multiple items to cart', async ({ inventoryPage }) => {
    await inventoryPage.expectToBeVisible();
    await inventoryPage.clickAddToCart('Sauce Labs Backpack');
    await inventoryPage.clickAddToCart('Sauce Labs Bike Light');
    await inventoryPage.expectCartBadge('2');
  });

  test('Remove item from cart', async ({ inventoryPage, cartPage }) => {
    await inventoryPage.expectToBeVisible();
    await inventoryPage.clickAddToCart('Sauce Labs Backpack');
    await inventoryPage.clickCart();
    await cartPage.removeItem('Sauce Labs Backpack');
    await cartPage.expectCartEmpty();
  });
});
```

---

## Customization

### Add More Patterns

Create new documentation in `.claude/docs/`:

```markdown
# .claude/docs/my-custom-pattern.md

Document your pattern with examples
```

Reference in agent config:

```markdown
**BEFORE GENERATING**, read:
- Read('.claude/docs/my-custom-pattern.md')
```

### Create Specialized Agents

Copy and customize for specific needs:

```bash
cp .claude/agents/playwright-test-generator-custom.md \
   .claude/agents/playwright-api-test-generator.md

# Edit to add API-specific rules
```

### Add Validation Rules

Update agent config with more checks:

```markdown
# After Generating

Validate:
- [ ] Uses TypeScript types from types/
- [ ] Has JSDoc comments
- [ ] Follows ESLint rules
- [ ] Has proper error handling
```

---

## Troubleshooting

### Agent Not Using POM Pattern

**Issue:** Generated tests still have raw locators

**Fix:**
1. Explicitly mention: `"Use the custom test generator with Page Object Model"`
2. Make custom agent the default (rename files)
3. Check agent can read documentation files

### Tests Don't Use beforeEach

**Issue:** Login repeated in every test

**Fix:**
1. Check `.claude/docs/test-generation-patterns.md` has beforeEach examples
2. Mention: `"Generate with beforeEach for authentication"`
3. Provide example of desired pattern

### One Test Per File Generated

**Issue:** Not grouping related tests

**Fix:**
1. Explicitly say: `"Generate multiple related tests in one file"`
2. Check test-generation-patterns.md has examples
3. Specify: `"Create one spec file with 4 test scenarios"`

---

## Documentation Files

### For Agents

Files that agents read:

- `.claude/docs/page-object-model-pattern.md` - Your POM structure
- `.claude/docs/test-generation-patterns.md` - Test patterns
- `.claude/docs/CUSTOM_AGENTS_GUIDE.md` - Agent creation guide

### For You

Guides for customization:

- `.claude/docs/CUSTOM_AGENTS_GUIDE.md` - How to create custom agents
- `SEED_CREATION_GUIDE.md` - How to create seed files
- `SEED_TESTING_WORKFLOW.md` - Seed development workflow
- `PLAYWRIGHT_AGENTS_ARCHITECTURE.md` - System architecture

---

## File Structure

```
.claude/
├── agents/                                      # Agent configurations
│   ├── playwright-test-generator-custom.md     # ⭐ Custom POM generator
│   ├── playwright-test-generator.md            # Default generator
│   ├── playwright-test-planner.md             # Test planner
│   └── playwright-test-healer.md              # Test healer
│
├── docs/                                       # Reference documentation
│   ├── page-object-model-pattern.md           # ⭐ Your POM pattern
│   ├── test-generation-patterns.md            # ⭐ Test patterns
│   └── CUSTOM_AGENTS_GUIDE.md                 # Guide to customization
│
└── README.md                                   # This file
```

---

## Next Steps

1. **Try the custom generator**:
   ```
   "Use playwright-test-generator-custom to create tests for {feature}"
   ```

2. **Review generated tests**:
   - Check they use page objects ✅
   - Check they have beforeEach ✅
   - Check multiple tests per file ✅

3. **Customize if needed**:
   - Add more patterns to documentation
   - Create specialized agents
   - Add validation rules

4. **Make it default** (optional):
   ```bash
   cd .claude/agents
   mv playwright-test-generator.md playwright-test-generator-original.md
   mv playwright-test-generator-custom.md playwright-test-generator.md
   ```

---

## Support

- **Agent not working?** Check `.claude/docs/CUSTOM_AGENTS_GUIDE.md`
- **Need custom patterns?** Add to `.claude/docs/`
- **Want to modify generator?** Edit `.claude/agents/playwright-test-generator-custom.md`

Your Playwright agents now understand and follow your project's patterns! 🎉
