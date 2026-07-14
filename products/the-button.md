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

**Status:** backend complete, not shipped. The `algovn.button.v1` contract is
published and `the-button-service` implements the proof-of-work gate, click
batching, achievements and the tick leader. The service is not deployed, not
registered with the API gateway, and has no frontend — see
[`../ROADMAP.md`](../ROADMAP.md).

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
