# Dialdex: Project Plan

> A speed dial for people with too many speed dials.
> Free and open source (GPL-3.0), for Chromium-based browsers and Firefox.

Status: planning. Last updated: 2026-11-17.

---

## 1. Vision

Classic speed dial addons work well with 20 dials. After some years of use you have 1000+ dials in 60+ groups,
and you end up with overflow groups like "Dev I", "Dev II", "Dev III". Thumbnails turn into black boxes,
and a simple text search is not enough to find things again.

Dialdex is built around one idea: **retrieval first, organization second.**

- A fast command palette with good ranking is the main way to find a dial.
- The thumbnail grid stays for the dials you use every day.
- Groups, tags and smart groups give structure without forcing one rigid hierarchy.
- Indexing and tag suggestions happen automatically, but the user stays in control.

### Non-goals

- No cloud service run by us, no accounts, no telemetry, no ads.
- No replacement for the browser's bookmark manager or history.
- No password storage.

---

## 2. Principles

1. **Local first.** All data lives in the browser (IndexedDB). Nothing leaves the machine unless the user sets up sync.
2. **One codebase, two targets.** Chrome (MV3) and Firefox (MV3) builds from the same source.
3. **Minimal permissions.** Every permission must be justified. Broad host access is opt-in.
4. **Suggestions, not silent magic.** Auto tags are proposals the user confirms. Otherwise tags turn into noise.
5. **Fast.** The new tab page must be usable in under 100 ms with 5000 dials.
6. **Keyboard first, mouse friendly.**
7. **Easy to leave.** Full export to an open JSON format at any time.

---

## 3. Feature map

Legend: **MVP** = needed for the first usable version, **v1** = needed before store release, **later** = nice to have.

### 3.1 Data and import

| Feature | Priority | Notes |
|---|---|---|
| Dials: URL, title, group, tags, notes, favicon, thumbnail | MVP | |
| Groups with one level of nesting ("Dev > gRPC") | MVP | Removes the need for "Dev I/II/III" |
| Import from browser bookmarks | MVP | Folder becomes group |
| Import from Group Speed Dial / FVD Speed Dial exports | MVP | Need real sample exports to reverse the format |
| Export / import of Dialdex JSON (versioned schema) | MVP | Also the backup mechanism |
| Archive instead of delete | v1 | Archived dials stay searchable on demand |
| Undo for destructive actions | v1 | |

### 3.2 Finding things

| Feature | Priority | Notes |
|---|---|---|
| Command palette (Ctrl+K, or just start typing on the new tab) | MVP | |
| Fuzzy full-text search over title, URL, domain, group, tags, notes | MVP | MiniSearch or FlexSearch |
| Frecency ranking (frequency + recency of use) | MVP | Based on own click counts |
| Search operators: `tag:`, `group:`, `domain:`, `-exclude` | v1 | |
| Optional: use browser history visits for ranking | later | Needs `history` permission, opt-in |
| Semantic search with local embeddings | later | See 3.4 |

### 3.3 Indexing

| Feature | Priority | Notes |
|---|---|---|
| Metadata capture on add: title, meta description, OpenGraph, keywords, canonical URL, favicon | MVP | |
| Index page text while the user visits a dial (content script) | v1 | Opt-in, needs host permissions |
| Background re-index as small resumable jobs | v1 | Chrome kills idle service workers after ~30 s |
| Health check: dead links, redirects, moved domains | v1 | |
| Duplicate and near-duplicate detection | v1 | Normalized URL + canonical URL |

### 3.4 Tagging

| Feature | Priority | Notes |
|---|---|---|
| Manual tags with autocomplete | MVP | |
| Rule-based tags (domain patterns, regex rules) | v1 | e.g. `*.microsoft.com` -> `dev`, `dotnet` |
| Keyword-based tag suggestions (TF-IDF over own collection) | v1 | No model needed |
| Tag management: rename, merge, delete | v1 | |
| Embedding-based tag suggestions and semantic search (transformers.js) | later | 25 to 90 MB model, must prove its value over TF-IDF first |
| Chrome built-in AI (Gemini Nano) as optional enhancer | later | Chrome only, never a core dependency |

