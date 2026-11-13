# Capítulo 9. Desarrollo de soluciones con inteligencia artificial en Azure

## Propósito de lectura

Al finalizar esta lectura podrás integrar un modelo generativo de Azure
OpenAI en una aplicación Python, distinguir modelo, deployment e inferencia,
crear un flujo de embeddings y búsqueda semántica con Azure AI Search, y
explicar una arquitectura Retrieval-Augmented Generation (RAG). También
podrás ubicar proyectos, modelos y evaluaciones en Azure AI Foundry, y definir
controles de seguridad, costo, límites, monitoreo y despliegue.

El capítulo cierra el recorrido del backend, la persistencia, la mensajería,
los contenedores, la seguridad y la operación. Una aplicación de IA no es solo
una llamada a un modelo: requiere datos recuperables, contratos, controles de
acceso, evaluación y una forma de detectar degradación.

## 1. Fundamentos de Azure OpenAI

**Azure OpenAI** ofrece acceso administrado a modelos de inteligencia
artificial generativa mediante recursos, deployments y APIs de Azure. Un
**modelo** es el sistema entrenado que transforma entradas en salidas. Un
**deployment** es una instancia configurada de un modelo a la que la
aplicación dirige solicitudes. El deployment puede definir nombre, versión,
capacidad y límites de consumo.

La **inferencia** es la ejecución del modelo para producir una salida a partir
de una entrada. En un modelo de lenguaje, la entrada suele incluir mensajes o
texto y la salida contiene tokens generados. Los **tokens** son unidades de
texto procesadas por el modelo; su cantidad afecta latencia, límites y costo.

Un **prompt** es la entrada que orienta la generación. Puede incluir
instrucciones del sistema, solicitud del usuario, contexto recuperado y
restricciones de formato. Un prompt bien diseñado define objetivo, límites,
fuentes permitidas y comportamiento ante información ausente. No debe
confiar en que el modelo obedecerá siempre instrucciones contradictorias o
rechazará por sí mismo contenido no autorizado.

Los modelos generativos pueden producir texto plausible pero incorrecto. La
aplicación debe tratar la salida como una respuesta que necesita validación,
no como una base de verdad. Para operaciones críticas, conviene estructurar,
verificar o revisar la salida antes de ejecutar efectos.

## 2. Modelos, deployments e inferencia mediante APIs

La aplicación llama un endpoint de Azure con una credencial autorizada,
identificador de deployment, versión de API y parámetros de inferencia. El
**endpoint** identifica el recurso; el nombre del deployment identifica la
configuración del modelo dentro de ese recurso.

Parámetros habituales:

- **temperature:** controla variabilidad; valores mayores pueden producir
  respuestas más diversas;
- **top_p:** limita la distribución de selección de tokens;
- **max output tokens:** limita tamaño de salida;
- **stop sequences:** detienen generación ante secuencias definidas;
- **response format:** solicita una forma estructurada cuando el modelo y API
  lo soportan.

Estos parámetros no garantizan exactitud ni una salida válida. El código debe
establecer timeouts, límites de entrada y salida, manejo de errores,
reintentos controlados y validación del resultado. Un retry indiscriminado
puede duplicar operaciones o aumentar el costo; las solicitudes deben ser
idempotentes cuando produzcan efectos externos.

Una integración segura:

1. obtiene credenciales mediante identidad administrada o secreto protegido;
2. selecciona el deployment desde configuración por ambiente;
3. valida y limita la entrada;
4. construye mensajes con instrucciones y contexto controlados;
5. llama al endpoint con timeout;
6. valida la salida y registra métricas sin contenido sensible;
7. responde o deriva a revisión humana si el resultado no cumple.

El deployment tiene cuotas y límites de capacidad. La aplicación debe manejar
respuestas de rate limit, saturación, timeout y errores de servicio sin revelar
la credencial ni convertir un fallo temporal en una cascada de solicitudes.

## 3. Integración de Azure OpenAI con Python

El **SDK para Python** proporciona clientes y modelos de datos para invocar
servicios de Azure. La integración debe encapsular configuración y cliente sin
crear una conexión o credencial nueva en cada solicitud innecesariamente.

Un componente de aplicación puede tener:

