---
name: mylist-cli
description: Read, search, create, edit, tag, reorder, publish and delete a person's lists and saved things on MyList (mylist.world) through the `mylist` command line, and script or automate their account. Use this whenever the user mentions MyList, mylist.world, "my lists", their collection or saved things there, wants a list made from what they have saved, wants things added to or taken off a list, or wants to bulk-edit, export or query MyList content — even if they never say "CLI" or "command line". Also use it when a task needs the user's MyList data as input (what films they rated, which lists are drafts).
---

# MyList from the command line

MyList is where people keep lists of things they love and share them. The
`mylist` command is the whole account in a terminal: it reads and writes the
same data the website and the app do, through the same API, as the person who
signed in. This skill is how to drive it without guessing.

## The two nouns

- **Things** (the API calls them *items*) are what the person has saved: a
  book, a film, an album, a place, a product. Each has a `kind` (`music`,
  `podcast`, `film`, `game`, `book`, `product`, `image`, `place`, `app`,
  `people`, `misc`), a title, an optional subtitle/url/image/note, a 1–5
  `rating`, and tags. Things live in one collection per account.
- **Lists** are ordered selections of things, with a title, a description, a
  template (`styleKey`), and a `status`: `draft`, `in_review`, `rejected` or
  `published`. A thing on a list is an **entry** — it has its own entry id,
  a position, and an optional per-list note. The same thing can sit on many
  lists.

So "add *Dune* to my sci-fi list" is two steps: find (or create) the thing,
then add its id to the list. Deleting an entry does not delete the thing.

## Before anything else

```bash
mylist --version || npm install -g mylist-cli
mylist auth status            # exit 4 = not signed in
```

If not signed in, ask the user to run `mylist auth login` in their own
terminal: it opens mylist.world in their browser, they click Authorize, and
the token lands on their machine. You can run it for them when you share
their machine and browser, but the click is theirs. Where no browser can
open (a sandbox, SSH), they create a token at
<https://mylist.world/settings/cli> and either run
`mylist auth login --with-token` and paste it, or hand it to you as the
`MYLIST_TOKEN` environment variable. Never ask for the token in chat when a
login on their side would do; a token is account access until revoked.

## Reading

Always add `--json` when you are going to parse. It prints the API payload
as is, with stable field names (`id`, `title`, `status`, `kind`, `rating`,
`entries[].item`…). Without it, piped output is tab-separated with no header
and terminal output is a table — fine for showing the user, wrong for
parsing.

```bash
mylist list ls --json                          # own lists, newest first (20)
mylist list ls --status draft --search "sci" --json
mylist list ls --all --json                    # every page
mylist list view LIST_ID --json                # one list with its entries
mylist item ls --kind book --json
mylist item ls --search "dune" --json
mylist item ls --tag favourites --all --json   # tag by name or id
mylist item view ITEM_ID --json
mylist tag ls --json
mylist search "dune" --json                    # own things + lists, then everyone's
mylist me --json
```

Ids are opaque strings. Get them from `ls`, `view` or `search`; never invent
or shorten one. `--limit N` caps rows (the server pages at 100, the CLI
walks cursors for you); `--all` fetches everything.

## Writing

```bash
mylist item create --kind book --title "Dune" --subtitle "Frank Herbert" \
  --rating 5 --tags "sci-fi,favourites" --json
mylist item edit ITEM_ID --note "Re-read in 2026" --rating 4
mylist item edit ITEM_ID --tags "sci-fi"          # replaces the whole set
mylist item edit ITEM_ID --note ""                # empty string clears
mylist item edit ITEM_ID --rating none

mylist list create --title "Books I loved" --description "…" --json
mylist list edit LIST_ID --title "Renamed"
mylist list add LIST_ID ITEM_ID ITEM_ID2 --note "why it's here"
mylist list remove LIST_ID ENTRY_OR_ITEM_ID
mylist list publish LIST_ID
mylist list unpublish LIST_ID
```

Rules that keep writes safe:

- `edit` is partial. Only the flags you pass change; nothing else is touched.
- An empty string clears a nullable field. Omitting the flag leaves it.
- `--tags` replaces the set; names that do not exist yet are created.
- Creating a thing that may already exist: search first (`item ls --search`)
  and reuse the id. Duplicates are the most common mistake.
- `publish` makes the list visible to everyone (or queues it for review).
  Confirm with the user before publishing or unpublishing on their behalf.
- `delete` archives. It refuses without `--yes` when not on a terminal, so
  from a script it is always `mylist list delete ID --yes`. Confirm with the
  user before deleting anything; there is no undo from the CLI.
- Piped, `create` prints only the new id. With `--json` it prints the record.

## When something fails

Errors go to stderr as `mylist: <message>`; exit codes are `1` request
failed, `2` wrong usage (the message says what), `4` not signed in. A `4`
mid-task means the token was revoked: tell the user, do not retry. A `1`
with field errors lists the fields under the message.

## Anything else

`mylist api` calls any route under `/api/v1` and prints the JSON —
settings, stats, comments, moving a thing between two others:

```bash
mylist api /lists/LIST_ID/settings
mylist api /lists/LIST_ID/settings -X PATCH -F showRatings=true
mylist api /stats/lists/LIST_ID
mylist api /items/ITEM_ID/move -X POST -f afterId=OTHER_ID
```

`-f` sends a string, `-F` parses JSON (numbers, booleans, null), `--input -`
reads a body from stdin.

Read [references/commands.md](references/commands.md) for every flag and the
JSON shapes, and [references/recipes.md](references/recipes.md) for
multi-step jobs (build a list from saved things, bulk tag, export to CSV,
find duplicates).
