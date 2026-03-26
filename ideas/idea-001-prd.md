# MODELUP — Product Requirements Document (MVP)

---

## 1. PRODUCT OVERVIEW

**Name:** modelup  
**Type:** CLI-first SaaS tool  
**One-liner:** One command to turn any HuggingFace model into a live API endpoint.  
**MVP Scope:** Internal tool, single user (you), HuggingFace models only, CLI only, runs on your existing EC2.

---

## 2. PROBLEM STATEMENT

Deploying an AI model from HuggingFace to a live usable endpoint requires:
- Writing a FastAPI wrapper manually
- Writing a Dockerfile
- Building and running the container
- Managing ports and routing
- Repeating this for every model

**modelup eliminates all of that with one command.**

---

## 3. MVP GOALS

- [ ] Deploy any HuggingFace model to a live `/predict` endpoint via one CLI command
- [ ] Auto-generate FastAPI wrapper based on model ID + task type
- [ ] Auto-build and run Docker container on EC2
- [ ] Track deployed models (ports, status, model IDs)
- [ ] Destroy deployments cleanly
- [ ] No dashboard. No UI. No auth. No billing. CLI only.

---

## 4. OUT OF SCOPE (MVP)

- Web dashboard
- Multi-user / auth / API keys
- Custom model file uploads
- GPU support
- Auto-scaling
- Billing
- ECR / ECS / Kubernetes
- Drift monitoring
- Logging UI

---

## 5. CLI INTERFACE

```bash
# Deploy a model
modelup deploy --model facebook/bart-large-cnn --task summarization

# List all running models
modelup list

# Get endpoint info for a specific model
modelup info --model facebook/bart-large-cnn

# Stop and remove a deployed model
modelup destroy --model facebook/bart-large-cnn

# Check backend server health
modelup status
```

**Supported --task values (MVP):**
- summarization
- text-generation
- text-classification
- translation
- question-answering
- zero-shot-classification
- image-classification

---

## 6. SYSTEM ARCHITECTURE

```
┌─────────────────────────────────────────────────────┐
│                   YOUR MACHINE / EC2                 │
│                                                      │
│  ┌──────────┐        ┌─────────────────────────┐    │
│  │          │  HTTP  │   modelup Backend        │    │
│  │  modelup │───────▶│   (FastAPI on port 9000) │    │
│  │  CLI     │        │                          │    │
│  │ (Typer)  │        │  - generator.py          │    │
│  └──────────┘        │  - docker_manager.py     │    │
│                      │  - registry.json         │    │
│                      └────────────┬────────────┘    │
│                                   │                  │
│                         Docker SDK (Python)          │
│                                   │                  │
│              ┌────────────────────▼──────────────┐  │
│              │         Docker Engine              │  │
│              │                                    │  │
│              │  ┌──────────┐  ┌──────────┐       │  │
│              │  │container │  │container │  ...  │  │
│              │  │port:8001 │  │port:8002 │       │  │
│              │  │bart-large│  │gpt2      │       │  │
│              │  └──────────┘  └──────────┘       │  │
│              └────────────────────────────────────┘  │
│                                                      │
│  Nginx (reverse proxy)                               │
│  /models/bart-large-cnn → localhost:8001             │
│  /models/gpt2           → localhost:8002             │
└─────────────────────────────────────────────────────┘

Endpoint exposed:
  http://<your-ec2-ip>/models/facebook-bart-large-cnn/predict
```

---

## 7. DETAILED WORKFLOW

### deploy flow

```
modelup deploy --model facebook/bart-large-cnn --task summarization

CLI
 └─▶ POST /deploy {model_id, task} → Backend

Backend: generator.py
 └─▶ fills Jinja2 template → generates model_app.py
      (FastAPI app that loads pipeline(task, model_id))

Backend: docker_manager.py
 └─▶ writes Dockerfile to temp dir
 └─▶ copies generated model_app.py into temp dir
 └─▶ docker build → image tagged as modelup-<model_slug>
 └─▶ docker run -d -p <free_port>:8000 modelup-<model_slug>
 └─▶ writes to registry.json {model_id, port, container_id, status}

Backend
 └─▶ updates Nginx config → /models/<model_slug>/predict → port

CLI
 └─▶ prints: Endpoint ready → http://<ec2-ip>/models/bart-large-cnn/predict
```

### destroy flow

```
modelup destroy --model facebook/bart-large-cnn

CLI
 └─▶ DELETE /deploy/{model_slug} → Backend

Backend
 └─▶ docker stop + docker rm container
 └─▶ docker rmi image
 └─▶ removes Nginx config block
 └─▶ removes entry from registry.json

CLI
 └─▶ prints: Destroyed bart-large-cnn
```

---

## 8. FILE STRUCTURE

```
modelup/
├── cli/
│   └── main.py                  # Typer CLI — deploy, list, destroy, info, status
│
├── server/
│   ├── main.py                  # FastAPI backend (runs on EC2, port 9000)
│   ├── generator.py             # generates model_app.py from Jinja2 template
│   ├── docker_manager.py        # builds, runs, stops Docker containers
│   ├── nginx_manager.py         # writes/removes Nginx proxy config blocks
│   ├── port_manager.py          # tracks and assigns free ports
│   └── registry.json            # local state: model_id, port, container_id, status
│
├── templates/
│   └── model_app.py.j2          # Jinja2 template for generated FastAPI app
│
├── Dockerfile.server            # Dockerfile for the modelup backend itself
├── requirements.txt             # CLI + server dependencies
└── README.md
```

