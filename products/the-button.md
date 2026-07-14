# The Button

One button. One goal. Millions of humans.

A single global click counter at https://algovn.com/the-button. Anyone can watch
it tick live; logging in lets you press the button — as many times as you can
stand. Every batch of clicks costs a little proof-of-work (your browser mines a
tiny hash puzzle), which is both the anti-bot throttle and part of the joke.

**Why:** stress-testing a home server · proving humans can work together ·
because the internet needs more joy · every click is a tiny rebellion.

**Features:** live counter (SSE), one big button, personal troll achievements
("Minimum Viable Human" — clicked once; "Carpal Diem" — 10,000 clicks;
"3am Rebellion"), global milestones ("One Million. Together We Did… This.").

**Scale target:** 10,000 concurrent users on the home cluster.

**Status:** live at [algovn.com/the-button](https://algovn.com/the-button) (2026-07).
The `algovn.button.v1` contract is implemented end to end: `the-button-service`
runs the proof-of-work gate, click batching, achievements and the tick leader; the
SPA is deployed and registered with the API gateway; login, clicks, the live SSE
counter and achievements all work in production.

Known limits, measured honestly: the concurrency target is 10,000 simultaneous
users, but the 2026-07-14 SSE load test broke at ~7,300-7,500 concurrent
connections — a cloudflared tunnel pod ran out of memory (since resized; a re-test
to confirm 10k holds has not yet been reported). The authenticated click-soak
(real proof-of-work submits through the gateway) was not run; write-path evidence
is a direct in-cluster Postgres soak (981 batch-txn/s, matching the 750 txn/s
engineered ceiling with margin) plus the service's integration tests. See
`iac/docs/load-results-the-button.md` and `iac/docs/runbooks/the-button.md` for
the full evidence and operational knobs.

**How it works:** clicks are batched in the browser and each batch pays a
hashcash-style proof of work, with difficulty scaling to server load; a hard
per-user minimum interval is the real throttle. Each batch is one Postgres
transaction (personal truth) and a transactional-outbox apply to a Redis counter
(the public total), so the counter can never double-count a retry. A single
elected tick leader publishes the counter once a second and claims global
milestones exactly once.

**Platform it rides on:** api.algovn.com gateway (api-control-plane), Zitadel
login, RabbitMQ→SSE push, shared Postgres, Redis — all described in
[`../ARCHITECTURE.md`](../ARCHITECTURE.md).