### 3.5 Organization and views

| Feature | Priority | Notes |
|---|---|---|
| Group tabs/sidebar that scales to 100+ groups | MVP | Collapsible tree instead of two rows of tabs |
| Drag and drop of dials and groups | MVP | |
| Display modes per group: thumbnail grid, compact tiles, list | MVP | Compact mode fits about 3x more dials |
| Multi-select and bulk operations (move, tag, archive) | v1 | |
| Smart groups (saved queries) | v1 | e.g. `tag:dev -visited:<90d` |
| Stale dial review ("not opened for a year") | v1 | |
| Pinned / favorites row | v1 | |

### 3.6 Thumbnails

| Feature | Priority | Notes |
|---|---|---|
| Fallback tile: favicon + title + generated color | MVP | Never show a black box |
| Capture via `tabs.captureVisibleTab` when the user is on the page | v1 | |
| Manual recapture, custom image upload | v1 | |
| Detect bad captures (blank, login page) | later | |

### 3.7 Settings, i18n, accessibility

| Feature | Priority | Notes |
|---|---|---|
| Light / dark / system theme | MVP | |
| English and German UI | v1 | Code and identifiers stay en-US |
| Keyboard navigation everywhere, screen reader labels | v1 | |

### 3.8 Sync (later)

Browser `storage.sync` is far too small (Chrome: about 100 KB). Options, all opt-in:

1. Export / import a file in a synced folder (works today, no extra code).
2. WebDAV (Nextcloud and others).
3. A small self-hostable backend.

Needs a conflict strategy (per-dial last-write-wins with tombstones is probably enough).

---

## 4. Architecture

### 4.1 Tech stack

