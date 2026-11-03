# Events — API Gateway

> The API gateway **neither publishes nor consumes events**. Its responsibility is
> infrastructure (routing, token validation, rate limiting), not business.

---

## Events this service PUBLISHES

**None.** The gateway has no RabbitMQ connection and no outbox.

---

## Events this service CONSUMES

**None.**

---

## Delivery guarantees

Not applicable.

---

## Error handling

Not applicable to events. For HTTP, the answers the gateway produces itself (401, 404, 429, 503,
504) use the shared error shape (`timestamp`, `status`, `error`, `message`, `path`).

---

## Correlations

- Events of the services behind the gateway → `09-microservices/services/02-security-service/events.md`,
  `03-device-service/events.md`, `04-measurement-service/events.md`
- Platform event transport → `05-architecture/decisions/records/ADR-008-rabbitmq-transactional-outbox.md`