- configuración del endpoint y deployment;
- proveedor de credenciales;
- cliente reutilizable;
- constructor de mensajes;
- validador de respuesta;
- métricas de tokens, latencia, errores y costo estimado.

La aplicación debe separar instrucciones del sistema de contenido del usuario.
El contenido del usuario se considera no confiable: puede contener instrucciones
que intenten modificar el objetivo del sistema, un problema conocido como
**prompt injection**. Las instrucciones recuperadas de documentos tampoco deben
tratarse automáticamente como reglas de mayor prioridad.

Para salidas estructuradas, define un esquema y valida tipos, campos,
enumeraciones y límites. Si la salida no cumple, la aplicación puede solicitar
una corrección acotada, rechazarla o enviarla a revisión. No debe ejecutar
comandos, enviar dinero, cambiar permisos o publicar información solo porque el
modelo produjo una cadena con apariencia válida.

## 4. Embeddings y búsqueda semántica

Un **embedding** es una representación numérica de un texto en un espacio
vectorial. Textos con significado relacionado tienden a estar próximos según
la métrica y el modelo utilizados. La **búsqueda semántica** recupera contenido
por significado, no solo por coincidencia exacta de palabras.

Un proceso de indexación suele ser:

1. recolectar documentos autorizados;
2. extraer texto y conservar metadatos de origen;
3. dividir el texto en chunks con tamaño y solapamiento razonables;
4. generar embeddings para cada chunk;
5. almacenar texto, vector, identificador, fuente y filtros;
6. actualizar o eliminar versiones cuando cambia la fuente.

Un **chunk** es una unidad de texto que se indexa y recupera. Chunks demasiado
grandes incluyen ruido y consumen contexto; demasiado pequeños pierden
relaciones y contexto. La división debe respetar títulos, párrafos, tablas o
unidades de significado cuando sea posible.

La búsqueda vectorial calcula cercanía entre el embedding de la consulta y
los vectores indexados. La búsqueda híbrida combina coincidencia textual y
vectorial. La búsqueda semántica puede reordenar resultados con señales de
significado y lenguaje, pero no garantiza que el documento recuperado sea
correcto o autorizado.

Los filtros de seguridad deben aplicarse durante la recuperación. No es
suficiente recuperar todo y pedir al modelo que ignore documentos que el
usuario no puede ver. Los metadatos de tenant, usuario, clasificación o
permisos deben acompañar el índice y formar parte de la consulta.

## 5. Azure AI Search

**Azure AI Search** es un servicio de búsqueda que puede indexar contenido,
campos textuales, metadatos y vectores. Un **índice** define campos, tipos,
capacidades de búsqueda, filtros y configuración vectorial.

Un **indexer** es un componente que extrae datos de una fuente y actualiza un
índice según una programación o ejecución definida. Un **skillset** es una
secuencia de habilidades enriquecidas que transforma contenido durante la
indexación, como extracción, división o reconocimiento de información.

Una consulta puede combinar:

- búsqueda textual por términos;
- búsqueda vectorial por embedding;
- búsqueda híbrida;
- filtros por metadatos;
- selección de campos;
- cantidad de resultados;
- reordenamiento semántico cuando esté configurado.

El diseño del índice debe conservar trazabilidad: cada chunk debe incluir
origen, identificador de documento, versión, página o sección cuando aplique,
y permisos necesarios para filtrar. Si un documento se elimina o cambia, el
índice debe reflejarlo; un índice obsoleto puede provocar respuestas basadas
en información retirada.

Los indexers y skillsets no reemplazan el control de acceso de la aplicación.
Las credenciales de las fuentes deben estar protegidas, y el pipeline de
indexación debe evitar copiar secretos o datos que no deban entrar al índice.

## 6. Retrieval-Augmented Generation (RAG)

**Retrieval-Augmented Generation (RAG)** es un patrón que recupera información
relevante y la incluye como contexto de una solicitud al modelo generativo. El
modelo genera una respuesta condicionada por ese contexto, en lugar de
depender exclusivamente del conocimiento aprendido durante el entrenamiento.

Flujo básico:

