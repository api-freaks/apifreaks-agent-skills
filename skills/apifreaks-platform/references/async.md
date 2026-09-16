# Asynchronous jobs

Some PDF operations do not return a result. They return a `taskId` and run in the
background. Treating that response as the result, or polling it wrongly, is the
most common way to waste time on this platform.

If any other endpoint returns a `taskId` rather than data, the same rules apply.
The endpoint's `reference.md` says which responses it returns.

## Table of contents

- [The task lifecycle](#the-task-lifecycle)
- [Polling](#polling)
- [Retrieving output](#retrieving-output)
- [Failed tasks](#failed-tasks)
- [Expiry](#expiry)
- [Webhooks](#webhooks)

## The task lifecycle

```
POST /pdf/<operation>      → { taskId, inputIds }
        ↓
GET  /pdf/task-status?task_id=<id>   → { status: ... }
        ↓ completed
     outputUrls / outputIds
        ↓
GET  /pdf/resource/download?resource_id=<id>
```

The first response contains no output. It confirms the job was accepted and names
the inputs it will work on. Do not report success to the user at this point; the
job can still fail.

## Polling

```bash
curl -s "https://api.apifreaks.com/v1.0/pdf/task-status?task_id=$TASK" \
  -H "X-apiKey: $APIFREAKS_API_KEY"
```

`task_id` is required. A completed task returns:

```json
{
  "taskId": "74004a68-...",
  "status": "completed",
  "createdAt": "2026-07-27 10:48:51",
  "outputUrls": ["https://api.apifreaks.com/v1.0/pdf/resource/download?resource_id=..."],
  "outputIds": ["21e3bb30-..."],
  "inputIds": ["0deab2f6-...", "5b6a4836-..."]
}
```

Poll with a delay and a ceiling. Start at a few seconds, back off, and cap the
total wait. Tight polling loops burn time without making the job finish sooner,
and an unbounded loop hangs the task when something upstream is stuck.

If the wait exceeds a reasonable ceiling, tell the user the task is still running
and give them the `taskId` so they can check later. Do not silently keep polling.

Separately, `GET /pdf/file-status?file_id=<id>` reports on an individual file
rather than a task. Use task status for the job and file status for a specific
input or output.

## Retrieving output

`outputUrls` are ready to fetch and already carry the resource id. `outputIds`
name the same resources if you prefer to build the download call yourself:

```
GET /pdf/resource/download?resource_id=<id>
```

Download before doing anything else with the result, because resources expire
(see below). Save to disk rather than holding the content in memory, since PDFs
are binary and often large.

Related endpoints: `/pdf/resource/upload` and `/pdf/resource/upload-binary` to
put a file in, `/pdf/files` to list what is stored, and `/pdf/file` with DELETE to
remove one. That DELETE is the platform's one non-GET, non-POST method.

## Failed tasks

A failed task returns a 200 with `status: failed` and the reason in the body:

```json
{
  "taskId": "9d8d803b-...",
  "status": "failed",
  "error": "Invalid Page Range",
  "message": "One or more specified page numbers exceed the total number of pages in the PDF document.",
  "expiresAt": "2026-08-03 10:52:12"
}
```

This matters: **the HTTP status is 200 even though the job failed.** An agent
checking only status codes reports success and hands the user nothing. Always
read the `status` field.

Report the `message` as written. It usually names the exact problem, such as a
page range exceeding the document length, which the user can fix and retry.

## Expiry

Tasks and their resources carry an `expiresAt`, typically about a week out. After
that the output is gone and the job must be rerun, which costs credits again.

Download outputs as soon as a task completes. If you are handing a `taskId` back
to a user for later, tell them the expiry date rather than leaving them to
discover it.

## Webhooks

Where an endpoint accepts a webhook URL, the platform calls back on completion
instead of being polled. Check the `reference.md` for whether the endpoint supports
one and for the parameter name, since it is not offered everywhere.

Use a webhook when the caller is a service that can receive one. In an
interactive agent session there is usually nowhere for the callback to land, so
polling with a ceiling is the right choice, and offering a webhook to a user who
has no endpoint to receive it only confuses the task.
