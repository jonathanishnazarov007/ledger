# Firebase Cross-Device Sync for Ledger

**Date:** 2026-09-02
**Status:** Approved

## Problem

Ledger stores all state in `localStorage` under a single key (`ledger-state`).
Data is trapped on whichever device created it. Moving between phone and laptop
means manually exporting and importing a JSON backup.

## Goal

Edits made on one device appear on the user's other devices automatically,
without degrading the current offline-first behaviour.

## Non-Goals

- Multi-user or shared/household budgets. Single user, multiple devices.
- Per-field or per-transaction conflict merging.
- Replacing the existing JSON export/import backup feature. It stays.

## Architecture

### Authentication

Google sign-in via Firebase Auth (`signInWithPopup`, `GoogleAuthProvider`).

A sign-in control lives in the header. The app has two modes:

- **Signed out** — behaves exactly as today: `localStorage` only, no network.
  This is the default and remains fully functional.
- **Signed in** — Firestore is the source of truth; `localStorage` continues
  to act as a fast local cache and offline fallback.

### Data Model

One document per user:

```
users/{uid}  ->  { income, bufferPct, bills[], spending[], month, updatedAt }
```

The document body is the existing in-memory `state` object verbatim. This is
deliberate: it keeps `loadState`/`saveState` as thin wrappers rather than
forcing a data-layer rewrite.

`updatedAt` is a server timestamp, used only for display and debugging.

### Sync Flow

1. On auth state change to signed-in, attach `onSnapshot` to `users/{uid}`.
2. Remote change arrives -> merge into `state` -> `renderAll()`.
3. Local mutation -> `saveState()` -> write to `localStorage` AND
   `setDoc(..., { merge: true })` to Firestore.
4. The snapshot listener ignores echoes of the device's own writes
   (`snapshot.metadata.hasPendingWrites`) to avoid a render loop.

### Offline

Firestore IndexedDB persistence is enabled. Writes made offline are queued
locally and flushed on reconnect. The app is fully usable with no network.

### Conflict Resolution

Last write wins on the whole document. Accepted trade-off: a single person
switching between their own devices rarely produces true concurrent edits, and
per-transaction merging would add substantial complexity for a rare case.

### First Sign-In Reconciliation

| Local data | Cloud data | Behaviour                                  |
|------------|------------|--------------------------------------------|
| yes        | empty      | Upload local to cloud                      |
| empty      | yes        | Download cloud to local                    |
| yes        | yes        | Cloud wins; user is told before it applies |
| empty      | empty      | Nothing to do                              |

The third case shows an explicit confirmation rather than silently discarding
local data.

### Security Rules

```
match /users/{uid} {
  allow read, write: if request.auth != null && request.auth.uid == uid;
}
```

A user can access only their own document. No public read path exists.

### Status Indicator

The existing `#save-status` element becomes a real sync indicator:

- `signed out - saved on this device only`
- `synced`
- `offline - will sync when you reconnect`
- `sync error - export a backup below`

## Provisioning

Performed via `firebase-tools` CLI and the Firebase Management / Identity
Toolkit REST APIs:

1. `firebase login` - **the one interactive step**, requires browser OAuth
   consent. Google permits no automated path to create a project owned by a
   user account.
2. Create Firebase project.
3. Provision Cloud Firestore in native mode.
4. Deploy security rules.
5. Enable the Google sign-in provider.
6. Add the GitHub Pages origin to authorised auth domains.
7. Register a web app and read back its `firebaseConfig`.

Step 5 may not be fully automatable; the Identity Toolkit API can require an
OAuth client that only the console auto-creates. Fallback is a single console
toggle by the user.

## Cost

Within the Firebase Spark (free) tier. A single-user budget app is far below
the 50k document reads/day allowance.

## Risk: Config Key Exposure

`firebaseConfig` values are shipped in client-side `index.html` and are public
by design; they identify the project, they do not authorise access. Security is
enforced entirely by the Firestore rules above. This is Firebase's intended
model.