1. el usuario envía una pregunta;
2. la aplicación autentica y autoriza la solicitud;
3. genera un embedding de la pregunta o construye una consulta híbrida;
4. busca chunks autorizados en Azure AI Search;
5. filtra y ordena resultados por relevancia;
6. construye un prompt con instrucciones, contexto y pregunta;
7. llama al deployment de Azure OpenAI;
8. valida la respuesta y devuelve citas o fuentes;
9. registra métricas y feedback sin exponer datos sensibles.

RAG puede reducir respuestas sin fundamento si recupera buenas fuentes, pero
no garantiza veracidad. Los fallos pueden originarse en extracción, chunks,
embeddings, filtros, ranking, prompt o generación. La respuesta debe mostrar
fuentes cuando sea posible y declarar cuando no hay contexto suficiente.

El **grounding** es el vínculo de la respuesta con evidencia recuperada. La
aplicación puede exigir que cada afirmación importante cite un chunk, rechazar
respuestas sin evidencia o permitir una respuesta limitada. No se debe
presentar información recuperada de un tenant como si estuviera disponible para
otro.

RAG tampoco sustituye entrenamiento de modelos ni una base transaccional. Es
adecuado para consultar conocimiento que cambia o que debe mantenerse fuera de
los parámetros del modelo. Los datos estructurados y operaciones de negocio
pueden requerir consultas directas y reglas deterministas.

## 7. Azure AI Foundry

**Azure AI Foundry** proporciona un entorno para organizar proyectos, modelos,
conexiones, evaluación y desarrollo de soluciones de IA. Un **proyecto** reúne
recursos y configuraciones de trabajo para un equipo o solución. Un **catálogo
de modelos** permite explorar modelos disponibles y sus capacidades, licencias,
modalidades y límites.

El desarrollo en Foundry debe distinguir:

- experimento de prototipo;
- deployment usado por una aplicación;
- conjunto de datos de evaluación;
- versión de prompt o configuración;
- métricas y criterios de aceptación;
- registro de versión promovida.

La plataforma puede ayudar a evaluar modelos y aplicaciones con datos
representativos. La evaluación debe medir el objetivo real, no solo la
fluidez. Para RAG, conviene separar recuperación de generación:

- **recall de recuperación:** si se recuperan fuentes relevantes;
- **precisión de recuperación:** proporción de resultados útiles;
- **groundedness:** si la respuesta está respaldada por el contexto;
- **corrección:** si responde correctamente;
- **relevancia:** si responde la pregunta sin desviarse;
- **seguridad:** si evita contenido o acciones no permitidas;
- **latencia y costo:** si cumple límites operativos.

Un conjunto de evaluación debe contener casos normales, ambiguos, sin
respuesta, adversariales, multilingües y con permisos distintos. Debe proteger
datos personales y documentar cómo se obtuvieron las etiquetas o juicios.

## 8. Seguridad para aplicaciones de IA

La seguridad de una aplicación de IA incluye identidad, datos, prompts,
respuestas, herramientas y registro. Deben aplicarse controles de los
capítulos anteriores:

- identidad administrada para llamar a OpenAI, Search y almacenamiento;
- mínimo privilegio por API, indexador, worker y operador;
- filtros de autorización antes de recuperar contexto;
- secretos en Key Vault o mecanismo equivalente;
- red privada y endpoints protegidos cuando corresponda;
- validación y límites de entrada y salida;
- protección contra prompt injection y exfiltración;
- revisión humana para acciones de alto impacto;
- redacción o exclusión de datos sensibles en telemetría.

La **prompt injection** intenta que contenido de usuario o documentos cambie
las instrucciones de la aplicación. La mitigación no consiste solo en añadir
una frase al prompt: se deben separar instrucciones de datos, filtrar
herramientas, validar autorización fuera del modelo y limitar efectos.

El modelo no debe tener permisos amplios para ejecutar operaciones. Un
asistente que propone una acción debe pasar por una capa determinista que
compruebe identidad, alcance, esquema, confirmación y reglas de negocio.

## 9. Control de costos y límites de consumo

El costo de una solución de IA depende de tokens de entrada y salida,
embeddings, consultas de búsqueda, almacenamiento, capacidad provisionada,
indexación y telemetría. Un prompt que incluye demasiado contexto puede
aumentar costo y reducir calidad.

Controles útiles:

