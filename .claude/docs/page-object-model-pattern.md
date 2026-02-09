# Page Object Model Pattern Reference

This document describes the Page Object Model (POM) pattern used in this project for the Playwright test generator agent.

---

## Architecture Overview

### Project Structure

```
├── fixtures/
│   └── pages.ts           # Custom test fixtures with page objects
├── pages/
│   ├── LoginPage.ts       # Login page object
│   ├── InventoryPage.ts   # Inventory/Products page object
│   └── MenuComponent.ts   # Shared menu component
├── utils/
│   └── constants.ts       # Shared constants (URLS, USERS, ERROR_MESSAGES)
└── tests/
    └── e2e/
        └── authentication/
            ├── login.spec.ts
            └── logout.spec.ts
```

---

## Page Object Pattern

### Page Object Class Structure

Each page object follows this pattern:

```typescript
import { Page, Locator, expect } from '@playwright/test';

export class PageName {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  // 1. LOCATORS (as getters)
  get elementName(): Locator {
    return this.page.locator('[data-test="element"]');
  }

  get elementByRole(): Locator {
    return this.page.getByRole('button', { name: 'Name' });
  }

  // 2. ACTION METHODS
  async performAction(): Promise<void> {
    await this.elementName.click();
  }

  // 3. ASSERTION METHODS (prefixed with "expect")
  async expectToBeVisible(): Promise<void> {
    await expect(this.elementByRole).toBeVisible();
  }
}
```

### Key Principles

1. **Locators as Getters**: Always use getters for locators (not properties)
   - Computed on each access for reliability
   - Avoids stale element references

2. **Three Sections**:
   - Locators (getters)
   - Action methods (async functions that perform interactions)
   - Assertion methods (async functions prefixed with "expect")

3. **Data-Test Attributes**: Prefer `[data-test="..."]` selectors
   - Fallback to role-based: `getByRole('button', { name: 'Login' })`

4. **Method Names**: Use descriptive, action-oriented names
   - Actions: `fillUsername()`, `clickLogin()`, `addToCart()`
   - Assertions: `expectToBeVisible()`, `expectErrorMessage()`

---

## Custom Test Fixtures

### Fixture Configuration

Located in `fixtures/pages.ts`:

```typescript
import { test as base } from '@playwright/test';
import { LoginPage } from '../pages/LoginPage';
import { InventoryPage } from '../pages/InventoryPage';
import { MenuComponent } from '../pages/MenuComponent';

type PageFixtures = {
  loginPage: LoginPage;
  inventoryPage: InventoryPage;
  menuComponent: MenuComponent;
};

export const test = base.extend<PageFixtures>({
  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },
  inventoryPage: async ({ page }, use) => {
    await use(new InventoryPage(page));
  },
  menuComponent: async ({ page }, use) => {
    await use(new MenuComponent(page));
  },
});

export { expect } from '@playwright/test';
```

### Usage in Tests

```typescript
// ✅ CORRECT: Import from fixtures
import { test } from '../../../fixtures/pages';
import { USERS } from '../../../utils/constants';

test.describe('Feature Tests', () => {
  test('test name', async ({ loginPage, inventoryPage }) => {
    // Page objects are automatically available
    await loginPage.goto();
    await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
    await inventoryPage.expectToBeVisible();
  });
});
```

```typescript
// ❌ WRONG: Don't import from @playwright/test directly
import { test } from '@playwright/test';  // ❌ No page objects available
```

---

## Example Page Object: LoginPage

### Full Implementation

```typescript
import { Page, Locator, expect } from '@playwright/test';
import { URLS } from '../utils/constants';

export class LoginPage {
  readonly page: Page;

  constructor(page: Page) {
    this.page = page;
  }

  // LOCATORS
  get usernameInput(): Locator {
    return this.page.locator('[data-test="username"]');
  }

  get passwordInput(): Locator {
    return this.page.locator('[data-test="password"]');
  }

  get loginButton(): Locator {
    return this.page.locator('[data-test="login-button"]');
  }

  get errorMessage(): Locator {
    return this.page.locator('[data-test="error"]');
  }

  // ACTIONS
  async goto(): Promise<void> {
    await this.page.goto(URLS.LOGIN);
  }

  async login(username: string, password: string): Promise<void> {
    await this.usernameInput.fill(username);
    await this.passwordInput.fill(password);
    await this.loginButton.click();
  }

  async fillUsername(username: string): Promise<void> {
    await this.usernameInput.fill(username);
  }

  async fillPassword(password: string): Promise<void> {
    await this.passwordInput.fill(password);
  }

  async clickLogin(): Promise<void> {
    await this.loginButton.click();
  }

  // ASSERTIONS
  async expectToBeVisible(): Promise<void> {
    await expect(this.page.getByRole('textbox', { name: 'Username' })).toBeVisible();
    await expect(this.page.getByRole('textbox', { name: 'Password' })).toBeVisible();
    await expect(this.page.getByRole('button', { name: 'Login' })).toBeVisible();
  }

  async expectErrorMessage(message: string): Promise<void> {
    await expect(this.page.getByText(message)).toBeVisible();
  }

  async expectUsernameValue(value: string): Promise<void> {
    await expect(this.usernameInput).toHaveValue(value);
  }
}
```

