# Documentación de Seguridad y Resiliencia

**Sistema de Triage de Tickets de Soporte con IA**

Este documento cubre los tres pilares exigidos por la rúbrica: minimización de datos, rutas de manejo de errores, y puntos de intervención humana (Human-in-the-Loop).

---

## 1. Minimización de datos

### 1.1 Qué datos se capturan y por qué

El sistema captura únicamente los campos estrictamente necesarios para operar el flujo de triage:

| Dato capturado | Origen | Propósito | ¿Se almacena de forma permanente? |
|---|---|---|---|
| Asunto del correo | Gmail Trigger (`Subject`) | Identificar el tema del ticket | Sí, en Airtable |
| Cuerpo del correo (snippet) | Gmail Trigger (`snippet`) | Dar contexto a la IA para clasificar y redactar | Sí, en Airtable |
| Clasificación (categoría/urgencia) | Generado por Claude Haiku | Enrutamiento y priorización | Sí, en Airtable |
| Estado del ticket | Generado por el flujo | Control de la máquina de estados | Sí, en Airtable |

**Explícitamente NO se capturan ni almacenan:**
- Datos personales del remitente más allá del correo electrónico (necesario para poder responder).
- Contenido de adjuntos del correo (el nodo Gmail Trigger no descarga archivos adjuntos — la opción "Download Attachments" permanece desactivada).
- Historial de conversación completo del hilo de correo — solo se toma el mensaje individual que dispara el ticket (`snippet`, no el hilo completo).
- Ninguna credencial (API keys, tokens de acceso) se almacena en el cuerpo de los mensajes ni en los registros de Airtable; viven exclusivamente en el sistema de credenciales cifradas de n8n.

### 1.2 Decisión de diseño: destinatario fijo durante desarrollo y pruebas

Durante la fase de desarrollo y pruebas de este proyecto, el campo **"To"** de los nodos de envío final (`Gmail Send`) se mantuvo fijo a una cuenta de control, en lugar de resolverse dinámicamente al remitente original del ticket.

**Razón:** esto evita el envío no intencional de comunicaciones automatizadas generadas por IA a personas reales mientras el sistema todavía estaba en pruebas de estrés (incluyendo pruebas del "camino infeliz" con datos incompletos o erróneos, donde una respuesta mal generada podría llegar a un destinatario real).

**Para producción**, la mejora identificada es capturar el remitente original en un campo dedicado de Airtable (`Correo Remitente`) al momento de crear el registro, y usarlo dinámicamente en el nodo de envío final mediante una expresión (`{{ $json.fields['Correo Remitente'] }}`), en vez de un valor fijo. Esta decisión se documenta aquí como una elección consciente de seguridad durante el desarrollo, no como una limitación técnica no resuelta.

---

## 2. Rutas de manejo de errores (Error Handlers)

### 2.1 Estrategia general

Todos los nodos que dependen de servicios externos (Anthropic API, Airtable API) están configurados con dos capas de resiliencia:

1. **Retry On Fail** — hasta 3 reintentos automáticos, con 1000 ms de espera entre cada uno. Esto cubre fallos transitorios comunes: límites de tasa (rate limits) momentáneos, timeouts de red, o caídas breves del servicio.
2. **Continue Using Error Output** — si después de los 3 reintentos el nodo sigue fallando, el flujo **no se detiene por completo**. En su lugar, el nodo genera una salida de error separada que se enruta hacia un nodo de registro centralizado.

### 2.2 Nodos cubiertos por esta estrategia

| Nodo | Tipo de fallo que podría ocurrir | Acción ante fallo |
|---|---|---|
| Claude Haiku (clasificación) | Timeout de API, límite de rate, respuesta malformada | Reintenta 3x → si falla, registra en LogError |
| Claude Sonnet (redacción) | Timeout de API, límite de rate | Reintenta 3x → si falla, registra en LogError |
| Update record (clasificación) | Fallo de conexión con Airtable, ID inválido | Reintenta 3x → si falla, registra en LogError |
| Update record (rama aprobada) | Fallo de conexión con Airtable | Reintenta 3x → si falla, registra en LogError |
| Update record (rama rechazada) | Fallo de conexión con Airtable | Reintenta 3x → si falla, registra en LogError |

### 2.3 Nodo LogError — registro centralizado

En vez de crear un nodo de manejo de errores independiente para cada punto de fallo (lo cual fragmentaría la visibilidad del sistema), todas las salidas de error convergen en un único nodo **LogError**, que actualiza el registro correspondiente en Airtable con `estado = "Error"`.

**Ventaja de este diseño:** cualquier ticket que haya fallado en cualquier punto del proceso queda visible de inmediato en el Dashboard de Control (ver sección de KPIs), bajo el estado "Error", sin necesidad de revisar logs técnicos dispersos.

