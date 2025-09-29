# Internationalization Plan

## Default Locale Behavior
- The application defaults to English (`en`) until a localized bundle is detected for the user's preferred language.
- Browser/OS locale detection occurs on first load; if a supported bundle is found it is applied immediately, otherwise the interface falls back to `en`.
- Persist the user's manual locale selection in local storage so returning users continue to see their chosen language even if detection differs.
- Translation fallbacks cascade: `requested locale -> language-only locale (e.g., pt-BR -> pt) -> en`.

## Language Toggle Placement
- Surface a language toggle in the global navigation header so it is always visible.
- On narrow viewports collapse the toggle into the primary menu to avoid crowding while keeping it accessible.
- Mirror the toggle in the footer to provide a secondary access point and reinforce discoverability.
- Include the toggle in onboarding and settings flows to highlight localization support during key touchpoints.

## String Extraction Strategy
- Centralize all user-facing copy in resource bundles stored under `locales/`.
- Replace hard-coded strings in components with keyed lookups (e.g., `t('diagnostics.intro.title')`).
- Adopt ICU MessageFormat for pluralization and variable interpolation to future-proof translations.
- Enforce extraction via linting or code review checklist items to ensure new strings are captured.
- Document string ownership: product content owners author English copy, localization leads manage translation requests.

## Resource Bundle Structure
- Store locale bundles as JSON: `locales/en.json`, `locales/id.json`, etc.
- Organize keys by feature area for clarity:
  - `diagnostics.*` – diagnostic question prompts, answer labels, and explanatory tooltips.
  - `rationale.*` – explanatory rationale text shown after diagnostic selections.
  - `roadmap.*` – roadmap step titles, descriptions, and call-to-action microcopy.
  - `navigation.*`, `settings.*`, `common.*` – shared UI chrome and reusable phrases.
- Maintain a `locales/meta.json` manifest listing supported locales, translators, and last updated timestamps for governance.

## Translation Scope
- Diagnose module: question prompts, answer options, follow-up clarifications, and any inline helper text.
- Rationale module: rationale paragraphs, evidence snippets, and recommended practice summaries.
- Roadmap module: step names, detailed descriptions, milestone labels, success criteria, and CTA buttons.
- Support copy: onboarding instructions, notifications, error states, and accessibility descriptions tied to the above modules.

## QA Process for Translation Accuracy & Currency
1. **Pre-translation preparation**
   - Provide translators with screenshots, domain glossary, and style guide to reduce ambiguity.
   - Flag strings with contextual notes in the resource files using adjacent comment metadata (e.g., `// @description`).
2. **Initial translation review**
   - Utilize a translation management system (TMS) workflow requiring reviewer sign-off separate from the translator.
   - Run automated linting to detect placeholder mismatches, HTML tag balance, and ICU syntax errors.
3. **In-product verification**
   - Spin up staging builds per locale with feature flags matching production rule sets.
   - Conduct spot checks using bilingual QA specialists to confirm terminology and layout integrity.
   - Capture screenshots for each major screen and archive alongside translation batches for regression tracking.
4. **Rule Set Evolution Monitoring**
   - When diagnostic rules or roadmap logic changes, automatically flag impacted keys via commit hooks that tag modified files.
   - Maintain a changelog mapping rule updates to copy keys; require localization review before release.
   - Schedule quarterly audits comparing English source strings against localized bundles to detect drift.
5. **Post-release feedback loop**
   - Instrument locale-specific analytics and feedback forms to capture user-reported translation issues.
   - Triaged issues feed back into the TMS with severity tags; high-severity items trigger expedited translation hotfixes.
6. **Currency Maintenance**
   - Establish SLAs: e.g., critical copy updates localized within 3 business days, minor within 10.
   - Track bundle freshness via the `meta.json` manifest and alert when timestamps exceed SLA thresholds.

