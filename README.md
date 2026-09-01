# finance_tracking

A single-page expense ledger — sibling of [`fitness`](https://github.com/AkshRk/fitness).
One `index.html`, no build step. Google sign-in gate, and every transaction is committed
to a private data repo as JSON.

## Deploy

1. **Settings → Pages → Source: `main` / root.** It lands on
   `https://akshrk.github.io/finance_tracking/`.
2. **Nothing to change in Google Cloud Console.** The OAuth client is authorised per
   *origin*, and this page shares `https://akshrk.github.io` with the fitness page, so
   the same client ID and email allowlist already cover it.
3. **Open it and add a GitHub token** under *Settings → Connection settings* — unless
   you already added one on the fitness page, in which case this page picks it up
   automatically (same browser, same origin, same `localStorage`).

## Where the data goes

| | |
|---|---|
| Repo | `AkshRk/finance_tracking_data` (private) |
| Path | `transactions/YYYY-MM.json` — one file per month |
| Token | Fine-grained PAT, **Contents: Read and write** |

**The token must be scoped to this repo.** Fine-grained PATs grant access to named
repositories, so the token used by the fitness page (scoped to `fitness-data`) will
return a 404 here until you either add `finance_tracking_data` to its repository access
or issue a separate token. The page inherits the fitness token automatically — same
browser, same origin — which means a wrong-scope token can look like it's configured
when it isn't. If you see *"Can't see AkshRk/finance_tracking_data"*, that's this.

Repo, folder and token are all editable under Connection settings; the values above are
just the defaults compiled into the page.

Each transaction is one commit, so `git log` reads as a spending history:

```
₹340.50 · Restaurant · Online — Lunch with Ravi (2026-09-01)
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
      "by": "you@example.com",
      "created": "2026-09-01T03:50:12.004Z"
    }
  ]
}
```

Transactions are sorted newest-first on write *and* on read, so hand-editing a file on
GitHub won't scramble the display. Writes are read-modify-write against the file's SHA
and retry up to three times on a conflict, so logging from your phone and laptop at the
same time won't clobber.

## Customising

Everything worth changing sits at the top of the tracker script in `index.html`:

```js
var TYPES = ["Cash", "Online", "Card"];
var DEFAULT_TYPE = "Online";
var TAGS = [ {id:"grocery", …}, {id:"restaurant", …}, {id:"misc", …},
             {id:"charity", …}, {id:"lending", …} ];
var DEFAULT_TAG = "misc";
```

The allowed-email hashes and the OAuth client ID are in the sign-in gate script at the
bottom. A rejected sign-in prints its own SHA-256 to the browser console — that is the
easiest way to collect a hash for a new address.

## A note on the gate

The sign-in gate is a curtain, not a lock: GitHub Pages serves this file to anyone, so
the markup is on the visitor's machine before the check runs. It keeps a casual onlooker
out. The actual transaction data is never in this repo — it lives in the private data
repo, behind the token.
