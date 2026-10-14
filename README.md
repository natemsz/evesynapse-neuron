# EveSynapse shared-cache seed

A Cloudflare Worker that seeds peer discovery for the EveSynapse
instance mesh. Instances already dial seeds exactly like peers
(`SHARED_CACHE_SEEDS`, gossip source `seed`), so the worker
speaks the identical wire protocol — sealed member lists and
Ed25519-signed entries, byte-compatible with `swarm_api.go`:

- `GET /swarm/members` → `{"entry": ...}`: the worker plus every
  registered peer, sealed under the shared secret, signed by the
  worker's own KV-persisted identity (pinned on first use). Each
  member is a tuple `{instance_id, base_url, public_key, url_sig,
  issued_at}`; `url_sig` is that member's own Ed25519 signature over
  `"member\n" + instance_id + "\n" + base_url + "\n" + issued_at`,
  and `issued_at` is its unix clock when it signed. The worker signs
  its own tuple and relays every other one exactly as it was
  registered.
- `POST /swarm/pull` → `{"entries": []}`: the seed holds no Eve
  data and serves none.
- `POST /swarm/seed/register` → `{"ok": true}`: heartbeat carrying
  the instance's self-signed announce entry (`kind: "announce"`,
  `key`/`signer` = instance id, payload = its base URL, public
  key, `url_sig` and `issued_at`). The worker verifies the envelope and the
  address signature against the claimed key (same
  trust-on-first-use as mesh gossip), requires an `https:` base
  URL, and stores the tuple with a 48h TTL. While a record
  lives, a heartbeat for its ID under another key answers 409,
  and a tuple issued no later than the one held refreshes the
  listing without replacing it (no replay of an old address). Instances
  re-announce every gossip round (12–24h), so dead peers age out
  on their own. At most 1000 peers stored, 500 served, newest
  heartbeats first.

Instances with no public address configured do not announce:
a URL nobody can dial would only pollute the directory.

## Darkness (not crawlable, not indexable)

- No index, no landing page, no robots file — unknown paths 404.
- Known paths without a bearer secret 404 (scanners, crawlers).
- A present-but-wrong bearer 403 (operator-typo signal only).
- Failures answer 404 with no reason body; nothing identifies
  the worker as EveSynapse-related.
- Knowing the worker URL grants nothing without the secret, and
  sealed responses stay opaque to non-secret-holders.

A compromised seed can at worst suggest peers: instances verify
every record signature themselves and spot-check pulled data
against ESI before serving it.

## Deploy

1. `wrangler kv:namespace create SEED_PEERS`, put the id in
   `wrangler.toml`.
2. `wrangler secret put SHARED_CACHE_SECRET` — the exact value
   from the instances' `.env` (32+ hex chars).
3. `wrangler deploy`, then set that URL as the instances'
   `SHARED_CACHE_SEEDS`.

No npm dependencies; WebCrypto only. The byte layout it must
match (envelope key order, Go string escaping, `swm:v1:`
seal envelope, HKDF salt/info) is pinned by
`TestSeedWorkerInterop` in `internal/app/swarm_seed_test.go`,
which opens a worker-format sealed blob produced by an
independent implementation — if either side drifts, that test
goes red.

## Identity

A seed is a role, not one identity. Each worker mints its own
instance ID and Ed25519 keypair on first use and keeps them in KV
(`seed-identity`), and instances pin it on first sight like any
member. Run more than one seed if you want backups or separate
setups; they do not share anything.

A worker that minted its identity while every seed shared the ID
`seed` keeps its keypair and gives itself a fresh ID the first time
it runs this version. Instances then see a new member; the upgraded
instance removes its old `seed` row by itself.

## Rolling out a format change

The member tuple and the heartbeat payload are decoded strictly on
both sides. A change to either (such as `url_sig`) is a flag day:
deploy the worker and upgrade every instance together.
