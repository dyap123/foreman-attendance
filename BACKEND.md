# Foreman Attendance — Backend Architecture

How the data layer of the Webcor Concrete foreman daily sign-in app works. Like the rest of
the OpenYap apps, it has **no server of its own** — it's a static page that talks directly to a
cloud database. This doc explains that backend: the data model, the write/read paths, why it's
concurrency-safe, and the honest security limits.

---

## 1. The shape of it (serverless / JAMstack)

```
   PHONE / TABLET (the client)                 CLOUD
 ┌──────────────────────────────┐      ┌──────────────────────────┐
 │  index.html (one file)       │ <──> │  Firebase Realtime DB    │
 │   ├─ UI + sign-in logic      │      │  (one shared JSON tree)  │
 │   ├─ fb* helpers (DB bridge) │─────►│  foreman-attendance/…    │
 │   ├─ SignaturePad (canvas)   │      └──────────────────────────┘
 │   └─ jsPDF / CSV / share     │
 └──────────────────────────────┘   (static files served by GitHub Pages)
```

There is **no backend server we wrote.** The single `index.html` runs in the foreman's browser
and reads/writes a Firebase Realtime Database directly. The "backend" is that managed database.
Everything else — building the PDF, computing hours, the Apple share sheet — happens in the
browser. This is a **thin-backend / thick-client** design: a dumb data store + a smart page.

- **Hosting:** GitHub Pages (static). Live at https://dyap123.github.io/foreman-attendance/
- **Database:** Firebase RTDB, shared project `gen-lang-client-0119642855` (same project as the
  other OpenYap apps), isolated under its own namespace so it can't collide with them.

---

## 2. Firebase connection + the bridge

```js
const FIREBASE_CONFIG = { databaseURL: 'https://gen-lang-client-0119642855-default-rtdb.firebaseio.com' };
const NS = 'foreman-attendance';                       // every path is prefixed with this
function fbSet(path, data)   { return db.ref(`${NS}/${path}`).set(data); }
function fbUpdate(path, data) { return db.ref(`${NS}/${path}`).update(data); }
function fbPush(path, data)  { return db.ref(`${NS}/${path}`).push(data); }   // server-generated key
function fbGet(path)         { return db.ref(`${NS}/${path}`).once('value').then(s => s.val()); }
function fbListen(path, cb)  { db.ref(`${NS}/${path}`).on('value', s => cb(s.val())); } // live subscribe
function fbRemove(path)      { return db.ref(`${NS}/${path}`).remove(); }
```

The `NS` prefix is the **namespacing trick**: a shared database can host many apps without
collision, because every read/write is scoped under `foreman-attendance/…`. The six `fb*`
functions are the *entire* boundary between the app and the database — nothing else touches Firebase.

---

## 3. The data model (one JSON tree)

```
foreman-attendance/
  config/                         { primeContractor, projectName, subcontractor, managerPin, superPin }
  foremen/{id}/                   { name, classification, order, active, role }   (role 'super' = manager)
  members/{id}/                   { name, classification, order, active, foremanId }  (crew under a foreman)
  areas/{id}/                     { name, order }
  attendance/{YYYY-MM-DD}/{personId}/   ← THE attendance records
        { timeIn, timeOut, totalHours, injured,
          signatureType, signatureDataUrl, signatureText, signedAt, signedBy }
```

The spine is **`attendance/{date}/{personId}`** — one record per person per day. A foreman's whole
roster lives in `foremen/` + `members/`; `config/` holds project info and the PIN. There are **no
"totals" or "counts" stored** — everything aggregate is computed on read (see §6).

### A record (what one sign-in looks like)
A normal sign-in writes:
```js
{ timeIn, timeOut, totalHours, injured,
  signatureType: 'drawn'|'typed',
  signatureDataUrl: <base64 PNG of the drawn signature> | null,
  signatureText: <name for typed sig> | null,
  signedAt: <epoch ms>, signedBy: 'self' }
```
"Mark off" writes the same shape with `signatureType:'off'`, null times, `totalHours:0`.

---

## 4. The write path

