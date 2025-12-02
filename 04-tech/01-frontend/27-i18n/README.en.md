# i18n Internationalization Best Practices

## Role Definition

You are an expert in frontend internationalization, skilled in multi-language support, date-time formatting, currency number localization, and RTL layout design. You are responsible for formulating internationalization solutions, translation management strategies, and localization best practices.

## Core Principles (NON-NEGOTIABLE)

| Principle | Description | Consequences of Violation |
|------|------|----------|
| Unified Translation Key Management | All text MUST use translation keys, hard-coded text content is prohibited | Leads to missing translations, maintenance difficulties, inability to switch languages |
| Namespace Isolation | Divide translation namespaces by functional modules (common/user/order, etc.) | Bloated translation files, key conflicts, loading performance degradation |
| Interpolation Parameter Typing | Dynamic parameters use explicit placeholder format (double curly braces) | Translation content cannot be dynamically replaced, display abnormalities |
| Plural Rules Compliance | Implement correct plural forms according to language characteristics (English/Chinese/Arabic rules differ) | Grammatical errors, poor user experience, appears unprofessional |
| Date Format Localization | Use date-fns or Intl API instead of hard-coded formats | Date format confusion, timezone errors, user confusion |
| Fallback Language Mechanism | Configure fallbackLng to ensure default display when translations missing | Shows translation key names, broken interface, user cannot understand |
| RTL Layout Adaptation | Arabic/Hebrew, etc. MUST support right-to-left layout | Layout confusion, incorrect text alignment, unusable |
| Structured Translation Files | Use nested objects to organize translation keys, avoid flat structure | Key conflicts, hard to maintain, logical confusion |

## Prompt Templates

### Scenario 1: Initialize Internationalization Configuration

```
Please help me configure internationalization for the project:

【Basic Information】
- Frontend Framework: [React/Vue/Next.js/Nuxt]
- Supported Languages: [en, zh, ja, ko, etc.]
- Default Language: [en]
- Fallback Language: [en]

【Loading Strategy】
- Translation File Source: [local static files/CDN remote loading/API dynamic fetching]
- On-demand Loading Strategy: [whether to enable namespace lazy loading]
- Cache Location: [localStorage/sessionStorage/memory]

【Detection Mechanism】
- Language Detection Order: [URL parameters > localStorage > browser language > default language]
- Allow Auto-detection: [yes/no]

【Special Features】
- Plural Form Support: [needed/not needed]
- Date-time Formatting: [need date-fns/use Intl API]
- Currency Number Formatting: [needed/not needed]
- RTL Layout Support: [need to support Arabic/Hebrew]

Please provide configuration step instructions and file organization structure recommendations.
```

### Scenario 2: Translation File Management

```
Please help me design the organization structure for translation files:

【Project Scale】
- Application Type: [e-commerce platform/admin dashboard/marketing website/mobile app]
- Estimated Page Count: [approximately XX pages]
- Estimated Translation Entries: [estimated XX entries]

【Module Division】
- Main Functional Modules: [homepage/products/orders/user center/settings]
- Shared Text Types: [navigation menu/button text/form validation/error messages/status prompts]

【File Organization】
- Namespace Division Plan: [by page/by function/hybrid approach]
- Nesting Level Suggestions: [suggest maximum how many levels]
- Key Naming Convention: [camelCase/underscore/dot-separated]

Please provide translation file structure design plan and key naming organization examples.
```

### Scenario 3: Date-time Formatting

```
Please help me implement multi-language date-time formatting:

【Requirement Scenarios】
- Display Full Date: [December 1, 2024 / 2024年12月1日]
- Display Short Date: [12/1/2024 / 2024/12/1]
- Relative Time: [3 minutes ago / 3分钟前]
- Timestamp Conversion: [need to display specific time]

【Technical Selection】
- Preferred Solution: [date-fns + language packs / Intl.DateTimeFormat]
- Timezone Handling: [need/don't need timezone conversion]
- Performance Requirements: [whether need to cache formatting results]

【Language Support】
- Target Languages: [en, zh, ja, ko, etc.]

Please provide date-time formatting implementation solution and usage instructions.
```

### Scenario 4: Language Switcher Function

```
Please help me implement a language switcher component:

【UI Display】
- Display Form: [dropdown menu/button group/icon+text]
- Display Content: [full language name/abbreviation/flag icon]
- Current Language Indicator: [highlight/selected state]

【Interaction Logic】
- Post-switch Behavior: [refresh page/no-refresh switching]
- Persistent Storage: [localStorage/cookie]
- URL Synchronization: [whether need to reflect language in URL]

【Accessibility】
- Keyboard Operation Support: [yes/no]
- Screen Reader Friendly: [need aria labels]

Please provide language switcher implementation approach and considerations.
```

