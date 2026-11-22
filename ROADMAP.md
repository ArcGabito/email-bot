# Plan por fases: EmailBot

> **Objetivo**: que EmailBot sea la **puerta de entrada y de salida del correo** de la empresa. Lee todos los buzones corporativos, entiende cada correo con reglas e IA, lo enruta al bot o al área que corresponde, convierte las consultas en tickets con SLA y envía los correos transaccionales, de cobranza y de avisos de los demás sistemas con plantillas, colas y control de rebotes. Debe desplegarse con la misma imagen en un **VPS** (Docker Compose) o en **Azure Container Apps**.
>
> Stack: **Python 3.13 + uv · FastAPI · SQL Server / Azure SQL · Redis · colas (Dramatiq) · Docker**.

## Punto de partida

**Existe**, extraído de invoice-bot:

- Conectores de buzón para IMAP, API de Gmail y Microsoft Graph, con OAuth (incluido certificado para Microsoft 365) y lectura de adjuntos con límites.
- Base técnica: conexión a SQL Server (cadena de SSMS, seguridad integrada, identidad administrada), runner de migraciones con copia previa, candado, usuarios, seguridad de la API, inicio de sesión, logs JSON y `Caddyfile`.

**Falta**:

- `pyproject.toml`, `uv.lock`, `config.py`, `cli.py`, la aplicación FastAPI y las capas `domain` y `application`.
- Migraciones del esquema, pruebas, Dockerfile, Compose y CI.
- Módulos que el código ya importa pero no existen: `emailbot.config`, `emailbot.application.ports` y `emailbot.infrastructure.db.rows`.
- El primer push: el repositorio de GitHub está vacío.

## Arquitectura objetivo

```text
 Gmail / M365 ── push (Pub/Sub, webhooks de Graph) ──┐
 IMAP ── sondeo incremental ─────────────────────────┤
                                                     ▼
          ┌─────────────────────┐   encola    ┌──────────────── Redis ────────────────┐
 Caddy ─► │ api (FastAPI) x N   │ ──────────► │ colas · caché · candados · rate limit │
 HTTPS    │ + webhooks entrantes│             └───┬───────────────────────▲───────────┘
          └──────────┬──────────┘                 │ consume               │ encola (cron)
                     │                            ▼                       │
                     │                 ┌────────────────────┐   ┌─────────┴──────────┐
                     └───────────────► │ SQL Server + blob  │◄──│ worker(s) Dramatiq │──► invoice-bot,
                                       │ (.eml y adjuntos)  │   │ scheduler (1 líder)│    bank-bot,
                                       └────────────────────┘   └────────────────────┘    document-bot
```

| Proceso | Comando | VPS (Docker Compose) | Azure Container Apps |
| --- | --- | --- | --- |
| `api` | `emailbot serve` | servicio `api` tras Caddy (también recibe los webhooks de Graph y Pub/Sub) | Container App con ingress, escala por HTTP (mín. 1) |
| `worker` | `emailbot worker --queues inbound,classify,route,default` | servicio `worker`, `--scale worker=N` | Container App sin ingress, de 0 a N con KEDA (`redis`) |
| `worker-outbound` | `emailbot worker --queues outbound` | servicio aparte, para que un envío masivo no frene la lectura | Container App de 0 a N con KEDA |
| `scheduler` | `emailbot scheduler` | 1 réplica, candado de líder en Redis | Container Apps Job programado: `emailbot schedule tick` cada minuto |
| `migrate` | `emailbot db migrate` | `docker compose run --rm api db migrate` | Container Apps Job manual |
| `clamav` (opcional) | — | contenedor `clamav/clamav` para escanear adjuntos | Container App interno |
| Redis | — | `redis` con AOF y `noeviction` | Azure Managed Redis |
| SQL Server | — | perfil `sql` (Express) o servidor externo | Azure SQL serverless con identidad administrada |
| Archivos | — | volumen o MinIO (S3) | Azure Blob Storage con identidad administrada |

### Decisiones técnicas

