---
name: colab-operator
description: Operate Google Colab environments via the `colab` CLI: provision GPU/TPU sessions, execute notebooks/scripts, monitor logs, and run an autonomous fix-and-verify loop.
---

# Skill: Colab Session Operator

Operate Google Colab VMs via pre-built scripts in `scripts/`.

## Pre-Built Operations (`scripts/`)

| Operation | Invocation | Purpose |
|:---|:---|:---|
| `deploy_colab` | `bash scripts/deploy_colab.sh <session> <nb.ipynb\|script.py> [T4\|L4\|A100]` | Provisions named session (if missing), executes notebook/script remotely, and prints session status. |
| `check_colab_run` | `bash scripts/check_colab_run.sh <session>` | Fetches `colab status` and `colab log` for the active session. |
| `stop_colab` | `bash scripts/stop_colab.sh <session>` | Terminates the Colab VM session (`colab stop`) to stop compute unit billing. |

## Execution Constraints
- **Persistent Kernel**: Variables and state under `/content` persist across `colab exec` calls within the same `-s <session>`.
- **Headless Only**: Never run interactive commands (`colab repl`, `colab console`, `colab auth`, `colab drivemount`).
- **ADC Auth**: If auth expires, re-mint via `gcloud auth application-default login --scopes=openid,https://www.googleapis.com/auth/cloud-platform,https://www.googleapis.com/auth/userinfo.email,https://www.googleapis.com/auth/colaboratory` and verify with `colab whoami`.

## Execution Pipeline
1. `deploy_colab(<session>, <nb.ipynb>, <gpu>)` — ensure external `.env` is uploaded, then launch notebook.
2. `check_colab_run(<session>)` — monitor VM state and `/content/logs/`.
3. **On Error**: diagnose port conflicts (`fuser -k -n tcp`), missing CUDA libs (`LD_LIBRARY_PATH`), or missing `.env` keys; patch locally and re-run `deploy_colab(...)`.
4. **On Ready**: verify all endpoints in `http.rest` (`/healthz`, `/api/v1/status`, `/v1/models`, `/v1/chat/completions`). Always call `stop_colab(<session>)` when finished.

## Trigger & Exit Criteria
- **Trigger**: Provisioning, executing, or debugging a Colab GPU/TPU session via `colab` CLI.
- **Exit Criteria**: Notebook executes cleanly, `http.rest` checks pass, and `stop_colab(<session>)` terminates the VM.

## Anti-Patterns
- Never invoke interactive CLI commands (`colab repl`, `colab console`, `colab auth`, `colab drivemount`).
- Never leave a rented GPU/TPU session running after verification completes.
