# PoC Report: autoswarm

## 1. Executive Summary

AutoSwarm, a self-improving OpenAI-compatible LLM proxy built with FastAPI, was successfully containerized and deployed on OpenShift. The proxy server starts correctly, serves its FastAPI documentation, and correctly handles upstream connectivity errors. 2 of 3 test scenarios passed fully, with the third demonstrating the endpoint exists but returns 500 instead of 502 due to missing error handling in the chat completions route (an upstream code quality issue, not a deployment problem).

## 2. Project Analysis

- **Repository:** [arteemg/autoswarm](https://github.com/arteemg/autoswarm)
- **Fork:** [aicatalyst-team/autoswarm](https://github.com/aicatalyst-team/autoswarm)
- **Description:** AutoSwarm is a self-improving LLM proxy that logs conversations, extracts "skills" via reflection, and injects them into future prompts. It supports Ollama, vLLM, and LM Studio as upstream LLM providers.
- **Classification:** api-service

| Component | Language | Build System | ML Workload | Port |
|-----------|----------|-------------|-------------|------|
| proxy | Python | pip/hatchling | No | 8080 |

- **Key Technologies:** Python 3.12, FastAPI, Uvicorn, httpx, Click, PyYAML
- **License:** Not specified

## 3. PoC Objectives

1. Validate containerization of the FastAPI proxy on OpenShift with UBI
2. Confirm the server starts and API endpoints respond
3. Verify graceful behavior when no upstream LLM is available

## 4. Pipeline Execution

- **Intake:** Identified single Python FastAPI component with minimal dependencies
- **Evaluate:** Score 70/100. Relationship: adjacent to model-serving
- **Fork:** Forked to `aicatalyst-team/autoswarm`
- **PoC Plan:** API service with 3 test scenarios, small resource profile
- **Containerize:** Created `Dockerfile.ubi` using `ubi9/python-312`. Required 3 build attempts:
  1. Failed: README.md excluded by .dockerignore (hatchling needs it)
  2. Failed: chgrp permissions (needed USER 0 for build steps)
  3. Succeeded
- **Build:** Image `quay.io/aicatalyst/autoswarm-proxy:latest` built and pushed via OpenShift BuildConfig
- **Deploy:** Deployment + Service manifests in `kubernetes/`
- **Apply:** Deployed to namespace `poc-autoswarm`. Required quay-pull-secret.
- **Test:** 2/3 scenarios passed, 1 partial pass (endpoint exists but returns 500 instead of 502)

## 5. Test Results

| Scenario | Status | Duration | Details |
|----------|--------|----------|---------|
| server-startup | PASS | 0.1s | GET /docs returns 200 with FastAPI OpenAPI documentation |
| models-endpoint | PASS | 0.1s | GET /v1/models returns 502 with JSON error (expected without upstream) |
| chat-endpoint | PARTIAL | 0.1s | POST /v1/chat/completions returns 500 (endpoint exists, but lacks try/except for upstream ConnectError) |

**Overall: 2/3 pass, 1/3 partial**

## 6. Infrastructure Deployed

- **Namespace:** `poc-autoswarm`
- **Container Image:** `quay.io/aicatalyst/autoswarm-proxy:latest`
- **Resources:**
  - `deployment/proxy` (1 replica, 256Mi/250m requests, 512Mi/500m limits)
  - `service/proxy` (ClusterIP on port 8080)
- **Service URL:** `http://proxy.poc-autoswarm.svc:8080`

## 7. Recommendations

- Fix the chat completions endpoint error handling to return 502 instead of 500 when upstream is unreachable
- Add a /health endpoint for proper Kubernetes HTTP probe support
- Deploy alongside an Ollama or vLLM instance to demonstrate full relay functionality
- The self-improving skill injection is the most interesting feature and should be demonstrated with a live LLM

## 8. Open Data Hub / OpenShift AI Considerations

- AutoSwarm could serve as a lightweight proxy layer in front of KServe or vLLM inference servers on OpenShift AI
- The skill injection mechanism could be integrated with Data Science Pipelines for prompt optimization workflows
- The conversation logging could feed into MLflow for prompt engineering analysis

## 9. Appendix

### Artifacts
- **Dockerfile:** [Dockerfile.ubi](https://github.com/aicatalyst-team/autoswarm/blob/main/Dockerfile.ubi)
- **Kubernetes Manifests:** [kubernetes/](https://github.com/aicatalyst-team/autoswarm/tree/main/kubernetes)
- **PoC Plan:** [poc-plan.md](https://github.com/aicatalyst-team/autoswarm/blob/main/poc-plan.md)

### Build Issues
1. Build #1: README.md excluded by .dockerignore, hatchling metadata generation failed
2. Build #2: chgrp permissions failed (UBI default user can't change group on COPY'd files)
3. Build #3: Success after adding USER 0 for build steps
