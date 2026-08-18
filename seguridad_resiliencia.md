# Seguridad y Resiliencia

**Sistema de Triage de Tickets de Soporte con IA**

---

## 1. Manejo de errores

Los nodos críticos del flujo (Claude Haiku, Claude Sonnet, y las operaciones de Update record en ambas ramas) están configurados con:

- **Retry On Fail:** 3 intentos automáticos, con 1 segundo de espera entre cada uno. Esto cubre fallos transitorios (timeouts de red, límites de rate temporales de la API) sin necesidad de intervención manual.
- **Continue Using Error Output:** si después de los 3 intentos el nodo sigue fallando, el flujo no se detiene por completo. En vez de eso, la ejecución continúa hacia un nodo de registro de errores.

### 1.1 Registro centralizado de fallos (LogError)

Todas las salidas de error de los nodos críticos convergen en un único nodo, `LogError`, que actualiza el registro correspondiente en Airtable con `estado = "Error"`. Esto tiene dos ventajas sobre manejar el error de forma distinta en cada nodo:

- Un solo punto de revisión: para saber si algo falló, basta con filtrar la tabla por `estado = "Error"`, sin tener que revisar cada rama del flujo por separado.
- Ningún ticket se pierde silenciosamente. Si la clasificación o la redacción fallan, el ticket no desaparece del sistema — queda marcado y visible para revisión manual.

## 2. Human-in-the-Loop como control de seguridad

El diseño del flujo asume que ningún texto generado por IA debe llegar a un cliente real sin que un humano lo revise primero. Esto se implementa con el nodo `Send message and wait for response`, que pausa la ejecución hasta que alguien aprueba o rechaza la respuesta sugerida.

Esta decisión responde a un riesgo concreto: un modelo de lenguaje puede generar una respuesta con tono inapropiado, información incorrecta, o una promesa que la institución no puede cumplir. El paso de aprobación humana existe específicamente para atrapar esos casos antes de que salgan del sistema.

### 2.1 Verificación de tipos en el nodo de decisión

Durante las pruebas se detectó que la respuesta del nodo de aprobación (`data.approved`) puede llegar como booleano o como texto, dependiendo de cómo n8n serializa el dato en cada ejecución. Si el nodo `If` se hubiera configurado comparando solo `{{ $json.data.approved }}` como verdadero, un valor como el string `"false"` se habría evaluado como verdadero (en JavaScript, cualquier string no vacío es "truthy"), lo que habría enviado al cliente una respuesta que un humano acababa de rechazar.

Para evitar esto, la condición usa comparación estricta:

```
{{ $json.data.approved === true }}
```

Este es un ejemplo concreto de un error de tipos que, sin la verificación adecuada, habría roto la garantía central del sistema (que nada llega al cliente sin aprobación real).

## 3. Prevención de bucles de reprocesamiento

El Gmail Trigger que dispara el flujo usa un filtro de búsqueda para evitar que el sistema reaccione a sus propios correos salientes, lo cual crearía un ciclo infinito de envío y reprocesamiento.

Para efectos de esta entrega, el filtro configurado es:

```
from:aurora.giacoman@udem.edu -subject:"Re:"
```

Este filtro se acotó intencionalmente a un remitente específico (la cuenta usada como "cliente" de prueba) para que, durante la grabación del video demo, correos ajenos a la prueba (spam, notificaciones, promociones) no dispararan el flujo por accidente y contaminaran la demostración.

**Esto es una limitación conocida, no un diseño final.** En un entorno de producción real, el filtro debería ampliarse para aceptar cualquier remitente externo, ya que el sistema necesita recibir tickets de clientes distintos, no de una sola cuenta. La forma recomendada de hacerlo sería:

```
-from:[cuenta-de-soporte] -subject:"Re:"
```

Es decir, excluir únicamente la propia cuenta de soporte (para no reaccionar a las respuestas que el sistema mismo envía), sin restringir de qué remitente externo puede venir un ticket.

### 3.1 Riesgo no cubierto: correos publicitarios o no deseados

Durante las pruebas, un correo publicitario (no relacionado con soporte) llegó a la bandeja y fue procesado por el sistema como si fuera un ticket real. La clasificación de IA lo identificó correctamente como fuera de alcance (categoría "Otro", urgencia "Baja") y generó una respuesta apropiada indicando que el mensaje parecía ser publicidad enviada por error. Aun así, esto representa un consumo innecesario de llamadas a la API y de tiempo de revisión humana.

Como mejora futura, se recomienda agregar un paso de filtrado previo (por ejemplo, verificación de dominio del remitente contra una lista de dominios institucionales permitidos) antes de crear el registro en Airtable, en vez de depender únicamente de que la IA identifique correos no relevantes después de haberlos procesado.

## 4. Minimización de datos

- El campo `Mensaje` en Airtable almacena únicamente el `snippet` (fragmento breve) del correo original, no el cuerpo completo del mensaje ni sus adjuntos.
- No se almacenan credenciales ni tokens de acceso dentro de los datos del flujo; la autenticación con Gmail se maneja mediante OAuth2 gestionado directamente por n8n, fuera de los registros de Airtable.
- El registro en Airtable no conserva metadatos técnicos del correo (headers completos, IPs de origen, firmas DKIM) que no son necesarios para el propósito del sistema.

## 5. Resumen de riesgos identificados y su mitigación

| Riesgo | Mitigación implementada | Estado |
|---|---|---|
| Fallo transitorio de la API de Anthropic | Retry automático (3x) | Implementado |
| Ticket perdido silenciosamente tras un error | Registro centralizado en LogError, estado "Error" visible | Implementado |
| Respuesta de IA inapropiada enviada sin revisión | Aprobación humana obligatoria antes del envío | Implementado |
| Error de tipos en la validación de aprobación (string vs. booleano) | Comparación estricta (`=== true`) en el nodo If | Implementado |
| Bucle infinito por reprocesar respuestas propias | Filtro de exclusión sobre la cuenta de soporte | Implementado |
| Correos no relacionados con soporte (spam, publicidad) consumiendo el flujo | No implementado en esta entrega | Pendiente (mejora futura) |
| Filtro de entrada acotado a un solo remitente (limitación de la demo) | Documentado como decisión de alcance para la demo | Pendiente para producción real |
| Reprocesamiento en bucle si el correo permanece sin leer entre polls | Nodo de Gmail para marcar el mensaje como leído inmediatamente después de capturarlo | Implementado |
