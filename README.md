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
| The Button | One global click counter; PoW-gated mashing, troll achievements | building | [spec](products/the-button.md) · [service](https://github.com/the-algovn/the-button-service) |

The Button's backend and platform prerequisites are complete; it is not deployed
and has no frontend yet. See [`ROADMAP.md`](ROADMAP.md).

Status flow: `idea` → `spec` → `building` → `live`. Status is tracked only
in this table.

## Lifecycle

idea → `BACKLOG.md` → spec in `products/` + row here (`spec`) → repo created
under the org (`building`) → shipped (`live`).
