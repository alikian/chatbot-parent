# NetBot Architecture

```mermaid
flowchart LR
    AdminUser(["Admin User"])
    PublicUser(["Public User"])

    subgraph AWS["AWS — Japan Region (ap-northeast-1)"]
        Amplify["AWS Amplify<br/>Dashboard + Widget Hosting"]
        ReactUI["React UI<br/>Admin Dashboard"]
        Widget["React Chat Widget<br/>Customer Website"]
        Cognito["Amazon Cognito<br/>Authentication"]
        ALB["Application Load Balancer"]

        subgraph ECS["Amazon ECS Fargate"]
            API["FastAPI Backend"]
            Processor["Document Processor"]
        end

        S3[("Amazon S3<br/>Documents")]
        SQS["Amazon SQS<br/>Processing Queue"]
        DynamoDB[("Amazon DynamoDB<br/>Persistence")]
        Bedrock["Amazon Bedrock<br/>LLM + Embeddings"]
    end

    subgraph External["External Service"]
        Gemini["Google Gemini<br/>OCR + Chunking"]
    end

    AdminUser -->|"Manage agents & knowledge base"| ReactUI
    PublicUser -->|"Chat & request support"| Widget

    Amplify -->|"Serves dashboard"| ReactUI
    Amplify -->|"Serves widget bundle"| Widget

    ReactUI <-->|"Authentication + JWT"| Cognito
    API -.->|"Validate JWT"| Cognito

    ReactUI -->|"Admin / Private API"| ALB
    Widget -->|"Public Chat API"| ALB
    ALB --> API

    ReactUI -->|"Direct document upload"| S3
    API -->|"Enqueue processing job"| SQS
    SQS -->|"Deliver job"| Processor
    Processor -->|"Fetch document"| S3

    API <-->|"Read / Write"| DynamoDB
    Processor -->|"Store processing results"| DynamoDB

    API -.->|"Generate responses"| Bedrock
    Processor -.->|"Create embeddings"| Bedrock
    Processor -.->|"OCR + chunking"| Gemini

    classDef admin fill:#dcfce7,stroke:#15803d,color:#14532d
    classDef public fill:#ccfbf1,stroke:#0f766e,color:#134e4a
    classDef dashboard fill:#bbf7d0,stroke:#16a34a,color:#14532d
    classDef compute fill:#ede9fe,stroke:#7c3aed,color:#4c1d95
    classDef storage fill:#dbeafe,stroke:#2563eb,color:#1e3a8a
    classDef queue fill:#cffafe,stroke:#0891b2,color:#164e63
    classDef auth fill:#fce7f3,stroke:#db2777,color:#831843
    classDef gateway fill:#fef3c7,stroke:#d97706,color:#78350f
    classDef hosting fill:#ffedd5,stroke:#ea580c,color:#7c2d12
    classDef ai fill:#f3e8ff,stroke:#9333ea,color:#581c87
    classDef external fill:#fee2e2,stroke:#dc2626,color:#7f1d1d

    class AdminUser admin
    class PublicUser public
    class ReactUI dashboard
    class Widget public
    class API,Processor compute
    class S3,DynamoDB storage
    class SQS queue
    class Cognito auth
    class ALB gateway
    class Amplify hosting
    class Bedrock ai
    class Gemini external

    style AWS fill:#f8fafc,stroke:#f59e0b,stroke-width:3px
    style ECS fill:#faf5ff,stroke:#8b5cf6,stroke-width:2px
    style External fill:#fff7ed,stroke:#ef4444,stroke-width:2px,stroke-dasharray:5 5
```

> [!IMPORTANT]
> Google Gemini is outside AWS. Before sending customer documents to it, verify that its configured endpoint stores and processes all data exclusively in Google Cloud's Japan region (`asia-northeast1`).