| Tema | Decisión | Motivo |
| --- | --- | --- |
| Dependencias | **uv** con `uv.lock`, `.python-version` y `[dependency-groups] dev` | Compilaciones reproducibles y rápidas |
| Colas | **Dramatiq + Redis** | Trabaja con hilos (conectores y pyodbc síncronos). Trae reintentos con backoff, DLQ, rate limiters sobre Redis (imprescindibles para los límites de envío de Gmail y Microsoft 365) y métricas |
| Almacenamiento | Metadatos en SQL Server; `.eml` y adjuntos en blob, direccionados por SHA-256 | Las bases no crecen con binarios y un mismo adjunto se guarda una sola vez |
| Consistencia | **Outbox transaccional** para eventos y envíos | Un correo no se envía dos veces ni se pierde si Redis cae |
| Idempotencia | Clave única por (buzón, `Message-ID`) y, en IMAP, (`UIDVALIDITY`, `UID`) | Releer un buzón no duplica |
| Seguridad del buzón | Lectura por defecto. Mover y etiquetar es opcional. **Nunca se borra** un mensaje | Igual que en invoice-bot |
| IA | El contenido del correo va delimitado como datos y con instrucciones de ignorar órdenes que contenga | Defensa contra inyección de prompts en correos |
| Modo sin Redis | `EMAILBOT_QUEUE_BACKEND=inline` | Pruebas y desarrollo |

### Colas de EmailBot

| Cola | Actores | Límites |
| --- | --- | --- |
| `inbound` | `mailbox.sync(mailbox_id)`, `mailbox.handle_push(notification_id)`, `message.store(message_id)` | Una sincronización por buzón a la vez (candado en Redis) |
| `classify` | `message.classify(message_id)`, `message.extract(message_id)`, `attachment.scan(attachment_id)` | Rate limit de tokens de la IA |
| `route` | `message.route(message_id)`, `ticket.create_or_update(message_id)`, `bots.forward(attachment_id, target)` | Rate limit por bot destino |
| `outbound` | `mail.send(outgoing_id)`, `campaign.step(campaign_id)` | Rate limit por buzón y proveedor en una ventana de Redis |
| `default` | `sla.check()`, `mailbox.renew_subscriptions()`, `bounces.process()`, `retention.apply()`, `webhooks.deliver(event_id)`, `outbox.relay()`, `db.backup()` | — |

---

## Fase 0: Base del proyecto y plataforma

**Objetivo**: un proyecto que arranca, se prueba, se publica y se despliega, con la misma base que invoice-bot.

- [ ] `pyproject.toml` con uv (`uv init --package`), `uv.lock`, `.python-version` (3.13) y grupo `dev` (pytest, ruff, mypy).
- [ ] Crear los módulos faltantes: `config.py` (variables `EMAILBOT_*`), `context.py`, `cli.py` (Typer), `api/app.py`, `application/ports.py` e `infrastructure/db/rows.py`.
- [ ] **Esquema inicial** (migración v1): `mailboxes`, `mail_messages`, `mail_attachments`, `mail_threads`, `users`, `review_sessions`, `api_keys`, `jobs`, `outbox_messages` y `bot_runs`.
- [ ] **Redis + Dramatiq**: broker, actores `inbound` y `default`, y comandos `emailbot worker` y `emailbot scheduler`.
- [ ] **Dockerfile** multietapa con uv, ODBC 18, usuario sin privilegios y `HEALTHCHECK`. `docker-compose.yml` con `api`, `worker`, `worker-outbound`, `scheduler`, `redis` y los perfiles `sql` y `proxy`. Bicep para Container Apps (API, workers con KEDA, Jobs `scheduler` y `migrate`, Managed Redis, Blob y Key Vault).
- [ ] `/healthz`, `/readyz` (SQL Server, esquema y Redis), `/metrics` y `emailbot doctor`, que prueba la conexión con cada buzón configurado.
- [ ] **CI/CD**: `astral-sh/setup-uv`, ruff, mypy, pytest con SQL Server, Redis y GreenMail (servidor IMAP/SMTP de prueba) como servicios, imagen en GHCR, Trivy y despliegue con OIDC. Primer push con README, `.env.example` y `docs/`.

**Hecho cuando**: `emailbot mail fetch` lee un buzón de prueba en CI, guarda los mensajes una sola vez aunque se ejecute dos veces y la imagen corre en un VPS y en Container Apps.

## Fase 1: Ingesta multibuzón confiable

**Objetivo**: leer en casi tiempo real todos los buzones de la empresa sin perder ni duplicar correos.

