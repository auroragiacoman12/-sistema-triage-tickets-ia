# Matriz de Optimización de Costos

**Sistema de Triage de Tickets de Soporte con IA**
Fuente de precios: [Anthropic API — Pricing oficial](https://www.anthropic.com), verificado agosto 2026.

---

## 1. Modelos disponibles y su costo base

| Modelo | Costo entrada (por 1M tokens) | Costo salida (por 1M tokens) | Perfil |
|---|---|---|---|
| Claude Haiku 4.5 | $1.00 | $5.00 | Rápido y económico — ideal para tareas simples y de alto volumen |
| Claude Sonnet 4.5 | $3.00 | $15.00 | Balanceado — mejor calidad de redacción y razonamiento |
| Claude Opus (referencia) | $5.00 | $25.00 | Máxima capacidad — usado aquí solo como punto de comparación |

---

## 2. Qué modelo usa cada tarea del flujo, y por qué

| Tarea | Modelo elegido | Justificación |
|---|---|---|
| **Clasificación del ticket** (categoría + urgencia) | **Claude Haiku 4.5** | Es una tarea de clasificación cerrada: el modelo elige entre opciones predefinidas y devuelve un JSON corto. No requiere razonamiento profundo ni redacción elaborada — usar un modelo más caro aquí no mejora el resultado, solo incrementa el costo. Se limitó `max_tokens=100` porque la respuesta esperada es mínima. |
| **Redacción de la respuesta sugerida al cliente** | **Claude Sonnet 4.5** | Esta tarea sí requiere calidad de lenguaje: tono profesional, coherencia, adaptación al contexto del mensaje original. Un texto mal redactado llega directo a un ser humano (el cliente), así que aquí se prioriza calidad sobre costo. Se limitó `max_tokens=200` para controlar la longitud de la respuesta (máx. ~100 palabras solicitadas en el prompt). |

Esta decisión de arquitectura — **usar dos modelos con roles distintos según la complejidad real de la tarea** — es el principio central de esta matriz: no todo el flujo necesita el modelo más potente disponible.

---

## 3. Costo real por ticket procesado

Estimación basada en el uso real observado durante las pruebas del sistema (longitud típica de prompts y respuestas).

### 3.1 Clasificación (Claude Haiku 4.5)

| Concepto | Tokens aprox. | Costo |
|---|---|---|
| Entrada (prompt + Asunto + Mensaje) | ~180 tokens | $0.00018 |
| Salida (JSON de categoría/urgencia) | ~30 tokens | $0.00015 |
| **Subtotal clasificación** | | **$0.00033** |

### 3.2 Redacción de respuesta (Claude Sonnet 4.5)

| Concepto | Tokens aprox. | Costo |
|---|---|---|
| Entrada (prompt + contexto del ticket) | ~200 tokens | $0.00060 |
| Salida (respuesta redactada, ~100-150 palabras) | ~150 tokens | $0.00225 |
| **Subtotal redacción** | | **$0.00285** |

### 3.3 Costo total por ticket

```
$0.00033 (clasificación) + $0.00285 (redacción) ≈ $0.0032 USD por ticket
```

---

## 4. Comparación: arquitectura actual vs. usar un solo modelo para todo

Para demostrar el impacto real de esta decisión, se compara el costo de procesar **10,000 tickets** (volumen mensual estimado para una institución mediana) bajo tres escenarios:

| Escenario | Costo por ticket | Costo mensual (10,000 tickets) | Diferencia vs. actual |
|---|---|---|---|
| **Arquitectura actual** (Haiku para clasificar + Sonnet para redactar) | $0.0032 | **$31.80** | — (línea base) |
| Usar solo Sonnet para todo (clasificar y redactar) | $0.0038 | $38.40 | +21% más costoso |
| Usar solo Opus para todo (clasificar y redactar) | $0.0064 | $64.00 | **+101% más costoso** |

**Conclusión:** la arquitectura de dos modelos genera un ahorro aproximado del **50% frente a usar el modelo más capaz (Opus) en todo el flujo**, sin sacrificar calidad donde importa (la redacción sigue usando Sonnet). El ahorro se vuelve más significativo a mayor volumen — en un escenario de 100,000 tickets/mes, la diferencia frente a un enfoque "todo con el modelo más potente" sería de aproximadamente **$322 USD/mes**.

---

## 5. Optimizaciones adicionales no implementadas (trabajo futuro)

Se documentan aquí como decisiones conscientes de alcance, no como omisiones:

| Optimización | Por qué no se implementó en esta entrega | Ahorro potencial estimado |
|---|---|---|
| **Batch API** (procesamiento por lotes, 50% más barato) | El flujo requiere respuesta síncrona: el ticket debe clasificarse y la respuesta debe estar lista antes de notificar al humano para aprobación. Batch API introduce latencia (procesamiento asíncrono, no en tiempo real), lo cual rompe la experiencia de Human-in-the-Loop. Sería aplicable si se separara un proceso de reclasificación masiva histórica, fuera del flujo en tiempo real. | ~50% sobre los tickets reprocesados en lote |
| **Prompt caching** | El prompt de clasificación reutiliza la misma instrucción base en cada ticket (solo cambian Asunto/Mensaje). Cachear esa porción fija del prompt reduciría hasta 90% el costo de esa parte del input. No se implementó por el bajo volumen de esta entrega, pero es la primera optimización recomendada al escalar a producción. | Hasta 90% del costo de entrada en la porción fija del prompt |

---

## 6. Resumen ejecutivo

- El sistema usa **Claude Haiku 4.5** para la tarea de clasificación (categórica, de bajo margen de error, alto volumen) y **Claude Sonnet 4.5** para la redacción de respuestas (donde la calidad de lenguaje impacta directamente al cliente final).
- Costo real observado: **~$0.0032 USD por ticket procesado**.
- Esta arquitectura de dos modelos genera un ahorro estimado del **50% frente a usar un único modelo de alta capacidad (Opus) para todo el flujo**, sin comprometer la calidad de la respuesta final.
- Se identificaron dos optimizaciones adicionales (Batch API y Prompt Caching) como mejoras de producción fuera del alcance de esta entrega, documentadas explícitamente como decisiones de diseño.
