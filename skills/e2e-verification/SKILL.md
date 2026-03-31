---
name: e2e-verification
description: "Use as the final verification gate after all implementation tasks are complete — runs full Playwright e2e suite against Docker containers covering Figma design compliance, business workflow validation, and structural DOM matching"
---

# E2E Verification

Final quality gate. Runs the complete Playwright e2e suite against Docker containers after all tasks are done, before finishing the development branch.

## When to Use

After all implementation tasks pass their per-task verification. This is the last gate before `finishing-a-development-branch`.

## Prerequisites

- All Docker containers healthy
- Per-task verification passed for all tasks
- `design-tokens.json` and `component-map.json` exist (if Figma was used)
- Playwright installed and configured

## Three Playwright Projects

### 1. Figma-Compliance (`testMatch: /.*figma\.spec\.ts/`)

Verifies the rendered UI matches Figma design data.

**A. Design Token Assertions**

Compare `getComputedStyle()` values against `design-tokens.json`:

```typescript
import { test, expect } from '@playwright/test';
import tokens from '../fixtures/design-tokens.json';

function hexToRgb(hex: string): string {
  const r = parseInt(hex.slice(1, 3), 16);
  const g = parseInt(hex.slice(3, 5), 16);
  const b = parseInt(hex.slice(5, 7), 16);
  return `rgb(${r}, ${g}, ${b})`;
}

async function getStyles(
  locator: import('@playwright/test').Locator,
  properties: string[]
): Promise<Record<string, string>> {
  return locator.evaluate((el, props) => {
    const cs = getComputedStyle(el);
    return Object.fromEntries(props.map(p => [p, cs.getPropertyValue(p)]));
  }, properties);
}

test('nav uses correct design tokens', async ({ page }) => {
  await page.goto('/');
  const nav = page.locator('nav');
  const styles = await getStyles(nav, ['background-color', 'height']);
  expect(styles['background-color']).toBe(hexToRgb(tokens.colors['nav-bg']));
});

test('heading typography matches Figma', async ({ page }) => {
  await page.goto('/');
  const h1 = page.locator('h1').first();
  const styles = await getStyles(h1, ['font-family', 'font-size', 'font-weight', 'line-height']);
  expect(styles['font-size']).toBe(tokens.typography.heading1.fontSize);
  expect(styles['font-weight']).toBe(tokens.typography.heading1.fontWeight);
});
```

**B. Visual Regression**

Compare screenshots against baselines:

```typescript
test('homepage visual regression', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveScreenshot('homepage.png', {
    fullPage: true,
    maxDiffPixelRatio: 0.01,
    threshold: 0.2,
    animations: 'disabled',
    mask: [
      page.locator('.timestamp'),
      page.locator('.dynamic-content'),
    ],
  });
});

test('article card visual regression', async ({ page }) => {
  await page.goto('/');
  const card = page.locator('.article-card').first();
  await expect(card).toHaveScreenshot('article-card.png', {
    animations: 'disabled',
  });
});
```

**C. Structural Matching**

Verify DOM structure matches component-map.json:

```typescript
import componentMap from '../fixtures/component-map.json';

test('header has correct structure', async ({ page }) => {
  await page.goto('/');
  const header = page.locator(componentMap.Header.selector);
  await expect(header).toBeVisible();

  for (const child of componentMap.Header.children) {
    await expect(header.locator(`[data-component="${child}"], .${child.toLowerCase()}`))
      .toHaveCount(1, { timeout: 5000 });
  }
});

test('page has correct semantic landmarks', async ({ page }) => {
  await page.goto('/');
  await expect(page.locator('header')).toHaveCount(1);
  await expect(page.locator('nav')).toHaveCount(1);
  await expect(page.locator('main')).toHaveCount(1);
  await expect(page.locator('footer')).toHaveCount(1);
});
```

### 2. Business-Workflows (`testMatch: /.*workflow\.spec\.ts/`)

Verifies user journeys work end-to-end.

```typescript
test('user can browse and read articles', async ({ page }) => {
  await page.goto('/');
  await expect(page.locator('.article-card')).toHaveCount.greaterThan(0);

  await page.locator('.article-card').first().click();

  await expect(page.locator('article')).toBeVisible();
  await expect(page.locator('article h1')).not.toBeEmpty();
});

test('404 page renders for invalid routes', async ({ page }) => {
  const response = await page.goto('/nonexistent-page');
  expect(response?.status()).toBe(404);
  await expect(page.locator('text=not found')).toBeVisible({ timeout: 5000 });
});

test('responsive layout at mobile breakpoint', async ({ page }) => {
  await page.setViewportSize({ width: 375, height: 812 });
  await page.goto('/');
  await expect(page.locator('nav')).toBeVisible();
  const cards = page.locator('.article-card');
  if (await cards.count() >= 2) {
    const box1 = await cards.nth(0).boundingBox();
    const box2 = await cards.nth(1).boundingBox();
    expect(box2!.y).toBeGreaterThan(box1!.y);
  }
});
```

### 3. Smoke (`testMatch: /.*smoke\.spec\.ts/`)

Quick critical-path check. Used per-task for fast feedback.

```typescript
test('app loads successfully', async ({ page }) => {
  const response = await page.goto('/');
  expect(response?.status()).toBe(200);
});

test('API health check responds', async ({ request }) => {
  const response = await request.get('/api/health');
  expect(response.status()).toBe(200);
});

test('primary content renders', async ({ page }) => {
  await page.goto('/');
  await expect(page.locator('main')).toBeVisible();
  await expect(page.locator('h1, h2').first()).toBeVisible();
});
```

## Playwright Configuration

```typescript
// playwright.config.ts
import { defineConfig } from '@playwright/test';

export default defineConfig({
  testDir: './e2e/tests',
  use: {
    baseURL: process.env.BASE_URL || 'http://localhost:5173',
    screenshot: 'only-on-failure',
    trace: 'on-first-retry',
  },
  expect: {
    toHaveScreenshot: {
      maxDiffPixelRatio: 0.01,
      threshold: 0.2,
      animations: 'disabled',
    },
  },
  projects: [
    {
      name: 'Figma-Compliance',
      testMatch: /.*figma\.spec\.ts/,
    },
    {
      name: 'Business-Workflows',
      testMatch: /.*workflow\.spec\.ts/,
    },
    {
      name: 'Smoke',
      testMatch: /.*smoke\.spec\.ts/,
      retries: 0,
    },
  ],
});
```

## Output

```json
{
  "e2e_report": {
    "figma_compliance": {
      "token_tests": { "passed": 12, "failed": 0 },
      "visual_regression": { "passed": 5, "failed": 1, "diffs": ["test-results/hero-diff.png"] },
      "structural": { "passed": 8, "failed": 0 }
    },
    "business_workflows": {
      "journeys": { "passed": 6, "failed": 0 }
    },
    "smoke": {
      "tests": { "passed": 3, "failed": 0 }
    },
    "overall": "FAIL",
    "failure_summary": "1 visual regression failure in hero section"
  }
}
```

## Integration

- Runs AFTER all per-task verification passes
- Runs BEFORE `finishing-a-development-branch`
- If failures: loop back to `docker-verified-execution` for the failing area
- Reports feed into the commit/PR description
