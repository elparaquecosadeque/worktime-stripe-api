# Handoff: Worktime multitenant + billing con Stripe (FastAPI)

> Punto de entrada para la sesión de implementación. Leer en este orden:
> 1. Este archivo (contexto, decisiones, fases)
> 2. [`docs/billing-service.md`](docs/billing-service.md): el servicio FastAPI (este repo)
> 3. [`docs/worktime-changes.md`](docs/worktime-changes.md): los cambios en el repo `worktime` (.NET)

## Objetivo

Proyecto de portafolio/CV: convertir **worktime** (ASP.NET Core 8 + Angular + Postgres + Redis, repo `elparaquecosadeque/worktime`) en un SaaS **multitenant** que se cobra con **Stripe (sandbox)** usando suscripción + facturación por uso. Toda la integración con Stripe vive en un **servicio de facturación separado en FastAPI** (este repo).

Lo que el proyecto debe demostrar en una entrevista:
- Suscripción con **soft cap** y **cobro por uso** por encima del cap (Stripe Billing Meters).
- **Webhooks correctos**: firma verificada, idempotencia por `event.id` y resistencia a eventos desordenados.
- **SAGA con compensación** entre dos servicios (aprovisionamiento de tenant ↔ pago).
- Concurrencia segura en el reconciliador (`FOR UPDATE SKIP LOCKED`), en la línea del caso de doble aprobación que worktime ya resuelve.

## Decisiones tomadas (sesión de grilling, 2026-10-24)

| # | Decisión | Motivo |
|---|---|---|
| 1 | Dominio = worktime multitenant, no una app de juguete | Reutiliza una app ya cuidada y da una historia de producto real |
| 2 | **1 tenant = 1 Stripe Customer = 1 Subscription** | Los Meters agregan por *customer*: si un customer tuviera varios tenants, se mezclaría el uso. Una "cuenta con varios tenants" queda fuera de v1 |
| 3 | Plan Pro: **base fija + 10 seats activos incluidos + precio por seat activo extra** | Soft cap: no bloquea, cobra el excedente |
| 4 | **Seat activo** = trabajador con ≥1 fichaje en el periodo | Cobra lo usado, no lo dado de alta |
| 5 | Periodo = **mes calendario UTC** (`billing_cycle_anchor` al día 1, con prorrateo) | Worktime puede contar seats sin conocer las fechas de Stripe |
| 6 | Arquitectura: **Angular → worktime (.NET) → billing (FastAPI) → Stripe**. Angular nunca habla con billing; solo billing habla con Stripe | Frontera clara y un único dueño de los secretos de Stripe |
| 7 | Comunicación entre servicios: **HTTP síncrono + token compartido** (`X-Service-Token`) en los dos sentidos | Lo más simple que funciona. mTLS o colas, más adelante |
| 8 | SAGA = **alta de tenant**: tenant `PendingPayment` → Checkout → webhook → activar en worktime; si falla, **cancelar la suscripción y reembolsar** | Compensación real contra un proveedor de pagos |
| 9 | Billing usa **Postgres** (BD `billing` en la misma instancia que worktime) | Bloqueos de fila reales y `UNIQUE` para idempotencia. Sin infraestructura nueva |
| 10 | Captura manual (`capture_method=manual`) **fuera de v1**, como fase extra | No aplica a suscripciones; solo encaja con una cuota de alta opcional |

## Decisiones abiertas (resolver al empezar; mi recomendación en negrita)

1. **Nombre del repo**: renombrar `stripe-sample` → **`worktime-billing`**.
2. **Moneda**: **USD** (precios en Stripe: base 49 USD, 4 USD por seat extra). Alternativas: EUR o PEN.
3. **Unicidad del email**: **global** (el login no necesita elegir tenant). Alternativa: único por tenant con login `slug + email`.
4. **Trial**: **ninguno en v1** (los trials con precios medidos complican la demo).

## Fases y criterios de aceptación

Cada fase se cierra con su criterio verificado. No avanzar con una fase a medias.

### Fase 0: Preparación
- Cuenta Stripe en modo sandbox, `STRIPE_SECRET_KEY` (`sk_test_...`).
- Stripe CLI: `stripe login`, o la imagen `stripe/stripe-cli` en compose.
- **Verificar en docs.stripe.com** que los Billing Meters soportan la agregación `last` (para el snapshot de seats). Si no la soportan, usar el plan B descrito en `docs/billing-service.md` §Meter.
- Script `scripts/seed_stripe.py` que crea el producto, los precios y el meter de forma idempotente (por `lookup_key`).
- ✅ El catálogo aparece en el Dashboard de test y el script se puede ejecutar dos veces sin duplicar nada.

### Fase 1: Worktime multitenant (todavía sin billing)
- Ver `docs/worktime-changes.md` §1.
- ✅ Test de integración: un usuario del tenant A no ve usuarios, registros, equipo ni eventos SignalR del tenant B.
- ✅ Los 47 tests existentes siguen en verde y el seeder crea 2 tenants demo.

### Fase 2: Billing service + alta de tenant (SAGA)
- Esqueleto FastAPI, Checkout, webhook idempotente, reconciliador y endpoints `/internal/*` en worktime.
- ✅ Alta → pagar con `4242 4242 4242 4242` → tenant `Active` → el admin inicia sesión.
- ✅ Con worktime caído durante la activación, el reconciliador reintenta y activa cuando worktime vuelve.
- ✅ Si la activación falla de forma permanente (forzar un 404), la suscripción se cancela, aparece el reembolso en el Dashboard y el tenant se borra.
- ✅ Reenviar el mismo evento (`stripe events resend evt_...`) no produce efectos dobles.

### Fase 3: Estado de la suscripción y dunning
- Customer Portal, sincronización de estado y avisos en Angular.
- ✅ Cambiar la tarjeta en el Portal por `4000 0000 0000 0341` y forzar un cobro → tenant `PastDue` → aviso al admin.
- ✅ Cancelar la suscripción en el Portal → tenant `Canceled` al terminar el periodo → solo el admin puede entrar y únicamente ve Facturación.

### Fase 4: Cobro por uso (soft cap)
- Reportador de uso en worktime → billing → Meter de Stripe.
- ✅ Con 12 trabajadores fichando, la página Facturación muestra "12 / 10 seats" y una vista previa de la factura con 2 seats extra.
- ✅ Con un **Test Clock** avanzado un mes, se genera una factura con la línea de excedente correcta.

### Fase 5 (extra, opcional)
- Cuota de alta con **captura manual**: autorizar → aprovisionar → capturar, o cancelar si falla.
- Postgres **RLS** como segunda barrera de aislamiento entre tenants.
- **Outbox** en worktime para las llamadas a billing, en lugar de HTTP directo.

## Cómo arrancar la próxima sesión

Clonar este repo y `worktime` como carpetas hermanas:
```
repos/
  worktime/
  worktime-billing/   (este repo)
```
Prompt sugerido para Claude Code, abierto en la carpeta `repos/`:

> Lee `worktime-billing/HANDOFF.md` y los dos documentos que enlaza. Resuelve conmigo las decisiones abiertas y empieza por la Fase 0. Respeta `worktime/CODE_STYLE.md` en el lado .NET.
