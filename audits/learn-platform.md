# Learn Platform Audit

> 2026-02-17

## 1. Features

### 1a. Overview

The site is a learning management system (LMS) for Founders and Coders' AI apprenticeship program.

### 1b. Global Features

- dark mode toggle
- an app switcher linking to apps.foundersandcoders.com
- a user profile avatar linking to accounts.foundersandcoders.com
- breadcrumb navigation on detail pages

### 1c. Sections

It contains five main sections:

#### 1c1. Learn

The core of the platform. Displays module cards (currently 7 modules covering topics like Kubernetes, Voice Agents, Docker, SSH, Python, and Conda). Each module has chapters, and each chapter contains individual exercises. Exercises include question-and-answer prompts with an XP reward system (25 XP per exercise). There's per-chapter progress tracking with percentage-based progress bars and "Begin Chapter" / "Continue Chapter" actions. A "View Chapter Details" expander reveals the full lesson plan per chapter. Modules also display instructor profiles, estimated hours, lesson counts, and category tags (icons for topics like infrastructure, AI, etc.). A "Course Outline" modal is available within exercises for quick navigation, and back/forward arrows allow stepping through exercises sequentially.

#### 1c2. Events

Split into "Upcoming" and "Past" tabs. Currently both are empty.

#### 1c3. Mastery

A spaced-repetition flashcard system with "Study" and "Decks" sub-tabs. Includes a Master Deck, deck creation, Anki import functionality, and a "Browse Public" option. Linked to a separate Bloom application on bloom.foundersandcoders.com.

#### 1c4. Dashboard

Learning analytics showing status (On Track), hours logged (5/420), topics started (0/70), topics mastered (0/70), hours breakdown (Workshops, Online Platform, Additional OTJ), a GitHub-style activity heatmap, and topic tracking tables organized by Knowledge (K1–K30+), Skill (S1–S34), and Behaviour (B1–B5) codes.

#### 1c5. Bloom

Opens in a new tab on a separate subdomain (bloom.foundersandcoders.com). A full-featured flashcard application with Study, My Decks, Stats, Browse, Settings, and About sections.

---

## 2. Functionality

### 2a. Working Correctly

- Module cards navigate to module detail pages on click
- Chapter progress percentages display and update (67%, 20%, 5% observed)
- "View Chapter Details" expander toggles properly (shows "Hide Details" when expanded)
- Breadcrumb navigation (Modules and module name are clickable links)
- Dark mode toggle works and applies a full dark theme
- "Course Outline" modal opens and displays all chapters with progress
- Back/forward navigation arrows within exercises function on click
- Exercise pages display questions, text input areas, and "Submit Answer"
- App switcher and profile avatar link to correct external domains
- Bloom opens correctly in a new tab
- Mastery section shows decks and provides deck management features
- Events tabs (Upcoming/Past) switch correctly

### 2b. Partially Working or Questionable

- Navigation items (Learn, Events, Mastery, Dashboard, Bloom) navigate correctly but are implemented as <div> elements with only JavaScript click handlers — not links or buttons, so browser features like middle-click-to-open-in-new-tab, right-click context menu, and URL previews on hover are unavailable
- The back/forward arrows and Course Outline button in the exercise view are also <div> elements with cursor: auto — they work on click but give no visual affordance that they're interactive
- Breadcrumb on exercise pages shows 4 levels but only the first two (Modules, Module Name) are clickable; the chapter name is not a link, so users can't navigate back to the chapter level from an exercise

---

## 3. UX

### 3a. Strengths

- Clean, visually appealing card-based layout for modules with abstract decorative backgrounds in distinct color palettes per course
- Clear visual hierarchy — progress bars, instructor avatars, lesson counts, and time estimates are well organized
- The XP system provides gamification and motivation
- Breadcrumb navigation provides wayfinding context
- Course Outline modal is useful for orientation within a module
- Dark mode is well-implemented with good color inversions
- The "View Chapter Details" accordion is a thoughtful progressive disclosure pattern

### 3b. Weaknesses

- Empty states: The Events page (both Upcoming and Past tabs) shows a completely blank white area with no messaging like "No upcoming events" or "Check back soon." This is confusing — users can't tell if content failed to load or if there are genuinely no events.
- Inconsistent card metadata: The "Bare Metal to Kubernetes" card shows "5 Lessons | null hours" — the literal string "null" is rendered to users (a data bug bleeding into the UI).
- Page title never changes: The <title> tag is always "Learn" regardless of which section or page you're on. This hurts browser tab management, bookmarking, accessibility, and SEO.
- No clear call-to-action on the Learn page: Module cards are clickable, but there's no explicit button like "Start Course" or "View Module" on the card itself. The clickable area is implied through the heading text and the full card, but the card footer icons (topic tags) may mislead users into thinking those are action buttons.
- Cursor styling: The main navigation items and exercise navigation arrows show cursor: auto (the default text cursor) rather than cursor: pointer, giving no visual hint that they're interactive.
- Content typos: The stretch goal chapter in Voice Agents contains the text "genuinly" (should be "genuinely"). The first chapter description says "we will walks you through" (should be "we will walk you through").
- No search or filtering: With 7 modules currently, this is manageable, but as the platform grows there's no way to search, filter, or sort modules.
- Dashboard information overload: The topics tracking tables (K1–K30+, S1–S34, B1–B5) are long lists all showing "No evidence yet" / "No evidence" for every row, which provides very little value and requires extensive scrolling.

