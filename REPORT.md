# puppeteer fix report: BiDi `page.setCacheEnabled()` with no argument disables the cache

## Branch

|                         |                                                                                                                     |
| ----------------------- | ------------------------------------------------------------------------------------------------------------------- |
| Fork                    | `b84330808/puppeteer`                                                                                               |
| Fix branch              | `fix/bidi-set-cache-enabled-default`                                                                                |
| Commit SHA              | `eadd7213862e134d630771030c3cc0d28cf9f73b`                                                                          |
| Base                    | `upstream/main` @ `884a8c3a` (`fix(bidi): return domain cookies for the domain itself in page.cookies() (#15689)`)  |
| Author / committer      | `b84330808 <b84330808@gmail.com>`, no `Co-Authored-By` trailer                                                      |
| Diff                    | `packages/puppeteer-core/src/bidi/Page.ts` (+1/-1), `test/src/page.test.ts` (+13)                                   |
| `TestExpectations.json` | no change needed. No entry matches `Page.setCacheEnabled`, and every suite reported "Test run matches expectations" |

Compare / open PR from: `https://github.com/puppeteer/puppeteer/compare/main...b84330808:puppeteer:fix/bidi-set-cache-enabled-default`

### The diff

```diff
--- a/packages/puppeteer-core/src/bidi/Page.ts
+++ b/packages/puppeteer-core/src/bidi/Page.ts
@@ -667,7 +667,7 @@ export class BidiPage extends Page {
-  override async setCacheEnabled(enabled?: boolean): Promise<void> {
+  override async setCacheEnabled(enabled = true): Promise<void> {
     if (!this.#browserContext.browser().cdpSupported) {
       await this.#frame.browsingContext.setCacheBehavior(
         enabled ? 'default' : 'bypass',
--- a/test/src/page.test.ts
+++ b/test/src/page.test.ts
@@ -2276,6 +2276,19 @@ describe('Page', function () {
+    it('should enable the cache when called without arguments', async () => {
+      const {page, server} = await getTestState();
+
+      await page.setCacheEnabled(false);
+      await page.setCacheEnabled();
+
+      await page.goto(server.PREFIX + '/cached/one-style.html');
+      const [cachedRequest] = await Promise.all([
+        server.waitForRequest('/cached/one-style.html'),
+        page.reload(),
+      ]);
+      expect(cachedRequest.headers['if-modified-since']).not.toBe(undefined);
+    });
```

## PR title

```
fix(bidi): make page.setCacheEnabled() enable the cache by default
```

## PR body

```markdown
**What kind of change does this PR introduce?**

Bug fix (WebDriver BiDi).

**Did you add tests for your changes?**

Yes: `[page.test] Page Page.setCacheEnabled should enable the cache when called without arguments`. Without the fix it fails on chrome-bidi, chrome-bidi-only and firefox-headless. With the fix it passes on all of them. It passes on chrome-headless either way, because the CDP implementation already handles the default correctly.

**If relevant, did you update the documentation?**

Not needed. The documented behavior (`@defaultValue true`) is unchanged; this makes the BiDi implementation follow it.

**Summary**

`Page.setCacheEnabled(enabled?: boolean)` documents `enabled` with `@defaultValue true`, and the CDP implementation is `setCacheEnabled(enabled = true)`. `BidiPage.setCacheEnabled(enabled?: boolean)` has no default, so calling `page.setCacheEnabled()` with no arguments **disables** the cache under WebDriver BiDi:

- Chrome (BiDi with CDP available): `Network.setCacheDisabled({cacheDisabled: !undefined})` sends `cacheDisabled: true`.
- Firefox, and Chrome in BiDi-only mode: `browsingContext.setCacheBehavior(undefined ? 'default' : 'bypass')` sends `'bypass'`.

So after `await page.setCacheEnabled(false)`, a call to `await page.setCacheEnabled()` turns caching back on with CDP but leaves it off with BiDi. This change gives the parameter the same `= true` default the CDP implementation uses.

**Does this PR introduce a breaking change?**

No. The only calls affected are ones that omit the argument under WebDriver BiDi. They now behave as documented, the same as on CDP.

**Other information**

Tested on Linux with Chrome 156.0.8078.73 and Firefox 159.0 using the mocha runner, on the chrome-headless, chrome-bidi, chrome-bidi-only and firefox-headless suites.

I used Claude Code to write this change.
```

## Fail-before / pass-after

