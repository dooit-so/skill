# Dooit skill

An [agent skill](https://code.claude.com/docs/en/skills) that teaches Claude to
run your [Dooit](https://dooit.so) task board — plan your day, capture and
schedule tasks, work a weekly review, and track weekly/monthly/yearly goals.

It works through Dooit's remote MCP server, so it reads and writes your real
board, in your own timezone, with your own permissions.

## Install

```sh
npx skills add dooit-so/skill
```

Then connect the MCP server once:

```sh
claude mcp add --transport http dooit https://app.dooit.so/mcp
```

Other clients (Claude Desktop, claude.ai, ChatGPT, Cursor) take the same URL as
a custom connector. Sign-in is OAuth — there is no API key to copy.

You need a [Dooit account](https://app.dooit.so). Reads work on any account;
writing to your board requires an active subscription.

## What it adds

The MCP server is the hands; this skill is the manual. It carries the things a
tool description can't: call `get_board` before anything else and never invent
an id, derive dates from the user's timezone rather than your own, archive
instead of delete, and the named plays — *plan my day*, *clear my day*,
*weekly review*, *capture this*.

## Links

- [MCP server](https://dooit.so/mcp) — connection guide, tools, limits
- [Agent guide](https://dooit.so/llm-info) — the same reference, written for LLMs

## License

MIT
