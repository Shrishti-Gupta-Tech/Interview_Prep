# 10. Git & DevOps — Deep Dive

> **Goal:** Master Git day-to-day + advanced flows, and understand the whole modern deployment stack — CI/CD, Docker, Kubernetes.

---

## 10.0 What is Version Control?

Imagine collaborating on a document. Without version control:
- Files named `report_v1.docx`, `report_v2.docx`, `report_final.docx`, `report_final_FINAL.docx`
- Two people editing → whose changes win?
- No history of "why did we make this change?"

**Git** solves this. It's a **distributed** version control system (every developer has the full history locally).

### Why Git?
- Full history (blame, log)
- Branch and merge easily
- Distributed (no central server needed to commit)
- Fast (mostly local operations)
- Industry standard

---

## 10.1 Git Fundamentals

### Setup

```bash
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
git config --global core.editor "code --wait"
git config --list
```

### The three areas

```
    Working Directory  ─── git add ──►  Staging Area (Index)  ─── git commit ──►  Local Repo
       (your files)                        (what will be committed)                (.git folder)
                                                                                        │
                                                                                git push│
                                                                                        ▼
                                                                                 Remote Repo
```

- **Working directory** — files you edit
- **Staging area** — a "photo booth" — what will go into the next commit
- **Local repo** — committed history on your machine
- **Remote repo** — GitHub / GitLab / Bitbucket

### Basic workflow

```bash
git clone https://github.com/user/repo.git         # copy remote

# Make changes...
git status                                          # what changed?
git diff                                            # see unstaged changes
git diff --staged                                   # see staged changes

git add file.txt                                    # stage one file
git add .                                           # stage everything
git add -p                                          # stage hunks interactively (great for cleanups)

git commit -m "Fix null pointer in checkout"
git commit -am "…"                                  # -a = auto-stage tracked files

git log                                             # history
git log --oneline --graph --all                     # pretty
git show <commit>                                   # details of a commit
git blame file.txt                                  # who wrote each line

git push origin main                                # send to remote
git pull origin main                                # fetch + merge
git fetch                                           # download without merge
```

### Commit message conventions (Conventional Commits)

```
<type>(<scope>): <short summary>

<longer description>

<footer: closes #123>
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`.

Example:
```
feat(cart): add coupon code support

Users can now enter a coupon at checkout. Applied via CartService.applyCoupon().

Closes #456
```

Good messages make history readable and enable auto-changelogs.

---

## 10.2 Branching — The Superpower

A branch is a lightweight pointer to a commit.

```bash
git branch                                          # list local
git branch -a                                       # include remote
git branch feature/login                            # create
git checkout feature/login                          # switch
git checkout -b feature/login                       # create + switch
git switch feature/login                            # modern equivalent
git switch -c feature/login

git branch -d feature/login                         # safe delete
git branch -D feature/login                         # force delete
git push origin feature/login                       # push a new branch
git push origin --delete feature/login              # delete remote
```

### What a merge looks like

Before:
```
main:     A---B---C
                  \
feature:           D---E
```

After `git merge feature`:
```
main:     A---B---C-------M          M is a merge commit
                  \       /
feature:           D---E---
```

---

## 10.3 Merge vs Rebase — The Key Distinction

### Merge — non-destructive, preserves history

```bash
git checkout main
git merge feature/x
```

Creates a merge commit. History shows exactly what happened.

Pros: safe, clear history of parallel work.  
Cons: history can be cluttered with merge commits.

### Rebase — rewrites history

Takes your commits and replays them on top of another branch.

Before:
```
main:     A---B---C
                  \
feature:           D---E
```

After `git rebase main`:
```
main:     A---B---C
                  \
feature:           D'---E'         (D' and E' are NEW commits with new hashes)
```

Then `git checkout main; git merge feature` gives a fast-forward — linear history.

Pros: clean, linear history.  
Cons: **rewrites history** — never rebase branches that have been pushed and others are using!

### The golden rule

> **Never rebase public/shared branches.** Rebasing is fine for your own private feature branches before merging.

### Interactive rebase — squash, edit, reorder commits

