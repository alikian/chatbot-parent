# Workspace guide

## Projects
- `chatbot/` = Python FastAPI backend
- `netbot-ui/` = main React UI
- `chatbot-ui/` = React chatbot widget
- `chatbot/aws/persistance.yaml` = AWS CloudFormation template for chatbot infrastructure

## Rules
- Keep changes scoped to the correct project.
- If backend API contracts change in `chatbot`, update clients in `netbot-ui` and/or `chatbot-ui` when needed.
- Do not mix frontend code between `netbot-ui` and `chatbot-ui`.
- Prefer minimal edits.
- Before major edits, explain the plan briefly.

## Commands
### Backend
- `cd chatbot && uvicorn app.main:app --reload`

### Main frontend
- `cd netbot-ui && npm install`
- `cd netbot-ui && npm run dev`

### Widget frontend
- `cd chatbot-ui && npm install`
- `cd chatbot-ui && npm run build:widget:dev`

## Notes
- `chatbot-ui` is the chatbot widget project.
- Use `npm run build:widget:dev` when validating widget build changes.

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