## Decision Guide

### Internationalization Library Selection Decision Tree

```
Select internationalization solution based on project framework
│
├─ React Project
│  │
│  ├─ Using Next.js Framework
│  │  └─ Recommended: next-i18next
│  │     - Reason: native SSR/SSG support, automatic route internationalization, SEO-friendly
│  │     - Configuration Path: configure i18n options in next.config.js
│  │     - Page-level Loading: load translations via getStaticProps/getServerSideProps
│  │
│  └─ Using CRA or Vite
│     └─ Recommended: react-i18next
│        - Reason: hooks-friendly, mature ecosystem, supports lazy loading
│        - Initialization Location: initialize i18n instance in entry file
│        - Usage: useTranslation hook to get t function
│
├─ Vue Project
│  │
│  ├─ Using Nuxt Framework
│  │  └─ Recommended: @nuxtjs/i18n
│  │     - Reason: Nuxt modular integration, automatic route prefix, SEO optimization
│  │     - Configuration Location: configure i18n module in nuxt.config.ts
│  │     - Component Usage: $t method or useI18n composable
│  │
│  └─ Using Vue CLI or Vite
│     └─ Recommended: vue-i18n
│        - Reason: Vue3 Composition API support, type safety
│        - Installation Version: Vue3 uses v9.x, Vue2 uses v8.x
│        - Usage: useI18n composable or global $t method
│
└─ Native JavaScript Project
   └─ Recommended: i18next (core library)
      - Reason: framework-agnostic, feature-complete, rich plugins
      - Browser-side: pair with i18next-browser-languagedetector
      - Backend Loading: pair with i18next-http-backend
```

### Translation File Loading Strategy Decision Tree

```
Select loading method based on application scenario
│
├─ Total translation content < 100KB
│  └─ Approach: bundle entirely into application code
│     - Advantages: no network requests, fast loading, offline available
│     - Disadvantages: increases bundle size, updates require redeployment
│     - Suitable: small applications, stable translation content, pursuing extreme performance
│
├─ Total translation content 100KB - 500KB
│  └─ Approach: lazy load by namespace
│     - Advantages: fast first screen load, on-demand loading reduces traffic
│     - Disadvantages: short loading delay when switching languages or pages
│     - Suitable: medium applications, clear module division, long user stay time
│
└─ Total translation content > 500KB or frequent updates
   └─ Approach: CDN remote loading + caching
      - Advantages: translations can update independently, don't affect main app, reduce bundle size
      - Disadvantages: depends on network, needs cache strategy, slow first load
      - Suitable: large applications, frequent translation adjustments, multi-team collaboration
      - Cache Strategy: localStorage persistence + version number verification mechanism
```

### Plural Rules Handling Decision

```
Select plural strategy based on target languages
│
├─ Only support Chinese/Japanese/Korean
│  └─ Simplified Handling: no plural rules needed
│     - Reason: these languages have no singular/plural changes
│     - Implementation: use unified translation key
│     - Example: "1 item" and "5 items" use same template
│
├─ Include English/French/German
│  └─ Standard Plural: implement singular/plural two forms
│     - Rule: count=1 use singular, count≠1 use plural
│     - Key Convention: key and key_plural
│     - Example: "1 item" vs "5 items"
│
└─ Include Arabic/Russian/Polish
   └─ Complex Plural: implement multiple plural forms
      - Rule: according to language-specific rules (0/1/2-4/5+)
      - Library Support: use i18next's pluralResolver
      - Example: Arabic has 6 plural forms
```

## Positive and Negative Examples

### Translation Key Usage Comparison

| ❌ Wrong Approach | ✅ Correct Approach | Explanation |
|------------|------------|------|
| Hard-coded Chinese: `<h1>欢迎使用</h1>` | Use translation key: `<h1>{t('app.welcome')}</h1>` | Hard-coded cannot switch languages |
| Mixed usage: `<button>Submit</button>` and `<button>{t('cancel')}</button>` | Unified usage: all text through translation keys | Mixed usage causes maintenance confusion |
| Translation key with language: `t('welcome_zh')` | Language-agnostic key: `t('welcome')` + auto language switching | Translation keys shouldn't include language identifiers |
| English as key name: `t('Welcome to our app')` | Semantic key name: `t('app.welcome_message')` | Long text as key name hard to maintain |

### Namespace Organization Comparison

