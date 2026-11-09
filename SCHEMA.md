# Schema: November 3, 2026 US general election feed recordings

These are the live results and market feeds as they were received on election night,
from 2026-11-03 21:10 UTC to 2026-11-04 19:03 UTC (results feeds: 22:30 to 10:00 UTC).
They are **not certified results**. Every publisher error, revision and later correction
is included as it was received (see [Known data issues](#known-data-issues)).

| Path | Source | Contents |
|---|---|---|
| `nyt/` | New York Times election results JSON | Senate, Governor and House payloads. Every version is stored, losslessly encoded as keyframes plus patches, with one file per UTC hour. |
| `ddhq/` | Decision Desk HQ (DDHQ) public results API | Race listings and per-race county results, stored exactly as captured. |
| `kalshi/` | Kalshi exchange order books | Top-of-book quotes and full-depth ladders, one file per UTC hour per kind. |
| `normalized/` | Built from the NYT, DDHQ and Tennessee SoS feeds | One record per county or district update, with D/R votes, expected vote and completion. |

## Conventions

- **Format.** Every data file is gzip-compressed JSON Lines (`*.jsonl.gz`, one JSON object
  per line). The index files (`*.json`) are plain JSON.
- **`t`.** Unix time in seconds (float, UTC) at which the recorder received or wrote the record.
  It is not the publisher's timestamp. On this date ET = UTC−5 and CT = UTC−6, so for example
  `2026-11-04T02:00Z` = 9:00 pm ET = 8:00 pm CT.
- **Hourly files.** A file named `YYYY-MM-DDTHHZ.jsonl.gz` holds the records whose `t` falls in
  that UTC hour. To get the full stream, read the files in name order.
- **Race ids.** The DDHQ race keys, Kalshi `market` values and normalized `race_id` share one
  scheme:
  - `ST-SEN` (Senate), `ST-GOV` (Governor), `ST-NN` (House district, two digits; at-large
    seats are `-01`, e.g. `AK-01`);
  - `US-SENATE-CONTROL` and `US-HOUSE-CONTROL` (chamber control).
- **Party sides.** "D" is the Democratic candidate. In NE-SEN and ID-SEN no Democrat ran, and
  the independents (Osborn, Achilles) take the D slot in `normalized/` and the `I` outcome in
  `kalshi/`.

---

## nyt/

Three NYT payloads were polled about every 30 seconds, with conditional GETs. A version was
recorded only when the payload changed: 3,797 versions were recorded, and 342 polls that
returned unchanged data were not.

| `key` | URL | Versions |
|---|---|---|
| `sen-county-data/2026-11-03.json` | `https://static01.nyt.com/elections-assets/pages/data/feeds/67880e0c-9bb4-497c-ac46-cac705cf6270/sen-county-data/2026-11-03.json` | 1,253 |
| `gov-county-data/2026-11-03.json` | `https://static01.nyt.com/elections-assets/pages/data/feeds/11093529-3f1d-454e-aa6e-e63660cac1e7/gov-county-data/2026-11-03.json` | 1,233 |
| `2026-11-03/results-house.json` | `https://static01.nyt.com/elections-assets/pages/data/2026-11-03/results-house.json` | 1,311 |

Coverage runs from 2026-11-03T22:30:01Z to 2026-11-04T09:59:50Z. Stored in full, every
version would come to 24.8 GB of JSON (2.1 GB gzipped).

### Encoding (lossless)

Each line of `nyt/YYYY-MM-DDTHHZ.jsonl.gz` is one of two record types:

```
keyframe: {"t": <float>, "key": <str>, "v": <int>, "full": <payload>}
patch:    {"t": <float>, "key": <str>, "v": <int>, "ops": [[<path>, <value>], ...]}
```

- **`v`** is the version number for that key, starting at 0 and running across the whole night.
- **Keyframes.** The first version of every key in each hourly file is a keyframe, so each
  file can be decoded on its own.
- **Patches.** A patch turns the previous version of the same key into this version. Each op
  is a `path`, a list of object keys (strings) and array indexes (integers) from the payload
  root, and a `value` that **replaces** the element at that path. Apply the ops in order.
  There are no insert or delete ops: when an object's key set or key order changes, or an
  array's length changes, the whole object or array is replaced. An empty path replaces the
  whole payload.
- **Byte-exact.** Serialize a reconstructed payload with
  `json.dumps(payload, separators=(",", ":"))` (Python, ASCII-escaped). The output is
  byte-identical to the payload as captured; this was verified for all 3,797 versions.

Reconstruction:

```python
import gzip, glob, json
state = {}
for fn in sorted(glob.glob("nyt/*.jsonl.gz")):
    for line in gzip.open(fn, "rt"):
        r = json.loads(line)
        if "full" in r:
            state[r["key"]] = r["full"]
        else:
            for path, value in r["ops"]:
                if not path:
                    state[r["key"]] = value
                    continue
                o = state[r["key"]]
                for p in path[:-1]:
                    o = o[p]
                o[path[-1]] = value
        # state[r["key"]] is now the payload as received at r["t"]
        # (copy it before keeping it: later patches mutate it in place)
```

### Payload structure (NYT's format, main fields)

- **Senate and Governor payloads:** `{id, slug, electionDate, data: [race, ...]}`, with one
  race per state contest (35 Senate, 36 Governor).
- **House payload:** `{races: [race, ...], partyControlData, partyControlTimeseriesData,
  pollClosingTimes, resultsMeta, pageConfig, composablePageLayout, promos, raceInsights,
  demographicSentences}`. `partyControlData` holds NYT's chamber seat tallies.

**`race`**

| Field | Meaning |
|---|---|
| `nyt_id` | Race id, `ST-G-O-2026-11-03`. O is S (Senate), G (Governor) or H (House). |
| `office`, `seat`, `seat_number` | Office name and seat. |
| `race_call_status` | Call status, e.g. `too_early`, `called`. |
| `outcome` | `won` (candidate ids), `called_by` (e.g. `ap`), `winner_accepted_at`. |
| `candidate_metadata` | Keyed by candidate id; holds `first_name`, `last_name`, `party.nyt_id` (`DEM`, `GOP`, …), `incumbent`, `ballot_order`. |
| `top_reporting_unit` | Totals for the whole state or district. |
| `reporting_units` | List of units: state, then counties (or towns or other subunits). |

**Reporting unit**

| Field | Meaning |
|---|---|
| `nyt_id` | Unit id, e.g. `GA-county-13-121`. |
| `name`, `fips_state`, `fips_county`, `fips_suffix` | Unit name and FIPS codes. |
| `level` | `state`, `county`, `district`, … |
| `total_votes` | Votes counted, all candidates. |
| `total_expected_vote` | NYT's estimate of the unit's final total vote. |
| `expected_vote_by_vote_type` | Expected vote split as `total`, `advance` (early in person plus mail), `advance_in_person`, `absentee_mail`, `election_day_in_person`, `provisional`. |
| `eevp` | Estimated expected vote counted, in **percent** (0–100). It never exceeds `eevp_max`. |
| `eevp_by_vote_type` | The same estimate per vote type. |
| `eevp_max` | Cap on `eevp`; 99 on this night. |
| `precincts_reporting`, `precincts_total` | Precinct counts. |
| `candidates[]` | `nyt_id`, `votes` (`total` plus the same vote-type split as above), `leader`, `nyt_model_estimates`. |
| `historical_comparison_race_data` | Prior results, e.g. the 2024 presidential `votes`, `pct_dem`, `pct_rep` and `margin`. |

---

## ddhq/

Base URL: `https://votes.decisiondeskhq.com`. Lines are unchanged from the capture, as
`{"t", "kind", "key", "body"}`. A record was written only when its body differed from the
previous record with the same `kind` and `key`.

### `ddhq_listing.jsonl.gz`: race listings

1,810 records, 2026-11-03T22:30:07Z to 2026-11-04T09:59:43Z.

- **Source.** `GET /api/filters/races?raceDates=2026-11-03&officeIds=<id>&limit=500&offset=<n>`,
  polled about every 15 seconds per office. All pages are concatenated into one body.
- **`key`.** DDHQ office id: `"4"` = Senate, `"2"` = Governor, `"3"` = House.
- **`body`.** A list of rows, each reduced to `{id, lastUpdated, totalVotes, toplineResults,
  turnout}`:
  - `id` is the DDHQ race id (see `races.json`).
  - `toplineResults[]` has candidate `candId`, `firstName`, `lastName`, `votes`, `early_votes`,
    `absentee_votes`, `election_day_votes`, `provisional_votes`, `percent`, `winner`,
    `called_at`, `party{id,name,abbrev}`, `incumbent`.
  - `turnout` has `pre_election_low/mid/high`, `estimated_votes_low/mid/high`, `precincts_in`
    and `precincts_tot`.

### `ddhq_races.jsonl.gz`: per-race county results

12,518 records for 486 races, 2026-11-03T22:58:00Z to 2026-11-04T09:58:06Z.

- **Source.** `GET /api/races/<ddhq_id>`. A race was fetched when the listing showed it had
  changed, and on a periodic sweep of races that were still open, from around the state's poll
  close.
- **`key`.** Race id, e.g. `GA-SEN`.
- **`body`.** `{lastUpdated, countyResults, toplineResults}`, a subset of the API's
  `raceData`. Each `countyResults[]` row has:
  - `county`, `fips`, `updatedAt`, `parent_vcu_name`;
  - `candidates[]`: `cand_id`, `first_name`, `last_name`, `votes`, `percent`,
    `party{id,name,abbrev}`;
  - `leader_margin`;
  - `liveTurnoutEstimate{estimatedVotesLow, estimatedVotesMid, estimatedVotesHigh,
    completedCounty, precinctsIn, precinctsTot, totalVotes}`.

### `races.json`

Maps each race id to `{ddhq_id, slug, office, election_type, district}` for 506 races. The race
page is `https://votes.decisiondeskhq.com/races/<slug>`.

---

## kalshi/

These files are the order books Kalshi published over its WebSocket API (`orderbook_delta`
channel). The recorder kept each book locally from the snapshot plus deltas. There are
5,511,789 records, 2026-11-03T21:10:32Z to 2026-11-04T19:03:31Z.

Every line has the envelope `{"v": 1, "t", "src": "kalshimd", "seq", "kind", "data"}`:

- **`seq`** is one strictly increasing sequence shared by both kinds. Merging `book/` and
  `book_depth/` by `seq` reproduces the recorder's single stream exactly (verified).
- **`data.market`** is a race id (`GA-SEN`, `PA-07`, `US-SENATE-CONTROL`, …) or, for
  multi-outcome events, a Kalshi event ticker (`KXDHOUSESEATS-27`, `KXMIDTERMMOV-GASEND`, …).
- **Outcome labels** are `D` and `R` in race markets (`I` for the independent in NE-SEN and
  ID-SEN) and bracket labels in event markets (e.g. `218-221`, `At least 51`,
  `Ossoff, 5+ pts`).

### `book/`: top of book

5,201,976 records. A record is written whenever a market's top of book changes, at most once
per second per market. Format:

```
data: {"market": <str>, "quotes": {<outcome>: [bid, ask, bid_size, ask_size], ...}}
```

- Prices are in dollars per YES contract (0–1). Sizes are in contracts and may be fractional.
- `ask` is 1 − the best NO bid, and `ask_size` is the size of that NO bid.
- **Placeholder values.** When a side has no resting orders, the recorder fills in a
  placeholder:
  - with no YES bids, `bid` = max(0, ask − 0.05) and `bid_size` = 0;
  - with no NO bids, `ask` = min(1, bid + 0.05) and `ask_size` = 0.

  Read a size of 0 as "no order on that side". It is not a tradable quote.

### `book_depth/`: full ladders

309,813 records, taken on a cycle of about 60 seconds. A market is written only when its
ladder changed since its last snapshot. Format:

```
data: {"market": <str>, "depth": {<outcome>: {"yes": [[price, size], ...], "no": [[price, size], ...]}}}
```

Both ladders list resting **bids**, highest price first. A NO bid at p is equivalent to a YES
ask at 1 − p.

### `markets.json`

Maps every `market` to its Kalshi tickers:

- `type`: `race`, `seat_count`, `nov_mov` (margin brackets) or `joint`;
- `event_ticker`;
- `outcomes`: outcome label → market ticker;
- for event markets, `related_market` and `bracket_bounds`: `{side, lo, hi}` per outcome,
  where `lo` and `hi` are inclusive seat counts or margin points.

These tickers are the recorder's subscription list, as harvested at 2026-11-03T19:01Z.

---

## normalized/

`updates_2026-11-03.jsonl.gz` has 136,917 records, 2026-11-03T22:30:01Z to
2026-11-04T09:59:48Z. Each record is the state of one reporting unit for one race from one
source, written each time that unit's values changed:

- 92,486 records are from NYT;
- 44,406 are from DDHQ;
- 25 are Tennessee SoS early-vote figures, entered by hand from elections.tn.gov.

This is a derived file. It is built from the payloads in `nyt/` and `ddhq/`, which remain the
primary record.

| Field | Type | Meaning |
|---|---|---|
| `t` | float | Time the record was written (unix seconds, UTC). |
| `race_id` | str | Race id, e.g. `GA-SEN` or `TX-28`. |
| `unit_id` | str | Unit id; formats below. |
| `level` | str | `county` (county or part of one) or `district` (a whole-district row). |
| `source` | str | `nyt`, `ddhq` or `sos`. |
| `votes_d`, `votes_r` | int | Votes for the Democratic and Republican candidates (the independents in NE-SEN and ID-SEN). For RCV races, first-round votes. |
| `votes_total` | int | Votes for all candidates. |
| `adv_d`, `adv_r` | int or null | Advance (early in person plus mail) votes for D and R. Null where the source has no vote-type breakdown; the field is absent from DDHQ records. |
| `expected_total` | number or null | Expected final total vote: NYT `total_expected_vote`, or the sum of DDHQ `estimatedVotesMid`. |
| `eevp` | float or null | NYT `eevp` / 100, clipped to the range 0–1. Null for DDHQ. |
| `complete` | bool | NYT: `eevp` ≥ 99.5, or `eevp` at `eevp_max` (≥ 99) with every precinct in. DDHQ: `completedCounty` with votes greater than 0. |
| `precincts_reporting`, `precincts_total` | int | Precinct counts. |
| `note` | str | Manual SoS records only: where the figures came from. |

**`unit_id` formats**

| Format | Example | Meaning |
|---|---|---|
| 5-digit county FIPS | `13121` | A county. |
| FIPS plus name | `17031:Chicago` | A named part of a county split between districts. |
| State plus name | `AK:State House District 10` | Alaska state house districts, Alaska overseas absentee votes. |
| The race id | `NY-12` | A district total row from NYT's House payload. |

Two DDHQ county records failed a plausibility check and are **absent** from this file. Both
remain in `ddhq/ddhq_races`:

- TN-GOV, Jefferson County (47089), at 2026-11-04T04:57Z: 102,041 votes, 4.0× the county's
  entire 2024 presidential vote.
- NY-22, Oneida County (36053), at 2026-11-04T05:52Z: 1,221,399,536 votes.

---

## Known data issues

These issues were observed while the feeds were live. Each one is present in the data as
received.

- **NYT SD-GOV D/R swap.**
  - From about 02:25Z to 03:15Z, `gov-county-data` alternated between two payload versions.
    In one of them, Ahlers' (D) and Rhoden's (R) votes are exchanged in 36 South Dakota
    counties, with identical totals. That accounts for 1,198 swapped county records, from
    02:25:20Z to 03:15:20Z.
  - The earliest SD-GOV county data, from about 02:01Z to 02:25Z, also shows Ahlers leading.
    Once the alternation ended, Rhoden led by about 2:1.
- **NYT NV-02 Washoe County spike.** From 08:13Z to 08:28Z, `results-house.json` reported
  Washoe County at 242,395 votes, about 115% of NYT's own expected vote. It then reverted to
  about 144k. The Governor payload showed about 145k for Washoe at the same time.
- **NYT Election Day expected vote ran high.**
  - Advance-vote expectations were close to the final count (about +1% overall).
  - Election Day in-person expectations were well above the final counts. The figures
    compare `expected_vote_by_vote_type.election_day_in_person` as of 02:30Z with the count
    at feed stop, in counties that report vote types: TN +38%, TX +21%, KY +18%, IL +17%,
    DE +14%, OK +13%, GA +12%. NC (−16%) and AR (−18%) were too low.
  - Later in the night NYT lowered county expected totals 808 times with no change in votes,
    removing about 6.7 million expected votes in all. The median cut came about an hour after
    the county's last vote change.
  - Examples: LA-SEN Orleans Parish expected 267k against about 140k counted, corrected around
    05:12Z; phantom outstanding vote in Davidson and Shelby counties (TN House districts 04,
    05, 07 and 09), later removed.
- **NYT `eevp` is capped at 99** (`eevp_max`), so a finished county can still show 99%. The
  practical completion signal is `eevp` = `eevp_max` with `precincts_reporting` =
  `precincts_total`.
- **NYT FL-26 Miami-Dade typo:** 3,000 extra Democratic votes, corrected about 20 minutes later.
- **DDHQ duplicate-county rows.** Some county rows briefly carried another county's numbers:
  - TN-04 Franklin had Coffee County's figures.
  - MS-SEN Jasper had Hancock County's figures for about 6 minutes, then reverted with D and
    R swapped.
- **DDHQ impossible totals:** TN-GOV Jefferson and NY-22 Oneida (see `normalized/`).
- **Results after 10:00Z.** The results feeds stopped at about 10:00Z (5:00 am ET), and
  races still counting at that point are incomplete. The Kalshi books continue to 19:03Z.
