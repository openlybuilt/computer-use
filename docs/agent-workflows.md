# Build and debug workflows as a coding agent

This guide is for an assistant or coding agent operating an Openly installation. Use the local **backend** API to create and run workflows; do not call the agents service or VM providers directly. The backend owns workflow versions, placeholder resolution, run state, recordings, and history. The agents service executes steps.

Work in the user's local workspace and obtain the user's authorization before starting a real run or using provider resources. Real workflows can interact with external websites and applications, consume provider/model credits, and produce recordings of sensitive screens. The interactive guide's prerecorded run is only an example; it does not exercise a provider.

## Prepare

Start the stack as described in the [main README](../README.md). The examples below use `curl`, `jq`, Node.js, and the backend's default loopback address:

```sh
OPENLY_BACKEND=http://127.0.0.1:3000
curl -fsS "$OPENLY_BACKEND/ready"
curl -fsS "$OPENLY_BACKEND/provider-capabilities" | jq .
```

Choose a configured provider and supported target from the response. OnKernel currently offers `browser`; Daytona offers `windows`. If a provider is not configured or the services are unavailable, do not invent a provider or silently switch to another. Use the UI or backend configuration to resolve the issue.

## Create and run an example

The following is a minimal Browser workflow against `example.com`. It creates a real saved workflow; starting the run uses the configured provider. Change the provider/target only to a supported combination, and do not run it unless the user wants that action.

```sh
cat > /tmp/openly-workflow.json <<'JSON'
{
  "name": "Visit example.com",
  "description": "Open a page and wait briefly.",
  "executionTarget": "browser",
  "provider": "onkernel",
  "inputs": [],
  "steps": [
    { "type": "navigate", "url": "https://example.com" },
    { "type": "wait", "seconds": 1 }
  ]
}
JSON

curl -fsS -H 'Content-Type: application/json' \
  --data-binary @/tmp/openly-workflow.json \
  "$OPENLY_BACKEND/workflows" > /tmp/openly-created.json

OPENLY_WORKFLOW_ID=$(jq -r '.id' /tmp/openly-created.json)
OPENLY_VERSION=$(jq -r '.version' /tmp/openly-created.json)
OPENLY_RUN_ID=$(node -e "console.log(require('node:crypto').randomUUID())")
```

Create the run using that saved version:

```sh
jq -n --arg id "$OPENLY_RUN_ID" --argjson version "$OPENLY_VERSION" \
  '{id: $id, version: $version, inputs: {}, timeoutMs: 120000}' \
  > /tmp/openly-run.json

curl -fsS -H 'Content-Type: application/json' \
  --data-binary @/tmp/openly-run.json \
  "$OPENLY_BACKEND/workflows/$OPENLY_WORKFLOW_ID/runs" | jq .
```

The endpoint returns `202` when the request is accepted; this is not proof of success. Poll `GET /runs/{id}` until the run reaches a terminal status, and inspect the step results and `result`. `completed`, `failed`, `cancelled`, `interrupted`, and `outcome_unknown` have different meanings; do not report success for an uncertain or failed run.

```sh
curl -fsS "$OPENLY_BACKEND/runs/$OPENLY_RUN_ID" | jq .
curl -fsS "$OPENLY_BACKEND/workflows/$OPENLY_WORKFLOW_ID/runs" | jq .
```

A run uses the exact saved version supplied in the request. The permitted workflow timeout is up to one hour; cleanup and recording finalization may continue afterward. Stop a real run only when the user requests it or it is part of the authorized task:

```sh
curl -fsS -X POST "$OPENLY_BACKEND/runs/$OPENLY_RUN_ID/cancel" | jq .
```

## Inspect logs and recordings

```sh
curl -fsS "$OPENLY_BACKEND/runs/$OPENLY_RUN_ID/events" | jq .
curl -fsS "$OPENLY_BACKEND/runs/$OPENLY_RUN_ID" | jq '{status, result, steps, recordingStatus}'
```

Events contain sanitized step, model-call, and action metadata: sequence, timestamp, status, duration, and token counts when available. The product calls this section **Logs**. These are not full model traces: prompts, page contents, screenshots, and action values are not included. Event delivery is best effort; an empty event list or `logsPartial` does not prove nothing happened.

If `recordingStatus` is `saved`, the backend exposes an MP4 recording. Save it only where the user has authorized, because it may contain sensitive information:

```sh
curl -fsS "$OPENLY_BACKEND/runs/$OPENLY_RUN_ID/recording" -o /tmp/openly-run-recording.mp4
```

If FFmpeg is installed, an agent can extract a frame and inspect the resulting image with its available image-viewing tool:

```sh
ffmpeg -nostdin -ss 5 -i /tmp/openly-run-recording.mp4 -frames:v 1 /tmp/openly-run-frame.png
```

Choose a timestamp within the recording. Playback or frame extraction should follow the user's authorization and the agent environment's computer-use rules. Remove temporary copies when they are no longer needed.

A recording may be pending, disabled, unavailable, deleted, or failed. A missing recording does not prove the workflow failed. Native Daytona recording has been unreliable in prior validation. Run history may also contain entries marked `example: true`: those replay the bundled guide video in the platform and are not agent executions or backend run recordings.

## Iterate safely

1. Read the current saved workflow with `GET /workflows/{id}`. Its response includes `version` and `definition`.
2. Modify the definition, then `POST /workflows/{id}/versions` with JSON and an `If-Match` header containing the version you read. A `409` means another edit won; reload and reapply the intended change rather than overwriting it.
3. Start a new run using the returned version and a new UUID. Reusing a run ID with different inputs returns a conflict; retries with the same ID are for the same request.
4. Compare the failed step, its message, and relevant events. View the recording if available, make the smallest change, and repeat only with the user's authorization.

Supported step types are `navigate`, `type`, `wait`, `scroll`, `press_keys`, `click`, and `ai`. Browser scroll steps use pixels; Windows scroll steps use wheel ticks, and Daytona currently does not support horizontal wheel input. A click can use a plain-language target or explicit screenshot pixel coordinates. Targeted text steps can include a field description and region. See the definitions and validation in the backend and agents repos rather than assuming other step shapes are supported.

Use saved placeholders for sensitive inputs rather than hard-coding them in workflow definitions. Define the input names in the workflow and use `{{name}}` references in supported fields. Configure values in Settings or through the backend placeholder API. Runs with secret placeholders require `acknowledgeSecretExposure: true`; do not set that on the user's behalf without authorization. Secrets may still be visible in the VM and its recording.

If the database becomes unavailable, the backend blocks new work and stops active runs safely; outcomes are reconciled after recovery. If provider setup or a browser action fails, inspect run status, safe events, and recording status. Do not claim that a workflow worked merely because it was accepted, or that a missing log proves an action did not occur.
