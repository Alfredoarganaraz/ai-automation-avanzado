# Checkpoint 4 · Integraciones avanzadas e interconexión de sistemas

**Alumno:** Alfredo Argañaraz · Coderhouse, AI Automation Avanzado · Cohorte 2026
**Archivo del workflow:** [`checkpoint4_alfredo_arganaraz.json`](./checkpoint4_alfredo_arganaraz.json)
**Caso:** Kairos Salud Laboral (empresa ficticia de salud corporativa, Argentina y México). El workflow parte del Manager del Módulo 3 (router multiagente + memoria persistente por Session_ID) y lo conecta con la casilla de soporte (Gmail), el CRM (HubSpot) y el canal del equipo de operaciones (Slack).

---

## 1. Credenciales en n8n: las 3 conexiones activas

![Credenciales de n8n: Gmail, Slack y HubSpot](./img/credenciales_n8n.png)

| Herramienta (rol en el caso) | Credencial en n8n | Autenticación | Estado |
|---|---|---|---|
| **Gmail** · casilla de soporte | `Kairos · Gmail OAuth2 (casilla de soporte)` | OAuth2 (Google) | ![Gmail conectado](./img/cred_gmail.png) |
| **Slack** · canal `#kairos-operaciones` | `Kairos · Slack OAuth2 (bot, chat:write + channels:read)` | OAuth2 (Slack, flujo `oauth/v2/authorize` → `oauth.v2.access`) | ![Slack conectado](./img/cred_slack.png) |
| **HubSpot** · CRM de la tienda | `Kairos · HubSpot CRM (Service Key, solo contactos)` | Service Key de HubSpot | ![HubSpot conectado](./img/cred_hubspot.png) |

**Notas sobre la autenticación**
- **HubSpot:** desde junio de 2026 HubSpot ya no permite crear las "legacy public apps", que eran las que usaba el OAuth2 del nodo de n8n en instalaciones propias. El método recomendado hoy para integraciones de una sola cuenta es la **Service Key**, creada con permisos acotados (ver tabla de mínimo privilegio). El nodo usado es el conector nativo de HubSpot.
- **Slack:** la autorización se hace por **OAuth2** con una app propia ("Kairos Operaciones Bot"). Se usa la credencial OAuth2 genérica de n8n para pedir el token del bot con solo dos permisos. El aviso al canal sale por un HTTP Request a `chat.postMessage` firmado con esa credencial.

## 2. Mínimo privilegio

| Conexión | Permisos otorgados | Para qué se usan |
|---|---|---|
| Gmail | Credencial OAuth2 con los scopes estándar de n8n. En el flujo solo se usan dos operaciones: leer (trigger) y **Create Draft**; no hay ningún nodo de envío | Leer los correos que llegan a la casilla de soporte (solo los dirigidos al alias de soporte) y dejar la respuesta como borrador |
| HubSpot | `crm.objects.contacts.read`, `crm.objects.contacts.write` (nada más) | Buscar el contacto por email y crearlo o actualizarlo |
| Slack | `chat:write`, `channels:read` (bot); el bot solo es miembro de `#kairos-operaciones` | Avisar al equipo de operaciones |

- La credencial de Slack tiene además el uso restringido al dominio `slack.com` (Allowed HTTP Request Domains).
- **Lectura (pasado):** memoria del cliente en Google Sheets por Session_ID y búsqueda del contacto en HubSpot.
- **Escritura (futuro):** alta o actualización del contacto, borrador de respuesta y aviso a Slack.

## 3. Arquitectura y los 4 nodos que pide la rúbrica

**Entrada:** casilla de soporte, IF anti auto-reply y Set de limpieza.

![Entrada por Gmail](./img/flujo_entrada.png)

**Salidas:** Look up en HubSpot, Create Draft y Set mínimo antes de Slack.

![Salidas a HubSpot, Gmail y Slack](./img/flujo_salidas.png)