| Area | Choice | Reason |
|---|---|---|
| Language | TypeScript (strict) | |
| Extension framework | [WXT](https://wxt.dev) | One codebase, builds for Chrome and Firefox, handles manifest differences |
| UI framework | Svelte 5 | Small bundle, fast new tab render, little boilerplate (decided 2026-11-17) |
| Storage | IndexedDB via Dexie | Large data, indexes, transactions |
| Search | MiniSearch | Small, fast, fuzzy + prefix, serializable index |
| Unit tests | Vitest | |
| E2E tests | Playwright (loads the unpacked extension in Chromium) | Firefox E2E via web-ext later |
| Lint / format | ESLint + Prettier | |
| CI | GitHub Actions: lint, test, build both targets, attach zips to releases | |

### 4.2 Building blocks

```
+-------------------------------------------------------------+
|  New tab page (UI)                                          |
|   - Grid / compact / list views                             |
|   - Group tree                                              |
|   - Command palette                                         |
+----------------------------+--------------------------------+
                             | messages / direct DB access
+----------------------------v--------------------------------+
|  Core (pure TypeScript, no browser APIs, fully unit tested) |
|   - Domain model: Dial, Group, Tag, SmartGroup              |
|   - Ranking (frecency), query parser, tag rules             |
|   - Import / export mappers                                 |
+----------------------------+--------------------------------+
                             |
+----------------------------v--------------------------------+
|  Adapters (browser specific, behind interfaces)             |
|   - Storage (Dexie), Search index (MiniSearch)              |
|   - Thumbnail capture, Favicon, Page fetch                  |
|   - Permissions, Context menu, Optional AI                   |
+----------------------------+--------------------------------+
                             |
+----------------------------v--------------------------------+
|  Background (service worker / event page)                   |
|   - Job queue: re-index, health check, thumbnail jobs       |
|   - Jobs are small and resumable (state in IndexedDB)       |
+-------------------------------------------------------------+
|  Content script (opt-in): extract text + metadata on visit  |
+-------------------------------------------------------------+
```

The core layer has no dependency on browser APIs. This keeps most of the logic testable with plain unit tests
and makes Chrome/Firefox differences a matter of adapters.

### 4.3 Data model (first draft)

```ts
interface Dial {
  id: string;            // UUID
  url: string;
  normalizedUrl: string; // for duplicate detection
  title: string;
  description?: string;
  groupId: string;
  tagIds: string[];
  notes?: string;        // markdown
  faviconRef?: string;
  thumbnailRef?: string; // blob stored separately
  position: number;      // order inside group
  createdAt: number;
  updatedAt: number;
  lastOpenedAt?: number;
  openCount: number;
  archived: boolean;
  health?: { status: "ok" | "dead" | "redirect" | "unknown"; checkedAt: number; redirectTo?: string };
}

interface Group     { id: string; name: string; parentId?: string; position: number; viewMode: "grid" | "compact" | "list"; }
interface Tag       { id: string; name: string; color?: string; source: "user" | "rule" | "suggested"; }
interface SmartGroup{ id: string; name: string; query: string; position: number; }
interface TagRule   { id: string; pattern: string; kind: "domain" | "regex"; tagIds: string[]; enabled: boolean; }
```

Thumbnails and page text live in separate object stores, so the main dial list stays small and fast to load.

### 4.4 Permissions plan

| Permission | When | Why |
|---|---|---|
| New tab override | always | Core function |
| `storage`, `unlimitedStorage` | always | IndexedDB with thumbnails |
| `activeTab`, `contextMenus` | always | "Add current page to Dialdex" |
| `bookmarks` | optional, requested on import | Bookmark import |
| `tabs` | optional | Thumbnail capture |
| Host permissions `<all_urls>` | optional, opt-in | Page indexing, health checks |
| `history` | optional, opt-in | Ranking from history |

Optional permissions are requested at runtime, at the moment the user enables the feature.

---

## 5. Milestones

| Milestone | Content | Result |
|---|---|---|
| **M0 Setup** | Repo, WXT skeleton, CI, lint, tests, GPL-3.0, README | Empty new tab builds for both browsers |
| **M1 Core data** | Data model, Dexie storage, JSON export/import | Dials survive restarts |
| **M2 Import** | Bookmarks, Group Speed Dial, FVD | Own real data loaded (dogfooding starts) |
| **M3 UI basics** | Group tree, grid/compact/list, drag and drop, fallback tiles | Usable as daily new tab |
| **M4 Retrieval** | Command palette, MiniSearch index, frecency, search operators | The main differentiator works |
| **M5 Indexing + tags** | Metadata capture, rule tags, TF-IDF suggestions, tag management | Auto tagging |
| **M6 Maintenance** | Health checks, duplicates, stale review, bulk ops, smart groups | Large collections stay clean |
| **M7 Thumbnails** | Capture, recapture, custom images | Visual polish |
| **M8 Release prep** | i18n, a11y, privacy policy, store listings, screenshots | Published on Chrome Web Store and AMO |
| **Later** | Sync, embeddings, Gemini Nano, history ranking | |

Personal use starts after M3. Store release after M8.

---

## 6. Store and release notes

- **Chrome Web Store:** one-time developer fee, privacy policy required when host permissions are used,
  single purpose description ("new tab speed dial with search").
- **Firefox AMO:** source code must be reviewable if the build bundles/minifies. A public repo plus
  build instructions in the README cover this.
- **Edge Add-ons / Opera:** the Chrome build should work; optional extra listings.
- Releases: tagged versions, CI builds both zips, changelog via Conventional Commits.
- Privacy policy: "All data stays local. No telemetry. Network requests only go to the sites of your own dials."

---

## 7. Open decisions

1. ~~**UI framework.**~~ Decided 2026-11-17: **Svelte 5**.
2. **Import formats.** Need real export files from Group Speed Dial and FVD Speed Dial to write the mappers.
3. **Nesting depth of groups.** One level (proposed) or unlimited?
4. **Minimum browser versions.** Proposal: last two major versions of Chrome and Firefox, plus Firefox ESR.
5. **Repo name / domain.** `holonsoft/dialdex`, check availability of the name on GitHub, both stores and a domain before M0.
