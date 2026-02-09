# Test Generation Patterns

Specific patterns for generating tests that follow project conventions.

---

## Pattern 1: Multiple Tests in One File

### ❌ DON'T: One Test Per File

```
tests/e2e/cart/
├── add-one-item.spec.ts          ❌ Separate files
├── add-multiple-items.spec.ts    ❌ For related
├── remove-item.spec.ts           ❌ Scenarios
└── update-quantity.spec.ts       ❌ Bad!
```

### ✅ DO: Group Related Tests

```
tests/e2e/cart/
└── manage-items.spec.ts          ✅ All cart scenarios together
```

**Implementation:**

```typescript
// tests/e2e/cart/manage-items.spec.ts
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
    await inventoryPage.clickAddToCart('Sauce Labs Bolt T-Shirt');
    await inventoryPage.expectCartBadge('3');
  });

  test('Remove item from cart', async ({ inventoryPage, cartPage }) => {
    await inventoryPage.expectToBeVisible();
    await inventoryPage.clickAddToCart('Sauce Labs Backpack');
    await inventoryPage.clickCart();
    await cartPage.expectToBeVisible();
    await cartPage.removeItem('Sauce Labs Backpack');
    await cartPage.expectCartEmpty();
  });

  test('Update item quantity', async ({ inventoryPage, cartPage }) => {
    await inventoryPage.expectToBeVisible();
    await inventoryPage.clickAddToCart('Sauce Labs Backpack');
    await inventoryPage.clickAddToCart('Sauce Labs Backpack'); // Add twice
    await inventoryPage.clickCart();
    await cartPage.expectItemQuantity('Sauce Labs Backpack', 2);
  });
});
```

**Guidelines:**
- **3-6 tests per file** is ideal
- Group by **feature** or **user flow**
- Tests should be **related** but **independent**

---

## Pattern 2: Reuse Authentication with beforeEach

### ❌ DON'T: Repeat Login in Every Test

```typescript
test.describe('Inventory Tests', () => {
  test('view products', async ({ loginPage, inventoryPage }) => {
    // ❌ Login repeated
    await loginPage.goto();
    await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);

    await inventoryPage.expectToBeVisible();
    await inventoryPage.expectProductCount(6);
  });

  test('filter products', async ({ loginPage, inventoryPage }) => {
    // ❌ Login repeated again
    await loginPage.goto();
    await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);

    await inventoryPage.selectSort('Price (low to high)');
    await inventoryPage.expectFirstProduct('Sauce Labs Onesie');
  });

  test('search products', async ({ loginPage, inventoryPage }) => {
    // ❌ Login repeated yet again
    await loginPage.goto();
    await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);

    await inventoryPage.searchProduct('backpack');
    await inventoryPage.expectProductCount(1);
  });
});
```

### ✅ DO: Use beforeEach Hook

```typescript
test.describe('Inventory Tests', () => {
  // ✅ Login once in beforeEach
  test.beforeEach(async ({ loginPage }) => {
    await loginPage.goto();
    await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
  });

  test('view products', async ({ inventoryPage }) => {
    // ✅ Already logged in
    await inventoryPage.expectToBeVisible();
    await inventoryPage.expectProductCount(6);
  });

  test('filter products', async ({ inventoryPage }) => {
    // ✅ Already logged in
    await inventoryPage.selectSort('Price (low to high)');
    await inventoryPage.expectFirstProduct('Sauce Labs Onesie');
  });

  test('search products', async ({ inventoryPage }) => {
    // ✅ Already logged in
    await inventoryPage.searchProduct('backpack');
    await inventoryPage.expectProductCount(1);
  });
});
```

---

## Pattern 3: Reuse Selectors via Page Objects

### ❌ DON'T: Raw Selectors in Tests

```typescript
// ❌ BAD: Duplicated selectors across tests
test('test 1', async ({ page }) => {
  await page.locator('[data-test="username"]').fill('standard_user');
  await page.locator('[data-test="password"]').fill('secret_sauce');
  await page.locator('[data-test="login-button"]').click();
});

test('test 2', async ({ page }) => {
  // ❌ Same selectors repeated
  await page.locator('[data-test="username"]').fill('invalid');
  await page.locator('[data-test="password"]').fill('wrong');
  await page.locator('[data-test="login-button"]').click();
});

test('test 3', async ({ page }) => {
  // ❌ Same selectors again
  await page.locator('[data-test="username"]').fill('locked_out_user');
  await page.locator('[data-test="password"]').fill('secret_sauce');
  await page.locator('[data-test="login-button"]').click();
});
```

