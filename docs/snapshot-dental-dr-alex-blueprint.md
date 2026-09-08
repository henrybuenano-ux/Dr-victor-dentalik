# 🦷 SNAPSHOT BLUEPRINT — Nicho DENTAL (ref: cliente "Dr. Alex")

> **Cómo usar este documento:** es el inventario completo, a nivel GoHighLevel, del snapshot dental del Dr. Alex — uno de los nichos que mejor está funcionando. Sirve como **blueprint** para construir/mergear un **snapshot maestro** junto con los de otros nichos. Incluye: estructura (qué), lógica (por qué), lo que funciona (replicar), y los bugs conocidos (corregir, NO copiar tal cual).
>
> **Location de referencia:** `cuYGsE1t8wdILMeIbGgz` · Zona horaria: America/Mexico_City · Moneda: MXN
> **Estado:** todos los workflows en `draft` (snapshot de plantilla, se publican al desplegar en cada subcuenta).

---

## 0. RESUMEN DEL NICHO Y ARQUITECTURA

**Negocio:** clínica dental multi-especialidad, local (Aguascalientes). Ticket medio/alto, no compite por precio.
**Objetivo del sistema:** captar pacientes por WhatsApp, prefiltrarlos con un bot de IA, derivar a agendamiento, reducir no-shows, generar reseñas y reactivar base fría.

**Flujo maestro (el "camino del paciente"):**
```
Ads / orgánico (WhatsApp, IG, FB)
      ↓
Landing / mensaje directo → BOT "Eva" (Conversation AI)
      ↓ (prefiltra: servicio, para quién, urgencia)
Eva envía el LINK de agenda del especialista correcto (Dentalink externo)  ó  deriva a humano (Michelle)
      ↓
Michelle registra la cita en "Calendario - Citas" de GHL  → dispara confirmación + recordatorio
      ↓
Asistió → pide reseña Google   |   No asistió → remarketing/reactivación
```

**Regla de oro de diseño:** el bot **nunca agenda ni promete horarios** (no ve la agenda). Solo prefiltra y comparte el link correcto o deriva. El agendamiento vive en **Dentalink (externo)**, no en GHL.

**Especialistas y ruteo (clave del nicho dental):** varios doctores = varios calendarios. Eva rutea por servicio.

---

## 1. PIPELINES (2)

### 1.1 `Ventas` — funnel del paciente
Etapas (en orden, NO renombrar — los guards filtran por nombre/ID):
1. **Nuevo Lead** — la pone WF-01 al entrar el lead
2. **En conversación humana** — WF-07 (handoff)
3. **Pre-reserva / Cita Reservada** — WF-02 (tag) / WF-02b (cita registrada)
4. **Cita Confirmada** — WF-06 (señal) o manual
5. **No Asistió** — manual/por definir
6. **Servicio Realizado (Ganado)** — manual; dispara WF-09 (reseña)
7. **Perdido** — WF-04 (sin reserva tras remarketing)

### 1.2 `Campaña Reactivación Base de datos` — RBD
Etapas: Mensaje enviado → Whatsapp inválido → Respondió algo → Da clic para votar → Votaron 1-3 → Votaron 4-5 → Dejó reseña en Google → Estancada 30 días

---

## 2. CUSTOM FIELDS (56)

**De captación/lead:**
Canal de primer contacto (dropdown) · Fecha de primer contacto (date) · Estado del lead (dropdown) · Servicio de interés (text libre, lo llena Eva) · Tipo de servicios Dropdown (valor normalizado por WF-05) · Tipo de cliente (¿primera vez?) · Fecha de cita (date) · Resumen de conversación (large text, lo escribe Eva) · Responsable / a quién derivar

**De cita / asistencia / pago:**
Antelación cancelación · Tolerancia de llegada · Señal pagada (checkbox) · ¿Pides señal? (radio) · Importe de la señal · Método de pago · Datos para el pago · Titular de la cuenta · Servicio realizado

