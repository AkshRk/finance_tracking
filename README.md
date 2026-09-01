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
3. **Open it and add a GitHub token** under *Settings → Connection settings*.

This app is self-contained: it keeps its own token and its own sign-in session, and it
never reads or writes `fitness-data`. Nothing is shared with the fitness page beyond the
OAuth client ID and the email allowlist.

## Where the data goes

| | |
|---|---|
| Repo | `AkshRk/finance_tracking_data` (private) |
| Path | `transactions/YYYY-MM.json` — one file per month |
| Token | Fine-grained PAT, **Contents: Read and write** |

**The token must be scoped to this repo.** Fine-grained PATs grant access to named
repositories, so a token issued for `fitness-data` will return a 404 here. Either add
`finance_tracking_data` to that token's repository access, or issue a separate one. If
you see *"Can't see AkshRk/finance_tracking_data"*, that's this.

Repo, folder and token are all editable under Connection settings; the values above are
just the defaults compiled into the page.

Each transaction is one commit, so `git log` reads as a spending history:

```
₹340.50 · Akshath · Restaurant · Online — Lunch with Ravi (2026-09-01)
₹1,100.00 · Pavithra · Charity · Online — Temple donation (2026-09-04) [logged by Akshath]
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
- **`by`** — who was signed in when it was logged.

So Akshath can log a spend *for* Pavithra: the ₹1,100 lands in *her* ledger, and the row
reads *"added by Akshath"*. When the two match — the normal case — the byline is omitted.

**History always shows exactly one person, never a combined total.** It opens on the
signed-in user's own spend; the *Whose spend* dropdown switches to someone else's. Tag
and payment-type filters, the month-over-month delta and the daily average all operate
inside the selected person, so the comparison is always like with like. The dropdown
stays hidden while you are the only person in the data.

**The roster is not configured anywhere.** Names and emails come straight off the Google
credential at sign-in, and the *Whose spend* picker is built from whoever appears in the
transaction data. A new person therefore shows up only *after* they have signed in and
logged something themselves; until then the picker stays hidden, because a one-option
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
