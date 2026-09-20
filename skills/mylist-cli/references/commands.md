# mylist — command and output reference

Everything here is `mylist <command> --help` plus the JSON shapes, so you can
plan a job without running the help first.

## Contents

- [Global flags and environment](#global-flags-and-environment)
- [auth](#auth)
- [me](#me)
- [list](#list)
- [item](#item)
- [tag](#tag)
- [search](#search)
- [api](#api)
- [JSON shapes](#json-shapes)
- [Output rules](#output-rules)

## Global flags and environment

| Flag / variable | Meaning |
| --- | --- |
| `--json` | Print the API response as JSON. Use this when parsing. |
| `--api <origin>` | Another deployment. Default `https://mylist.world`; the China deployment is a different origin and a different account. |
| `-h`, `--help` | Help for the command. |
| `MYLIST_TOKEN` | Use this token instead of the stored one. Never written to disk. |
| `MYLIST_API` | Same as `--api`. |
| `MYLIST_CONFIG_DIR` | Where `config.json` lives. Default `~/.config/mylist`. |

Exit codes: `0` ok · `1` request failed · `2` usage error · `4` not signed in.

## auth

```
mylist auth login [--with-token] [--api <origin>]
mylist auth logout
mylist auth status
mylist auth token
```

- `login` opens `<origin>/settings/cli?port=…&state=…` in a browser, waits
  up to five minutes for the page to hand a token back to `127.0.0.1`, checks
  it against `/me`, and writes `~/.config/mylist/config.json` (mode 0600).
- `login --with-token` reads a token from stdin (`mylist auth login
  --with-token < token.txt`). Tokens are made at mylist.world/settings/cli.
- `logout` revokes the stored token on the server, then deletes the file.
- `status` prints origin, account and token name; exit 4 when signed out.
  `--json` gives `{ api, user: { id, handle, displayName }, token: { source, id } }`.
- `token` prints the token, for handing to another tool.

## me

`mylist me [--json]` — the signed-in account: `id`, `handle`, `displayName`,
`email`, `region`, `role`.

## list

```
mylist list ls [--status <s>] [--search <q>] [--limit <n>] [--all]
mylist list view <id> [--pass-code <code>]
mylist list create --title <t> [--description <d>] [--style <key>]
mylist list edit <id> [--title <t>] [--description <d>] [--style <key>] [--cover <url>] [--background-color <#rrggbb>]
mylist list delete <id>... [--yes]
mylist list publish <id>
mylist list unpublish <id>
mylist list add <list-id> <item-id>... [--note <text>]
mylist list remove <list-id> <entry-or-item-id>...
```

- `ls` is the caller's own lists, any status, newest first. `--status` is one
  of `draft`, `in_review`, `rejected`, `published`. `--search` is a fuzzy
  title match. Default 20 rows; `--limit` up to any number (paged at 100);
  `--all` for everything. `mylist lists` is a shorthand for `mylist list ls`.
- `view` works on any published list, not only your own; entries are absent
  (`entries` missing from JSON, "locked" in text) on a paid or pass-coded
  list you cannot open. Table columns for entries: `#`, `ENTRY`, `ITEM`,
  `KIND`, `TITLE`, `NOTE`. The record includes the share `URL`.
- `create` makes a draft. `--style` is a template key, default `stack`.
- `edit`: an empty string clears `--description`, `--cover`,
  `--background-color`. `--title` cannot be empty.
- `add` appends one entry per item id, in the order given, with the same
  note on each if `--note` is passed. Prints the entry count when piped.
- `remove` takes entry ids or item ids and rewrites the list's entries
  without them. Order of the rest is kept.
- `delete` archives; `--yes` is required when stdin/stdout is not a terminal.
- `publish` publishes, or submits for review where the deployment reviews
  first (status becomes `in_review`). `unpublish` returns to `draft`.

Table columns: `ID`, `STATUS`, `TITLE`, `UPDATED`.

## item

```
mylist item ls [--kind <k>] [--search <q>] [--tag <name-or-id>] [--limit <n>] [--all]
mylist item view <id>
mylist item create --kind <k> --title <t> [--subtitle <s>] [--url <u>] [--image <url>]
                   [--note <text>] [--category <c>] [--rating <1-5>] [--tags <a,b>]
mylist item edit <id> [--kind <k>] [--title <t>] [--subtitle <s>] [--url <u>] [--image <url>]
                      [--note <text>] [--category <c>] [--rating <1-5|none>] [--tags <a,b>]
mylist item delete <id>... [--yes]
```

- Kinds: `music`, `podcast`, `film`, `game`, `book`, `product`, `image`,
  `place`, `app`, `people`, `misc`.
- `ls` is the caller's collection, in the owner's order. `--tag` accepts a
  tag name (case-insensitive) or id. `mylist items` is a shorthand for `ls`.
- `create` needs `--kind` and `--title`. `--rating` and `--tags` are applied
  in follow-up calls, so the printed record already has them.
- `edit`: empty string clears `--subtitle`, `--url`, `--image`, `--note`,
  `--category`. `--rating none` clears the rating. `--tags` replaces the
  whole set and `--tags ""` clears it.
- `delete` archives; same `--yes` rule as lists.

Table columns: `ID`, `KIND`, `TITLE`, `SUBTITLE`, `RATING`.

## tag

`mylist tag ls [--json]` — the caller's tags: `id`, `name`, `color`,
`itemCount`. Table columns: `ID`, `NAME`, `ITEMS`.

## search

`mylist search <query> [--limit <n>]` — `--limit` is per bucket (default
12, max 50). JSON has four arrays: `items` (your things), `ownLists` (your
lists, drafts included), `lists` (other people's published lists), `users`.
Piped text is one flat stream with the bucket in column 1: `item`, `list` or
`user`. Works signed out too, with the first two buckets empty.

## api

```
mylist api <path> [-X <method>] [-f key=value]... [-F key=json]... [--input <file|->]
```

- `path` is relative to `/api/v1`, starts with `/`, may carry a query string.
- `-X` is `GET` (default), `POST`, `PUT`, `PATCH`, `DELETE`.
- `-f` adds a string field; `-F` parses the value as JSON. For GET/DELETE
  they become query parameters, otherwise the JSON body.
- `--input` reads a JSON body from a file or `-` (stdin); `-f`/`-F` merge
  over it.
- Prints the JSON response; a 204 prints nothing.

Routes worth knowing (all under `/api/v1`, all need the token unless noted):

| Route | What |
| --- | --- |
| `GET /lists/:id/settings` · `PATCH` | Publish settings: `allowJumping`, `allowMessages`, `showRatings`, `showDescriptions`, `featurable`, `showOnProfile`, `hideLogo`, `tippingEnabled`, `priceMinor`, `passCode`. |
| `PUT /lists/:id/entries` | Replace the whole ordered entry set: `{ entries: [{ id, itemId, position, note }] }`. How reordering is done. |
| `POST /lists/:id/move` | Reorder lists on the shelf: `{ afterId?, beforeId? }`. |
| `POST /items/:id/move` | Reorder a thing in the collection: `{ afterId?, beforeId? }`. |
| `GET /item-categories` · `POST` · `PATCH /:id` · `DELETE /:id` | The user's tabs (`name`, `kind`). |
| `GET /stats/overview` · `GET /stats/lists/:id` | Creator numbers. |
| `GET /lists/:id/comments` · `POST` | Comments on a list, when the list allows them. |
| `POST /lists/:id/like` `{ liked }` · `POST /lists/:id/favorite` `{ favorited }` | Idempotent toggles. |
| `GET /users/:handle` · `GET /users/:handle/lists` | Somebody's public page (no token needed). |
| `GET /notifications` · `POST /notifications/read` | Inbox. |
| `PATCH /me` | `displayName`, `bio`, `avatarUrl`, `coverImageUrl`, `handle`. |

## JSON shapes

`List` (from `list ls`, `list create`, `list edit`, `publish`, `unpublish`):

```json
{
  "id": "…", "title": "…", "description": null,
  "owner": { "id": "…", "handle": "alice", "displayName": "Alice", "avatarUrl": null, "coverImageUrl": null, "bio": null, "createdAt": "…" },
  "coverImageUrl": null, "coverColor": null, "backgroundColor": null, "backgroundImageUrl": null,
  "styleKey": "stack", "status": "draft",
  "render": { "allowJumping": true, "allowMessages": false, "showRatings": false, "showDescriptions": true, "hideLogo": false },
  "priceMinor": 0, "unlocked": true,
  "counts": { "views": 0, "likes": 0, "comments": 0 },
  "publishedAt": null, "createdAt": "2026-09-20T10:00:00.000Z", "updatedAt": "…"
}
```

`list view` adds `entries` (absent when locked):

```json
"entries": [ { "id": "ENTRY_ID", "item": { …Item… }, "position": 1, "note": null, "imageUrl": null } ]
```

`Item` (from `item ls`, `item view`, `item create`, `item edit`, and inside entries):

```json
{
  "id": "…", "kind": "book", "title": "Dune", "subtitle": "Frank Herbert",
  "url": null, "imageUrl": null, "imageColor": null, "cutoutUrl": null,
  "note": null, "rating": 5, "category": null, "categoryId": null,
  "position": 1024, "tagIds": ["…"], "source": "manual",
  "metadata": { "isbn": "…", "year": 1965 },
  "latitude": null, "longitude": null,
  "createdAt": "…", "updatedAt": "…", "archivedAt": null
}
```

`tagIds` are ids; names come from `mylist tag ls`. `metadata` keys depend on
kind: `album`/`year` (music), `isbn` (book), `bundleId` (app), `address`
(place), `price` (product), `sourceUrl` (where it was found).

`Tag`: `{ "id", "name", "color", "itemCount" }`.

`Me`: `{ "id", "handle", "displayName", "avatarUrl", "coverImageUrl", "bio", "createdAt", "region", "role", "email", "phone" }`.

## Output rules

- Terminal: header row, aligned columns, long titles cut with `…`.
- Piped (any time stdout is not a TTY): tab-separated, no header, nothing
  cut. `create` prints the new id only; `publish`/`unpublish` print the new
  status only; `delete` prints each id.
- `--json`: the payload, pretty-printed. `ls` commands print a JSON array
  (pages already merged); `view`/`create`/`edit` print the record.
- Errors: stderr, `mylist: …`, plus one indented line per field error.