| ❌ Wrong Approach | ✅ Correct Approach | Explanation |
|------------|------------|------|
| Single file for all translations: `common.json` contains everything | Split by module: `common.json`, `user.json`, `order.json` | Single file too large causes slow loading |
| Over-splitting: each page one file (50+ files) | Reasonable granularity: split by functional domain (5-15 files) | Over-splitting increases management cost |
| No namespaces: global flat all keys | Clear spaces: `useTranslation('user')` specify namespace | Key conflicts easily |

### Interpolation Parameter Comparison

| ❌ Wrong Approach | ✅ Correct Approach | Explanation |
|------------|------------|------|
| String concatenation: `'欢迎' + userName + '!'` | Interpolation expression: `t('welcome', { name: userName })` | Concatenation cannot adapt to different language word orders |
| Single curly brace: `{name}` | Double curly braces: `{{name}}` | i18next specification requires double curly braces |
| Unescaped HTML: directly insert user input content | Safe handling: use `interpolation.escapeValue` | Prevent XSS attacks |

### Date Formatting Comparison

| ❌ Wrong Approach | ✅ Correct Approach | Explanation |
|------------|------------|------|
| Hard-coded format: `2024-12-01` fixed format | Localized format: `formatDate(date)` auto-adapts to language | Different regions have different date format habits |
| String concatenation: `year + '年' + month + '月'` | date-fns: use language pack for auto-formatting | Concatenation cannot support multi-language |
| Ignore timezone: directly display server time | Timezone conversion: convert to user's local time | Timezone errors cause user confusion |

### Plural Handling Comparison

| ❌ Wrong Approach | ✅ Correct Approach | Explanation |
|------------|------------|------|
| Conditional judgment: `count === 1 ? 'item' : 'items'` | Plural key: `t('items', { count })` auto-selects | Hard-coded logic cannot adapt to other languages |
| Only English plural: only handle English singular/plural changes | Multi-language plural: configure plural rules for each language | Other language plural rules ignored |

### RTL Layout Comparison

| ❌ Wrong Approach | ✅ Correct Approach | Explanation |
|------------|------------|------|
| Fixed left-align: `text-align: left` | Direction-aware: `text-align: start` or use `dir` attribute | RTL language text alignment wrong |
| Hard-coded spacing: `margin-left: 16px` | Logical spacing: `margin-inline-start: 16px` | Spacing doesn't reverse when direction switches |
| Ignore direction: don't set `dir` attribute | Root direction: `<html dir="rtl">` or `<div dir="rtl">` | Browser cannot correctly render RTL layout |

### Type Safety Comparison

| ❌ Wrong Approach | ✅ Correct Approach | Explanation |
|------------|------------|------|
| String keys: `t('unknown.key')` no type checking | Type definition: declare translation resource type for auto-completion | Spelling errors only found at runtime |
| Dynamic keys: `t(dynamicKey)` cannot predict | Static keys: use explicit translation key strings | Dynamic keys cannot be statically analyzed |

## Validation Checklist

### Configuration Completeness

- [ ] Configured fallbackLng fallback language (usually en)
- [ ] Configured supportedLngs supported language list
- [ ] Configured defaultNS default namespace (usually common)
- [ ] Configured ns array including all namespaces
- [ ] Configured backend loading path (if using remote loading)
- [ ] Configured detection language detection order and cache strategy
- [ ] Configured interpolation options (escapeValue, etc.)
- [ ] Enabled debug mode in development environment for troubleshooting

### Translation File Organization

- [ ] Translation files organized by language code (en/zh/ja directories)
- [ ] Translation files split by namespace (common/user/order, etc.)
- [ ] All language translation file structures consistent (same key names)
- [ ] Use nested objects to organize translation keys (avoid over-flattening)
- [ ] Shared text concentrated in common namespace
- [ ] Plural forms use _plural suffix or count parameter
- [ ] Interpolation parameters use double curly brace format `{{param}}`
- [ ] Translation content has no hard-coded language-specific symbols

### Functionality Implementation

- [ ] All visible text obtained through translation keys (no hard-coding)
- [ ] Interface completely updates after language switch (no remaining old language text)
- [ ] Language switch persistently stored (localStorage/cookie)
- [ ] Shows fallback content instead of key name when translation key missing
- [ ] Date-time uses localized formatting (date-fns or Intl)
- [ ] Numbers and currency use localized formatting (Intl.NumberFormat)
- [ ] Plural forms auto-select based on count parameter
- [ ] Interpolation parameters correctly passed and replaced

### RTL Support (if needed)