---

## Constants Pattern

### Constants File

Located in `utils/constants.ts`:

```typescript
export const URLS = {
  LOGIN: 'https://www.saucedemo.com/',
  INVENTORY: 'https://www.saucedemo.com/inventory.html',
  CART: 'https://www.saucedemo.com/cart.html',
};

export const USERS = {
  STANDARD: {
    username: 'standard_user',
    password: 'secret_sauce',
  },
  LOCKED_OUT: {
    username: 'locked_out_user',
    password: 'secret_sauce',
  },
};

export const ERROR_MESSAGES = {
  INVALID_CREDENTIALS: 'Epic sadface: Username and password do not match any user in this service',
  LOCKED_OUT: 'Epic sadface: Sorry, this user has been locked out.',
};
```

### Usage

```typescript
import { USERS, ERROR_MESSAGES } from '../../../utils/constants';

test('login test', async ({ loginPage }) => {
  await loginPage.fillUsername(USERS.STANDARD.username);
  await loginPage.expectErrorMessage(ERROR_MESSAGES.INVALID_CREDENTIALS);
});
```

---

## Test File Pattern

### Standard Test Structure

```typescript
// 1. IMPORTS
import { test } from '../../../fixtures/pages';
import { USERS, ERROR_MESSAGES } from '../../../utils/constants';

// 2. DESCRIBE BLOCK
test.describe('Feature Name', () => {

  // 3. TESTS (multiple tests per file)
  test('scenario 1', async ({ loginPage, inventoryPage }) => {
    // Arrange
    await loginPage.goto();

    // Act
    await loginPage.fillUsername(USERS.STANDARD.username);
    await loginPage.fillPassword(USERS.STANDARD.password);
    await loginPage.clickLogin();

    // Assert
    await inventoryPage.expectToBeVisible();
  });

  test('scenario 2', async ({ loginPage }) => {
    // Another test in the same file
    await loginPage.goto();
    await loginPage.fillUsername('invalid');
    await loginPage.expectErrorMessage(ERROR_MESSAGES.INVALID_CREDENTIALS);
  });
});
```

### With Shared Setup (beforeEach)

```typescript
import { test } from '../../../fixtures/pages';
import { USERS } from '../../../utils/constants';

test.describe('Authenticated User Tests', () => {

  // Shared setup - runs before each test
  test.beforeEach(async ({ loginPage }) => {
    await loginPage.goto();
    await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
  });

  test('test 1', async ({ inventoryPage }) => {
    // Already logged in
    await inventoryPage.expectToBeVisible();
    // Test logic...
  });

  test('test 2', async ({ inventoryPage }) => {
    // Also already logged in
    await inventoryPage.expectToBeVisible();
    // Different test logic...
  });
});
```

---

## DO's and DON'Ts

### ✅ DO

```typescript
// ✅ Use page objects
await loginPage.fillUsername(USERS.STANDARD.username);

// ✅ Use constants for test data
await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);

// ✅ Multiple tests per file (related scenarios)
test.describe('Login Tests', () => {
  test('valid login', async ({ loginPage }) => { /* ... */ });
  test('invalid login', async ({ loginPage }) => { /* ... */ });
  test('locked out user', async ({ loginPage }) => { /* ... */ });
});

// ✅ Use beforeEach for shared setup
test.beforeEach(async ({ loginPage }) => {
  await loginPage.goto();
  await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
});

// ✅ Use page object assertion methods
await loginPage.expectToBeVisible();
await inventoryPage.expectProductCount(6);
```

### ❌ DON'T

