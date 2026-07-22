# Workspace guide

## Workspace layout
- Current workspace root: `/Users/alikianzadeh/git/chatbot-parent`
- The application projects are sibling directories under `/Users/alikianzadeh/git/`, not nested within each other. This allows for clear separation of concerns and easier management of each project.

## Projects
- `/Users/alikianzadeh/git/chatbot/` = Python FastAPI backend
- `/Users/alikianzadeh/git/netbot-ui/` = main React UI, this is frontend for `chatbot/`
- `/Users/alikianzadeh/git/chatbot-ui/` = React chatbot widget that uses api `/chats/events` in `chatbot/`
- `/Users/alikianzadeh/git/chatbot/aws/persistance.yaml` = AWS CloudFormation template for chatbot infrastructure
- `/Users/alikianzadeh/git/s3-sync/` = This is AWS SAM Project, includes AWS lambadas behind AWS API Gateway, user upload files using netbot-ui app on Knowledgebase, files upload directly to s3 with a presigned url and then and event bridge event trigger sync-file lambda to upload file to vendor like VoiceFlow. 


## Rules
- Keep changes scoped to the correct project.
- If backend API contracts change in `chatbot`, update clients in `netbot-ui` and/or `chatbot-ui` when needed.
- Do not mix frontend code between `netbot-ui` and `chatbot-ui`.
- Prefer minimal edits.
- We are processing migrating s3-sync project into chatbot project
- Before major edits, explain the plan briefly.

## Commands
### Backend
- `cd /Users/alikianzadeh/git/chatbot && uvicorn app.main:app --reload`

### Main frontend
- `cd /Users/alikianzadeh/git/netbot-ui && npm install`
- `cd /Users/alikianzadeh/git/netbot-ui && npm run dev`

### Widget frontend
- `cd /Users/alikianzadeh/git/chatbot-ui && npm install`
- `cd /Users/alikianzadeh/git/chatbot-ui && npm run build:widget:dev`

## Notes
- `chatbot-ui` is the chatbot widget project.
- Use `npm run build:widget:dev` when validating widget build changes.

## Support ticket process
- Support tickets are created by the backend chatbot agent flow in `/Users/alikianzadeh/git/chatbot/app/vendor/NetBot.py`.
- Each agent can configure `supportTicketPrompt`; this controls when the agent should offer or open a support ticket.
- Admins edit the ticket prompt in `/Users/alikianzadeh/git/netbot-ui/src/components/AdminAgentForm.js`.
- Customers edit the same ticket prompt in `/Users/alikianzadeh/git/netbot-ui/src/components/Agents.js`.
- The backend stores the prompt on `AgentModel.supportTicketPrompt` and exposes it through `app/view/Agent.py` and `app/controller/AgentController.py`.
- When a user asks for a ticket, accepts an offered ticket, or appears blocked/frustrated, `NetBot` classifies the intent with the agent's ticket prompt.
- A ticket is only created after an email address is available. If the user has not provided one, the conversation is marked `supportTicketState="awaiting_email"` and the bot asks for a follow-up email.
- Created tickets are stored in the DynamoDB table `${EnvName}-support-ticket` using `/Users/alikianzadeh/git/chatbot/app/model/SupportTicketModel.py`.
- Ticket IDs are per-client numeric sequence values stored with a `__counter__` item in the support-ticket table.
- Private customer ticket lookup routes are mounted at `/private/support-tickets`.
- Admin ticket lookup routes are mounted at `/admin/clients/{clientId}/support-tickets`.
- Support-ticket infrastructure lives in `/Users/alikianzadeh/git/chatbot/aws/persistence.yaml`.

## Live agent WebSocket background
- Live agent chat uses an API Gateway WebSocket API plus the FastAPI backend.
- The browser widget in `/Users/alikianzadeh/git/chatbot-ui/` connects to the WebSocket URL from `WSS_CHAT_URL`.
- The dashboard agent conversation UI in `/Users/alikianzadeh/git/netbot-ui/` connects to the WebSocket URL from `REACT_APP_WEBSOCKET_API_URL`.
- When a socket opens, the browser sends a small registration message to the WebSocket Lambda:
  - Widget/user side sends `{"sender":"user","conversationId":"..."}`
  - Dashboard/agent side sends `{"sender":"agent","conversationId":"..."}`
