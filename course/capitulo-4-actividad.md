# Actividad del Capítulo 4. Diseño de procesamiento asíncrono

## Propósito

Transformar una operación larga de la API de documentos en un flujo asíncrono,
seleccionando el servicio de mensajería adecuado y definiendo entrega,
reintentos, idempotencia y recuperación ante errores. La actividad no requiere
crear recursos reales.

## Caso de aplicación

Después de cargar un documento, la API debe iniciar un procesamiento que:

1. lee el blob;
2. valida tipo y tamaño;
3. extrae metadata;
4. actualiza el estado del documento;
5. notifica a otros componentes que el procesamiento terminó.

El procesamiento puede tardar varios minutos y la API no debe mantener abierta
la solicitud HTTP durante todo ese tiempo. Puede haber dos consumidores
futuros: uno para clasificación y otro para notificación. Ocasionalmente el
servicio de almacenamiento responde con timeout y algunos documentos tienen
formato inválido.

## Entregable

Crea un documento Markdown en `course/` llamado
`capitulo-4-entrega.md`. Incluye:

1. **Flujo general:** diagrama Mermaid o ASCII desde la solicitud HTTP hasta
   los consumidores y el estado final.
2. **Decisión de mensajería:** comparación de Service Bus, Event Grid y Queue
   Storage, seguida de una recomendación para cada flujo.
3. **Contrato de mensajes:** campos, identificador de operación, versión,
   referencias al blob, productor, consumidor y datos que deliberadamente no
   se incluyen.
4. **Diseño de Azure Functions:** triggers, entradas, salidas, permisos y
   secuencia de procesamiento.
5. **Estados y confirmación:** transición entre recibido, procesando,
   completado, rechazado y pendiente de recuperación.
6. **Reintentos:** errores reintentables, máximo de intentos, backoff, timeout
   y destino después del límite.
7. **Idempotencia:** estrategia para impedir que una reentrega duplique
   metadata, clasificación o notificaciones.
8. **Errores y DLQ:** tratamiento de formato inválido, timeout, permisos
   insuficientes y fallo después de actualizar el estado.
9. **Observaciones:** supuestos, límites y métricas o alertas necesarias.

## Pasos

1. Separa el evento “documento cargado” del trabajo “procesar documento”.
   Explica qué componente publica cada mensaje.
2. Decide si usarías Service Bus queue, Service Bus topic, Event Grid, Queue
   Storage o una combinación. Justifica por entrega, consumidores, filtros,
   DLQ, costo y complejidad.
3. Define un mensaje pequeño y versionado. Incluye `operation_id`,
   `document_id`, `blob_reference` y `schema_version`; no incluyas secretos ni
   el contenido binario.
4. Diseña los triggers de Functions y especifica cuándo una función confirma o
   elimina el mensaje.
5. Clasifica los fallos del caso como transitorios, permanentes del mensaje,
   permanentes del entorno o de negocio.
6. Define una política de reintentos para el timeout de almacenamiento y una
   ruta de rechazo para formato inválido.
7. Describe cómo registrarías una clave de idempotencia antes de actualizar la
   base de datos, cargar resultados o publicar una notificación.
8. Explica cómo recuperarías un mensaje desde DLQ sin volver a producir
   efectos duplicados.
9. Define pruebas para reentrega, dos consumidores, caída durante el
   procesamiento, mensaje inválido, timeout y error de autorización.

## Criterios de aceptación

La entrega es aceptable cuando:

- distingue evento, mensaje, cola, topic, subscription y consumidor;
- compara explícitamente Service Bus, Event Grid y Queue Storage;
- el diagrama muestra productores, consumidores, estado y camino de error;
- el contrato incluye identificador, versión y referencia al dato sin secretos;
- describe integración con Azure Functions mediante triggers o bindings;
- confirma o elimina mensajes solo después del efecto requerido;
- clasifica fallos y limita reintentos con backoff o política equivalente;
- incluye una estrategia concreta de idempotencia para cada efecto relevante;
- define DLQ, motivo de rechazo y procedimiento de recuperación;
- contempla reentrega, concurrencia, timeout y fallo parcial;
- incluye supuestos, límites y criterios de observación verificables;
- no implementa recursos, credenciales ni código de producción.

## Resultado esperado

Obtendrás un diseño ejecutable en una etapa posterior para procesar documentos
sin bloquear la API, con responsabilidades claras entre productor,
mensajería, funciones y consumidores. La propuesta debe hacer visible qué
ocurre cuando todo funciona y qué ocurre cuando falla.

## Preguntas abiertas para la revisión

- ¿El flujo necesita una cola de trabajo, distribución a varios consumidores o
  ambas?
- ¿Qué significa “completado” si clasificación termina pero notificación falla?
- ¿Dónde se conserva la clave de idempotencia y durante cuánto tiempo?
- ¿Qué tamaño y retención de DLQ justifican alertas o escalamiento?
- ¿Qué operación compensa un estado `procesando` abandonado?
- ¿Qué métricas demostrarían que reintentos están recuperando fallos sin crear
  una tormenta de mensajes?

## Fuera de alcance

No implementes recursos de Azure, workers containerizados, Kubernetes,
pipelines CI/CD ni dashboards de observabilidad. No diseñes todavía
autenticación de usuarios finales, contratos de API Management o modelos de IA.
