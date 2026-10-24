# Billing service (FastAPI)

Único componente que habla con Stripe. Worktime lo llama por HTTP interno; Stripe lo llama por webhook.

```
Angular ──► worktime API (.NET) ──X-Service-Token──► billing (FastAPI) ──► Stripe
                 ▲                                        │   ▲
                 └──────────X-Service-Token───────────────┘   └── webhooks (firma Stripe)
```

## Stack

Python 3.12+, **Poetry**, FastAPI + uvicorn, `stripe` (SDK oficial, `StripeClient` con métodos `*_async`), SQLAlchemy 2 async + asyncpg, Alembic, pydantic-settings, httpx (para llamar a worktime), pytest + pytest-asyncio + testcontainers[postgres].

Estructura objetivo (pocos archivos; no hay capas que no se necesiten):
```
app/
  main.py         # FastAPI app, lifespan (arranca el reconciliador), routers
  config.py       # Settings (pydantic-settings)
  db.py           # engine, session, modelos (3 tablas)
  stripe_api.py   # creación de checkout/portal, meter events, cancel+refund
  worktime.py     # cliente httpx hacia /internal/* de worktime
  routes.py       # endpoints internos + webhook
  reconciler.py   # loop de la SAGA y de la sincronización de estado
scripts/seed_stripe.py
migrations/       # Alembic
tests/
pyproject.toml
Dockerfile
```

## Configuración (env)

| Variable | Ejemplo | Uso |
|---|---|---|
| `DATABASE_URL` | `postgresql+asyncpg://worktime:worktime@postgres/billing` | BD propia en la misma instancia |
| `STRIPE_SECRET_KEY` | `sk_test_...` | Solo modo test |
| `STRIPE_WEBHOOK_SECRET` | `whsec_...` | Lo imprime `stripe listen` |
| `SERVICE_TOKEN` | secreto compartido | Se valida en `/internal/*` y se envía a worktime. Comparar con `hmac.compare_digest` |
| `WORKTIME_URL` | `http://nginx` | Base para `/internal/*` de worktime |
| `INCLUDED_SEATS` | `10` | Debe coincidir con el `up_to` del tramo 1 del precio medido |

## Catálogo Stripe (`scripts/seed_stripe.py`, idempotente por `lookup_key`)

- **Product** `Worktime Pro`.
- **Price base**: `lookup_key=worktime_pro_base`, 49 USD/mes, `recurring.usage_type=licensed`.
- **Meter**: `event_name=active_seats`, payload con `stripe_customer_id` y `value`, agregación **`last`** (ver plan B).
- **Price de seats**: `lookup_key=worktime_pro_seats`, `recurring.usage_type=metered` ligado al meter, `billing_scheme=tiered`, `tiers_mode=graduated`:
  - tramo 1: `up_to=10`, `unit_amount=0` ← **soft cap incluido**
  - tramo 2: `up_to=inf`, `unit_amount=400` (4 USD por seat extra)

La suscripción lleva **dos items**: la base (cantidad 1) y los seats (medido, sin cantidad). Se separan en dos precios porque un precio medido con `flat_amount` en el tramo 1 tiene un comportamiento ambiguo cuando el uso es 0.

El código referencia los precios por `lookup_key`, nunca por ID, para que el script pueda recrear el catálogo en otra cuenta sin cambiar configuración.

### Meter: plan A y plan B
- **A (preferido)**: worktime envía cada hora el **total acumulado** de seats activos del mes y el meter agrega con `last`. El valor final del periodo es el que se factura.
- **B (si `last` no existe)**: worktime envía **un evento por trabajador la primera vez que ficha en el mes**, con `value=1` y agregación `sum`. Billing deduplica con `UNIQUE (tenant_id, worker_id, month)` antes de enviarlo a Stripe, porque el `identifier` de Stripe solo deduplica durante una ventana corta.

## Base de datos (3 tablas)

```sql
billing_tenants (
  tenant_id            uuid PRIMARY KEY,         -- id generado por worktime
  email                text NOT NULL,
  stripe_customer_id   text UNIQUE,
  stripe_subscription_id text UNIQUE,
  checkout_session_id  text UNIQUE,
  saga_state           text NOT NULL,            -- ver máquina de estados
  subscription_status  text,                     -- último status leído de Stripe
  status_dirty         boolean NOT NULL DEFAULT false,  -- falta notificar a worktime
  attempts             int NOT NULL DEFAULT 0,
  next_attempt_at      timestamptz NOT NULL DEFAULT now(),
  last_error           text,
  updated_at           timestamptz NOT NULL DEFAULT now()
)
stripe_events (id text PRIMARY KEY, type text NOT NULL, received_at timestamptz NOT NULL DEFAULT now())
usage_reports (tenant_id uuid, hour timestamptz, active_seats int NOT NULL, PRIMARY KEY (tenant_id, hour))
```

## API interna (todas exigen `X-Service-Token`)

