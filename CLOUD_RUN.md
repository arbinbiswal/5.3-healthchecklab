# Cloud Run Health Checks

This service exposes two checks with deliberately different responsibilities:

- `/health` is the liveness check. It only proves that the Node.js process can accept requests and never depends on Postgres.
- `/ready` is the readiness check. It runs `SELECT 1` through the Postgres connection pool and returns `200 READY` when the database is reachable, or `503 NOT READY` otherwise.

## Cloud Run mapping

Configure a Cloud Run **startup probe** against `/health` so a revision receives traffic only after the HTTP server is listening. Configure a **liveness probe** against `/health` so Cloud Run can replace an instance whose process is no longer responding.

Cloud Run does not provide a Kubernetes-style readiness probe for removing an individual instance from traffic based on a dependency check. Use `/ready` in the application, monitoring, and load-test checks to report database availability. Keep the dependency check out of `/health`; otherwise a temporary database outage can cause unnecessary instance replacement.

The local Compose healthcheck targets `/ready`, which makes database loss visible in `docker compose ps`. The `unless-stopped` restart policy is also configured, but Docker only restarts a container when its main process exits; marking a running container unhealthy alone does not trigger a Docker restart. When Postgres is restored, the readiness check returns healthy again without restarting the API process.

## Optional Kubernetes equivalent

For Kubernetes, map `/health` to `livenessProbe` and `startupProbe`, and map `/ready` to `readinessProbe`. Kubernetes can then stop routing traffic to an instance that cannot reach Postgres without restarting it solely because the dependency is temporarily unavailable.