- [ ] **Buzones en la base de datos** (no solo en variables), por empresa y área (facturas@, tesoreria@, ventas@, rrhh@…). Credenciales en Key Vault o cifradas.
- [ ] **Sincronización incremental**: IMAP con `UIDVALIDITY`/`UID`, Gmail con `historyId` y Graph con *delta query*.
- [ ] **Push**: Gmail `watch` con Pub/Sub y notificaciones de cambio de Graph, con renovación programada de suscripciones. Si fallan, se recurre al sondeo.
- [ ] **Hilos** reconstruidos por `Message-ID`, `In-Reply-To` y `References`.
- [ ] **Adjuntos**: guardado en blob por SHA-256, extracción de ZIP con límites (ya existe), escaneo antivirus (ClamAV) y bloqueo de extensiones peligrosas.
- [ ] **Señales de suplantación**: lectura de `Authentication-Results` (SPF, DKIM, DMARC), dominios parecidos a los de proveedores conocidos y marca de "posible phishing".

**Hecho cuando**: un correo llega a la base en menos de un minuto en Gmail y Microsoft 365 con push, y reiniciar los workers a mitad de una sincronización no pierde ni duplica mensajes.

## Fase 2: Clasificación y extracción con IA

**Objetivo**: saber qué es cada correo y qué datos trae, sin leerlo una persona.

- [ ] **Categorías configurables** por empresa: comprobante de proveedor, extracto o aviso bancario, pedido o cotización, reclamo, consulta, postulación (CV), notificación de entidad (SUNAT, SUNAFIL, municipalidad), cobranza recibida, spam y publicidad.
- [ ] **Reglas deterministas primero** (remitente, dominio, asunto, tipo de adjunto, XML UBL adjunto) y **LLM después**, con salida estructurada validada (categoría, confianza, prioridad, sentimiento y entidades como RUC, número de pedido, comprobante, importe y fechas).
- [ ] **Humano en el bucle**: si la confianza es baja, el correo va a una bandeja de revisión. Las correcciones alimentan las reglas sugeridas.
- [ ] Proveedor de IA intercambiable: Claude, Ollama local (los correos no salen del servidor) o simulado para pruebas.
- [ ] Registro por correo de modelo, versión de prompt, tokens y costo, como `extraction_runs` en invoice-bot.

**Hecho cuando**: sobre un conjunto etiquetado de correos reales, la precisión por categoría se mide en CI (evaluación automática) y no baja entre versiones.

## Fase 3: Enrutamiento y orquestación con los otros bots

**Objetivo**: que cada correo termine, solo, donde tiene que estar.

- [ ] **Motor de reglas sin código** (condición → acciones) editable desde la pantalla, con prueba en seco sobre correos anteriores.
- [ ] **Acciones disponibles**:
  - enviar los adjuntos a **invoice-bot** (`POST /api/v1/documents`), los extractos a **bank-bot** y los contratos y demás documentos a **document-bot**;
  - mover o etiquetar en el buzón y reenviar a un área;
  - crear un ticket;
  - llamar a un webhook;
  - responder con una plantilla.
- [ ] **Acuse de recibo automático** al remitente cuando corresponda (por ejemplo, "Recibimos su factura F001-123").
- [ ] **Prioridad alta y alerta inmediata** para notificaciones de entidades, avisos de fraude o cambios de cuenta bancaria de proveedores (coordinado con bank-bot).
- [ ] Trazabilidad de cada correo: qué regla se aplicó, a dónde fue y qué respondió el bot destino.

**Hecho cuando**: un correo con factura XML llega a invoice-bot y su resultado (aceptada u observada) queda visible en el detalle del correo en EmailBot.

## Fase 4: Mesa de ayuda (ticketing)

**Objetivo**: atender los buzones compartidos (ventas@, soporte@, proveedores@) como una mesa de ayuda, sin otra herramienta.

- [ ] **Correo → ticket**: cada hilo es un ticket con estado, prioridad, categoría, cliente y responsable.
- [ ] **Colas por área** con asignación por turnos o por carga, horario laboral y feriados de Perú.
- [ ] **SLA** de primera respuesta y de resolución, con avisos antes de vencer y escalamiento al supervisor (actor `sla.check`).
- [ ] **Respuestas** desde la pantalla, por el mismo buzón (Graph `sendMail`, Gmail `send` o SMTP) y conservando el hilo. Plantillas con variables y **borradores sugeridos por IA** que una persona revisa antes de enviar.
- [ ] Notas internas, menciones, fusión de tickets duplicados y encuesta CSAT al cerrar.
- [ ] Indicadores: tiempo de primera respuesta, tiempo de resolución, tickets por agente y cumplimiento de SLA.

