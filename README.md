# Módulo 1 — Agente autónomo básico funcional

**Caso:** Agente Calificador de Leads B2B de **Kairos Salud Laboral** (empresa ficticia de salud corporativa: salud ocupacional, planes corporativos, cumplimiento normativo y prevención · Argentina y México).
**Entregable:** `checkpoint1_alfredo_arganaraz.json` exportado de n8n después de la prueba · link al repo en la plataforma.

## Arquitectura

```
[Chat de Consultas] → [Agente Calificador de Leads] → [Log de Observabilidad (Gmail)] → [Respuesta al Chat]
                              │  (Tools Agent · max 6 iteraciones)
          ┌───────────────────┼─────────────────────────┐
   [OpenAI GPT-4o]   [Memoria de la Conversación]   [Registrar Lead en Google Sheets]  ← tool lateral
```

| Nodo | Rol |
|---|---|
| Chat de Consultas (Chat Trigger) | Recibe el mensaje libre de la empresa |
| Agente Calificador de Leads (AI Agent, Tools Agent) | Conversa, pide datos, califica ALTA/MEDIA/BAJA, decide cuándo registrar o escalar. Max Iterations = 6 |
| OpenAI GPT-4o | Modelo de lenguaje (temperatura 0.2, prioriza datos estructurados limpios) |
| Memoria de la Conversación (Simple Memory, 10 mensajes) | Recuerda lo que dijo el usuario en mensajes anteriores del mismo chat |
| Registrar Lead en Google Sheets (tool) | Append or Update por Email: no duplica leads |
| Log de Observabilidad (Gmail) | Mail al supervisor con mensaje, respuesta y herramientas usadas (entrada y resultado) |
| Respuesta al Chat (Set) | Devuelve al chat la respuesta del agente |

`system_prompt.md` tiene el System Message y la Description de la herramienta en texto plano.

## Pasos en n8n

1. **Planilla:** en Google Drive creá una hoja "Leads Kairos" e importá `plantilla_leads_kairos.csv` (solo trae los encabezados).
2. **Importar:** en n8n → *Workflows → Import from File* → `checkpoint1_alfredo_arganaraz.json`.
3. **Credenciales** (n8n las marca en rojo hasta que las cargues):
   - *OpenAI GPT-4o* → credencial OpenAI (API key con saldo).
   - *Registrar Lead en Google Sheets* → credencial Google Sheets OAuth2, y elegí la planilla y la hoja en *Document* y *Sheet*.
   - *Log de Observabilidad (Gmail)* → credencial Gmail OAuth2 y reemplazá `TU_EMAIL_DE_SUPERVISION@ejemplo.com` por tu mail.
4. **Probar** con *Open chat*, por ejemplo:
   - "Hola, somos Logística del Sur, una empresa de transporte en Córdoba con 120 empleados. Nos interesan los exámenes preocupacionales."
   - Luego darle nombre, cargo y email cuando lo pida.
   - Prueba de escalamiento: "¿Cuánto sale por empleado?"
5. **Auditar** en *Executions*: recorrido en verde, la rama de la tool abierta, fila nueva en la planilla y mail recibido.
6. **Exportar:** *⋯ → Download*, renombrar a `checkpoint1_alfredo_arganaraz.json` y reemplazar el de esta carpeta.

## Checklist de la consigna

- [x] Trigger: Chat Trigger
- [x] AI Agent en modo Tools Agent
- [x] Chat Model nativo: OpenAI GPT-4o
- [x] Max Iterations entre 5 y 10 → **6**
- [x] System Message modular: Rol → Ámbito (qué NO puede hacer) → Objetivo → Reglas de herramientas → Escalamiento, sin lenguaje inclusivo
- [x] Tool nativa (Google Sheets) acoplada lateralmente, con Description extensa de cuándo usarla y cuándo no
- [x] El agente no queda después de un IF/Switch
- [x] Nodo final de observabilidad (Gmail) con el resultado y las herramientas ejecutadas
- [ ] Prueba manual con recorrido en verde
- [ ] Exportado desde n8n con el nombre exacto y subido acá
