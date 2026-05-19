---
description: 'Playwright E2E testing: .NET, Python and TypeScript'
applyTo: '**'
---


# Playwright .NET Test Generation Instructions

## Test Writing Guidelines

### Code Quality Standards

- **Locators**: Prioritize user-facing, role-based locators (`GetByRole`, `GetByLabel`, `GetByText`, etc.) for resilience and accessibility. Use `await Test.StepAsync()` to group interactions and improve test readability and reporting.
- **Assertions**: Use auto-retrying web-first assertions. These assertions use `Expect()` from Playwright assertions (e.g., `await Expect(locator).ToHaveTextAsync()`). Avoid checking visibility unless specifically testing for visibility changes.
- **Timeouts**: Rely on Playwright's built-in auto-waiting mechanisms. Avoid hard-coded waits or increased default timeouts.
- **Clarity**: Use descriptive test and step titles that clearly state the intent. Add comments only to explain complex logic or non-obvious interactions.

### Test Structure

- **Usings**: Start with `using Microsoft.Playwright;` and either `using Microsoft.Playwright.Xunit;` or `using Microsoft.Playwright.NUnit;` or `using Microsoft.Playwright.MSTest;` for MSTest.
- **Organization**: Create test classes that inherit from `PageTest` (available in NUnit, xUnit, and MSTest packages) or use `IClassFixture<PlaywrightFixture>` for xUnit with custom fixtures. Group related tests for a feature in the same test class.
- **Setup**: Use `[SetUp]` (NUnit), `[TestInitialize]` (MSTest), or constructor initialization (xUnit) for setup actions common to all tests (e.g., navigating to a page).
- **Titles**: Use the appropriate test attribute (`[Test]` for NUnit, `[Fact]` for xUnit, `[TestMethod]` for MSTest) with descriptive method names following C# naming conventions (e.g., `SearchForMovieByTitle`).

### File Organization

- **Location**: Store all test files in a `Tests/` directory or organize by feature.
- **Naming**: Use the convention `<FeatureOrPage>Tests.cs` (e.g., `LoginTests.cs`, `SearchTests.cs`).
- **Scope**: Aim for one test class per major application feature or page.

### Assertion Best Practices

- **UI Structure**: Use `ToMatchAriaSnapshotAsync` to verify the accessibility tree structure of a component. This provides a comprehensive and accessible snapshot.
- **Element Counts**: Use `ToHaveCountAsync` to assert the number of elements found by a locator.
- **Text Content**: Use `ToHaveTextAsync` for exact text matches and `ToContainTextAsync` for partial matches.
- **Navigation**: Use `ToHaveURLAsync` to verify the page URL after an action.

## Example Test Structure

```csharp
using Microsoft.Playwright;
using Microsoft.Playwright.Xunit;
using static Microsoft.Playwright.Assertions;

namespace PlaywrightTests;

public class MovieSearchTests : PageTest
{
    public override async Task InitializeAsync()
    {
        await base.InitializeAsync();
        // Navigate to the application before each test
        await Page.GotoAsync("https://debs-obrien.github.io/playwright-movies-app");
    }

    [Fact]
    public async Task SearchForMovieByTitle()
    {
        await Test.StepAsync("Activate and perform search", async () =>
        {
            await Page.GetByRole(AriaRole.Search).ClickAsync();
            var searchInput = Page.GetByRole(AriaRole.Textbox, new() { Name = "Search Input" });
            await searchInput.FillAsync("Garfield");
            await searchInput.PressAsync("Enter");
        });

        await Test.StepAsync("Verify search results", async () =>
        {
            // Verify the accessibility tree of the search results
            await Expect(Page.GetByRole(AriaRole.Main)).ToMatchAriaSnapshotAsync(@"
                - main:
                  - heading ""Garfield"" [level=1]
                  - heading ""search results"" [level=2]
                  - list ""movies"":
                    - listitem ""movie"":
                      - link ""poster of The Garfield Movie The Garfield Movie rating"":
                        - /url: /playwright-movies-app/movie?id=tt5779228&page=1
                        - img ""poster of The Garfield Movie""
                        - heading ""The Garfield Movie"" [level=2]
            ");
        });
    }
}
```

## Test Execution Strategy

1. **Initial Run**: Execute tests with `dotnet test` or using the test runner in your IDE
2. **Debug Failures**: Analyze test failures and identify root causes
3. **Iterate**: Refine locators, assertions, or test logic as needed
4. **Validate**: Ensure tests pass consistently and cover the intended functionality
5. **Report**: Provide feedback on test results and any issues discovered

## Quality Checklist

Before finalizing tests, ensure:

- [ ] All locators are accessible and specific and avoid strict mode violations
- [ ] Tests are grouped logically and follow a clear structure
- [ ] Assertions are meaningful and reflect user expectations
- [ ] Tests follow consistent naming conventions
- [ ] Code is properly formatted and commented

---


# Playwright Python Test Generation Instructions

## Test Writing Guidelines

### Code Quality Standards
- **Locators**: Prioritize user-facing, role-based locators (get_by_role, get_by_label, get_by_text) for resilience and accessibility.
- **Assertions**: Use auto-retrying web-first assertions via the expect API (e.g., expect(page).to_have_title(...)). Avoid expect(locator).to_be_visible() unless specifically testing for a change in an element's visibility, as more specific assertions are generally more reliable.
- **Timeouts**: Rely on Playwright's built-in auto-waiting mechanisms. Avoid hard-coded waits or increased default timeouts.
- **Clarity**: Use descriptive test titles (e.g., def test_navigation_link_works():) that clearly state their intent. Add comments only to explain complex logic, not to describe simple actions like "click a button."

