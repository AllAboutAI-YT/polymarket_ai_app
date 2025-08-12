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

### 2) Get token IDs for the first market (example)

```bash
curl -s "https://gamma-api.polymarket.com/events/$EVENT_ID" \
| jq '.markets[0].clobTokenIds | fromjson'
```

Example output:

```json
[
  "103143573874045117971604635752039925340931165959446595394046689161646463222428",
  "21156766303400458094022561722221008224798395708803668102392410074584994172215"
]
```

### 3) Extract binary markets with Yes/No, token IDs, and prices

```bash
curl -s "https://gamma-api.polymarket.com/events/$EVENT_ID" \
| jq -r '
  [ .markets[]
    | select(.outcomes == ["Yes","No"])
    | {
        question: .question,
        yes_token: (.clobTokenIds | fromjson | .[0]),
        no_token:  (.clobTokenIds | fromjson | .[1]),
        yes_price: (.prices[0]),
        no_price:  (.prices[1])
      }
  ]
'
```

This produces a clean JSON array of relevant markets.

---

## Notes

* `clobTokenIds` is returned as a **stringified JSON array**; always run through `fromjson` before indexing.
* Prices are dollar-based and often used as implied probabilities.
* Use `jq` filters to sort or further transform the data.

---

**Example market output:**

```json
[
  {
    "question": "Will Bitcoin reach $127k August 11–17?",
    "yes_token": "103143573874045117971604635752039925340931165959446595394046689161646463222428",
    "no_token": "21156766303400458094022561722221008224798395708803668102392410074584994172215",
    "yes_price": 0.075,
    "no_price": 0.925
  }
]
```
