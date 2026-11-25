<h1 align="center">playwright-migrate</h1>

<p align="center">
  <strong>Move Selenium and Cypress tests to Playwright, without silent passes.</strong><br/>
  <strong>Detect • Migrate • Verify</strong><br/>
  <sub>Deterministic conversion of Selenium Java, Selenium Python and Cypress JS tests into Playwright Test TypeScript, with a report of everything it could not convert.</sub>
</p>

<p align="center">
  <a href="https://github.com/sprintsynergyio/playwright-migrate/actions/workflows/ci.yml"><img src="https://github.com/sprintsynergyio/playwright-migrate/actions/workflows/ci.yml/badge.svg" alt="CI" /></a>
  <a href="https://www.npmjs.com/package/@sprintsynergy/playwright-migrate"><img src="https://img.shields.io/npm/v/%40sprintsynergy%2Fplaywright-migrate.svg" alt="npm version" /></a>
  <a href="https://www.npmjs.com/package/@sprintsynergy/playwright-migrate"><img src="https://img.shields.io/npm/dm/%40sprintsynergy%2Fplaywright-migrate.svg" alt="npm downloads" /></a>
  <a href="LICENSE"><img src="https://img.shields.io/badge/license-MIT-blue.svg" alt="MIT license" /></a>
  <img src="https://img.shields.io/badge/node-%3E%3D22.13-339933?logo=node.js&logoColor=white" alt="Node 22.13 or newer" />
</p>

<p align="center">
  <a href="#quick-start">Quick Start</a> · <a href="#features">Features</a> · <a href="#how-it-works">How it works</a> · <a href="#usage">Usage</a> · <a href="#blocked-patterns">Blocked patterns</a> · <a href="#documentation">Docs</a> · <a href="https://www.npmjs.com/package/@sprintsynergy/playwright-migrate">npm</a>
</p>

---

`playwright-migrate` is a command line tool that converts Selenium (Java and Python) and Cypress (JavaScript) tests into Playwright Test suites written in TypeScript. It keeps each test's intent, steps, locators and assertions, and it lists anything it cannot convert safely in a report, so nothing turns into a test that passes silently.

<table>
<tr>
<td width="33%" valign="top">

### Deterministic

Each source file is parsed into a syntax tree and mapped through rule tables. There is no guessing and no AI step, so the same input gives the same output.

</td>
<td width="33%" valign="top">

### Never a silent pass

Anything it cannot convert safely is flagged. A blocked test is emitted as `test.fixme` with its original source kept as a comment, so it can never pass.

</td>
<td width="33%" valign="top">

### Verified output

`verify` runs the converted specs with Playwright, five times each by default and with no retries, and adds the results to the migration report.

</td>
</tr>
</table>

<!-- A short GIF of detect, migrate and verify on the SauceDemo fixtures would help here. Record a real run; do not use a generated image. -->

---

## Quick Start

Node.js 22.13 or later. Install it as a dev dependency in the project you are migrating:

```sh
npm install -D @sprintsynergy/playwright-migrate
npx playwright-migrate detect <path>
npx playwright-migrate migrate path/to/tests --out playwright-migrated
npx playwright-migrate verify playwright-migrated
```

