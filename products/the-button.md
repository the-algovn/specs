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

Technical design: `the-button-service` repo,
`docs/superpowers/specs/2026-07-14-the-button-design.md`.
Platform it rides on: api.algovn.com gateway (api-control-plane), Zitadel login,
RabbitMQ→SSE push, shared Postgres, Redis.