### Test Structure
- **Imports**: Every test file should begin with from playwright.sync_api import Page, expect.
- **Fixtures**: Use the page: Page fixture as an argument in your test functions to interact with the browser page.
- **Setup**: Place navigation steps like page.goto() at the beginning of each test function. For setup actions shared across multiple tests, use standard Pytest fixtures.

### File Organization
- **Location**: Store test files in a dedicated tests/ directory or follow the existing project structure.
- **Naming**: Test files must follow the test_<feature-or-page>.py naming convention to be discovered by Pytest.
- **Scope**: Aim for one test file per major application feature or page.

## Assertion Best Practices
- **Element Counts**: Use expect(locator).to_have_count() to assert the number of elements found by a locator.
- **Text Content**: Use expect(locator).to_have_text() for exact text matches and expect(locator).to_contain_text() for partial matches.
- **Navigation**: Use expect(page).to_have_url() to verify the page URL.
- **Assertion Style**: Prefer `expect` over `assert` for more reliable UI tests.


## Example

```python
import re
import pytest
from playwright.sync_api import Page, expect

@pytest.fixture(scope="function", autouse=True)
def before_each_after_each(page: Page):
    # Go to the starting url before each test.
    page.goto("https://playwright.dev/")

def test_main_navigation(page: Page):
    expect(page).to_have_url("https://playwright.dev/")

def test_has_title(page: Page):
    # Expect a title "to contain" a substring.
    expect(page).to_have_title(re.compile("Playwright"))

def test_get_started_link(page: Page):
    page.get_by_role("link", name="Get started").click()
    
    # Expects page to have a heading with the name of Installation.
    expect(page.get_by_role("heading", name="Installation")).to_be_visible()
```

## Test Execution Strategy

1. **Execution**: Tests are run from the terminal using the pytest command.
2. **Debug Failures**: Analyze test failures and identify root causes

---


## Test Writing Guidelines

### Code Quality Standards
- **Locators**: Prioritize user-facing, role-based locators (`getByRole`, `getByLabel`, `getByText`, etc.) for resilience and accessibility. Use `test.step()` to group interactions and improve test readability and reporting.
- **Assertions**: Use auto-retrying web-first assertions. These assertions start with the `await` keyword (e.g., `await expect(locator).toHaveText()`). Avoid `expect(locator).toBeVisible()` unless specifically testing for visibility changes.
- **Timeouts**: Rely on Playwright's built-in auto-waiting mechanisms. Avoid hard-coded waits or increased default timeouts.
- **Clarity**: Use descriptive test and step titles that clearly state the intent. Add comments only to explain complex logic or non-obvious interactions.


### Test Structure
- **Imports**: Start with `import { test, expect } from '@playwright/test';`.
- **Organization**: Group related tests for a feature under a `test.describe()` block.
- **Hooks**: Use `beforeEach` for setup actions common to all tests in a `describe` block (e.g., navigating to a page).
- **Titles**: Follow a clear naming convention, such as `Feature - Specific action or scenario`.


### File Organization
- **Location**: Store all test files in the `tests/` directory.
- **Naming**: Use the convention `<feature-or-page>.spec.ts` (e.g., `login.spec.ts`, `search.spec.ts`).
- **Scope**: Aim for one test file per major application feature or page.

### Assertion Best Practices
- **UI Structure**: Use `toMatchAriaSnapshot` to verify the accessibility tree structure of a component. This provides a comprehensive and accessible snapshot.
- **Element Counts**: Use `toHaveCount` to assert the number of elements found by a locator.
- **Text Content**: Use `toHaveText` for exact text matches and `toContainText` for partial matches.
- **Navigation**: Use `toHaveURL` to verify the page URL after an action.


## Example Test Structure

```typescript
import { test, expect } from '@playwright/test';

test.describe('Movie Search Feature', () => {
  test.beforeEach(async ({ page }) => {
    // Navigate to the application before each test
    await page.goto('https://debs-obrien.github.io/playwright-movies-app');
  });

  test('Search for a movie by title', async ({ page }) => {
    await test.step('Activate and perform search', async () => {
      await page.getByRole('search').click();
      const searchInput = page.getByRole('textbox', { name: 'Search Input' });
      await searchInput.fill('Garfield');
      await searchInput.press('Enter');
    });

    await test.step('Verify search results', async () => {
      // Verify the accessibility tree of the search results
      await expect(page.getByRole('main')).toMatchAriaSnapshot(`
        - main:
          - heading "Garfield" [level=1]
          - heading "search results" [level=2]
          - list "movies":
            - listitem "movie":
              - link "poster of The Garfield Movie The Garfield Movie rating":
                - /url: /playwright-movies-app/movie?id=tt5779228&page=1
                - img "poster of The Garfield Movie"
                - heading "The Garfield Movie" [level=2]
      `);
    });
  });
});
```

## Test Execution Strategy

1. **Initial Run**: Execute tests with `npx playwright test --project=chromium`
2. **Debug Failures**: Analyze test failures and identify root causes
3. **Iterate**: Refine locators, assertions, or test logic as needed
4. **Validate**: Ensure tests pass consistently and cover the intended functionality
5. **Report**: Provide feedback on test results and any issues discovered

## Quality Checklist

Before finalizing tests, ensure:
- [ ] All locators are accessible and specific and avoid strict mode violations
- [ ] Tests are grouped logically and follow a clear structure
- [ ] Assertions are meaningful and reflect user expectations
- [ ] Tests follow consistent naming conventions
- [ ] Code is properly formatted and commented