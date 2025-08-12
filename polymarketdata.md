# Polymarket: From event **slug** → data (quick guide)

This guide shows how to turn a Polymarket **event slug** (e.g., `what-price-will-bitcoin-hit-august-11-17`) into structured data for each binary market: question, token IDs, and prices. It uses `curl` + `jq` against the public Gamma API.

> Requirements: `curl`, `jq` (macOS/Linux; on Windows, use WSL or Git Bash).

---

## 1) Resolve **slug → event id**

```bash
SLUG="what-price-will-bitcoin-hit-august-11-17"
curl -s "https://gamma-api.polymarket.com/events?slug=$SLUG" \
  | jq '[.[] | {id, ticker, slug, title}]'
```

To capture just the id:

```bash
EVENT_ID=$(curl -s "https://gamma-api.polymarket.com/events?slug=$SLUG" | jq -r '.[0].id')
echo "$EVENT_ID"
```

## 2) Get the **event** payload and inspect fields

```bash
curl -s "https://gamma-api.polymarket.com/events/$EVENT_ID" | jq '.' | less
```

Useful fields inside `.markets[]`:

* `outcomes` → expect `["Yes","No"]` for binary markets
* `prices` → `[yes, no]`
* `clobTokenIds` → JSON **as a string**; must run `| fromjson`
* `question` → label for the strike/bracket

## 3) Extract **binary** markets with tokens and prices

```bash
curl -s "https://gamma-api.polymarket.com/events/$EVENT_ID" \
| jq -r '
  [ .markets[]
    | select(.outcomes == ["Yes","No"])  # keep binary markets
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

---

## One‑liner (copy/paste)

> Replace `SLUG` if needed. Prints an array of objects with `question`, `yes_token`, `no_token`, `yes_price`, `no_price`.

```bash
SLUG="what-price-will-bitcoin-hit-august-11-17" && EVENT_ID=$(curl -s "https://gamma-api.polymarket.com/events?slug=$SLUG" | jq -r '.[0].id') && curl -s "https://gamma-api.polymarket.com/events/$EVENT_ID" | jq -r '[ .markets[] | select(.outcomes == ["Yes","No"]) | {question: .question, yes_token: (.clobTokenIds | fromjson | .[0]), no_token: (.clobTokenIds | fromjson | .[1]), yes_price: (.prices[0]), no_price: (.prices[1])} ]'
```

### Flat, TSV‑friendly table

```bash
SLUG="what-price-will-bitcoin-hit-august-11-17" && EVENT_ID=$(curl -s "https://gamma-api.polymarket.com/events?slug=$SLUG" | jq -r '.[0].id') && curl -s "https://gamma-api.polymarket.com/events/$EVENT_ID" | jq -r '["question","yes_token","no_token","yes_price","no_price"], ( .markets[] | select(.outcomes == ["Yes","No"]) | [ .question, (.clobTokenIds | fromjson | .[0]), (.clobTokenIds | fromjson | .[1]), (.prices[0]), (.prices[1]) ] ) | @tsv'
```

Pipe through `column` for prettier CLI output:

```bash
... | column -t
```

---

## Notes

* Always run `| fromjson` on `clobTokenIds` before indexing it.
* Prices are in dollars for Yes/No shares; treat them as rough implied probabilities (subject to fees/liquidity).
* If a slug returns multiple events, filter by `.title`, `.startDate`, or other metadata before selecting `[0]`.