For each run I put `.only` on the new `it`, then removed it before committing. Commands: `npm run test:chrome:headless`, `npm run test:chrome:bidi`, `npm run test:firefox:headless`, `npm run test:chrome:bidi-only`. Environment notes:

- The container runs as root, so all runs used `EXTRA_LAUNCH_OPTIONS='{"args":["--no-sandbox"]}'`. The test harness reads this variable.
- Chrome 156.0.8078.73 and chrome-headless-shell came from `packages/puppeteer/install.mjs`.
- The cloud network policy blocks Mozilla's hosts (archive.mozilla.org, ftp.mozilla.org, download-installer.cdn.mozilla.net). Firefox 159.0 came instead from conda-forge's `firefox-159.0-hee9eb32_0.conda` (sha256 `ab94a5ff54d4dc5a18be18939cba098bdbff1d39dd28d6cf4e99fd19af8589de`, verified). That package is Mozilla's official linux-x86_64 tarball, unpacked with its `.sig` files intact. It was placed at `~/.cache/puppeteer/firefox/linux-stable_159.0/firefox/`, and `firefox --version` reports `Mozilla Firefox 159.0`.

"Before" means the new test was run with the `Page.ts` change reverted. "After" means the commit above.

### chrome-headless (CDP): control, passes both before and after

Before (fix reverted):

```
Running tests with:
  -> product: chrome
  -> mode: headless
  -> protocol: cdp
  Page
    Page.setCacheEnabled
      ✔ should enable the cache when called without arguments (165ms)
  1 passing (1s)
Finished ["chrome","headless","cdp"]
Test run matches expectations
Run succeeded.
```

After:

```
Running tests with:
  -> product: chrome
  -> mode: headless
  -> protocol: cdp
  Page
    Page.setCacheEnabled
      ✔ should enable the cache when called without arguments (190ms)
  1 passing (1s)
Finished ["chrome","headless","cdp"]
Test run matches expectations
Run succeeded.
```

### chrome-bidi: fails before, passes after

Before (fix reverted):

```
Running tests with:
  -> product: chrome
  -> mode: headless
  -> protocol: webDriverBiDi
  Page
    Page.setCacheEnabled
      1) should enable the cache when called without arguments
  0 passing (1s)
  1 failing
  1) Page
       Page.setCacheEnabled
         should enable the cache when called without arguments:
     Error: expect(received).not.toBe(expected) // Object.is equality
Expected: not undefined
      at Context.<anonymous> (file:///home/user/puppeteer/test/src/page.test.ts:2290:62)
Finished ["chrome","headless","webDriverBiDi"]
Run failed: 1 unexpected result(s).
```

After:

```
Running tests with:
  -> product: chrome
  -> mode: headless
  -> protocol: webDriverBiDi
  Page
    Page.setCacheEnabled
      ✔ should enable the cache when called without arguments (194ms)
  1 passing (1s)
Finished ["chrome","headless","webDriverBiDi"]
Test run matches expectations
Run succeeded.
```

### firefox-headless: fails before, passes after

Before (fix reverted):

```
Running tests with:
  -> product: firefox
  -> mode: headless
  -> protocol: webDriverBiDi
  Page
    Page.setCacheEnabled
      1) should enable the cache when called without arguments
  0 passing (6s)
  1 failing
  1) Page
       Page.setCacheEnabled
         should enable the cache when called without arguments:
     Error: expect(received).not.toBe(expected) // Object.is equality
Expected: not undefined
      at Context.<anonymous> (file:///home/user/puppeteer/test/src/page.test.ts:2290:62)
Finished ["firefox","headless","webDriverBiDi"]
Run failed: 1 unexpected result(s).
```

After:

```
Running tests with:
  -> product: firefox
  -> mode: headless
  -> protocol: webDriverBiDi
  Page
    Page.setCacheEnabled
      ✔ should enable the cache when called without arguments (297ms)
  1 passing (6s)
Finished ["firefox","headless","webDriverBiDi"]
Test run matches expectations
Run succeeded.
```

### chrome-bidi-only (extra: exercises the `setCacheBehavior` path in Chrome): fails before, passes after

Before (fix reverted):

```
Running tests with:
  -> product: chrome
  -> mode: headless
  -> protocol: webDriverBiDi
  Page
    Page.setCacheEnabled
      1) should enable the cache when called without arguments
  0 passing (1s)
  1 failing
  1) Page
       Page.setCacheEnabled
         should enable the cache when called without arguments:
     Error: expect(received).not.toBe(expected) // Object.is equality
Expected: not undefined
      at Context.<anonymous> (file:///home/user/puppeteer/test/src/page.test.ts:2290:62)
Finished ["chrome","headless","webDriverBiDiOnly"]
Run failed: 1 unexpected result(s).
```

