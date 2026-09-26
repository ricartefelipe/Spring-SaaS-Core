# Spring SaaS Core

[![CI](https://github.com/ricartefelipe/spring-saas-core/actions/workflows/ci.yml/badge.svg)](https://github.com/ricartefelipe/spring-saas-core/actions/workflows/ci.yml)
[![Build & Push](https://github.com/ricartefelipe/spring-saas-core/actions/workflows/build-push.yml/badge.svg)](https://github.com/ricartefelipe/spring-saas-core/actions/workflows/build-push.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Java](https://img.shields.io/badge/Java-21-ED8B00?logo=openjdk&logoColor=white)](https://openjdk.org/)
[![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.2-6DB33F?logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791?logo=postgresql&logoColor=white)](https://www.postgresql.org/)
[![Redis](https://img.shields.io/badge/Redis-7-DC382D?logo=redis&logoColor=white)](https://redis.io/)
[![RabbitMQ](https://img.shields.io/badge/RabbitMQ-3-FF6600?logo=rabbitmq&logoColor=white)](https://www.rabbitmq.com/)
[![Docker](https://img.shields.io/badge/Docker-ready-2496ED?logo=docker&logoColor=white)](docker-compose.yml)
[![OpenAPI](https://img.shields.io/badge/OpenAPI-3.0-6BA539?logo=openapiinitiative&logoColor=white)](docs/api/openapi.yaml)

Control plane para uma plataforma SaaS B2B multi-tenant.

O Spring SaaS Core centraliza tenants, identidade, autorização, feature flags, auditoria e publicação confiável de eventos. A proposta é manter essas regras em um núcleo comum para que serviços escritos em Java, Node.js e Python compartilhem o mesmo contrato de identidade e governança.

Na suíte de referência, ele se integra a `node-b2b-orders` e `py-payments-ledger` por JWT e eventos.

## Visão geral

| Área | Responsabilidade |
|------|------------------|
| **Tenants** | Organizações, planos, região e ciclo de vida |
| **RBAC / ABAC** | Permissões por papel, plano e região, com `DENY` prioritário e `default-deny` |
| **Feature flags** | Ativação por tenant, rollout percentual e targeting por roles |
| **Identidade** | Claims JWT comuns para serviços Java, Node e Python |
| **Auditoria** | Registro de operações e negações de acesso |
| **Outbox** | Persistência transacional de eventos e publicação via RabbitMQ |
| **Observabilidade** | Métricas, tracing e correlação por requisição e tenant |

## Stack

| Camada | Tecnologia |
|--------|------------|
| Runtime | Java 21, Spring Boot 3.x, Maven |
| Segurança | Spring Security, JWT Resource Server, HS256 local e OIDC em produção |
| Dados | PostgreSQL, Liquibase |
| Cache | Redis |
| Mensageria | RabbitMQ |
| Observabilidade | Micrometer, Prometheus, OpenTelemetry, MDC |
| API | REST, OpenAPI, Swagger UI |
| Operação | Docker Compose, GitHub Actions, Grafana |

## Arquitetura

O serviço funciona como ponto central de governança da plataforma.

```text
Frontend / Admin
       |
       v
spring-saas-core
       |
       +--> PostgreSQL
       +--> Redis
       +--> RabbitMQ
       |
       +--> contrato JWT compartilhado
                 |
                 +--> node-b2b-orders
                 +--> py-payments-ledger
```

Estrutura principal do código:

```text
src/main/java/com/union/solutions/saascore/
├── domain/              # Entidades e regras de domínio
├── application/         # Casos de uso, services e ABAC evaluator
├── adapters/
│   ├── in/rest/         # Controllers REST
│   ├── in/auth/         # Filtros JWT e dev token
│   └── out/persistence/ # JPA e repositories
├── config/              # Security, OpenAPI, Web e JWT
├── infrastructure/      # Token issuer
└── observability/       # Correlação e métricas
```

Diagramas C4 e ERD estão em `docs/architecture/`.

## Regras de autorização

1. O JWT é validado.
2. `X-Tenant-Id` deve corresponder à claim `tid`, salvo exceções previstas para administração global.
3. Os endpoints exigem `permission_code`.
4. Políticas podem restringir acesso por plano e região.
5. `DENY` tem precedência sobre `ALLOW`.
6. Sem política aplicável, o acesso é negado.
7. Negações são registradas no audit log como `ACCESS_DENIED`.

Exemplo de claims:

```json
{
  "sub": "user@example.com",
  "tid": "uuid-do-tenant",
  "roles": ["admin"],
  "perms": ["tenants:read", "tenants:write"],
  "plan": "enterprise",
  "region": "us-east-1"
}
```

## Quick Start

### Pré-requisitos

- Java 21+
- Maven 3.9+
- Docker e Docker Compose

A plataforma utiliza a rede Docker externa `fluxe_shared`.

```bash
docker network create fluxe_shared
```

Se ela já existir, o comando pode ser ignorado.

### Stack completa

```bash
./scripts/up.sh
./scripts/seed.sh
./scripts/smoke.sh
```

### Apenas infraestrutura + aplicação local

```bash
docker compose up -d postgres redis rabbitmq
./mvnw spring-boot:run
```

### Testes unitários sem Docker

```bash
./mvnw test -Dtest='!*Integration*'
```

## Serviços locais

| Serviço | URL |
|---------|-----|
| API | http://localhost:8080 |
| Swagger UI | http://localhost:8080/docs |
| OpenAPI | http://localhost:8080/v3/api-docs |
| Liveness | http://localhost:8080/actuator/health/liveness |
| Readiness | http://localhost:8080/actuator/health/readiness |
| Prometheus | http://localhost:8080/actuator/prometheus |
| Grafana | http://localhost:3030 |
| RabbitMQ UI | http://localhost:15672 |

## API

A API é versionada em `/v1`.

### Tenants

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| POST | `/v1/tenants` | Criar tenant |
| GET | `/v1/tenants` | Listar tenants |
| GET | `/v1/tenants/{id}` | Consultar por ID |
| PATCH | `/v1/tenants/{id}` | Atualizar |
| DELETE | `/v1/tenants/{id}` | Soft delete |

### Policies

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| POST | `/v1/policies` | Criar política |
| GET | `/v1/policies` | Listar políticas |
| GET | `/v1/policies/{id}` | Consultar por ID |
| PATCH | `/v1/policies/{id}` | Atualizar |
| DELETE | `/v1/policies/{id}` | Soft delete |

### Feature flags

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| POST | `/v1/tenants/{tenantId}/flags` | Criar flag |
| GET | `/v1/tenants/{tenantId}/flags` | Listar |
| PATCH | `/v1/tenants/{tenantId}/flags/{name}` | Atualizar |
| DELETE | `/v1/tenants/{tenantId}/flags/{name}` | Remover |

### Auditoria

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| GET | `/v1/audit` | Consulta paginada |
| GET | `/v1/audit/export` | Exportação JSON ou CSV |

### Endpoints para consumidores

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| GET | `/v1/tenants/{id}/snapshot` | Snapshot do tenant |
| GET | `/v1/tenants/{id}/policies` | Políticas aplicáveis |
| GET | `/v1/tenants/{id}/flags` | Feature flags |

### Desenvolvimento

| Método | Endpoint | Descrição |
|--------|----------|-----------|
| POST | `/v1/dev/token` | Geração de JWT HS256 local |
| GET | `/v1/me` | Claims e correlation id |
| GET | `/healthz` | Liveness |
| GET | `/readyz` | Readiness |

## Headers

| Header | Obrigatório | Descrição |
|--------|-------------|-----------|
| `Authorization` | Sim | `Bearer <JWT>` |
| `X-Tenant-Id` | Sim* | Identificador validado contra a claim `tid` |
| `X-Correlation-Id` | Não | Gerado automaticamente quando ausente |

## Configuração

| Variável | Default | Descrição |
|----------|---------|-----------|
| `SPRING_PROFILES_ACTIVE` | `local` | Profile |
| `DB_URL` | — | URL PostgreSQL |
| `DB_USER` | — | Usuário |
| `DB_PASS` | — | Senha |
| `AUTH_MODE` | `hs256` | `hs256` ou `oidc` |
| `JWT_ISSUER` | `spring-saas-core` | Issuer |
| `JWT_SECRET` | — | Chave HS256 |
| `JWT_SECRET_PREVIOUS` | — | Chave anterior para rotação |
| `OIDC_ISSUER_URI` | — | Issuer OIDC |
| `REDIS_HOST` | `localhost` | Redis |
| `RABBITMQ_HOST` | `localhost` | RabbitMQ |
| `OUTBOX_PUBLISH_ENABLED` | `false` | Habilita publicação do outbox |
| `SERVER_PORT` | `8080` | Porta HTTP |

## Scripts

| Script | Uso |
|--------|-----|
| `./scripts/up.sh` | Sobe a stack |
| `./scripts/migrate.sh` | Verifica migrations |
| `./scripts/seed.sh` | Valida dados seed |
| `./scripts/smoke.sh` | Smoke tests |
| `./scripts/e2e-invite-user.sh` | Fluxo E2E de convite |
| `./scripts/api-export.sh` | Exporta OpenAPI |

## Observabilidade

- health e readiness pelo Actuator;
- métricas Prometheus;
- logs JSON em produção;
- MDC com `correlationId` e `tenantId`;
- tracing via OpenTelemetry OTLP.

Métricas de domínio incluem:

- `saas_tenants_created_total`
- `saas_policies_updated_total`
- `saas_flags_toggled_total`
- `saas_access_denied_total`

## Testes

```bash
# Unitários
./mvnw test -Dtest='!*Integration*'

# Integração com Testcontainers
./mvnw test -Dtest="com.union.solutions.saascore.integration.**"
```

Os testes cobrem o motor ABAC, regras de domínio, CRUD, negações e auditoria.

## Build e publicação

A imagem é publicada no GHCR pelo workflow `.github/workflows/build-push.yml` em pushes para `develop`/`master` ou por execução manual.

Detalhes em [docs/DEPLOY-GITHUB.md](docs/DEPLOY-GITHUB.md).

## Troubleshooting

| Problema | Verificação |
|----------|-------------|
| 401 | Token e issuer |
| 403 | Claims, permissões e políticas ABAC |
| Tenant mismatch | `X-Tenant-Id` versus claim `tid` |
| Aplicação não inicia | `docker compose logs app` |
| Falha de migration | Conectividade e credenciais PostgreSQL |
| Checksum Liquibase | Verificar changesets antes de recriar ambiente |

## Documentação

- [Contrato de identidade](docs/contracts/identity.md)
- [Headers HTTP](docs/contracts/headers.md)
- [Eventos Outbox](docs/contracts/events.md)
- [Compliance e auditoria](docs/compliance.md)
- [Backlog de evolução](docs/BACKLOG-EVOLUCAO.md)

## Licença

MIT — ver [LICENSE](LICENSE).
