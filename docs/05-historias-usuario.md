# 05 — Historias de Usuario y Criterios de Aceptación

> Formato estándar: *"Como [Actor], quiero [Acción] para [Beneficio]"*. Criterios de aceptación en formato Dado/Cuando/Entonces para historias de agendamiento y recompensas (las de mayor complejidad de negocio); checklist para el resto. Cada historia referencia sus RF de origen ([03-requerimientos-funcionales.md](./03-requerimientos-funcionales.md)).

---

## 5.1 Agendamiento y Citas

### US-01 — Reservar una cita desde la página pública
**Como** cliente final (guest, sin cuenta), **quiero** reservar una cita eligiendo servicio, proveedor y horario disponible desde el link público del negocio, **para** agendar sin tener que llamar o escribir por WhatsApp.
*(RF-AGE-020, RF-AGE-021, RF-AGE-023)*

**Criterios de aceptación:**
- **Dado** que ingreso a `agendix.app/:tenantSlug/reservar`, **cuando** selecciono un servicio, **entonces** veo únicamente el staff habilitado para ese servicio (RF-TEN-012).
- **Dado** que elijo un staff, **cuando** consulto horarios, **entonces** veo solo slots realmente libres (sin solapamiento con citas `pending`/`confirmed` ni bloqueos de agenda).
- **Dado** que confirmo un slot con mis datos de contacto, **cuando** otro cliente reservó ese mismo slot milisegundos antes, **entonces** recibo un error claro (`409`) y se me ofrece elegir otro horario, sin crear una cita duplicada.
- **Dado** que la reserva se confirma, **cuando** el proceso termina, **entonces** se crea (o reutiliza por email/teléfono) mi registro de `client` y recibo un email de confirmación, sin que el email fallido bloquee la confirmación de la cita (RF-NOT-007).

### US-02 — Bloquear la agenda por vacaciones
**Como** staff (barbero/doctor/coach), **quiero** bloquear un rango de fechas en mi calendario, **para** que nadie pueda reservarme mientras estoy de vacaciones o incapacitado.
*(RF-AGE-010, RF-AGE-011)*

**Criterios de aceptación:**
- **Dado** que defino un bloqueo del 10 al 15 de agosto, **cuando** un cliente intenta reservar en ese rango, **entonces** no ve ningún slot disponible conmigo en esas fechas.
- **Dado** que ya existen citas `confirmed` dentro del rango que quiero bloquear, **cuando** intento guardar el bloqueo, **entonces** el sistema me advierte y me pide confirmar explícitamente, mostrando cuántas citas se ven afectadas.
- **Dado** que confirmo el bloqueo sobre citas existentes, **cuando** se guarda, **entonces** se dispara notificación a cada cliente afectado (RF-NOT-003).

### US-03 — Marcar una cita como completada o no-show
**Como** staff, **quiero** marcar cada cita como `completed` o `no_show` al finalizar mi jornada, **para** mantener actualizado el historial del cliente y disparar el otorgamiento de puntos cuando corresponda.
*(RF-AGE-032, RF-AGE-033)*

**Criterios de aceptación:**
- **Dado** una cita en estado `confirmed` cuya hora de inicio ya pasó, **cuando** la marco `completed`, **entonces** el estado cambia y, si el tenant tiene recompensas activas, se otorgan puntos/sellos automáticamente (RF-LOY-020) de forma idempotente.
- **Dado** una cita en estado `confirmed` donde el cliente no se presentó, **cuando** la marco `no_show`, **entonces** el estado cambia y **no** se otorgan puntos.
- **Dado** que soy `client`, **cuando** intento marcar una cita como `completed`, **entonces** el sistema rechaza la acción con `403` (RF-AGE-032).

### US-04 — Cancelar o reprogramar mi propia cita
**Como** cliente final, **quiero** cancelar o reprogramar una cita que ya reservé, **para** liberar el horario si mis planes cambian sin tener que llamar al negocio.
*(RF-AGE-031)*

**Criterios de aceptación:**
- **Dado** una cita `confirmed` dentro de la ventana mínima de anticipación configurada por el negocio, **cuando** intento cancelarla, **entonces** el sistema me lo permite y notifica al staff (RF-NOT-004).
- **Dado** una cita `confirmed` fuera de la ventana mínima de anticipación (ej. faltan 30 min y la política exige 2h), **cuando** intento cancelarla, **entonces** el sistema rechaza la acción y me explica la política vigente.
- **Dado** que intento cancelar una cita de **otro** cliente, **cuando** envío la solicitud, **entonces** el sistema rechaza con `403` (aislamiento por `client_id` + `tenant_id`).

---

## 5.2 Sistema de Recompensas y Fidelización

### US-05 — Configurar la mecánica de puntos del negocio
**Como** business_admin, **quiero** configurar si mi negocio otorga puntos por monto gastado o sellos por visita, y su expiración, **para** adaptar la fidelización a mi modelo de negocio.
*(RF-LOY-001, RF-LOY-002, RF-LOY-003)*

**Criterios de aceptación:**
- **Dado** que elijo la mecánica "sellos por visita", **cuando** un cliente completa una cita de un servicio incluido, **entonces** se le otorga exactamente 1 sello, sin importar el precio del servicio.
- **Dado** que elijo la mecánica "puntos por monto", **cuando** un cliente completa una cita de $15.000 con tasa 1 punto/$1.000, **entonces** se le otorgan 15 puntos.
- **Dado** que configuro expiración a 180 días, **cuando** consultan su saldo pasado ese plazo, **entonces** esos puntos específicos ya no cuentan en el saldo vigente (RF-LOY-011).
- **Dado** que excluyo un servicio promocional del programa (RF-LOY-003), **cuando** un cliente completa una cita de ese servicio, **entonces** no se otorgan puntos.

