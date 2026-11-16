# Proyecto Integrador · Agente Calificador de Leads

Autora: Justina Gil

Agente de IA construido en n8n que conversa con potenciales clientes de **Atlas Consultoría Digital** (empresa ficticia), releva sus datos comerciales, los califica con criterio **BANT** (presupuesto, autoridad, necesidad y plazo) y los registra en una planilla. El proyecto crece módulo a módulo hasta el Proyecto Final Integrador.

## Checkpoint 1 · Agente base y motor de razonamiento

Archivo: [`checkpoint1_justina_gil.json`](./checkpoint1_justina_gil.json)

```
[Chat Trigger] → [AI Agent (Tools Agent) + System Prompt + Tool] → [Log de observabilidad (Gmail)] → [Respuesta al chat]
                          │                    │
                 OpenAI GPT-4o        Google Sheets Tool
```

| Criterio | Implementación |
|---|---|
| **System Prompt** | Modular: Rol → Ámbito → Objetivo → Reglas → Restricciones → Escalamiento. Define explícitamente lo que el agente no puede hacer (precios, promesas, temas fuera de alcance). |
| **Herramienta** | Google Sheets Tool conectada lateralmente al agente. Agrega una fila por lead en la hoja `Leads`. Su descripción detalla cuándo usarla y cuándo no. Los campos los completa el modelo con `$fromAI()`. |
| **Guardrail** | Máximo **8 iteraciones** del ciclo ReAct (Options → Max Iterations). |
| **Observabilidad** | `Return Intermediate Steps` activado. Un nodo de Gmail envía un reporte por cada ejecución con el mensaje, la respuesta y el Execution Log (herramienta, razonamiento, entrada y observación de cada paso). |

### Pruebas realizadas

1. **"Hola, ¿qué servicios ofrecen?"**: el agente responde y repregunta. **No usa la herramienta** (0 herramientas en el log).
2. **Lead completo** (Martín Pérez, Logística Sur, USD 5.000, decide, 2 meses): el agente **usa la herramienta de forma autónoma**, registra la fila con calificación **CALIENTE** y el log llega por mail con el paso completo.

### Cómo importarlo

1. En n8n: **Create workflow → ⋯ → Import from File** y elegir el `.json`.
2. Asignar credenciales: OpenAI, Google Sheets y Gmail.
3. Crear una planilla con una hoja `Leads` y estos encabezados en la fila 1:
   `Fecha, Session_ID, Nombre, Email, Empresa, Necesidad, Presupuesto, Rol_Decision, Plazo, Calificacion, Resumen`.
4. Pegar la URL de la planilla en el nodo **Registrar lead calificado** y el email destino en el nodo de Gmail.

## Hoja de ruta

- [x] M1 · Agente base
- [ ] M2 · Multi-agente
- [ ] M3 · Memoria (Airtable por Session_ID)
- [ ] M4 · Integraciones (CRM / Calendario vía OAuth2)
- [ ] M5 · RAG
- [ ] M6 · Voz
- [ ] M11 · Proyecto Final Integrador
