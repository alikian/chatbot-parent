# Workspace guide

## How to use this file
This file tells you **where to look**, not what the answers are. Code and
templates are the source of truth; this file is only orientation. If anything
here conflicts with the code, trust the code and flag the discrepancy.

## Accuracy rules (read first)
- Never invent endpoints, env var names, file paths, or npm scripts. Confirm
  with Read/Grep before referencing them.
- API contracts: the FastAPI route definitions under `chatbot/app/api/` are
  authoritative. Do not assume an endpoint exists from the client side.
- Available npm scripts: check the `scripts` section of each project's
  `package.json` — do not guess script names.
- Deployment values (ports, CPU/memory, env vars, container commands): read
  the CloudFormation/SAM templates listed below — do not quote from memory.
- If you cannot verify a fact, say so instead of asserting it.

## Conversation accuracy
- Treat facts established earlier in a long conversation as possibly stale:
  before acting on a file path, API shape, or config value from many turns
  ago, re-verify it against the code.
- When a request is ambiguous, state the interpretation you are acting on
  ("I'm reading this as X") before making changes, so a wrong guess is cheap
  to correct.
- Report outcomes as observations, not predictions: "tests pass" only after
  running them, "fixed" only after verifying. If something was skipped or
  failed, say so plainly.
- Distinguish what you verified from what you inferred. Prefix unverified
  claims with "likely" or "I haven't confirmed this".

## Workspace layout
- Workspace root: `/Users/alikianzadeh/git/chatbot-parent`
- Application projects are sibling directories under `/Users/alikianzadeh/git/`,
  not nested within each other.

## Projects
- `/Users/alikianzadeh/git/chatbot/` — Python FastAPI backend.
  - App code in `app/` (routes in `app/api/`, business logic in `app/service/`
    and `app/controller/`, models in `app/model/`).
  - Background worker entrypoint: `app/processor.py`.
  - Tests in `tests/`.
- `/Users/alikianzadeh/git/netbot-ui/` — main React UI; frontend for `chatbot/`.
- `/Users/alikianzadeh/git/chatbot-ui/` — React chatbot widget; talks to the
  `/chats/events` API in `chatbot/`.
- `/Users/alikianzadeh/git/s3-sync/` — AWS SAM project (lambdas behind API
  Gateway). Users upload files from netbot-ui's Knowledgebase page directly to
  S3 via presigned URL; an EventBridge event then triggers the sync-file lambda
  to push the file to a vendor (e.g. VoiceFlow). Architecture source of truth:
  `s3-sync/template.yaml`.

## Infrastructure source-of-truth files
Do not restate values from these; read them when needed.
- `chatbot/ecs-cloudformation.yml` — deploys the backend to ECS Fargate as two
  services in one cluster: `Web` (the API, behind the ALB, built from
  `chatbot/DockerfileWeb`) and `Processor` (SQS background worker, not behind
  the ALB, built from `chatbot/DockerfileProcessor`). Ports, health checks,
  task sizes, and env vars: read the template.
- `chatbot/aws/persistence.yaml` — CloudFormation template for chatbot
  persistence infrastructure (note spelling: persist**e**nce).
- `s3-sync/template.yaml` — SAM template for the upload/sync pipeline.

## Migration in progress (as of June 2026)
The `s3-sync` project is being migrated into the `chatbot` project. Before
adding or changing upload/sync functionality, check both projects to see where
the relevant code currently lives, and ask if ownership is unclear. Update this
section when the migration status changes.

## Rules
- Keep changes scoped to the correct project.
- If backend API contracts change in `chatbot`, update clients in `netbot-ui`
  and/or `chatbot-ui` as needed.
- Do not mix frontend code between `netbot-ui` and `chatbot-ui`.
- Prefer minimal edits.
- Before major edits, explain the plan briefly.
- After changes, run the relevant tests/build below and report actual results —
  do not claim success without verifying.

## Commands
Verified against each project as of June 2026. If a command fails, check the
project's `package.json` / README rather than guessing alternatives.

### Backend (`chatbot`)
- Run: `cd /Users/alikianzadeh/git/chatbot && uvicorn app.main:app --reload`
- Test: `cd /Users/alikianzadeh/git/chatbot && python -m pytest tests/`
  (pytest is not pinned in `requirements.txt`; install it in your env if missing)

### Main frontend (`netbot-ui`)
- Install: `cd /Users/alikianzadeh/git/netbot-ui && npm install`
- Run: `npm run start` (or `npm run start:local`) — there is **no** `dev` script
- Test: `npm test`
- Build: `npm run build`

### Widget frontend (`chatbot-ui`)
- Install: `cd /Users/alikianzadeh/git/chatbot-ui && npm install`
- Run: `npm run dev`
- Build for validating widget changes: `npm run build:widget:dev`
- Lint: `npm run lint`
