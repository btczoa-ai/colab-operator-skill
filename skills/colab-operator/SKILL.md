---
name: colab-operator
description: Operate Google Colab environments via the `colab` CLI. Use when asked to create or manage GPU/TPU sessions, run Python/shell on a remote Colab VM, sync files, automate environment setup (packages, auth, Drive), or export session history.
---

# Skill: Colab Session Operator

Operate Google Colab environments via the `colab` CLI: provision GPU/TPU sessions, run Python/shell on the VM, sync files, and manage LLM server lifecycles.

---

## 1. Mental Model & Core Facts

- **Kernel State Persists**: A session is a live Jupyter kernel on a rented VM. Variables, imports, and functions persist across separate `colab exec` calls within the same session.
- **Working Directory**: Defaults to `/content`. Prefer absolute paths (`/content/...`).
- **Safety**: Always run `colab stop -s <name>` when done to avoid burning compute units.
- **Headless Non-Interactive**: Never run interactive commands (`colab repl`, `colab console`, `colab auth`, `colab drivemount`) from agents as they hang waiting for a TTY.

---

## 2. Authentication (ADC Setup)

The default auth mode is `adc`. Re-mint credentials with all 4 required scopes for headless agent use:

```bash
gcloud auth application-default login \
  --scopes=openid,\
https://www.googleapis.com/auth/cloud-platform,\
https://www.googleapis.com/auth/userinfo.email,\
https://www.googleapis.com/auth/colaboratory
```

- **Verify Auth**: Run `colab whoami` (displays email, scopes, expiry) or `colab sessions`.
- *Note*: `colab auth` is for VM-side GCP credentials, not CLI auth.

---

## 3. CLI Command Reference

| Category | Command | Description |
| :--- | :--- | :--- |
| **Provision** | `colab new -s <name> [--gpu T4\|L4\|A100] [--tpu v6e1]` | Create session (**always name with `-s`**). Fall back to CPU/T4 on quota error. |
| **Execute** | `colab exec -s <name> -f <script.py\|nb.ipynb>` | Run local script or notebook remotely without manual upload. |
| **Ephemeral Run** | `colab run [--gpu T4] [-s NAME] script.py [args...]` | One-shot job: provision + exec + auto-teardown. |
| **Manage** | `colab status [-s <name>]` / `colab log -s <name>` | Check VM state, busy/idle status, and structured event logs. |
| **Reset / Stop** | `colab restart-kernel -s <name>` / `colab stop -s <name>` | Reset wedged kernel or terminate VM. |

---

## 4. Autonomous Self-Improvement Deployment Loop (LLM Server)

When deploying a notebook and starting an LLM backend on Colab, execute this loop autonomously until verified:

```
[Deploy: exec nb.ipynb] -> [Monitor: status & logs] ──(READY)──> [Verify: http.rest] ──(PASS)──> [Report to User]
                                   │                                     │
                                (ERROR)                               (FAIL)
                                   ▼                                     ▼
                      [Diagnose logs] ──> [Patch locally & re-exec] ─────┘
```

1. **Deploy & Launch**: Upload `.env` and launch via `colab exec -s <name> -f nb.ipynb`.
2. **Periodic Log Monitoring**: Monitor `colab status -s <name>`, `colab log -s <name>`, and `/content/logs/`.
3. **Diagnose on Error**:
   - Port conflict (`Address already in use`) → Ensure `fuser -k -n tcp {PORT}` / `pkill -9` runs before server startup.
   - Missing CUDA libraries (`libcudart.so.12`) → Verify `LD_LIBRARY_PATH` and wheel compatibility.
   - Missing `.env` / Unset variables → Verify `.env` upload and required keys.
4. **Re-deploy & Re-run**: Patch locally and re-execute until healthy.
5. **API Verification & Report**: Test all endpoints via `http.rest` (`/healthz`, `/api/v1/status`, `/v1/models`, `/v1/chat/completions`). Report completion only after all tests succeed.
