# Azure Multi-modal Compliance Ingestion Engine

An Azure-based video compliance auditing pipeline that ingests YouTube videos, extracts transcript and OCR signals with Azure Video Indexer, retrieves policy context from Azure AI Search, and uses Azure OpenAI inside a LangGraph workflow to produce a compliance report.

The project is also referred to in the code as **Brand Guardian AI**.

![Azure Multi-modal Compliance Ingestion Engine Architecture](Project2_Langgraph_Architecture.png)

## What This Project Does

This engine audits marketing or influencer-style video content against compliance guidance stored in PDF documents.

The workflow:

1. Accepts a YouTube video URL from the CLI or FastAPI backend.
2. Downloads the video locally with `yt-dlp`.
3. Uploads the video to Azure Video Indexer.
4. Extracts transcript and OCR text from the processed video.
5. Queries Azure AI Search for the most relevant compliance rules.
6. Sends the transcript, OCR text, metadata, and retrieved rules to Azure OpenAI.
7. Returns a structured PASS/FAIL audit report with detected compliance issues.

## Architecture

The architecture is organized into four main areas:

- **Entry Points**
  - `main.py` provides a CLI-style simulation.
  - `backend/src/api/server.py` exposes a FastAPI API with `/audit` and `/health`.

- **Orchestration**
  - `backend/src/graph/workflow.py` defines the LangGraph workflow.
  - `backend/src/graph/nodes.py` contains the video indexing and compliance auditing nodes.
  - `backend/src/graph/state.py` defines the shared graph state.

- **Azure Infrastructure**
  - Azure Video Indexer extracts transcript and OCR insights.
  - Azure AI Search stores vectorized compliance policy chunks.
  - Azure OpenAI provides chat completion and embedding models.

- **Observability**
  - Azure Application Insights / Azure Monitor telemetry is configured in `backend/src/api/telemetry.py`.
  - LangSmith can be enabled through environment variables for LangGraph tracing.

## Repository Structure

```text
.
|-- main.py                              # CLI simulation entry point
|-- Project2_Langgraph_Architecture.png  # Architecture diagram
|-- pyproject.toml                       # Python project dependencies
|-- README.md
|-- backend/
|   |-- data/                            # Compliance PDFs used for RAG indexing
|   |-- scripts/
|   |   `-- index_documents.py           # Indexes PDFs into Azure AI Search
|   `-- src/
|       |-- api/
|       |   |-- server.py                # FastAPI app
|       |   `-- telemetry.py             # Azure Monitor setup
|       |-- graph/
|       |   |-- workflow.py              # LangGraph definition
|       |   |-- nodes.py                 # Indexer and auditor nodes
|       |   `-- state.py                 # Typed graph state
|       `-- services/
|           `-- video_indexer.py         # Azure Video Indexer integration
`-- azure_functions/                     # Placeholder for Azure Functions deployment files
```

## Prerequisites

- Python 3.12+
- `uv` for dependency management
- Azure CLI authenticated with an identity that can access Azure Video Indexer
- Azure resources:
  - Azure OpenAI with a chat deployment
  - Azure OpenAI embedding deployment, such as `text-embedding-3-small`
  - Azure AI Search index
  - Azure Video Indexer account
  - Optional: Azure Application Insights

Authenticate to Azure before running video indexing:

```bash
az login
```

## Setup

Install dependencies:

```bash
uv sync
```

Create a `.env` file in the project root:

```env
# Azure OpenAI
AZURE_OPENAI_ENDPOINT=https://<your-openai-resource>.openai.azure.com/
AZURE_OPENAI_API_KEY=<your-azure-openai-key>
AZURE_OPENAI_API_VERSION=2024-02-01
AZURE_OPENAI_CHAT_DEPLOYMENT=<your-chat-deployment-name>
AZURE_OPENAI_EMBEDDING_DEPLOYMENT=text-embedding-3-small

# Azure AI Search
AZURE_SEARCH_ENDPOINT=https://<your-search-service>.search.windows.net
AZURE_SEARCH_API_KEY=<your-search-admin-key>
AZURE_SEARCH_INDEX_NAME=<your-index-name>

