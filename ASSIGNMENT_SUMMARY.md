# Health Check Lab Assignment Summary

## 1. Original Requirements

The assignment started with a Docker Compose project containing an `orders-api`
Node.js service and a PostgreSQL database. The API process could remain running
even when the database connection was broken, so Docker reported only `Up` and
could not identify or recover the failed service.

The requested work was:

- Create `GET /health` as a liveness check.
  - Return HTTP `200` with `OK` when the application process is alive.
  - Do not check the database from this endpoint.
- Create `GET /ready` as a readiness check.
  - Check a real database dependency.
  - Return HTTP `200` with `READY` when PostgreSQL is reachable.
  - Return HTTP `503` with `NOT READY` when PostgreSQL is unavailable.
- Add a container healthcheck and restart policy.
  - Make Compose report `healthy` or `unhealthy`, not only `Up`.
  - Configure `restart: unless-stopped` for runtime recovery behavior.
- Validate the complete failure flow.
  - Start the services and confirm healthy status.
  - Confirm `/ready` returns `200`.
  - Stop the database and confirm `/ready` returns `503` and the API becomes `unhealthy`.
  - Restore the database and confirm the API becomes healthy again.
  - Capture evidence with `docker compose ps`, `curl`, `docker inspect`, or logs.
- Add a short cloud mapping document for Cloud Run startup/liveness checks and,
  optionally, Kubernetes probes.
- Submit the work through a branch, meaningful commit, GitHub pull request, and
  a short video walkthrough.

These tasks separate process health from dependency health. A liveness check
answers whether the process can respond, while a readiness check answers whether
the service can currently perform useful work.

## 2. Changes Implemented

### Modified files

- `src/server.js`
  - Added `/health` and `/ready` routes.
- `src/db.js`
  - Added a reusable database ping operation.
- `Dockerfile`
  - Installed `curl` so the container healthcheck command is available.
- `docker-compose.yml`
  - Added API and database restart policies.
  - Added API and database healthchecks.
  - Made the API wait for a healthy database.
  - Removed the obsolete Compose `version` field.
- `README.md`
  - Added endpoint checks and the database failure/recovery workflow.
  - Documented the Docker restart-policy limitation.

### Added files

- `CLOUD_RUN.md`
  - Maps the endpoints to Cloud Run and Kubernetes health-probe concepts.
- `.gitignore`
  - Prevents locally installed `node_modules` from being committed.
- `package-lock.json`
  - Records the installed Node.js dependency versions.

### Unchanged supporting files

- `package.json` continues to define the Express, PostgreSQL, and `npm start`
  setup.
- `db/init.sql` continues to create and seed the `orders` table.
- `docs/architecture.md` continues to describe the API and PostgreSQL data flow.

No project files were removed.

## 3. Implementation Details

### Liveness endpoint

`src/server.js` now exposes:

```text
GET /health -> 200 OK
```

The route sends its response directly and never calls the database. This makes
it suitable for checking whether the Node.js process and HTTP server are alive.

### Readiness endpoint

`src/server.js` exposes:

```text
GET /ready -> 200 READY       when PostgreSQL responds
GET /ready -> 503 NOT READY   when PostgreSQL cannot be reached
```

The route calls `db.ping()`. In `src/db.js`, `db.ping()` executes `SELECT 1`
through the existing PostgreSQL connection pool. The failed query is caught and
converted into HTTP `503`, so a database outage is reported honestly without
crashing the request handler.

### Compose healthchecks and startup ordering

The API service has this behavior configured in `docker-compose.yml`:

- `restart: unless-stopped` allows Docker to restart the service if its main
  process exits, unless a user intentionally stopped it.
- The healthcheck calls `curl -f http://localhost:3000/ready` every 10 seconds.
- Each check has a 3-second timeout, a 20-second startup grace period, and three
  retries.
- The API depends on the database reaching its own `pg_isready` healthcheck.

The API healthcheck intentionally targets `/ready`, rather than `/health`, so a
database outage changes the Compose status to `unhealthy`. The Dockerfile
installs `curl` because the Node Alpine image does not otherwise guarantee that
the healthcheck command is available.

The database also has `restart: unless-stopped` and a `pg_isready` healthcheck.