### ✅ DO: Use Page Object Methods

```typescript
// Page Object (pages/LoginPage.ts) - Selectors defined ONCE
export class LoginPage {
  get usernameInput(): Locator {
    return this.page.locator('[data-test="username"]');  // ✅ Defined once
  }

  get passwordInput(): Locator {
    return this.page.locator('[data-test="password"]');  // ✅ Defined once
  }

  get loginButton(): Locator {
    return this.page.locator('[data-test="login-button"]');  // ✅ Defined once
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
}

// Tests - Selectors reused via page object
test('test 1', async ({ loginPage }) => {
  await loginPage.fillUsername('standard_user');     // ✅ Reuses selector
  await loginPage.fillPassword('secret_sauce');      // ✅ Reuses selector
  await loginPage.clickLogin();                      // ✅ Reuses selector
});

test('test 2', async ({ loginPage }) => {
  await loginPage.fillUsername('invalid');           // ✅ Reuses selector
  await loginPage.fillPassword('wrong');             // ✅ Reuses selector
  await loginPage.clickLogin();                      // ✅ Reuses selector
});

test('test 3', async ({ loginPage }) => {
  await loginPage.fillUsername('locked_out_user');   // ✅ Reuses selector
  await loginPage.fillPassword('secret_sauce');      // ✅ Reuses selector
  await loginPage.clickLogin();                      // ✅ Reuses selector
});
```

**Benefits:**
- Selectors defined **once** in page object
- If selector changes, update **one place**
- Tests are **readable** (no cryptic selectors)
- Tests are **maintainable**

---

## Pattern 4: Avoid Login in Every Test File

### Scenario: Multiple Test Files Need Authentication

#### ❌ DON'T: Login in Every File

```typescript
// tests/e2e/inventory/view-products.spec.ts
test('view products', async ({ loginPage, inventoryPage }) => {
  await loginPage.goto();  // ❌ Login
  await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
  // test...
});

// tests/e2e/cart/add-items.spec.ts
test('add items', async ({ loginPage, inventoryPage }) => {
  await loginPage.goto();  // ❌ Login repeated
  await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
  // test...
});

// tests/e2e/checkout/complete-order.spec.ts
test('checkout', async ({ loginPage, cartPage }) => {
  await loginPage.goto();  // ❌ Login repeated
  await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
  // test...
});
```

#### ✅ DO: Use Seed Files

Create an authenticated seed:

```typescript
// tests/seeds/authenticated-seed.spec.ts
test.describe('Authenticated User', () => {
  test('seed', async ({ page }) => {
    await page.goto('https://www.saucedemo.com/');
    await page.locator('[data-test="username"]').fill('standard_user');
    await page.locator('[data-test="password"]').fill('secret_sauce');
    await page.locator('[data-test="login-button"]').click();
    await expect(page.locator('.inventory_list')).toBeVisible();
  });
});
```

Then reference it in test plan:

```markdown
### Inventory Tests
**Seed:** tests/seeds/authenticated-seed.spec.ts

#### Test scenarios that all start from logged-in state
```

When Test Generator uses this seed:
- Browser starts logged in
- Tests don't repeat login
- Faster execution
- DRY principle

---

## Pattern 5: Test File Organization

### File Naming Convention

```
tests/e2e/{feature}/{functionality}.spec.ts
```

### Examples

#### ✅ Good Organization

```
tests/e2e/
├── authentication/
│   ├── login.spec.ts              # All login scenarios
│   └── logout.spec.ts             # All logout scenarios
├── inventory/
│   ├── view-products.spec.ts      # Viewing, browsing products
│   ├── filter-sort.spec.ts        # Filtering and sorting
│   └── product-details.spec.ts    # Product detail page
├── cart/
│   ├── manage-items.spec.ts       # Add, remove, update quantity
│   └── navigation.spec.ts         # Continue shopping, checkout
└── checkout/
    ├── information.spec.ts        # Checkout information step
    ├── overview.spec.ts           # Order overview step
    └── complete.spec.ts           # Order completion
```

#### ❌ Bad Organization

```
tests/e2e/
├── test1.spec.ts                  ❌ Not descriptive
├── test2.spec.ts                  ❌ Not descriptive
├── login-valid.spec.ts            ❌ Should be in authentication/login.spec.ts
├── login-invalid.spec.ts          ❌ Should be in authentication/login.spec.ts
├── add-one-item.spec.ts           ❌ Should be in cart/manage-items.spec.ts
├── add-two-items.spec.ts          ❌ Should be in cart/manage-items.spec.ts
└── checkout-1.spec.ts             ❌ Not descriptive
```

