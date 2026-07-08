# Chatbot Workspace Architecture

This workspace contains sibling projects under `/Users/alikianzadeh/git/`. The frontend apps, backend API, widget, and AWS upload/sync services are separate projects that integrate through HTTP APIs, Cognito authentication, S3 events, SQS, and external vendor APIs.

## System Context

```mermaid
flowchart LR
    Admin["Admin user"]
    Customer["Customer user"]
    SiteVisitor["Website visitor"]

    NetbotUI["netbot-ui\nMain React UI"]
    Widget["chatbot-ui\nEmbeddable chat widget"]
    Backend["chatbot\nFastAPI backend"]
    Processor["chatbot Processor\nBackground ECS service"]
    Infra["chatbot/aws/persistance.yaml\nAWS persistence infrastructure"]
    ECS["chatbot/ecs-cloudformation.yml\nECS Fargate deployment"]
    S3Sync["s3-sync\nSAM upload/sync service"]

    Cognito["Amazon Cognito"]
    ALB["Application Load Balancer"]
    S3["S3 upload bucket"]
    EventBridge["EventBridge"]
    SQS["SQS queue\nBASE_PROCESS_QUEUE_NAME"]
    OpenSearch["OpenSearch"]
    Vendor["Vendor integrations\nVoiceFlow / others"]

    Admin -->|"manage clients, plans, vector stores"| NetbotUI
    Customer -->|"manage knowledge bases, files, conversations"| NetbotUI
    SiteVisitor -->|"chat"| Widget

    NetbotUI -->|"Cognito login"| Cognito
    Widget -->|"chat events\n/chats/events"| Backend
    NetbotUI -->|"API requests"| ALB
    ALB -->|"port 8000"| Backend

    Backend -->|"auth validation / user context"| Cognito
    Backend -->|"presigned upload URLs / metadata"| S3
    Backend -->|"enqueue work"| SQS
    Backend -->|"search / vector data"| OpenSearch
    Backend -->|"persistent app data"| Infra

    Processor -->|"consume messages"| SQS
    Processor -->|"read/write files"| S3
    Processor -->|"index/query data"| OpenSearch

    NetbotUI -->|"direct file upload using presigned URL"| S3
    S3 -->|"object events"| EventBridge
    EventBridge -->|"trigger sync-file lambda"| S3Sync
    S3Sync -->|"sync uploaded files"| Vendor

    ECS --> Backend
    ECS --> Processor
```

## Runtime Services

```mermaid
flowchart TB
    subgraph Frontend["Frontend projects"]
        NetbotUI["netbot-ui\nReact admin/customer UI"]
        ChatbotUI["chatbot-ui\nReact widget build"]
    end

    subgraph BackendProject["chatbot project"]
        FastAPI["Web service\nDockerfileWeb\nuvicorn app.main:app"]
        Worker["Processor service\nDockerfileProcessor\npython -m app.processor"]
        Persistence["CloudFormation persistence\naws/persistance.yaml"]
        ECSDeploy["ECS CloudFormation\necs-cloudformation.yml"]
    end

    subgraph AWS["AWS runtime"]
        ALB["ALB\nhealth check /health"]
        WebTask["ECSServiceWeb\n512 CPU / 2048 MB"]
        ProcessorTask["ECSServiceProcessor\n2048 CPU / 4096 MB"]
        Cognito["Cognito"]
        UploadBucket["UPLOAD_BUCKET"]
        Queue["SQS\nBASE_PROCESS_QUEUE_NAME"]
        Search["OPENSEARCH_ENDPOINT"]
    end

    subgraph UploadSync["s3-sync project"]
        ApiGateway["API Gateway"]
        PresignLambda["presigned upload URL lambda"]
        SyncLambda["sync-file lambda"]
    end

    NetbotUI -->|"REST API"| ALB
    ChatbotUI -->|"POST /chats/events"| ALB
    ALB --> WebTask
    WebTask --> FastAPI
    ProcessorTask --> Worker

    FastAPI --> Cognito
    FastAPI --> UploadBucket
    FastAPI --> Queue
    FastAPI --> Search
    Worker --> Queue
    Worker --> UploadBucket
    Worker --> Search

    NetbotUI -->|"Knowledgebase uploads"| ApiGateway
    ApiGateway --> PresignLambda
    PresignLambda --> UploadBucket
    UploadBucket --> SyncLambda
    SyncLambda -->|"vendor upload"| Vendor["VoiceFlow / vendor APIs"]

    ECSDeploy --> WebTask
    ECSDeploy --> ProcessorTask
    Persistence --> Cognito
    Persistence --> UploadBucket
    Persistence --> Queue
    Persistence --> Search
```

## Live Agent WebSocket Flow

Live agent chat combines normal HTTP endpoints in `chatbot` with an API Gateway WebSocket API currently defined in the `s3-sync` project.