---

## 9. GENERATED MODEL APP (what goes inside each container)

From `templates/model_app.py.j2`:

```python
from fastapi import FastAPI
from transformers import pipeline
from pydantic import BaseModel

app = FastAPI()
model = pipeline("{{ task }}", model="{{ model_id }}")

class Input(BaseModel):
    input: str

@app.post("/predict")
def predict(body: Input):
    result = model(body.input)
    return {"model": "{{ model_id }}", "result": result}

@app.get("/health")
def health():
    return {"status": "ok"}
```

---

## 10. TECH STACK

| Layer | Tool | Why |
|---|---|---|
| CLI | Python + Typer | clean commands, auto --help |
| Backend | FastAPI | lightweight, fast |
| Template engine | Jinja2 | dynamic model_app.py generation |
| Container management | Docker SDK for Python | programmatic docker build/run/stop |
| Model loading | HuggingFace transformers pipeline() | abstracts all model types |
| Reverse proxy | Nginx | route /models/<slug> to container port |
| State tracking | registry.json | simple, no DB needed for MVP |
| Infra | Your existing EC2 | no new infra cost |

---

## 11. REQUIREMENTS

### System Requirements (EC2)
- Docker installed and running
- Python 3.10+
- Nginx installed
- Port 9000 open for backend
- Ports 8001–8100 available for model containers

### Python Dependencies
```
typer
fastapi
uvicorn
jinja2
docker
requests
httpx
```

### Each model container needs
```
fastapi
uvicorn
transformers
torch
```

---

## 12. TO-DO TASK LIST

### Phase 0 — Setup
- [ ] Create project repo `modelup`
- [ ] Set up virtual environment
- [ ] Install dependencies
- [ ] Verify Docker SDK works on EC2
- [ ] Verify Nginx is running on EC2

### Phase 1 — Core Backend
- [ ] `server/main.py` — FastAPI backend with `/deploy`, `/destroy`, `/list`, `/status` routes
- [ ] `server/generator.py` — reads model_id + task, fills Jinja2 template, writes model_app.py
- [ ] `templates/model_app.py.j2` — base FastAPI app template
- [ ] `server/port_manager.py` — scans ports 8001–8100, returns next free port
- [ ] `server/registry.json` — initialized as empty `{}`
- [ ] `server/docker_manager.py` — build image, run container, stop container, remove image
- [ ] `server/nginx_manager.py` — write proxy block to Nginx config, reload Nginx

### Phase 2 — CLI
- [ ] `cli/main.py` — Typer app with commands: deploy, destroy, list, info, status
- [ ] `deploy` command — POST to backend, poll until ready, print endpoint
- [ ] `list` command — GET /list, pretty print table of running models
- [ ] `destroy` command — DELETE to backend, confirm removal
- [ ] `info` command — GET /info/<model>, print port + endpoint + status
- [ ] `status` command — GET /status, check if backend is alive

### Phase 3 — Dockerize Backend
- [ ] `Dockerfile.server` — containerize the FastAPI backend itself
- [ ] Run backend as Docker container on EC2
- [ ] Ensure backend container can access Docker socket (docker-in-docker mount)
- [ ] Nginx proxies port 9000 for backend too

### Phase 4 — Testing
- [ ] Deploy `facebook/bart-large-cnn` with `--task summarization` end to end
- [ ] Hit `/predict` endpoint with a real input, get real output
- [ ] Deploy a second model simultaneously, verify port isolation
- [ ] Destroy both, verify containers gone, Nginx blocks removed, registry clean
- [ ] Test `modelup list` shows correct state

### Phase 5 — Polish
- [ ] Add loading spinner to CLI during build (model download can be slow)
- [ ] Add error handling: model not found on HF, port exhausted, Docker build fail
- [ ] Add `--dry-run` flag to deploy (shows what would be generated without running)
- [ ] Write README with usage examples

---

## 13. KNOWN RISKS & MITIGATIONS

| Risk | Mitigation |
|---|---|
| Model download is slow (can be GBs) | Show progress in CLI, don't timeout |
| Port conflicts | port_manager.py scans before assigning |
| Docker build fails (bad model ID) | Validate model exists on HF before building |
| Nginx reload fails | Wrap in try/catch, log error, revert config |
| registry.json gets out of sync | On startup, reconcile registry against running containers |

---

## 14. SUCCESS CRITERIA FOR MVP

- `modelup deploy --model facebook/bart-large-cnn --task summarization` works end to end in under 10 minutes
- `/predict` endpoint returns real model output
- Two models can run simultaneously without conflict
- `modelup destroy` cleanly removes everything
- `modelup list` shows accurate state

---

*modelup MVP — internal tool phase*  
*Author: Ash Reddy*  
*Stack: Python · FastAPI · Typer · Docker · Nginx · HuggingFace Transformers*