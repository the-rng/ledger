# The-RNG Ledger

This repository is the **public ledger** for all random number generation (RNG) events conducted by [The-RNG](https://the-rng.com).

## Purpose

We believe in absolute transparency and long-term accountability. Every time a draw is conducted on our platform, the cryptographic proofs and parameters are automatically committed to this repository. This ensures that:

1. **Independent persistence** — this ledger exists outside our internal databases. Even if The-RNG ceased to exist, these results remain accessible and verifiable forever.
2. **History cannot be altered** — the Git commit history serves as a timestamped, tamper-evident log. Each record is also SHA-256 hash-chained to the previous one inside our system.
3. **Verification is public** — anyone can audit any draw using only the data stored here and the public drand beacon.

## How It Works

Our system is verifiably fair. It relies on **drand**, the distributed randomness beacon operated by the League of Entropy (Cloudflare, EPFL, University of Chile and others). Drand provides unpredictable, publicly verifiable randomness which we use as the seed for every draw.

For each draw we combine:

1. **Drand Server Seed** — the randomness of a specific drand round, committed to **before that round exists** (live round + 3).
2. **Client Seed** — a millisecond timestamp assigned server-side at the moment of the request.
3. **Round ID** — a unique UUID for the draw.
4. **Static Salt** — a public constant unique to The-RNG.
5. **Ticket parameters** — tickets sold and max tickets.

These inputs are combined with **HMAC-SHA256** and converted to winning ticket numbers using **unbiased rejection sampling**.

## File Structure

Records are organised by UTC date:

```
/YYYY
  /MM
    /DD
      /round-uuid.json
```

## Example Record

Real record ([2026/07/15/94f64eb9-4a97-40a6-af16-c6c45f368102.json](2026/07/15/94f64eb9-4a97-40a6-af16-c6c45f368102.json)):

```json
{
    "round_id": "94f64eb9-4a97-40a6-af16-c6c45f368102",
    "draw_title": "Test",
    "competition_url": "",
    "operator": {
        "name": "The-RNG",
        "website": "https://the-rng.com",
        "company_number": "SC1111111"
    },
    "drand_server_seed": "2645cb4017c599c3924a0b043c9ab6620793a5c529de4aa0ec3d5d6a2e8cb9c5",
    "drand_seed_hash": "912084b9f80878f57f54f759e6fe533bf7f43aecb5f1681e5985b33ac94f9d3dd8e94284dc10f6e52a9adeba24a99bfb",
    "client_seed": "1784148155269",
    "static_salt": "aee874815cd18f485a56d43f49fab9d1",
    "tickets_sold": 6,
    "max_tickets": 9000,
    "combined_hash": "60db0c12c6a65e265475b881fc26812da6cb2f81768804f89b2a804f2dc7b2f7",
    "result": "481",
    "external_round_id": "30448266",
    "processing_status": "completed",
    "created_at": "2026-07-15T20:42:35.000000Z",
    "revealed_at": "2026-07-15T20:42:44.000000Z"
}
```

## Field Descriptions

### Draw identification

| Field | Type | Description |
| --- | --- | --- |
| `round_id` | `string` | Unique UUID identifying this draw. Used as the filename and primary reference. |
| `draw_title` | `string` | Human-readable name or prize description. |
| `competition_url` | `string` | URL of the competition this draw was conducted for. |
| `external_round_id` | `string` | The drand beacon round number used. Verifiable on any public drand relay. |

### Operator information

| Field | Type | Description |
| --- | --- | --- |
| `operator.name` | `string` | Company or individual that conducted the draw. |
| `operator.website` | `string` | Official website of the operator. |
| `operator.company_number` | `string` | Registered company number. |

### Cryptographic inputs

| Field | Type | Description |
| --- | --- | --- |
| `drand_server_seed` | `string` | The raw randomness published by the drand beacon for the committed round. Equals `SHA-256(drand_seed_hash bytes)`. |
| `drand_seed_hash` | `string` | The BLS threshold signature proving the randomness came from the distributed beacon. |
| `client_seed` | `string` | Millisecond timestamp assigned server-side when the draw was initiated. |
| `static_salt` | `string` | Public constant unique to The-RNG, identical across all draws. |

### Ticket parameters

| Field | Type | Description |
| --- | --- | --- |
| `tickets_sold` | `integer` | Tickets sold at draw time (part of the hash input). |
| `max_tickets` | `integer` | Draw range — winning numbers fall between 1 and `max_tickets`. |

### Result data

| Field | Type | Description |
| --- | --- | --- |
| `combined_hash` | `string` | `HMAC-SHA256(clientSeed:roundId:staticSalt:ticketsSold:maxTickets, key = drand_server_seed)`. |
| `result` | `string` | Winning ticket number(s), derived from `combined_hash` by unbiased rejection sampling. Multiple winners are comma-separated in draw order. |
| `processing_status` | `string` | `completed`, or `voided` (voided draws remain in the ledger with their data intact). |

### Timestamps

| Field | Type | Description |
| --- | --- | --- |
| `created_at` | `string` | ISO 8601 UTC time the draw was committed — always **before** the drand round existed. |
| `revealed_at` | `string` | ISO 8601 UTC time the beacon was fetched and the result calculated. |

## Verifying a Draw

1. **Verify the drand source.** Fetch the committed round from independent public relays and confirm `randomness` equals `drand_server_seed` and `signature` equals `drand_seed_hash`:

```
https://drand.cloudflare.com/52db9ba70e0cc0f6eaf7803dd07447a1f5477735fd3f661792ba94600c84e971/public/{external_round_id}
https://api.drand.sh/52db9ba70e0cc0f6eaf7803dd07447a1f5477735fd3f661792ba94600c84e971/public/{external_round_id}
```

2. **Recreate the hash.**

```
combined_hash = HMAC-SHA256( message: "client_seed:round_id:static_salt:tickets_sold:max_tickets",
                             key:     drand_server_seed )
```

3. **Verify the result.** Read the hash 4 bytes at a time as unsigned 32-bit little-endian integers. With `limit = 0xFFFFFFFF − (0xFFFFFFFF mod max_tickets)`, reject values ≥ limit (this removes modulo bias), and each accepted value gives `ticket = (value mod max_tickets) + 1`. Additional winners continue the same walk, skipping duplicates; if the 32 bytes are exhausted, the walk continues on `HMAC-SHA256(data + ":extend:" + n, drand_server_seed)` for n = 1, 2, 3…

Use our in-browser [verification tool](https://the-rng.com/verify-draw/), follow the [manual verification guide](https://the-rng.com/manual-verification/) with ready-made PHP/JavaScript scripts, or write your own from the specification above.

## Automation

Records are committed automatically by our open-source WordPress plugin: [the-rng/plugin](https://github.com/the-rng/plugin). The plugin also exposes a public integrity endpoint publishing SHA-256 hashes of its running files, so anyone can confirm the deployed code matches the published source.
