# Actividad del Capítulo 5. Containerizar y desplegar API y worker

## Propósito

Empaquetar conceptualmente los componentes del caso de documentos como
contenedores y diseñar su despliegue en Azure Container Apps. La actividad
aplica Docker, ACR, configuración, revisiones, escalamiento y versionado sin
requerir recursos reales.

## Caso de aplicación

La solución tiene dos componentes:

- una API Python que recibe documentos y consulta su estado;
- un worker Python que consume mensajes de procesamiento y actualiza la
  persistencia definida en capítulos anteriores.

La API debe recibir tráfico HTTP variable. El worker debe escalar según la
cantidad de mensajes pendientes y no debe exponerse públicamente. Ambos
componentes comparten algunas dependencias Python, pero necesitan permisos,
variables y ciclos de vida distintos.

## Entregable

Crea un documento Markdown en `course/` llamado
`capitulo-5-entrega.md`. Incluye:

1. **Diseño de imágenes:** Dockerfile o pseudodockerfile para API y worker,
   imagen base, dependencias, usuario, comando y puerto cuando aplique.
2. **Contexto de construcción:** `.dockerignore` propuesto y lista de archivos
   que nunca deben entrar en la imagen.
3. **Registro:** nombres de repositorios ACR, tags, digest esperado, permisos
   de push/pull y política de retención.
4. **Arquitectura de Container Apps:** environment, API, worker, ingreso,
   configuración, secretos e identidades.
5. **Escalamiento:** mínimo, máximo y señal para cada componente, con límites
   de CPU, memoria y dependencias.
6. **Revisiones:** estrategia blue-green o canary, validaciones antes de
   promover y procedimiento de rollback.
7. **Flujo de despliegue:** pasos desde código hasta una prueba HTTP y un
   mensaje procesado.
8. **Comparación:** justificación de Container Apps frente a App Service y
   Functions para cada componente.
9. **Riesgos y supuestos:** imágenes vulnerables, secretos, costos, arranque,
   duplicados, pérdida de estado y límites.

## Pasos

1. Separa responsabilidades de API y worker. Indica por qué no deben compartir
   automáticamente ingreso, identidad, escala o permisos.
2. Define una imagen reproducible para cada proceso. Incluye runtime, comando
   de inicio, puerto de la API y comportamiento de terminación.
3. Especifica qué archivos excluirías con `.dockerignore` y cómo evitarías que
   secretos lleguen a una capa de la imagen.
4. Define tags basados en versión o commit y explica cuándo usarías digest.
   Indica qué imágenes conservarías para rollback.
5. Diseña dos Container Apps dentro de un environment. Describe ingreso
   público o interno, variables, secretos, identidad y acceso a ACR.
6. Elige señales de escalamiento para API y worker. Relaciona máximos con
   límites de base de datos, Blob Storage y mensajería.
7. Diseña una revisión nueva para una versión del worker. Define pruebas de
   salud, procesamiento, reentrega e idempotencia antes de promover.
8. Simula un fallo de la nueva imagen y describe rollback, incluyendo qué
   comprobarías si ya hubo cambios en datos o mensajes.
9. Redacta criterios que permitan a otra persona repetir el despliegue sin
   conocer decisiones implícitas.

## Criterios de aceptación

La entrega es aceptable cuando:

- distingue imagen, contenedor, Dockerfile, registro, tag y digest;
- define API y worker con comandos, puertos y responsabilidades claras;
- excluye secretos, caches, repositorios y archivos innecesarios del contexto;
- describe ACR con permisos de publicación y extracción separados;
- usa configuración externa y almacenamiento persistente fuera del contenedor;
- define Container Apps, environment, ingreso e identidades;
- propone escalamiento con señales, mínimos, máximos y límites de dependencias;
- incluye revisiones, validación, promoción y rollback;
- compara App Service, Functions y Container Apps para las cargas del caso;
- contempla duplicados del worker, salud, arranque y pérdida de una instancia;
- identifica riesgos de cadena de suministro, seguridad y costo;
- no requiere Kubernetes, AKS, CI/CD ni recursos reales.

## Resultado esperado

Obtendrás una propuesta reproducible para publicar una API y un worker
containerizados, con separación de responsabilidades y una estrategia segura
de promoción. La entrega debe demostrar que la imagen, el registro y la
plataforma de ejecución forman un flujo versionado de extremo a extremo.

## Preguntas abiertas para la revisión

- ¿La API necesita un entorno compartido con el worker o pueden aislarse más?
- ¿Qué ocurre si el worker escala a cero mientras hay mensajes pendientes?
- ¿Qué permisos exactos necesita cada identidad para extraer imágenes y usar
  persistencia?
- ¿Qué evidencia permite promover una revisión sin duplicar trabajo?
- ¿Qué cambios de esquema de datos impedirían un rollback simple?
- ¿Qué costo máximo y qué latencia de arranque son aceptables?

## Fuera de alcance

No construyas ni publiques imágenes reales, no configures ACR o Container Apps y
no implementes pipelines. No uses secretos, no introduzcas Kubernetes, AKS,
Ingress, HPA ni detalles de operación de clúster; esos temas pertenecen al
Capítulo 6.
