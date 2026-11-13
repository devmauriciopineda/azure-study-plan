# Capítulo 2. Desarrollo de aplicaciones backend en Azure

## Propósito de lectura

Al finalizar esta lectura podrás identificar la estructura de una aplicación
en Azure App Service, desplegar conceptualmente una aplicación Python o una API
REST, explicar el modelo de ejecución de Azure Functions y elegir entre App
Service y Functions según la carga. También podrás organizar configuración,
variables de entorno y archivos por ambiente, y reconocer el papel del Azure
SDK para Python.

El capítulo parte de los fundamentos de organización, identidad, red y
almacenamiento del capítulo anterior. Su objetivo es comprender el ciclo de
ejecución y despliegue de una aplicación, no resolver todavía persistencia
avanzada, mensajería, contenedores u observabilidad.

## 1. Azure App Service

**Azure App Service** es un servicio administrado para hospedar aplicaciones
web, APIs REST y otros backends HTTP. Azure administra buena parte de la
infraestructura subyacente, mientras el equipo configura la aplicación, su
runtime, escala, red y variables de entorno.

Un **App Service plan** define la capacidad de cómputo, el sistema operativo, la
región y ciertas características de precio y escalamiento. Una **app** es la
unidad de aplicación que se ejecuta en ese plan. Varias aplicaciones pueden
compartir un plan y, por tanto, compartir capacidad y costos; esta decisión
requiere revisar aislamiento, consumo y riesgo de competencia por recursos.

La estructura básica es:

- **Plan:** capacidad, región, sistema operativo y nivel de servicio.
- **Aplicación web:** configuración y código de una aplicación concreta.
- **Deployment slot:** entorno de ejecución adicional asociado a una app, útil
  para validar o preparar cambios antes de intercambiarlos con producción
  cuando el nivel de servicio lo permite.
- **Application settings:** configuración expuesta a la aplicación como
  variables de entorno.
- **Logs y diagnósticos:** mecanismos para revisar arranque, errores y
  solicitudes, que deben configurarse de acuerdo con las necesidades del
  entorno.

App Service simplifica el alojamiento, pero no elimina decisiones de
arquitectura. Hay que elegir región, tamaño, reglas de red, identidad,
configuración, estrategia de despliegue y límites de consumo. Un servicio
administrado reduce trabajo operativo; no convierte una aplicación mal
configurada en una aplicación confiable.

## 2. Despliegue de aplicaciones Python y APIs REST

Una **aplicación Python** ejecutada en App Service necesita un punto de entrada,
dependencias declaradas y un comando de arranque compatible con el framework
utilizado. Una **API REST** es una interfaz HTTP que expone recursos u
operaciones mediante métodos y representaciones, normalmente JSON. Azure
proporciona el proceso de hospedaje, pero la aplicación sigue siendo responsable
de sus rutas, validación, códigos de estado y manejo de errores.

Un flujo de despliegue reproducible suele incluir:

1. preparar el código en un repositorio;
2. declarar dependencias en el formato usado por el proyecto;
3. definir el runtime y la versión de Python compatibles;
4. ejecutar pruebas y validaciones antes de publicar;
5. configurar el comando de inicio y variables del ambiente;
6. publicar el artefacto mediante Portal, CLI, un repositorio o un pipeline;
7. revisar logs, estado de salud y una operación HTTP representativa.

El **artefacto de despliegue** es el conjunto de archivos que se entrega al
servicio: código, dependencias, configuración no sensible y archivos
necesarios para iniciar. No debe contener secretos ni archivos temporales. El
comando de inicio debe ejecutar un servidor adecuado para producción y escuchar
en la configuración de puerto que proporcione el entorno, en vez de depender
de un servidor de desarrollo local.

En una API REST, el despliegue no termina cuando la aplicación aparece como
“running”. También se debe comprobar que:

- el proceso arranca sin intervención manual;
- las rutas responden con el contrato esperado;
- errores de entrada producen respuestas controladas;
- la configuración apunta al ambiente correcto;
- el acceso a dependencias externas usa identidad o secretos administrados;
- los logs permiten distinguir fallo de arranque, red y lógica de aplicación.

## 3. Azure Functions y su modelo de ejecución

**Azure Functions** es un servicio de ejecución orientado a funciones
individuales activadas por eventos. Una **function** es una unidad de código
que se ejecuta cuando ocurre un evento, como una solicitud HTTP o un horario.
El servicio administra gran parte del aprovisionamiento y puede ajustar la
capacidad según el plan y la carga.

