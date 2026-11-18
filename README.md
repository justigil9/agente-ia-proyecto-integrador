# Proyecto Integrador · Asistente Comercial de Atlas Consultoría Digital

Autora: Justina Gil

Agente de IA construido en n8n para **Atlas Consultoría Digital** (empresa ficticia): conversa con potenciales clientes, los califica con criterio **BANT** (presupuesto, autoridad, necesidad y plazo), los registra y avisa al equipo. El proyecto crece módulo a módulo hasta el Proyecto Final Integrador.

| Módulo | Entregable | Archivo |
|---|---|---|
| M1 · Agente base | Workflow (GitHub) | [`checkpoint1_justina_gil.json`](./checkpoint1_justina_gil.json) |
| M2 · Multi-agente (Manager-Worker) | PDF (plataforma) | — |
| M3 · Memoria en Airtable | PDF (plataforma) | — |
| **M4 · Integraciones HubSpot + Gmail + Slack** | **Workflow (GitHub)** | [`checkpoint4_justina_gil.json`](./checkpoint4_justina_gil.json) |

---

## Checkpoint 4 · Integraciones avanzadas (HubSpot + Gmail + Slack)

Canal de **correo electrónico** del asistente. Usa el mismo rol, las mismas reglas y la misma calificación BANT de los módulos anteriores, y consulta la memoria de Airtable del M3 antes de redactar.

### Credenciales conectadas (OAuth2)

![Credenciales de n8n](./img/m4_credenciales.png)

| Rol | Herramienta | Autenticación | Operaciones usadas (mínimo privilegio) |
|---|---|---|---|
| CRM | **HubSpot** | OAuth2 | Solo **contactos**: `search` (lectura) y `create or update` (escritura) con 5 propiedades (email, nombre, apellido, empresa y mensaje). No toca negocios, empresas ni tickets. |
| Casilla de correo | **Gmail** | OAuth2 | Lectura de la bandeja (solo no leídos, excluyendo los propios) y **Create Draft**. El workflow **nunca usa Send**. |
| Canal del equipo | **Slack** | OAuth2 | Solo `post message` en el canal **#ventas**. |
| Memoria (M3) | Airtable | Personal Access Token | Solo lectura (`search`) de la tabla Memoria. |

### Arquitectura

```
Email entrante (Gmail Trigger · in:inbox -from:me · no leídos)
  │
  ▼
① IF ¿auto-reply / no-reply? ── sí ──► Stop: correo automático ignorado
  │ no
  ▼
Agente Comercial (Tools Agent · GPT-4o · salida JSON estructurada)
  │   herramienta: Consultar memoria del cliente (Airtable, M3)
  ▼
④ Set: limpiar y validar payload ──► ¿Email válido? ── no ──► Stop (evita error 400)
  │ sí
  ▼
② Look up en HubSpot: ¿el contacto ya existe?
  ├─ sí ──► Actualizar contacto existente (solo agrega el mensaje)
  └─ no ──► Crear contacto nuevo (nombre, apellido, empresa, etapa Lead)
  ▼
③ Gmail Create Draft: borrador en el mismo hilo (HITL: una persona lo revisa y lo envía)
  ▼
⑤ Set: payload mínimo ──► Aviso al equipo en Slack (#ventas)
```

### Controles preventivos de la rúbrica

| # | Nodo | Problema que evita | Cómo |
|---|---|---|---|
| ① | **¿Es auto-reply o no-reply?** (IF, después del trigger) | Bucle infinito de respuestas automáticas | Condición OR, sin distinguir mayúsculas: asunto con *auto-reply, automatic reply, respuesta automática, out of office, fuera de la oficina, undeliverable, delivery status notification, mail delivery*, o remitente con *no-reply, noreply, mailer-daemon, postmaster*. Además, el trigger excluye los mails propios (`-from:me`). |
| ② | **Look up: ¿el contacto ya existe en HubSpot?** | Error 409 (contactos duplicados) | Búsqueda por email antes de escribir. Si existe, solo se actualiza el mensaje y no se pisan los datos cargados. Si no existe, se crea. |
| ③ | **Create Draft** (Gmail) | Envíos sin supervisión | La respuesta de la IA se guarda como **borrador dentro del hilo original**. Nada sale sin aprobación humana (Human-in-the-loop). |
| ④ | **Limpiar y validar payload** (Set + IF) | Error 400 (payload mal formado) | Deja solo email (normalizado), nombre, apellido, empresa, asunto (máximo 150 caracteres), categoría, calificación, resumen, borrador e id de hilo. Valida el email con una expresión regular antes de llegar al CRM. |
| ⑤ | **Payload mínimo para Slack** (Set) | Saturar el canal con payloads pesados o binarios | Envía un único campo de texto con el resumen (máximo 300 caracteres). Sin adjuntos, sin HTML y sin el cuerpo completo. |

### Pruebas de regresión

**Prueba 1 · Lead nuevo:** mail de Martín Pérez (Logística Sur). Recorrido completo: el contacto se creó en HubSpot, el borrador quedó en Gmail y el aviso llegó a Slack.

![Prueba 1](./img/m4_prueba1_lead_nuevo.png)

| HubSpot | Borrador en Gmail (HITL) | Aviso en Slack |
|---|---|---|
| ![HubSpot](./img/m4_hubspot_contacto.png) | ![Gmail](./img/m4_borrador_gmail.png) | ![Slack](./img/m4_aviso_slack.png) |

**Prueba 2 · Duplicado:** el mismo remitente otra vez. El Look up lo encuentra y el flujo va por **Actualizar contacto existente**: HubSpot sigue con un solo contacto.

![Prueba 2](./img/m4_prueba2_duplicado.png)

**Prueba 3 · Auto-reply:** asunto "Respuesta automática: fuera de la oficina". El IF ① corta el flujo antes del agente: no se usa IA, ni CRM, ni correo, ni Slack.

![Prueba 3](./img/m4_prueba3_autoreply.png)

### Cómo importarlo

1. En n8n: **Create workflow → ⋯ → Import from File** y elegir `checkpoint4_justina_gil.json`.
2. Asignar las credenciales de Gmail, HubSpot y Slack (OAuth2), OpenAI y Airtable.
3. En **Avisar al equipo (Slack)**, elegir el canal. En **Consultar memoria del cliente**, elegir la base y la tabla de Airtable.

---

## Checkpoint 1 · Agente base

`[Chat Trigger] → [AI Agent (Tools Agent) + System Prompt + Google Sheets Tool] → [Log de observabilidad (Gmail)]`

- System Prompt modular: Rol → Ámbito → Objetivo → Reglas → Restricciones → Escalamiento.
- Herramienta lateral: Google Sheets, con una descripción extensa de cuándo usarla y cuándo no.
- Guardrail: máximo 8 iteraciones.
- Observabilidad: Execution Log con los pasos intermedios, enviado por Gmail.

## Hoja de ruta

- [x] M1 · Agente base
- [x] M2 · Multi-agente (Manager-Worker con sub-workflows)
- [x] M3 · Memoria en Airtable por Session_ID con resumen automático
- [x] M4 · Integraciones HubSpot + Gmail + Slack (OAuth2)
- [ ] M5 · RAG
- [ ] M6 · Voz
- [ ] M11 · Proyecto Final Integrador
