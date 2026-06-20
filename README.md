# sc-order-service (Backend) - Ecossistema Hubinity - Planned

> Parte integrante do ecossistema distribuído Hubinity.
> ⚠️ **Status atual: Planned** — código de implementação ainda não foi escrito. Este README descreve o papel arquitetural pretendido conforme PRD seção 4 e roadmap em `docs/phases/`.

---

## 💻 Visão Geral

- **O que faz:** Backend da Star Coffee para pedidos originados no totem self-service. Implementa o fluxo completo `IDLE → CATEGORIES → PRODUCTS → REVIEW → IDENTIFY → PAYMENT_METHOD → PAYMENT (PIX ou dinheiro) → PRINTING → IDLE`. Persiste o pedido com **snapshot** do produto (preço, nome, SKU no momento da venda — sem FK direto ao catálogo), garantindo histórico imutável.
- **Problema que resolve:** tira a fila do balcão da cafeteria — o cliente pede e paga sozinho — e elimina a digitação dupla da venda no caixa publicando `OrderPaid` automaticamente após confirmação de pagamento.
- **Posicionamento no Ecossistema:** consome catálogo do `hb-catalog-service` (REST + cache local + invalidação por evento), originando vendas e publicando para o `hb-cashier-service`. É o único serviço que fala com gateway de pagamento (InfinitePay) e com hardware físico (impressora térmica).

## 🏗️ Papel na Arquitetura

- **Tipo de Componente:** Microsserviço Spring Boot, database-per-service (Postgres dedicado no Supabase), publicador via Transactional Outbox.
- **Responsabilidades Principais (planejadas):**
  - CRUD de `Order` com máquina de estados (`DRAFT`/`AWAITING_PAYMENT`/`PAID`/`PRINTED`/`CANCELLED`).
  - Integração PIX com InfinitePay (QR Code dinâmico + webhook HMAC).
  - Confirmação manual de pagamento em dinheiro (operador no balcão).
  - Saga de reserva de estoque contra o `hb-catalog-service` (reserve → commit no `OrderPaid` ou release no cancelamento).
  - Driver ESC-POS para impressão da comanda na térmica USB/rede.
  - Publicação de `OrderPaid` e `OrderCancelled` via outbox transacional.
- **Limites e Fronteiras (Boundaries):** não é dono do catálogo (apenas proxy cacheado), não escreve no caixa (publica evento e desacopla), não autentica usuários humanos pelo formulário (usa token de device kiosk).

## 🔗 Dependências e Comunicação (Planejadas)

### Serviços Internos da Hubinity

- **`hb-catalog-service`** — lê catálogo via REST (cache local Caffeine + ETag); chama `POST /api/v1/stock/reserve` e `/release` síncronos; consome eventos `catalog.events.ProductUpdated`, `StockChanged`, `PriceChanged`, `ProductDeactivated` na queue `sc-order.catalog` para invalidação de cache.
- **`hb-cashier-service`** — destinatário (indireto, via broker) dos eventos `order.events.OrderPaid` e `order.events.OrderCancelled`.
- **`platform-iam` (Keycloak)** — valida JWT da realm `star-coffee`; suporta token de device de longa duração (refresh token ~30 dias) emitido para cada totem cadastrado.
- **`platform-shared-contracts`** — depende de `contracts-order`, `contracts-catalog` (proxy) e `contracts-events`.

### Infraestrutura e Serviços Externos

- **Supabase** — projeto Postgres dedicado `sc-order`.
- **CloudAMQP** — RabbitMQ compartilhado.
- **InfinitePay** — gateway PIX (sandbox em staging); webhook valida assinatura HMAC.
- **Impressora térmica** — USB direta ou via rede, protocolo ESC-POS.
- **Railway Hobby** — host (recomendado para evitar hibernação no horário de pico).

## 🛠️ Tecnologias e Ferramentas (Stack Prevista)

| Camada | Tecnologia | Versão |
| :--- | :--- | :--- |
| Linguagem | Java | 21 (LTS) |
| Build | Maven | 3.9+ |
| Framework | Spring Boot | 4.1 |
| Módulos Spring | Web, Data JPA, Flyway, AMQP, Security, Resource Server, Cache | — |
| Cache local (catálogo) | Caffeine | última estável |
| Mapper | MapStruct | 1.6 |
| Resiliência (REST client tipado) | Resilience4j | última estável |
| Banco | PostgreSQL (Supabase) | 15+ |
| Broker | RabbitMQ (CloudAMQP) | 3.x |
| Driver impressora | ESC-POS (lib Java) | a definir na 3.14 |
| Testes | JUnit 5 + Testcontainers (Postgres + RabbitMQ) | última estável |
| Container | Docker (multi-stage) | — |