### Restart-policy limitation

Docker Compose restart policies restart a container when its main process exits;
they do not restart a still-running container solely because a healthcheck marks
it `unhealthy`. Therefore, this implementation correctly detects database loss,
reports the API as unhealthy, and becomes healthy again after PostgreSQL returns.
It does not falsely claim that Docker automatically restarted the still-running
API process. `CLOUD_RUN.md` and the README document this behavior.

### Cloud mapping

`CLOUD_RUN.md` documents the following mapping:

- Cloud Run startup probe: `/health`.
- Cloud Run liveness probe: `/health`.
- Application and monitoring readiness check: `/ready`.
- Kubernetes startup/liveness probes: `/health`.
- Kubernetes readiness probe: `/ready`.

The document also explains that Cloud Run does not provide a Kubernetes-style
readiness probe for removing one instance from traffic based on a dependency
check.

## 4. Important Commands

### Branch and project setup

```bash
git clone https://github.com/kalviumcommunity/health-check-lab.git
```

Clones the assignment repository.

```bash
cd health-check-lab
```

Moves into the project directory.

```bash
git checkout -b feat/health-checks
```

Creates and switches to the feature branch requested by the assignment.

### Install and start

```bash
npm install
```

Installs the Express and PostgreSQL Node.js dependencies and creates the lockfile.

```bash
docker compose up -d --build
```

Builds the API image and starts the API and database containers in the background.

```bash
docker compose ps
```

Shows container state, including the API health status after healthchecks run.

### Endpoint checks

```bash
curl -i localhost:3000/health
```

Checks process liveness and should return HTTP `200` with `OK`.

```bash
curl -i localhost:3000/ready
```

Checks database readiness and should return `200 READY` or `503 NOT READY`.

```bash
curl localhost:3000/orders
```

Checks the existing orders endpoint and its database query.

### Failure and recovery demonstration

```bash
docker compose stop db
```

Stops PostgreSQL to simulate a dependency failure.

```bash
docker compose ps
```

After the configured retries, confirms that the API is marked `unhealthy`.

```bash
curl -i localhost:3000/health
curl -i localhost:3000/ready
```

Shows that liveness remains available while readiness returns `503 NOT READY`.

```bash
docker compose start db
```

Starts PostgreSQL again.

```bash
docker compose ps
```

Confirms that the API health status returns to `healthy` after PostgreSQL is
reachable again.

```bash
docker inspect --format='{{json .State.Health}}' <orders-api-container>
```

Displays detailed healthcheck state and recent healthcheck results.

### Local validation commands used

```bash
node --check src/server.js && node --check src/db.js
```

Checks JavaScript syntax without starting the service.

```bash
docker compose config --quiet
```

Validates the Compose configuration without starting containers.

```bash
git diff --check
```

Checks the Git diff for whitespace errors.

```bash
git status --short --branch
```

Shows the current branch and any uncommitted changes.

## 5. Overall Progress

### Completed

- The requested feature branch was created: `feat/health-checks`.
- Liveness and readiness endpoints were implemented.
- Database ping behavior was implemented.
- Compose healthchecks and restart policies were added.
- Database-aware API startup ordering was added.
- Cloud Run and Kubernetes mapping documentation was added.
- README validation instructions were updated.
- JavaScript syntax validation passed.
- Compose configuration validation passed.
- Dependency installation passed with zero reported vulnerabilities.
- Git diff validation passed.
- Commit created: `e2d61b6 Add service health checks`.
- Branch pushed to the configured GitHub repository.
- Pull request opened: [Add liveness and readiness health checks](https://github.com/arbinbiswal/5.3-healthchecklab/pull/1).

### Remaining or requiring attention

- Live Docker healthy, broken, and recovered output was not captured in this
  environment. Docker failed while pulling `node:18-alpine` with:
  `x509: certificate signed by unknown authority`.
- The Docker Hub certificate/trust configuration must be fixed before running
  the full Compose demonstration and adding real `docker compose ps` and `curl`
  output to the pull request.
- The required short video walkthrough remains a submission task and was not
  created by this repository change.
- The pull request records the validation limitation honestly; it does not claim
  that the blocked live Docker evidence was completed.
