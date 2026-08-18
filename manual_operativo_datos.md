# Manual Operativo de Datos

**Sistema de Triage de Tickets de Soporte con IA**
Orquestador: n8n (self-hosted, Docker) · Base de datos: Airtable · Motor de IA: Anthropic API (Claude Haiku / Sonnet) · Canal: Gmail

---

## 1. Esquema de la base de datos (Airtable)

### 1.1 Identificadores de la base

| Elemento | Valor |
|---|---|
| Base | `Tickets Soporte IA` |
| Base ID | `appSuLAfznvdFKpCI` |
| Tabla | `Table 1` |
| Table ID | `tblTKlO8I3GHZNxVb` |
| Vistas | `Entrantes`, `Pendientes`, `Procesados` (vistas sobre la misma tabla, no tablas separadas) |

> **Nota de diseño:** el sistema usa una sola tabla con un campo de estado como máquina de estados, en vez de mover registros entre tablas distintas. Las "vistas" (`Entrantes`, `Pendientes`, `Procesados`) son simplemente filtros guardados sobre esa tabla para facilitar la revisión manual, no estructuras de datos separadas.

### 1.2 Campos de la tabla `Table 1`

| Campo | Tipo | Descripción | Valores posibles |
|---|---|---|---|
| `Asunto` | Texto de una línea (campo primario) | Asunto del correo original del ticket | Texto libre |
| `Mensaje` | Texto multilínea | Cuerpo del correo original (snippet de Gmail) | Texto libre |
| `estado` | Selección única (Single Select) | Estado actual del ticket dentro del flujo — funciona como máquina de estados | `Pendiente`, `Procesado por IA`, `Aprobado por Humano`, `Rechazado por Humano`, `Error` |
| `Clasificación IA` | Texto multilínea | Resultado combinado de la clasificación de Claude Haiku | Texto con formato `Categoría: X \| Urgencia: Y` |
| `Fecha de Creación` | Fecha | Fecha de creación del registro | Fecha (formato local) |
| `Resumen (Clasificación IA)` | Campo de IA nativo de Airtable (`aiText`) | Resumen automático generado por Airtable sobre el campo `Clasificación IA` | Texto generado |
| `Titular (Clasificación IA)` | Campo de IA nativo de Airtable (`aiText`) | Titular automático generado por Airtable sobre el campo `Clasificación IA` | Texto generado |

### 1.3 Diagrama de estados del campo `estado`

```
Pendiente
    │  (Claude Haiku clasifica)
    ▼
Procesado por IA
    │  (Claude Sonnet redacta + humano revisa)
    ▼
   ┌────────────┴────────────┐
   ▼                         ▼
Aprobado por Humano   Rechazado por Humano


   (en cualquier punto, si un nodo crítico falla
    después de sus reintentos, el registro pasa a:)
Error
```

### 1.4 Por qué esta estructura evita "datos aislados"

El requisito del curso pide relaciones entre tablas para evitar datos aislados. En este caso, la relación funcional se resuelve con el campo `estado` como eje central: cada nodo del flujo de n8n lee y escribe sobre el mismo registro, nunca se duplica información ni se crean registros huérfanos. El `id` de Airtable (generado automáticamente, ej. `recdelpTxHbTdP8xh`) es la clave que viaja entre nodos para garantizar que todas las actualizaciones (clasificación, aprobación, respuesta) apunten siempre al mismo ticket.

---

## 2. Esquemas JSON de transferencia entre integraciones

A continuación se documenta la forma real de los datos que viajan entre cada nodo del flujo, en el orden en que ocurren.

### 2.1 Gmail Trigger → n8n (entrada del ticket)

Disparador: polling cada minuto sobre la bandeja de entrada, con filtro de búsqueda `from:aurora.giacoman@udem.edu -subject:"Re:"` para evitar que el sistema reprocese sus propias respuestas (prevención de bucles infinitos). Para esta entrega, el filtro se acotó a un remitente específico con fines de control durante las pruebas; el detalle de esta decisión y su implicación en producción se documenta en `seguridad_resiliencia.md`.

```json
{
  "id": "19fddcfe18ca529f",
  "threadId": "19fddcfe18ca529f",
  "snippet": "Texto breve del cuerpo del correo...",
  "payload": {
    "mimeType": "multipart/alternative"
  },
  "sizeEstimate": 52585,
  "historyId": "2055802",
  "internalDate": "1786132815000",
  "labels": [
    { "id": "INBOX", "name": "INBOX" },
    { "id": "IMPORTANT", "name": "IMPORTANT" }
  ],
  "From": "\"Nombre del remitente\" <correo@dominio.com>",
  "Subject": "Asunto del correo",
  "To": "correo-de-soporte@dominio.com"
}
```