## 📐 Padrões de Projeto e Arquitetura do Código (Previstos)

- **Estilo Arquitetural:** Hexagonal com adaptadores explícitos para gateway de pagamento (`PaymentGateway` interface, implementação InfinitePay swappable) e impressora (`ReceiptPrinter` interface).
- **Padrões Relevantes:**
  - **Transactional Outbox** — `OrderPaid` e `OrderCancelled` gravados na mesma transação JPA da escrita de negócio; dispatcher `@Scheduled` publica no broker (at-least-once).
  - **Saga (coreografada)** — reserva síncrona de estoque na criação; commit/release via evento após resolução do pagamento.
  - **Snapshot de domínio** — `OrderItem` guarda `productSku`/`productName`/`unitPrice` no momento da venda; histórico fica imutável mesmo se o produto for editado depois no catálogo.
  - **State Machine** — transições de `Order.status` controladas (TTL de 5min em `DRAFT`).
  - **Idempotency-Key** em todas as mutações públicas; webhook PIX verifica `(externalId, signature)`.

## 🗺️ Roadmap & Posição no Board

- **Fase do PRD:** Fase 3 — Totem (PRD seção 9).
- **Tasks no board:**
  - `3.1` — Bootstrap.
  - `3.2` — Flyway: `order`, `order_item`, `payment`, `outbox`, `processed_messages`.
  - `3.3` — Cliente REST tipado pro catálogo (Resilience4j + Caffeine).
  - `3.4` — Endpoints criar/ver/cancelar + saga de reserva.
  - `3.5` — Integração PIX (InfinitePay sandbox) + webhook HMAC.
  - `3.6` — Fluxo dinheiro + confirmação manual.
  - `3.7` — Outbox dispatcher (`@Scheduled` → RabbitMQ).
  - `3.8` — Consumer no caixa (handler `OrderPaid`).
  - `3.14` — Driver ESC-POS.
  - `3.16` — Deploy Railway + setup totem físico.
- **Dependências bloqueadoras:** `hb-catalog-service` em staging com endpoint de reserva (Fase 1.7) e CRUD de produto (Fase 1.5–1.6) funcionando. `hb-cashier-service` precisa ter consumer skeleton (Fase 2.5) pronto antes da 3.8.

## ⚙️ Variáveis de Ambiente (Previstas)

```bash
# App
SPRING_PROFILES_ACTIVE=staging
SERVER_PORT=8083

# Banco — Supabase (projeto sc-order)
SPRING_DATASOURCE_URL=jdbc:postgresql://<host>.pooler.supabase.com:6543/postgres?sslmode=require
SPRING_DATASOURCE_USERNAME=
SPRING_DATASOURCE_PASSWORD=

# Broker
SPRING_RABBITMQ_HOST=
SPRING_RABBITMQ_PORT=5671
SPRING_RABBITMQ_SSL_ENABLED=true
SPRING_RABBITMQ_USERNAME=
SPRING_RABBITMQ_PASSWORD=
SPRING_RABBITMQ_VIRTUAL_HOST=

# IAM — Keycloak (realm star-coffee)
KEYCLOAK_ISSUER_URI=https://iam.hubinity.app/realms/star-coffee

# Catálogo (REST client tipado)
CATALOG_BASE_URL=https://hb-catalog-service.up.railway.app
CATALOG_CLIENT_ID=sc-order-svc
CATALOG_CLIENT_SECRET=

# InfinitePay
INFINITEPAY_BASE_URL=https://api.sandbox.infinitepay.io
INFINITEPAY_MERCHANT_ID=
INFINITEPAY_API_KEY=
INFINITEPAY_WEBHOOK_HMAC_SECRET=

# Impressora (totem físico — vazio em staging)
PRINTER_TYPE=NETWORK   # ou USB
PRINTER_HOST=192.168.1.50
PRINTER_PORT=9100
```

## 🚀 Como Será Executado (Quando Implementado)

### Pré-requisitos

- JDK 21, Maven 3.9+
- Stack local do `platform-infra` rodando
- `hb-catalog-service` acessível (local ou staging)
- Credenciais sandbox da InfinitePay

### Execução (Será disponível após bootstrap da Fase 3)

```bash
# Subir local
SPRING_PROFILES_ACTIVE=local mvn spring-boot:run

# Testes (Testcontainers para Postgres + RabbitMQ; PIX e impressora mockados)
mvn -B verify

# Build container
docker build -t ghcr.io/hubinity/sc-order-service:dev .
```
