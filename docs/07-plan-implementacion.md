# 07 — Plan de Implementación (Arquitectura de Bajo Costo)

> Objetivo: llegar a un MVP en producción con **costo de infraestructura ≈ $0/mes** mientras el negocio no tiene tráfico relevante, sin comprometer la arquitectura ya decidida en [01-arquitectura.md](./01-arquitectura.md) (single DB + `tenant_id`, GCP Cloud Run + Firebase Hosting). El plan avanza por fases; cada fase termina en un estado desplegable y usable, no en código a medio terminar.

---

## 7.1 Principios del enfoque de bajo costo

1. **Todo serverless / scale-to-zero.** Nada corre (ni cobra) cuando nadie lo usa. Cloud Run y Firebase Hosting ya cumplen esto por defecto; se evita cualquier componente con costo fijo mensual (VM siempre encendida, cluster dedicado, Redis administrado, etc.).
2. **Un solo cluster Mongo compartido (ya decidido).** El patrón single-DB + `tenant_id` de [01-arquitectura.md §1.1](./01-arquitectura.md#11-decisión-de-arquitectura-de-tenancy) encaja perfecto con el tier gratuito de MongoDB Atlas (M0): un negocio nuevo no implica un cluster nuevo.
3. **Diferir todo lo que tenga costo variable por uso** (SMS/WhatsApp, email transaccional de alto volumen) a fases posteriores, y mientras tanto usar el equivalente gratuito de menor escala (SMTP gratuito, Cloud Scheduler free tier).
4. **No pagar por observabilidad/CI dedicada.** Logs nativos de Cloud Run + GitHub Actions (minutos gratuitos) son suficientes hasta que haya ingresos que justifiquen herramientas pagas.
5. **Definir triggers explícitos de upgrade**, no fechas. Se sube de tier cuando se toca un límite real (ver tabla 7.2), nunca por anticipación especulativa.

---

## 7.2 Stack de bajo costo por componente

| Componente | Elección MVP ($0) | Límite del free tier | Trigger de upgrade |
|---|---|---|---|
| Base de datos | **MongoDB Atlas M0** (shared, 512 MB) | 512 MB almacenamiento, conexiones limitadas, sin backups automáticos | Acercarse a 400 MB usados, o necesitar backups point-in-time → subir a M10 (~$9-60/mes según región) |
| Backend | **Cloud Run** (contenedor FastAPI), min instances = 0 | 2M requests/mes, 360.000 GB-seg, 180.000 vCPU-seg gratis/mes | Tráfico sostenido que agote el free tier, o necesidad de `min_instances ≥ 1` para evitar cold start en horario pico |
| Frontend | **Firebase Hosting** (Spark plan) | 10 GB almacenamiento, 360 MB/día de transferencia | Transferencia diaria sostenida cerca del límite → plan Blaze (pago por uso, sigue siendo barato) |
| Email transaccional | **SMTP gratuito** (Brevo/Sendinblue free: 300 emails/día, o Gmail SMTP para desarrollo) | 300 emails/día (Brevo) | Superar 300 emails/día reales → plan pago de Brevo/SendGrid (~$15-25/mes) |
| Recordatorios programados | **Cloud Scheduler** (3 jobs gratis/mes) → invoca endpoint interno de Cloud Run | 3 jobs gratis por cuenta de facturación | Necesitar más de 3 jobs distintos → consolidar en 1 job "orquestador" antes de pagar por más |
| SMS / WhatsApp | **Diferido a Fase 4** (no se implementa en MVP) | — | Cuando el negocio piloto lo pida explícitamente y esté dispuesto a absorber costo variable (Twilio/similar, RF-NOT-006) |
| CI/CD | **GitHub Actions** (repo privado: 2.000 min/mes gratis) | 2.000 min/mes | Build/test muy lento o múltiples pipelines → optimizar cache antes de pagar |
| Dominio propio | Subdominio gratuito de Firebase Hosting (`agendix.web.app`) hasta tener negocio piloto confirmado | — | Confirmar primer negocio piloto → comprar dominio (`agendix.app`, costo anual bajo, único gasto real del MVP) |
| Monitoreo | Logs de Cloud Run + panel de Atlas (incluidos) | — | Necesidad de alertas proactivas → recién ahí evaluar herramienta paga |

**Costo estimado Fase 0-3 (MVP sin tráfico real): $0/mes** (excepto, opcionalmente, el dominio propio si se compra antes de tener el primer cliente).

---

## 7.3 Fases de implementación

### Fase 0 — Bootstrap de infraestructura y repos (0.5-1 semana)

- [ ] Crear cluster **MongoDB Atlas M0** (`agendix_db`), whitelist de IP abierta a Cloud Run (0.0.0.0/0 + usuario/password fuerte, dado que Atlas M0 no soporta VPC peering).
- [ ] Crear proyecto GCP, habilitar Cloud Run + Artifact Registry, y **configurar alerta de presupuesto** (ej. $5) para detectar cualquier desvío del free tier de inmediato.
- [ ] Crear proyecto Firebase (Spark plan) para Hosting.
- [ ] Bootstrap del backend siguiendo el árbol de [01-arquitectura.md §1.6](./01-arquitectura.md#16-estructura-de-directorios-del-backend) + extensiones de [06-trazabilidad.md §6.2](./06-trazabilidad.md#62-componentes-nuevos-requeridos-no-listados-en-el-árbol-original-de-backendagendix).
- [ ] Bootstrap del frontend Angular 20 con `NovexAgendixPreset` ([01-arquitectura.md §1.7](./01-arquitectura.md#17-frontend-angular-20--primeng)).
- [ ] `docker-compose.yml` local (backend + Mongo local para desarrollo, **no** contra Atlas, para no gastar conexiones del free tier en dev).
- [ ] `release.py` apuntando al proyecto GCP/Firebase reales; primer despliegue "hola mundo" para validar el pipeline completo antes de escribir lógica de negocio.
- [ ] GitHub Actions: lint + `pytest` + `tsc --noEmit` en cada PR.

**Criterio de salida:** un `GET /health` responde en producción desde Cloud Run, y el frontend vacío carga desde Firebase Hosting.

---

### Fase 1 — MVP de Agendamiento (2-3 semanas)

Implementa el núcleo de valor: RF del módulo `AGE` + `TEN` básicos ([03-requerimientos-funcionales.md §3.1-3.2](./03-requerimientos-funcionales.md)).

- [ ] `TenantMiddleware`, JWT (`core/jwt.py`, `core/security.py`), `auth_service.py` + `/auth/login`, `/auth/me`.
- [ ] `models/tenant.py`, `tenant_repository.py`, alta de tenant vía script de seed (RF-TEN-001, sin UI de super_admin todavía — se pospone a Fase 3 si no es crítico para el piloto).
- [ ] CRUD de `services` y `staff` (RF-TEN-010 a 012) — solo lo necesario para que un negocio configure su catálogo.
- [ ] `schedules` + `availability_service.py` (RF-AGE-001 a 004) — cálculo de slots libres.
- [ ] `appointment_service.py` + `appointment_repository.py` con validación de solapamiento (RF-AGE-023, RF-AGE-024).
- [ ] Router público `/public/{tenant_slug}/...` (RF-AGE-020, 021) + página Angular `/:tenantSlug/reservar`.
- [ ] Estados de cita y transiciones (RF-AGE-030 a 034), sin el otorgamiento de puntos todavía (eso es Fase 3).
- [ ] Índices de Mongo de [01-arquitectura.md §1.3](./01-arquitectura.md#13-índices-mongodb-por-colección) creados al startup.
- [ ] `test_availability.py`, `test_multitenancy.py` (RNF-SEC-003, RNF-OBS-003) — no avanzar a Fase 2 sin estos tests en verde.
- [ ] Deploy a Cloud Run con `min_instances = 0` (acepta cold start de MVP; ver 7.2 para el trigger de upgrade).

**Criterio de salida:** un negocio piloto puede configurar su agenda y un cliente externo puede reservar una cita real desde el link público, de punta a punta, en producción.

---

### Fase 2 — Notificaciones de bajo costo (1 semana)

RF del módulo `NOT` ([03-requerimientos-funcionales.md §3.4](./03-requerimientos-funcionales.md#34-módulo-de-notificaciones-y-recordatorios-not)).

- [ ] `notification_service.py` (fire-and-forget) + `notification_log_repository.py`.
- [ ] Integración SMTP gratuita (Brevo free tier o similar) — confirmación de reserva (RF-NOT-001) y notificación de cancelación (RF-NOT-003).
- [ ] Endpoint interno `/internal/send-reminders` protegido (ej. header secreto compartido, no expuesto públicamente).
- [ ] **Cloud Scheduler** (1 de los 3 jobs gratis) invocando ese endpoint una vez por hora para evaluar recordatorios de 24h (RF-NOT-002).
- [ ] Verificar que un fallo de envío no rompe el flujo de reserva (RNF-DISP-002) con un test dedicado.

**Criterio de salida:** el negocio piloto y sus clientes reciben confirmaciones y al menos un recordatorio automático, con costo $0.

---

### Fase 3 — CRM y Recompensas (2-3 semanas)

RF de los módulos `CRM` y `LOY` ([03-requerimientos-funcionales.md §3.3, §3.5](./03-requerimientos-funcionales.md)), usando el diseño de colecciones nuevas de [06-trazabilidad.md §6.3](./06-trazabilidad.md#63-nuevas-colecciones-requeridas-módulo-loy).

- [ ] `client_service.py` + `admin/clients.py` — historial de citas, servicios contratados (RF-CRM-001, 002).
- [ ] `client_note.py` / `client_note_repository.py` — notas privadas con flag `is_sensitive` y RBAC reforzado (RF-CRM-004 a 006, RNF-SEC-006).
- [ ] `loyalty_account.py`, `loyalty_transaction.py`, `reward.py`, `redemption.py` + repos correspondientes.
- [ ] `loyalty_service.py`: otorgamiento idempotente al completar cita (RF-LOY-020, índice único `sparse` sobre `appointment_id`), canje atómico (`find_one_and_update`, RF-LOY-012/014).
- [ ] UI de configuración de mecánica de recompensas para `business_admin` (RF-LOY-001 a 004) y catálogo de premios (RF-LOY-010).
- [ ] UI de saldo/canje para `client` (RF-LOY-011, 022).
- [ ] Exportación CSV de clientes (RF-CRM-008) verificando que excluye `client_notes` por diseño (mitigación de riesgo ya documentada en [06-trazabilidad.md §6.5](./06-trazabilidad.md#65-riesgos-técnicos-identificados)).

**Criterio de salida:** un cliente que completa varias citas acumula puntos/sellos automáticamente y puede canjearlos; el staff tiene notas privadas por cliente.

---

### Fase 4 — Hardening, dominio propio y piloto real (1-2 semanas)

- [ ] Revisión de seguridad enfocada en RNF-SEC-001 a 008 (aislamiento multi-tenant, RBAC, notas sensibles).
- [ ] Política de cancelación configurable (RF-TEN-013, RF-AGE-031).
- [ ] PWA básica para la página pública de reserva (RNF-USA-002).
- [ ] Compra de dominio propio (`agendix.app`) — **único gasto real del MVP** — y configuración en Firebase Hosting.
- [ ] Ajustar `ALLOWED_ORIGINS` / CORS al dominio final.
- [ ] Alta del primer negocio piloto real vía script/endpoint de super_admin (si aún no existía UI, se prioriza según necesidad real del piloto).
- [ ] Monitoreo manual de consumo real vs. límites free tier de la tabla 7.2 durante las primeras semanas con tráfico real.

**Criterio de salida:** un negocio real opera en producción sobre dominio propio, con el equipo monitoreando activamente los límites de los free tiers.

---

### Fase 5 — Escalamiento condicionado (bajo demanda, sin fecha fija)

No se ejecuta por calendario sino **cuando se dispara alguno de los triggers de la tabla 7.2**. Ejemplos:

| Señal observada | Acción de escalamiento |
|---|---|
| Atlas M0 > ~80% de 512 MB | Migrar a M10 (primer tier pago), sin cambio de esquema (ya previsto en RNF-ESC-001) |
| Cold start de Cloud Run afecta UX en horario pico | Configurar `min_instances = 1` para el servicio backend (deja de ser 100% $0 pero sigue siendo bajo costo) |
| > 300 emails/día reales | Subir de tier en Brevo/SendGrid |
| Negocio piloto pide recordatorios por WhatsApp | Implementar RF-NOT-006 fase SMS/WhatsApp (Twilio u similar), con costo variable asumido explícitamente por el negocio o el plan comercial de Agendix |
| Necesidad de reportes/analítica más allá de exportación CSV | Evaluar recién ahí una herramienta de BI, nunca antes |

---

## 7.4 Resumen de dependencias entre fases

```
Fase 0 (infra) ──▶ Fase 1 (Agendamiento) ──▶ Fase 2 (Notificaciones)
                                    │
                                    ▼
                         Fase 3 (CRM + Recompensas)
                                    │
                                    ▼
                         Fase 4 (Hardening + piloto real)
                                    │
                                    ▼
                    Fase 5 (Escalamiento — disparado por métricas, no por fecha)
```

Fase 2 puede desarrollarse en paralelo a la segunda mitad de Fase 1 si hay más de una persona en el equipo, ya que `notification_service.py` no bloquea a `appointment_service.py` (son fire-and-forget por diseño). Fase 3 depende de que Fase 1 ya tenga citas `completed` reales para disparar el otorgamiento de puntos.