**Campos usados por el flujo:** `Subject` → `Asunto`, `snippet` → `Mensaje`.

---

### 2.2 Create a record (n8n → Airtable)

```json
{
  "fields": {
    "Asunto": "{{ $json.Subject }}",
    "Mensaje": "{{ $json.snippet }}",
    "estado": "Pendiente"
  }
}
```

**Respuesta de Airtable:**

```json
{
  "id": "recdelpTxHbTdP8xh",
  "createdTime": "2026-08-08T19:59:49.000Z",
  "fields": {
    "estado": "Pendiente",
    "Mensaje": "...",
    "Asunto": "..."
  }
}
```

---

### 2.3 Search records (Airtable → n8n)

Este paso ocurre dentro de la misma ejecución disparada por el Gmail Trigger (no existe un Schedule Trigger independiente): inmediatamente después de crear el registro, el flujo busca ese mismo registro en Airtable para continuar con la clasificación. 
Filtro por fórmula: `{estado} = 'Pendiente'`.


```json
{
  "id": "recdelpTxHbTdP8xh",
  "createdTime": "2026-08-08T19:59:49.000Z",
  "fields": {
    "estado": "Pendiente",
    "Mensaje": "Texto del mensaje del cliente...",
    "Asunto": "Asunto del ticket",
    "Modificacion": "2026-08-08T19:59:49.000Z"
  }
}
```

---

### 2.4 Claude Haiku — clasificación (n8n → Anthropic API → n8n)

**Petición (resumen de parámetros relevantes):**

```json
{
  "model": "claude-haiku-4-5-20251001",
  "max_tokens": 100,
  "messages": [
    {
      "role": "user",
      "content": "Clasifica el siguiente ticket de soporte en una categoría (Técnico, Administrativo, Académico, Otro) y asigna una urgencia (Alta, Media, Baja). Responde SOLO con este formato JSON: {\"categoria\": \"...\", \"urgencia\": \"...\"}\n\nAsunto: {{ $json.fields.Asunto }}\nMensaje: {{ $json.fields.Mensaje }}"
    }
  ]
}
```

**Respuesta cruda de la API:**