```typescript
// ❌ Don't use raw locators in tests
await page.locator('[data-test="username"]').fill('user');

// ❌ Don't hardcode test data
await loginPage.fillUsername('standard_user');
await loginPage.fillPassword('secret_sauce');

// ❌ Don't create one test per file (unless truly independent)
// login-valid.spec.ts
test('valid login', async ({ loginPage }) => { /* ... */ });

// login-invalid.spec.ts  // ❌ Should be in same file as above
test('invalid login', async ({ loginPage }) => { /* ... */ });

// ❌ Don't repeat login in every test
test('test 1', async ({ loginPage, inventoryPage }) => {
  await loginPage.goto();
  await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
  // ...
});

test('test 2', async ({ loginPage, inventoryPage }) => {
  await loginPage.goto();  // ❌ Repeated
  await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);  // ❌ Repeated
  // ...
});

// ❌ Don't import from @playwright/test
import { test } from '@playwright/test';  // ❌ Missing page objects
```

---

## File Naming Conventions

### Test Files

```
tests/e2e/{feature}/{functionality}.spec.ts
```

Examples:
- `tests/e2e/authentication/login.spec.ts`
- `tests/e2e/authentication/logout.spec.ts`
- `tests/e2e/cart/add-items.spec.ts`
- `tests/e2e/checkout/payment.spec.ts`

### Page Objects

```
pages/{PageName}.ts
```

Examples:
- `pages/LoginPage.ts`
- `pages/InventoryPage.ts`
- `pages/CartPage.ts`
- `pages/CheckoutPage.ts`

---

## Common Patterns

### Pattern 1: Login Helper Method

```typescript
// In LoginPage.ts
async login(username: string, password: string): Promise<void> {
  await this.usernameInput.fill(username);
  await this.passwordInput.fill(password);
  await this.loginButton.click();
}

// Usage in test
await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
```

### Pattern 2: Chained Actions

```typescript
// In test
await loginPage.goto();
await loginPage.fillUsername(USERS.STANDARD.username);
await loginPage.fillPassword(USERS.STANDARD.password);
await loginPage.clickLogin();
await inventoryPage.expectToBeVisible();
```

### Pattern 3: Verification Methods

```typescript
// In Page Object
async expectProductCount(count: number): Promise<void> {
  await expect(this.page.locator('.inventory_item')).toHaveCount(count);
}

async expectURL(): Promise<void> {
  await expect(this.page).toHaveURL(/.*inventory.html/);
}
```

### Pattern 4: Component Reuse

```typescript
// Shared component
export class MenuComponent {
  constructor(private page: Page) {}

  async logout(): Promise<void> {
    await this.page.locator('#react-burger-menu-btn').click();
    await this.page.locator('#logout_sidebar_link').click();
  }
}

// Usage
test('logout', async ({ menuComponent }) => {
  // ...after login
  await menuComponent.logout();
});
```

---

## For Test Generator Agent

When generating tests, **always follow these rules**:

1. **Import from fixtures**: `import { test } from '../../../fixtures/pages';`
2. **Use page objects**: Access via fixtures, e.g., `{ loginPage, inventoryPage }`
3. **Use constants**: Import from `../../../utils/constants`
4. **Multiple tests per file**: Group related scenarios in one spec file
5. **Use beforeEach**: For shared setup (like login)
6. **Call page object methods**: Never use raw `page.locator()` in tests
7. **Follow naming**: `tests/e2e/{feature}/{functionality}.spec.ts`

### Generated Test Template

```typescript
import { test } from '../../../fixtures/pages';
import { USERS } from '../../../utils/constants';

test.describe('{Feature Name}', () => {
  test.beforeEach(async ({ loginPage }) => {
    await loginPage.goto();
    await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
  });

  test('{scenario 1}', async ({ inventoryPage }) => {
    // Test steps using page objects
    await inventoryPage.expectToBeVisible();
    // ...
  });

  test('{scenario 2}', async ({ inventoryPage }) => {
    // Another related test
    await inventoryPage.expectProductCount(6);
    // ...
  });
});
```

---

## Summary

- **Page Objects**: Encapsulate locators and actions
- **Fixtures**: Provide page objects to tests automatically
- **Constants**: Centralize test data
- **Multiple Tests**: Group related tests in same file
- **beforeEach**: Share setup logic
- **No Raw Locators**: Always use page object methods

This pattern makes tests:
- More readable
- Easier to maintain
- Less repetitive
- More reliable