Un **trigger** es la condición que inicia una función. El trigger define la
entrada principal y solo puede existir uno por función. Un **binding** es una
declaración de entrada o salida que conecta la función con otro servicio sin
obligar a escribir todo el código de conexión. Los bindings pueden simplificar
la integración, pero sus contratos, permisos, serialización y errores siguen
siendo responsabilidad del diseño.

La secuencia conceptual es:

1. llega un evento al trigger;
2. Functions crea o selecciona un contexto de ejecución;
3. la función recibe la entrada y su configuración;
4. procesa la operación;
5. produce una respuesta o escribe en un binding;
6. el runtime registra resultado, error y duración según la configuración.

Una **ejecución sin estado** no debe depender de variables de memoria de una
invocación anterior. Aunque una instancia pueda reutilizarse, ese
comportamiento no debe tratarse como almacenamiento persistente. El código
debe ser capaz de ejecutarse en otra instancia y tolerar reintentos cuando el
trigger o el servicio de integración los permitan.

Functions ofrece distintos planes con compromisos de escalamiento, arranque,
duración, red y costo. La elección del plan es parte del diseño y no una
propiedad abstracta de “serverless”. **Serverless** significa que el proveedor
gestiona servidores y capacidad operativa, no que la aplicación carezca de
recursos, límites, latencia o costo.

## 4. Triggers HTTP y temporizadores

Un **HTTP trigger** inicia una función cuando recibe una solicitud HTTP. Puede
servir para endpoints pequeños, webhooks, tareas administrativas o APIs con
operaciones acotadas. Debe definir método, ruta, autenticación, validación de
entrada, formato de respuesta y comportamiento ante errores.

Un **timer trigger** inicia una función conforme a una programación, como una
expresión de horario. Es útil para limpieza, sincronizaciones, generación de
resúmenes o tareas periódicas. El diseño debe considerar zona horaria,
duración, concurrencia y qué ocurre si una ejecución se retrasa o se repite.

HTTP y timer se diferencian por su iniciador, no necesariamente por la lógica
de negocio. Conviene extraer una operación común y mantener delgados los
adaptadores de entrada, siempre que eso no introduzca abstracciones
innecesarias. Una función activada por HTTP no debe asumir que el cliente
esperará indefinidamente; para trabajos largos puede ser necesario aceptar la
solicitud, registrar el trabajo y procesarlo de forma asíncrona, tema que se
desarrollará más adelante.

## 5. Azure SDK para Python

El **Azure SDK para Python** es el conjunto de bibliotecas que permite a un
programa Python interactuar con servicios de Azure mediante APIs tipadas o
abstracciones específicas del servicio. No es lo mismo que Azure CLI: la CLI
administra recursos desde comandos, mientras el SDK permite que el código de
la aplicación consuma servicios durante su ejecución.

Un cliente del SDK suele requerir:

- una biblioteca correspondiente al servicio;
- una credencial;
- el endpoint o identificador del recurso;
- una configuración de reintentos, timeout y transporte adecuada;
- manejo explícito de errores y cierre de recursos cuando aplique.

La autenticación debe preferir una identidad administrada u otro mecanismo
sin secretos embebidos cuando el entorno lo permita. En desarrollo local puede
usarse una credencial de desarrollador, pero el comportamiento y los permisos
del ambiente deben documentarse. El SDK no concede permisos por sí mismo:
la identidad usada por el cliente debe tener autorización suficiente en el
servicio.

Las bibliotecas del SDK pueden hacer más fácil llamar a un servicio, pero no
eliminan límites de cuota, latencia, errores transitorios ni validación de
datos. Un cliente bien diseñado centraliza configuración y aplica timeouts
razonables, sin ocultar errores con excepciones generales o reintentos
ilimitados.

## 6. Configuración y variables de entorno

La **configuración** reúne valores que cambian el comportamiento de una
aplicación sin modificar su código. Una **variable de entorno** es un valor
que el proceso recibe del entorno de ejecución. Puede contener valores no
secretos, como el nombre del ambiente, una URL o una bandera de funcionalidad;
también puede referenciar información sensible, aunque la variable en sí no
debe convertirse en un canal informal para compartir secretos.

Separar configuración y código permite promover el mismo artefacto entre
ambientes con valores distintos. La aplicación debe validar al arrancar los
valores obligatorios, producir un error claro si faltan y evitar mostrar
secretos en logs o mensajes de error.

Ejemplos de configuración apropiada:

- `APP_ENV=dev`;
- endpoint de un servicio;
- nivel de logging;
- límite funcional de una operación;
- nombre de un contenedor o recurso.

Las credenciales, claves y tokens requieren un mecanismo de secretos adecuado,
como una integración con un almacén seguro o identidad administrada. Guardar un
secreto en el repositorio, en una imagen o en un archivo de configuración
versionado expone el valor aunque luego se elimine del código.

