---
name: test-form-ui
description: >
  Tests the rendered UI of an AEM Adaptive Form (Core Components, AEM as a Cloud Service)
  against the SOURCE REFERENCE the form was built from — a reference screenshot (image file),
  a reference URL (another HTML form), or an XDP/PDF form — and produces a shareable comparison
  report. UI parity against a reference is MANDATORY in every run, on every form; it is never
  skipped and never replaced by functional-only checks. The target may be a CLOUD AUTHOR or
  CLOUD PUBLISH URL as well as the local SDK. The capture is CROPPED TO THE EMBEDDED FORM REGION,
  never the whole Sites page, so page header/footer/nav chrome cannot make parity fail spuriously.
  Captures the live form with the project's existing Cypress module (ui.tests/test-module), runs a
  PIXEL diff for a hard pass/fail gate, then a Claude VISION pass that explains what actually
  differs (missing/extra fields, wrong labels, field order, layout, colour, spacing) with severity.
  Use whenever someone asks to visually test / QA / verify / compare a form's UI against a design,
  mockup, screenshot, XDP, or another URL.
version: 1.0.0
ide:
  cursor: .cursor/skills/test-form-ui/
  github-copilot: .github/skills/test-form-ui/
  claude-code: .claude/skills/test-form-ui/
---

# Skill: test-form-ui

## Role

You are an AEM Adaptive Forms UI QA engineer. Given a form (a JCR path or a rendered URL) and
a **reference** (a screenshot image OR a URL), you capture the form as it actually renders,
compare it to the reference two ways, and deliver a single **report** the team can read:

1. **Pixel diff** — a deterministic pass/fail gate (mismatch % vs a threshold) with a diff image.
2. **Vision findings** — Claude reads the reference, the actual capture, and the diff, then lists
   concrete differences (missing field, extra field, wrong label, wrong order, layout shift,
   off-brand colour/spacing) graded by severity.

You **reuse the project's existing Cypress module** at `ui.tests/test-module/` — do not introduce
a second browser-automation stack. You never invent a form URL or a selector; you confirm the
inputs below first. The deliverable is the report file plus the three images, not just a console pass.

> This skill **tests** an existing, deployed form. It does not author or fix one. If the form
> isn't deployed yet, run `create-adaptive-form` (or `migrate-form`) and deploy first — a UI test
> against a 404 is meaningless.

---

## Step 1 — Collect inputs

Resolve tokens from `.aem-forms-config.yaml` (`{project}`, `formsContentRoot`); ask
the user only for what's genuinely unknown.

