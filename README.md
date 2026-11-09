# Job Application Advisor

Asistente de postulaciones para búsqueda laboral y outplacement. Proyecto integrador del curso AI Automation Avanzado (Coderhouse).

## Estado del proyecto

| Checkpoint | Módulo | Qué suma | Archivos |
|---|---|---|---|
| 1 | Agentes autónomos | Agente base que analiza una vacante contra el CV y escribe la cover letter | `checkpoint1_facundo_barrile.json` |
| 2 | Arquitectura multi-agente | Manager con Recepcionista + Workers Analista y Redactor (sub-workflows), log y manejo de fallos | Incluidos en `checkpoint-3/` (evolucionados) |
| 3 | Memoria, contexto y persistencia | Memoria persistente por sesión en Airtable, inyección con delimitadores y resumen automático | `checkpoint-3/` |


## Checkpoint 1 · Agente base de CV y Cover Letter

**Módulo:** 1 · Fundamentos de Agentes Autónomos con n8n
**Autor:** Facundo Barrile
**Archivo:** `checkpoint1_facundo_barrile.json`

### Qué hace

Es la primera versión del proyecto integrador: un asistente para personas en búsqueda laboral o en un proceso de outplacement. La persona pega en el chat una vacante y el link a su CV en Google Docs. El agente lee el CV, revisa la elegibilidad, evalúa el match y devuelve:

1. Elegibilidad por ubicación
2. Los requisitos de la vacante que más pesan
3. Nivel de match (Alto, Medio o Bajo) con justificación
4. Ajustes concretos al CV, solo con hechos que ya están en el CV
5. Brechas: lo que la vacante pide y el CV no respalda
6. Un borrador de cover letter

El agente nunca se postula ni envía mensajes en nombre de la persona. Todo lo que entrega es una propuesta para que la persona revise y decida.

### Arquitectura

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

### Decisiones de diseño

**Máximo de 8 iteraciones.** Una ejecución normal usa 2 vueltas (leer el CV y responder). El peor caso permitido usa 3 o 4, porque el prompt permite un reintento de lectura. El límite deja margen para las herramientas que se suman en los próximos módulos y corta cualquier bucle antes de que genere costo.

**Temperatura 0.3.** El análisis (elegibilidad, match y brechas) tiene que ser consistente: la misma vacante no debería dar resultados distintos. La cover letter necesita algo de variación para no sonar a plantilla. 0.3 prioriza la consistencia. Es un punto de partida a validar en el Módulo 8.

**Modelo Claude Sonnet 4.5.** La consigna sugiere Claude 3.5 Sonnet. Usé una versión más reciente de la misma familia. Este caso exige seguir reglas restrictivas en un prompt largo, porque la regla más importante es no inventar experiencia. La comparación formal entre modelos queda para el Módulo 8.

**Modo Tools Agent.** El agente usa la versión 1.7 del nodo AI Agent, que funciona en modo Tools Agent por defecto. Como n8n no exporta los valores por defecto, el modo no aparece escrito en el JSON. Elegí esa versión porque permite activar "Return Intermediate Steps", que alimenta la sección de decisiones del log.

**Herramienta de solo lectura.** El agente solo puede leer el documento que la persona comparte. No tiene permisos de escritura ni de envío, siguiendo el principio de privilegio mínimo.

**El log registra todas las ejecuciones, no solo las exitosas.** El objetivo de la observabilidad es entender qué decide el agente, y eso incluye cuándo pide un dato, cuándo no puede leer el CV y cuándo rechaza un pedido. Por eso no hay un filtro antes del log. El agente empieza cada respuesta con una etiqueta de estado (`COMPLETO`, `FALTA UN DATO`, `SIN ACCESO AL CV`, `FUERA DE ALCANCE`) que va en el asunto del mail para clasificar y filtrar en Gmail. Cuando el estado es `COMPLETO`, el mail dice "Tarea completada". Si el modelo omite la etiqueta, el mail sale como `[SIN ETIQUETA]`, y ese desvío de las instrucciones también queda registrado. La etiqueta se quita antes de responderle a la persona.

**Salida de error.** Si el agente llega al límite de iteraciones, el modelo no responde o falla una credencial, la salida de error del agente envía un log "Tarea fallida" y la persona recibe un mensaje claro.

### Batería de prueba

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

## Checkpoint 3 · Memoria persistente y resumen automático

**Módulo:** 3 · Memoria, Contexto y Persistencia
**Archivos:** `checkpoint-3/manager.json`, `checkpoint-3/worker-analista.json`, `checkpoint-3/worker-redactor.json`

### Qué hace

El asistente recuerda a cada persona entre conversaciones sin mezclar datos: el CV, si ya resolvió sus criterios, la última vacante analizada y un resumen de lo conversado. Quien vuelve no repite datos, y el Recepcionista la saluda por su nombre y retoma lo pendiente.

### Arquitectura

