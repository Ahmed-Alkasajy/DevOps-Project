# Book Shop — DevOps Project

## Students
- Ahmad Yaseer — 20210401
- Loay Saleh — 20220373

## Group Size
Group of 2

---

## Phase 1 — Docker Setup

### Requirements
- Docker
- Docker Compose

### Run locally
```bash
cd book-shop
cp .env.example .env
# Fill in your values in .env
docker compose up --build
```

Visit: http://localhost

### Services
| Service | Description |
|---------|-------------|
| `db` | PostgreSQL 15 — stores all Book Shop data with a persistent named volume |
| `backend` | Django app — serves all views and HTML templates on port 8000 |
| `nginx` | Reverse proxy on port 80 — forwards requests to backend:8000 and serves /static/ files |

---

## Phase 2 — CI/CD Pipelines

### Branch Strategy

| Branch | Philosophy | What ships |
|--------|-----------|------------|
| `dev` | Artifact-first | Image built from a saved artifact committed to the repo |
| `test` | Image-first | Fresh image built from source and pushed to Docker Hub |
| `prod` | Promotion only | Pulls existing image from Docker Hub — no build |

---

### Branch 1 — `dev` (Artifact-First)

**Trigger:** Every push to `dev`

**How it works:**
1. Installs dependencies and collects static files
2. Packages the source code, static files, and dependencies into `artifacts/app-<commit-sha>.tar.gz`
3. Commits the artifact back to the `dev` branch — every build keeps its own file (audit trail)
4. Builds the Docker image **from the committed artifact** — not from source directly
5. Pushes the image to Docker Hub tagged `dev-latest`
6. SSHs into EC2 and deploys using `docker compose -p dev up -d` on port **8000**

**Why artifact-first?** The artifact is the unit of truth. Building the image from the artifact guarantees the running container contains exactly what was packaged.

---

### Branch 2 — `test` (Image-First)

**Trigger:** Every push to `test` (typically a merge from `dev`)

**How it works:**
1. Rebuilds a **fresh artifact from source** — does NOT reuse the artifact committed by dev
2. Builds the Docker image from the freshly-built artifact
3. Pushes the image to **Docker Hub** tagged `test-latest`
4. SSHs into EC2 and deploys using `docker compose -p test up -d` on port **8001**

**Why rebuild?** Reproducibility. This proves the build process itself is reliable — not just that one specific artifact works.

---

### Branch 3 — `prod` (Promotion Only)

**Trigger:** Push or merge to `prod`

**How it works:**
1. Reads the version from the GitHub Actions repository variable `IMAGE_VERSION`
2. Pulls that exact image from Docker Hub — **no building at all**
3. SSHs into EC2 and deploys using `docker compose -p prod up -d` on port **8002**
4. Requires manual approval via the `production` environment before deploying

**Hard rule:** The prod workflow contains zero `docker build` commands. It only pulls and deploys.

**Why a repo variable?** `vars.IMAGE_VERSION` is auditable and editable from the GitHub UI — creating a clear paper trail when promoting a new version to production.

---

### How Three Deployments Coexist on One EC2

All three environments run on the same EC2 instance without conflict by using:

| Separation mechanism | Details |
|---------------------|---------|
| **Compose project names** | `-p dev`, `-p test`, `-p prod` — isolates container names and networks |
| **Different host ports** | dev=8000, test=8001, prod=8002 |
| **Separate deployment folders** | `~/deployments/dev`, `~/deployments/test`, `~/deployments/prod` |
| **Separate `.env` files** | Each environment has its own credentials and image version |

---

### GitHub Secrets and Variables

**Secrets (sensitive):**
| Name | Purpose |
|------|---------|
| `EC2_SSH_KEY` | Private key for SSH access to EC2 |
| `DOCKERHUB_USERNAME` | Docker Hub username |
| `DOCKERHUB_TOKEN` | Docker Hub access token |
| `SECRET_KEY` | Django secret key |
| `POSTGRES_PASSWORD` | PostgreSQL password |
| `POSTGRES_USER` | PostgreSQL user |
| `POSTGRES_DB` | PostgreSQL database name |

**Variables (non-sensitive):**
| Name | Purpose |
|------|---------|
| `EC2_HOST` | Public IP of the EC2 instance |
| `IMAGE_NAME` | Docker image name |
| `REGISTRY_NAME` | Docker Hub username (registry prefix) |
| `IMAGE_VERSION` | Source of truth for the version deployed to production |

---

### Live Deployments

| Environment | URL |
|-------------|-----|
| dev | http://13.60.24.147:8000 |
| test | http://13.60.24.147:8001 |
| prod | http://13.60.24.147:8002 |

---

### Registry Choice (Group of 2)
As a group of 2, we push images to **Docker Hub** as required by the assignment.