| Método | Ruta | Body → respuesta | Notas |
|---|---|---|---|
| POST | `/checkout-sessions` | `{tenant_id, tenant_name, email, success_url, cancel_url}` → `{url}` | Si no existe, crea el Customer con `idempotency_key=f"customer:{tenant_id}"` y `metadata.tenant_id`. Checkout `mode=subscription`, `client_reference_id=tenant_id`, `subscription_data.metadata.tenant_id`, `billing_cycle_anchor` = día 1 del mes siguiente UTC. Inserta `billing_tenants` en `PENDING_PAYMENT` |
| POST | `/portal-sessions` | `{tenant_id, return_url}` → `{url}` | Customer Portal |
| POST | `/usage` | `{tenant_id, active_seats, as_of}` → 202 | Ignora tenants no activos. Upsert en `usage_reports` + `MeterEvent` con `identifier=f"{tenant_id}:{hour}"` |
| GET | `/tenants/{id}/billing` | → `{subscription_status, included_seats, active_seats, current_period_end, upcoming_total, upcoming_lines[]}` | `upcoming_*` viene de la vista previa de la próxima factura (`invoices.create_preview`) |

## Webhook `POST /webhooks/stripe` (público, sin token)

1. `stripe.Webhook.construct_event(raw_body, sig_header, STRIPE_WEBHOOK_SECRET)` con el body **crudo** (`await request.body()`). Si la firma no es válida → 400.
2. En **la misma transacción**: `INSERT INTO stripe_events ... ON CONFLICT DO NOTHING`. Si no se insertó ninguna fila, el evento es un duplicado → 200 sin hacer nada.
3. Actualizar el estado y devolver **200 rápido**. El trabajo lento lo hace el reconciliador, nunca el webhook.

| Evento | Acción |
|---|---|
| `checkout.session.completed` | `saga_state = PAID`, guardar `stripe_subscription_id` |
| `checkout.session.expired` | `saga_state = EXPIRED` (el reconciliador avisa a worktime para borrar el tenant pendiente) |
| `customer.subscription.updated` / `.deleted` | **Volver a leer la suscripción desde Stripe** (no confiar en el payload, porque los eventos pueden llegar desordenados), guardar `subscription_status` y poner `status_dirty = true` |
| `invoice.paid` / `invoice.payment_failed` | Igual que el anterior (el status de la suscripción cambia en consecuencia) |

## SAGA de alta de tenant

```
worktime: Tenant(PendingPayment) + admin inactivo
        │ POST /checkout-sessions
        ▼
PENDING_PAYMENT ──checkout.session.expired──► EXPIRED ──► DELETE worktime tenant ──► DONE
        │ checkout.session.completed
        ▼
      PAID ──reconciler: POST worktime /internal/tenants/{id}/activate──► ACTIVE
        │ fallo transitorio (5xx/timeout): attempts++, backoff exponencial, máx. 5
        │ fallo permanente (404/409) o attempts agotados
        ▼
  COMPENSATING ──cancelar suscripción + reembolsar el último pago──► DELETE worktime tenant ──► COMPENSATED
```

- **Reconciliador**: tarea asyncio que arranca en el `lifespan`. Cada 5 s ejecuta `SELECT ... WHERE saga_state IN (...) OR status_dirty AND next_attempt_at <= now() FOR UPDATE SKIP LOCKED LIMIT 10`. Es seguro con varias réplicas: cada fila la procesa una sola.
- Cada llamada a Stripe dentro de la compensación usa `idempotency_key` (`cancel:{tenant_id}`, `refund:{tenant_id}`), así que repetirla es inofensivo.
- Todas las llamadas a worktime son **idempotentes**: activar un tenant activo → 200; borrar uno que no existe → 204.
- `status_dirty` → `POST worktime /internal/tenants/{id}/status`. Si funciona, se pone a `false`; si falla, se reintenta con el mismo backoff.
- Si un tenant pendiente se borra en worktime por antigüedad y el pago llega tarde, `activate` responde 404 → compensación → reembolso. La carrera queda cubierta.

## Tests mínimos

- Webhook: firma inválida → 400; el mismo `event.id` dos veces → un solo efecto.
- Reconciliador (Postgres real con testcontainers, Stripe y worktime sustituidos con `app.dependency_overrides` o respx):
  - `PAID` + worktime OK → `ACTIVE`.
  - `PAID` + 5xx → reintento; tras 5 fallos → compensación (se llaman cancel y refund).
  - Dos reconciliadores concurrentes sobre la misma fila → un solo `activate`.

## Demo local

```bash
stripe listen --forward-to localhost:8000/webhooks/stripe   # copia el whsec_ a STRIPE_WEBHOOK_SECRET
stripe trigger checkout.session.completed                   # humo
stripe events resend evt_...                                # probar idempotencia
```
Para el excedente: crear el Customer con un **Test Clock** (`test_helpers.test_clocks`), fichar con 12 trabajadores y avanzar el reloj un mes. Verificar en la Fase 4 si Checkout acepta customers con test clock; si no, crear esa suscripción de demo directamente por API.

Tarjetas de test: `4242 4242 4242 4242` (OK), `4000 0025 0000 3155` (3DS), `4000 0000 0000 0341` (se puede asociar pero el cobro falla).
