# go-microservices

A small microservice system written in Go, built while working through Trevor Sawler's
*Working with Microservices in Go* course. It demonstrates how independent services talk
to each other over REST, RPC, gRPC and a message queue, and how the whole stack is
containerised and deployed with Docker Compose and Docker Swarm behind Caddy.

## Architecture

```
                 ┌────────────┐
  front-end ───▶ │   broker   │ ── REST ──▶ authentication-service ──▶ PostgreSQL
  (Go templates) │  service   │ ── REST ──▶ mail-service ──▶ MailHog (SMTP)
                 │            │ ── gRPC ──▶ logger-service ──▶ MongoDB
                 │            │ ── AMQP ──▶ RabbitMQ ──▶ listener-service ──▶ logger-service
                 └────────────┘
```

| Service | Responsibility | Tech |
|---|---|---|
| `broker-service` | Single entry point; routes requests by `action` (`auth`, `log`, `mail`) | chi router, REST, gRPC client, RabbitMQ publisher |
| `authentication-service` | Validates user credentials | PostgreSQL (`pgx` via `database/sql`), bcrypt |
| `logger-service` | Stores log entries | MongoDB; exposes REST, `net/rpc` and gRPC |
| `mail-service` | Renders HTML/plain templates and sends mail | go-simple-mail, MailHog in development |
| `listener-service` | Consumes events from a RabbitMQ topic exchange and forwards them | amqp091-go |
| `front-end` | Simple test page to exercise the services | `html/template` |

### Go features used
- **Concurrency:** the logger runs its HTTP, RPC and gRPC servers in separate goroutines;
  the listener handles each consumed message in its own goroutine.
- **Context:** database calls in the authentication service use `context.WithTimeout`.
- **Protocol Buffers / gRPC:** shared `logs.proto` contract between broker and logger.
- **Small images:** each service is compiled as a static Linux binary
  (`CGO_ENABLED=0`) and copied into a minimal Alpine image.

## Running locally

Requirements: Go, Docker, Docker Compose, `make`.

```bash
make up_build     # build all binaries and start the stack
make down         # stop everything
```

- Broker: http://localhost:8080
- MailHog UI: http://localhost:8025

Credentials in `project/docker-compose.yml` are local development defaults only.

## Deploying with Docker Swarm

`project/swarm.yml` describes the same stack for Docker Swarm, with Caddy as the reverse
proxy (automatic HTTPS) in front of the front-end and broker.

```bash
docker swarm init
docker stack deploy -c project/swarm.yml myapp
```

## Notes

My Turkish-language learning notes from building this project are in
[`MICROSERVICES_OGRENME_GUNLUGU.md`](MICROSERVICES_OGRENME_GUNLUGU.md).
