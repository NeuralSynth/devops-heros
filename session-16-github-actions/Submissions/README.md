# Session 16: CI/CD with GitHub Actions

Verification of CI/CD concepts and the GitHub Actions workflows in this session: the local CI/CD simulations and build/test stages (screenshots below), plus the GitHub-hosted workflow runs (staged for completion on GitHub).

**Environment:** Windows 11, PowerShell, Git Bash (`bash`), Python 3.14, pytest 9, flake8.

---

## Table of Contents

- [Part 1 — CI vs CD Simulation](#part-1--ci-vs-cd-simulation)
- [Part 2 — Pipeline Stages](#part-2--pipeline-stages)
- [Part 3 — Build & Test Pipeline (lint + pytest)](#part-3--build--test-pipeline-lint--pytest)
- [Part 4 — Demo App: Tests, Coverage & Build Artifact](#part-4--demo-app-tests-coverage--build-artifact)
- [Part 5 — GitHub Actions Workflow Runs (to complete)](#part-5--github-actions-workflow-runs-to-complete)

---

## Part 1 — CI vs CD Simulation

```bash
cd "session-16-github-actions\01-ci-vs-cd 10-33-34-211"
bash ci_simulation.sh
bash cd_simulation.sh
```

**Result:** **CI** covers checkout → runtime setup → lint → unit tests → coverage gate (92.4% ≥ 80%), producing a verified artifact. **CD** covers image build/tag → vulnerability scan → staging deploy → health check (HTTP 200 on `/health`).

![Part 1 — CI vs CD](./SS/01-ci-vs-cd-simulation.png)

---

## Part 2 — Pipeline Stages

```bash
cd "session-16-github-actions\02-pipeline-concepts 10-33-34-222"
bash pipeline_stages.sh
```

**Result:** Source → Build & Dependencies → Quality Gates → Artifacts → Deploy, each stage reporting `[PASS]`.

![Part 2 — Pipeline Stages](./SS/02-pipeline-stages.png)

---

## Part 3 — Build & Test Pipeline (lint + pytest)

These are the same commands the `lint` and `test` jobs of [`ci.yml`](../09-build-test-pipeline%2010-33-34-262/ci.yml) run:

```bash
cd "session-16-github-actions\09-build-test-pipeline 10-33-34-262"
python -m flake8 app.py tests/
python -m pytest tests/ -v
```

**Result:** flake8 reports no violations; `4 passed` (`test_add`, `test_subtract`, `test_divide`, `test_divide_by_zero`).

![Part 3 — Lint & Tests](./SS/03-build-test-pipeline-pytest.png)

---

## Part 4 — Demo App: Tests, Coverage & Build Artifact

```bash
cd session-16-github-actions\session-16-github-actions\demo
python -m pytest tests/ -v --cov=app --cov-report=term-missing
bash build.sh
Get-Content build/build-info.txt
```

**Result:**
- `5 passed` for the calculator (`add`, `subtract`, `multiply`, `divide`, `divide_by_zero`).
- Coverage **30%** for `app/calculator.py`: lines 22–54 are untested, a good candidate for more tests.
- `build.sh` produces the `build/` artifact (`calculator.py` + `build-info.txt`, *Build Status: SUCCESS*). This is what `actions/upload-artifact` uploads in CI.

> The `.sh` scripts had been checked out with Windows CRLF line endings, which breaks `bash` (`set: -\r: invalid option`). They were normalized to LF; the git index already stores LF, so there's no content diff.

![Part 4 — Demo Tests & Build](./SS/04-demo-calculator-pytest-build.png)

---

## Part 5 — GitHub Actions Workflow Runs (to complete)

GitHub only runs workflows from `.github/workflows/` at the **root** of a repository. To produce the run screenshots, push each demo as (or into) a repo whose root contains its `.github/workflows/` folder, then capture the **Actions** tab:

| # | Workflow                                             | Source                                                  | Needs                                                                                                      | Screenshot                    |
|:--|:-----------------------------------------------------|:--------------------------------------------------------|:-----------------------------------------------------------------------------------------------------------|:------------------------------|
| 1 | Hello GitHub Actions                                 | `03-github-actions-intro …/.github/workflows/hello.yml` | —                                                                                                          | `SS/05-hello-actions.png`     |
| 2 | Triggers (push / PR / dispatch / cron / path filter) | `04-workflows …/`                                       | —                                                                                                          | `SS/06-workflow-triggers.png` |
| 3 | Jobs & steps (parallel / sequential / conditional)   | `05-jobs-steps …/.github/workflows/jobs-steps.yml`      | —                                                                                                          | `SS/07-jobs-steps.png`        |
| 4 | Runners & matrix                                     | `06-runners …/.github/workflows/matrix-demo.yml`        | —                                                                                                          | `SS/08-matrix.png`            |
| 5 | Secrets                                              | `07-secrets …/.github/workflows/secrets-demo.yml`       | repo secret `DEMO_SECRET` (and `DOCKER_USERNAME`/`DOCKER_PASSWORD`, `PROD_API_KEY` for the other examples) | `SS/09-secrets.png`           |
| 6 | Artifacts                                            | `08-artifacts …/.github/workflows/artifacts-demo.yml`   | —                                                                                                          | `SS/10-artifacts.png`         |
| 7 | CI pipeline (lint → test → build)                    | `09-build-test-pipeline …/.github/workflows/ci.yml`     | —                                                                                                          | `SS/11-ci-pipeline.png`       |

Useful CLI checks once pushed:

```bash
gh workflow list
gh workflow run ci.yml
gh run list --limit 10
gh run view <RUN_ID> --log
```
