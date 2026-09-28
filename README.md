# Marvel Reading Atlas

[![Quality](https://github.com/MykolaDotsenko/marvel-reading-atlas/actions/workflows/quality.yml/badge.svg)](https://github.com/MykolaDotsenko/marvel-reading-atlas/actions/workflows/quality.yml)

**Find an issue. Build a reading order. Keep your place.**

Marvel Reading Atlas is a React 19 comic-discovery and reading-planning app built around the community-maintained [Marvel Metadata API](https://marvel.emreparker.com/). It turns a large external catalogue into a focused reading workflow: search, inspect, save, order, and track progress without creating an account.

[**Open the live app**](https://marvel-starter-phi.vercel.app/) · [Architecture](./ARCHITECTURE.md) · [Browser tests](./e2e/reading-atlas.spec.js)

## What the product does

- search 37,500+ indexed Marvel issues by title
- browse recent issues with bounded pagination
- open shareable issue dossiers through URL state
- inspect covers, series, publication dates, page counts, and creator credits
- save issues and keep a bounded recent-history list
- build an ordered reading list of up to 200 issues
- move items without relying on drag-only interaction
- mark issues read/unread and continue from the next unread issue
- keep Saved, Recent, and Reading useful when the provider is temporarily unavailable
- work across desktop and mobile without accounts, analytics, cookies, or tracking

The core journey is intentionally simple:

~~~text
Discover → inspect → save → order → read → continue
~~~

## The rebuild

This repository began as a small Marvel character training app. Then its original data source — the Marvel Developer API — became unavailable.

Rather than preserve an interface around a dead dependency, I changed the product around the data that was still available and useful:

1. isolate the provider boundary;
2. move to the maintained Marvel Metadata API;
3. normalize provider data into an application-owned issue model;
4. redesign the product around comic discovery and reading progress instead of character lookup.

That change is the main story of the repository. It shows how an external API failure can become a product and architecture decision rather than just an endpoint replacement.

## Engineering highlights

### Provider data never becomes application state directly

External payloads are validated and normalized before the UI sees them. The reading library stores compact product-domain snapshots instead of raw provider responses, so local state stays stable even if the upstream schema changes.

### Reading state remains useful during provider downtime

Saved issues, recent history, and the reading list render from local snapshots. The app does not need to rehydrate every saved item through N+1 detail requests before the user can continue.

### URL state is shareable by default

Search query, selected issue, and active view live in the URL:

~~~text
/?q=Secret+Wars&issue=52447&view=reading
~~~

Browser Back/Forward therefore works naturally, and an issue dossier can be shared without introducing a routing library solely for this scope.

### Network work is bounded

- obsolete list/detail requests are aborted
- every provider request has a 10-second timeout
- successful responses use a 5-minute / 50-entry in-memory cache
- pagination failure preserves already-loaded results
- malformed list items fail closed instead of breaking the full grid
- one-character searches are stopped before reaching the provider
- a weekly live-contract smoke check detects upstream drift separately from deterministic PR checks

## Runtime

- React **19.3**
- React DOM **19.3**
- Vite **8.3**
- semantic HTML
- layered CSS
- Fetch + AbortController
- History / URLSearchParams
- Web Storage
- [Marvel Metadata API](https://marvel.emreparker.com/)

Navigation is handled with browser History and URLSearchParams, while durable reading state sits behind a small local-storage adapter. For the current product scope, React and React DOM are the only runtime packages the application needs.

## Provider contract

The active provider is an unofficial open-source metadata service, not Marvel Entertainment. The current integration uses:

- `/v1/issues`
- `/v1/issues/{id}`
- `/v1/search/issues`
- browser GET CORS
- metadata only; no comic content
- a documented 60 requests/minute limit with burst allowance

Series and creator endpoints remain available for future expansion, but the current app does not pull extra detail simply because the provider exposes it.

Explore and search cards use summary payloads directly. A detail request is made only when an issue dossier is actually opened.

## Architecture

~~~text
React UI
  |
  +--> feature hooks
  |       |
  |       +--> Marvel Metadata provider adapter
  |       |       |
  |       |       +--> pure issue normalization
  |       |
  |       +--> URL / history state
  |       |
  |       +--> local reading-library adapter
  |
  +--> semantic HTML + layered CSS
~~~

The dependency rule is straightforward:

> External provider payloads are normalized before UI components see them. Local reading state stores product-domain snapshots, not provider response blobs.

See [ARCHITECTURE.md](./ARCHITECTURE.md) for the detailed rationale.

## Local reading model

The browser persists deliberate user intent plus compact issue snapshots:

~~~json
{
  "saved": [{ "id": 52447, "title": "Secret Wars (2015) #1" }],
  "recent": [{ "id": 52447, "title": "Secret Wars (2015) #1" }],
  "readingList": [
    {
      "issue": { "id": 52447, "title": "Secret Wars (2015) #1" },
      "read": true
    }
  ]
}
~~~

Transient loading state, request errors, and provider payloads stay in memory.

Corrupt localStorage data resets safely instead of blocking the product.

## Accessibility and interaction

The interface includes:

- skip navigation
- semantic headings, lists, buttons, and native progress
- descriptive action names
- `aria-pressed` for durable toggles
- visible keyboard focus
- reduced-motion support
- forced-colors support
- non-drag controls for reading-list ordering
- responsive mobile reflow without horizontal overflow

## Verification

~~~bash
npm ci
npm run check
npm run test:e2e
~~~

Pull requests verify:

- ESLint with zero warnings
- Vitest domain/API/storage tests
- Vite production build
- production dependency audit
- Chromium
- Firefox
- WebKit
- Pixel 7 viewport
- axe automated accessibility checks
- horizontal-overflow regression
- GitHub Pages base-path/provider bundle contract

The live provider is checked separately on a weekly schedule so upstream availability does not make ordinary pull requests flaky.

## Local development

Requirements: Node.js 24+

~~~bash
npm ci
npm run dev
~~~

No API key is required.

Optional provider override:

~~~bash
VITE_MARVEL_METADATA_API=https://example.test/v1 npm run dev
~~~

## Deployment

The canonical public demo is the [Vercel deployment](https://marvel-starter-phi.vercel.app/).

The repository also retains a GitHub Pages deployment path from before the repository rename. Its build currently uses the legacy base path:

~~~text
VITE_BASE_PATH=/Marvel-Starter-React-App/
~~~

That compatibility detail is kept out of the product identity; the Vercel deployment is the recruiter-facing link.

## Data and trademark note

Marvel Reading Atlas is an unofficial portfolio project and is not affiliated with Marvel Entertainment. Metadata comes from the community-maintained Marvel Metadata API. No comic pages or paid comic content are distributed by this repository.