**De atribución:** UTM Source / Medium / Campaign · Ad / Anuncio

**De reseña/encuesta:** Califícanos del 1 al 5 (radio) · Enlace reseña Google

**De negocio (heredados del form de onboarding, ver §7 "Actualización de CV"):**
Nombre del negocio · Nombre del bot · Dirección · Localidad · Enlace Google Maps · Enlace calendario/agenda · Horario de atención · Email del negocio · WhatsApp del negocio (phone) · Web · Instagram · TikTok · Página de Facebook · Moneda · Política de privacidad (URL) · Preguntas frecuentes (large text) · Info adicional servicios (large text) · Promoción vigente (large text) · Servicio 1–6 (text) · Precio 1–6 (numérico) · Logos Horizontal 16:9 (file) · Logo Vertical 9:16 (file)

> ⚠️ **Nota de limpieza para el master:** muchos campos "de negocio" **duplican** custom values (§3). Es herencia del form de onboarding que escribe ambos. En el master, decidir UNA fuente de verdad (recomendado: custom values para el bot, y solo los campos de contacto que de verdad varían por lead).

---

## 3. CUSTOM VALUES (55) — variables del negocio y del bot

Los referencia el prompt de Eva (merge fields) y los workflows. Cambiar un dato aquí actualiza todo sin tocar prompts.

**Identidad (llenos):** nombre_negocio · nombre_responsable (Michelle) · nombre_bot (Eva) · direccion · localidad · link_maps · whatsapp_negocio · email_negocio · web · instagram · facebook · horario_atencion · moneda

**Servicios y precios (llenos):** servicio_1–6 + precio_1–6 · info_servicios · faq · datos_pago · cancelacion (24 h) · tolerancia (10 min, máx 15)

**Links de agenda por especialista (llenos — Dentalink/healthatom):**
- `link_agenda` (DEFAULT → valoración general / Dr. Alex)
- `link_agenda_dr_alex` (ortodoncia/general)
- `link_agenda_implantes` (Dr. Josué)
- `link_agenda_endodoncia` (Dr. Carlos)
- `link_agenda_odontopediatria` (Dra. Elizabeth)

**Conocimiento del bot (llenos, creados vía API):** politica_precios_bot · equipo_especialistas · diferenciales_clinica

**⚠️ VACÍOS a resolver por subcuenta (13):** promo_activa · politica_privacidad · link_resena (duplicado) · nombre_empresa (dup) · numero_whatsapp (dup) · logo_negocio · metodo_pago · nombre_pago · monto_senal · imagen_funnel · tiktok · form_comentarios · survey

> **Para el master:** consolidar los duplicados (link_resena, nombre_empresa, numero_whatsapp) en una sola clave cada uno.

---

## 4. TAGS (37) — con lógica de sistema

**Tags de sistema (los usan triggers/guards — nombre EXACTO importa):**
`lead-nuevo` (WF-01 → dispara WF-04) · `wf01-procesado` (guard anti-reproceso WF-01) · `pre-reserva` (trigger WF-02) · `pre-reserva-pendiente-agendar` · `cita-agendada` · `cita-cancelada` · `recordatorio-24h-pendiente` (WF-03) · `no-reservo` (WF-04) · `senal-pagada` (trigger WF-06) · `solicitud-asesor-humano` (trigger WF-07) · `baja-marketing` (WF-08) · `campana de reactivación` (WF-09 → enrola RBD) · `canal-whatsapp` / `canal-instagram` / `canal-facebook` (ruteo de Eva) · `reactivacion-bd` · `whatsapp-invalido` · `estancada-30-dias` · `google_review` · `[whatsapp] - contact is not registered on whatsapp` (trigger RBD 06)

