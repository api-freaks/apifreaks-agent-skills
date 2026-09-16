# APIFreaks error reference

Everything here is taken from the per-endpoint `reference.md` files. Each
endpoint declares its own codes and messages, and those always override this
file. Read the endpoint's `reference.md` before acting on an unfamiliar error.

## Error body

Failed requests return a standard envelope:

| Field | Type | Required | Description |
|---|---|---|---|
| `message` | string | yes | Human-readable description of the failure |
| `error` | string | no | Short error category or exception type |
| `path` | string | no | The endpoint path that produced the error |
| `status` | integer | no | HTTP status code |
| `timestamp` | string | no | When the error occurred, ISO 8601 |

`message` is the only guaranteed field and usually names the specific problem.
Surface it as written rather than paraphrasing.

Credits are charged only for successful queries, defined as a 2xx status. A 4xx
or 5xx deducts nothing, and anything already charged is refunded.

## The same code means different things on different endpoints

This is the most important thing in this file. Status codes are reused across the
platform with unrelated meanings, so acting on the code alone produces wrong
answers.

**402** is not always about credits. On the commodity endpoints it means the
request asked for more symbols than allowed, which the user fixes by requesting
fewer. Read the message before telling anyone to buy credits.

**403** covers both permission problems and unsupported inputs.

Inside an organization, members share one key and one balance, and roles differ:
an admin manages everything, a maintainer manages most things but is still
subject to organization-wide endpoint restrictions, and a member is read-only.
So a 403 naming a role or a restriction is the account's configuration, not the
request, and the user needs their admin rather than a new key. Personal accounts
have none of this.
 "You need to be
organization admin to access this endpoint" is a permissions problem. "Unavailable
domain extension" and "We are not providing the whois of this domain extension"
mean the input is out of scope, which is a data limitation and not something the
user can authorise away.

**404** covers both a wrong URL and a genuine absence. "The requested resource
could not be found. Please verify the URL and try again" is about the request.
"Location not found", "No Ssl Certificate exists for entered Domain", and
"Rates of provided currency are not available in our database" are about the
data. Check the version and path before reporting absence.

## Codes seen across the catalog

| Status | Typical meaning | Handling |
|---|---|---|
| 206 | Partial response: some of the request succeeded | Use what returned, say what is missing. Billed |
| 400 | Invalid or missing parameters, bad body, bad file ID | Fix against the `reference.md`. Never retry unchanged |
| 401 | API key invalid | Stop. Never retry |
| 402 | Varies by endpoint, including request limits | Read the message before deciding whether it is fixable |
| 403 | Permissions, account state, or unsupported input | Read the message. Do not retry |
| 404 | Wrong path or version, or the data does not exist | Re-check version and path first |
| 408 | Timed out reaching a remote resource | Retry once or twice with backoff |
| 413 | Request body over the size limit | Split the batch |
| 415 | File type or `Content-Type` not supported | Fix the input format. Never retry unchanged |
| 422 | Request contains invalid or missing data | Fix against the `reference.md` |
| 423 | Locked: on IP endpoints, a bogon or reserved address | Not retryable. The input is not a routable address |
| 429 | Request limit reached | Back off, reduce parallelism, retry |
| 500 | The operation failed server-side | Retry with backoff |
| 503 | Service temporarily unavailable | Retry with backoff, then check https://status.apifreaks.com/ |
| 504 | Processing exceeded the time limit | Reduce the input size and retry |

Not every endpoint declares every code. The `reference.md` lists what that
endpoint can return.

## Notes on specific codes

**408** appears on endpoints that fetch from third parties, such as scraping and
WHOIS. It means the remote side was slow, not that the request was wrong, so a
retry is reasonable where a 400 retry would not be.

**504** on PDF work means the job passed the processing time limit. Retrying the
same input repeats the same outcome; reduce the input size instead.

**415** appears on file endpoints and covers both an unsupported file type and an
unsupported `Content-Type` header. The `reference.md` lists accepted types.

**423** appears on IP endpoints for bogon and reserved addresses. Explain that
the address is not publicly routable rather than reporting a lookup failure.

## Successful responses that are not successes

| Case | Looks like | Actually |
|---|---|---|
| A negative field in a 200 body | success | the question is answered, and it is billed |
| An async task reporting `status: failed` | 200 | the job failed; read `error` and `message` |
| Optional fields absent | 200 with gaps | normal, those fields are conditional |
| 206 | partial | some records succeeded, some did not |

Checking the HTTP status alone misses all four. Read the body against the schema.

## Retry decisions

Operational guidance rather than platform documentation. When in doubt, read the
message.

| Status | Retry? |
|---|---|
| 400, 415, 422 | Only after fixing the request |
| 401 | Never |
| 402, 403 | Only if the message describes something fixable in the request |
| 404 | Only after correcting path or version |
| 408 | Yes, once or twice with backoff |
| 413 | Yes, with a smaller batch |
| 429 | Yes, with backoff and lower parallelism |
| 500, 503 | Yes, with backoff, then check https://status.apifreaks.com/ |
| 504 | Only with a smaller input |

## Headers

| Header | Meaning |
|---|---|
| `X-AF-Credits-Cost` | Credits deducted by this request. Present on error responses too |
| `X-Concurrent-Threads` | The plan's concurrent request ceiling |
| `X-Concurrent-Threads-Active` | Concurrent requests in flight |

Support: https://apifreaks.com/contact
