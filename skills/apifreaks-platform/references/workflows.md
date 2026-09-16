# A worked call

One complete request, start to finish, showing the resolve-then-call cycle this
skill describes. Follow the shape rather than the values, since paths,
parameters, and costs are resolved per task from the endpoint's `reference.md`.

## Single lookup

**Task:** "Who owns example.com?"

**1. Resolve.** `llms-full.txt` names the Domain WHOIS Lookup API, links its
`reference.md`, and gives its credit cost. Fetch that `reference.md`:

```
servers: https://api.apifreaks.com/v2.0
GET /domain/whois/live
domainName  query  string  required
```

**2. Call.**

```bash
curl -s "https://api.apifreaks.com/v2.0/domain/whois/live?domainName=example.com" \
  -H "X-apiKey: $APIFREAKS_API_KEY" -D headers.txt -o out.json
```

**3. Verify cost.** The response headers carry `X-AF-Credits-Cost`, reporting
what was deducted. It confirms the figure `llms-full.txt` already published
rather than revealing a new one.

**4. Read against the schema, not against expectations.** The response carries
registration dates, registrar, name servers, status, contacts, and raw WHOIS
text. Contact fields often hold a privacy-service redirect rather than a real
address, which is normal for privacy-protected domains and not missing data.

**5. Answer** with what the schema returned, saying when contacts are
privacy-protected rather than reporting them as absent.

## Reading an outcome correctly

The most common mistake on this platform is treating a successful negative as a
failure, or a failure as a negative.

| Response | Meaning | Billed |
|---|---|---|
| 200 with a negative field, such as a domain reported as not registered | the question is answered | yes |
| 200 with optional fields absent | normal, those fields are conditional | yes |
| 206 | partial success, use what returned and say what is missing | yes |
| 403 on an unsupported input | we cannot answer for this input | no |
| 404 | usually a wrong path or version, not missing data | no |

Check the schema's meaning before reporting an outcome. "No record exists", "we
do not support this input", and "you called the wrong path" are three different
answers, and only one of them is about the thing the user asked for.
