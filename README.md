# Health Check Lab

## Setup
```bash
docker compose up -d
```

## Verify Service
```bash
curl localhost:3000/orders
```

Liveness and readiness checks:

```bash
curl -i localhost:3000/health  # 200 OK; does not use the database
curl -i localhost:3000/ready   # 200 READY or 503 NOT READY
docker compose ps              # includes the orders-api health status
```

## Stop Database
```bash
docker compose stop db
```

## Observe Behaviour
```bash
docker compose ps
```

After stopping the database, `/health` remains available while `/ready` returns
`503 NOT READY` and the API container becomes `unhealthy` after the configured
healthcheck retries. Restore the database with `docker compose start db` and
wait for `docker compose ps` to report `healthy` again. See [CLOUD_RUN.md](CLOUD_RUN.md)
for the cloud mapping and the Docker restart-policy limitation.
