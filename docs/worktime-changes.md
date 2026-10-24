# Cambios en worktime (.NET)

Estado actual (revisado el 2026-10-24): **single-tenant**. No hay concepto de organización; `User` no tiene `TenantId`, el JWT lleva `sub/name/email/role/stamp/perm/supervisor_id` y las claves de Redis y los grupos de SignalR no tienen prefijo de tenant. Seguir `CODE_STYLE.md` y la estructura por módulos (`AddXModule()`).

## 1. Multitenancy (Fase 1, sin billing)

**Estrategia: esquema compartido + `tenant_id` en todas las tablas + global query filter de EF Core.**

- Nuevo módulo `Tenants`: agregado `Tenant { Id, Name, Slug, Status, IncludedSeats, CreatedAt }` con `Status ∈ PendingPayment | Active | PastDue | Canceled`.
- `tenant_id uuid NOT NULL` en `users`, `punch_sessions`, `work_logs`, `work_log_events`, `assignment_requests` y `role_permissions` (la matriz de permisos pasa a ser por tenant). Se desnormaliza a propósito para que los filtros y los índices sean locales a cada tabla.
- `ITenantContext` (scoped), que se resuelve desde el claim `tenant_id` del JWT. `HasQueryFilter(e => e.TenantId == tenant.Id)` en cada configuración y `TenantId` asignado al crear entidades (en el agregado o en el `UnitOfWork`).
- Índices: los `UNIQUE` parciales existentes siguen siendo por `worker_id` (el worker ya pertenece a un tenant). El email es único globalmente si se mantiene la decisión abierta nº 3.
- **JWT**: nuevo claim `tenant_id` en `WorktimeClaims`, y `Actor` incorpora `TenantId`.
- **Redis**: prefijo `t:{tenantId}:` en las claves de caché del resumen de equipo y de presencia (`Infrastructure/Redis/Keys.cs`). Las claves de stamp siguen siendo por usuario.
- **SignalR**: los grupos de `RealtimeGroups` incluyen el tenant.
- **Seeder**: dos tenants `Active` (por ejemplo, "Demo" con los usuarios actuales y "Acme" con unos pocos) para poder demostrar el aislamiento.
- **Test de integración** `TenantIsolationTests`: con el token de A, listar usuarios, la bandeja, el equipo y `GET` por id de un recurso de B → vacío o 404.

## 2. Alta de tenant (Fase 2)

- `POST /api/signup` (público, con rate limit) `{orgName, adminName, email, password}`:
  1. Transacción: `Tenant(PendingPayment)` + admin con `IsActive=false`.
  2. `POST billing /checkout-sessions` → devuelve `{checkoutUrl}` a Angular, que redirige.
  - El password nunca sale de worktime. Billing solo recibe `tenant_id`, el nombre y el email.
- Página Angular `/signup` y una `/signup/done` (`success_url`) que hace polling de `GET /api/signup/{tenantId}/status` hasta `Active`, y luego manda al login.
- `BackgroundService` `PendingTenantSweep`: borra tenants `PendingPayment` con más de 24 h (lo máximo que dura un Checkout). Si el pago llega después, billing compensa con un reembolso.

## 3. Endpoints internos `/internal/*` (Fases 2–3)

Protegidos por el header `X-Service-Token` (comparación en tiempo constante) y **no publicados por nginx** hacia fuera. Todos son idempotentes.

| Método | Ruta | Efecto |
|---|---|---|
| POST | `/internal/tenants/{id}/activate` | `PendingPayment → Active` y activa al admin. Si ya está `Active` → 200. Si no existe → 404 (billing compensa) |
| POST | `/internal/tenants/{id}/status` | `{subscriptionStatus}` → mapear: `active/trialing → Active`, `past_due → PastDue`, `canceled/unpaid → Canceled` |
| DELETE | `/internal/tenants/{id}` | Solo si está `PendingPayment`; si no existe → 204 |

## 4. Efectos del estado del tenant (Fase 3)

- `PastDue`: todo funciona; aviso al admin ("pago pendiente, actualiza tu tarjeta").
- `Canceled`: el login solo se permite al admin, y solo ve Facturación. Se aplica en `LoginCommand` y en un filtro o policy de la API.
- Página Angular **Facturación** (solo admin): datos de `GET /api/billing`, que hace de proxy a `billing GET /tenants/{id}/billing`, y botón "Gestionar facturación" → `POST /api/billing/portal` → redirige al Customer Portal.

## 5. Reportador de uso (Fase 4)

- `BackgroundService` `UsageReporter`, cada `Billing:UsageReportInterval` (por defecto `01:00:00`; `00:01:00` en la demo). Para cada tenant `Active` o `PastDue`:
  ```sql
  SELECT count(DISTINCT worker_id) FROM punch_sessions
  WHERE tenant_id = @t AND started_at >= date_trunc('month', now() AT TIME ZONE 'UTC')
  ```
  → `POST billing /usage {tenant_id, active_seats, as_of}`. Si falla, se registra en el log y se reintenta en el siguiente ciclo (el valor es acumulado, así que no se pierde nada).
- Coordinar varias réplicas con un lock de Redis (como `LockKeys` en `BackgroundJobs`) para no enviar el mismo reporte dos veces. Aunque se duplicara, sería inofensivo: billing hace upsert por hora.
- Aviso de soft cap para el admin cuando `activeSeats > IncludedSeats`: "Has superado los 10 seats incluidos: cada seat activo extra cuesta 4 USD este mes".
- Si se elige el plan B del meter (ver `billing-service.md`), en lugar de lo anterior se emite un evento en el **primer fichaje del mes de cada trabajador** desde `PunchInHandler`, despachado después del commit, como los eventos de dominio actuales.

## 6. Compose

Añadir al `docker-compose.yml` de worktime:
```yaml
  billing:
    build: { context: ${BILLING_CONTEXT:-../worktime-billing} }
    environment:
      DATABASE_URL: postgresql+asyncpg://worktime:${POSTGRES_PASSWORD:-worktime}@postgres/billing
      STRIPE_SECRET_KEY: ${STRIPE_SECRET_KEY}
      STRIPE_WEBHOOK_SECRET: ${STRIPE_WEBHOOK_SECRET}
      SERVICE_TOKEN: ${SERVICE_TOKEN:-dev-service-token}
      WORKTIME_URL: http://nginx
    depends_on: { postgres: { condition: service_healthy } }

  stripe-cli:
    image: stripe/stripe-cli
    command: ["listen", "--forward-to", "billing:8000/webhooks/stripe"]
    environment: { STRIPE_API_KEY: "${STRIPE_SECRET_KEY}" }
```
- La BD `billing` se crea con un script en `/docker-entrypoint-initdb.d` (en un volumen ya existente hay que crearla a mano con `CREATE DATABASE billing`).
- Las réplicas de la API reciben `Billing__Url=http://billing:8000` y `Billing__ServiceToken`.
- Para el servidor con Jenkins (`docker-compose.server.yml`) queda pendiente decidir: fuera de v1.