```bash
git rebase -i HEAD~3
```
Opens an editor:
```
pick abc123 First commit
pick def456 Fix typo
pick ghi789 Add feature

# Change 'pick' to 'squash' or 's' to combine into previous
# Change 'pick' to 'reword' to change message
# Reorder lines to reorder commits
# Delete lines to drop commits
```

Powerful for cleaning up before opening a PR.

### Fast-forward vs no-fast-forward

```bash
git merge feature/x                                 # fast-forward if possible
git merge --no-ff feature/x                         # always create a merge commit
```

Some teams use `--no-ff` for feature branches so history clearly shows branch points.

---

## 10.4 Undoing Things — The Safety Net

### Discard changes

```bash
git restore file.txt                                # discard unstaged (was git checkout)
git restore --staged file.txt                       # unstage (was git reset HEAD)
git clean -fd                                       # remove untracked files (careful!)
```

### Undo commits

```bash
git commit --amend                                  # tweak last commit (message or add forgotten file)

git reset --soft HEAD~1                             # undo last commit, keep changes staged
git reset HEAD~1                                     # undo last commit, keep changes in working dir
git reset --hard HEAD~1                             # undo last commit, DISCARD changes ⚠️

git revert <commit>                                 # create a new commit that undoes another
                                                     # (safe for shared history)
```

**Rules:**
- Use `revert` on shared branches (main).
- Use `reset` only on private branches.

### `reflog` — your safety net

`git reflog` records HEAD movements. If you `reset --hard` and lose commits, you can recover:

```bash
git reflog                                          # find the SHA you lost
git checkout <sha>                                  # or reset back
git reset --hard HEAD@{2}
```

**Nothing is ever really lost in Git** for at least 30 days.

---

## 10.5 Stashing — Temporary Storage

Save uncommitted changes so you can switch context.

```bash
git stash                                           # save
git stash push -m "wip: cart bug"                   # save with description
git stash list                                      # see all stashes
git stash show -p stash@{0}                         # inspect
git stash pop                                       # apply and remove
git stash apply                                     # apply, keep in stash
git stash drop                                      # discard
```

Real scenario: you're editing feature branch A, urgent hotfix needed. `git stash`, switch to main, fix, come back, `git stash pop`.

---

## 10.6 Cherry-pick — Apply a Specific Commit

```bash
git cherry-pick <commit-sha>
```

Useful for backporting: fix a bug on `main`, cherry-pick to `release/1.0`.

Careful: creates a new commit (different SHA), so history diverges.

---

## 10.7 Merge Conflicts — Handling Them

Happens when Git can't automatically merge changes.

```
<<<<<<< HEAD
your changes
=======
their changes
>>>>>>> feature/x
```

**Steps:**
1. Open conflicted file
2. Decide what to keep (yours, theirs, or both)
3. Remove the `<<<<<<`, `=======`, `>>>>>>>` markers
4. `git add <file>`
5. `git commit` (or `git merge --continue`)

Abort a merge if you get stuck:
```bash
git merge --abort
git rebase --abort
```

**Tools that help:**
- IDE (IntelliJ/VS Code) built-in 3-way merge
- `git mergetool` (kdiff3, meld)

### Prevention
- Pull frequently (`git pull` before you start work)
- Small PRs
- Communicate about files you're touching

---

## 10.8 Tags — Mark Releases

```bash
git tag v1.0.0                                      # lightweight tag
git tag -a v1.0.0 -m "Release 1.0"                  # annotated (recommended)
git tag                                              # list
git show v1.0.0
git push origin v1.0.0
git push origin --tags                               # push all tags
git tag -d v1.0.0                                    # delete local
git push origin --delete v1.0.0                      # delete remote
```

Semantic Versioning: `MAJOR.MINOR.PATCH` (breaking change / feature / fix).

---

## 10.9 Remotes

```bash
git remote -v                                       # list remotes
git remote add upstream https://github.com/other/repo.git
git remote set-url origin https://new-url.git
git fetch upstream                                  # get their changes
git pull upstream main                              # merge them in
```

For forks:
```
origin    → your fork (github.com/YOU/repo)
upstream  → original (github.com/COMPANY/repo)
```

---

## 10.10 Git Workflows

### GitFlow

Complex, prescriptive.

- `main` — production
- `develop` — integration
- `feature/*` — new features (branch off develop, merge back)
- `release/*` — release preparation
- `hotfix/*` — urgent fixes to main