```mermaid
sequenceDiagram
    participant Visitor as Website visitor widget
    participant Agent as Dashboard agent UI
    participant WS as API Gateway WebSocket
    participant WSLambda as s3-sync chat Lambda
    participant DDB as Conversation DynamoDB table
    participant API as chatbot FastAPI
    participant Mgmt as API Gateway Management API

    Visitor->>API: POST /conversations/{id}/agent
    API->>DDB: Set agentRequestedAt
    Visitor->>WS: Connect to WSS_CHAT_URL
    Visitor->>WSLambda: {"sender":"user","conversationId":"..."}
    WSLambda->>DDB: Save userConnectionId

    Agent->>API: POST /conversations/{id}/join
    API->>DDB: Set agent and updatedAt
    Agent->>WS: Connect to REACT_APP_WEBSOCKET_API_URL
    Agent->>WSLambda: {"sender":"agent","conversationId":"..."}
    WSLambda->>DDB: Save agentConnectionId

    Agent->>API: POST /conversations/{id}/messages
    API->>DDB: Store conversation history
    API->>Mgmt: post_to_connection(userConnectionId)
    Mgmt-->>Visitor: Live agent message

    Visitor->>API: POST /conversations/{id}/messages
    API->>DDB: Store conversation history
    API->>Mgmt: post_to_connection(agentConnectionId)
    Mgmt-->>Agent: Visitor message
```

### Live Agent Components

- Widget WebSocket URL: `/Users/alikianzadeh/git/chatbot-ui/` uses `WSS_CHAT_URL`.
- Dashboard WebSocket URL: `/Users/alikianzadeh/git/netbot-ui/` uses `REACT_APP_WEBSOCKET_API_URL`.
- WebSocket registration Lambda: `/Users/alikianzadeh/git/s3-sync/chat/app.py`.
- Backend WebSocket sender: `/Users/alikianzadeh/git/chatbot/app/service/MessageService.py`.
- Live agent HTTP controller: `/Users/alikianzadeh/git/chatbot/app/controller/conversationController.py`.
- Live agent public routes: `/Users/alikianzadeh/git/chatbot/app/api/public_conversation.py`.

### Production Domain Contract

The widget, dashboard, and backend must all use the same API Gateway WebSocket custom domain.

- Production WebSocket domain: `chat.netbot.jp`.
- Older/separate WebSocket domain: `chat.chishiki.link` from the `sync-s3-vs` stack.
- Backend ECS env var: `APIGATEWAY_DOMAIN_NAME` must match the frontend WebSocket domain.
- GitHub Actions passes this into `chatbot/ecs-cloudformation.yml` from `chatbot/.github/workflows/main.yml` using the CloudFormation parameter `ApiGatewayDomainName`.

If the browser connects to `chat.netbot.jp` but the backend posts via `chat.chishiki.link`, HTTP requests can still return `200`, but live messages will not arrive because the connection IDs belong to a different API Gateway WebSocket API.

When debugging live-agent delivery, verify:

- Browser Network tab shows a WebSocket `101` to the expected domain.
- WebSocket Lambda logs show both `sender:"user"` and `sender:"agent"` for the same `conversationId`.
- The conversation DynamoDB row has `userConnectionId` and `agentConnectionId`.
- ECS task env has `APIGATEWAY_DOMAIN_NAME=chat.netbot.jp`.
- Backend logs do not show missing `ConnectionId` or posting to the wrong API Gateway domain.

## Knowledge Base Upload Flow

```mermaid
sequenceDiagram
    participant User as Customer user
    participant UI as netbot-ui
    participant API as chatbot FastAPI
    participant S3Sync as s3-sync API Gateway/Lambda
    participant S3 as S3 upload bucket
    participant EB as EventBridge
    participant Vendor as VoiceFlow / vendor
    participant Worker as chatbot Processor
    participant Search as OpenSearch

    User->>UI: Add document in Knowledgebase / Data Management
    UI->>S3Sync: Request presigned upload URL
    S3Sync-->>UI: Presigned S3 URL
    UI->>S3: Upload file directly
    S3-->>EB: Object-created event
    EB->>S3Sync: Trigger sync-file lambda
    S3Sync->>Vendor: Upload/sync file to vendor

    UI->>API: Refresh/list documents
    API->>S3: Read document metadata
    API->>Search: Read indexed/vector state
    API-->>UI: Document list and status

    API->>Worker: Queue processing work via SQS
    Worker->>S3: Read uploaded documents
    Worker->>Search: Index/update vector data
```

## Notes

- `netbot-ui` is the main authenticated React UI for admins, customers, and members.
- `chatbot-ui` is the embeddable chatbot widget and calls the backend `/chats/events` API.
- `chatbot` deploys as two ECS Fargate services: `Web` for FastAPI and `Processor` for background SQS work.
- `s3-sync` is currently a separate SAM project and is being migrated into `chatbot`.
- Backend API contract changes in `chatbot` should be reflected in `netbot-ui` and/or `chatbot-ui` when needed.