```
Chat → Identificar sesión → ¿Sesión válida? → Buscar memoria (Airtable, filtro por Session_ID) → ¿Lectura OK?
     → ¿Tiene memoria? ─ no → Crear memoria ─┐
                         └ sí ───────────────┴→ Memoria (bloque de contexto)
     → Recepcionista (memoria entre delimitadores) → Switch → Workers del Checkpoint 2
     → Preparar memoria → Guardar memoria → ¿Más de 5 intercambios?
         └ sí → Leer historial → Resumir (Claude Haiku 4.5, JSON estricto) → Guardar resumen
     → Respuesta final
```

**Memoria híbrida:**
- **Corta:** la ventana de 20 mensajes del Recepcionista, con la tarea en curso.
- **Larga:** una fila por sesión en Airtable, que sobrevive a reinicios.

### Esquema de la base (Airtable, tabla `Memoria`)

| Campo | Tipo | Contenido |
|---|---|---|
| Session_ID | Texto (primario) | sessionId del chat: toda lectura y escritura filtra por este valor |
| Fecha de actualización | Fecha y hora | Última escritura |
| Resumen consolidado | Texto largo (JSON) | `asunto_principal`, `puntos_clave`, `accion_requerida` |
| Estado del caso | Selección única | NUEVO, FALTA_DATO, VACANTE_ANALIZADA, CARTA_REDACTADA, FUERA_DE_ALCANCE, ERROR |
| Datos clave | Texto largo (JSON) | Nombre, links al CV y a los criterios, criterios resueltos, 5 datos de la última vacante y contadores |
| Intercambios | Número | Contador desde el último resumen |

**Nunca se guarda:** texto del CV, de la vacante ni de la carta, ni la conversación.

### Decisiones de diseño

- **Identidad por Session_ID.** No se puede suplantar porque lo genera el sistema, y se valida antes de usarlo en la fórmula de búsqueda. Límite: reconoce un navegador, no a una persona. En el Módulo 4 la misma columna recibe la identidad que verifica el canal (Telegram o Gmail).
- **Falla segura.** Si Airtable no responde, el flujo corta con una alerta y no responde sin contexto ni crea duplicados.
- **Se guardan links, no contenido.** El documento de la persona sigue siendo la fuente de verdad.
- **El resumen no vacía la memoria corta.** La vacante pegada vive solo ahí: si se vaciara, "escribime la carta" después del análisis la perdería.
- **El Analista devuelve `nombre_candidato`** tomado del encabezado del CV. Es un cambio aditivo en su contrato.

### Pruebas

11 pruebas, todas pasaron:

- **Aislamiento y recuperación:** sesión nueva, dos sesiones sin mezclarse, amnesia (sin memoria corta, recupera CV y criterios desde Airtable), Session_ID con comilla y falla de lectura de Airtable.
- **Contexto y resumen:** criterios que no se vuelven a preguntar, resumen al sexto intercambio e intento de inyección.
- **Regresión y limpieza:** carta después del análisis, revisión de la tabla y nombre desde el CV.

### Hallazgos

- **Always Output Data + salida de error emite por las dos ramas:** una caída de Airtable creaba filas duplicadas. Se resolvió con un IF sobre `$json.error`.
- **Sin agrupar, el historial llega como un ítem por mensaje,** y el resumen corría 6 veces. Se resolvió con Group Messages + Execute Once.
- **`listo` mezclaba "tengo los datos" con "hay que ejecutar":** un "gracias" volvía a escribir la carta. Se redefinió en el prompt del Recepcionista.
- **Abierto:** el resumen todavía guarda requisitos de la vacante. Próximo paso: filtrar el historial antes del modelo.

### Para importar el Checkpoint 3

1. Importar los tres JSON de `checkpoint-3/`.
2. En el Manager, apuntar `Llamar Analista` y `Llamar Redactor` a los Workers importados.
3. Crear en Airtable la tabla `Memoria` con los 6 campos, y un token con `data.records:read`, `data.records:write` y `schema.bases:read` limitado a esa base.
4. Cargar las credenciales de Anthropic, Airtable, Google Docs, Google Sheets y Gmail, y cambiar `tu-email@ejemplo.com` por tu mail en los nodos de Gmail.

## Cómo crece

| Módulo | Qué se suma |
|---|---|
| M2 | Hecho: Manager + Workers Analista y Redactor (el Editor de CV quedó para más adelante) |
| M3 | Hecho: memoria persistente por sesión, inyección con delimitadores y resumen automático |
| M4 | Entrada de vacantes desde alertas de empleo por Gmail y registro en planilla |
| M5 | RAG sobre logros verificados de la persona |
| M8 | Juez que revisa que la cover letter no invente ni exagere |

## Para importar el Checkpoint 1

1. En n8n: **Import from File** y seleccionar el JSON.
2. Cargar tus credenciales de Anthropic, Google Docs y Gmail. El archivo no incluye claves.
3. Cambiar `tu-email@ejemplo.com` por tu propio mail en los nodos de Gmail.