- [ ] Root element sets `dir` attribute (rtl/ltr)
- [ ] Text alignment uses `start/end` instead of `left/right`
- [ ] Spacing uses logical properties (margin-inline-start, etc.)
- [ ] Flexbox layout considers direction reversal
- [ ] Icons and symbols adapt to RTL direction
- [ ] Input box cursor position correct
- [ ] Scrollbar position adapts (some browsers)

### Performance Optimization

- [ ] Large applications enable namespace lazy loading
- [ ] Translation files use CDN acceleration (if applicable)
- [ ] Translation content enables cache mechanism
- [ ] Date formatting results cached (avoid repeated calculations)
- [ ] Avoid full re-rendering when switching languages
- [ ] Production environment disables debug mode

### Type Safety (TypeScript projects)

- [ ] Declare translation resource type definitions
- [ ] useTranslation gets key name auto-completion
- [ ] Interpolation parameter type checking
- [ ] Namespace type safety

### Test Coverage

- [ ] Every supported language manually tested
- [ ] Language switching functionality tested successfully
- [ ] Fallback strategy tested for missing translation keys
- [ ] Plural forms tested with various counts (0/1/2/multiple)
- [ ] RTL layout visually tested (if applicable)
- [ ] Date-time displays correctly in different languages

## Guardrail Constraints

### Translation Key Naming Rules

**MUST Follow**
- Use lowercase letters and underscores or dot separators (user_profile or user.profile)
- Use semantic naming instead of display text (use submit_button not submit)
- Namespace prefix clear (common.actions.save not save)
- Avoid abbreviations unless industry-standard (btn → button)

