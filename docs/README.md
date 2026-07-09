# Agendix — Reservas y Agendas para Negocios

## Documentación del Proyecto (base — extraída de Zity)

Este directorio contiene la arquitectura base de **Agendix**, producto Novex para que negocios de servicios (barberías, spas, consultorios, estudios) gestionen su agenda: catálogo de servicios, disponibilidad de staff, reservas de clientes y recordatorios automáticos.

La arquitectura técnica (multi-tenancy, capas, stack, convenciones) se extrajo de [Zity](../../zity/docs/01-arquitectura.md) porque ambos productos comparten la misma base Novex (FastAPI + Motor + Angular + GCP). El **dominio** es distinto y se adaptó desde cero: Zity controla accesos físicos con QR; Agendix administra citas y disponibilidad.

---

## Índice de Documentos

| # | Documento | Descripción |
|---|-----------|-------------|
| 01 | [Arquitectura y Multi-tenancy](./01-arquitectura.md) | Estrategia de aislamiento de datos, middleware, capas y diagrama de componentes |
| 02 | [Visión General y Actores](./02-vision-general.md) | Problema, propuesta de valor por vertical, objetivos de negocio y actores del sistema |
| 03 | [Requerimientos Funcionales](./03-requerimientos-funcionales.md) | RF de Agendamiento, Multitenancy, Recompensas/Fidelización, Notificaciones y CRM |
| 04 | [Requerimientos No Funcionales](./04-requerimientos-no-funcionales.md) | Rendimiento, escalabilidad, seguridad/privacidad, usabilidad móvil/PWA, observabilidad |
| 05 | [Historias de Usuario](./05-historias-usuario.md) | US con criterios de aceptación Dado/Cuando/Entonces y checklists, por módulo |
| 06 | [Matriz de Trazabilidad](./06-trazabilidad.md) | RF ↔ componentes de arquitectura, colecciones nuevas (módulo de recompensas) y riesgos técnicos |
| 07 | [Plan de Implementación (bajo costo)](./07-plan-implementacion.md) | Stack $0/mes con free tiers, fases de desarrollo y triggers explícitos de upgrade |

*(Los documentos 08+ — modelo de datos detallado a nivel de esquema, contrato de endpoints — se agregan a medida que avanza la implementación, siguiendo la misma numeración que usó Zity.)*

---

## Stack Tecnológico

```
Backend       : FastAPI 0.115+ · Python 3.11+ · PyJWT · Motor (async MongoDB driver)
Base de Datos : MongoDB 7+ (estrategia single-DB + tenant_id por documento)
Frontend      : Angular 20 · PrimeNG (Aura) · TypeScript 5+
Auth          : JWT (access token 15 min + refresh token 7 días)
Notificaciones: Email (SMTP, fire-and-forget) — SMS/WhatsApp evaluado en fase 2
Infra         : Docker Compose (dev) · GCP Cloud Run + Firebase Hosting (prod)
```

---

## Roles del Sistema

| Rol | Scope | Permisos clave |
|-----|-------|----------------|
| `super_admin` | Global | CRUD de tenants (negocios), métricas globales |
| `business_admin` | Tenant | CRUD de staff, servicios, horarios; ve todas las citas del negocio |
| `staff` | Tenant + propio calendario | Ve/gestiona sus propias citas, marca completada/no-show |
| `client` | Tenant | Reserva citas, ve su propio historial |

---

## Convenciones de Nomenclatura

(Idénticas a Zity, para mantener consistencia entre productos Novex)

- Colecciones MongoDB: `snake_case` (ej. `appointments`)
- Endpoints REST: `kebab-case` (ej. `/appointments/{id}/cancel`)
- Componentes Angular: `PascalCase` (ej. `BookingCalendarComponent`)
- Variables de entorno: `SCREAMING_SNAKE_CASE` (ej. `JWT_SECRET_KEY`)

---

## Estado del Proyecto

Fases redefinidas siguiendo la arquitectura de bajo costo de [07-plan-implementacion.md](./07-plan-implementacion.md) (stack ≈ $0/mes hasta que el tráfico real dispare un trigger de upgrade):

- **Fase 0:** Bootstrap de infraestructura (Atlas M0, Cloud Run, Firebase Hosting) y repos ← *en curso*
- Fase 1 (MVP): Catálogo de servicios + staff + reserva de citas (pública y privada) + validación de solapamiento
- Fase 2: Notificaciones de bajo costo (email SMTP gratuito + recordatorios vía Cloud Scheduler)
- Fase 3: CRM básico (historial, notas privadas) + Sistema de Recompensas y Fidelización
- Fase 4: Hardening de seguridad, dominio propio (`agendix.app`) y alta del primer negocio piloto real
- Fase 5: Escalamiento condicionado a métricas reales (sin fecha fija — ver tabla de triggers en 07)