After:

```
Running tests with:
  -> product: chrome
  -> mode: headless
  -> protocol: webDriverBiDi
  Page
    Page.setCacheEnabled
      ✔ should enable the cache when called without arguments (187ms)
  1 passing (1s)
Finished ["chrome","headless","webDriverBiDiOnly"]
Test run matches expectations
Run succeeded.
```

### Regression check: whole `Page.setCacheEnabled` block, with the fix

`describe.only('Page.setCacheEnabled')` was set temporarily and removed before committing:

```
== chrome:headless EXIT 0
      ✔ should enable or disable the cache based on the state passed (228ms)
      ✔ should enable the cache when called without arguments (225ms)
      ✔ should stay disabled when toggling request interception on/off (222ms)
  3 passing (2s)
Run succeeded.
== chrome:bidi EXIT 0
      ✔ should enable or disable the cache based on the state passed (230ms)
      ✔ should enable the cache when called without arguments (221ms)
      ✔ should stay disabled when toggling request interception on/off (175ms)
  3 passing (2s)
Run succeeded.
== firefox:headless EXIT 0
      ✔ should enable or disable the cache based on the state passed (371ms)
      ✔ should enable the cache when called without arguments (232ms)
      ✔ should stay disabled when toggling request interception on/off (175ms)
  3 passing (7s)
Run succeeded.
```

Other checks: `npm run build` succeeded, `npx eslint` on both changed files is clean, and `npx prettier --check` on both files reports "All matched files use Prettier code style!". Every other `setCacheEnabled` call in `test/src` passes an explicit boolean, so those tests are unaffected.

## Duplicate search (all states)

### Done from the cloud session

This session could not use the GitHub API on `puppeteer/puppeteer`. The session is scoped to the fork, and attaching the upstream repo was refused because it has the same repo name. So pull requests were searched through git instead:

- **All upstream PRs, open + closed + merged.** I listed `git ls-remote upstream 'refs/pull/*/head'`, which returned 8,109 refs, and fetched every PR head with `--depth=1`. In each head I read `BidiPage.setCacheEnabled` from `packages/puppeteer-core/src/bidi/Page.ts`. 3,470 PR heads contain the method, and all 3,470 declare it as `setCacheEnabled(enabled?: boolean)`. Across all of them the method body takes only 6 distinct forms. I inspected each one (for example PRs 11169, 12678, 13146, 13227, 14269 and 15696), and none defaults `enabled` to `true`. **No PR in any state contains this fix.**
- **PR head commit subjects** matching `setCacheEnabled|cache ?enabled|cacheDisabled|setCacheBehavior|cache behavior`: the only hit is puppeteer/puppeteer#1609, a 2017 api.md docs PR that is unrelated. Subjects matching `cache` turned up nothing about this bug.
- **upstream/main** at `884a8c3a` still has `setCacheEnabled(enabled?: boolean)` with no default.

### Still to do locally before opening the PR (issues could not be searched from the cloud)

```bash
gh search issues --repo puppeteer/puppeteer "setCacheEnabled" --limit 100
```

```bash
gh search prs --repo puppeteer/puppeteer "setCacheEnabled" --limit 100
```

```bash
gh search issues --repo puppeteer/puppeteer "setCacheEnabled firefox" --limit 100
```

```bash
gh search issues --repo puppeteer/puppeteer "cache bidi" --limit 100
```

With no `--state` flag, `gh search` returns open and closed results. Open the PR only if none of them reports this bug or already has a fix.

## Other leads noted (not taken)

- **BiDi `setGeolocation` accuracy.** The lead is real but weak, and it is still present on upstream/main. `BidiPage.setGeolocation` destructures `accuracy = 0` for validation, then sends `options.accuracy` instead. When `accuracy` is omitted, WebDriver BiDi's `emulation.GeolocationCoordinates.accuracy` defaults to `1` (`@defaultValue 1` in `webdriver-bidi-protocol`), while CDP sends `0`. So `coords.accuracy` is 1 under BiDi and 0 under CDP. Puppeteer does not document a default for `accuracy`, which makes the case for changing it weaker than this one. It could be a candidate for another day.