# Azure Video Indexer
AZURE_SUBSCRIPTION_ID=<your-subscription-id>
AZURE_RESOURCE_GROUP=<your-resource-group>
AZURE_VI_ACCOUNT_ID=<your-video-indexer-account-id>
AZURE_VI_LOCATION=<your-video-indexer-location>
AZURE_VI_NAME=<your-video-indexer-resource-name>

# Optional observability
APPLICATIONINSIGHTS_CONNECTION_STRING=<your-application-insights-connection-string>
LANGSMITH_TRACING=true
LANGSMITH_API_KEY=<your-langsmith-api-key>
LANGSMITH_PROJECT=<your-project-name>
```

## Index Compliance Documents

Compliance reference PDFs live in `backend/data/`. The current project includes:

- `1001a-influencer-guide-508_1.pdf`
- `youtube-ad-specs.pdf`

Index these documents into Azure AI Search before running audits:

```bash
uv run python backend/scripts/index_documents.py
```

The script loads PDFs, splits them into overlapping chunks, embeds them with Azure OpenAI, and uploads the chunks to the configured Azure AI Search index.

## Run the CLI Simulation

Run the local CLI workflow:

```bash
uv run python main.py
```

`main.py` currently uses a sample YouTube URL inside the script and prints the final compliance status, violations, and summary report.

## Run the FastAPI Server

Start the API locally:

```bash
uv run uvicorn backend.src.api.server:app --reload
```

Open the interactive API docs:

```text
http://localhost:8000/docs
```

Health check:

```bash
curl http://localhost:8000/health
```

Run an audit:

```bash
curl -X POST http://localhost:8000/audit \
  -H "Content-Type: application/json" \
  -d '{"video_url":"https://youtu.be/dT7S75eYhcQ"}'
```

Example response shape:

```json
{
  "session_id": "ce6c43bb-c71a-4f16-a377-8b493502fee2",
  "video_id": "vid_ce6c43bb",
  "status": "FAIL",
  "final_report": "Summary of findings...",
  "compliance_results": [
    {
      "category": "Claim Validation",
      "severity": "CRITICAL",
      "description": "Explanation of the violation..."
    }
  ]
}
```

## LangGraph Workflow

The compiled graph is created in `backend/src/graph/workflow.py`:

```text
START -> indexer -> auditor -> END
```

### Indexer Node

Implemented in `index_video_node`:

- validates that the input is a YouTube URL
- downloads the video with `yt-dlp`
- uploads the file to Azure Video Indexer
- waits for Azure processing to complete
- extracts transcript, OCR text, and metadata

### Auditor Node

Implemented in `audit_content_node`:

- combines transcript and OCR text into a retrieval query
- retrieves the top matching policy chunks from Azure AI Search
- prompts Azure OpenAI as a senior brand compliance auditor
- parses the model response into structured JSON
- returns `PASS` or `FAIL`, a final report, and issue details

## Notes

- The workflow requires live Azure services and valid credentials.
- The `azure_functions/` files and `backend/Dockerfile` are currently empty placeholders.
- Temporary video files are downloaded as `temp_audit_video.mp4` during processing and removed after upload.
- The current implementation supports YouTube URLs only.
- If Application Insights is not configured, telemetry is skipped without stopping the API.

## Troubleshooting

If document indexing fails:

- confirm all Azure OpenAI and Azure AI Search environment variables are set
- verify the embedding deployment name matches your Azure OpenAI deployment
- make sure the target Azure AI Search index exists or can be created by the SDK

If video processing fails:

- run `az login`
- confirm the Azure Video Indexer account ID, name, location, subscription ID, and resource group
- verify your Azure identity has permission to generate a Video Indexer account token
- check that the YouTube URL is accessible to `yt-dlp`

If the audit returns no transcript:

- check Azure Video Indexer processing status
- confirm the uploaded video was not quarantined or rejected
- review logs from `backend/src/services/video_indexer.py`
