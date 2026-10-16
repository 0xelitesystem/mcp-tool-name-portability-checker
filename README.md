# MCP Tool Name Portability Checker

Paste an MCP server's tools/list result (or a plain list of tool names) and see, name by name, which documented consumer rule accepts it as is: the MCP spec, the Anthropic Messages API, OpenAI Chat Completions and the Gemini API. It also flags names a client prefix pushes over a length limit and names that collide once dots, colons and spaces are replaced, and it builds a deterministic, collision-free rename map that fits every rule you select.

**Live demo:** https://0xelitesystem.github.io/mcp-tool-name-portability-checker/

## Use

1. **Paste.** Any of: a full JSON-RPC response to `tools/list`, the bare result object (`{"tools":[...]}`), a JSON array of tool objects, a JSON array of strings, several result pages in one JSON array, or plain text with one name per line. Only each tool's `name` is read. Names from JSON are never trimmed, because the name is what the server sent. Text that starts with `{` or `[` but does not parse is reported with the parser's message, and a **Treat as plain lines** button appears; it is never silently read as lines.
2. **Pick the rules.** All five are on by default. The strictest selected limit is shown (64 with the defaults).
3. **Set a prefix**, if your client adds one. **Claude Code style** fills `mcp__{server}__`, and **Claude Code plugin style** fills `mcp__plugin_{plugin}_{server}__`. With a prefix, every rule is checked against prefix plus name, and the prefix itself is checked separately.
4. **Read the table.** One row per name: its length in code points, PASS or FAIL per rule with the reason (`contains "." at position 6`, `length 70 > 64`, `empty`), flags (DUPLICATE, CASE-ONLY, COLLIDES-AFTER-RENAME) and the proposed name. Spaces show as a middle dot and non-ASCII characters as their U+ code.
5. **Copy the map** as a JSON object `{"original":"renamed"}` or as a JSON array of `{from, to}`.

A gateway can apply the map to tools/list and reverse it on tools/call. The reverse lookup is needed because the server still expects its original name. The page does not rewrite anyone's traffic.

### The rename algorithm

Deterministic, so the same input always gives the same map:

1. Each Unicode code point outside `A-Z a-z 0-9 _ -` becomes one `_`. This mirrors the replacement rule Claude Code documents for plugin tool names.
2. The budget is the strictest selected limit minus the prefix length. Under 8, no map is built.
3. Names that already use only those characters and fit the budget are kept, in input order. On exact duplicates, the first one wins.
4. Every other name is sanitized. If it is still too long, it is cut to `budget - 7` characters plus `_` and the first six hex digits of the FNV-1a 32-bit hash of the original UTF-8 name. Then `_2`, `_3` and so on are appended until it is unique.

With the defaults, `admin.tools.list` beside `admin_tools_list` becomes `admin_tools_list_2`, a 70-character name of `a` becomes 57 `a` plus `_590474`, and `café_lookup` becomes `caf__lookup`.

### Use cases

- **Your server works in one client and fails in another.** Paste the tools/list output with all rules on. The failing column names the character and its position.
- **You are writing a gateway that merges several servers.** Set the prefix your gateway adds, and copy the map it should apply on the way out and reverse on the way in.
- **You are naming tools for a new server.** Check the names before you ship, and keep to the set every rule shares: letters, digits, `_` and `-`, at most 64 characters including any client prefix.
- **A review flags dotted names.** The table shows that the MCP spec lists `admin.tools.list` as a valid example, and which consumer rules reject it.

## Why this exists

The MCP spec lists `admin.tools.list` as an example of a valid tool name. The Anthropic and OpenAI documented rules have no dot. Each side follows its own published rule, and a name that is valid at the source fails at the consumer unless something in between renames it. These public issues report it happening:

- anthropics/claude-code#858, filed 2025-04-23: https://github.com/anthropics/claude-code/issues/858
- anthropics/claude-code#68850, filed 2026-06-16: https://github.com/anthropics/claude-code/issues/68850
- jacob-hartmann/kinsta-mcp#35, filed 2026-03-25: https://github.com/jacob-hartmann/kinsta-mcp/issues/35
- jgravelle/jmunch-mcp#2, filed 2026-04-30: https://github.com/jgravelle/jmunch-mcp/issues/2
- docker/mcp-gateway#228, filed 2025-11-10: https://github.com/docker/mcp-gateway/issues/228