| Input | Description | Example |
|---|---|---|
| `{formName}` | Kebab-case node name of the form under test | `lab-test-form` |
| `{formUrl}` | Full URL of the **rendered** form (or the Sites page that embeds it). Author: `{AEM_AUTHOR_URL}{formsContentRoot}/{project}/{formName}.html` (needs login). Publish: `{AEM_PUBLISH_URL}/...` (often anonymous). **When invoked by Sentinel this is the CLOUD DEV URL** (`cloudDev.publishUrl` from `.aem-forms-config.yaml`), not localhost — the local SDK URL applies only to a standalone/dev run of this skill. | `https://publish-p12345-e67890.adobeaemcloud.com/content/{project}/us/en/test-adaptive-form.html` |
| **Reference — MANDATORY, exactly ONE of the three** | Parity is never optional. If no reference can be obtained, **stop and ask for one**; do not run a functional-only pass and call it a UI test. | |
| `{referenceImage}` | Absolute path to a reference screenshot/mockup (`.png`/`.jpg`) | `C:\designs\lab-test.png` |
| `{referenceUrl}` | A URL to screenshot and use as the reference (a design preview, a prod form, the old form) | `https://example.com/old-form` |
| `{referenceXdp}` | Path to an XDP/PDF form used as the reference. Render it to PNG first (`create-form-tests`/any PDF rasteriser, or open + export), then treat the PNG as `{referenceImage}`. Record in the report that the reference was rasterised from an XDP. | `C:\forms\policy.xdp` |
| `{captureScope}` | **`form`** (default) = crop the capture to the embedded form region only. `page` = whole page, for the rare case the reference genuinely *is* a full page. | `form` |
| `{formSelector}` | The element to crop to when `{captureScope}=form`. Default `.cmp-adaptiveform-container`. Use the AEM Form Container wrapper if the page nests it. | `.cmp-adaptiveform-container` |
| `{viewport}` | Viewport WxH the comparison runs at (match the reference's intent) | `1280x1024` |
| `{maxMismatchPct}` | Pixel-diff gate: max allowed mismatch % before FAIL. **Replica delivery (screenshot/URL) = `10` (≥90% pixel match required).** Migration-parity = `100` (advisory). | `10` |

> ⚠️ **Pixel diff is viewport-sensitive.** Both images are normalised to the captured form's
> dimensions before diffing, but a reference taken at a wildly different aspect ratio will inflate
> the mismatch %. When the reference is a mockup of unknown size, **trust the vision findings over
> the raw pixel %** and say so in the report. Capture at a fixed `{viewport}` so reruns are stable.

> ⚠️ **Author render needs auth; publish usually doesn't.** If `{formUrl}` points at a **local** author
> instance (`:4502`), the spec logs in via the existing `AEMLogin` command first. For a public
> `:4503`/published URL it visits directly. State which path you used in the report.
>
> ⚠️ **On AEMaaCS (cloud author or publish) there is no `admin:admin`.** Both tiers are valid targets;
> pick in this order, and **name the tier and URL you actually captured** in the report:
>
> 1. **Cloud publish** (`https://publish-p{prog}-e{env}.adobeaemcloud.com/...`) — anonymous, most
>    reliable, no login flow to break. Prefer this whenever the form/page is published.
> 2. **Cloud author + Bearer token** (`$AEM_DEV_ACCESS_TOKEN` from Developer Console) — use when the
>    page is only on author. Never basic auth.
> 3. **Cloud author + interactive Adobe IMS login** (username/password supplied by the human) — the
>    fallback. Be aware this is genuinely fragile in Cypress: IMS is a **different origin**, so the
>    login must run inside `cy.origin()`, and federated SSO or MFA on the account can make it
>    unautomatable. If IMS login cannot complete, report **BLOCKED** and ask for a publish URL or a
>    Bearer token — do **not** fall back to localhost to manufacture a "successful" capture, and do not
>    report a localhost capture as a cloud result.

### Mode: migration parity (when invoked by `migrate-form` / Phase 13 after a migration)

When the reference is the **original** form and the form under test is the **migrated** one, the
goal is *structural parity*, not pixel identity — migration deliberately remaps the theme, so
colour/typography/spacing are *expected* to change. In this mode:

- Treat the **pixel gate as advisory**: pass `{maxMismatchPct}` high (e.g. `100`) so an intended
  theme change can't read as failure. Still capture and show the diff image.
- **PASS = zero Critical findings.** Critical = a field/section/submit present in the original but
  missing (or extra) in the migrated form, a changed label, a reordered section, or a control whose
  type changed (radio→dropdown). Those are real migration regressions.
- Colour/font/spacing differences are **Major** and *expected* — list them, label them "intended
  theme remap", and do **not** fail the verdict on them.
- Say in the report's verdict line that this was a migration-parity run (structural basis).

---

## Step 2 — One-time setup of the Cypress module (idempotent — skip what already exists)

The capture, pixel diff, and path plumbing live in the existing module at `ui.tests/test-module/`.
Add these if they are not already present; do not duplicate.

**2.1 Dev dependencies** (pixel diff is done transparently with `pixelmatch`, not a baseline-managed
plugin, because we compare against an *arbitrary supplied* reference, not a stored baseline):

```bash
cd ui.tests/test-module
npm install --save-dev pixelmatch@5 pngjs sharp
```

> ⚠️ **Pin `pixelmatch@5`.** v6+ is ESM-only and `require('pixelmatch')` in the CommonJS
> `cypress.config.js` throws `ERR_REQUIRE_ESM`. v5.x is CommonJS and `require`-able. `pngjs` and
> `sharp` are both CommonJS-safe. If `sharp`'s native binary fails to install behind a proxy, set a
> mirror or install with `--os`/`--cpu` flags for your platform.

**2.2 Register the Node-side tasks** in `cypress.config.js` inside `e2e.setupNodeEvents(on, config)`
(keep the existing `cypress-terminal-report` line):

```js
// --- form-ui-compare: capture path + pixel diff tasks ---
const fs = require('fs');
const path = require('path');
const { PNG } = require('pngjs');
const pixelmatch = require('pixelmatch');
const sharp = require('sharp');

let lastScreenshotPath = null;
on('after:screenshot', (details) => { lastScreenshotPath = details.path; return details; });

on('task', {
  getLastScreenshot() { return lastScreenshotPath; },

  // Normalises the reference to the actual capture's dimensions, then pixel-diffs.
  async pixelDiff({ actualPath, referencePath, diffPath, threshold = 0.1 }) {
    const actual = PNG.sync.read(fs.readFileSync(actualPath));
    const refBuf = await sharp(referencePath)
      .resize(actual.width, actual.height, { fit: 'fill' })
      .png().toBuffer();
    const reference = PNG.sync.read(refBuf);
    const { width, height } = actual;
    const diff = new PNG({ width, height });
    const mismatched = pixelmatch(actual.data, reference.data, diff.data, width, height, { threshold });
    fs.mkdirSync(path.dirname(diffPath), { recursive: true });
    fs.writeFileSync(diffPath, PNG.sync.pack(diff));
    // copy the normalised reference next to the outputs so the report can show all three
    fs.writeFileSync(path.join(path.dirname(diffPath), 'reference.png'), refBuf);
    fs.copyFileSync(actualPath, path.join(path.dirname(diffPath), 'actual.png'));
    return { mismatched, total: width * height,
             mismatchPercentage: (mismatched / (width * height)) * 100, width, height };
  },
});
return config; // setupNodeEvents must return config
```

**2.3 The spec** — create `ui.tests/test-module/cypress/e2e/form-ui-compare.cy.js`:

```js
/* form-ui-compare — capture a form and pixel-diff it against a reference.
   Driven by Cypress env vars (see the run command in Step 3). */
const OUT = 'cypress/results/form-ui';

describe('Adaptive Form UI comparison', () => {
  const formUrl     = Cypress.env('FORM_URL');
  const refImage    = Cypress.env('REFERENCE_IMAGE'); // absolute path, OR
  const refUrl      = Cypress.env('REFERENCE_URL');   // a URL to screenshot
  const [vw, vh]    = (Cypress.env('VIEWPORT') || '1280x1024').split('x').map(Number);
  const maxMismatch = Number(Cypress.env('MAX_MISMATCH_PCT') || 1.0);
  // Crop target: compare the EMBEDDED FORM only, not the surrounding Sites page.
  const formSel     = Cypress.env('FORM_SELECTOR') || '.cmp-adaptiveform-container';
  const scope       = (Cypress.env('CAPTURE_SCOPE') || 'form').toLowerCase(); // 'form' | 'page'

  before(() => {
    // Local author (:4502) → Granite login form.
    if (formUrl.includes(':4502')) {
      cy.AEMForceLogout();
      cy.visit(Cypress.env('AEM_AUTHOR_URL'));
      cy.AEMLogin(Cypress.env('AEM_AUTHOR_USERNAME'), Cypress.env('AEM_AUTHOR_PASSWORD'));
    }
    // Cloud author (author-p{prog}-e{env}) → Adobe IMS, a DIFFERENT ORIGIN. Prefer a Bearer
    // token; only attempt interactive login when credentials are supplied, and wrap it in
    // cy.origin() because Cypress cannot drive a cross-origin form otherwise.
    else if (/author-p\d+-e\d+/.test(formUrl)) {
      const tok = Cypress.env('AEM_DEV_ACCESS_TOKEN');
      if (tok) {
        cy.intercept({ url: `${new URL(formUrl).origin}/**` }, (req) => {
          req.headers.authorization = `Bearer ${tok}`;
        });
      } else {
        const u = Cypress.env('AEM_IMS_USERNAME'), p = Cypress.env('AEM_IMS_PASSWORD');
        expect(u && p, 'cloud author needs AEM_DEV_ACCESS_TOKEN, or AEM_IMS_USERNAME + AEM_IMS_PASSWORD').to.be.ok;
        cy.visit(formUrl); // redirects to IMS
        cy.origin('https://auth.services.adobe.com', { args: { u, p } }, ({ u, p }) => {
          cy.get('input[name="username"], #EmailPage-EmailField', { timeout: 30000 }).type(u);
          cy.contains('button', /continue/i).click();
          cy.get('input[type="password"]', { timeout: 30000 }).type(p, { log: false });
          cy.contains('button', /continue|sign in/i).click();
        });
      }
    }
    // Publish / public URL → no auth.
  });

  it('captures the form and matches the reference within threshold', () => {
    cy.viewport(vw, vh);

    // (a) Resolve the reference to a local PNG. If a URL was given, capture it first — cropped
    //     to the SAME form region so we compare like with like.
    let referencePath = refImage;
    if (!refImage && refUrl) {
      cy.visit(refUrl);
      cy.wait(1500);
      if (scope === 'form') {
        cy.get(formSel, { timeout: 30000 }).should('exist')
          .screenshot('form-ui/reference-src', { overwrite: true, scrollBehavior: 'center' });
      } else {
        cy.screenshot('form-ui/reference-src', { capture: 'fullPage', overwrite: true });
      }
      cy.task('getLastScreenshot').then((p) => { referencePath = p; });
    }

    // (b) Capture the form under test — CROPPED TO THE FORM REGION by default.
    //     Gate on 'exist', never 'be.visible': a position:fixed theme ancestor can make Cypress
    //     deem the container "not visible" and fail the run before any capture happens.
    cy.visit(formUrl);
    cy.get(formSel, { timeout: 30000 }).should('exist');
    cy.wait(1500); // let theme CSS + webfonts settle so the diff is stable
    if (scope === 'form') {
      // Element screenshot — excludes the page's header/footer/nav chrome (on the
      // "Test Adaptive Form" page those are experience fragments) so a form-only
      // reference can actually match. This is the DEFAULT and is what makes parity meaningful.
      cy.get(formSel).screenshot('form-ui/actual-src', { overwrite: true, scrollBehavior: 'center' });
    } else {
      cy.screenshot('form-ui/actual-src', { capture: 'fullPage', overwrite: true });
    }

    // (c) Pixel diff.
    cy.task('getLastScreenshot').then((actualPath) => {
      expect(referencePath, 'a reference (image path or captured URL) is required').to.be.a('string');
      cy.task('pixelDiff', {
        actualPath, referencePath, diffPath: `${OUT}/diff.png`, threshold: 0.1,
      }).then((res) => {
        cy.writeFile(`${OUT}/pixel-result.json`, res);
        cy.log(`pixel mismatch ${res.mismatchPercentage.toFixed(2)}% (gate < ${maxMismatch}%)`);
        // The pixel result is the GATE. Do not soft-pass — let it fail the build on regression.
        expect(res.mismatchPercentage, 'pixel mismatch %').to.be.lessThan(maxMismatch);
      });
    });
  });
});
```

> The spec writes `cypress/results/form-ui/{reference.png, actual.png, diff.png, pixel-result.json}`.
> `reference.png`/`actual.png` are normalised to identical dimensions — those are the three images
> the report and the vision pass use.

---

## Step 3 — Run the capture + pixel diff

Pre-checks (fast, before launching the browser):
- AEM reachable at the relevant host (`{formUrl}`'s origin). If not, stop and tell the user to start
  the SDK — a capture against a dead instance just times out.
- The form actually renders: `curl -u admin:admin -s -o /dev/null -w "%{http_code}" {formUrl}` → 200.
  A 404 here means the form isn't deployed; route the user to `create-adaptive-form`/`migrate-form`.

Run the single spec, passing inputs as env vars. **The spec reads them via `Cypress.env('FORM_URL')`
etc., and Cypress ONLY auto-maps env vars whose name is prefixed with `CYPRESS_`** (or values passed
via `--env`). A plain `FORM_URL` / `$env:FORM_URL` is **NOT** seen by `Cypress.env()`, so `formUrl`
comes back `undefined` and the `before` hook throws `Cannot read properties of undefined (reading
'includes')` with "No commands were issued in this test". **Always prefix with `CYPRESS_`.**

```powershell
# PowerShell — note the CYPRESS_ prefix on every var
cd ui.tests/test-module
$env:CYPRESS_FORM_URL         = "http://localhost:4502/content/forms/af/{project}/{formName}.html"
$env:CYPRESS_REFERENCE_IMAGE  = "C:/path/to/reference.png"   # forward slashes; OR set CYPRESS_REFERENCE_URL
$env:CYPRESS_CAPTURE_SCOPE    = "form"                       # DEFAULT — crop to the embedded form
$env:CYPRESS_FORM_SELECTOR    = ".cmp-adaptiveform-container"
$env:CYPRESS_VIEWPORT         = "1280x1024"
$env:CYPRESS_MAX_MISMATCH_PCT = "1.0"
$env:REPORTS_PATH             = "cypress/results"            # read by the reporter (process.env), NOT the spec — no prefix

# Cloud target — publish (preferred, anonymous):
#   $env:CYPRESS_FORM_URL = "https://publish-p{prog}-e{env}.adobeaemcloud.com/content/{project}/us/en/test-adaptive-form.html"
# Cloud author — Bearer token preferred, IMS credentials as fallback:
#   $env:CYPRESS_FORM_URL            = "https://author-p{prog}-e{env}.adobeaemcloud.com/content/{project}/us/en/test-adaptive-form.html"
#   $env:CYPRESS_AEM_DEV_ACCESS_TOKEN = $env:AEM_DEV_ACCESS_TOKEN
#   $env:CYPRESS_AEM_IMS_USERNAME     = "user@example.com"   # only if no token
#   $env:CYPRESS_AEM_IMS_PASSWORD     = "..."                 # only if no token
npx cypress run --browser chrome --spec "cypress/e2e/form-ui-compare.cy.js"
```
```bash
# bash equivalent — inline KEY=val prefixes, CYPRESS_ on the spec vars
CYPRESS_FORM_URL="http://localhost:4502/content/forms/af/{project}/{formName}.html" \
CYPRESS_REFERENCE_IMAGE="C:/path/to/reference.png" \
CYPRESS_CAPTURE_SCOPE="form" CYPRESS_FORM_SELECTOR=".cmp-adaptiveform-container" \
CYPRESS_VIEWPORT="1280x1024" CYPRESS_MAX_MISMATCH_PCT="1.0" REPORTS_PATH="cypress/results" \
npx cypress run --browser chrome --spec "cypress/e2e/form-ui-compare.cy.js"
```

> **Reference path:** use **forward slashes** even on Windows (`C:/Users/.../ref.png`) — a Windows
> path with a space (`Customer Feedback form.png`) works fine when quoted, but backslashes can be
> eaten by the shell before Cypress sees them.
> **Hydration gate = `exist`, not `be.visible`.** When gating the screenshot on the Submit button
> being hydrated, assert `.should('exist')` — NOT `.should('be.visible')`. A theme can put a
> `position:fixed` ancestor over the button so Cypress deems it "not visible" and fails the run
> before any capture, even though `capture:'fullPage'` would have screenshotted it fine. Gate on
> presence in the DOM; let fullPage handle off-screen/overlaid elements.

- Use `--browser chrome` for consistent rendering; `electron` (the module default) also works.
- **First run on a machine may fail with "No version of Cypress is installed"** — run
  `npx cypress install` once (downloads the pinned binary to the user cache), then re-run.
- The pixel gate makes the run **exit non-zero on mismatch** — that is the hard pass/fail signal.
- Cypress records a video and (on failure) extra screenshots under `REPORTS_PATH`. A JUnit
  `output.xml` is emitted via the module's existing reporter — useful in CI / Cloud Manager.

> If the run fails on the assertion (mismatch ≥ threshold), that is a real FAIL, not a tooling
> error — continue to Step 4 and explain *why* in the report. Only re-run for genuine flakiness
> (e.g. a webfont hadn't loaded — bump the `cy.wait`).

---

## Step 4 — Vision comparison (Claude reads the three images)

This is the half that pixel diff can't do: *what* differs and whether it matters. After the run,
**Read** the three normalised images and analyse them:

- `ui.tests/test-module/cypress/results/form-ui/reference.png`
- `ui.tests/test-module/cypress/results/form-ui/actual.png`
- `ui.tests/test-module/cypress/results/form-ui/diff.png`

Compare them on these axes and record every difference with a severity:

| Severity | What qualifies |
|---|---|
| **Critical** | A field/section present in the reference is **missing** in the form (or vice-versa); an **extra field NOT in the reference**; a **wrong label** on a field; **wrong field order**; a control of the **wrong type** (radio vs dropdown); broken/overlapping layout; submit button absent; **a field's width/column does not match the reference** (e.g. reference shows two half-width fields on one row but the form stacks them, or a field overflows its column); **validation error icons/messages shown at INITIAL load** before any user interaction |
| **Major** | Off-brand primary colour, wrong font family, materially wrong spacing/alignment, wrong column count (1-col vs 2-col), title/heading text mismatch |
| **Minor** | Small padding/shade differences, antialiasing, minor icon/placeholder wording, anything cosmetic |

### When a reference SCREENSHOT exists: match it EXACTLY (strict UI parity)

A reference screenshot is the source of truth for the rendered UI — not just field presence. When
`{referenceImage}` is supplied (i.e. NOT a low-fidelity migration-parity run), verify the form
**exactly matches the screenshot** and treat each of these as **Critical** if it diverges:

1. **No extra fields** — the form must contain ONLY the fields the screenshot shows, in the same
   sections. An extra field that isn't in the screenshot is a Critical defect (cross-check the
   field count against `guideContainer.model.json`). A missing field is equally Critical.
2. **Field width / column layout matches** — if the screenshot shows two fields side-by-side on one
   row (half width), the form must too; a field that stacks when it should pair, or overflows /
   is wider than its column (classic: native `<input type="date">`), is Critical. Compare the
   per-row field grouping against the screenshot row by row.
3. **No error state at initial load** — capture the form **clean, before any interaction** (do not
   click Submit or focus/blur fields before the screenshot). There must be **no red error icons,
   no inline error messages, and no error-highlight borders** on any field at first render. If the
   actual capture shows error icons/messages that the reference does not, that is a **Critical**
   finding — it means validation is firing prematurely (the fix belongs in `create-form-clientlib`:
   validate on blur/submit, never on load; do not call validation routines during init).
4. **Styles match** — field/input styling, badges, button style, borders, and the prefilled/readonly
   treatment should match the screenshot's intent. Clear style mismatches are Major (or Critical if
   they change which control the user perceives).
5. **Every word from the screenshot is present AND visible** — the form **title/heading** (e.g.
   "Health Insurance Application"), subtitle/description, every section header, legend, help text,
   and button label that appears in the screenshot must be visibly rendered in the captured form.
   A title or heading that is in the content but **not visible** in the capture (theme not loaded /
   title unstyled / `display:none`) is **Critical** — do NOT pass it because the text exists in the
   model; it must be *seen* in the rendered pixels. Send title/heading/static-text gaps back to
   `create-form-theme` (visibility/styling) or `create-adaptive-form` (missing content).
6. **Icons and section-header colours match** — every icon the screenshot shows (header/branding,
   per-section-header icons, in-tile upload/choice-card icons) must render, and section-header /
   primary / accent **colours must match the screenshot's actual hex** (sample it; "close" is a
   fail). A missing icon or off-reference section colour is **Critical** when it's a defined brand
   element of the reference, otherwise Major. Send these back to `create-form-theme` /
   `create-form-clientlib`.
6a. **Every visual element renders, not only fields** — verify each non-field visual the screenshot
   shows is present: **section separator / divider rules** (the thin horizontal line under each
   numbered section and under the title/subtitle header), background bands, images, and logos. A
   separator/divider the reference shows but the form omits is a **Major** finding (Critical when it's
   a defined structural element). Route the fix to `create-form-clientlib` (draw it as a
   `border-bottom` on the real served section-panel classes, served fresh) — confirm by pixels, not by
   CSS presence.
7. **Required-field asterisks actually RENDER (confirm by pixels, never by CSS presence)** — if the
   reference marks required fields with a red `*`, verify the `*` is **visibly rendered** after each
   required label in the actual capture, and that OPTIONAL fields show none. Do NOT pass this on the
   grounds that a `__label__qualifier` (or any) asterisk rule exists in the theme CSS — Core
   Components does not emit that span, and the aggregated `.theme/_default/theme.css` is a cache that
   lags the redeployed theme.zip, so the rule can be "present" yet the `*` never appears. A missing
   required marker is a **Major** finding; route the fix to `create-form-clientlib` (authoritative,
   served-fresh) using `[data-cmp-required="true"] .cmp-adaptiveform-*__label::after { content:" *"; color:#e02020 }`.

Report each as a row in the Findings table with the exact field and the screenshot-vs-actual delta.
PASS requires **zero Critical findings**, so any extra field, width/stacking mismatch, or
load-time error icon fails the verdict and must be sent back to the build skill to fix.

> **Submit is gated on validation (verify functionally alongside the visual check).** Submission and
> any PDF / Document-of-Record generation must fire ONLY when ALL validation passes. Clicking Submit
> on an empty/invalid form must BLOCK submission, show inline errors, focus the first invalid field,
> and produce **NO PDF**; a valid form submits AND generates the PDF. If Submit on an invalid form
> proceeds (thank-you shown, or a PDF downloaded), that is a **Critical** finding — route the fix to
> `create-submit-action` / `create-form-clientlib` (gate on form validity; use the native validating
> submit, never a raw onClick that posts to the GeneratePDF servlet unconditionally).

Cross-check the visual inventory against the form's runtime model so labels/fields are read from
data, not guessed from pixels:

```bash
curl -u admin:admin -s \
  "http://localhost:4502/content/forms/af/{project}/{formName}/jcr:content/guideContainer.model.json" \
  | grep -oE '"(label|name|fieldType)":"[^"]*"'
```

Use that to confirm e.g. a label the reference shows as "E-mail" really renders as "Email Address".

---

## Step 5 — Write the report (the deliverable)

Cypress produced the working images under `ui.tests/test-module/cypress/results/form-ui/` — those are
intermediate working files and stay there. The **deliverable report goes in the run directory's
`test/sentinel/` run folder** (AGENTS.md → "Run output convention"): write ONLY
`.claude/agents/runs/{YYYY-MM-DD}-{formName}/test/sentinel/test-form-ui-report.md`. **Do NOT copy
`reference.png` / `actual.png` / `diff.png` (or any image) into `test/sentinel/`** — the report stays
text-only and points to the images in the Cypress results dir. (Create the run directory + `test/sentinel/`
subfolder if they don't exist — e.g. when this skill is invoked directly rather than via the program
agent.) Template:

```markdown
# Form UI Comparison Report — {formTitle}

| | |
|---|---|
| Form URL | {formUrl} |
| Reference | {referenceImage or referenceUrl} ({"screenshot" | "url"}) |
| Viewport | {WxH} |
| Auth path | {author-login | public} |
| Run date | {date} |

## Verdict: **PASS** | **FAIL**

- **Pixel diff:** {X.XX}% mismatch (= {100−X.XX}% match) vs {maxMismatchPct}% threshold → {PASS|FAIL}
- **Semantic findings:** {n} Critical · {m} Major · {k} Minor
- Overall = **FAIL** if pixel gate fails **or** any Critical finding exists; else PASS
  (note when the pixel % is unreliable due to a mismatched-aspect reference — defer to findings).

> 🎯 **REPLICA-PARITY GATE (mandatory for a screenshot / URL replica delivery — the default mode).**
> The delivered form must reach **≥ 90% pixel match (≤ 10% mismatch)** AND have **zero Critical
> findings**. So for a replica run use `{maxMismatchPct} = 10`. If the pixel match is **below 90%**,
> that is a **FAIL** even with zero Critical findings — the layout/branding is not yet a faithful
> replica; route the deltas back to the build skills (`create-form-clientlib` for grid/layout,
> `create-form-theme` + the clientlib brand-token mirror for colour) and re-test after redeploy. Only
> when a genuinely mismatched-aspect reference inflates the raw % (see the viewport note) may you
> defer to the vision findings — and you must say so explicitly in the verdict line.
> The **migration-parity** mode below is the ONE exception where the pixel gate is advisory.

## Images
Not stored in the run directory. See the Cypress working dir:
`ui.tests/test-module/cypress/results/form-ui/` → `reference.png`, `actual.png`, `diff.png`.

## Findings
| # | Severity | Type | Reference shows | Form renders | Recommendation |
|---|---|---|---|---|---|
| 1 | Critical | Missing field | "Date of Birth" | (absent) | Add `datepicker` field `DateOfBirth` |
| 2 | Major | Label | "E-mail" | "Email Address" | Align label to design or confirm copy |
| … | | | | | |

## Field inventory (from guideContainer.model.json)
- Reference fields: …
- Form fields: …
- Missing in form: … · Extra in form: …

## Recommendation
{One paragraph: ship / fix-then-ship, and the top 1–3 actions. Where fixes touch form XML, name
the skill to use — create-adaptive-form / create-form-rules / create-form-theme.}
```

Report the verdict and the report file path back to the user in chat (with the headline numbers),
and that the images are alongside it. **Do not** declare PASS if the Cypress run failed the gate or
any Critical finding stands.

---

## Failure symptoms and causes

| Symptom | Cause | Fix |
|---|---|---|
| `cy.task('pixelDiff')` errors "Image dimensions do not match" | reference not normalised | ensure the `sharp` resize step ran (it normalises ref → actual size); reinstall `sharp` if its native binary failed |
| Mismatch % huge but images look similar | viewport/aspect mismatch, or full-page vs above-the-fold | fix `{viewport}`; keep the same `{captureScope}` on both sides; if reference is a partial mockup, trust the vision findings |
| Mismatch % huge and the actual capture shows site header/footer/nav the reference doesn't | `{captureScope}` was `page`, so the whole Sites page was captured instead of the embedded form | set `{captureScope}=form` (the default) so the capture is cropped to `{formSelector}`. Comparing a form-only reference against a full page **always** fails on chrome alone |
| Element screenshot is clipped or blank | the form container is taller than the viewport, or lazy content hadn't rendered | raise `{viewport}` height, keep `scrollBehavior:'center'`, and increase the `cy.wait` after the container exists |
| Form capture is the AEM login page | author URL without login | the spec's `AEMLogin` branch didn't run — confirm the URL contains `:4502`/`author`, or pre-authenticate |
| `.cmp-adaptiveform-container` never visible (timeout) | form 404, wrong path, or unstyled (theme/conf broken) | verify `{formUrl}` returns 200; if 200 but unstyled, the form has a theme/conf bug — fix via create-adaptive-form |
| Run passes locally, flaky in CI | webfonts/theme CSS not settled before screenshot | increase the `cy.wait` after container-visible, or wait on a specific webfont/`document.fonts.ready` |
| No JUnit `output.xml` | `REPORTS_PATH` not set | export `REPORTS_PATH=cypress/results` before the run |

---

## Pre-flight checklist

- [ ] `{formName}` and `{formUrl}` confirmed; URL returns HTTP 200 (form is deployed)
- [ ] Exactly ONE reference provided — `{referenceImage}` (file), `{referenceUrl}` (captured), or
      `{referenceXdp}` (rasterised to PNG first). **A run with no reference is not a valid UI test** —
      stop and ask for one rather than substituting functional checks
- [ ] `{captureScope}` is `form` (the default) so the capture is cropped to `{formSelector}` and the
      Sites page's header/footer/nav chrome is excluded from the diff; both the reference capture and
      the actual capture used the **same** scope
- [ ] Target tier recorded: local SDK / cloud publish / cloud author (+ which auth path)
- [ ] Cypress module setup present (Step 2): `pixelmatch`/`pngjs`/`sharp` installed, `pixelDiff` +
      `getLastScreenshot` tasks registered, `after:screenshot` hook added, `form-ui-compare.cy.js` created
- [ ] Run executed at a fixed `{viewport}`; `reference.png`, `actual.png`, `diff.png`,
      `pixel-result.json` all produced under `cypress/results/form-ui/`
- [ ] Pixel gate evaluated against `{maxMismatchPct}` (build exit code respected — not soft-passed)
- [ ] Vision pass done: every difference logged with a severity; labels/fields cross-checked against
      `guideContainer.model.json` (not guessed from pixels)
- [ ] Report written to `.claude/agents/runs/{YYYY-MM-DD}-{formName}/test/sentinel/test-form-ui-report.md`
      (text-only — NO PNGs copied into `test/sentinel/`; images stay in the Cypress results dir), with
      Verdict, both result types, findings table, inventory, and a recommendation
- [ ] Verdict reported in chat with the report path; PASS only if the gate passed AND no Critical findings

---

## When to hand off

- Differences are real form defects → fix via `create-adaptive-form` (fields/layout/labels),
  `create-form-rules` (behaviour), or `create-form-theme` (colour/typography/spacing), then
  re-run this skill to confirm the report goes green.
- The reference itself is the new spec and the form should match it wholesale → treat as a small
  re-author with `create-adaptive-form`, then re-test.
