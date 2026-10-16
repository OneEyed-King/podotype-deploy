# owl-trace

A self-hosted dashboard that shows you what's actually happening inside your
Docker stack — which services are healthy, how requests move between them,
and what their logs say — without touching a line of your application code.

Point it at a running Docker Compose stack and it watches HTTP/gRPC traffic
between your containers using eBPF (via [Beyla](https://github.com/grafana/beyla)),
polls Docker for health/CPU/memory, and gives you one dashboard for traces,
logs, and metrics. No SDKs, no code changes, no restarting the services you
want to watch.

Everything it collects stays on your own machine. See
[Data & privacy](#data--privacy).

## What's in this repo

This repo is a single install file (`docker-compose.yaml`) and this README.
It's enough to run owl-trace with one command — you don't need to clone
anything or build from source. The engine, web UI, and Beyla images it
pulls are prebuilt and published separately.

Licensed under the [Elastic License 2.0](./LICENSE): free to self-host and
use, including in production, for as long as you like — the only thing you
can't do is turn around and resell it as a hosted service to other people.
Full terms in `LICENSE`.

## Requirements

- A Linux host (Beyla's eBPF tracing needs a Linux kernel directly — this
  won't work on macOS or Windows unless you run it inside a Linux VM)
- Docker Engine with the Compose plugin (`docker compose version` should
  work)
- 1–2 GB of free RAM is comfortable for a demo-sized stack (ClickHouse is
  the main consumer)

## Install it

```bash
curl -fsS https://raw.githubusercontent.com/OneEyed-King/owl-trace-deploy/main/docker-compose.yaml \
  | docker compose -f - up -d
```

This pulls four images (`engine`, `web`, `beyla`, `clickhouse`) and starts
them. The first run takes a minute or two while the images download.

Once it's up, open **http://localhost:8973** (or whichever port you set
below). Running this on a remote server instead of your laptop? Read
[Accessing it remotely](#accessing-it-remotely) before you touch any
firewall rules.

### Changing the ports

If something on your machine already uses port 8973 or 4318, set these
environment variables before the `curl` command:

```bash
OWLTRACE_UI_PORT=9000 OWLTRACE_OTLP_PORT=4319 \
  curl -fsS https://raw.githubusercontent.com/OneEyed-King/owl-trace-deploy/main/docker-compose.yaml \
  | docker compose -f - up -d
```

| Variable | Default | What it's for |
|---|---|---|
| `OWLTRACE_UI_PORT` | `8973` | The dashboard itself |
| `OWLTRACE_OTLP_PORT` | `4318` | Where any service you fully instrument sends trace data |
| `OWL_ENGINE_API_KEY` | a fixed dev default | Shared secret between the internal `web`/`engine` containers — only matters if you expose the engine beyond this one Docker network |

ClickHouse's own port is **not published** by default (see
[Data & privacy](#data--privacy) for why). If you want to point your own
Grafana or BI tool at it, there's a note at the bottom of
`docker-compose.yaml` on how to do that safely.

## How it works

- **Beyla** runs as a privileged container using eBPF to watch network
  traffic on the host. It sees HTTP/gRPC requests between your other
  containers with no code changes, no agents installed into them, and no
  restarts required.
- **engine** receives that trace data, separately polls the Docker socket
  for container health/CPU/memory, stores everything in **ClickHouse**, and
  serves a small internal API.
- **web** serves the dashboard UI and talks to `engine` on your behalf.
- All four containers communicate over one internal Docker network
  (`owl-trace`) that this compose file creates. Nothing here touches any
  other stack you have running unless you deliberately opt a service into
  deeper tracing.

## Data & privacy

Everything owl-trace collects — traces, logs, container metrics — stays in
the ClickHouse container on your own host. Nothing goes to us (Observingowl
Studios) or any third party by default.

Two exceptions, both things you have to turn on yourself:

- **Alert notifications.** If you configure a Telegram bot or webhook URL in
  the alerts settings, owl-trace sends messages *there* when something
  trips an alert. That's a destination you chose, not a hidden default.
- **Optional instrumentation agents.** If you opt a specific service into
  full OpenTelemetry auto-instrumentation (a deeper tracing mode than
  Beyla's default eBPF capture), owl-trace downloads the matching
  OpenTelemetry agent package from GitHub's own release URLs the first time
  you turn that on for that service. That's a one-time download, not data
  being sent out.

Outside those two cases, nothing phones home. There's no telemetry or
analytics built into the tool itself.

## Using it

1. Open the dashboard and you'll see every container Docker knows about,
   listed as "discovered."
2. Click a service to start watching it. This turns on health/CPU/memory
   polling, and if Beyla already sees HTTP traffic to it, live traces show
   up automatically with no extra setup.
3. Each watched service has tabs for recent traces, live/clustered/historical
   logs, and metrics history. Logs are read live from Docker by default; the
   "History" tab lets you opt a service into ClickHouse-backed log storage
   for searchable, longer-retained logs (uses more disk — the UI tells you
   the tradeoff before you turn it on).
4. Retention for both traces and stored log patterns is adjustable from the
   dashboard, with a live estimate of how much disk space each setting will
   use.

If a service shows "no traces yet," it's either genuinely idle (no traffic
to capture) or it doesn't speak an HTTP/gRPC-shaped protocol Beyla can see
as a trace at all — Kafka, InfluxDB, and Telegraf are common examples. That's
expected, not a misconfiguration.

## Accessing it remotely

Running owl-trace on a VPS or cloud box instead of your laptop? Docker
publishing a port (what this compose file does) is separate from your
cloud provider's firewall — both have to allow the traffic through.

**Just testing privately and don't want to touch firewall rules at all:**

```bash
ssh -L 8973:localhost:8973 you@your-server
```

Then open `http://localhost:8973` in your own browser as if it were local.
Nothing on the server's firewall changes — the tunnel does the work.

**Want to show someone else the dashboard directly:**
Open only `OWLTRACE_UI_PORT` (8973 by default) in your provider's firewall or
security group. Do **not** open ClickHouse's port for this — it has no
authentication, which is exactly why this compose file keeps it unpublished
by default. The dashboard port itself is genuine, unencrypted HTTP at this
stage: fine for a quick demo to someone you trust the link with, not
something to leave open long-term or put real credentials behind. A domain
with TLS via a reverse proxy is a natural next step, but it's outside the
scope of this quick-start.

## Removing it

```bash
docker compose -p owl-trace down -v --rmi all
```

- `-v` removes the named volumes — this is what actually frees the
  ClickHouse data and owl-trace's own state, not `down` alone.
- `--rmi all` removes all four images this compose file references.

If Beyla's container refuses to stop (`container ... PID ... is zombie and
can not be killed` — a known quirk of eBPF containers without their own
init), force-remove it first, then re-run the command above:

```bash
docker rm -f owl-trace-beyla
docker compose -p owl-trace down -v --rmi all
```

Check that nothing owl-trace-related is left:

```bash
docker ps -a --filter "label=com.docker.compose.project=owl-trace"
docker volume ls --filter "label=com.docker.compose.project=owl-trace"
docker images | grep -E "owl-trace|clickhouse"
```

All three commands should come back empty.
