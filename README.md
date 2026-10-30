# Inodra skills

Agent skills for building on Sui with [Inodra](https://inodra.com). They teach coding agents (Claude Code, Cursor, Codex and the other agents the `skills` CLI supports) how to use the Inodra gRPC and GraphQL gateways, Warp streams, webhooks, and how to debug an integration.

| Skill                              | Use it when                                                        |
| ---------------------------------- | ------------------------------------------------------------------ |
| [inodra-build](inodra-build/)      | Writing an app that reads Sui data through Inodra                  |
| [inodra-grpc](inodra-grpc/)        | Calling the `sui.rpc.v2` services, paging `List*`, field masks     |
| [inodra-warp](inodra-warp/)        | Live streams over SSE or WebSocket                                 |
| [inodra-webhook](inodra-webhook/)  | HTTP notifications for events, addresses, coins and objects        |
| [inodra-debug](inodra-debug/)      | A request to Inodra fails or returns nothing                       |

## Install

With the [`skills`](https://github.com/vercel-labs/skills) CLI, into the agents of your choice:

```bash
npx skills add inodrahq/skills
```

One skill only:

```bash
npx skills add inodrahq/skills --skill inodra-grpc
```

Claude Code without the CLI (global, every project):

```bash
git clone https://github.com/inodrahq/skills ~/.claude/skills/inodra
```

Or per project, committed with the code:

```bash
git clone https://github.com/inodrahq/skills .claude/skills/inodra
```

## Sui skills

These skills cover Inodra. For Move, PTBs, the TypeScript SDK and dApp Kit, install the skills maintained by Mysten Labs alongside them:

```bash
npx skills add mystenlabs/skills --skill sui-move --skill ptbs --skill accessing-data --skill sui-ts-sdk-backend --skill frontend-apps
```

## MCP server

The [`@inodra/mcp-server`](https://www.npmjs.com/package/@inodra/mcp-server) package gives agents live tools for the same APIs and ships copies of these skills.

## Contributing

Each skill is a folder with a `SKILL.md` that starts with `name` and `description` frontmatter. Keep the sentences short, and keep examples on the gRPC and GraphQL gateways.

## License

MIT
