# Actividad del Capítulo 2. Selección y preparación del modelo de ejecución

## Propósito

Diseñar el despliegue de un backend Python para decidir, con criterios
observables, si conviene usar Azure App Service o Azure Functions. La actividad
aplica estructura de servicio, despliegue, triggers, SDK, configuración y
separación por entorno sin requerir una suscripción real.

## Caso de aplicación

Continúa el caso del Capítulo 1: una aplicación Python expone una API REST para
administrar documentos. El equipo necesita estas capacidades:

1. recibir solicitudes HTTP para registrar y consultar documentos;
2. ejecutar una tarea nocturna que elimine archivos temporales vencidos;
3. acceder a la cuenta de almacenamiento definida para el entorno `dev`;
4. ejecutar el mismo código en `dev` y `test` con configuración diferente;
5. desplegar una versión verificable sin incluir secretos en el artefacto.

El equipo aún no ha decidido entre App Service y Functions. Puedes recomendar
una sola opción, o una combinación si justificas qué carga corresponde a cada
servicio.

## Entregable

Crea un documento Markdown en `course/` llamado
`capitulo-2-entrega.md`. Incluye:

1. **Decisión de arquitectura:** opción elegida para la API y la tarea
   nocturna, con una comparación breve entre App Service y Functions.
2. **Estructura de ejecución:** plan o aplicación de App Service, o funciones y
   sus triggers; incluye runtime, punto de entrada y comando de arranque cuando
   aplique.
3. **Contrato de la API:** tabla con al menos tres rutas, método HTTP, entrada,
   respuesta esperada y error principal.
4. **Diseño de triggers:** descripción del HTTP trigger y del timer trigger,
   incluidos frecuencia, entradas, salida y comportamiento ante repetición.
5. **Configuración por ambiente:** tabla para `dev` y `test` con variables no
   secretas, valores de ejemplo y validaciones. No incluyas credenciales reales.
6. **Acceso desde Python:** pseudocódigo o fragmento breve que muestre cómo
   usarías el Azure SDK y qué identidad o credencial requiere. No es necesario
   que sea ejecutable.
7. **Flujo de despliegue y verificación:** pasos desde código hasta una prueba
   HTTP o ejecución programada, incluyendo revisión de logs.
8. **Supuestos, riesgos y límites:** decisiones pendientes sobre duración,
   escalamiento, permisos, archivos y costo.

## Pasos

1. Clasifica cada carga como solicitud HTTP, tarea programada o trabajo
   persistente. Anota duración estimada, frecuencia y necesidad de estado.
2. Compara App Service y Functions para la API mediante al menos cinco
   criterios: modelo de ejecución, escalamiento, operación, configuración y
   costo o límites.
3. Decide si la API y la tarea nocturna usan el mismo servicio. Explica qué
   beneficios y riesgos tiene compartir modelo, plan o configuración.
4. Define el contrato mínimo de la API. Incluye validación de entrada y
   respuestas de error; no diseñes todavía el esquema de base de datos.
5. Describe el HTTP trigger y el timer trigger. Explica cómo evitarías que una
   ejecución repetida de limpieza cause resultados incorrectos.
6. Enumera variables de entorno para `dev` y `test`. Separa configuración de
   secretos y especifica qué debe ocurrir si falta una variable obligatoria.
7. Indica cómo usaría el backend el Azure SDK para acceder a Blob Storage.
   Distingue la credencial local de desarrollo de la identidad del entorno
   desplegado.
8. Escribe una secuencia de despliegue reproducible y una lista de pruebas de
   humo: arranque, endpoint válido, entrada inválida, acceso a almacenamiento,
   timer y revisión de logs.

## Criterios de aceptación

La entrega es aceptable cuando:

- recomienda App Service, Functions o una combinación con una justificación
  basada en la carga y no solo en preferencias;
- cubre API HTTP y tarea programada, con triggers y comportamiento descritos;
- define runtime, punto de entrada y comando de arranque donde sean
  pertinentes;
- presenta al menos tres rutas con entradas, respuestas y errores observables;
- separa configuración de `dev` y `test` sin incluir secretos;
- explica qué ocurre ante una variable obligatoria ausente;
- muestra la relación entre el Azure SDK, la credencial y los permisos del
  recurso;
- distingue archivos temporales o subidos de los archivos del artefacto
  desplegado;
- contiene pasos de despliegue y pruebas que otra persona pueda repetir;
- identifica al menos un supuesto, un límite de ejecución y un riesgo de
  seguridad, confiabilidad o costo;
- no exige todavía bases de datos, mensajería, contenedores, CI/CD ni
  observabilidad avanzada.

## Resultado esperado

Obtendrás una propuesta de ejecución técnicamente defendible para el backend
de documentos, con una ruta clara desde el código hasta una prueba de
despliegue. La entrega debe mostrar que el servicio elegido, la configuración,
la identidad y los triggers forman un solo diseño, no decisiones aisladas.

## Preguntas abiertas para la revisión

- ¿La duración y frecuencia reales de la limpieza caben en los límites del
  plan seleccionado?
- ¿La API necesita un proceso continuo o sus operaciones pueden expresarse como
  invocaciones independientes?
- ¿Qué sucede si dos ejecuciones del timer se solapan?
- ¿Qué permisos exactos necesita la identidad de la aplicación sobre Blob
  Storage?
- ¿Cómo se promoverá el artefacto sin mezclar valores de `dev` y `test`?
- ¿Qué evidencia confirmaría que el despliegue está atendiendo la versión
  correcta?

## Fuera de alcance

No implementes una suscripción, no despliegues recursos reales, no agregues
secretos, no diseñes el esquema de una base de datos y no construyas todavía
un pipeline CI/CD. Tampoco es necesario resolver mensajería, contenedores,
autenticación de usuarios finales u observabilidad avanzada; se tratarán en
capítulos posteriores.
