# Session 17: DevSecOps — Security in the CI/CD Pipeline

Hands-on verification of the DevSecOps pipeline in [`demo/.github/workflows/devsecops.yml`](../demo/.github/workflows/devsecops.yml), run **locally** stage by stage: unit tests → SAST → SCA → secret scanning → Docker build → image scan → run. Then the hand-off to Docker Hub, Kubernetes and GitHub Actions.

**Environment:** Windows 11, PowerShell, Python 3.14, Docker 29.8, Docker Desktop Kubernetes. The scanners run from their official images (`zricethezav/gitleaks`, `aquasec/trivy`).

---

## Table of Contents

- [Pipeline Overview](#pipeline-overview)
- [Part 1 — Unit Tests & Coverage](#part-1--unit-tests--coverage)
- [Part 2 — SAST (Bandit)](#part-2--sast-bandit)
- [Part 3 — SCA (pip-audit)](#part-3--sca-pip-audit)
- [Part 4 — Secret Scanning (Gitleaks)](#part-4--secret-scanning-gitleaks)
- [Part 5 — Docker Build](#part-5--docker-build)
- [Part 6 — Container Image Scanning (Trivy)](#part-6--container-image-scanning-trivy)
- [Part 7 — Run the Container](#part-7--run-the-container)
- [Part 8 — Registry, Kubernetes & GitHub Actions (to complete)](#part-8--registry-kubernetes--github-actions-to-complete)
- [Findings Summary](#findings-summary)

---

## Pipeline Overview

```text
 push / PR → main
      │
      ├── Unit Tests (pytest + coverage)
      ├── SAST (CodeQL in CI · Bandit locally)
      └── SCA  (pip-audit)
              │  all must pass
              ▼
        Docker Build ──► Image Scan (Trivy HIGH/CRITICAL) ──► Push to Docker Hub ──► Deploy to Kubernetes
```

---

## Part 1 — Unit Tests & Coverage

```bash
cd session-17-devsecops\demo
python -m pytest --cov=app --cov-report=term-missing
```

**Result:** `8 passed`; coverage **69%** of `app/app.py` (the pipeline-simulator endpoint, lines 179–209, has no tests).

![Part 1 — Unit Tests](./SS/01-unit-tests.png)

---

## Part 2 — SAST (Bandit)

CI uses **CodeQL**; locally the equivalent Python SAST tool is **Bandit**:

```bash
python -m bandit -r app/
```

**Result:** 7 issues, exit code 1, so this stage would **fail the gate**:

| Severity | Rule                                 | Location     | Meaning                                                                |
|:---------|:-------------------------------------|:-------------|:-----------------------------------------------------------------------|
| **High** | B201 `flask_debug_true`              | `app.py:234` | `debug=True` exposes the Werkzeug debugger → remote code execution     |
| Medium   | B104 `hardcoded_bind_all_interfaces` | `app.py:234` | Binds `0.0.0.0` (expected inside a container)                          |
| Low ×5   | B311 `random`                        | various      | `random` isn't cryptographically secure (only used for demo data here) |

**Fix for the High finding:** drive debug mode from the environment, e.g. `debug=os.getenv("FLASK_DEBUG") == "1"`.

![Part 2 — SAST Bandit](./SS/02-sast-bandit.png)

---

## Part 3 — SCA (pip-audit)

```bash
python -m pip_audit -r requirements.txt
```

**Result:** `No known vulnerabilities found` for `Flask==3.1.3` and its dependency tree.

![Part 3 — SCA pip-audit](./SS/03-sca-pip-audit.png)

---

## Part 4 — Secret Scanning (Gitleaks)

```bash
docker run --rm -v ${PWD}:/repo zricethezav/gitleaks:latest dir /repo -v
```

**Result:** ~111 KB scanned, **no leaks found**.

![Part 4 — Gitleaks](./SS/04-secret-scanning-gitleaks.png)

---

## Part 5 — Docker Build

```bash
docker build -t session17-python:local .
docker images session17-python
```

**Result:** Image built from `python:3.12-slim`.

![Part 5 — Docker Build](./SS/05-docker-build.png)

---

## Part 6 — Container Image Scanning (Trivy)

```bash
# Full HIGH/CRITICAL scan (summary table)
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache/ aquasec/trivy:latest image -q --severity HIGH,CRITICAL --table-mode summary session17-python:local

# Only vulnerabilities that already have a fix available
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock -v trivy-cache:/root/.cache/ aquasec/trivy:latest image -q --severity HIGH,CRITICAL --ignore-unfixed session17-python:local
```

**Result:**
- OS layer (Debian 13.7): **44 HIGH, 0 CRITICAL**, mostly `util-linux` CVEs with status `affected` (no upstream fix yet).
- Python packages (Flask, Werkzeug, Jinja2, …): **0**.
- With `--ignore-unfixed`: **0**, so there is nothing actionable today. A gate using `--ignore-unfixed --exit-code 1` would pass; rebuilding on a newer base image picks up fixes as they ship.

![Part 6 — Trivy](./SS/06-image-scan-trivy.png)

---

## Part 7 — Run the Container

```bash
docker run -d -p 5001:5001 --name python-web session17-python:local
docker ps --filter name=python-web
curl.exe -s http://localhost:5001/health
curl.exe -s http://localhost:5001/api/status
docker logs python-web
```

**Result:** Container `Up`, `/health` returns `"status": "healthy"`, `/api/status` returns app/runtime info.

![Part 7 — Docker Run](./SS/07-docker-run.png)

### Web Application in the Browser

<http://localhost:5001> serves the **DevSecOps Hub** dashboard (Status / Greet / Calc / Pipeline) with the health badge showing **Healthy**.

![Part 7 — Web App](./SS/08-web-app.png)

---

## Part 8 — Registry, Kubernetes & GitHub Actions (to complete)

These steps need personal accounts or a browser, so they're staged for completion:

1. ~~**Web UI screenshot**~~: done, see [Web Application in the Browser](#web-application-in-the-browser).
2. **Docker Hub push:**
   ```bash
   docker login
   docker tag session17-python:local <dockerhub-user>/python-web:latest
   docker push <dockerhub-user>/python-web:latest
   ```
3. **Kubernetes deploy:** replace `nensiravaliya28/hey-cicd:__IMAGE_TAG__` in `demo/k8s/deployment.yaml` with your pushed image, then:
   ```bash
   kubectl apply -f k8s/deployment.yaml -f k8s/service.yaml
   kubectl get pods,svc -l app=session17-python
   ```
   (NodePort `30001` → container port `5001`.)
4. **GitHub Actions run:** push `demo/` to a repo where `.github/workflows/devsecops.yml` sits at the **repo root**. Replace the hardcoded Docker Hub user `nensiravaliya28` (login `username:` and the `hey-cicd` image name) with your own, add the `DOCKERHUB_TOKEN` repo secret, and screenshot the run → `SS/09-workflow-run.png`. The deploy job runs on a throwaway `kind` cluster inside the runner.

---

## Findings Summary

| Stage      | Tool      | Result                                | Gate                       |
|:-----------|:----------|:--------------------------------------|:---------------------------|
| Unit tests | pytest    | 8/8 passed, 69% coverage              | ✅                         |
| SAST       | Bandit    | 1 High (Flask debug), 1 Medium, 5 Low | ❌ fix `debug=True`        |
| SCA        | pip-audit | 0 known vulnerabilities               | ✅                         |
| Secrets    | Gitleaks  | no leaks                              | ✅                         |
| Image scan | Trivy     | 44 HIGH (OS, unfixed) · 0 fixable     | ✅ with `--ignore-unfixed` |
