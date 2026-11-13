# Actividad del Capítulo 9. Diseñar una solución RAG segura y evaluable

## Propósito

Diseñar una API Python que responda preguntas sobre documentos autorizados
usando Azure OpenAI, Azure AI Search y Azure AI Foundry. La actividad integra
datos, identidad, búsqueda, generación, evaluación, costos, monitoreo y
despliegue sin exigir recursos reales.

## Caso de aplicación

La aplicación de documentos debe ofrecer una función de preguntas y respuestas:

- cada usuario solo puede consultar documentos de su organización;
- los documentos se almacenan en Blob Storage y sus metadatos en la
  persistencia definida anteriormente;
- el texto se divide e indexa en Azure AI Search;
- la API recupera contexto autorizado y solicita una respuesta a Azure OpenAI;
- la respuesta debe incluir fuentes o declarar que no existe evidencia
  suficiente;
- el equipo quiere evaluar una nueva versión de prompt y controlar costo;
- la solución debe ejecutarse en `dev` y promoverse a `test` sin secretos en la
  imagen.

No se proporcionan modelos, cuota, volumen, región, datos reales ni criterios
de calidad ya medidos. Declara supuestos y decisiones pendientes.

## Entregable

Crea un documento Markdown en `course/` llamado
`capitulo-9-entrega.md`. Incluye:

1. **Arquitectura:** diagrama de API, identidad, Blob Storage, indexación,
   Azure AI Search, Azure OpenAI, AI Foundry y usuario.
2. **Modelo y deployment:** modelo elegido, nombre lógico del deployment,
   parámetros, límites, versión y manejo de rate limits.
3. **Pipeline de indexación:** extracción, chunks, embeddings, metadata,
   permisos, índice, indexer, skillset y actualización o eliminación.
4. **Flujo RAG:** pregunta, autenticación, filtros, búsqueda, prompt, fuentes,
   generación, validación y respuesta.
5. **Contrato de respuesta:** formato esperado, citas, ausencia de evidencia,
   errores y fallback.
6. **Seguridad:** identidad, permisos mínimos, Key Vault, filtrado por tenant,
   prompt injection, datos sensibles y acciones que el modelo no puede ejecutar.
7. **Costos y límites:** tokens, top-k, longitud, cuota, rate limiting,
   cache, presupuesto y alertas.
8. **Evaluación en AI Foundry:** conjunto de casos, métricas de recuperación,
   grounding, corrección, seguridad, latencia y costo.
9. **Monitoreo y despliegue:** métricas, logs, trazas, versiones, promoción,
   rollback y compatibilidad entre prompt, índice y modelo.

## Pasos

1. Define quién pregunta, qué fuentes puede consultar y qué debe ocurrir si
   no existe evidencia suficiente.
2. Elige un deployment de Azure OpenAI y documenta parámetros, límites,
   timeout, reintentos y respuesta ante rate limit.
3. Diseña chunks y metadata. Incluye `document_id`, organización, versión,
   fuente, sección y permisos necesarios para filtrar.
4. Define índice de Azure AI Search con campos textuales, vectoriales y de
   filtro. Explica qué harían indexer y skillset.
5. Describe el flujo de consulta híbrida o vectorial. Aplica autorización
   antes de entregar contexto al prompt.
6. Redacta la estructura de prompt sin usar documentos reales. Separa
   instrucciones, contexto no confiable y pregunta del usuario.
7. Define un esquema de respuesta con texto, fuentes, estado de evidencia y
   errores. Indica cómo validarías la salida.
8. Diseña controles ante prompt injection y explica qué herramientas o acciones
   quedan prohibidas para el modelo.
9. Estima factores de costo y define límites por solicitud, usuario, tenant y
   ambiente.
10. Crea un conjunto de evaluación con casos normales, ambiguos, sin respuesta,
    permisos cruzados y ataques de instrucciones. Define métricas y umbrales.
11. Define telemetría y despliegue: versiones de prompt, modelo, índice y
    código; pruebas; canary; observación; rollback.

## Criterios de aceptación

La entrega es aceptable cuando:

- distingue modelo, deployment, inferencia, tokens y parámetros;
- integra Azure OpenAI con una API Python mediante identidad o secreto
  protegido;
- describe embeddings, chunks, búsqueda semántica, híbrida y filtros;
- incluye Azure AI Search con índice, indexer, skillset y consultas
  vectoriales;
- explica un flujo RAG completo con fuentes y comportamiento sin evidencia;
- mantiene autorización por usuario u organización antes de generar contexto;
- incluye protección contra prompt injection y limita acciones del modelo;
- usa Azure AI Foundry para versiones, casos y evaluación;
- mide recuperación, grounding, corrección, seguridad, latencia y costo;
- define límites de tokens, cuota, rate limiting, reintentos y alertas;
- incluye monitoreo y despliegue con versiones de prompt, modelo e índice;
- contempla rollback y compatibilidad de datos e índices;
- no incluye datos reales, secretos, llamadas a modelos ni recursos creados.

## Resultado esperado

Obtendrás una arquitectura de RAG que puede implementarse de forma segura y
medirse antes de exponerse a usuarios. La propuesta debe dejar claro qué
evidencia recibe el modelo, qué permisos se aplican antes de recuperarla,
cómo se evalúa la respuesta y cómo se controla el costo.

## Preguntas abiertas para la revisión

- ¿Qué métrica indica que la falla está en recuperación y no en generación?
- ¿Cómo se elimina del índice un documento que ya no está autorizado?
- ¿Qué ocurre si el modelo responde con una cita inexistente?
- ¿Qué límite de tokens mantiene calidad y costo dentro del presupuesto?
- ¿Qué versión debe revertirse si cambia el prompt pero no la imagen?
- ¿Qué casos de evaluación representan mejor consultas reales y adversariales?

## Fuera de alcance

No crees deployments, índices, proyectos, claves ni recursos de Azure. No uses
documentos reales, datos personales o secretos. No construyas una interfaz
completa, agentes con acciones externas ni un pipeline de producción; el
objetivo es diseñar y evaluar el flujo de IA.
