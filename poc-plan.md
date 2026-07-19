# PoC Plan: autoswarm

## Project Classification
- **Type:** api-service
- **Key Technologies:** Python 3.12+, FastAPI, Uvicorn, httpx
- **ODH Relevance:** Self-improving LLM proxy that can sit in front of OpenShift AI inference servers, adding intelligent skill injection and conversation logging. Demonstrates how middleware can enhance model-serving on the platform.

## PoC Objectives
1. Validate that AutoSwarm's online proxy can be containerized and deployed on OpenShift with UBI
2. Confirm the FastAPI server starts and the /v1/models and /v1/chat/completions endpoints respond
3. Verify the proxy works in standalone mode (without an upstream LLM, returning appropriate error responses)

## Infrastructure Requirements
- **Resource Profile:** small (256Mi RAM, 250m CPU)
- **GPU Required:** No
- **Persistent Storage:** None (conversations and skills can use ephemeral storage for PoC)
- **Sidecar Containers:** None
- **LLM API Required:** No (proxy starts without upstream; will return 502 errors for relay requests, but server itself is functional)

## Test Scenarios

### Scenario 1: server-startup
- **Description:** Verify the FastAPI server starts and responds to requests
- **Type:** http
- **Input:** GET /docs (FastAPI auto-generated OpenAPI docs)
- **Expected:** Returns 200 with HTML content
- **Timeout:** 30 seconds

### Scenario 2: models-endpoint
- **Description:** Verify /v1/models endpoint responds (will return 502 without upstream, which is expected)
- **Type:** http
- **Input:** GET /v1/models
- **Expected:** Returns HTTP response (200 with models or 502 with error JSON)
- **Timeout:** 30 seconds

### Scenario 3: chat-endpoint-structure
- **Description:** Verify /v1/chat/completions endpoint accepts POST requests with correct structure
- **Type:** http
- **Input:** POST /v1/chat/completions with {"model":"test","messages":[{"role":"user","content":"hello"}]}
- **Expected:** Returns HTTP response (502 without upstream, but validates the endpoint exists and accepts the request format)
- **Timeout:** 30 seconds

## Dockerfile Considerations
- Use registry.access.redhat.com/ubi9/python-312 as base image
- Install only the online proxy dependencies (not benchmark extras)
- Use pip install with the project's pyproject.toml
- CMD should run: autoswarm start --host 0.0.0.0 --port 8080 --upstream http://localhost:11434
- Port 8080 (non-privileged)

## Deployment Considerations
- **Deployment Model:** Deployment (long-running server)
- **Service:** ClusterIP on port 8080
- **Probes:** TCP socket on port 8080 (no health endpoint built-in)
- **Environment Variables:** UPSTREAM_URL for the LLM endpoint (optional, can default)
