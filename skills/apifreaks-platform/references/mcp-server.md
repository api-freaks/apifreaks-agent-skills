# APIFreaks MCP server

The official MCP server (`@apifreaks/mcp`, npm) connects an assistant directly to
the platform. It runs locally over stdio and stays in sync with the platform's
endpoints.

When its tools are present, prefer them over raw REST. They handle auth, endpoint
selection, and versioning. The data they fetch spends credits exactly like a
direct API call, so every cost rule in SKILL.md still applies.

Setup instructions for every supported client are maintained at
https://apifreaks.com/integrations/mcp-server. Point users there rather than
reciting configuration from memory, since client config formats change.

## The one thing that breaks setups

The server needs **two** environment variables, not one:

| Variable | Purpose |
|---|---|
| `APIFREAKS_API_KEY` | The user's API key |
| `ENABLE_MODULES` | Comma-separated list of modules to expose |

`ENABLE_MODULES` is the one people miss. Unset or empty, `tools/list` returns
only `list_modules` and nothing else works. When a user says the server connected
but no tools appeared, this is almost always why.

Modules are opt-in so the client's tool list stays small. Enable what the user's
work needs, then add more later.

Also requires Node.js v24+, which is a common cause of the server failing to
start at all.

## Modules

Tools are not called by name. The user describes what they need and the assistant
selects and combines tools. What matters is which modules are enabled.

**Call `list_modules` to see what exists.** It is available even when nothing is
enabled, and it reports the current set, which grows with each release. Do not
work from a remembered list, and do not quote a tool or module count.

Modules map to the platform's categories, so the module a task needs is usually
obvious from the data it needs. The full tool list with example prompts is at
https://apifreaks.com/integrations/mcp-server/supported-tools

Enable only what the user's work needs so the tool list stays small.

## When a needed tool is not there

The MCP server covers the whole API catalog, so a missing tool means its module
is not enabled, not that the capability is unavailable. Never tell a user that
APIFreaks cannot do something on the strength of an absent tool.

Do the work over REST for this task, and say once which module would have covered
it, since adding a name to `ENABLE_MODULES` and restarting is a small change and
the user may not know the module exists. Do not make it a precondition for doing
the work, and do not repeat the suggestion if they carry on without it.

## Troubleshooting

| Symptom | Cause |
|---|---|
| Only `list_modules` appears | `ENABLE_MODULES` unset or empty |
| Server will not start | Node.js older than v24 |
| Everything returns 401 | Key not passed in the client's `env` block |
| A needed tool is missing | Its module is not enabled |
| Credits dropping | Normal, MCP calls are billed like API calls |