Single file, no build step, no network call, MIT licensed. Nothing to install, and nothing watching the tool list you paste.

## The rules, with sources

Retrieved 2026-10-16.

| Profile | Rule | Level | Source |
| --- | --- | --- | --- |
| MCP spec 2025-11-25 (same text in 2026-07-28) | 1 to 128 characters of `A-Z a-z 0-9 _ - .` | SHOULD | https://modelcontextprotocol.io/specification/2025-11-25/server/tools and https://modelcontextprotocol.io/specification/2026-07-28/server/tools |
| Anthropic Messages API | `^[a-zA-Z0-9_-]{1,128}$` | documented | https://platform.claude.com/docs/en/agents-and-tools/tool-use/define-tools |
| OpenAI Chat Completions `tools[].function.name` | a-z, A-Z, 0-9, `_`, `-`, maximum 64 | documented | https://developers.openai.com/api/reference/resources/chat/subresources/completions/methods/create |
| Gemini `FunctionDeclaration.name` | a-z, A-Z, 0-9, `_`, `:`, `.`, `-`, maximum 128 | documented | https://ai.google.dev/api/generate-content |
| Observed pattern | `^[a-zA-Z0-9_-]{1,64}$` | observed in error strings, not a published rule | anthropics/claude-code#858 and #68850 |

Notes on each:

- The MCP spec also says tool names SHOULD be considered case-sensitive, SHOULD NOT contain spaces, commas or other special characters, and SHOULD be unique within a server. Its 2026-07-28 revision adds that clients or proxies that aggregate tools from several servers SHOULD implement a disambiguation strategy such as prefixing tool names with a server identifier.
- OpenAI's Responses API reference describes the function name only as "The name of the function to call." with no character rule. This page applies the Chat Completions text and says nothing about the Responses API's behaviour. The Chat Completions reference states no minimum length, so an empty name is an input error under every rule.
- The same Gemini reference describes `FunctionCall.name` and `FunctionResponse.name` as letters, digits, underscores and dashes only, with no colons or dots. Both quotes are on the page; the page does not say which one the service enforces.
- The observed pattern is quoted from error strings in public issues, with their filed dates. The page does not say whether it is still current, and names no enforcing component beyond the field name `FrontendRemoteMcpToolDefinition.name` in the reported string.
- Claude Code's hooks documentation says MCP tools follow the naming pattern `mcp__<server>__<tool>` (https://code.claude.com/docs/en/hooks), and its MCP documentation gives the plugin form `mcp__plugin_<plugin-name>_<server-name>__<tool-name>`, with any character outside `A-Z`, `a-z`, `0-9`, `_` and `-` replaced with `_` (https://code.claude.com/docs/en/mcp). The page states no Claude Code length limit and no renaming behaviour for servers outside plugins.

## Privacy

Everything runs in your browser. There is no network call of any kind: no request for code, fonts, analytics or telemetry, and nothing you paste is transmitted anywhere. A Content-Security-Policy in the page blocks every connection. Tool descriptions are never displayed. The only thing stored is your light or dark theme choice, in your own browser's localStorage. Save the file and it works offline from `file://`.

## Run locally

```
git clone https://github.com/0xelitesystem/mcp-tool-name-portability-checker.git
cd mcp-tool-name-portability-checker
```

Open `index.html` in a browser. If you would rather serve it:

```
python -m http.server 8000
```

then open `http://localhost:8000/`.

## Build

No build. One self-contained `index.html` with inline CSS and inline JavaScript. No dependencies, no package manager, no bundler, no CDN, no web font.

## Independence

Independent and unaffiliated with Anthropic, OpenAI, Google or the MCP project. Rules are quoted from their public documentation.

## Related

- https://github.com/0xelitesystem/mcp-header-annotation-validator
- https://github.com/0xelitesystem/mcp-tool-poisoning-scanner
- https://github.com/0xelitesystem/json-schema-to-tool-definition

## License

MIT. Copyright (c) 2026 0xelitesystem.