**⚠️ Duplicados a limpiar en el master** (elegir una variante por par):
`voto 1-3` / `voto-1-3` · `votó 4-5` / `voto-4-5` · `dejó reseña en google` / `dejo-resena-google` / `dio-clic-review` · `no dio respuesta` / `no-dio-respuesta` · `respondió a campana reactivacion bd` / `respondio-reactivacion` · `link directo google` / `link-directo-google`
**Eliminar tags de prueba:** `test a`, `test b`, `test/wp`

---

## 5. CALENDARIOS (4)

- **`Calendario - Citas`** ⭐ el importante: es el **trigger de WF-02b (confirmación) y WF-03 (recordatorio)**. Aquí Michelle registra la cita (agendada en Dentalink) para disparar la automatización.
- 3 personales (auto-creados por usuario): Dr. Alex, Michelle, Rubria.

> **Nota de arquitectura:** el agendamiento del paciente es **EXTERNO (Dentalink)**. GHL no crea el appointment; Michelle lo registra en "Calendario - Citas" para activar los flujos. (Alternativa avanzada para el master: puente Dentalink→Google Calendar→GHL vía n8n/Make.)

---

## 6. AGENTE IA "EVA" (Conversation AI) — el corazón del nicho

**Tipo:** GHL Conversation AI (no Agent Studio). Modelo OpenAI GPT-5. Prompt (con política de precios híbrida) ~1,800 palabras (límite editor 2,000). Ver `docs/eva-prompt.md`.

**Función:** atiende TODO el WhatsApp entrante (leads y pacientes). Prefiltra (servicio, para quién, urgencia), responde dudas y precios, y **envía el link del especialista correcto** o **deriva a humano**.

**Política de precios (v2 — híbrida):** da precios cuando el paciente los pide:
- **Precios CERRADOS** (valoración $500, limpieza $800, resina/caries $800, blanqueamiento $4,500, brackets inicial $6,000 + $800/mes) → los cotiza directo + engancha con valoración.
- **Precios VARIABLES** (muela del juicio $1,800/$2,500/$3,000, carillas $2,900–$6,000, placas) → da rango + valoración.
- Antes (v1) NUNCA cotizaba; el doctor pidió el cambio a cotizar los cerrados.

**Acciones configuradas (fuera del prompt):**
- **Contact Info** (guarda nombre, teléfono, Servicio de interés — SIN correo)
- **Human Handover** (pausa bot + tag `solicitud-asesor-humano` → WF-07)
- **Appointment Booking: ELIMINADA** a propósito (evitaba que ofreciera horarios inexistentes)

**Ruteo por especialista (regla clave del nicho):**
| Caso | Custom value | Doctor |
|---|---|---|
| Niños ≤12 | link_agenda_odontopediatria | Dra. Elizabeth |
| Implantes | link_agenda_implantes | Dr. Josué |
| Endodoncia | link_agenda_endodoncia | Dr. Carlos |
| Brackets/alineadores/orto | link_agenda_dr_alex | Dr. Alex |
| Primera vez / duda | link_agenda (default) | valoración |

---

## 7. WORKFLOWS (21)

### Núcleo de captación/atención (WF)
| WF | Nodos | Trigger | Propósito |
|---|---|---|---|
| **WF-01 Bienvenida + Activación Bot** | 18 | Customer Replied (WA) sin tag `wf01-procesado` | Marca procesado, detecta canal (WA/IG/FB), crea oportunidad Nuevo Lead, tag `lead-nuevo`, activa a Eva |
| **WF-02 Pre-reserva** | 7 | Tag `pre-reserva` | Mueve etapa, asigna, notifica al equipo, nota operativa, reactiva bot |
| **WF-02b Cita Reservada** | 5 | Appointment en `Calendario - Citas` | Tag cita-agendada, mueve etapa, **envía confirmación** |
| **WF-03 Recordatorio 24h** | 5 | Appointment en `Calendario - Citas` | Recordatorio antes de la cita |
| **WF-04 Remarketing** | 10 | Tag `lead-nuevo` | 2 toques a leads que no reservaron → Perdido + `no-reservo` |
| **WF-05 Normalización Servicio** | 14 | Contact Changed (Servicio de interés) | Mapea texto libre → `Tipo de servicios Dropdown` (7 ramas) |
| **WF-06 Confirmación de Señal** | 3 | Tag `senal-pagada` | DORMIDO (pide_senal=No) |
| **WF-07 Handoff a Humano** | 2 | Tag `solicitud-asesor-humano` | Mueve a "En conversación humana" + notifica |
| **WF-08 Opt-out / Baja** | 3 | Reply "Baja/Stop" | Tag baja-marketing + DND + confirmación |
| **WF-09 Solicitud de Reseña** | 2 | Etapa "Servicio Realizado (Ganado)" | Espera 1 día + tag `campana de reactivación` (enrola RBD) |