- The WebSocket Lambda is currently in `/Users/alikianzadeh/git/s3-sync/chat/app.py`. It receives `$default` messages and writes the API Gateway `connectionId` into the conversation DynamoDB item:
  - `userConnectionId` for widget/user sockets
  - `agentConnectionId` for dashboard/agent sockets
- The backend sends live messages through `/Users/alikianzadeh/git/chatbot/app/service/MessageService.py`, using the ECS env var `APIGATEWAY_DOMAIN_NAME` as the API Gateway Management API endpoint.
- The live-agent HTTP endpoints are in `chatbot/app/api/public_conversation.py` and `chatbot/app/controller/conversationController.py`:
  - `POST /conversations/{conversationId}/agent` requests handoff.
  - `POST /conversations/{conversationId}/join` marks an agent joined.
  - `POST /conversations/{conversationId}/messages` stores a message and forwards it over WebSocket to the target connection.
- Important production pitfall: all three places must use the same WebSocket API/custom domain:
  - Widget `WSS_CHAT_URL`
  - Dashboard `REACT_APP_WEBSOCKET_API_URL`
  - Backend ECS `APIGATEWAY_DOMAIN_NAME`
- Important production pitfall: the WebSocket Lambda must also write to the same environment conversation table as the backend:
  - Prod Lambda env `DYNAMODB_CONVERSATION_TABLE` must be `chatbot-prod-conversation`.
  - Dev Lambda env `DYNAMODB_CONVERSATION_TABLE` should be `chatbot-dev-conversation`.
  - `/Users/alikianzadeh/git/s3-sync/samconfig.toml` must pass `ConversationTable=chatbot-prod-conversation` for prod deploys; otherwise `s3-sync/template.yaml` falls back to its dev default.
- Current production WebSocket domain should be `chat.netbot.jp`. `chat.chishiki.link` is a separate older WebSocket API from the `sync-s3-vs` stack. If frontend connects to `chat.netbot.jp` but backend posts to `chat.chishiki.link`, HTTP requests can return `200` while the other browser never receives the live-agent message.
- The backend ECS value is passed by `/Users/alikianzadeh/git/chatbot/.github/workflows/main.yml` as the CloudFormation parameter `ApiGatewayDomainName`. Keep the parameter casing exact.
- When debugging live-agent delivery, check these first:
  - Browser Network tab has a WebSocket `101` connection to the expected domain.
  - CloudWatch WebSocket Lambda logs show both `sender:"user"` and `sender:"agent"` registration for the same `conversationId`.
  - WebSocket Lambda env has `DYNAMODB_CONVERSATION_TABLE=chatbot-prod-conversation` in prod.
  - DynamoDB conversation row contains `userConnectionId` and `agentConnectionId`.
  - ECS task env has `APIGATEWAY_DOMAIN_NAME=chat.netbot.jp`.
  - Backend logs do not show missing `ConnectionId` or posting to the wrong API Gateway domain.

## Deployment
- `chatbot/ecs-cloudformation.yml` deploys the backend to ECS Fargate as two separate services in the same ECS cluster.
- `Web` is the API service. Its task definition is `ECSTaskDefinitionWeb`, its ECS service is `ECSServiceWeb`, and it runs the image built from `chatbot/DockerfileWeb`.
- `DockerfileWeb` starts FastAPI with `uvicorn app.main:app --host 0.0.0.0 --port 8000`.
- `Web` is attached to the ALB target group on port `8000`, and the ALB health check path is `/health`.
- `Processor` is the background worker service. Its task definition is `ECSTaskDefinitionProcessor`, its ECS service is `ECSServiceProcessor`, and it runs the image built from `chatbot/DockerfileProcessor`.
- `DockerfileProcessor` starts the worker with `python -m app.processor`.
- `Processor` is intended to process SQS messages from the queue named by `BASE_PROCESS_QUEUE_NAME` and is not attached to the ALB.
- Both tasks receive shared app configuration from CloudFormation environment variables such as `EnvName`, `APP_ENV`, `UPLOAD_BUCKET`, `BASE_PROCESS_QUEUE_NAME`, Cognito settings, and `OPENSEARCH_ENDPOINT`.
- The processor task has higher Fargate resources than the web task in `ecs-cloudformation.yml`:
  Web = `512` CPU and `2048` MB memory.
  Processor = `2048` CPU and `4096` MB memory.
