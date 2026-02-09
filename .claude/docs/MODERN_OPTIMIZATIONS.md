# Modern Playwright Test Optimizations

Comprehensive guide to modern optimizations for Playwright test automation.

---

## Table of Contents

1. [Performance Optimizations](#1-performance-optimizations)
2. [Authentication State Management](#2-authentication-state-management)
3. [Advanced Playwright Config](#3-advanced-playwright-config)
4. [Code Quality Improvements](#4-code-quality-improvements)
5. [AI Agent Optimizations](#5-ai-agent-optimizations)
6. [CI/CD Optimizations](#6-cicd-optimizations)
7. [Modern Testing Patterns](#7-modern-testing-patterns)
8. [Developer Experience](#8-developer-experience)

---

## 1. Performance Optimizations

### 1.1 Authentication State Reuse

**Problem:** Logging in before every test is slow.

**Solution:** Use Playwright's storage state to reuse authenticated sessions.

#### Setup: Global Authentication

Create `tests/setup/auth.setup.ts`:

```typescript
import { test as setup, expect } from '@playwright/test';
import { USERS } from '../../utils/constants';

const authFile = 'playwright/.auth/user.json';

setup('authenticate as standard user', async ({ page }) => {
  await page.goto('https://www.saucedemo.com/');
  await page.locator('[data-test="username"]').fill(USERS.STANDARD.username);
  await page.locator('[data-test="password"]').fill(USERS.STANDARD.password);
  await page.locator('[data-test="login-button"]').click();

  // Wait for successful login
  await page.waitForURL('**/inventory.html');

  // Save authentication state
  await page.context().storageState({ path: authFile });
});
```

#### Update playwright.config.ts:

```typescript
export default defineConfig({
  // ... other config

  // Setup project - runs first
  projects: [
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/,
    },
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        // Use saved authentication state
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'], // Run setup first
    },
  ],
});
```

#### Update Tests:

```typescript
// No more login in beforeEach!
test.describe('Inventory Tests', () => {
  // ✅ Already authenticated from storage state

  test('view products', async ({ page }) => {
    await page.goto('https://www.saucedemo.com/inventory.html');
    // Already logged in!
    await expect(page.locator('.inventory_list')).toBeVisible();
  });
});
```

**Benefits:**
- 🚀 **10-50x faster** - Login once, reuse everywhere
- ✅ **Parallel execution** - All tests start authenticated
- 🔄 **Session persistence** - Maintains cookies, localStorage, etc.

---

### 1.2 API-Based Test Setup

**Problem:** Using UI for test data setup is slow.

**Solution:** Use API requests for data setup, UI for verification.

#### Create API Helper

Create `utils/api-helper.ts`:

```typescript
import { APIRequestContext } from '@playwright/test';

export class SaucedemoAPI {
  constructor(private request: APIRequestContext) {}

  async addItemsToCart(sessionCookie: string, itemIds: number[]): Promise<void> {
    // Use API to add items (if API available)
    await this.request.post('https://api.saucedemo.com/cart', {
      headers: {
        'Cookie': sessionCookie,
      },
      data: { items: itemIds },
    });
  }

  async clearCart(sessionCookie: string): Promise<void> {
    await this.request.delete('https://api.saucedemo.com/cart', {
      headers: {
        'Cookie': sessionCookie,
      },
    });
  }
}
```

#### Use in Tests:

```typescript
test('checkout flow', async ({ page, request }) => {
  const api = new SaucedemoAPI(request);

  // ✅ Fast: Use API to setup cart
  const cookies = await page.context().cookies();
  await api.addItemsToCart(cookies[0].value, [1, 2, 3]);

  // ✅ Test UI: Verify checkout flow
  await page.goto('https://www.saucedemo.com/cart.html');
  await expect(page.locator('.cart_item')).toHaveCount(3);

  // Test the checkout UI flow...
});
```

**Benefits:**
- ⚡ **Much faster** than clicking through UI
- 🎯 **Focus on what matters** - Test UI, setup via API
- 🔄 **Reliable** - Less flaky than UI automation

---

### 1.3 Test Sharding

**Problem:** Test suite takes too long on CI.

**Solution:** Split tests across multiple machines.

#### In playwright.config.ts:

```typescript
export default defineConfig({
  // Allow sharding via CLI
  // Run with: npx playwright test --shard=1/4

  // Or configure directly:
  shard: process.env.CI
    ? {
        current: parseInt(process.env.SHARD_INDEX || '1'),
        total: parseInt(process.env.TOTAL_SHARDS || '4')
      }
    : undefined,
});
```

#### GitHub Actions Example:

```yaml
# .github/workflows/tests.yml
jobs:
  test:
    strategy:
      matrix:
        shard: [1, 2, 3, 4]
    steps:
      - run: npx playwright test --shard=${{ matrix.shard }}/4
```

**Benefits:**
- ⚡ **4x faster** - Run 4 shards = 4x parallelization
- 💰 **Cost effective** - Faster CI = lower costs
- 🔄 **Scalable** - Add more shards as suite grows

---

### 1.4 Selective Test Execution

**Problem:** Running all tests for small changes wastes time.

**Solution:** Run only affected tests.

#### Add Test Tags:

```typescript
test.describe('Cart Tests', () => {
  test('add item @smoke @cart', async ({ inventoryPage }) => {
    // Critical path test
  });

  test('remove item @cart', async ({ cartPage }) => {
    // Non-critical test
  });
});
```

#### Run Selectively:

```bash
# Smoke tests only
npx playwright test --grep @smoke

# Cart tests only
npx playwright test --grep @cart

# Exclude slow tests
npx playwright test --grep-invert @slow
```

#### Smart Selection in CI:

```typescript
// tests/utils/changed-tests.ts
import { execSync } from 'child_process';

export function getChangedTests(): string[] {
  const changed = execSync('git diff --name-only HEAD~1').toString();
  return changed
    .split('\n')
    .filter(file => file.includes('tests/'))
    .filter(file => file.endsWith('.spec.ts'));
}
```

**Benefits:**
- ⚡ **Faster feedback** - Run critical tests first
- 🎯 **Focused testing** - Test what changed
- 💰 **Resource efficient** - Don't waste CI time

---

## 2. Authentication State Management

### 2.1 Multiple User Roles

**Problem:** Need to test with different user types.

**Solution:** Create separate authentication states.

#### Setup Multiple Auth States:

```typescript
// tests/setup/auth.setup.ts
import { test as setup } from '@playwright/test';
import { USERS } from '../../utils/constants';

// Standard user
setup('authenticate standard user', async ({ page }) => {
  await page.goto('https://www.saucedemo.com/');
  await page.locator('[data-test="username"]').fill(USERS.STANDARD.username);
  await page.locator('[data-test="password"]').fill(USERS.STANDARD.password);
  await page.locator('[data-test="login-button"]').click();
  await page.context().storageState({ path: 'playwright/.auth/standard.json' });
});

// Problem user
setup('authenticate problem user', async ({ page }) => {
  await page.goto('https://www.saucedemo.com/');
  await page.locator('[data-test="username"]').fill(USERS.PROBLEM.username);
  await page.locator('[data-test="password"]').fill(USERS.PROBLEM.password);
  await page.locator('[data-test="login-button"]').click();
  await page.context().storageState({ path: 'playwright/.auth/problem.json' });
});

// Performance user
setup('authenticate performance user', async ({ page }) => {
  await page.goto('https://www.saucedemo.com/');
  await page.locator('[data-test="username"]').fill(USERS.PERFORMANCE_GLITCH.username);
  await page.locator('[data-test="password"]').fill(USERS.PERFORMANCE_GLITCH.password);
  await page.locator('[data-test="login-button"]').click();
  await page.context().storageState({ path: 'playwright/.auth/performance.json' });
});
```

#### Configure Projects:

```typescript
export default defineConfig({
  projects: [
    { name: 'setup', testMatch: /.*\.setup\.ts/ },

    {
      name: 'chromium-standard',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'playwright/.auth/standard.json',
      },
      dependencies: ['setup'],
    },

    {
      name: 'chromium-problem',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'playwright/.auth/problem.json',
      },
      dependencies: ['setup'],
    },
  ],
});
```

#### Use in Tests:

```typescript
// tests/e2e/problem-user/visual-issues.spec.ts
test.use({ storageState: 'playwright/.auth/problem.json' });

test.describe('Problem User Visual Issues', () => {
  test('images display incorrectly', async ({ page }) => {
    // Runs as problem user
  });
});
```

---

### 2.2 Dynamic User Fixture

**Problem:** Want flexible user switching in tests.

**Solution:** Create custom fixture for user management.

#### Create User Fixture:

```typescript
// fixtures/users.ts
import { test as base } from '@playwright/test';
import { USERS } from '../utils/constants';

type UserFixtures = {
  loginAs: (userType: keyof typeof USERS) => Promise<void>;
};

export const test = base.extend<UserFixtures>({
  loginAs: async ({ page }, use) => {
    const login = async (userType: keyof typeof USERS) => {
      const user = USERS[userType];
      await page.goto('https://www.saucedemo.com/');
      await page.locator('[data-test="username"]').fill(user.username);
      await page.locator('[data-test="password"]').fill(user.password);
      await page.locator('[data-test="login-button"]').click();
      await page.waitForURL('**/inventory.html');
    };
    await use(login);
  },
});
```

#### Use in Tests:

```typescript
import { test } from '../../fixtures/users';

test('switch between users', async ({ page, loginAs }) => {
  // Login as standard user
  await loginAs('STANDARD');
  // Test something...

  // Logout and switch
  await page.locator('#react-burger-menu-btn').click();
  await page.locator('#logout_sidebar_link').click();

  // Login as different user
  await loginAs('PERFORMANCE_GLITCH');
  // Test different behavior...
});
```

---

## 3. Advanced Playwright Config

### 3.1 Optimized Configuration

Replace your current `playwright.config.ts` with:

```typescript
import { defineConfig, devices } from '@playwright/test';
import dotenv from 'dotenv';
import path from 'path';

// Load environment variables
dotenv.config({ path: path.resolve(__dirname, '.env') });

export default defineConfig({
  testDir: './tests',

  // Timeout configurations
  timeout: 30 * 1000, // 30 seconds per test
  expect: {
    timeout: 5 * 1000, // 5 seconds for assertions
  },

  // Parallel execution
  fullyParallel: true,
  workers: process.env.CI ? 2 : undefined, // 2 workers on CI

  // Retries
  retries: process.env.CI ? 2 : 0,

  // CI-specific settings
  forbidOnly: !!process.env.CI,

  // Reporter configuration
  reporter: [
    ['html', { open: 'never' }],
    ['json', { outputFile: 'test-results/results.json' }],
    ['junit', { outputFile: 'test-results/junit.xml' }],
    ['allure-playwright', {
      detail: true,
      outputFolder: 'allure-results',
      suiteTitle: false,
    }],
    // Slack/Teams notifications (optional)
    // ['./reporters/slack-reporter.ts'],
  ],

  // Global configuration
  use: {
    // Base URL
    baseURL: process.env.BASE_URL || 'https://www.saucedemo.com',

    // Trace configuration
    trace: process.env.CI ? 'retain-on-failure' : 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',

    // Navigation timeout
    navigationTimeout: 15 * 1000,
    actionTimeout: 10 * 1000,

    // Viewport
    viewport: { width: 1280, height: 720 },

    // Ignore HTTPS errors (only for dev/test environments)
    ignoreHTTPSErrors: true,

    // User agent
    userAgent: 'PlaywrightTests/1.0',

    // Locale and timezone
    locale: 'en-US',
    timezoneId: 'America/New_York',

    // Permissions
    permissions: ['geolocation'],

    // Extra HTTP headers
    extraHTTPHeaders: {
      'X-Test-Run': 'true',
    },
  },

  // Projects configuration
  projects: [
    // Setup authentication
    {
      name: 'setup',
      testMatch: /.*\.setup\.ts/,
    },

    // Chromium - Desktop
    {
      name: 'chromium',
      use: {
        ...devices['Desktop Chrome'],
        storageState: 'playwright/.auth/user.json',
        // Enable Chrome DevTools Protocol
        launchOptions: {
          args: ['--disable-web-security'], // Only for testing
        },
      },
      dependencies: ['setup'],
    },

    // Firefox
    {
      name: 'firefox',
      use: {
        ...devices['Desktop Firefox'],
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'],
    },

    // WebKit (Safari)
    {
      name: 'webkit',
      use: {
        ...devices['Desktop Safari'],
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'],
    },

    // Mobile Chrome
    {
      name: 'mobile-chrome',
      use: {
        ...devices['Pixel 5'],
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'],
    },

    // Mobile Safari
    {
      name: 'mobile-safari',
      use: {
        ...devices['iPhone 13'],
        storageState: 'playwright/.auth/user.json',
      },
      dependencies: ['setup'],
    },
  ],

  // Global setup/teardown
  globalSetup: require.resolve('./tests/global-setup.ts'),
  globalTeardown: require.resolve('./tests/global-teardown.ts'),

  // Web server (if testing local app)
  webServer: process.env.START_LOCAL_SERVER ? {
    command: 'npm run start',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
    timeout: 120 * 1000,
  } : undefined,
});
```

---

### 3.2 Environment Variables

Create `.env`:

```bash
# Environment
NODE_ENV=test
BASE_URL=https://www.saucedemo.com

# CI Configuration
CI=false
WORKERS=4

# Authentication
TEST_USERNAME=standard_user
TEST_PASSWORD=secret_sauce

# Feature Flags
ENABLE_API_TESTS=true
ENABLE_VISUAL_TESTS=false
ENABLE_PERFORMANCE_TESTS=false

# Reporting
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/xxx
SEND_NOTIFICATIONS=false

# Database (if needed)
TEST_DB_URL=postgresql://localhost:5432/test_db

# Screenshots/Videos
SCREENSHOT_ON_FAILURE=true
VIDEO_ON_FAILURE=true

# Debug
DEBUG=false
HEADED=false
```

---

## 4. Code Quality Improvements

### 4.1 TypeScript Strict Mode

Update `tsconfig.json`:

```json
{
  "compilerOptions": {
    "target": "ESNext",
    "module": "commonjs",
    "moduleResolution": "node",
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,

    // Strict type checking
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true,
    "strictBindCallApply": true,
    "strictPropertyInitialization": true,
    "noImplicitThis": true,
    "alwaysStrict": true,

    // Additional checks
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,

    // Path mapping
    "baseUrl": ".",
    "paths": {
      "@pages/*": ["pages/*"],
      "@fixtures/*": ["fixtures/*"],
      "@utils/*": ["utils/*"],
      "@tests/*": ["tests/*"]
    }
  },
  "include": ["**/*.ts"],
  "exclude": ["node_modules", "test-results", "playwright-report"]
}
```

#### Use Path Aliases:

```typescript
// Before
import { LoginPage } from '../../../pages/LoginPage';
import { USERS } from '../../../utils/constants';

// After
import { LoginPage } from '@pages/LoginPage';
import { USERS } from '@utils/constants';
```

---

### 4.2 Custom Matchers

Create `utils/custom-matchers.ts`:

```typescript
import { expect } from '@playwright/test';
import type { Page, Locator } from '@playwright/test';

expect.extend({
  async toHaveLoadedSuccessfully(page: Page) {
    const performanceTiming = await page.evaluate(() => {
      const perf = window.performance.timing;
      return {
        loadTime: perf.loadEventEnd - perf.navigationStart,
        domReady: perf.domContentLoadedEventEnd - perf.navigationStart,
      };
    });

    const pass = performanceTiming.loadTime < 3000; // 3 seconds

    return {
      message: () => pass
        ? `Page loaded in ${performanceTiming.loadTime}ms`
        : `Page took ${performanceTiming.loadTime}ms to load (expected < 3000ms)`,
      pass,
    };
  },

  async toBeInCart(locator: Locator, itemName: string) {
    const cartItems = await locator.allTextContents();
    const pass = cartItems.some(item => item.includes(itemName));

    return {
      message: () => pass
        ? `Item "${itemName}" is in cart`
        : `Item "${itemName}" not found in cart. Found: ${cartItems.join(', ')}`,
      pass,
    };
  },
});

// Type definitions
declare global {
  namespace PlaywrightTest {
    interface Matchers<R> {
      toHaveLoadedSuccessfully(): R;
      toBeInCart(itemName: string): R;
    }
  }
}
```

#### Use Custom Matchers:

```typescript
import './utils/custom-matchers';

test('performance check', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveLoadedSuccessfully();
});

test('item in cart', async ({ page }) => {
  await expect(page.locator('.cart_item')).toBeInCart('Sauce Labs Backpack');
});
```

---

### 4.3 Test Data Builders

Create `utils/test-data-builders.ts`:

```typescript
export class UserBuilder {
  private user = {
    firstName: 'John',
    lastName: 'Doe',
    email: 'john@example.com',
    zipCode: '12345',
  };

  withFirstName(name: string): this {
    this.user.firstName = name;
    return this;
  }

  withLastName(name: string): this {
    this.user.lastName = name;
    return this;
  }

  withEmail(email: string): this {
    this.user.email = email;
    return this;
  }

  withZipCode(zip: string): this {
    this.user.zipCode = zip;
    return this;
  }

  build() {
    return { ...this.user };
  }
}

export class CartBuilder {
  private items: string[] = [];

  withItem(itemName: string): this {
    this.items.push(itemName);
    return this;
  }

  withItems(...itemNames: string[]): this {
    this.items.push(...itemNames);
    return this;
  }

  build() {
    return { items: [...this.items] };
  }
}
```

#### Use Builders:

```typescript
import { UserBuilder, CartBuilder } from '@utils/test-data-builders';

test('checkout with custom user', async ({ checkoutPage }) => {
  const user = new UserBuilder()
    .withFirstName('Alice')
    .withLastName('Smith')
    .withZipCode('90210')
    .build();

  await checkoutPage.fillUserInfo(user);
});

test('checkout with multiple items', async ({ inventoryPage, cartPage }) => {
  const cart = new CartBuilder()
    .withItems('Sauce Labs Backpack', 'Sauce Labs Bike Light')
    .build();

  for (const item of cart.items) {
    await inventoryPage.addToCart(item);
  }
});
```

---

## 5. AI Agent Optimizations

### 5.1 Agent Context Caching

Create `.claude/agents/playwright-test-generator-cached.md`:

```markdown
---
name: playwright-test-generator-cached
description: 'Test generator with cached context'
tools: Glob, Grep, Read, Write, mcp__playwright-test__*
model: sonnet
color: purple
---

# Context Cache (Read Once, Reference Many Times)

At the start of EVERY session, load and cache:

\`\`\`typescript
// Cache these at session start
const CACHED_CONTEXT = {
  pomPattern: await Read('.claude/docs/page-object-model-pattern.md'),
  testPatterns: await Read('.claude/docs/test-generation-patterns.md'),
  loginPage: await Read('pages/LoginPage.ts'),
  inventoryPage: await Read('pages/InventoryPage.ts'),
  fixtures: await Read('fixtures/pages.ts'),
  constants: await Read('utils/constants.ts'),
  exampleTest: await Read('tests/e2e/authentication/login.spec.ts'),
};
\`\`\`

Then reference cached context instead of re-reading files.

# Generation Rules

[Rest of your agent configuration...]
```

---

### 5.2 Template Library

Create `.claude/templates/`:

```typescript
// .claude/templates/test-with-beforeeach.ts
import { test } from '../../../fixtures/pages';
import { USERS } from '../../../utils/constants';

test.describe('{{FEATURE_NAME}}', () => {
  test.beforeEach(async ({ loginPage }) => {
    await loginPage.goto();
    await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
  });

  test('{{TEST_SCENARIO_1}}', async ({ {{PAGE_OBJECTS}} }) => {
    // Test implementation
  });

  test('{{TEST_SCENARIO_2}}', async ({ {{PAGE_OBJECTS}} }) => {
    // Test implementation
  });
});
```

```typescript
// .claude/templates/page-object.ts
import { Page, Locator, expect } from '@playwright/test';

export class {{PAGE_NAME}} {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  // Locators
  get {{ELEMENT_NAME}}(): Locator {
    return this.page.locator('{{SELECTOR}}');
  }

  // Actions
  async {{ACTION_NAME}}(): Promise<void> {
    await this.{{ELEMENT_NAME}}.click();
  }

  // Assertions
  async expectToBeVisible(): Promise<void> {
    await expect(this.{{ELEMENT_NAME}}).toBeVisible();
  }
}
```

---

### 5.3 Validation Agent

Create `.claude/agents/test-validator.md`:

```markdown
---
name: test-validator
description: 'Validates generated tests against project standards'
tools: Read, Grep
model: haiku
color: yellow
---

You validate generated test files against project standards.

# Validation Checklist

For each test file, verify:

1. **Imports**
   - [ ] Imports from `fixtures/pages` (not `@playwright/test`)
   - [ ] Imports constants from `utils/constants`

2. **Structure**
   - [ ] Has `test.describe()` block
   - [ ] Has `beforeEach` for shared setup (if applicable)
   - [ ] Has 2-6 related test scenarios

3. **Page Objects**
   - [ ] No raw `page.locator()` calls
   - [ ] Only uses page object methods
   - [ ] Page objects exist in `pages/` directory

4. **Constants**
   - [ ] Uses `USERS.*` not hardcoded usernames
   - [ ] Uses `ERROR_MESSAGES.*` not hardcoded messages
   - [ ] Uses `URLS.*` not hardcoded URLs

5. **File Naming**
   - [ ] Follows: `tests/e2e/{feature}/{functionality}.spec.ts`

# Output Format

\`\`\`
✅ PASSED: All checks passed
❌ FAILED:
  - Issue 1: Description
  - Issue 2: Description

Suggestions:
- Fix suggestion 1
- Fix suggestion 2
\`\`\`
```

---

## 6. CI/CD Optimizations

### 6.1 GitHub Actions Workflow

Create `.github/workflows/playwright.yml`:

```yaml
name: Playwright Tests

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  test:
    timeout-minutes: 60
    runs-on: ubuntu-latest
    strategy:
      fail-fast: false
      matrix:
        shard: [1, 2, 3, 4]

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: 'npm'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright Browsers
        run: npx playwright install --with-deps chromium

      - name: Run Playwright tests
        run: npx playwright test --shard=${{ matrix.shard }}/${{ strategy.job-total }}
        env:
          CI: true

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report-${{ matrix.shard }}
          path: playwright-report/
          retention-days: 30

      - name: Upload Allure results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: allure-results-${{ matrix.shard }}
          path: allure-results/

  merge-reports:
    if: always()
    needs: [test]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 20

      - name: Download all reports
        uses: actions/download-artifact@v4
        with:
          path: all-reports

      - name: Merge Allure reports
        run: |
          npm install -g allure-commandline
          allure generate all-reports/allure-results-*/ -o allure-report --clean

      - name: Deploy to GitHub Pages
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./allure-report
```

---

### 6.2 Docker Support

Create `Dockerfile`:

```dockerfile
FROM mcr.microsoft.com/playwright:v1.40.0-jammy

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .

# Run tests
CMD ["npx", "playwright", "test"]
```

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  playwright-tests:
    build: .
    environment:
      - CI=true
      - BASE_URL=https://www.saucedemo.com
    volumes:
      - ./playwright-report:/app/playwright-report
      - ./test-results:/app/test-results
    command: npx playwright test
```

Run with:

```bash
docker-compose up --abort-on-container-exit
```

---

## 7. Modern Testing Patterns

### 7.1 Component Testing

Test individual components in isolation:

```typescript
// tests/component/login-form.spec.ts
import { test, expect } from '@playwright/experimental-ct-react';
import { LoginForm } from '@/components/LoginForm';

test('login form validation', async ({ mount }) => {
  const component = await mount(<LoginForm />);

  await component.getByRole('button', { name: 'Login' }).click();
  await expect(component.getByText('Username is required')).toBeVisible();
});
```

---

### 7.2 Visual Regression Testing

```typescript
import { test, expect } from '@playwright/test';

test('visual regression', async ({ page }) => {
  await page.goto('/inventory.html');

  // Full page screenshot
  await expect(page).toHaveScreenshot('inventory-page.png');

  // Element screenshot
  await expect(page.locator('.inventory_list')).toHaveScreenshot('product-list.png');

  // With threshold
  await expect(page).toHaveScreenshot('page.png', {
    maxDiffPixels: 100,
  });
});
```

---

### 7.3 Accessibility Testing

```typescript
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

test('accessibility check', async ({ page }) => {
  await page.goto('/');

  const results = await new AxeBuilder({ page }).analyze();

  expect(results.violations).toEqual([]);
});
```

---

## 8. Developer Experience

### 8.1 Custom npm Scripts

Update `package.json`:

```json
{
  "scripts": {
    "test": "playwright test",
    "test:headed": "playwright test --headed",
    "test:debug": "playwright test --debug",
    "test:ui": "playwright test --ui",
    "test:smoke": "playwright test --grep @smoke",
    "test:chromium": "playwright test --project=chromium",
    "test:mobile": "playwright test --project=mobile-chrome",
    "test:update-snapshots": "playwright test --update-snapshots",
    "test:report": "playwright show-report",
    "test:allure": "allure generate allure-results --clean && allure open",
    "test:codegen": "playwright codegen",
    "lint": "eslint . --ext .ts",
    "format": "prettier --write \"**/*.{ts,json,md}\"",
    "type-check": "tsc --noEmit"
  }
}
```

---

### 8.2 VS Code Configuration

Create `.vscode/settings.json`:

```json
{
  "editor.formatOnSave": true,
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "files.associations": {
    "*.spec.ts": "typescript"
  },
  "playwright.reuseBrowser": true,
  "playwright.showTrace": true
}
```

Create `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug Current Test",
      "program": "${workspaceFolder}/node_modules/@playwright/test/cli.js",
      "args": ["test", "${file}", "--debug"],
      "console": "integratedTerminal",
      "internalConsoleOptions": "neverOpen"
    }
  ]
}
```

---

## Summary of Key Optimizations

### Performance
- ✅ **Storage State** - Reuse authentication (10-50x faster)
- ✅ **API Setup** - Use APIs for data, UI for verification
- ✅ **Test Sharding** - Split tests across machines
- ✅ **Selective Execution** - Run only relevant tests

### Code Quality
- ✅ **TypeScript Strict Mode** - Catch errors early
- ✅ **Custom Matchers** - Domain-specific assertions
- ✅ **Test Data Builders** - Flexible test data creation
- ✅ **Path Aliases** - Cleaner imports

### AI Agents
- ✅ **Context Caching** - Load patterns once
- ✅ **Template Library** - Reusable code templates
- ✅ **Validation Agent** - Automated code review

### CI/CD
- ✅ **GitHub Actions** - Automated test execution
- ✅ **Docker Support** - Consistent environments
- ✅ **Parallel Execution** - Faster feedback

### Developer Experience
- ✅ **Custom Scripts** - Quick access to common tasks
- ✅ **VS Code Config** - Integrated debugging
- ✅ **Component Testing** - Test in isolation
- ✅ **Visual Testing** - Catch UI regressions

---

## Next Steps

1. **Implement Storage State** (biggest win)
2. **Add Test Tags** for selective execution
3. **Enable TypeScript Strict Mode**
4. **Update Playwright Config**
5. **Create Global Setup/Teardown**
6. **Add Custom Matchers**
7. **Optimize CI/CD Pipeline**

Each optimization is independent - implement them incrementally!