## 7. Archivos y configuración por entorno

La **configuración específica por entorno** permite que `dev`, `test` y `prod`
usen recursos, niveles de logging, dominios y permisos distintos. Debe
mantenerse una forma común de configuración y variar únicamente los valores
necesarios. Un archivo local de ejemplo puede documentar nombres y tipos sin
contener valores reales.

Los **archivos de aplicación** son distintos de los archivos persistentes
subidos por usuarios. El paquete desplegado puede incluir plantillas o
recursos estáticos, pero el sistema de archivos local de una instancia no debe
tratarse automáticamente como almacenamiento permanente. En App Service y
Functions, reinicios, escalamiento o nuevos despliegues pueden cambiar la
instancia que atiende la solicitud.

Para archivos generados o subidos conviene usar un servicio de almacenamiento
apropiado y conservar en la aplicación solo referencias, metadatos o rutas
lógicas. Esta separación evita perder datos al reciclar la aplicación y
permite aplicar controles de acceso, retención y costo de forma independiente.

Una estrategia práctica de configuración es:

1. definir un esquema común y documentar campos obligatorios;
2. proporcionar valores seguros para desarrollo local;
3. inyectar valores del ambiente en App Service o Functions;
4. mantener secretos fuera del código y del artefacto;
5. validar al inicio y fallar explícitamente ante configuración inválida;
6. registrar qué versión de configuración se usó, sin registrar secretos.

## 8. Elegir entre App Service y Functions

La decisión no depende solo de preferencia tecnológica. **App Service** suele
ser adecuado cuando existe una aplicación web o API relativamente continua,
con rutas relacionadas, un proceso de larga duración controlado y necesidad de
configurar el hospedaje como una unidad. **Functions** suele ser adecuado
cuando el trabajo se expresa naturalmente como funciones activadas por eventos,
con ejecuciones independientes y escalamiento ligado a invocaciones.

| Criterio | App Service | Azure Functions |
|---|---|---|
| Forma de trabajo | Aplicación web o API completa | Funciones activadas por eventos |
| Entrada habitual | Solicitudes HTTP continuas | HTTP, temporizador y otros triggers |
| Organización | Un proceso y sus rutas | Unidades de ejecución separadas |
| Escalamiento | Capacidad del plan y reglas de escala | Plan y demanda de invocaciones |
| Control operativo | Mayor control sobre proceso y hospedaje | Mayor abstracción de infraestructura |
| Riesgo principal | Pagar capacidad o compartirla mal | Límites de ejecución, arranque y acoplamiento a eventos |

Una API HTTP puede ejecutarse en ambos servicios. La pregunta correcta es qué
modelo representa mejor el comportamiento, la duración de operaciones, la
necesidad de control del proceso, el patrón de escala, la integración con
eventos y el presupuesto. También se deben evaluar red, identidad, soporte del
runtime, despliegue, observabilidad y requisitos de disponibilidad.

No conviene usar Functions solo para evitar administrar un servidor si la
aplicación necesita un proceso continuo, operaciones largas o comportamiento
que no encaja en invocaciones aisladas. Tampoco conviene usar App Service para
cada tarea programada si una función simple expresa mejor el trabajo y sus
límites.

## 9. Método de aplicación

Para preparar el despliegue de un backend, usa esta secuencia:

1. **Describe la carga:** rutas, eventos, frecuencia, duración, concurrencia y
   dependencia de estado.
2. **Elige el modelo:** compara App Service y Functions con criterios
   explícitos, no por costumbre.
3. **Define el contrato de ejecución:** runtime, punto de entrada, comando de
   arranque, triggers y respuestas.
4. **Prepara el artefacto:** código, dependencias y archivos necesarios; excluye
   secretos y temporales.
5. **Separa configuración:** identifica valores por ambiente y valida los
   obligatorios.
6. **Diseña identidad:** elige credencial o identidad administrada y asigna
   solo permisos necesarios.
7. **Configura despliegue:** selecciona Portal, CLI o pipeline y define cómo
   verificar el resultado.
8. **Prueba operación realista:** revisa arranque, endpoint, logs, errores,
   límites y conexión con dependencias.

## Cierre

App Service y Azure Functions ofrecen modelos administrados distintos para
ejecutar backend Python y APIs REST. El primero organiza una aplicación web o
un proceso HTTP; el segundo organiza funciones iniciadas por triggers y
conectadas mediante bindings. En ambos casos, un despliegue correcto requiere
artefacto reproducible, configuración separada, identidad adecuada y
validación del comportamiento real. El siguiente capítulo aplicará estas
decisiones a bases de datos relacionales, NoSQL y almacenamiento de objetos.