```js
async function writeEntry(entry) {
  const path = `attendance/${state.signin.date}/${state.profile.id}`;   // ← keyed by date + person
  const existing = await fbGet(path);
  if (existing) { if (!confirm('overwrite?')) return; }                 // already-signed-in guard
  await fbSet(path, entry);                                            // one write to that person's key
  ...
}
```

Every sign-in is a single `set()` to **that person's own key for that day**. The optional
`fbGet` first is just a UX guard ("you already signed in — overwrite?"). Roster/config edits go
through `fbUpdate` on `foremen/`, `members/`, `config/`; new foremen/members/areas use `fbPush`
(Firebase generates a unique key, so two managers adding at once never collide).

---

## 5. Read + live sync (the observer pattern)

`fbListen(path, cb)` **subscribes** to a path. When anyone changes it, Firebase **pushes** the new
value to every subscribed device, which re-renders. The app doesn't poll ("any new sign-ins?") —
it's *notified*. That's why a manager's history view updates the instant a foreman signs in.
History/exports use `fbGet` for point-in-time reads of a date range.

---

## 6. Derived data — store raw, compute the rest

The app **never stores** hours totals, daily counts, or the attendance percentage. It stores only
the raw inputs (times, present/off, injured) and **computes everything on read**: `hoursBetween()`
for hours, sums for crew totals, and `present / (people × working-days)` for the attendance rate.

This is the most important backend decision, and it pays off twice:
- **Correctness:** a derived value can never drift out of sync with its source (the old paper/CSV
  flow had exactly that problem).
- **Concurrency:** because no shared counter is being read-modify-written, there's no lost-update
  race to begin with (see §8).

---

## 7. Signatures, auth, exports (all client-side)

- **Signatures** are captured on a `<canvas>` via SignaturePad and stored **inline as a base64 PNG**
  in the record (`signatureDataUrl`). Simple and self-contained; the trade-off is the `attendance`
  node grows with image data over time.
- **Auth** is a **roster picker** (tap your name) plus a **manager PIN** (`050103`, in `config/`).
  This is a friction gate, not real authentication — there are no passwords (see §9).
- **Exports** (weekly PDF via jsPDF, CSV) are generated **in the browser** and handed to the OS
  via the Apple/Android **share sheet** (`navigator.share`) with a download fallback. The server
  does no work.

---

## 8. Concurrency — designed away, not locked

The "two people at once" problem is **mostly impossible here by data-model design**:

- **Writes are partitioned by writer.** Two foremen signing in at the same second write to
  *different keys* (`attendance/2026-06-02/fausto` vs `…/pedro`). Firebase merges different keys
  with zero conflict. Every actor owns their own path — the strongest possible concurrency design.
- **No shared counters.** All aggregates are derived (§6), so there's no contended field for two
  clients to lose-update.
- **Manager vs. foremen** write to separate subtrees (`foremen/`/`config/` vs `attendance/`) — no
  contention.

Residual races are small and low-impact: the same person double-tapping submit is a check-then-act
(TOCTOU) gap on **their own** key (last-write-wins of their own data); a two-manager roster reorder
could interleave (but there's one manager). Contrast this with an app that has a shared `points`
counter — *that* needs transactions; this one doesn't, because nothing is genuinely shared.

> The lesson: the cheapest way to solve a concurrency problem is to **shape the data so it can't
> happen** — partition writes by actor, and derive aggregates instead of storing them.

---

## 9. Security model (and its honest limits)

Because there's no server, the Firebase config and the manager PIN live in client code — anyone who
views source can see them, and the database is currently **open** (a client could in principle write
any record). That's an accepted trade-off for an internal jobsite tool. If this ever needed to be
tamper-proof, the fix is the one trusted layer this architecture has:

- **Firebase Security Rules** (run on Google's servers): restrict each person to writing only their
  own `attendance/{date}/{their-id}` path, validate field shapes, and gate roster/config edits to
  the manager. Rules are also what would close the only real "anyone can overwrite" gap.

---

## 10. One-line summary

A static page that writes one record per person per day to a shared Firebase tree
(`attendance/{date}/{personId}`), subscribes for live updates, and computes every total in the
browser — so it's collision-free by construction, with PDFs/CSVs generated client-side and handed
to the OS share sheet. The only thing it can't do without a real server is *enforce* trust.
