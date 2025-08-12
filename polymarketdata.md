# Polymarket: slug → TSV of brackets, token IDs, and prices

Use these **two commands** exactly. They print a tab‑separated table with a header and one line per bracket.

---

## 1) Resolve slug → event id

```bash
SLUG="what-price-will-bitcoin-hit-august-11-17"
EVENT_ID=$(curl -s "https://gamma-api.polymarket.com/events?slug=$SLUG" | jq -r '.[0].id')
echo "$EVENT_ID"
```

Expected: prints something like `37057`.

## 2) Fetch event and emit TSV (sorted by parsed bracket)

```bash
EVENT=$EVENT_ID
curl -s "https://gamma-api.polymarket.com/events/$EVENT" \
| jq -r '
  def prices: (.outcomePrices | fromjson | map(tonumber));
  def tokens: (.clobTokenIds   | fromjson);
  def outs:   (.outcomes       | fromjson);
  def skey(s):
    if   (s|test("(?i)<\\$?([0-9]+)k"))                                      then (s|capture("(?i)<\\$?(?<n>[0-9]+)k")                              | {kind:0, lo:(.n|tonumber), hi:0})
    elif (s|test("(?i)(less|under|below)\\s+\\$?([0-9]+)k"))                 then (s|capture("(?i)(?:less|under|below)\\s+\\$?(?<n>[0-9]+)k")     | {kind:0, lo:(.n|tonumber), hi:0})
    elif (s|test("(?i)between\\s+\\$?([0-9]+)k\\s+and\\s+\\$?([0-9]+)k")) then (s|capture("(?i)between\\s+\\$?(?<a>[0-9]+)k\\s+and\\s+\\$?(?<b>[0-9]+)k") | {kind:1, lo:(.a|tonumber), hi:(.b|tonumber)})
    elif (s|test("(?i)\\$?([0-9]+)k\\s*[–-]\\s*\\$?([0-9]+)k"))            then (s|capture("(?i)\\$?(?<a>[0-9]+)k\\s*[–-]\\s*\\$?(?<b>[0-9]+)k") | {kind:1, lo:(.a|tonumber), hi:(.b|tonumber)})
    elif (s|test("(?i)>\\$?([0-9]+)k"))                                       then (s|capture("(?i)>\\$?(?<n>[0-9]+)k")                             | {kind:2, lo:(.n|tonumber), hi:0})
    elif (s|test("(?i)(greater|over|above)\\s+\\$?([0-9]+)k"))               then (s|capture("(?i)(?:greater|over|above)\\s+\\$?(?<n>[0-9]+)k")   | {kind:2, lo:(.n|tonumber), hi:0})
    else {kind:9, lo:0, hi:0} end;

  [ .markets[]
    | select(outs == ["Yes","No"])                # only binary Yes/No
    | {bracket: .question, p: prices, t: tokens}
    | select((.p|length)==2 and (.t|length)==2)      # ensure 2 prices & 2 tokens
    | . + {key: skey(.bracket)}
    | . + {yes_token: .t[0], no_token: .t[1], yes_price: .p[0], no_price: .p[1]}
    | del(.p,.t)
  ]
  | sort_by(.key.kind, .key.lo, .key.hi)
  | map(del(.key))
  | ( ["bracket","yes_token","no_token","yes_price","no_price"] | @tsv ),
    ( .[] | [ .bracket, .yes_token, .no_token, (.yes_price|tostring), (.no_price|tostring) ] | @tsv )
'
```

### Notes

* This version uses `.outcomePrices` (stringified) → `fromjson` → numeric, and matches only `Yes/No` markets.
* Output is **TSV**: first line is a header, following lines are data.
* If your event uses different outcome labels or structures, we can make the selector more permissive (e.g., drop the `outs == ["Yes","No"]` check).
