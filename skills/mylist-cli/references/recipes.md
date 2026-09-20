# Recipes

Multi-step jobs, written the way they should be done: find ids first, reuse
what exists, write with `--json` so the next step has what it needs, confirm
before anything the user cannot undo.

## Build a list from things already saved

"Make a list of my five-star films."

```bash
mylist item ls --kind film --all --json | jq -r '.[] | select(.rating == 5) | .id' > ids.txt
LIST=$(mylist list create --title "Films I'd watch again" --json | jq -r .id)
xargs mylist list add "$LIST" < ids.txt
mylist list view "$LIST"
```

Show the user the result and the share URL from `list view`; ask before
`mylist list publish "$LIST"`.

## Add new things, then put them on a list

"Add these three books to my reading list: …"

For each title, check the collection first so nothing is duplicated:

```bash
mylist item ls --search "Piranesi" --json | jq '.[] | {id, title, subtitle}'
```

Create only what is missing (`--subtitle` is the author, artist or maker;
`--url` a link if the user gave one), then add the ids:

```bash
ID=$(mylist item create --kind book --title "Piranesi" --subtitle "Susanna Clarke" --json | jq -r .id)
mylist list add LIST_ID "$ID"
```

Find the list with `mylist list ls --search "reading" --json`; if there is
more than one match, ask which.

## Bulk tag

"Tag everything by Studio Ghibli as 'ghibli'."

`--tags` replaces, so read the current tags first and add to them:

```bash
mylist tag ls --json > tags.json
mylist item ls --search "Ghibli" --all --json | jq -c '.[] | {id, tagIds}' | while read -r row; do
  id=$(jq -r .id <<<"$row")
  names=$(jq -r --slurpfile t tags.json '[.tagIds[] as $i | $t[0][] | select(.id == $i) | .name] + ["ghibli"] | unique | join(",")' <<<"$row")
  mylist item edit "$id" --tags "$names"
done
```

## Reorder a list

Entries are replaced as a whole set. Read them, rewrite `position`, put them
back:

```bash
mylist list view LIST_ID --json | jq '{entries: [.entries | sort_by(.item.title) | to_entries[] | {id: .value.id, itemId: .value.item.id, position: (.key + 1), note: .value.note}]}' \
  | mylist api /lists/LIST_ID/entries -X PUT --input -
```

## Export the collection to CSV

```bash
mylist item ls --all --json | jq -r '["id","kind","title","subtitle","rating","url"], (.[] | [.id, .kind, .title, .subtitle, .rating, .url]) | @csv' > collection.csv
```

Piped table output is already TSV: `mylist item ls --all > things.tsv`.

## Find duplicates

```bash
mylist item ls --all --json | jq -r 'group_by(.kind, (.title | ascii_downcase)) | map(select(length > 1) | map(.id + "  " + .title) | join("\n")) | .[]'
```

Show the user the groups and let them pick what to delete; then
`mylist item delete ID... --yes`. Deleting a thing removes it from every
list it was on.

## Read someone else's published list

`list view` works on any published list id, and `search` finds them:

```bash
mylist search "best albums 2025" --json | jq '.lists[] | {id, title, owner: .owner.handle}'
mylist list view THEIR_LIST_ID --json
```

To copy their things into your own collection, create each with
`item create` from the entry's `item` fields (`kind`, `title`, `subtitle`,
`url`, `imageUrl`).

## Change publish settings

```bash
mylist api /lists/LIST_ID/settings
mylist api /lists/LIST_ID/settings -X PATCH -F showRatings=true -F allowMessages=true
```

`priceMinor` and `passCode` gate readers; leave those to the user unless
asked.