### Onboarding y utilidades
| WF | Nodos | Propósito |
|---|---|---|
| **Actualización de CV** | 41 | Form "Onboarding y CV" → escribe los 41 custom values del negocio. **El motor de configuración del snapshot.** ⚠️ Cada reenvío SOBREESCRIBE todos los values. |
| **Test bot Sofía** | 1 | Prueba interna — **eliminar antes del go-live** |

### Serie RBD — Reactivación Base de Datos + reseñas
| WF | Nodos | Trigger | Propósito |
|---|---|---|---|
| **RBD 01 Reactivación** | 48 | Form "Registro Clientes Nuevos" | Campaña madre: split A/B, drip 4/10, secuencia de mensajes pidiendo valoración 1-5 |
| **RBD 02 Clic trigger-link** | 4 | Trigger link | Detecta clic para votar |
| **RBD 03 Cliente responde** | 5 | Customer Reply (de RBD 01) | Marca respuesta + notifica |
| **RBD 04 Votaron 1-3** | 5 | Form valoración 1-3 | Feedback negativo → gestión privada (sin reseña pública) |
| **RBD 05 Votaron 4-5** | 5 | Survey ≥4 | Promotores → se les pide reseña Google |
| **RBD 06 WhatsApp inválido** | 4 | Tag [whatsapp] not registered | Higiene: saca del flujo + DND |
| **RBD 07 "No puedo entrar al link"** | 10 | Reply "no puedo entrar…" | Reenvía link directo de reseña Google |
| **RBD 08 Dejó reseña Google** | 12 | Reputation review received | Cierra ciclo: oportunidad ganada + notifica |
| **RBD 09 Estancada 30 días** | 5 | Opportunity decay 30d | Limpia pipeline de reactivación |

---

## 8. PLANTILLAS DE WHATSAPP (6) — para aprobación de Meta

| Nombre | Categoría | La usa | Notas |
|---|---|---|---|
| `sp01_confirmacion_cita` | Utility | WF-02b | Confirmación (fecha/hora/dirección) |
| `sp02_recordatorio_24h` | Utility | WF-03 | Recordatorio día previo |
| `sp03_recordatorio_mismo_dia` | Utility | (opcional) | Recordatorio matinal |
| `ps01_resena_google` | Marketing | reseña post-visita | Botón → survey de gating (1-3 form interno / 4-5 Google) |
| `ls01_seguimiento_lead` | Marketing | WF-04 | Seguimiento lead sin agendar |
| `ls02_reactivacion_pacientes` | Marketing | RBD/reactivación | Paciente 6+ meses |

> **Aprendizaje del nicho (review gating):** el botón de reseña apunta a una **página-survey** que califica 1-5; ≤3 → form interno privado, 4-5 → Google. Protege el rating público. La landing y los templates apuntan al mismo survey.

---

## 9. INTEGRACIONES / DEPENDENCIAS EXTERNAS