- límite de tamaño de entrada y salida;
- selección y compresión de contexto;
- top-k razonable y filtros antes de recuperar;
- cache de respuestas seguras cuando aplique;
- cuotas por usuario, tenant y ambiente;
- rate limiting y backpressure;
- presupuesto y alertas;
- expiración o limpieza de índices de prueba;
- separación de modelos y deployments por carga;
- medición de costo por solicitud y flujo.

Los límites de tasa pueden devolver errores temporales. La aplicación debe
aplicar backoff con jitter y mostrar una respuesta controlada, no lanzar
reintentos ilimitados. Un límite de tokens de salida protege costo, pero puede
truncar una respuesta; el cliente debe detectar esa condición.

Optimizar costo no significa eliminar evaluación, seguridad o fuentes. Una
respuesta barata pero no fundamentada puede costar más en soporte, incidentes o
decisiones incorrectas.

## 10. Evaluación y monitoreo

La **evaluación offline** ejecuta una versión contra un conjunto fijo antes de
promoverla. La **evaluación online** observa uso real, feedback, errores,
latencia, costo y distribución de consultas. Ambas son necesarias: una nueva
versión puede mejorar una métrica de laboratorio y degradar un caso real.

Monitorear una aplicación de IA requiere medir:

- latencia por etapa: recuperación, modelo y respuesta;
- tokens de entrada y salida;
- errores y rate limits;
- resultados sin fuentes o con baja relevancia;
- calidad de recuperación y groundedness;
- costo por solicitud, usuario o tenant;
- rechazos de seguridad y prompt injection;
- uso de versiones de prompt, modelo e índice.

Los logs deben incluir identificadores de correlación, versión y métricas
agregadas, no necesariamente el prompt o respuesta completa. Si se conservan
textos para evaluación, deben tener controles de acceso, retención y
anonimización adecuados.

## 11. Despliegue de aplicaciones con IA

Una aplicación de IA puede desplegarse como API, worker o ambos en App Service,
Container Apps o AKS. El deployment debe versionar código, prompts,
configuración, índice, modelo/deployment y conjunto de evaluación.

El **prompt versionado** es parte del artefacto lógico: cambiar instrucciones
puede cambiar comportamiento aunque la imagen no cambie. La versión del índice
también importa: una respuesta puede variar por documentos y embeddings
distintos.

Una promoción segura puede ser:

1. validar código, esquema de salida y seguridad;
2. construir y publicar artefacto;
3. probar recuperación y generación offline;
4. desplegar a `dev`;
5. ejecutar pruebas de integración y casos adversariales;
6. promover una revisión o porcentaje pequeño;
7. observar calidad, costo, latencia y errores;
8. ampliar tráfico o revertir;
9. conservar versiones compatibles de modelo, prompt e índice.

Revertir la imagen no basta si el índice ya se actualizó o si cambió el
deployment del modelo. La compatibilidad entre versiones debe documentarse.

## 12. Método de aplicación

Para diseñar una solución RAG en Azure, usa esta secuencia:

1. **Define el caso:** pregunta, usuario, datos autorizados y acción
   esperada.
2. **Elige modelo y deployment:** capacidad, límites, modalidad, costo y
   configuración.
3. **Prepara fuentes:** extracción, chunks, metadata, permisos y versiones.
4. **Indexa:** embeddings, índice, indexer, skillset y actualización.
5. **Recupera:** búsqueda híbrida/vectorial, filtros de seguridad y ranking.
6. **Genera:** prompt versionado, contexto limitado, formato y fuentes.
7. **Valida:** esquema, grounding, seguridad, acción y fallback.
8. **Evalúa:** recuperación, respuesta, seguridad, latencia y costo.
9. **Despliega:** identidad, secretos, observabilidad, límites y promoción.
10. **Opera:** monitorea calidad, índice, modelo, costos y feedback; revierte
    cuando la evidencia lo indique.

## Cierre

Azure OpenAI proporciona inferencia generativa; Azure AI Search recupera
conocimiento indexado; RAG conecta fuentes con respuestas; Azure AI Foundry
organiza modelos, proyectos y evaluación. Una solución confiable necesita
filtros de autorización, control de costos, límites, telemetría, evaluación y
versionado de código, prompts, índices y deployments. Con este capítulo se
completa el recorrido desde fundamentos de Azure hasta una solución
cloud-native de IA operable y evaluable.
