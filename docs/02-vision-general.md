# 02 — Visión General, Propuesta de Valor y Actores

> Documento de requerimientos de producto. Coherente con la arquitectura definida en [01-arquitectura.md](./01-arquitectura.md): single DB + `tenant_id`, capas Router → Service → Repository, roles `super_admin` / `business_admin` / `staff` / `client`.

---

## 2.1 Problema y Propuesta de Valor

Los negocios de servicios (barberías, consultorios, gimnasios, centros de estética) gestionan hoy su agenda con una combinación de WhatsApp manual, cuadernos, hojas de cálculo o sistemas genéricos de calendario que no entienden su dominio: no distinguen proveedores con horarios propios, no calculan disponibilidad real por servicio, y no ofrecen ningún mecanismo para retener clientes más allá del boca a boca.

**Agendix** resuelve dos problemas simultáneos:

1. **Fricción operativa de agendar** — eliminar el ida y vuelta manual para reservar, confirmar, reprogramar o cancelar una cita, tanto para el negocio como para el cliente final.
2. **Fuga de clientes recurrentes** — dar al negocio una herramienta de fidelización (puntos/sellos canjeables) sin necesitar integrar un CRM externo.

### Propuesta de valor por vertical

| Vertical | Dolor principal hoy | Qué resuelve Agendix | Mecánica de recompensa típica |
|---|---|---|---|
| **Barberías / Peluquerías** | Agenda por WhatsApp, choques de horario entre barberos, clientes que no vuelven tras 1-2 visitas | Reserva pública por slug, un calendario por barbero, recordatorios que bajan el no-show | Sellos por corte (ej. "10 cortes = 1 gratis") |
| **Consultorios médicos/odontológicos** | Necesidad de historial clínico ligado a cada visita, confidencialidad de notas, ausentismo de pacientes | Notas privadas del proveedor por cliente, control de acceso estricto, recordatorios que reducen no-show | Puntos por controles periódicos cumplidos (fomenta adherencia a tratamiento, no "premios" tipo retail) |
| **Gimnasios / Centros de entrenamiento (coach 1:1, clases)** | Reservas de cupos/clases, ausentismo, necesidad de incentivar constancia | Reserva de sesiones con coach, bloqueo de agenda por clase llena, gamificación de asistencia | Puntos por racha de asistencia, canjeables por sesiones o merchandising |
| **Centros de estética / Spa** | Servicios de duración variable, upsell de tratamientos, clientes esporádicos que hay que recuperar | Catálogo de servicios con duración/precio propios, historial de tratamientos por cliente | Puntos por gasto acumulado, canjeables por descuentos o tratamientos |

### Objetivos de negocio (medibles)

| Objetivo | Métrica de éxito |
|---|---|
| Reducir el no-show | ↓ % de citas marcadas `no_show` tras activar recordatorios (meta: -30% vs. baseline sin recordatorio) |
| Aumentar recurrencia | ↑ % de clientes con ≥2 citas completadas en 90 días tras activar recompensas |
| Reducir fricción de reserva | Tiempo medio de reserva pública < 90 segundos, sin necesidad de cuenta previa |
| Autoservicio del negocio | Un `business_admin` configura servicios, staff y horarios sin soporte de Novex (onboarding < 30 min) |

---

## 2.2 Actores del Sistema

Se reutiliza el modelo de roles ya definido en [docs/README.md](./README.md) (idéntico en nombre técnico al de Zity, adaptado en scope). La tabla siguiente traduce la terminología de negocio a los roles técnicos que ya existen en el modelo `User` y en `require_role()`:

| Actor (negocio) | Rol técnico (`UserRole`) | Scope de datos | Responsabilidades clave |
|---|---|---|---|
| **SuperAdmin (Plataforma Novex)** | `super_admin` | Global, cross-tenant | Alta/baja de tenants (negocios), suspensión por impago, métricas agregadas de la plataforma, soporte de nivel 2 |
| **Owner / Admin del negocio** | `business_admin` | Su `tenant_id` | CRUD de servicios, staff, horarios y catálogo de recompensas; ve todas las citas y clientes del negocio; configura políticas de cancelación y expiración de puntos |
| **Provider / Staff** (barbero, doctor, coach, esteticista) | `staff` | Su `tenant_id` + su propio calendario | Gestiona su propia disponibilidad y bloqueos, ve/actualiza sus citas, marca `completed`/`no_show`, añade notas privadas al cliente |
| **Guest / Client** (cliente final) | `client` | Su `tenant_id` + sus propios datos | Reserva/cancela/reprograma sus citas (público o autenticado), consulta su historial y balance de puntos, canjea premios |

> **Nota de coherencia:** el rol `guard` de Zity no existe en Agendix (no hay control de acceso físico). El equivalente al "cliente anónimo que reserva sin cuenta" descrito en [01-arquitectura.md §1.2.1](./01-arquitectura.md#121-diferencia-clave-con-zity-página-pública-de-reservas) se modela como un `client` mínimo (email/teléfono, sin password obligatorio en el MVP) creado o reutilizado al confirmar la primera reserva pública.

### Matriz Actor × Módulo (alto nivel)

| Módulo | super_admin | business_admin | staff | client (autenticado o guest) |
|---|:---:|:---:|:---:|:---:|
| Agendamiento y Citas | — | CRUD total (tenant) | CRUD propio calendario | Crear/cancelar/reprogramar propias |
| Multitenancy y Personalización | CRUD tenants | Configurar su negocio | Lectura (su perfil) | — |
| Recompensas y Fidelización | — | Configurar reglas y catálogo | Otorgar puntos (al completar cita) | Consultar saldo y canjear |
| Notificaciones | — | Configurar plantillas/canales | Lectura de su bandeja | Recibe confirmaciones/recordatorios |
| Clientes (CRM básico) | — | Lectura total (tenant) | Lectura/escritura de sus clientes asignados | Lectura de su propio perfil |

Ver el detalle de cada módulo en [03-requerimientos-funcionales.md](./03-requerimientos-funcionales.md).