**Prohibited Behaviors**
- MUST NOT use complete sentences as key names (prohibited: "Welcome to our application")
- MUST NOT include language identifiers in key names (prohibited: welcome_zh, welcome_en)
- MUST NOT use special characters (prohibited: @ # $ %, etc.)
- MUST NOT start with numbers (prohibited: 1st_step)

### Translation Content Constraints

**MUST Follow**
- Interpolation parameters wrapped in double curly braces `{{paramName}}`
- Plural forms provide complete variants (singular/plural or more)
- HTML content MUST be marked as unescaped (use after configuring escapeValue: false)
- Translation content keep concise avoid overly long text (suggest single entry < 200 characters)

**Prohibited Behaviors**
- MUST NOT hard-code date formats in translation text (prohibited: "2024-12-01")
- MUST NOT include business logic in translations (prohibited: "You have {{count}} messages" with processing logic)
- MUST NOT mix multiple languages (prohibited: "Welcome 欢迎")
- MUST NOT include unescaped user input (prevent XSS)

### File Organization Constraints

**MUST Follow**
- All language file structures MUST be completely consistent
- New translation keys MUST sync to all language files
- Namespace file sizes recommend controlling within 50KB
- Use JSON format ensure parseability

**Prohibited Behaviors**
- MUST NOT have inconsistent language file structures (A language has keys B language doesn't)
- MUST NOT mix multiple namespaces in single file
- MUST NOT use overly deep nesting (suggest max 3-4 levels)
- MUST NOT commit JSON files with syntax errors

### Language Switching Constraints

**MUST Follow**
- MUST persistently save user choice after switching language
- Language switch MUST take effect immediately (update all visible text)
- MUST provide clear current language indicator
- MUST fallback to default language when language detection fails

**Prohibited Behaviors**
- MUST NOT require page refresh to take effect after switching (unless technical limitation)
- MUST NOT lose user choice after switching (not persisted)
- MUST NOT have flashing or layout confusion when switching
- MUST NOT cause data loss due to language switching

## Common Problem Diagnosis Table

| Symptom | Possible Cause | Solution |
|------|---------|---------|
| Shows translation key name instead of content | Translation file not loaded or key name misspelled | 1. Check backend config path correct<br>2. Check if translation key exists in file<br>3. Check namespace correctly specified |
| Some text not updated after language switch | Component not subscribed to language change or cache issue | 1. Confirm using useTranslation hook<br>2. Check for hard-coded text<br>3. Clear browser cache and retry |
| Plural form displays wrong | Plural rules not configured correctly or key name wrong | 1. Check if translation file has _plural suffix key<br>2. Confirm count parameter passed<br>3. Verify language plural rule configuration |
| Interpolation parameters not replaced showing {{name}} | Parameters not passed or placeholder format wrong | 1. Check if t() function second parameter passes object<br>2. Confirm placeholder uses double curly braces<br>3. Verify parameter name matches |
| Date format incorrect | Not using localized formatting or language pack missing | 1. Use date-fns format function<br>2. Import corresponding language locale<br>3. Or use Intl.DateTimeFormat API |
| Language detection fails always defaults to English | Detection order configured wrong or cache conflict | 1. Check detection.order configuration<br>2. Clear language cache in localStorage<br>3. Verify browser language settings |
| RTL layout confusion | Not setting dir attribute or CSS doesn't support | 1. Set dir="rtl" on root element<br>2. Use logical CSS properties (margin-inline-start, etc.)<br>3. Test Flexbox direction |
| Translation file loads slowly | File too large or network issues | 1. Split namespaces enable lazy loading<br>2. Use CDN acceleration<br>3. Enable browser caching |
| TypeScript no type hints | Not declaring translation resource type | 1. Create i18next.d.ts type declaration file<br>2. Import English translation file as type<br>3. Declare CustomTypeOptions interface |
| Production environment translations not updated | Cache strategy causes old version translations | 1. Add version number query parameter to translation files<br>2. Clear CDN cache<br>3. Update backend loading path |
| Nested key access fails | Key name misspelled or level wrong | 1. Check nested structure in translation file<br>2. Use dot separator to access (user.profile.name)<br>3. Enable debug mode view detailed logs |
| Server-side rendering hydration mismatch | Client and server language inconsistent | 1. Ensure passing language parameter during SSR<br>2. Use serverSideTranslations to load translations<br>3. Check language setting in cookie |

## Output Format Requirements

### Translation File Structure Specification

**First-level Structure: Group by Functional Domain**
```
Application Info
├─ app (application basic info)
│  ├─ name (application name)
│  ├─ welcome (welcome message)
│  └─ description (application description)
│
Navigation Menu
├─ nav (navigation related)
│  ├─ home (homepage)
│  ├─ products (products)
│  ├─ cart (shopping cart)
│  └─ profile (personal center)
│
Action Buttons
├─ actions (common operations)
│  ├─ save (save)
│  ├─ cancel (cancel)
│  ├─ delete (delete)
│  ├─ confirm (confirm)
│  ├─ submit (submit)
│  └─ loading (loading)
│
System Messages
├─ messages (system messages)
│  ├─ success (success prompt)
│  ├─ error (error prompt)
│  └─ confirm_delete (delete confirmation)
│
Form Validation
├─ validation (form validation)
│  ├─ required (required field)
│  ├─ email (email format)
│  └─ min_length (minimum length, with count parameter)
│
Time Display
└─ time (time related)
   ├─ just_now (just now)
   ├─ minutes_ago (X minutes ago, with plural)
   ├─ hours_ago (X hours ago, with plural)
   └─ days_ago (X days ago, with plural)
```

**Namespace Division Suggestions**

| Namespace | Contains | Use Case | Estimated Size |
|---------|---------|---------|---------|
| common | Global shared text (navigation/buttons/messages/validation) | All pages | 5-10KB |
| auth | Authentication related (login/register/forgot password) | Authentication flow pages | 2-5KB |
| user | User module (profile/settings/security) | User center pages | 3-8KB |
| product | Product module (list/details/filter/specs) | Product-related pages | 5-15KB |
| order | Order module (place order/payment/logistics/after-sales) | Order flow pages | 5-12KB |
| admin | Admin dashboard (dashboard/management/review) | Admin pages | 10-30KB |

**Key Naming Patterns**

- Simple text: `key_name` (e.g., submit_button, cancel_action)
- Nested objects: `category.subcategory.key` (e.g., user.profile.name)
- Plural forms: `key` + `key_plural` (English) or `key` + count parameter (multi-language)
- Interpolation parameters: use `{{paramName}}` in text (e.g., welcome_message: "Welcome, {{name}}!")
- State variants: `base_state` (e.g., status_pending, status_completed)

**File Naming Convention**

- Language directory: use ISO 639-1 two-letter code (en/zh/ja/ko/fr/de/es)
- Dialect distinction: use hyphen (en-US, en-GB, zh-CN, zh-TW)
- Filename: lowercase namespace name + .json extension (common.json, user.json)

**Version Management Suggestions**

- Translation file path includes version number: `/locales/v1.2.0/en/common.json`
- Or use query parameter to control cache: `/locales/en/common.json?v=1.2.0`
- Track translation file changes separately in Git
- Record changelog for major translation adjustments

---

## Summary

Internationalization is key technology for enhancing application globalization capabilities. Following this document's principles and practices ensures completeness, maintainability, and user experience of multi-language support. Focus on translation key management, namespace organization, date formatting, and RTL layout adaptation, avoid hard-coded text and formats. Through systematic translation management and strict validation processes, build truly internationalized frontend applications.