```json
{
  "content": [
    {
      "type": "text",
      "text": "```json\n{\"categoria\": \"Administrativo\", \"urgencia\": \"Baja\"}\n```"
    }
  ]
}
```

> **Nota:** el modelo devuelve el JSON envuelto en un bloque de código Markdown (```` ```json ... ``` ````). Este formato no es directamente utilizable por los siguientes nodos, de ahí el nodo de limpieza (2.5).

---

### 2.5 Code (JavaScript) — normalización del JSON

Este nodo limpia el texto crudo devuelto por Claude y lo combina con los datos originales del ticket para que estén disponibles en un solo objeto.

**Lógica aplicada:**
1. Extrae `content[0].text` de la respuesta de Claude.
2. Elimina las marcas ```` ```json ```` y ```` ``` ````.
3. Convierte el texto limpio con `JSON.parse()`.
4. Combina el resultado con los datos originales del registro (`id`, `fields`, etc.).

**Salida normalizada:**

```json
{
  "id": "recdelpTxHbTdP8xh",
  "fields": {
    "estado": "Pendiente",
    "Mensaje": "...",
    "Asunto": "..."
  },
  "categoria": "Administrativo",
  "urgencia": "Baja"
}
```

---

### 2.6 Update record — guarda la clasificación (n8n → Airtable)

```json
{
  "Columns to match on": "id",
  "id": "{{ $('Code in JavaScript').item.json.id }}",
  "fields": {
    "Clasificación IA": "Categoría: {{ $json.categoria }} | Urgencia: {{ $json.urgencia }}",
    "estado": "Procesado por IA"
  }
}
```

---

### 2.7 Claude Sonnet — redacción de la respuesta sugerida

**Petición:**

```json
{
  "model": "claude-sonnet-4-5-20250929",
  "max_tokens": 200,
  "messages": [
    {
      "role": "user",
      "content": "Eres un asistente de soporte de una institución educativa. Redacta una respuesta profesional, breve y cordial (máximo 100 palabras)...\n\nCategoría: {{ categoria }}\nUrgencia: {{ urgencia }}\nAsunto: {{ Asunto }}\nMensaje del cliente: {{ Mensaje }}"
    }
  ]
}
```

**Respuesta:**

```json
{
  "content": [
    {
      "type": "text",
      "text": "Estimado/a usuario/a:\n\nGracias por contactarnos. Hemos recibido su consulta sobre..."
    }
  ]
}
```

> A diferencia de Claude Haiku, aquí el prompt pide texto plano (no JSON), por lo que la respuesta no requiere limpieza adicional — se usa directamente `content[0].text`.

---

### 2.8 Send message and wait for response — Human-in-the-Loop (Gmail)

Este nodo envía un correo de aprobación con botones interactivos y **pausa la ejecución del workflow** hasta que un humano responde.

**Correo enviado (resumen de campos):**

```json
{
  "to": "correo-de-revision@dominio.com",
  "subject": "Aprobación requerida: {{ Asunto }}",
  "message": "Categoría: {{ categoria }}\nUrgencia: {{ urgencia }}\n\nMensaje del cliente:\n{{ Mensaje }}\n\nRespuesta sugerida por IA:\n{{ respuesta_sonnet }}\n\n¿Apruebas el envío de esta respuesta al cliente?",
  "responseType": "approval",
  "approveButtonLabel": "Aprobar",
  "disapproveButtonLabel": "Desaprobar"
}
```

**Resultado al recibir la respuesta del humano:**

```json
{
  "data": {
    "approved": true,
    "respondedAt": "2026-08-10T16:58:04.240Z"
  }
}
```

---

### 2.9 If — nodo de decisión

Evalúa la respuesta humana para bifurcar el flujo:

```
Condición: {{ $json.data.approved === true }}
```

> **Nota de implementación:** se usa comparación estricta (`===`) en vez de solo `{{ $json.data.approved }}` con el operador "is true", porque n8n puede convertir el valor booleano a texto (`"false"`) durante el procesamiento, y en JavaScript `Boolean("false")` evalúa a `true` (cualquier string no vacío es "verdadero"). La comparación estricta evita este error de tipos.

- **Salida `true`** → rama de aprobación
- **Salida `false`** → rama de rechazo

---

### 2.10 Update record (por rama) — cierre del ciclo

**Rama aprobada:**
```json
{ "id": "...", "fields": { "estado": "Aprobado por Humano" } }
```

**Rama rechazada:**
```json
{ "id": "...", "fields": { "estado": "Rechazado por Humano" } }
```

---

### 2.11 Gmail Send — salida final

**Rama aprobada (respuesta al cliente):**
```json
{
  "to": "{{ correo del remitente original }}",
  "subject": "Re: {{ Asunto }}",
  "message": "{{ respuesta generada por Claude Sonnet }}"
}
```

**Rama rechazada (notificación interna):**
```json
{
  "to": "correo-de-revision@dominio.com",
  "subject": "Ticket requiere revisión manual: {{ Asunto }}",
  "message": "El siguiente ticket fue marcado como 'Rechazado por Humano'..."
}
```

---

### 2.12 LogError — registro centralizado de fallos

Nodo de destino común para las salidas de error de los nodos críticos (Claude Haiku, Claude Sonnet, Update record en ambas ramas). Ver sección de Seguridad y Resiliencia para el detalle de la configuración de reintentos.

```json
{
  "Columns to match on": "id",
  "id": "{{ $('Code in JavaScript').item.json.id || $('Search records').item.json.id }}",
  "fields": { "estado": "Error" }
}
```

---

## 3. Resumen del flujo de datos de punta a punta

```
Gmail (correo entrante)
      │  { Subject, snippet, From }
      ▼
Airtable · Create record
      │  { Asunto, Mensaje, estado: "Pendiente" }
      ▼
Airtable · Search records (polling)
      │  { id, fields }
      ▼
Claude Haiku
      │  { content[0].text: "```json {categoria, urgencia}```" }
      ▼
Code (JS) — normaliza
      │  { id, fields, categoria, urgencia }
      ▼
Airtable · Update record
      │  estado: "Procesado por IA"
      ▼
Claude Sonnet
      │  { content[0].text: "respuesta en texto plano" }
      ▼
Gmail · Send & Wait for Response (humano)
      │  { data: { approved: true/false } }
      ▼
   If (approved === true)
      │
   ┌──┴───────────────┐
   ▼                  ▼
Aprobado           Rechazado
   │                  │
Airtable Update    Airtable Update
   │                  │
Gmail Send         Gmail Send
(al cliente)       (notificación interna)
```
