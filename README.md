# Currently

Currently is a movie/TV tracker for iOS (App Store review pending) and Android (Play Store listing ready), plus a web app. Users track what they're watching across shared or personal lists, get status alerts when a tracked show is renewed/canceled/airs a new episode, rate titles (Rotten Tomatoes/IMDb/Letterboxd scores shown alongside), and get AI-assisted recommendations — either a personalized "For You" feed or a natural-language ask ("something funny, not too long, similar to Superbad"). It started as a two-person tool, opened up to friends and family, and is evolving toward an opt-in public/private social layer.

---

## Why is this repo private?

This isn't a commercial product — no billing, no ads, open registration, friends-and-family scale. It's private mainly because the Firestore security model, TMDB/MDBList integration details, and the recommendation-ranking logic are things I'd rather iterate on without an audience, and because the app handles real user accounts and data. I'm happy to grant temporary read access or walk through the architecture live — reach out via [LinkedIn](https://www.linkedin.com/in/danielkalo) or the contact on my resume.

---

## Technical Architecture

### Frontend — Next.js (App Router) + Capacitor (iOS, Android)

The app is a single Next.js codebase, deployed to Vercel as the canonical web app, and wrapped by [Capacitor](https://capacitorjs.com/) for native iOS and Android distribution. There's no separate mobile codebase — the native shells point `server.url` at the live production deployment rather than bundling a local web build, so a real server-side API (auth-gated TMDB proxying, cron-driven polling, Claude calls) stays possible without a static export.

- **Next.js 16 (App Router) + React 19 + TypeScript** — no JS-only files
- **Tailwind CSS v4** for styling
- **Native bridges via Capacitor plugins**: Firebase Auth (`@capacitor-firebase/authentication`) for native Google/Apple sign-in (required — Google's OAuth policy rejects sign-in from an embedded WebView), Firebase Messaging (`@capacitor-firebase/messaging`) for push
- **Drag-and-drop manual list ordering** via `@dnd-kit`
- Mobile-specific UI: a bottom tab bar below the `md` breakpoint, safe-area-aware padding for the iOS Dynamic Island/home indicator, and an offline fallback page for no-connectivity launches

### Backend — Vercel Serverless (Next.js API routes)

All third-party API calls are proxied through auth-gated Next.js API routes so no API key ever reaches the client, and every route verifies a real Firebase ID token before doing anything:

- **`/api/tmdb/*`** — search, details, person credits, recommendations, and batch-resolve, all backed by [TMDB](https://www.themoviedb.org/) for movie/TV metadata, cast, trailers, and streaming-provider availability
- **`/api/mdblist/ratings`** — Rotten Tomatoes (critics + audience), IMDb, and Letterboxd scores via [MDBList](https://mdblist.com/), one lookup per tracked title by TMDB ID
- **`/api/recommendations/ask`** — natural-language recommendation requests parsed by Claude (Anthropic Messages API, direct `fetch`, no SDK) into structured filters (genre, runtime, cast, watch provider, rating thresholds, franchise inclusion/exclusion), then turned into real TMDB `/discover` + `/recommendations` queries
- **`/api/notes-import/extract-image`** and **`/extract-pdf`** — bulk list import from a photo (Claude vision) or PDF (`pdfjs-dist` text extraction, falling back to vision when the PDF has no recoverable text layer)
- **`/api/cron/poll-status`** — a Vercel Cron job (Admin SDK, not Firebase Cloud Functions — avoids requiring Firebase's paid plan since the app is already on Vercel) that runs daily, diffs every tracked title's TMDB status/season/cast/trailer/streaming data against its last known state, writes change events for the News feed, matches tracked titles against entertainment RSS feeds for press coverage, sends push notifications for anything new, and prunes events older than 30 days

### Auth — Firebase

| Provider | Path |
|---|---|
| Google OAuth | Native Capacitor Firebase Auth plugin (mobile) / popup flow via a custom first-party auth domain (web) |
| Apple OAuth | Native Sign in with Apple (iOS only — required once Google Sign-In is offered, per App Review Guideline 4.8) |
| Email/password | Firebase native, with a required display name + username at sign-up |

`auth-context.tsx` wraps the Firebase JS SDK and exposes auth state app-wide; the native paths branch on `Capacitor.isNativePlatform()` rather than duplicating the auth UI.

### Persistence — Firestore, real per-list access control

Firestore is the only data store — no separate cache layer. Every collection is access-controlled by `firestore.rules`, verified by an emulator-based rules test suite covering every invariant (membership, field-pinning on writes, ownership checks) rather than assumed correct from the rule text alone:

- **`titles`** — tracked titles per list: TMDB metadata, status, personal watch state, per-person ratings, streaming providers, and the "last known" fields the daily poll diffs against
- **`lists`** — personal or shared lists, owner + member UIDs, per-list display preferences
- **`listInvites`** — invite-link tokens for joining a shared list
- **`titleEvents`** — an append-only change-history log (status changes, new seasons, cast changes, new trailers, streaming availability, press mentions) that powers the News feed and push notifications; written only by the server, never by a client
- **`personalRatings`** — ratings on titles not tracked on any list, private to the rater
- **`users`** / **`usernames`** — per-account preferences and a uniqueness registry for usernames, enforced by Firestore rules rather than app logic

A rating or a watch-status change on a title tracked across multiple of a user's own lists syncs to every copy in one atomic batch write, rather than only the list open in the UI.

### Push Notifications — FCM/APNs via Capacitor

Real native push (not web push): client-side permission + token registration on sign-in, tokens keyed by device rather than account (explicitly cleaned up on sign-out so a second account on the same device doesn't inherit stale routing), server-side sending from the daily cron job via the Admin SDK, cross-list dedup so one title tracked on two lists doesn't double-notify, and a per-user daily rate limit enforced with an atomic Firestore transaction.

### Recommendations — TMDB graph + Claude

Two entry points on `/recommendations`:
- **"For You"** — aggregates TMDB's own recommendation graph across a user's highest-rated or completed titles, ranked by how often a candidate recurs across seeds, with genre/media-type/rating filters
- **Natural-language ask** — free text parsed into structured filters by Claude, covering genre, runtime, cast, watch provider, franchise handling (distinguishing "X movies" from "similar to X"), and multi-source rating thresholds, then resolved against real TMDB queries — not a canned response

### Social — shared lists today, opt-in public profiles as a researched direction

The current social model is invite-gated: shared lists have an owner and members, ratings from other list members are visible on shared titles, and there's no unauthenticated browsing of the app at all. A public/private social layer (discoverable public lists and profiles, opt-in per user) has been researched against Letterboxd, Trakt, Serializd, and TV Time, but is intentionally not built yet — visibility stays private-by-default until that ships.

---

## Project Structure

```
src/
  app/
    api/                  # Auth-gated Next.js API routes (TMDB, MDBList, recommendations,
    │                     #   notes-import, cron, account deletion)
    account/, news/, recommendations/, login/, register/, invite/[inviteId]/, ...
  components/
    titles/               # TitleCard, TitleDetailModal, TitleList, TitleSummary
    lists/                # ListPicker, ListSwitcher
    add-title/, auth/, layout/, news/, notes-import/, recommendations/, search/, ui/
  hooks/                  # useLists, useModalA11y
  lib/
    firebase.ts           # Client Firebase SDK init
    auth-context.tsx      # Auth state provider
    titles.ts, lists.ts, titleEvents.ts, tmdb.ts, mdblist.ts, userPreferences.ts, ...
    server/               # Admin-SDK-only: firebaseAdmin, verifyFirebaseIdToken,
                           #   push notifications, recommendation ranking/parsing, RSS press
ios/                       # Capacitor iOS project (Xcode)
android/                   # Capacitor Android project (Gradle)
firestore.rules            # Per-list membership access control, field-pinned writes
tests/                     # Firestore rules test suite (emulator-based)
```

Testing: Vitest for unit tests (400+ across the `lib`/`api` layers) and an emulator-backed Firestore rules suite (`@firebase/rules-unit-testing`) covering every access-control invariant — run together via `npm test`.

---

## App

- **Live web app**: [currently.kalotech.dev](https://currently.kalotech.dev)
- **iOS**: [Available on the App Store](https://apps.apple.com/us/app/currently-tv-movie-tracker/id6805608081)
- **Android**: [Available on the Google Play Store](https://play.google.com/store/apps/details?id=com.kalotech.currently)
