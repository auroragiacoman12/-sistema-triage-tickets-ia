# Sistema de Triage de Tickets de Soporte con IA

Sistema de automatización que recibe tickets de soporte por correo, los clasifica automáticamente con IA, redacta una respuesta sugerida, y la envía al cliente solo después de la aprobación de un humano.

**Curso:** Building Your AI Automation Ecosystem — Entrega Final
**Stack:** n8n (self-hosted) · Gmail · Airtable · Anthropic API (Claude Haiku 4.5 / Sonnet 4.5)

---

## 📋 Descripción del caso de uso

Un ticket llega por correo → un modelo de IA económico (Claude Haiku) lo clasifica por categoría y urgencia → un modelo de mayor calidad (Claude Sonnet) redacta una respuesta sugerida → un humano revisa y aprueba o rechaza esa respuesta antes de que se envíe realmente al cliente. Todo el proceso queda registrado en Airtable, con manejo de errores y un dashboard de control con KPIs en tiempo real.

## 🏗️ Arquitectura

```
Gmail Trigger → Create record (Airtable)
      ↓
Schedule + Search records (detecta pendientes)
      ↓
Claude Haiku (clasifica) → Update record
      ↓
Claude Sonnet (redacta respuesta)
      ↓
Human-in-the-Loop (Gmail: Aprobar / Desaprobar)
      ↓
   ┌──┴──┐
Aprobado  Rechazado
   ↓         ↓
Envío al   Notificación
cliente    interna
```

Ver el diagrama completo en [`diagrama_arquitectura.pdf`](./diagrama_arquitectura.pdf).

## 📁 Contenido de este repositorio

| Archivo | Descripción |
|---|---|
| [`diagrama_arquitectura.pdf`](./diagrama_arquitectura.pdf) | Mapa visual completo del flujo: triggers, routers, APIs, nodos de IA y destino de los datos |
| [`manual_operativo_datos.md`](./manual_operativo_datos.md) | Esquema de la base de datos (Airtable) y los JSON de transferencia entre cada integración |
| [`matriz_costos.md`](./matriz_costos.md) | Justificación de qué modelo de IA se usa por tarea y comparación de costos |
| [`seguridad_resiliencia.md`](./seguridad_resiliencia.md) | Minimización de datos, manejo de errores, y puntos de Human-in-the-Loop |
| [`workflow_n8n.json`](./workflow_n8n.json) | Blueprint técnico del flujo, exportado directamente de n8n |

## 🔗 Enlaces

| Recurso | Enlace |
|---|---|
| Dashboard de control (KPIs y tasa de errores) | [Ver en Airtable Interfaces](https://airtable.com/appSuLAfznvdFKpCI/pagpJ5bBGHbiUlj49) |
| Base de datos (modo lectura) | [Ver base completa](https://airtable.com/appSuLAfznvdFKpCI/shr7lmiFop6sArm9H) |
| Video demo (3 min) | [Ver video](https://drive.google.com/file/d/1y8MP31pShHlsL4emzG9xyZjzW8mW_bz3/view?usp=sharing) |

## ⚙️ Componentes técnicos

- **Trigger inteligente:** Gmail Trigger con polling cada minuto y filtro anti-bucle (`from:aurora.giacoman@udem.edu -subject:"Re:"`) para evitar reprocesar las propias respuestas del sistema. *Nota: para esta entrega, el filtro se acotó a un remitente específico como medida de control durante las pruebas y la grabación del video; en producción se ampliaría para aceptar cualquier remitente externo (ver `seguridad_resiliencia.md`).*
- **Motor de IA de dos niveles:** Claude Haiku 4.5 para clasificación (tarea simple, bajo costo) y Claude Sonnet 4.5 para redacción de respuestas (mayor calidad de lenguaje).
- **Resiliencia:** reintentos automáticos (3x) + registro centralizado de errores (`LogError`) en los nodos críticos.
- **Human-in-the-Loop:** ninguna respuesta generada por IA se envía al cliente sin aprobación humana explícita.
- **Dashboard de control:** KPIs (total de tickets, distribución por estado, tasa de errores) publicados como Shared View en Airtable Interfaces.

## 🧪 Pruebas de estrés realizadas

- Camino feliz completo (ticket → clasificación → aprobación → envío)
- Rama de rechazo (ticket → clasificación → desaprobación → notificación interna)
- Camino infeliz: correo de rebote automático (*Delivery Status Notification*) procesado sin romper el flujo
- Simulación realista cliente-soporte (correo institucional → bandeja de soporte)
- Verificación del filtro anti-bucle en modo de ejecución real

---

*Proyecto desarrollado como entrega final del curso de automatización con IA.*