Good for: apps with scheduled releases, versioned software.

### GitHub Flow

Simpler.

- `main` — always deployable
- `feature/*` — branch off main, PR back, deploy after merge

Good for: continuous deployment, web apps.

### Trunk-Based Development

- Everyone commits to `main` (or short-lived branches merged same day)
- Feature flags hide unfinished work
- Requires excellent CI

Good for: high-velocity teams, cloud apps.

---

## 10.11 Pull Requests — The Review Ritual

1. Create feature branch
2. Push to remote
3. Open PR against `main` (or `develop`)
4. CI runs (tests, build, lint)
5. Teammates review, comment, request changes
6. Iterate until approved
7. Merge (squash, rebase, or merge commit — team choice)

### Good PR practices
- Small (< 400 lines is ideal)
- Clear title + description
- Linked to a ticket
- Screenshots for UI changes
- Passing tests before requesting review

---

## 10.12 CI/CD — Continuous Integration & Deployment

### Definitions
- **CI** — every push triggers automated build + test
- **CD** — Continuous Delivery (auto to staging, manual to prod) or Continuous Deployment (auto to prod)

### Typical Pipeline

```
   Push to Git
        │
        ▼
   ┌─────────────┐
   │   Build     │  compile, package
   ├─────────────┤
   │   Test      │  unit + integration
   ├─────────────┤
   │  Lint/SAST  │  SonarQube, security scan
   ├─────────────┤
   │  Docker     │  build & push image
   ├─────────────┤
   │  Deploy Dev │  auto
   ├─────────────┤
   │  Smoke Test │
   ├─────────────┤
   │Deploy Prod  │  manual approval
   └─────────────┘
```

### Jenkins Pipeline example

```groovy
pipeline {
    agent any
    tools { maven 'Maven3'; jdk 'JDK17' }
    environment {
        REGISTRY = 'my-registry.com'
        IMAGE = "bookapp:${env.BUILD_NUMBER}"
    }
    stages {
        stage('Checkout') { steps { checkout scm } }

        stage('Build') { steps { sh 'mvn -B clean compile' } }

        stage('Test') {
            steps { sh 'mvn -B test' }
            post { always { junit '**/target/surefire-reports/*.xml' } }
        }

        stage('SonarQube') {
            steps { sh 'mvn sonar:sonar -Dsonar.host.url=$SONAR_URL' }
        }

        stage('Package') { steps { sh 'mvn package -DskipTests' } }

        stage('Docker') {
            steps {
                sh "docker build -t $REGISTRY/$IMAGE ."
                sh "docker push $REGISTRY/$IMAGE"
            }
        }

        stage('Deploy Dev') {
            steps { sh "kubectl set image deploy/bookapp app=$REGISTRY/$IMAGE -n dev" }
        }

        stage('Deploy Prod') {
            input { message "Deploy to production?" }
            steps { sh "kubectl set image deploy/bookapp app=$REGISTRY/$IMAGE -n prod" }
        }
    }
    post {
        success { slackSend "Build ${env.BUILD_NUMBER} succeeded" }
        failure { slackSend "Build ${env.BUILD_NUMBER} FAILED" }
    }
}
```

### GitHub Actions equivalent

`.github/workflows/ci.yml`:
```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'
          cache: maven
      - run: mvn -B verify

      - name: Upload jar
        uses: actions/upload-artifact@v4
        with: { name: jar, path: target/*.jar }

  docker:
    needs: build
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USER }}
          password: ${{ secrets.DOCKER_PASS }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: myuser/bookapp:${{ github.sha }}
```

### Other CI tools

- **GitLab CI** — `.gitlab-ci.yml`, tightly integrated with GitLab
- **CircleCI** — cloud-focused
- **Azure DevOps** — for Microsoft shops
- **AWS CodePipeline** — AWS-native
- **Travis CI** — historically popular

### Deployment strategies

- **Blue-Green** — two identical envs; switch traffic → zero downtime
- **Canary** — roll out to 5% of users, then 25%, then 100%
- **Rolling** — replace instances one at a time (K8s default)
- **Recreate** — kill all, start new (downtime)

---

## 10.13 Docker — Containerization