---

## 4. Bugs

### 4a. Critical

- "null hours" displayed on Bare Metal to Kubernetes card — The module card shows the literal text "5 Lessons | null hours" because the hours data is null/undefined and isn't being handled with a fallback. This is a data-rendering bug.

### 4b. Moderate

- Typo: "genuinly" — In the Voice Agents module, chapter 4 ("Stretch Goal") description reads "genuinly useful" instead of "genuinely useful."
- Grammar error: "we will walks you through" — In chapter 1 ("Build a pipeline agent with LiveKit") of Voice Agents, the description says "we will walks you through" instead of "we will walk you through."
- Exercise navigation elements not keyboard-accessible — The back arrow, forward arrow, and "Course Outline" button in the exercise view are <div> elements without tabindex, role, or cursor: pointer. They cannot be reached via Tab key and are invisible to screen readers.
- Navigation bar items not focusable — All five main nav items (Learn, Events, Mastery, Dashboard, Bloom) are <div> elements with React onClick handlers but no tabindex, role="button" or role="link", and no cursor: pointer. They're entirely inaccessible to keyboard and assistive technology users.

### 4c. Minor

- No empty state for Events — Both Upcoming and Past tabs show completely blank content with no placeholder message.
- Static page title — The document <title> is always "Learn" regardless of the current page/section.
- Activity heatmap table uses <td> for header cells — The month labels (Feb, Mar, etc.) are rendered as <td> instead of <th>, which hurts table semantics.

---

## 5. Accessibility

### 5a. Critical Issues

- No <nav> landmark — The entire site has zero <nav> elements. The main navigation bar, breadcrumbs, and exercise navigation are all built with unsemantic <div> elements. Screen reader users have no way to identify or jump to navigation regions.
- No <main> landmark — There is no <main> element wrapping page content. Screen readers cannot identify the primary content area.
- No <footer> landmark — No footer element exists.
- Navigation items are inaccessible — All main nav items and exercise nav controls are <div> elements with no role, tabindex, or ARIA attributes. Keyboard-only users and screen reader users cannot interact with the primary navigation.
- No skip navigation link — There is no "Skip to content" link, forcing keyboard users to Tab through the entire header and navigation on every page.
- No <h1> on any page — The module listing page starts with <h2> elements for each module. Module detail and exercise pages also lack an <h1>. This breaks the heading hierarchy and makes it difficult for screen reader users to understand page structure.
- Exercise page has zero headings — The exercise/question page has no heading elements at all, making it impossible for screen reader users to orient themselves.

### 5b. Serious Issues

- Multiple links without accessible names — 7 links have no text, aria-label, or title: the app switcher icon link, the profile avatar link, and all instructor avatar links on module cards. Screen readers would announce these as just "link" with no description.
- Progress bars lack ARIA attributes — Progress indicators on module cards have no role="progressbar", aria-valuenow, aria-valuemin, or aria-valuemax attributes. Screen reader users cannot perceive progress information.
- No ARIA roles used anywhere — The entire page has zero role attributes on any element. Tabs (Upcoming/Past on Events, Study/Decks on Mastery) are not marked with role="tablist", role="tab", or role="tabpanel".
- Color contrast failure on metadata text — The lesson count and hours text (e.g., "5 Lessons | null hours") uses rgb(180, 190, 200) on a white background, yielding a contrast ratio of approximately 1.89:1 — far below the WCAG AA minimum of 4.5:1 for text at 12px. This text is effectively unreadable for users with low vision.
- Activity heatmap table has no <th> elements — All cells including headers are <td>, making the table meaningless to screen readers.
- Tables lack captions — Dashboard tables for Knowledge, Skill, and Behaviour topics have no <caption> elements to describe their purpose.

### 5c. Moderate Issues

- Missing meta description and Open Graph tags — No <meta name="description"> or OG tags exist, impacting both SEO and when links are shared on social platforms.
- lang="en" is correctly set — This is done properly.
- Viewport meta is correctly set — width=device-width, initial-scale=1 is present.
- Focus indicator exists — There is a visible orange focus ring (rgb(229, 151, 0) auto 1px) on focusable elements, which is good — but since most interactive elements aren't focusable, this only applies to the few actual links and the dark mode button.
- CSS :focus-visible rules exist — The codebase has focus-visible styles, suggesting awareness of keyboard accessibility, but the implementation falls short because most interactive elements can't receive focus.

### 5d. Summary of Severity Ratings

| Category | Critical | Serious | Moderate | Minor |
| --- | --- | --- | --- | --- |
| Bugs | 1 | 3 | 2 | — |
| Accessibility | 5 | 5 | 2 | — |
| UX | — | 2 | 4 | 2 |

The platform has solid visual design and a thoughtful feature set, but suffers from fundamental accessibility barriers. The most impactful fix would be converting the navigation, exercise controls, and tab interfaces from <div> elements to proper semantic HTML (<a>, <button>, <nav>, <main>) with appropriate ARIA attributes. The "null hours" data bug and the color contrast failure on metadata text are also high-priority fixes.