**Hecho cuando**: un equipo atiende un buzón compartido solo desde EmailBot y los reportes de SLA cuadran con los tiempos reales.

## Fase 5: Correo saliente empresarial

**Objetivo**: un servicio de envío único para todos los sistemas de la empresa.

- [ ] **API de envío** `POST /api/v1/messages` para otros sistemas (con clave de API e `Idempotency-Key`), con adjuntos o enlaces a documentos de document-bot.
- [ ] **Plantillas versionadas** (Jinja2, HTML responsivo y texto plano) por empresa e idioma, con vista previa y datos de prueba.
- [ ] **Cola `outbound`** con rate limit por buzón y proveedor, reintentos con backoff, envíos programados y ventana horaria.
- [ ] Varios canales: SMTP, Microsoft Graph, API de Gmail y un proveedor transaccional (Amazon SES, Azure Communication Services).
- [ ] **Rebotes**: lectura de los DSN, lista de supresión por rebote duro y quejas. Encabezado `List-Unsubscribe` con baja en un clic (RFC 8058) en los envíos masivos. Comprobación de SPF, DKIM y DMARC del dominio en `emailbot doctor`.
- [ ] **Secuencias automáticas**:
  - cobranza escalonada (las dispara bank-bot);
  - vencimientos de contratos y pólizas (document-bot);
  - avisos a proveedores (invoice-bot);
  - reportes programados por correo.

**Hecho cuando**: invoice-bot, bank-bot y document-bot envían todos sus correos por EmailBot y cada envío tiene estado (en cola, enviado, rebotado o abierto si se activa).

## Fase 6: Cumplimiento y archivo

- [ ] **Retención** por categoría y empresa (por ejemplo, comprobantes durante el plazo tributario y spam 30 días) con borrado seguro de los datos propios. En el buzón de origen nunca se borra nada.
- [ ] **Archivo y eDiscovery**: búsqueda de texto completo (SQL Server Full-Text Search) sobre asunto, cuerpo y texto de adjuntos, y **retención legal** que bloquea el borrado.
- [ ] **DLP**: detección de DNI, tarjetas (algoritmo de Luhn) y cuentas bancarias en correos salientes, con alerta o bloqueo según la regla.
- [ ] **Ley 29733 de Protección de Datos Personales**: registro de consentimiento y baja de los envíos masivos, atención de solicitudes ARCO (exportar o eliminar los datos de una persona) y auditoría de accesos.

## Fase 7: Producción a escala

- [ ] **Observabilidad** con OpenTelemetry (FastAPI, Dramatiq, httpx y SQL) hacia Azure Monitor o Grafana. Alertas por suscripciones de push vencidas, buzones sin sincronizar, cola `outbound` creciendo, tasa de rebote y DLQ.
- [ ] **Alta disponibilidad y recuperación**: API con 2 o más réplicas (los webhooks de Gmail y Graph no deben perderse), Redis con persistencia y réplica, Azure SQL con redundancia de zona o copias probadas, blob con *soft delete* y un runbook con RPO y RTO.
- [ ] **Despliegues sin cortes** (revisiones de Container Apps o actualización gradual en el VPS) y migraciones como job previo.
- [ ] **Seguridad**: Renovate o Dependabot para `uv.lock`, Trivy, SBOM, OWASP ASVS nivel 2, secretos OAuth solo en Key Vault y rotación de certificados de Microsoft 365.
- [ ] **Pruebas de carga**: 50 000 correos entrantes y 20 000 salientes por día como referencia.

---

## Integración con la suite de bots

Cada bot tiene **su propia base de datos** y se despliega por separado. Se comunican por **API REST con claves** y **webhooks firmados** (HMAC-SHA256) en formato CloudEvents, publicados desde la outbox.

| Evento | Emisor → receptor | Efecto |
| --- | --- | --- |
| `email.attachment.received` (comprobante) | **email-bot** → invoice-bot | Procesa el comprobante |
| `email.attachment.received` (extracto) | **email-bot** → bank-bot | Importa el extracto |
| `email.attachment.received` (otro documento) | **email-bot** → document-bot | Clasifica y archiva |
| `invoice.rejected` / `invoice.paid` | invoice-bot → **email-bot** | Aviso al proveedor |
| `receivable.overdue` / `payment.confirmed` | bank-bot → **email-bot** | Cobranza o aviso de pago |
| `document.expiring` / `document.missing` | document-bot → **email-bot** | Alerta de vencimiento o pedido de documentos |