---

## Pattern 6: Test Structure Template

### Standard Template

```typescript
// Import from fixtures (required)
import { test } from '../../../fixtures/pages';
import { USERS, ERROR_MESSAGES } from '../../../utils/constants';

// Describe block with descriptive name
test.describe('{Feature} - {Functionality}', () => {

  // Optional: beforeEach for shared setup
  test.beforeEach(async ({ loginPage }) => {
    await loginPage.goto();
    await loginPage.login(USERS.STANDARD.username, USERS.STANDARD.password);
  });

  // Test 1: Happy path scenario
  test('{happy path description}', async ({ pageObject1, pageObject2 }) => {
    // Arrange (if needed beyond beforeEach)

    // Act
    await pageObject1.performAction();

    // Assert
    await pageObject1.expectResult();
  });

  // Test 2: Alternative scenario
  test('{alternative scenario}', async ({ pageObject1, pageObject2 }) => {
    // Different test logic
    await pageObject1.performDifferentAction();
    await pageObject1.expectDifferentResult();
  });

  // Test 3: Edge case
  test('{edge case description}', async ({ pageObject1 }) => {
    // Edge case logic
    await pageObject1.performEdgeCaseAction();
    await pageObject1.expectEdgeCaseResult();
  });

  // Test 4: Error scenario
  test('{error scenario}', async ({ pageObject1 }) => {
    // Error case logic
    await pageObject1.performInvalidAction();
    await pageObject1.expectErrorMessage(ERROR_MESSAGES.SOME_ERROR);
  });
});
```

### Real Example

```typescript
import { test } from '../../../fixtures/pages';
import { USERS, ERROR_MESSAGES } from '../../../utils/constants';

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

  test('Cannot add out of stock item', async ({ inventoryPage }) => {
    await inventoryPage.expectToBeVisible();
    await inventoryPage.expectAddToCartDisabled('Out of Stock Item');
  });
});
```

---

## Pattern 7: Comments in Generated Tests

### When to Add Comments

✅ **DO add comments** for:
- Complex business logic
- Non-obvious steps
- Clarifying why something is done

❌ **DON'T add comments** for:
- Self-explanatory code
- Every single line
- Repeating method names

### Examples

#### ❌ Over-commented

```typescript
test('add item', async ({ inventoryPage }) => {
  // Expect inventory page to be visible
  await inventoryPage.expectToBeVisible();

  // Click add to cart for Sauce Labs Backpack
  await inventoryPage.clickAddToCart('Sauce Labs Backpack');

  // Expect cart badge to show 1
  await inventoryPage.expectCartBadge('1');
});
```

#### ✅ Well-commented

```typescript
test('add item', async ({ inventoryPage }) => {
  await inventoryPage.expectToBeVisible();
  await inventoryPage.clickAddToCart('Sauce Labs Backpack');

  // Verify cart badge updates (important UX indicator)
  await inventoryPage.expectCartBadge('1');
});
```

#### ✅ Complex logic needs comments

```typescript
test('bulk add with quantity limit', async ({ inventoryPage, cartPage }) => {
  await inventoryPage.expectToBeVisible();

  // Saucedemo has a 6-item limit, attempting to add 7 should cap at 6
  for (let i = 0; i < 7; i++) {
    await inventoryPage.clickAddToCart('Sauce Labs Backpack');
  }

  await inventoryPage.clickCart();

  // Should be capped at maximum allowed quantity
  await cartPage.expectItemQuantity('Sauce Labs Backpack', 6);
});
```

---

## Summary for Test Generator Agent

When generating tests:

1. **✅ Multiple tests per file** (3-6 related scenarios)
2. **✅ Use beforeEach** for shared setup (especially login)
3. **✅ Use page objects** (never raw locators)
4. **✅ Import from fixtures** (`import { test } from '../../../fixtures/pages'`)
5. **✅ Use constants** (`USERS.STANDARD.username` not `'standard_user'`)
6. **✅ Descriptive file names** (`tests/e2e/{feature}/{functionality}.spec.ts`)
7. **✅ Minimal comments** (only for complex logic)
8. **✅ Use seed files** for consistent starting state

### Generated Test Checklist

- [ ] Imports from `fixtures/pages`
- [ ] Multiple related tests in one file
- [ ] Has `beforeEach` for shared setup
- [ ] Uses page object methods only
- [ ] Uses constants from `utils/constants`
- [ ] File naming follows convention
- [ ] No raw `page.locator()` in tests
- [ ] Tests are independent
- [ ] Tests are focused and readable