```
Claude Haiku ──error──┐
Claude Sonnet ─error──┤
Update record ─error──┼──► LogError ──► Airtable (estado = "Error")
Update record (T) ─────┤
Update record (F) ─────┘
```

### 2.4 Evidencia de robustez observada en pruebas

Durante las pruebas de estrés del sistema (ver sección de pruebas), el flujo procesó correctamente incluso un correo de tipo "camino infeliz" no anticipado: una notificación automática de rebote de correo (*Delivery Status Notification — Failure*), que Gmail Trigger capturó como si fuera un ticket normal. El sistema lo procesó sin romperse — lo clasificó (aunque con una categoría no del todo precisa, dado que no es un ticket real) y continuó el flujo con normalidad, sin generar ningún error técnico.

Al momento de esta entrega, el Dashboard de Control reporta **0 tickets en estado "Error"** sobre el total de tickets procesados, lo cual confirma que la estrategia de reintentos ha sido suficiente para las condiciones de prueba evaluadas.

---

## 3. Puntos de Human-in-the-Loop

### 3.1 Por qué existe un punto de intervención humana

El sistema **nunca envía una respuesta generada por IA directamente al cliente sin aprobación humana previa**. Esto evita lo que el material del curso llama el "Efecto Metralleta": un sistema autónomo que actúa en cascada sin supervisión, amplificando cualquier error de la IA (una mala clasificación, una respuesta con tono inapropiado, información incorrecta) directamente hacia el usuario final.

### 3.2 Cómo funciona el punto de control

1. Tras la clasificación (Claude Haiku) y la redacción de la respuesta sugerida (Claude Sonnet), el sistema **se detiene** en el nodo `Send message and wait for response`.
2. Se envía un correo a un humano responsable, mostrando: la categoría y urgencia asignadas, el mensaje original del cliente, y la respuesta completa que la IA propone enviar.
3. El flujo queda pausado (estado "Waiting" en n8n) hasta que el humano toma una decisión mediante dos botones: **Aprobar** o **Desaprobar**.
4. Solo si el humano aprueba, el sistema procede a enviar la respuesta real al cliente.
5. Si el humano rechaza, el ticket se marca como `Rechazado por Humano` y se genera una notificación interna para que alguien le dé seguimiento manual — el sistema no reintenta generar otra respuesta automáticamente ni escala el ticket sin supervisión adicional.

### 3.3 Corrección de un bug crítico relacionado con este punto de control

Durante las pruebas se identificó y corrigió un error de tipos en la condición que evalúa la decisión humana. La expresión inicial (`{{ $json.data.approved }}` con el operador "is true") fallaba en distinguir correctamente un rechazo, porque bajo ciertas condiciones de conversión de tipos, JavaScript evalúa `Boolean("false")` como `true` (cualquier cadena de texto no vacía se considera "verdadera"). Esto podía provocar que un ticket **rechazado por el humano se procesara igualmente como aprobado**, comprometiendo directamente el propósito del punto de control.

**Corrección aplicada:** se cambió la condición a una comparación estricta de tipos:
```
{{ $json.data.approved === true }}
```
Esto garantiza que solo un valor booleano `true` genuino (no un texto que se parezca a "true") permite que el flujo continúe por la rama de aprobación. Se documenta este hallazgo porque ilustra un riesgo real de seguridad en sistemas de Human-in-the-Loop: una condición de aprobación mal implementada puede anular por completo el propósito del control humano sin que sea evidente a simple vista.

---

## 4. Checklist de seguridad (autoevaluación)

| Pregunta | Respuesta |
|---|---|
| ¿El flujo tiene un filtro para evitar bucles infinitos? | Sí — Gmail Trigger usa el filtro `-from:[cuenta propia] -subject:"Re:"` para excluir las propias respuestas automatizadas del sistema y evitar que se reprocesen como tickets nuevos. |
| ¿Se están comparando tipos de datos correctos en los filtros? | Sí — corregido explícitamente en el nodo IF (ver sección 3.3) mediante comparación estricta de booleanos. |
| ¿El prompt de IA es dinámico y usa variables del sistema? | Sí — ambos prompts (Haiku y Sonnet) incorporan variables reales del ticket (`Asunto`, `Mensaje`, `categoria`, `urgencia`) mediante expresiones de n8n, no texto estático. |
| ¿Existen rutas de error que eviten que el sistema se detenga por completo? | Sí — Retry On Fail + Continue Using Error Output en los nodos críticos, con registro centralizado en LogError. |
| ¿Existe un punto de aprobación humana antes de contactar al cliente final? | Sí — nodo `Send message and wait for response`, con las dos ramas (aprobado/rechazado) manejadas explícitamente. |
