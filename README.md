# algovn specs

Product specs for the [algovn](https://github.com/the-algovn) portfolio —
free tools and APIs built as a personal SaaS showcase, not for revenue.

- One free-form spec per product in [`products/`](products/).
- Raw ideas start in [`BACKLOG.md`](BACKLOG.md).
- Build order lives in [`ROADMAP.md`](ROADMAP.md).
- How the platform is put together — and what a new product inherits for free —
  is in [`ARCHITECTURE.md`](ARCHITECTURE.md).

This repo is the product catalog and the platform's living technical
description. Individual repos carry their own operational docs (`iac` has the
runbooks and the API/gRPC/authnz conventions).

## Portfolio

| Product | Description | Status | Links |
| ------- | ----------- | ------ | ----- |
| The Button | One global click counter; PoW-gated mashing, troll achievements | live | [spec](products/the-button.md) · [service](https://github.com/the-algovn/the-button-service) · [live](https://algovn.com/the-button) |
| Tần Số 42 | AI Vietnamese radio: song requests, on-air dedications, an AI DJ streaming 24/7 | building | [spec](products/radio.md) · [service](https://github.com/the-algovn/radio-service) |

The Button is live at [algovn.com/the-button](https://algovn.com/the-button). See
[`products/the-button.md`](products/the-button.md) for known limits.

Status flow: `idea` → `spec` → `building` → `live`. Status is tracked only
in this table.

## Lifecycle

idea → `BACKLOG.md` → spec in `products/` + row here (`spec`) → repo created
under the org (`building`) → shipped (`live`).
