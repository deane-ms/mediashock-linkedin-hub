# Firebase migration — work in progress, NOT yet live

This branch (`firebase-migration`) holds a rebuilt version of the Content Hub dashboard
that replaces per-browser `localStorage` with a shared Firebase backend, so anyone with
a `@mediashock.com.sg` Google account can edit it and see everyone else's changes live.

**This is not deployed.** The live site (`index.html` on `master`) is still the original
localStorage-only version. Nothing here goes live until it's manually finished and merged.

## What's done and tested

Built and verified against a local Firebase Emulator Suite (fully offline, no real
project needed for this part):

- Google Sign-In gate, restricted to `@mediashock.com.sg` accounts
- Restriction enforced twice: once in the UI, and independently confirmed at the
  database level via Firestore Security Rules (a non-mediashock account gets a real
  `permission-denied`, not just a UI block)
- Live sync verified across two independent signed-in sessions: calendar posts, planner
  ideas + feedback, and content buckets all propagate to other open sessions without a
  refresh
- Post images are stored as base64 directly on the post document (no Firebase Storage --
  new Firebase projects require the paid Blaze plan for Storage, and we opted to skip
  that rather than add billing). A size guard blocks saving before Firestore's 1MiB
  per-document limit is hit, with a clear error telling the user to remove/shrink images,
  instead of a cryptic failure after the fact. In practice this means posts work fine with
  a handful of normal-sized photos; a post stuffed with many large images may need to trim
  some down. Confirmed against the emulator's admin data view that a saved post's `images`
  field really does hold `data:image/jpeg;base64,...` inline, with no Storage involved.

## Data model

- `posts` collection — one doc per calendar post, doc ID = post ID
- `buckets` collection — one doc per content bucket, doc ID = bucket key, has an `order` field
- `ideas` collection — one doc per planner idea, doc ID = idea ID, `feedback` is an array field on the doc
- `analytics/current` — single doc holding the shared analytics data (days/topPosts/demographics/follower count).
  Date-range and granularity view preferences stay in each browser's `localStorage`, not Firestore,
  so one person changing their view doesn't change what everyone else sees.

## Status: real Firebase project created

Project `mediashock-content-hub-2237f` exists (Spark/free plan) and `firebaseConfig` in
`content-hub-firebase.html` and the project ID in `.firebaserc` are already filled in with
real values. Firebase Storage was deliberately skipped -- as of late 2024, new Firebase
projects require upgrading to the paid Blaze plan to use Storage at all (even within the
free-tier quota), and we chose the base64-in-Firestore fallback above instead of adding
billing.

## What's left to ship this for real

1. In the Firebase Console for this project, confirm **Firestore Database** (production
   mode) and **Authentication → Sign-in method → Google** are both enabled.
2. Publish `firestore.rules` (copy-paste into Firestore Database → Rules tab → Publish).
3. **Authentication → Settings → Authorized domains → Add domain** → add the real GitHub
   Pages domain (`deane-ms.github.io`).
4. Wrap `content-hub-firebase.html` with the DOCTYPE/head/body structure (same transformation
   `sync_from_scratchpad.py` in the repo root does for the localStorage version -- this file
   hasn't been run through it yet), copy it over `index.html` on `master`, and push.

## Local testing setup (for reference, not needed for production)

Local testing used a portable JRE + `firebase-tools` installed ad hoc in a scratch folder (not part of
this repo) to run `firebase emulators:start --project demo-mediashock-hub`, plus a plain
`python -m http.server` to serve the file over `http://localhost` (Firebase Auth's popup sign-in flow
needs a real http(s) origin, not `file://`). None of that tooling is required for the real deployment —
Firebase Hosting/GitHub Pages + a real project is all that's needed once configured.
