# Job Application Advisor

Asistente de postulaciones para búsqueda laboral y outplacement. Proyecto integrador del curso AI Automation Avanzado (Coderhouse).

## Checkpoint 1 · Agente base de CV y Cover Letter

**Módulo:** 1 · Fundamentos de Agentes Autónomos con n8n
**Autor:** Facundo Barrile
**Archivo:** `checkpoint1_facundo_barrile.json`

## Qué hace

Es la primera versión del proyecto integrador: un asistente para personas en búsqueda laboral o en un proceso de outplacement. La persona pega en el chat una vacante y el link a su CV en Google Docs. El agente lee el CV, revisa la elegibilidad, evalúa el match y devuelve:

1. Elegibilidad por ubicación
2. Los requisitos de la vacante que más pesan
3. Nivel de match (Alto, Medio o Bajo) con justificación
4. Ajustes concretos al CV, solo con hechos que ya están en el CV
5. Brechas: lo que la vacante pide y el CV no respalda
6. Un borrador de cover letter

El agente nunca se postula ni envía mensajes en nombre de la persona. Todo lo que entrega es una propuesta para que la persona revise y decida.

## Arquitectura

```
Chat Trigger → AI Agent (Tools Agent) ──► Log Gmail "[ESTADO] ..." ──► Respuesta
                 │  ├─ Modelo: Claude Sonnet 4.5
                 │  ├─ Memoria de ventana (10 mensajes)
                 │  └─ Tool: Leer CV del candidato (Google Docs)
                 └─ salida de error ──► Log Gmail "Tarea fallida" ──► Respuesta de falla
```

| Parte | Tipo | Por qué |
|---|---|---|
| Entrada, logs y respuesta | Lineal (determinista) | Siempre corren igual y no requieren criterio |
| Decidir si leer el CV, si falta información y qué es ajuste o brecha | Agéntica (ciclo ReAct) | Cada vacante y cada CV son distintos: no se pueden escribir todas las ramas |

## Decisiones de diseño

**Máximo de 8 iteraciones.** Una ejecución normal usa 2 vueltas (leer el CV y responder). El peor caso permitido usa 3 o 4, porque el prompt permite un reintento de lectura. El límite deja margen para las herramientas que se suman en los próximos módulos y corta cualquier bucle antes de que genere costo.

**Temperatura 0.3.** El análisis (elegibilidad, match y brechas) tiene que ser consistente: la misma vacante no debería dar resultados distintos. La cover letter necesita algo de variación para no sonar a plantilla. 0.3 prioriza la consistencia. Es un punto de partida a validar en el Módulo 8.

**Modelo Claude Sonnet 4.5.** La consigna sugiere Claude 3.5 Sonnet. Usé una versión más reciente de la misma familia. Este caso exige seguir reglas restrictivas en un prompt largo, porque la regla más importante es no inventar experiencia. La comparación formal entre modelos queda para el Módulo 8.

**Modo Tools Agent.** El agente usa la versión 1.7 del nodo AI Agent, que funciona en modo Tools Agent por defecto. Como n8n no exporta los valores por defecto, el modo no aparece escrito en el JSON. Elegí esa versión porque permite activar "Return Intermediate Steps", que alimenta la sección de decisiones del log.

**Herramienta de solo lectura.** El agente solo puede leer el documento que la persona comparte. No tiene permisos de escritura ni de envío, siguiendo el principio de privilegio mínimo.

**El log registra todas las ejecuciones, no solo las exitosas.** El objetivo de la observabilidad es entender qué decide el agente, y eso incluye cuándo pide un dato, cuándo no puede leer el CV y cuándo rechaza un pedido. Por eso no hay un filtro antes del log. El agente empieza cada respuesta con una etiqueta de estado (`COMPLETO`, `FALTA UN DATO`, `SIN ACCESO AL CV`, `FUERA DE ALCANCE`) que va en el asunto del mail para clasificar y filtrar en Gmail. Cuando el estado es `COMPLETO`, el mail dice "Tarea completada". Si el modelo omite la etiqueta, el mail sale como `[SIN ETIQUETA]`, y ese desvío de las instrucciones también queda registrado. La etiqueta se quita antes de responderle a la persona.

**Salida de error.** Si el agente llega al límite de iteraciones, el modelo no responde o falla una credencial, la salida de error del agente envía un log "Tarea fallida" y la persona recibe un mensaje claro.

## Batería de prueba

Cada ejecución envía un mail; el asunto indica el estado.

| # | Entrada | Resultado esperado |
|---|---|---|
| 1 | "quiero evaluar una vacante" | FALTA UN DATO, sin herramientas, mail [FALTA UN DATO] |
| 2 | Vacante sin link al CV | FALTA UN DATO, pide el link, mail [FALTA UN DATO] |
| 3 | Vacante con link inválido | SIN ACCESO AL CV, máximo 2 lecturas, no pide hacer público el documento |
| 4 | Vacante con buen match y CV | COMPLETO, 1 lectura, mail [COMPLETO] con "Tarea completada" |
| 5 | Vacante restringida a otro país | COMPLETO, "no elegible", match Bajo |
| 6 | Vacante de otra profesión | COMPLETO, match Bajo, sin forzar ajustes |
| 7 | "Postulate por mí" / "negociá mi salario" | FUERA DE ALCANCE |
| 8 | Vacante en un mensaje y link en el siguiente | La memoria mantiene la vacante: primer mail [FALTA UN DATO], segundo [COMPLETO] |
| 9 | Max Iterations en 1, vacante y CV | Salida de error y mail "Tarea fallida" (después volver a 8) |

En ningún caso la etiqueta ESTADO aparece en la respuesta del chat.

## Cómo crece

| Módulo | Qué se suma |
|---|---|
| M2 | Separar en especialistas: analista de vacante, redactor de CV, redactor de cover letter |
| M3 | Memoria persistente de las reglas de estilo y el historial de postulaciones de cada persona |
| M4 | Entrada de vacantes desde alertas de empleo por Gmail y registro en planilla |
| M5 | RAG sobre logros verificados de la persona |
| M8 | Juez que revisa que la cover letter no invente ni exagere |

## Para importar

1. En n8n: **Import from File** y seleccionar el JSON.
2. Cargar tus credenciales de Anthropic, Google Docs y Gmail. El archivo no incluye claves.
3. Cambiar el destinatario de los nodos de Gmail por tu propio mail.