### US-06 — Canjear un premio del catálogo
**Como** cliente final, **quiero** canjear mis puntos/sellos acumulados por un premio del catálogo, **para** obtener un beneficio tangible por mi fidelidad.
*(RF-LOY-010 a RF-LOY-014)*

**Criterios de aceptación:**
- **Dado** que tengo 120 puntos vigentes y un premio cuesta 100, **cuando** solicito el canje, **entonces** mi saldo baja a 20 y el canje queda `pending` de entrega (RF-LOY-013).
- **Dado** que tengo 80 puntos vigentes y el premio cuesta 100, **cuando** intento canjear, **entonces** el sistema rechaza la operación indicando el saldo faltante.
- **Dado** que dos solicitudes de canje del mismo cliente llegan casi simultáneamente y el saldo solo alcanza para una, **cuando** ambas se procesan, **entonces** solo una tiene éxito y la otra es rechazada (sin saldo negativo, RF-LOY-014).
- **Dado** un canje `pending`, **cuando** el staff entrega el premio y confirma en el sistema, **entonces** el canje pasa a estado entregado y queda en el historial del cliente (RF-LOY-022).

### US-07 — Consultar historial de puntos
**Como** cliente final, **quiero** ver el historial de mis puntos ganados, canjeados y expirados, **para** entender cómo se compone mi saldo actual.
*(RF-LOY-022)*

**Checklist de aceptación:**
- [ ] Veo cada movimiento con fecha, tipo (ganado/canjeado/expirado/ajuste manual) y monto.
- [ ] El saldo mostrado coincide con la suma de movimientos vigentes (no expirados).
- [ ] Los ajustes manuales del `business_admin` (RF-LOY-021) aparecen identificados como tales, no confundidos con puntos ganados por visita.

---

## 5.3 Multitenancy y Personalización

### US-08 — Dar de alta un nuevo negocio en la plataforma
**Como** super_admin, **quiero** crear un nuevo tenant con su slug único, **para** habilitar a un negocio nuevo en Agendix.
*(RF-TEN-001)*

**Checklist de aceptación:**
- [ ] El sistema rechaza slugs duplicados (índice único en `tenants.slug`).
- [ ] El negocio queda operativo de inmediato: su `business_admin` puede loguearse y configurar servicios/staff.
- [ ] El slug queda reflejado en la URL pública `agendix.app/:slug/reservar`.

### US-09 — Configurar el catálogo de servicios y personal
**Como** business_admin, **quiero** crear mis servicios con duración y precio, y asignar qué staff puede prestarlos, **para** que la disponibilidad pública refleje correctamente mi operación real.
*(RF-TEN-010, RF-TEN-011, RF-TEN-012)*

**Checklist de aceptación:**
- [ ] Puedo crear un servicio con nombre, duración en minutos, precio y categoría.
- [ ] Puedo asignar uno o más miembros del staff a ese servicio.
- [ ] Al desactivar un servicio, desaparece de la reserva pública pero las citas históricas que lo referencian no se ven afectadas.
- [ ] La disponibilidad pública de un staff solo muestra los servicios que tiene asignados.

---

## 5.4 Notificaciones y Recordatorios

### US-10 — Recibir confirmación y recordatorio de mi cita
**Como** cliente final, **quiero** recibir un email al confirmar mi cita y un recordatorio antes de la hora agendada, **para** no olvidarme de asistir.
*(RF-NOT-001, RF-NOT-002, RF-NOT-005)*

**Checklist de aceptación:**
- [ ] Recibo un email de confirmación inmediatamente después de reservar, sin que su envío retrase la respuesta de la reserva (fire-and-forget).
- [ ] Recibo un recordatorio 24h antes del inicio de la cita.
- [ ] Si el envío falla (ej. email inválido), la cita permanece válida y el fallo queda registrado en `notifications_log` para que el negocio pueda darse cuenta.

---

## 5.5 Clientes (CRM Básico)

### US-11 — Consultar el perfil e historial de un cliente
**Como** staff, **quiero** ver el historial de citas, servicios contratados y saldo de puntos de un cliente, **para** atenderlo con contexto (ej. saber que es cliente frecuente o que tiene un premio por canjear).
*(RF-CRM-001, RF-CRM-002, RF-CRM-003)*

**Checklist de aceptación:**
- [ ] Veo el listado cronológico de citas del cliente con su estado.
- [ ] Veo qué servicios ha contratado históricamente.
- [ ] Veo su saldo de puntos/sellos vigente.
- [ ] No veo clientes de otros tenants bajo ninguna combinación de filtros o búsqueda.

### US-12 — Agregar una nota privada sobre un cliente
**Como** staff de un consultorio, **quiero** dejar una nota privada sobre un paciente marcada como sensible, **para** registrar información clínica relevante sin exponerla al propio cliente ni a personal no autorizado.
*(RF-CRM-004, RF-CRM-005, RF-CRM-006, RNF-SEC-006)*

**Criterios de aceptación:**
- **Dado** que escribo una nota y la marco como sensible, **cuando** la guardo, **entonces** solo yo (autor) y el `business_admin` del tenant pueden leerla — no otros `staff` del mismo negocio.
- **Dado** que soy el `client` dueño del perfil, **cuando** consulto mi propio historial, **entonces** no veo ninguna nota interna, sensible o no.
- **Dado** que otro `staff` del mismo tenant intenta leer la nota sensible, **cuando** hace la request, **entonces** el sistema la rechaza y registra el intento en el log de auditoría (RNF-OBS-001).
