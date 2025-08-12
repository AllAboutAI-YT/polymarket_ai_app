# From Polymarket event slug → data

This quick guide shows how to go from a **slug** (e.g., `what-price-will-bitcoin-hit-august-11-17`) to the key **markets**, **token IDs**, and **prices** using `curl` and `jq`.

---

## Step-by-step guide

### 1) Resolve the slug to an event id

```bash
SLUG="what-price-will-bitcoin-hit-august-11-17"
EVENT_ID=$(curl -s "https://gamma-api.polymarket.com/events?slug=$SLUG" | jq -r '.[0].id')
echo "$EVENT_ID"
```

This returns the event ID needed to query the event details.

### 2) Get the event payload

```bash
curl -s "https://gamma-api.polymarket.com/events/$EVENT_ID" | jq '.' | less
```

Inspect the `.markets[]` array to see all markets, outcomes, token IDs, and prices.

### 3) Extract brackets, token IDs, and prices (sorted & NDJSON)

This version matches what you showed: it parses the strike from the question text, sorts the brackets, and prints **one JSON object per line** (NDJSON).

```bash
curl -s "https://gamma-api.polymarket.com/events/$EVENT_ID" \
| jq -r '
  def skey(s):
    if (s|test("(?i)\$?([0-9]+)k\s*[–-]\s*\$?([0-9]+)k")) then
      (s|capture("(?i)\$?(?<a>[0-9]+)k\s*[–-]\s*\$?(?<b>[0-9]+)k")|{kind:1, lo:(.a|tonumber), hi:(.b|tonumber)})
    elif (s|test("(?i)>\$?([0-9]+)k")) then
      (s|capture("(?i)>\$?(?<n>[0-9]+)k")|{kind:2, lo:(.n|tonumber), hi:0})
    elif (s|test("(?i)(greater|over|above)\s+\$?([0-9]+)k")) then
      (s|capture("(?i)(?:greater|over|above)\s+\$?(?<n>[0-9]+)k")|{kind:2, lo:(.n|tonumber), hi:0})
    elif (s|test("(?i)\$([0-9]{2,6})")) then
      (s|capture("(?i)\$(?<n>[0-9]{2,6})")|{kind:0, lo:(.n|tonumber), hi:(.n|tonumber)})
    else {kind:9, lo:0, hi:0} end;

  [ .markets[]
    | {bracket: .question,
       p: .prices,
       t: (.clobTokenIds | fromjson),
       outs: .outcomes}
    | select((.outs|length)==2 and (.p|length)==2 and (.t|length)==2)
    | . + {key: skey(.bracket)}
    | . + {yes_token: .t[0], no_token: .t[1], yes_price: .p[0], no_price: .p[1]}
    | del(.p,.t,.outs)
  ]
  | sort_by(.key.kind, .key.lo, .key.hi)
  | map(del(.key))
  | .[]
'
```

**Example output:**

```json
{
  "bracket": "Will Bitcoin reach $127k August 11–17?",
  "yes_token": "103143573874045117971604635752039925340931165959446595394046689161646463222428",
  "no_token": "21156766303400458094022561722221008224798395708803668102392410074584994172215",
  "yes_price": 0.075,
  "no_price": 0.925
}
```

---

## Notes

* This method works even if `.outcomes` is not exactly `["Yes","No"]`.
* `clobTokenIds` is returned as a **stringified JSON array**; always run through `fromjson` before indexing.
* The sort key logic can be adjusted to match different strike formats.
