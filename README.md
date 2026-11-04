# ProjectX

A one-person **company factory**: a Captain agent and a crew of bots that run
your products through a business loop — research → ship → market → measure →
fix conversion — built as plugins on [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness).

Working name. By [LieuGroup](https://github.com/LieuGroup). MIT.

## Status

Day 0. Research and decisions only; no code yet.
See [docs/decisions.md](docs/decisions.md) for the founding decisions and
[docs/research/](docs/research/) for the background research.

## Ideas in one table

| Word     | Meaning                                              |
|----------|------------------------------------------------------|
| Captain  | Manager bot. Talks to you, hands out work, never does the work itself. |
| Bot      | A permanent, named worker with memory.               |
| Issue / Thread | A unit of work with a status, owned by one bot. |
| Routine  | Work that runs by itself on a schedule or trigger.   |
| Inbox    | The only things that need you: approvals, decisions. |

## Stack

- Engine: DeepSeek Harness (plugins only, no fork).
- Hands: Grok Bot (via `grokbot` CLI) now, OpenBot later, behind one bot-provider interface.
- First product in the factory: FillLight (iOS).

## Licence

MIT — see [LICENSE](LICENSE).