- **WhatsApp WABA** conectado en GHL (canal principal de Eva).
- **Dentalink (healthatom)** — agendamiento externo. 1 link por especialista (custom values §3).
- **Landing / funnel de GHL** (`registro.doctoralexdentista.com`) con widget de WhatsApp (mensaje pre-cargado para atribución) + prueba social (reseñas Google reales).
- **Google Business Profile** — para RBD 08 (reputation) y reseñas.
- **Google Ads (PMAX) + GA4** — aprendizaje clave: la conversión que importa es el **clic al botón de WhatsApp** (medirlo como conversión web para que PMAX optimice hacia leads reales, no solo "acciones locales" automáticas).

---

## 10. ✅ LO QUE FUNCIONA — REPLICAR EN EL MASTER

1. **Bot Eva con ruteo por especialista** (multi-calendario) — el diferenciador del nicho; probado en conversaciones reales.
2. **Separación bot ↔ humano** limpia (Human Handover como acción, no en prompt).
3. **Conocimiento en custom values** (no en el prompt) → cambios sin tocar al bot.
4. **Normalización de servicio (WF-05)** → datos limpios para reportes/ruteo.
5. **Ciclo completo:** captación → prefiltro → agenda → confirmación/recordatorio → asistencia → reseña → reactivación.
6. **Review gating** (survey 1-5 → Google/privado) para reputación.
7. **Landing de conversión** (copy de dolor + presupuesto cerrado + WhatsApp como CTA único + reseñas reales).
8. **Form de onboarding → 41 custom values** = despliegue rápido por subcuenta.

---

## 11. ⚠️ BUGS / MEJORAS CONOCIDAS — CORREGIR EN EL MASTER (no copiar tal cual)

1. **WF-03 (recordatorio):** el Wait es "1 día DESPUÉS del registro", debe ser **"1 día ANTES de la cita"** (Appointment Time − 1 día).
2. **WF-02b / WF-03 / WF-04:** nodos de mensaje son **placeholders** — conectar las plantillas reales aprobadas.
3. **RBD 01:** 8 `template_id` **rotos** (apuntan a la cuenta master) + `assign_user` a **usuario inexistente** → recrear plantillas y reasignar. (Afecta notificaciones de RBD 03/04/05/08/09.)
4. **WF-07:** notificación "al asignado" cae al vacío si el lead pide humano antes de tener asignado → añadir assign o usuario fijo.
5. **RBD 07:** usa el CV `link_resea_en_google` que está VACÍO (el lleno es el duplicado) → consolidar.
6. **WF-05:** el catálogo servicio_1–6 no incluía implantes/endodoncia/alineadores (servicios estrella) → ampliar ramas/catálogo.
7. **Tags duplicados** (§4) y **workflows/tags de prueba** → limpiar.
8. **Custom fields vs custom values duplicados** (§2) → una sola fuente de verdad.
9. **Recordatorio "mismo día":** Wait Until Date no soporta offset de horas → aproximar con Wait fijo o n8n.

---

## 12. DECISIONES DE DISEÑO CLAVE (el "por qué")

- **El bot NO agenda** → evita prometer horarios que no ve (agenda en Dentalink). Solo comparte link o deriva.
- **Agendamiento externo (Dentalink)** → el paciente elige día/hora ahí; Michelle lo replica en GHL para disparar flujos. Puente unidireccional.
- **Ventana de 24h de WhatsApp** → fuera de ventana solo templates aprobados; dentro, mensaje libre. (Define qué mensaje es template y cuál libre.)
- **Ruteo por especialista** → varios doctores = varios links; el bot decide por servicio/edad.
- **Conocimiento en custom values** → mantenimiento sin tocar prompt.
- **Review gating** → los descontentos a canal privado, los contentos a Google.
- **Precios híbridos (v2)** → cotizar los cerrados reduce fricción; los variables van a valoración.

---

*Fin del blueprint — Nicho DENTAL (Dr. Alex). Estructura reutilizable: los blueprints de otros nichos deberían seguir estas mismas 12 secciones para facilitar el merge en el snapshot maestro.*