`detect` prints the framework for each file. `migrate` writes the specs, a `playwright.config.ts` and `migration-report.json` / `migration-report.md`. `verify` needs `@playwright/test` and Chromium (`npx playwright install chromium`) where the output folder is. Exit codes and every option are in [Usage](#usage). A full run on this repo's fixtures is in [Example](#example).

---

## Features

- **Three sources, one target:** Selenium Java (JUnit 5 and TestNG), Selenium Python (pytest and unittest) and Cypress JS, to Playwright Test in TypeScript. See [Sources And Target](#sources-and-target).
- **One file or a whole folder:** a folder run mirrors the layout in `--out`, writes one `playwright.config.ts` and one report, and lists files it cannot detect as skipped.
- **A report per run:** every step is converted, converted with a note, or blocked, with INFO, WARN and BLOCK flags. `migration-report.json` follows [`schemas/migration-report.schema.json`](schemas/migration-report.schema.json).
- **Blocked patterns:** network intercepts, custom commands, frames, windows and JavaScript executor calls are never guessed. See [Blocked Patterns](#blocked-patterns).
- **Safe writes:** `migrate` never overwrites a file without `--force`, and `--dry-run` writes nothing.
- **CI exit codes:** 0 when nothing is blocked, 2 when at least one test is blocked, 1 on an error.
- **Library API:** `planMigration`, `writeMigration` and `convertSource`, with type declarations. See [Library Use](#library-use).
- **Mapping reference:** [docs/mapping.md](docs/mapping.md) is generated from the rule tables.

## Status

Version 0.1.0 is the first release. `detect` works for all three sources. `migrate` converts one Cypress JS, Selenium Python or Selenium Java file, or every supported file under a folder, and `verify` runs the converted specs and records the results. See [CHANGELOG.md](CHANGELOG.md).

## Sources And Target

Sources:

- Selenium Java with JUnit 5 and TestNG
- Selenium Python with pytest and unittest
- Cypress JS

Target: Playwright Test in TypeScript.

---

## How It Works

```
  Selenium Java / Selenium Python / Cypress JS file or folder
        │
  detect    framework per file, from its contents
  parse     each file into a syntax tree (no AI step)
  map       commands, locators and assertions through the rule tables
  emit      Playwright Test TypeScript; blocked tests become test.fixme with the source as a comment
  report    migration-report.json and migration-report.md
  verify    optional: run the specs with Playwright and add the results to the report
```

- Deterministic parsing: each source file is parsed into a syntax tree. There is no guessing and no AI step.
- Mapping tables: each source command, locator and assertion maps to its Playwright form through a table of rules. The full list is in [docs/mapping.md](docs/mapping.md), which is generated from the tables.
- A report of what was not converted: every step is listed as converted, converted with a note, or blocked.
- Blocked tests are emitted as `test.fixme`, with the original source kept as a comment, so a blocked test can never pass.
- A source test that cannot be read at all, for example a Cypress test whose callback takes a `done` parameter, is not emitted. Its source is printed as a comment under the import, the report lists it as blocked, and its assertions are counted as blocked.
- Code outside the tests that is not converted, for example a module-level statement, a `setUpClass` method or another class member, adds a BLOCK flag. Every test in that file is then emitted as `test.fixme` and listed as blocked, because that code may set up state the tests need.
- `it.only` and `describe.only` are converted without `.only` and get a WARN flag, because `test.only` would skip every other test in the run. `xit`, `xdescribe` and `.skip` become `test.skip` and `test.describe.skip`.
- Selenium Java: JUnit 5 `@Disabled` and TestNG `@Test(enabled = false)` become `test.skip`, and a JUnit 5 `@DisplayName` becomes the test title. JUnit 5 and TestNG assertions are both read, so `assertEquals` works with either order of expected and actual values. Fully qualified Selenium types, such as `new org.openqa.selenium.chrome.ChromeDriver()`, are read like their short names.

## Blocked Patterns

These patterns are never converted. Each one adds a BLOCK flag to the report, and the test that contains it is emitted as `test.fixme` with the source kept as a comment. The list is generated from the rule catalog.

<!-- block-rules:start (generated by npm run docs:generate) -->

- `CORE_UNKNOWN_STATEMENT` (All sources): Unrecognized statement. Statement not recognized, so it was not converted. Convert it by hand.
- `CORE_UNSUPPORTED_LOCATOR` (All sources): Unsupported locator. Locator {detail} is not supported in v0.1. Convert it by hand.
- `CORE_UNSUPPORTED_ASSERTION` (All sources): Unsupported assertion. Assertion {detail} is not supported in v0.1. Convert it by hand.
- `CORE_UNSUPPORTED_MEMBER` (Selenium Python and Java): Unsupported class member. Class member {detail} is not converted. Move shared setup into a hook or a fixture by hand.
- `CORE_CUSTOM_WAIT` (Selenium Python and Java): Custom wait condition. Wait with a custom condition is not converted. Rewrite it as a web-first assertion.
- `CORE_FRAME` (Selenium Python and Java): Frame switching. Frame switching is not converted. Use page.frameLocator for the steps inside the frame.
- `CORE_WINDOW` (Selenium Python and Java): Window, tab or alert handling. Window, tab or alert handling is not converted. Use Playwright popup, page or dialog events.
- `CORE_JS_EXECUTOR` (Selenium Python and Java): JavaScript executor. JavaScript executor calls are not converted. Use a Playwright action, or page.evaluate after review.
- `CY_CUSTOM_COMMAND` (Cypress JS): Custom command. Custom command cy.{name} is not converted. Port it to a helper function or a fixture.
- `CY_INTERCEPT` (Cypress JS): Network intercept. cy.intercept is not converted. Rewrite it with page.route or page.waitForResponse, registered before the action that sends the request.
- `CY_ALIAS` (Cypress JS): Alias. Cypress aliases (.as and cy.wait('@alias')) are not converted. Keep a variable or a page.waitForResponse promise instead.
- `CY_THEN` (Cypress JS): Callback. Callbacks (.then, .each, .spread, should with a function) are not converted. Rewrite the callback with locators and expect.
- `CY_UNSUPPORTED_COMMAND` (Cypress JS): Unsupported Cypress command. Cypress command .{name}() has no safe Playwright mapping in v0.1. Convert this step by hand.
- `CY_BEFORE_HOOK` (Cypress JS): Run-once hook. before and after hooks are not converted, because Playwright beforeAll and afterAll hooks cannot use the page fixture. Move these steps into beforeEach or afterEach by hand.

<!-- block-rules:end -->

## Known Limits

- Only Cypress JS (`.cy.js` and `.js` specs), Selenium Python and Selenium Java files are read. Cypress TypeScript, Selenium JavaScript and other languages are not.
- Page objects, shared helpers in other files and Cypress support files are not converted. A call to a custom command or helper is blocked.
- Network stubbing, aliases, sessions, `cy.origin`, `cy.task` and `cy.fixture` are blocked, and so are frames, extra windows and tabs, alerts and JavaScript executor calls.
- Fixed waits (`cy.wait(ms)`, `time.sleep`, `Thread.sleep`) are kept as `page.waitForTimeout` with a WARN flag, because the condition they wait for is not known.
- Locators are kept as written. XPath stays XPath, with an INFO flag.
- Driver setup and teardown are removed with an INFO flag, including remote drivers and their capabilities, so a suite that ran on a Selenium Grid runs locally after conversion. The generated `playwright.config.ts` runs Chromium only, with retries 0.
- The conversions are checked against the SauceDemo fixtures in this repo. A real suite will usually have more blocked steps. Run `verify` on the result and read the report before you rely on it.

---

## Usage

Install it as a dev dependency in the project you are migrating. It needs Node.js 22.13 or later.

```sh
npm install -D @sprintsynergy/playwright-migrate
npx playwright-migrate detect <path>
```

`detect` prints the detected framework for each source file under `<path>` and a summary line. Add `--json` to print the result as JSON.

```sh
npx playwright-migrate migrate path/to/login.cy.js --out playwright-migrated
```

`migrate` writes `login.spec.ts`, `migration-report.json` and `migration-report.md` to the `--out` folder (default `playwright-migrated`). Add `--dry-run` to print the file names without writing anything. The exit code is 0 when nothing is blocked, 2 when at least one test is blocked, and 1 on an error.

```sh
npx playwright-migrate migrate path/to/tests --out playwright-migrated
```

For a folder, `migrate` converts every Cypress JS, Selenium Python and Selenium Java file it detects and mirrors the folder layout in `--out`. It also writes a `playwright.config.ts` and one report for the whole run. Files it cannot detect are listed as skipped. Folders named `node_modules`, `.git`, `dist` and `expected` are not scanned. With a folder, `--from` limits the run to one framework.

`migrate` never overwrites a file by default. When an output file already exists, it lists the files, writes nothing and exits with 1. Add `--force` to overwrite them.

```sh
npx playwright-migrate verify playwright-migrated
```

`verify` runs the converted specs in a folder written by `migrate` with Playwright, five times each by default (`--repeat-each <n>`) and with no retries. It needs `@playwright/test` and the Chromium browser installed where the folder is. It prints the passed, failed and skipped runs per spec file and writes the counts into `migration-report.json` and `migration-report.md`. Blocked tests are `test.fixme`, so they are counted as skipped. The exit code is 0 when at least one test passed and none failed or were flaky, and 1 otherwise, so a folder where every test is skipped returns 1. Run it on a folder written by a folder run of `migrate`, because only a folder run writes the `playwright.config.ts` that `verify` needs.

## Example

These are real runs of version 0.1.0 on the SauceDemo fixtures in this repo. Cypress JS source, `fixtures/saucedemo/cases/login-success/cypress-js/login-success.cy.js`:

```js
describe('Login', () => {
  beforeEach(() => {
    cy.visit('https://www.saucedemo.com/');
  });

  it('logs in with the standard user', () => {
    cy.get('[data-test="username"]').type('standard_user');
    cy.get('[data-test="password"]').type('secret_sauce');
    cy.get('[data-test="login-button"]').click();
    cy.url().should('include', '/inventory.html');
    cy.get('[data-test="title"]').should('have.text', 'Products');
  });
});
```

```sh
npx playwright-migrate migrate fixtures/saucedemo/cases/login-success/cypress-js/login-success.cy.js --out .demo/readme1
```

```text
Wrote .demo/readme1/login-success.spec.ts
Wrote .demo/readme1/migration-report.json
Wrote .demo/readme1/migration-report.md
login-success.cy.js: converted, 1 test(s), INFO 2, WARN 0, BLOCK 0
```

The written `login-success.spec.ts`:

```ts
// Migrated from: login-success.cy.js
import { test, expect } from '@playwright/test';

test.describe('Login', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('https://www.saucedemo.com/');
  });

  test('logs in with the standard user', async ({ page }) => {
    await page.locator('[data-test="username"]').fill('standard_user');
    await page.locator('[data-test="password"]').fill('secret_sauce');
    await page.locator('[data-test="login-button"]').click();
    await expect(page).toHaveURL(/\/inventory\.html/);
    await expect(page.locator('[data-test="title"]')).toHaveText('Products');
  });
});
```

The two INFO flags in the report say that `type` became `fill`, which replaces the field value in one step.

A folder run, then `verify`:

```sh
npx playwright-migrate migrate fixtures/saucedemo/cases --from cypress-js --out .demo/readme
npx playwright-migrate verify .demo/readme
```

The last line of `migrate` (the Wrote, Skipped and per-file lines are left out here):

```text
Files: 6 (converted 6, with warnings 0, blocked 0), skipped 12. Exit code 0.
```

The output of `verify`:

```text
Running Playwright in .demo/readme with --repeat-each=5 and retries 0.
add-to-cart/cypress-js/add-to-cart.spec.ts: 5 passed, 0 failed, 0 skipped of 5 runs
cart-badge/cypress-js/cart-badge.spec.ts: 5 passed, 0 failed, 0 skipped of 5 runs
checkout-complete/cypress-js/checkout-complete.spec.ts: 5 passed, 0 failed, 0 skipped of 5 runs
login-locked-out/cypress-js/login-locked-out.spec.ts: 5 passed, 0 failed, 0 skipped of 5 runs
login-success/cypress-js/login-success.spec.ts: 5 passed, 0 failed, 0 skipped of 5 runs
sort-products/cypress-js/sort-products.spec.ts: 5 passed, 0 failed, 0 skipped of 5 runs
Updated .demo/readme/migration-report.json and .demo/readme/migration-report.md
Verified with --repeat-each=5: 30 runs, 30 passed, 0 failed, 0 skipped, 0 flaky.
```

The same folder run with `--from selenium-python` or `--from selenium-java` also converts 6 files with nothing blocked, and `verify` gives 30 passed of 30 runs for each. The skipped files in the run above are the Selenium files, which `--from cypress-js` leaves out.

## Library Use

The package also exports its API, with type declarations:

```ts
import { convertSource, planMigration, writeMigration } from '@sprintsynergy/playwright-migrate';

const plan = await planMigration('path/to/tests');
await writeMigration('playwright-migrated', plan);
console.log(plan.report.summary);
```

`planMigration` converts in memory and writes nothing. `convertSource` converts the text of one file.

---

## Configuration

There is no config file. Every option is a flag:

| Command           | Options                                                                                                                    |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------- |
| `detect <path>`   | `--json`                                                                                                                   |
| `migrate <input>` | `--out <dir>` (default `playwright-migrated`), `--from cypress-js\|selenium-python\|selenium-java`, `--dry-run`, `--force` |
| `verify [dir]`    | `--repeat-each <n>` (default 5)                                                                                            |

`npx playwright-migrate <command> --help` prints the same list. The generated `playwright.config.ts` runs Chromium only, with retries 0.

## Platform notes

- **Node.js 22.13 or later.** CI runs on Ubuntu with Node 22 and 24.
- **Browsers:** `verify` and the expected specs need Chromium: `npx playwright install chromium`. On Linux, add `--with-deps` if Chromium's system libraries are missing.
- **Network:** `verify` on the SauceDemo fixtures, `npm run test:pack` and `npm run demo:verify` call the network.
- **Selenium Grid:** remote driver setup and capabilities are removed with an INFO flag, so the converted suite runs locally.

---

## Development

Requires Node.js 22.13 or later. To work on the tool itself, clone the repo and run `npm ci` and `npm run build`.

| Script                    | What It Does                                                                                                                                                                   |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `npm run build`           | Removes `dist/`, then compiles `src/` to `dist/`.                                                                                                                              |
| `npm run typecheck`       | Type-checks the source, tests, expected specs and config files.                                                                                                                |
| `npm run lint`            | Runs ESLint.                                                                                                                                                                   |
| `npm run format`          | Formats the repo with Prettier.                                                                                                                                                |
| `npm run format:check`    | Checks formatting with Prettier.                                                                                                                                               |
| `npm run check:style`     | Fails if any tracked file contains an em dash or an en dash.                                                                                                                   |
| `npm test`                | Runs the unit and golden layout tests with Vitest.                                                                                                                             |
| `npm run test:smoke`      | Runs the built CLI: `--version`, `detect`, a parse check and a `migrate --dry-run`.                                                                                            |
| `npm run test:pack`       | Builds and packs the package, checks the tarball for stale build output, installs it into a temp folder, imports the API and runs the installed command. Needs network access. |
| `npm run e2e:expected`    | Runs the expected Playwright specs against SauceDemo in Chromium.                                                                                                              |
| `npm run e2e:expected:5x` | Runs the expected Playwright specs five times each.                                                                                                                            |
| `npm run demo:verify`     | Converts the SauceDemo cases once per source framework into `.demo/` and runs `verify` on each folder. Run `npm run build` first. Needs network access.                        |
| `npm run docs:generate`   | Builds, then regenerates `docs/mapping.md` and the blocked pattern list in this README from the rule catalog and the mapping tables.                                           |
| `npm run release:check`   | Checks that a tag, for example `v0.1.0`, matches `package.json` and the CHANGELOG, and that the package is set up to publish. The Release workflow runs it.                    |
| `npm run check`           | Runs lint, format check, style check, typecheck and tests.                                                                                                                     |

Before running the expected specs for the first time, install Chromium with `npx playwright install chromium`.

## Releasing

Changes are listed in [CHANGELOG.md](CHANGELOG.md). To release, a maintainer pushes a tag `v<version>` that matches the version in `package.json`. The Release workflow (`.github/workflows/release.yml`) checks the tag with `npm run release:check`, runs every check, the expected specs and the demo suites, and then publishes to npm with provenance. If that exact version is already on the registry, publish is skipped.

## Fixtures

All fixtures use SauceDemo (https://www.saucedemo.com), a public practice app. Never add client code to this repo. See [fixtures/saucedemo/README.md](fixtures/saucedemo/README.md) for the layout and the expected spec conventions.

## Documentation

| Doc                                                                          | For                                                                 |
| ---------------------------------------------------------------------------- | ------------------------------------------------------------------- |
| [docs/mapping.md](docs/mapping.md)                                           | every source command, locator and assertion and its Playwright form |
| [schemas/migration-report.schema.json](schemas/migration-report.schema.json) | the migration report JSON Schema                                    |
| [fixtures/saucedemo/README.md](fixtures/saucedemo/README.md)                 | the fixture layout and expected spec conventions                    |
| [CONTRIBUTING.md](CONTRIBUTING.md)                                           | working on the tool                                                 |
| [CHANGELOG.md](CHANGELOG.md)                                                 | release notes                                                       |

## Contributing

Found a bug or have an idea? Open an issue. We review pull requests case by case.

## License

MIT. See [LICENSE](LICENSE).

---

Built by SprintSynergy (https://sprintsynergy.com) and maintained by [Muhammad Usman](https://linkedin.com/in/sdet-muhammadusman).

---

<p align="center">
  <strong>Built by <a href="https://sprintsynergy.com">SprintSynergy</a></strong><br/>
  <sub>SprintSynergy is an AI-based QA automation company: AI testing agents plus our QA team.</sub><br/>
  <sub>Maintained by <a href="https://linkedin.com/in/sdet-muhammadusman">Muhammad Usman</a>.</sub>
</p>
