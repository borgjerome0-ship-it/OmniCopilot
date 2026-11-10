# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A VS Code extension ("OmniRoute for GitHub Copilot Chat") that registers an [OmniRoute](https://github.com/diegosouzapw/OmniRoute) server as a native language-model chat provider, so every model OmniRoute serves appears in Copilot Chat's model picker. This checkout is the owner's fork of `diegosouzapw/OmniCopilot`: the `publisher`, `repository` and marketplace links in `package.json` and `README.md` still point at upstream. Do not publish to the VS Code Marketplace or Open VSX from this fork (that needs upstream's tokens; see `docs/PUBLISHING.md`).

## Commands

```bash
npm ci                                   # install (dev dependencies only; nothing ships at runtime but dist/)
npm run compile                          # tsc --noEmit, then esbuild -> dist/extension.js
npm run watch                            # esbuild in watch mode (no type check)
npm run check-types                      # tsc --noEmit only
npm run lint                             # eslint src
npm test                                 # vitest run (all of test/**/*.test.ts)
npx vitest run test/client.test.ts       # one test file
npx vitest run -t "prefix=alias"         # tests whose name matches
npm run package                          # type check + minified production bundle
npm run vsix                             # build omnicopilot.vsix with vsce
```

Before a pull request, run `check-types`, `lint` and `test`; all three must pass. ESLint forbids `any`, `eval` and implied eval, and unused variables unless prefixed with `_`. TypeScript is `strict` with `noUnusedLocals`, `noUnusedParameters` and `noImplicitReturns`.

## Architecture

`src/extension.ts` `activate()` wires everything and registers the commands from `package.json` `contributes.commands`. A configuration change under `omnicopilot.*` restarts the status bar, refreshes the model list and refreshes the panel.

- **`client.ts` (`OmniRouteClient`)**: the only HTTP layer. Talks to OmniRoute's OpenAI-compatible API under `<baseUrl>/v1` (`normalizeBaseUrl` adds `/v1`; `serverRootUrl` strips it for dashboard links). `listModels` fetches the catalog, `streamChat` posts `/chat/completions` and parses the SSE stream into `StreamEvent`s, accumulating partial tool-call deltas until each call is complete. `ping` and `canEmbedDashboard` are short-timeout probes.
- **`provider.ts` (`OmniRouteChatProvider`)**: implements `vscode.LanguageModelChatProvider`. Model information comes from the catalog after `catalogFilter.ts` (chat models only, `modelFilter` setting) and `agentsWindow.ts` (optional duplicate entries scoped to the Copilot Agents window). Requests go through `convert.ts` (VS Code messages and tools to OpenAI shape, token estimates) and `reasoning.ts` (which models get `reasoning_effort`, and at what tier).
- **`statusBar.ts`**: periodic health probe and the status-bar dot; the provider also reports each request's success to it so it does not need extra polling.
- **`panel.ts`**: the Activity Bar webview (server URL, API key, Save & Test, model count, usage). `usage.ts` reads `GET /api/usage/om-usage?format=json`; `embed.ts` decides whether the dashboard can open inside an editor tab.
- **`cliBridge.ts`**: "Configure Coding CLI" types an `omniroute <subcommand>` command (`omnicopilot.cliPath`) into an integrated terminal, adding a shell-quoted `--remote <url>` only for a non-local server. The API key goes in the terminal's environment as `OMNIROUTE_API_KEY`, never on the command line or in the log.
- **`types.ts`**: catalog and stream types shared by the modules above.

esbuild bundles `src/extension.ts` to `dist/extension.js` (CommonJS, `vscode` external). `.vscodeignore` leaves `src/`, `test/`, `scripts/`, every `.ts` file and `node_modules/` out of the `.vsix`, so runtime code must be reachable from the bundle.

## Rules that are easy to break

- **Catalog shaping** (full reasoning in `docs/CATALOG.md`): request the catalog with `?prefix=alias`, never `?prefix=canonical` (canonical silently drops providers that have no separate alias). Also drop mirror rows by their `parent` back-reference for servers that ignore `?prefix`. Filter out non-chat models by the catalog `type` field, not by `supported_endpoints` (Responses-API-only models still work on `/v1/chat/completions`).
- **Reasoning effort**: an explicit choice from the chat UI wins; `omnicopilot.defaultReasoningEffort` applies only to models the catalog marks as reasoning-capable (others may answer 400). Values are OmniRoute's `none|low|medium|high|xhigh`.
- **API key**: stored only in VS Code SecretStorage under `omnicopilot.apiKey` (`SECRET_API_KEY`), never in settings or logs.
- **Agents window**: built on the proposed `targetChatSessionType` field and off by default (`omnicopilot.exposeToAgentsWindow`).

## Tests

Vitest aliases the `vscode` module to `test/vscode.mock.ts` (see `vitest.config.ts`), so tests cover the pure logic modules (client, convert, catalog filter, reasoning, usage, embed, agents window) without a VS Code host. If code under test starts using a new `vscode` API, extend the mock. There are no integration tests that launch VS Code.

## Localisation (42 languages)

Two separate systems, and every locale must keep the same keys as the English file:

- `package.nls.json` plus `package.nls.<lang>.json`: the `%key%` strings in `package.json` `contributes` (command titles, setting descriptions).
- `l10n/bundle.l10n.json` plus `l10n/bundle.l10n.<lang>.json`: runtime strings from `vscode.l10n.t("English source", ...)`, keyed by the English text itself. Changing the English wording of a `t()` call changes the key, so update every bundle.

## Releases

Version bumps go in `package.json` with a matching section at the top of `CHANGELOG.md`. Store publishing steps are in `docs/PUBLISHING.md`; publish tokens live in a git-ignored `.env` and must never be committed or shipped in the `.vsix`.