### Why containers?

Before Docker: "Works on my machine" 😩.

Docker packages an app + all its dependencies (JDK, libs, config) into a single **image**. The image runs identically on any machine with Docker.

### Image vs Container

- **Image** — read-only blueprint (like a class)
- **Container** — running instance of an image (like an object)

```bash
docker images          # list images
docker ps              # running containers
docker ps -a           # all (including stopped)
```

### Dockerfile — recipe for an image

**Multi-stage build** (recommended — smaller final image):

```dockerfile
# ---------- Build stage ----------
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline                   # cache deps
COPY src ./src
RUN mvn package -DskipTests

# ---------- Runtime stage ----------
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app
COPY --from=build /app/target/*.jar app.jar

# Non-root user for security
RUN addgroup -S app && adduser -S app -G app
USER app

EXPOSE 8080
ENV JAVA_OPTS=""

HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -q --spider http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh","-c","java $JAVA_OPTS -jar app.jar"]
```

Why multi-stage? The final image doesn't need Maven or source — just JRE + jar. Saves 500 MB+.

### Common instructions

| Directive | Purpose |
|-----------|---------|
| `FROM` | Base image |
| `WORKDIR` | Change directory (creates if needed) |
| `COPY` | Copy from host to image |
| `ADD` | Like COPY but can unpack tarballs, fetch URLs |
| `RUN` | Execute during **build** |
| `CMD` | Default command at runtime (overridable) |
| `ENTRYPOINT` | Runtime command (harder to override; args from CMD) |
| `EXPOSE` | Document the port (doesn't actually publish it) |
| `ENV` | Runtime env variables |
| `ARG` | Build-time variables |
| `VOLUME` | Mount point (persistent) |
| `USER` | User to run as |
| `HEALTHCHECK` | Container health |
| `LABEL` | Metadata |

### CMD vs ENTRYPOINT

```dockerfile
ENTRYPOINT ["java","-jar","app.jar"]
CMD ["--server.port=8080"]           # default args
```
Run: `docker run img` → runs `java -jar app.jar --server.port=8080`  
Run: `docker run img --server.port=9090` → args override CMD, ENTRYPOINT stays.

### Build & run

```bash
docker build -t bookapp:1.0 .
docker run -d -p 8080:8080 --name book bookapp:1.0
docker logs -f book
docker exec -it book sh                                 # get a shell
docker stop book && docker rm book
docker rmi bookapp:1.0                                  # remove image
```

### Volumes — persistent data

```bash
# Named volume (managed by Docker)
docker volume create pgdata
docker run -v pgdata:/var/lib/postgresql/data postgres

# Bind mount (host path)
docker run -v $(pwd):/app -w /app node:18 npm test
```

### Networks

Containers on the same network can talk by name.
```bash
docker network create app-net
docker run --network app-net --name db postgres
docker run --network app-net -e DB_HOST=db my-app
```

### .dockerignore — like .gitignore

```
target/
.git/
.idea/
*.log
node_modules/
```

Reduces build context sent to Docker daemon → faster builds.

### Registries

Public: Docker Hub.  
Private: Amazon ECR, Google GCR, Azure ACR, GitLab Container Registry.

```bash
docker tag bookapp:1.0 myuser/bookapp:1.0
docker login
docker push myuser/bookapp:1.0
docker pull myuser/bookapp:1.0
```

### Best practices

1. Use **specific tags** (`node:18-alpine`), not `latest`
2. **Multi-stage builds** for smaller images
3. **Non-root user** (security)
4. **`.dockerignore`** to reduce context
5. **Combine RUN commands** to reduce layers:
   ```dockerfile
   RUN apt-get update && \
       apt-get install -y curl && \
       rm -rf /var/lib/apt/lists/*
   ```
6. **Copy dependencies before code** for better caching
7. **Health checks** for orchestrators to know status
8. **Don't store secrets** in the image; inject via env vars

---

## 10.14 Docker Compose — Multi-Container Apps

Define a whole stack in one YAML.

```yaml
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://db:5432/mydb
      SPRING_DATASOURCE_USERNAME: user
      SPRING_DATASOURCE_PASSWORD: pass
    depends_on:
      db: { condition: service_healthy }
    restart: unless-stopped

  db:
    image: postgres:15
    environment:
      POSTGRES_DB: mydb
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U user"]
      interval: 10s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]

  kafka:
    image: bitnami/kafka:latest
    environment:
      KAFKA_CFG_NODE_ID: 1
      KAFKA_CFG_PROCESS_ROLES: broker,controller
      # ...

volumes:
  pgdata:
```

Commands:
```bash
docker-compose up -d                                    # start all in background
docker-compose logs -f app                              # follow app logs
docker-compose ps                                        # status
docker-compose exec app sh                              # shell into app
docker-compose down                                     # stop & remove
docker-compose down -v                                  # also remove volumes
```

Perfect for local development: everything you need in one command.

---

## 10.15 Kubernetes (K8s) — Container Orchestration

When you have dozens/hundreds of containers, you need an orchestrator.

**Kubernetes does:**
- Runs containers on a cluster of nodes
- Auto-restarts failed containers
- Scales up/down based on load
- Load balances traffic
- Handles rolling deployments & rollbacks
- Manages secrets and config

### Key concepts

- **Pod** — smallest unit (1+ containers sharing network/storage)
- **Deployment** — declares desired state (e.g., "3 replicas of bookapp:1.0"); manages ReplicaSets
- **ReplicaSet** — ensures N pods are running
- **Service** — stable network endpoint for pods (Load balances)
- **Ingress** — HTTP routing from outside
- **ConfigMap** — non-secret config
- **Secret** — sensitive config
- **Namespace** — logical grouping

### Simple Deployment + Service

```yaml
apiVersion: apps/v1
kind: Deployment
metadata: { name: bookapp }
spec:
  replicas: 3
  selector: { matchLabels: { app: bookapp } }
  template:
    metadata: { labels: { app: bookapp } }
    spec:
      containers:
        - name: app
          image: myuser/bookapp:1.0
          ports: [{ containerPort: 8080 }]
          env:
            - name: SPRING_PROFILES_ACTIVE
              value: prod
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef: { name: db-secret, key: password }
          resources:
            requests: { cpu: 250m, memory: 256Mi }
            limits:   { cpu: 500m, memory: 512Mi }
          livenessProbe:
            httpGet: { path: /actuator/health, port: 8080 }
            initialDelaySeconds: 30
          readinessProbe:
            httpGet: { path: /actuator/health, port: 8080 }
            initialDelaySeconds: 10
---
apiVersion: v1
kind: Service
metadata: { name: bookapp }
spec:
  selector: { app: bookapp }
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP                              # internal-only. LoadBalancer for external
```

### kubectl commands

```bash
kubectl apply -f deploy.yaml
kubectl get pods
kubectl get pods -o wide                                # more info
kubectl describe pod bookapp-xxx
kubectl logs -f bookapp-xxx
kubectl exec -it bookapp-xxx -- sh
kubectl scale deploy bookapp --replicas=5
kubectl rollout status deploy bookapp
kubectl rollout undo deploy bookapp                     # rollback!
kubectl set image deploy/bookapp app=myuser/bookapp:1.1  # update
kubectl delete pod bookapp-xxx                          # will be recreated
```

### Helm — the "package manager" for K8s

Templated YAML with versioned charts.

```bash
helm install bookapp ./mychart --values values-prod.yaml
helm upgrade bookapp ./mychart
helm rollback bookapp 1
```

---

## 10.16 Monitoring & Observability

### The 3 pillars

1. **Metrics** — Prometheus scrapes `/actuator/prometheus`, Grafana visualizes
2. **Logs** — ELK stack (Elasticsearch + Logstash + Kibana), or Loki + Grafana
3. **Traces** — Zipkin/Jaeger with Sleuth/OpenTelemetry

### Alerting

- Prometheus AlertManager
- PagerDuty / Opsgenie for on-call
- Slack notifications

### Common metrics to alert on

- Error rate > 1%
- P99 latency > 500ms
- CPU > 80% for > 5 min
- Memory > 85%
- Disk > 80%
- Kafka consumer lag > threshold
- HTTP 5xx spike

---

## 10.17 Infrastructure as Code (IaC)

Instead of clicking around cloud consoles, describe infra as code.

- **Terraform** — cloud-agnostic, declarative
- **AWS CloudFormation** — AWS-specific
- **Pulumi** — using real programming languages
- **Ansible** — configuration management

Example (Terraform):
```hcl
resource "aws_instance" "app" {
  ami           = "ami-12345"
  instance_type = "t3.medium"
  tags = { Name = "bookapp-server" }
}
```

Version-controlled, reproducible infrastructure.

---

## 10.18 Interview Question Bank

1. **git merge vs rebase?**  
   Merge preserves history with a merge commit. Rebase replays commits for linear history. Never rebase shared branches.

2. **How to resolve merge conflicts?**  
   Edit files to keep desired changes, remove conflict markers, `git add`, `git commit`.

3. **What is `git stash`?**  
   Temporarily save uncommitted changes so you can switch context. `git stash pop` to restore.

4. **`git reset` vs `git revert`?**  
   Reset rewrites history (moves HEAD). Revert creates a new commit that undoes (safe on shared branches).

5. **What is CI/CD?**  
   CI = auto build/test on push. CD = auto deploy (delivery = staging manual prod; deployment = auto prod).

6. **What is Docker? Why use it?**  
   Container runtime — packages app + dependencies into portable images. Solves "works on my machine".

7. **Image vs Container?**  
   Image is the blueprint (class). Container is a running instance (object).

8. **`CMD` vs `ENTRYPOINT`?**  
   ENTRYPOINT: the always-run command. CMD: default args for ENTRYPOINT (overridable).

9. **Explain multi-stage Docker build.**  
   Build in a heavy image (with Maven/Node), copy the artifact to a slim runtime image. Final image is small and secure.

10. **What is Docker Compose?**  
    Tool to define/run multi-container apps via YAML.

11. **What is Kubernetes?**  
    Container orchestrator — schedules containers, auto-heals, scales, load balances.

12. **How do you deploy your app?**  
    Push → CI (Jenkins) builds/tests → Docker image → push to registry → K8s deploy via Helm → smoke tests → traffic switch.

13. **How to rollback a deployment?**  
    `kubectl rollout undo deploy/my-app` or redeploy previous image tag.

14. **What is `.dockerignore`?**  
    Files to exclude from build context (target/, .git/, etc.). Speeds up builds.

15. **How to view logs?**  
    `docker logs -f <container>`, `kubectl logs -f <pod>`. Centralized via ELK.

16. **Blue-green vs canary deployment?**  
    Blue-green: two envs, swap traffic. Canary: gradual rollout (1% → 10% → 100%).

17. **How do you store secrets?**  
    K8s Secrets, HashiCorp Vault, AWS Secrets Manager. NEVER in code or images.

18. **What are health probes in K8s?**  
    Liveness (restart if failing), Readiness (route traffic only when ready), Startup (grace period on boot).

19. **How to reduce Docker image size?**  
    Alpine base, multi-stage, minimize RUN layers, use distroless.

20. **What is `git cherry-pick`?**  
    Apply a specific commit from another branch. Common for backporting fixes.

---

## 10.19 Cheat Sheet

```
Git basics: clone, add, commit, push, pull
Branch: git switch -c feat/x
Merge: preserves history; Rebase: linear history (never on public branches)
Undo: restore (unstaged), reset (private), revert (public)
Stash: save WIP; cherry-pick: apply specific commit
CI/CD: build → test → image → deploy (Jenkins/GH Actions)
Docker: multi-stage, non-root user, .dockerignore, specific tags
Docker Compose: local multi-container dev
K8s: Deployment + Service + Ingress; kubectl / helm
Observability: metrics (Prom) + logs (ELK) + traces (Zipkin)
```

---

## Practical Assignments

1. Clone a repo, create a feature branch, make changes, open a PR with a good description.
2. Interactively rebase 3 commits into 1 (`rebase -i`).
3. Simulate a merge conflict and resolve it.
4. Dockerize a Spring Boot app with a multi-stage Dockerfile; run it.
5. Write a Docker Compose file with app + PostgreSQL; verify DB connection works.
6. Set up a GitHub Actions workflow that builds and tests on every push.
7. Deploy to K8s locally via minikube: Deployment + Service + verify with `kubectl port-forward`.

Master these and DevOps interviews are done. 🚀