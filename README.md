# Dockerized LiteLLM Proxy + Langfuse Observability Stack

Self-hosted LLM gateway and observability platform powered by [LiteLLM Proxy](https://docs.litellm.ai/) and [Langfuse](https://langfuse.com/).

This stack provides a unified, OpenAI-compatible API gateway across multiple LLM providers (OpenAI, Anthropic, and more), combined with deep observability, prompt tracking, latency analysis, token usage metrics, and cost monitoring out of the box.

---

## Architecture Overview

```mermaid
flowchart TD
    Client["Client Applications\n(OpenAI SDK, LangChain, cURL)"]
    
    subgraph Gateway ["LiteLLM Gateway (:4000)"]
        LiteLLM["LiteLLM Proxy\n(/v1/chat/completions)"]
        LiteLLMUI["LiteLLM Admin Dashboard\n(/ui)"]
    end

    subgraph Providers ["Upstream LLM Providers"]
        OpenAI["OpenAI (e.g. gpt-4o)"]
        Anthropic["Anthropic (e.g. Claude 3.5 Sonnet)"]
        Other["Other Providers (Azure, Bedrock, Groq, Ollama)"]
    end

    subgraph Observability ["Langfuse Platform (:3000)"]
        LangfuseIngest["Langfuse Ingestion API"]
        LangfuseUI["Langfuse Web UI\n(Traces, Costs, Sessions)"]
    end

    subgraph Storage ["Persistence (:5432)"]
        Postgres[("PostgreSQL 16")]
    end

    Client -->|"HTTP /v1/chat/completions (Bearer Key)"| LiteLLM
    LiteLLM -->|"Routes requests"| OpenAI
    LiteLLM -->|"Routes requests"| Anthropic
    LiteLLM -->|"Routes requests"| Other
    LiteLLM -.->|"Async telemetry & traces"| LangfuseIngest
    LangfuseIngest --> Postgres
    LiteLLM --> Postgres
    LangfuseUI --> Postgres
```

### How It Works
1. **Gateway**: Client applications make standard OpenAI-formatted API calls to the LiteLLM Proxy at `http://localhost:4000`.
2. **Routing**: LiteLLM handles routing, model aliases, load balancing, retries, and API key management for upstream providers (OpenAI, Anthropic, etc.).
3. **Observability**: Every request, prompt, completion, token count, cost, and latency metric is automatically captured and streamed to Langfuse via LiteLLM's native callback system.
4. **Persistence**: Langfuse and LiteLLM state (including dynamic models, users, and audit records) are stored in PostgreSQL.
5. **Zero-Touch Bootstrap**: Environment variables pre-configure Langfuse organizations, project keys, and admin credentials on initial boot without requiring manual UI configuration.

---

## Services & Ports Reference

| Service | Port | Endpoint / URL | Default Credentials | Description |
| :--- | :--- | :--- | :--- | :--- |
| **LiteLLM Proxy** | `4000` | [http://localhost:4000](http://localhost:4000) | Master Key: `sk-litellm-master-key-xyz` | OpenAI-compatible proxy gateway |
| **LiteLLM Admin UI** | `4000` | [http://localhost:4000/ui](http://localhost:4000/ui) | Master Key: `sk-litellm-master-key-xyz` | Gateway dashboard & virtual key manager |
| **Langfuse Web UI** | `3000` | [http://localhost:3000](http://localhost:3000) | `admin@localhost.com` / `adminpassword123` | LLM observability & tracing dashboard |
| **PostgreSQL** | `5432` | `localhost:5432` | `langfuse` / `langfuse_secret` | Shared database for LiteLLM & Langfuse |

---

## Project Structure

```text
.
├── docker-compose.yml       # Orchestrates PostgreSQL, Langfuse, and LiteLLM services
├── litellm_config.yaml      # LiteLLM routing rules, model definitions, and callback settings
├── .env.example             # Template for required environment variables & secrets
├── .env                     # Local configuration (API keys, secrets, DB credentials)
├── .gitignore               # Excludes secrets and temporary files from version control
└── README.md                # Project documentation
```

---

## Quickstart

### 1. Prerequisites
- [Docker](https://docs.docker.com/get-docker/) (v24.0 or newer)
- [Docker Compose](https://docs.docker.com/compose/) (v2.20 or newer)
- API key(s) from OpenAI, Anthropic, or any provider you wish to use.

### 2. Configure Environment Variables
Copy `.env.example` to `.env` if you have not already:

```bash
cp .env.example .env
```

Open `.env` and fill in your upstream API keys:

```dotenv
# Upstream Model Provider Keys
OPENAI_API_KEY=sk-proj-your-actual-openai-key
ANTHROPIC_API_KEY=sk-ant-your-actual-anthropic-key

# LiteLLM Root Master Key (Change for production)
LITELLM_MASTER_KEY=sk-litellm-master-key-xyz

# Langfuse Admin Credentials
LANGFUSE_ADMIN_EMAIL=admin@localhost.com
LANGFUSE_ADMIN_PASSWORD=adminpassword123
```

> [!TIP]
> For production environments, generate cryptographically secure values for `NEXTAUTH_SECRET`, `SALT`, and `LITELLM_MASTER_KEY`:
> ```bash
> openssl rand -base64 32
> ```

### 3. Start the Stack
Launch the containers in detached mode:

```bash
docker compose up -d
```

### 4. Verify Service Health
Check the container status:

```bash
docker compose ps
```

All three services (`postgres`, `langfuse`, and `litellm`) should display a status of `Up` (and `healthy` for PostgreSQL).

---

## Testing & Usage

### 1. Check Model Catalog
Query the proxy's model registry to see configured models:

```bash
curl -s http://localhost:4000/v1/models \
  -H "Authorization: Bearer sk-litellm-master-key-xyz"
```

### 2. Chat Completion with cURL (OpenAI gpt-4o)

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-litellm-master-key-xyz" \
  -d '{
    "model": "gpt-4o",
    "messages": [
      {
        "role": "user",
        "content": "Hello! Explain quantum computing in one sentence."
      }
    ],
    "user": "developer-local-test"
  }'
```

### 3. Chat Completion with cURL (Anthropic Claude 3.5 Sonnet)

Using the exact same OpenAI protocol endpoint, query Claude:

```bash
curl http://localhost:4000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer sk-litellm-master-key-xyz" \
  -d '{
    "model": "claude-3-5-sonnet",
    "messages": [
      {
        "role": "user",
        "content": "Provide a 3-step checklist for code reviews."
      }
    ],
    "user": "developer-local-test"
  }'
```

### 4. Python Integration (Official OpenAI SDK)

You can use the standard `openai` Python SDK by pointing `base_url` to LiteLLM:

```python
import os
from openai import OpenAI

client = OpenAI(
    base_url="http://localhost:4000",
    api_key=os.environ.get("LITELLM_MASTER_KEY", "sk-litellm-master-key-xyz"),
)

# Call OpenAI GPT-4o
response_openai = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "What is the capital of Bulgaria?"}],
    user="user-123",
)
print("GPT-4o Response:", response_openai.choices[0].message.content)

# Call Claude 3.5 Sonnet through the same client
response_claude = client.chat.completions.create(
    model="claude-3-5-sonnet",
    messages=[{"role": "user", "content": "Name three advantages of containerization."}],
    user="user-123",
)
print("Claude 3.5 Sonnet Response:", response_claude.choices[0].message.content)
```

### 5. Inspecting Traces in Langfuse
1. Open your browser and navigate to [http://localhost:3000](http://localhost:3000).
2. Log in using the admin credentials specified in `.env` (default: `admin@localhost.com` / `adminpassword123`).
3. Select the **Production** project.
4. Navigate to **Tracing > Traces**:
   - Inspect individual generation requests.
   - View full request/response payloads, latency breakdown, exact prompt/completion token counts, and dollar costs.
   - Filter traces by user ID or tags passed during the request.

---

## Configuration

### Adding Models
Edit [litellm_config.yaml](file:///Users/yovko/Developer/novarto/litellm-langfuse/litellm_config.yaml) to add new models or change provider settings:

```yaml
model_list:
  - model_name: gpt-4o
    litellm_params:
      model: openai/gpt-4o
      api_key: os.environ/OPENAI_API_KEY

  - model_name: claude-3-5-sonnet
    litellm_params:
      model: anthropic/claude-3-5-sonnet-20241022
      api_key: os.environ/ANTHROPIC_API_KEY

  # Example: Add Gemini 1.5 Pro
  # - model_name: gemini-1.5-pro
  #   litellm_params:
  #     model: gemini/gemini-1.5-pro
  #     api_key: os.environ/GEMINI_API_KEY

  # Example: Add local Ollama model
  # - model_name: llama3
  #   litellm_params:
  #     model: ollama/llama3
  #     api_base: http://host.docker.internal:11434

litellm_settings:
  callbacks: ["langfuse"]
```

After modifying [litellm_config.yaml](file:///Users/yovko/Developer/novarto/litellm-langfuse/litellm_config.yaml), reload LiteLLM:

```bash
docker compose restart litellm
```

### Passing Metadata and User Tracking
LiteLLM automatically propagates request parameters to Langfuse. Include any of the following fields in your requests:

```json
{
  "model": "gpt-4o",
  "messages": [{"role": "user", "content": "Hello"}],
  "user": "user-uuid-123",
  "metadata": {
    "session_id": "session-abc-456",
    "environment": "staging",
    "tags": ["customer-support", "v2-agent"]
  }
}
```

These fields will be indexed directly in Langfuse under the trace attributes.

---

## Maintenance & Operations

### Viewing Logs
To stream logs from all services or a specific container:

```bash
# All logs
docker compose logs -f

# LiteLLM logs only
docker compose logs -f litellm

# Langfuse logs only
docker compose logs -f langfuse
```

### Stopping the Services
To stop running containers:

```bash
docker compose down
```

### Resetting Persistent Data
If you need to start fresh (erasing database state, users, and traces):

```bash
docker compose down -v
```

> [!CAUTION]
> `docker compose down -v` deletes the `postgres_data` volume permanently.

---

## Troubleshooting

### 1. 401 Unauthorized Error on LiteLLM
- **Symptom**: `{"error":{"message":"Authentication Error, No api key passed in."}}`
- **Fix**: Ensure your request header contains `Authorization: Bearer <LITELLM_MASTER_KEY>` where the key matches `LITELLM_MASTER_KEY` in `.env`.

### 2. 401 Unauthorized from Upstream Model Provider
- **Symptom**: `AuthenticationError: OpenAIException - Incorrect API key provided: sk-proj-xxxx`
- **Fix**: Make sure valid provider keys are set in `.env` (`OPENAI_API_KEY`, `ANTHROPIC_API_KEY`) and restart the proxy (`docker compose restart litellm`).

### 3. Langfuse UI Sign-In Issues
- **Symptom**: Unable to log in with initial credentials.
- **Fix**: Langfuse only auto-provisions the admin user on the first database initialization. If you changed credentials after the volume was already created, either reset the database volume with `docker compose down -v` or use the "Sign Up" option in the Langfuse UI if enabled.

### 4. Database Connection Errors on Startup
- **Symptom**: Containers restarting while attempting to connect to PostgreSQL.
- **Fix**: PostgreSQL includes a healthcheck (`pg_isready`). `langfuse` and `litellm` depend on `postgres` reaching a healthy state before starting. Wait a few seconds for the database migrations to complete.

---

## Security Best Practices
- **Do not commit `.env`**: Always keep actual secrets out of git. Keep `.env` listed in `.gitignore`.
- **Change Master Keys**: Never use default credentials (`sk-litellm-master-key-xyz`, `adminpassword123`) in production or public-facing deployments.
- **Network Isolation**: By default, PostgreSQL exposes port `5432` to `localhost`. In production, consider removing `ports: - "5432:5432"` from [docker-compose.yml](file:///Users/yovko/Developer/novarto/litellm-langfuse/docker-compose.yml) so only the internal Docker network can communicate with the database.
