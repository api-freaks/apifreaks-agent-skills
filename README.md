# APIFreaks Agent Skills

[![skills.sh](https://skills.sh/b/api-freaks/apifreaks-agent-skills)](https://skills.sh/api-freaks/apifreaks-agent-skills)

Agent skills for the [APIFreaks](https://apifreaks.com) platform. Install them in Claude, Cursor, Codex, GitHub Copilot, or any client that supports [Agent Skills](https://agentskills.io/specification), and the agent can query live APIFreaks APIs from a conversation.

Use the [MCP server](https://apifreaks.com/integrations/mcp-server) when you want live tools. These skills cover the same platform over REST when MCP is not connected, and still apply for credits and errors when it is.

```bash
npx skills add api-freaks/apifreaks-agent-skills
```

## Skills

### apifreaks-platform

Gives an agent a reliable way to use APIFreaks. If MCP tools are connected, it uses them. If not, it calls REST against the live catalog: authenticate, resolve the current endpoint and version, state the credit cost, then handle errors, concurrency, bulk requests, and async jobs.

**Use when**

- Looking up WHOIS, DNS, SSL, domains, or IP intelligence
- Validating email, phone, VAT, IBAN, or SWIFT
- Taking screenshots, scraping pages, or processing PDFs
- Enriching a list of domains, IPs, or emails

Requires an APIFreaks API key. Instructions: [`SKILL.md`](skills/apifreaks-platform/SKILL.md).

## Install

```bash
# list skills in this repo
npx skills add api-freaks/apifreaks-agent-skills --list

# current project
npx skills add api-freaks/apifreaks-agent-skills --skill apifreaks-platform

# all projects
npx skills add api-freaks/apifreaks-agent-skills --skill apifreaks-platform -g
```

```bash
npx skills update apifreaks-platform
npx skills remove apifreaks-platform
```

Add `-g` on update or remove if you installed globally.

To install by hand, copy `skills/apifreaks-platform/` into your agent's skills directory.

## API key

Create a key at [apifreaks.com](https://apifreaks.com). New accounts include 10,000 free credits, no card required.

The skill reads `APIFREAKS_API_KEY`. Put it where the agent can see it: a gitignored `.env`, your shell profile, or the current session.

```bash
export APIFREAKS_API_KEY="your_key"
```

For MCP, set the same variable in the client's `env` block.

## Usage

The skill is for work an agent cannot do from memory: live lookups, lists, logs, and files.

```
Enrich unique client IPs in access.log with country, ASN, and VPN/proxy.
Skip private ranges. Write a table of the suspicious ones only.
```

```
Validate every email and phone in signups.csv. Keep rows the APIs accept;
list failures with the returned reason, not a guess.
```

```
Check these 12 product names for domain availability, then run a typosquat
lookup on the one we pick.
```

```
Merge invoices/*.pdf, compress the result, and save it. Quote the credit
cost before you start the job.
```

## License

[MIT](LICENSE)
