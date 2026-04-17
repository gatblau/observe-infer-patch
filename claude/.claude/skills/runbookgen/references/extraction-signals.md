# Extraction signals

What source patterns map to what runbook sections. Use these as search anchors — every fact in the runbook must cite `path:line`, never a pattern match alone.

## Startup

- **Go:** `func main()` in any `main` package. Look for `flag.Parse()`, `os.Getenv`, config file loads, DB/NATS/HTTP client construction.
- **Node/TS:** entry from `package.json` `"main"` or `"bin"`; `process.env` reads; `app.listen(...)`.
- **Python:** `if __name__ == "__main__":`; `argparse`, `os.environ`, framework bootstrap (`FastAPI`, `Flask`, `Django manage.py`).
- **Container entry:** `Dockerfile` `ENTRYPOINT` / `CMD`; `docker-compose.yml` `command:`; Kubernetes manifest `spec.containers[].command`.
- **Systemd:** `*.service` `ExecStart=`.

Record: the precise line that launches the listener or worker loop.

## Shutdown

- **Go:** `signal.Notify` with `SIGTERM`/`SIGINT`; `context.WithCancel`; `server.Shutdown(ctx)`; deferred `Close()` on clients.
- **Node:** `process.on('SIGTERM', ...)`; `server.close()`.
- **Python:** `signal.signal(signal.SIGTERM, ...)`; framework-specific shutdown hooks.
- **Container:** `STOPSIGNAL` in Dockerfile; `terminationGracePeriodSeconds` in Kubernetes manifests.

Capture the timeout if set. If no signal handling is found, note it — the service may be killed abruptly.

## Ports and endpoints

- **HTTP listeners:** `http.ListenAndServe`, `http.Server{Addr:}`, `app.listen`, `uvicorn.run`.
- **gRPC:** `grpc.NewServer` + `Serve(listener)`.
- **NATS subscriptions:** `nats.Subscribe`, `QueueSubscribe`, `js.Subscribe`. Record the subject.
- **Metrics / pprof:** `promhttp.Handler`, `/debug/pprof`.

For each, cite the exact bind line and any route registration.

## Configuration

- **Env vars:** `os.Getenv`, `os.LookupEnv`, `process.env.X`, `os.environ.get`. Check for defaults set inline or via a config struct.
- **Flags:** `flag.String/Int/Bool`, `cobra.Command.Flags`, `argparse.add_argument`.
- **Config files:** `viper.ReadInConfig`, YAML/TOML loaders, `.env` loaders. Record the default search path.

Mark anything whose name contains `SECRET`, `TOKEN`, `PASSWORD`, `KEY`, `CREDENTIAL` as a secret regardless of how it is read.

## Dependencies

- **Databases:** `sql.Open`, `pgx.Connect`, `mongo.Connect`. DSN source.
- **Queues / streaming:** `nats.Connect`, `kafka.NewReader`, `amqp.Dial`.
- **Outbound HTTP:** `http.Client`, `http.NewRequest`. Note base URLs built from config.
- **Object storage:** `s3.New`, `minio.New`.

Role = what the service uses it for. Failure impact = what the service does when the dependency is unreachable (reads existing connection pool behaviour — retries, backoff, fail-fast).

## Health and liveness

- **HTTP routes** named `/healthz`, `/readyz`, `/livez`, `/health`, `/ready`. Also check for custom handlers registered on a metrics listener.
- **Kubernetes probes:** `livenessProbe`, `readinessProbe`, `startupProbe`.

If no readiness endpoint exists and the service has dependencies, call this out — operators cannot tell warm-up from failure.

## Alert-worthy signals

- **Panics / fatal logs:** `log.Fatal`, `panic(...)`, `logger.Error(...)` with non-debug severity.
- **Sentinel errors:** `var ErrX = errors.New(...)` that are logged or returned from handlers.
- **HTTP 5xx paths:** `http.Error(w, ..., 500)` or `w.WriteHeader(500)`.
- **Circuit-breaker trips / retries exhausted:** library-specific — grep for the breaker package.
- **Prometheus counters** with names containing `errors`, `failures`, `drops`.

## Rollback

- **Image tags:** previous `git tag` output; `image:` field in deployment manifests.
- **Migrations:** `schema/**` directory; paired `up.sql` / `down.sql` files.
- **Feature flags:** look for a flags package import (LaunchDarkly, GrowthBook, internal flag loader).

If the deployment manifest pins `latest` or has no tag discipline, flag it as a gap — rollback is not reliably reproducible.