```
[Gmail Trigger · Casilla de soporte]
        │
        ▼
① [IF anti auto-reply] ── true ─▶ (Stop · Auto-reply ignorado)
        │ false
        ▼
④ [Set · Datos limpios del email]  → solo From, Subject, BodyText (+ Session_ID = email)
        │
[IF ¿Email válido?] ── false ─▶ (Descartar · payload inválido → evita el 400)
        │ true
        ▼
[Memoria M3 por Session_ID] → [Router de Intención (AI Agent)] → [Workers W1 / W2] → [Respuesta del agente]
        │
        ▼
④ [Set · Payload CRM (limpio)]
        │
② [Look up · ¿Contacto existe en HubSpot?]
   ┌────┴─────┐
   Sí          No
   ▼            ▼
[Update]    [Create]          (el Look up evita el 409)
        │
③ [Gmail · Create Draft (HITL)]   → nunca envía: un humano revisa y manda
        │
④ [Set · Payload mínimo Slack] → [Slack · Aviso a #kairos-operaciones]
```

| Nodo de la rúbrica | Nodo en el workflow | Configuración |
|---|---|---|
| ① IF anti auto-reply | `① IF anti auto-reply` | Inmediatamente después del trigger, modo **OR**, sin distinguir mayúsculas. Asunto contiene *Auto-reply*, *Out of office*, *Undeliverable* o *Respuesta automática*, o el remitente contiene *no-reply@*, *noreply@* o *mailer-daemon*. Si se cumple, el flujo termina en un Stop |
| ② Look up antes del Create | `② Look up · ¿Contacto existe en HubSpot?` + `¿Existe? (evita 409)` | Búsqueda por `email` (operador EQ) con *Always Output Data*. Existe → `Update · Contacto existente`, no existe → `Create · Contacto nuevo` |
| ③ Create Draft (HITL) | `③ Gmail · Create Draft (HITL)` | Resource *Draft*, operación *Create*, en el mismo hilo (`threadId`) y dirigido al remitente. Ningún nodo del flujo usa *Send* |
| ④ Set de limpieza | `Datos limpios del email`, `④ Set · Payload CRM (limpio)`, `④ Set · Payload mínimo Slack` | Cada Set deja pasar solo los campos que necesita el siguiente nodo. El cuerpo del correo se recorta y el email se valida con regex antes de seguir (evita el 400). A Slack llegan solo `channel` y `text` |

Además, el log por Gmail que tenía el Módulo 2 se reemplazó por el aviso a Slack, para que la salida por correo sea únicamente en borrador.

## 4. Test de regresión

| Prueba | Resultado |
|---|---|
| Correo con asunto "Out of office: vuelvo el lunes" | `① IF anti auto-reply` → rama true → `Stop · Auto-reply ignorado`. No se ejecuta ningún otro nodo |
| Primer correo "Consulta salud ocupacional" (contacto nuevo) | Memoria: usuario nuevo → Router → `Create · Contacto nuevo` en HubSpot → borrador creado en Gmail → aviso en Slack |
| Mismo remitente, segundo correo (contacto existente) | Memoria: usuario recurrente → Worker 1 registra el lead → Worker 2 redacta → `Update · Contacto existente` (sin duplicar) → borrador → aviso en Slack |

Todas las ejecuciones terminaron en estado *success*. Cada credencial se probó también por separado: HubSpot (búsqueda de contactos), Slack (`auth.test` y un mensaje real al canal) y Gmail (lectura de la casilla).

## 5. Cómo importarlo

1. En n8n: **Workflows → Import from File** y elegir `checkpoint4_alfredo_arganaraz.json`.
2. Asignar en cada nodo las credenciales propias de Gmail (OAuth2), HubSpot (Service Key con `crm.objects.contacts.read/write`) y Slack (OAuth2 con `chat:write` y `channels:read`). El `.json` no incluye secretos.
3. Para Slack: crear una app, agregar el *Redirect URL* que muestra n8n, invitar al bot al canal y reemplazar el ID del canal en `④ Set · Payload mínimo Slack`.
4. Los subflujos Worker 1 y Worker 2 son los del Módulo 2.
