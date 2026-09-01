# finance_tracking

A single-page expense tracker. One `index.html`, no build step. Google sign-in gate, and
every transaction is committed to a private data repository as JSON.

## Deploy

1. **Settings → Pages → Source: `main` / root.** It lands on
   `https://akshrk.github.io/finance_tracking/`.
2. **Nothing to change in Google Cloud Console.** The OAuth client is authorised per
   *origin*, so any page on `https://akshrk.github.io` is already covered by the same
   client ID and email allowlist.
3. **Open it and paste an access token** under *Settings → Access token*.

The app is self-contained: its own token, its own sign-in session, and it never reads or
writes `fitness-data`.

## Where the data goes

| | |
|---|---|
| Repo | `AkshRk/finance_tracking_data` (private) |
| Path | `transactions/YYYY-MM.json` — one file per month |
| Token | Fine-grained PAT, **Contents: Read and write** |

**The token must be scoped to this repository.** Fine-grained PATs grant access to named
repositories, so one issued for a different repo returns a 404 here. Set **Repository
access → Only select repositories → `finance_tracking_data`**; the default *Public
repositories* setting cannot see a private repo at all.

The owner, repository and folder are fixed in the page (`STORE` in the tracker script) and
are deliberately not exposed in the UI, so a stale saved value can't strand the app. Only
the token is stored in the browser. The repository name does appear in the 404 error —
that is the one moment it is worth knowing.

Each transaction is one commit, so `git log` reads as a spending history:

```
₹340.50 · Akshath · Restaurant · Online — Lunch with Ravi (2026-09-01)
₹1,100.00 · Pavithra · Charity · Online — Temple donation (2026-09-04) [recorded by Akshath]
```

### File shape

```json
// transactions/2026-09.json
{
  "month": "2026-09",
  "updated": "2026-09-01T12:03:35.863Z",
  "count": 9,
  "total": 10665.25,
  "transactions": [
    {
      "id": "2026-09-01T0920-ly1wzc",
      "date": "2026-09-01",
      "time": "09:20",
      "at": "2026-09-01T09:20:00+05:30",
      "amount": 2480,
      "type": "Card",
      "tag": "grocery",
      "note": "Monthly stock-up",
      "person": "pavithra@example.com",
      "personName": "Pavithra",
      "by": "akshath@example.com",
      "byName": "Akshath",
      "created": "2026-09-01T03:50:12.004Z"
    }
  ]
}
```

Transactions are sorted newest-first on write *and* on read, so hand-editing a file on
GitHub won't scramble the display. Writes are read-modify-write against the file's SHA
and retry up to three times on a conflict, so logging from your phone and laptop at the
same time won't clobber.

## Two people, one ledger

Every record carries two identities, and they are deliberately different things:

- **`person`** — whose spend it is. Everything in History is keyed on this.
- **`by`** — who was signed in when it was recorded.

So one person can record a spend *for* another: the ₹1,100 lands in *her* ledger, and the row
reads *"recorded by Akshath"*. When the two match — the normal case — the byline is omitted.

**History always shows exactly one person, never a combined total.** It opens on the
signed-in user's own spend; the *Account holder* dropdown switches to someone else's. Category
and payment-method filters, the month-over-month delta and the daily average all operate
inside the selected person, so the comparison is always like with like. The dropdown
stays hidden while you are the only person in the data.

**The roster is not configured anywhere.** Names and emails come straight off the Google
credential at sign-in, and the *Account holder* picker is built from whoever appears in the
transaction data. A new person therefore shows up only *after* they have signed in and
recorded something themselves; until then the picker stays hidden, because a one-option
picker is just noise. The learned roster is cached in `localStorage` so it survives a
reload, and adding a third person needs no code change.

### Adding a person

Two separate gates, and both must be open:

1. **The sign-in gate** — the SHA-256 of their email must be in the `ALLOW` list in
   `index.html`. A rejected sign-in prints its own hash to the browser console; paste
   that in.
2. **Repo access** — they need their own fine-grained PAT with **Contents: Read and
   write** on `finance_tracking_data`, which means their GitHub account must be a
   collaborator on it. Passing the Google gate alone will leave them staring at a 404.

## Customising

Everything worth changing sits at the top of the tracker script in `index.html`:

```js
var STORE = { owner:"AkshRk", repo:"finance_tracking_data", folder:"transactions" };
var TYPES = ["Cash", "Online", "Card"];
var DEFAULT_TYPE = "Online";
var TAGS = [ {id:"grocery", label:"Grocery"}, … ];
var DEFAULT_TAG = "misc";
```

The allowed-email hashes and the OAuth client ID are in the sign-in gate script at the
bottom. A rejected sign-in prints its own SHA-256 to the browser console — that is the
easiest way to collect a hash for a new address.

## A note on the gate

The sign-in gate is a curtain, not a lock: GitHub Pages serves this file to anyone, so
the markup is on the visitor's machine before the check runs. It keeps a casual onlooker
out. The transaction data is never in this repository — it lives in the private data
repository, behind the token.
