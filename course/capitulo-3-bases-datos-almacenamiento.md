# Capítulo 3. Bases de datos y almacenamiento de aplicaciones

## Propósito de lectura

Al finalizar esta lectura podrás seleccionar una alternativa de persistencia
relacional, NoSQL o de objetos para una aplicación Python, explicar las
operaciones básicas de Azure SQL Database, Azure Database for PostgreSQL, Azure
Cosmos DB y Blob Storage, y diseñar una conexión que mantenga configuración,
seguridad y datos separados por ambiente.

El capítulo parte del backend desplegado en el capítulo anterior. La pregunta
central no es “¿qué servicio es más moderno?”, sino “¿qué garantías y patrón de
acceso necesita cada dato?”. Una aplicación puede usar más de un servicio si
cada uno tiene una responsabilidad clara.

## 1. Persistencia y elección del modelo

La **persistencia** es la capacidad de conservar datos más allá de la ejecución
actual de una aplicación. Una instancia de App Service o Functions puede
reiniciarse o escalar, por lo que su memoria y su sistema de archivos local no
deben usarse como almacenamiento permanente.

Una **base de datos relacional** organiza información en tablas relacionadas
mediante un esquema definido. Ofrece consultas estructuradas, restricciones,
relaciones y transacciones. Una **base de datos NoSQL** usa modelos distintos
al relacional tradicional, como documentos, clave-valor, columnas o grafos;
normalmente prioriza una forma de consulta y escalamiento concreta. El
**almacenamiento de objetos** conserva archivos o blobs completos y sus
metadatos, no un conjunto de filas relacionado para consultas transaccionales.

Antes de elegir, describe:

- qué entidad se almacena y cuál es su tamaño;
- cómo se escribe, consulta, actualiza y elimina;
- si se requieren transacciones entre varios datos;
- qué consistencia, latencia y disponibilidad se esperan;
- cómo crecerán volumen, particiones y consultas;
- qué controles de acceso, retención y residencia aplican.

No es necesario forzar todos los datos a un solo servicio. Por ejemplo, una
aplicación de documentos puede guardar metadatos consultables en una base de
datos y el contenido binario en Blob Storage, vinculándolos mediante un
identificador. La separación debe documentar cómo se mantiene la consistencia
entre ambas partes.

## 2. Azure SQL Database

**Azure SQL Database** es un servicio administrado de base de datos relacional
basado en el motor SQL de Microsoft. Azure administra tareas de plataforma
como aprovisionamiento, parches y opciones de disponibilidad, mientras el
equipo sigue siendo responsable del esquema, consultas, índices, permisos,
capacidad y comportamiento de la aplicación.

Una base relacional contiene **tablas**, formadas por filas y columnas. Una
**clave primaria** identifica una fila; una **clave foránea** representa una
relación con otra tabla. Una **restricción** impide estados inválidos, como
duplicados o valores nulos donde no corresponden. El **esquema** describe esta
estructura y sus reglas.

Una **transacción** agrupa operaciones para que cumplan propiedades ACID:

- **Atomicidad:** todas las operaciones se confirman o ninguna se confirma.
- **Consistencia:** las restricciones válidas se mantienen antes y después.
- **Aislamiento:** las operaciones concurrentes no observan estados
  intermedios de forma indebida.
- **Durabilidad:** una operación confirmada sobrevive a la finalización de la
  transacción según las garantías del servicio.

La operación básica comprende crear tablas y restricciones, abrir una
conexión, ejecutar consultas parametrizadas, confirmar cambios y cerrar la
conexión. Las consultas deben parametrizar valores recibidos por usuarios; no
se deben concatenar cadenas para formar SQL, porque eso puede introducir
inyección SQL. Los índices deben responder a consultas reales: demasiados
índices encarecen escrituras y mantenimiento.

Azure SQL Database es apropiado cuando existen relaciones, integridad
referencial, consultas flexibles y operaciones transaccionales. No debe
elegirse solo porque la aplicación use Python: la decisión depende del modelo
de datos y las garantías requeridas.

## 3. Azure Database for PostgreSQL

**Azure Database for PostgreSQL** es un servicio administrado que ofrece el
motor PostgreSQL para aplicaciones que necesitan un sistema relacional
compatible con su ecosistema y extensiones. Al igual que en Azure SQL
Database, el equipo diseña tablas, relaciones, consultas, índices, permisos y
capacidad lógica de la base de datos.

PostgreSQL comparte fundamentos relacionales: esquemas, tablas, claves,
restricciones, transacciones y consultas SQL. Sus dialecto, tipos, extensiones,
drivers y herramientas no son idénticos a los de SQL Server. La compatibilidad
de una aplicación debe verificarse con el controlador Python y las bibliotecas
que realmente se usarán.

La conexión desde Python necesita un endpoint, puerto, base de datos,
identidad o credencial y configuración de seguridad. Un **pool de conexiones**
mantiene un conjunto limitado de conexiones reutilizables para evitar abrir una
conexión nueva por cada solicitud. El tamaño del pool debe considerar
concurrencia de la aplicación y límites de la base de datos; un pool ilimitado
puede agotar conexiones.

Azure SQL Database y PostgreSQL pueden resolver problemas similares, pero no
son sustitutos transparentes. La elección puede depender de:

- compatibilidad con el código y bibliotecas existentes;
- dialecto SQL y extensiones requeridas;
- experiencia operativa del equipo;
- herramientas, migraciones y soporte disponibles;
- necesidades de rendimiento, costo y escalamiento.

En ambos casos se debe evitar colocar la cadena de conexión directamente en el
código o en el repositorio. La aplicación debe validar conectividad sin
exponer credenciales en mensajes de error.

## 4. Azure Cosmos DB

**Azure Cosmos DB** es un servicio de base de datos distribuida orientado a
modelos NoSQL. En el modelo de documentos, los datos se representan como
documentos JSON dentro de contenedores. El servicio ofrece capacidades de
distribución, escalamiento y consistencia configurables, pero el diseño de
partición y consultas determina gran parte del comportamiento real.

Una **base de datos** agrupa contenedores. Un **contenedor** almacena elementos,
define una clave de partición y concentra configuración de capacidad e
indexación. Un **elemento** es una unidad de datos, normalmente un documento
JSON. La **clave de partición** distribuye elementos entre particiones lógicas;
también debe aparecer en consultas frecuentes para evitar consultar muchas
particiones.

Una **partición lógica** es el conjunto de elementos con el mismo valor de
clave de partición. Una clave bien elegida distribuye carga y permite consultas
eficientes. Una clave mal elegida puede concentrar tráfico, limitar el
crecimiento o forzar consultas amplias. No se debe elegir una clave solo por
ser el campo más visible: hay que analizar cardinalidad, patrón de consulta,
crecimiento y distribución.

Los **índices** son estructuras que aceleran consultas sobre campos. Cosmos DB
puede indexar campos automáticamente según la configuración, pero indexar todo
no garantiza consultas baratas ni elimina la necesidad de diseñar particiones.
Una consulta que cruza muchas particiones puede consumir más recursos que una
consulta dirigida.

Cosmos DB permite elegir niveles de consistencia, desde modelos más fuertes
hasta modelos eventualmente consistentes con menor coordinación. **Consistencia
eventual** significa que réplicas pueden observar valores distintos durante un
intervalo antes de converger. La opción debe relacionarse con el caso de uso:
un catálogo puede tolerar una lectura ligeramente atrasada; una operación que
requiere una decisión inmediata puede necesitar garantías más fuertes.

Cosmos DB es útil para documentos con estructura flexible, alta distribución o
patrones de consulta conocidos. Flexibilidad de esquema no significa ausencia
de diseño: conviene definir convenciones de documentos, identificadores,
partición, índices y evolución de versiones.

## 5. Comparar relacional y NoSQL

Una base relacional suele ser preferible cuando el dominio tiene relaciones
importantes, integridad referencial, transacciones entre entidades y consultas
que deben evolucionar con flexibilidad. Cosmos DB puede ser preferible cuando
el acceso se centra en documentos, la distribución global o el escalamiento
horizontal, y el modelo puede diseñarse alrededor de una clave de partición.

La comparación debe considerar el costo de mover complejidad:

| Pregunta | Relacional | NoSQL documental |
|---|---|---|
| Forma de datos | Tablas y relaciones | Documentos y agregados |
| Consistencia | Transacciones y restricciones | Nivel configurable y alcance definido |
| Consultas | Relacionales y flexibles | Diseñadas alrededor de particiones |
| Evolución | Migraciones de esquema | Versiones y validación de documentos |
| Escalamiento | Capacidad y diseño de índices | Partición, capacidad y distribución |

No existe una frontera absoluta. Una solución puede usar PostgreSQL para
entidades transaccionales, Cosmos DB para documentos de acceso distribuido y
Blob Storage para archivos. Cada elección agrega conexiones, permisos,
operación y posibles inconsistencias; por eso debe justificarse por un
requisito, no por moda.

## 6. Conectividad desde aplicaciones Python

La **conectividad** comprende resolución de endpoint, ruta de red, TLS,
autenticación, autorización, protocolo y administración de conexiones. Que un
servicio esté en Azure no implica que cualquier aplicación pueda conectarse a
él.

Un cliente Python de base de datos o del SDK necesita normalmente:

1. identificar el servicio y su endpoint;
2. seleccionar una biblioteca compatible;
3. obtener una credencial o usar una identidad admitida;
4. crear el cliente o pool con límites explícitos;
5. ejecutar operaciones parametrizadas;
6. manejar errores transitorios y permanentes;
7. liberar conexiones y recursos de forma segura.

La aplicación de App Service o Functions debe recibir endpoints y nombres de
recursos mediante configuración por ambiente. Las credenciales deben provenir
de identidad administrada o de un almacén seguro cuando sea posible. Las
reglas de red, firewalls, private endpoints y permisos de datos deben
coordinarse: un permiso correcto no soluciona una ruta bloqueada, y una ruta
abierta no autoriza una consulta.

Los errores de conexión deben distinguir, al menos, configuración ausente,
fallo de red, autenticación rechazada, autorización insuficiente y timeout.
Reintentar indiscriminadamente una operación de escritura puede duplicar datos;
los reintentos deben considerar idempotencia y límites, temas que se
profundizarán en capítulos posteriores.

## 7. Persistencia, configuración y seguridad

Una aplicación debe separar código, configuración y datos. El código define
operaciones; la configuración identifica el recurso y el ambiente; el servicio
de persistencia conserva el dato. La cadena de conexión, token o clave no debe
versionarse ni aparecer en logs.

La seguridad de acceso tiene varias capas:

- identidad de la aplicación o usuario;
- autorización del servicio y del dato;
- cifrado en tránsito y en reposo;
- restricciones de red;
- protección y rotación de secretos;
- auditoría y revisión de permisos.

En desarrollo local, puede usarse una credencial de desarrollador con permisos
acotados. En Azure, una identidad administrada evita distribuir secretos a la
aplicación. En ambos casos, el código debe pedir solo las operaciones
necesarias, por ejemplo leer y escribir blobs de un contenedor, no administrar
toda la cuenta.

La persistencia también exige decidir qué sucede cuando una operación parcial
falla. Si se guarda un blob y después falla el registro de metadatos, o al
contrario, pueden quedar datos huérfanos. El diseño debe documentar el orden de
operaciones, la posibilidad de compensación y la forma de detectar y limpiar
inconsistencias.

## 8. Blob Storage: operaciones, metadata y ciclo de vida

**Blob Storage** es almacenamiento de objetos para contenido como documentos,
imágenes, exportaciones y archivos temporales. Una cuenta contiene
contenedores; cada contenedor contiene blobs. La aplicación normalmente guarda
en la base de datos el identificador, propietario, tipo, tamaño, estado y
referencia del blob, no el archivo completo.

Operaciones comunes incluyen:

- cargar un blob;
- descargarlo o leerlo por partes;
- listar blobs de un contenedor;
- copiarlo o eliminarlo;
- consultar y modificar sus propiedades;
- establecer metadata y etiquetas cuando el caso lo requiere.

La **metadata** son pares clave-valor definidos por la aplicación para describir
un blob. Las **propiedades** son atributos administrados por el servicio, como
tipo de contenido, tamaño o fecha de modificación. No se deben confundir:
metadata no reemplaza el registro transaccional de una base de datos y puede
requerir una operación separada para mantenerse consistente.

Los **niveles de acceso** clasifican blobs según frecuencia de lectura, como
Hot, Cool y Archive, con compromisos distintos de costo de almacenamiento,
operación y latencia de recuperación. Archive está pensado para retención
prolongada y acceso poco frecuente, no para respuestas interactivas
inmediatas.

Una **política de ciclo de vida** automatiza acciones basadas en edad o
condiciones, como mover blobs a otro nivel o eliminarlos. Debe considerar
retención legal, recuperación, datos huérfanos, prefijos y excepciones. Una
regla de eliminación demasiado amplia puede borrar información necesaria.

## 9. Método de selección e integración

Para elegir persistencia para una funcionalidad, usa esta secuencia:

1. **Clasifica el dato:** fila relacional, documento consultable, archivo u
   objeto temporal.
2. **Describe operaciones:** lecturas, escrituras, actualizaciones, borrados,
   volumen, concurrencia y consultas.
3. **Define garantías:** relaciones, transacciones, consistencia, durabilidad,
   disponibilidad y retención.
4. **Evalúa crecimiento:** tamaño, distribución, partición, índices y costo.
5. **Selecciona servicio:** SQL Database, PostgreSQL, Cosmos DB o Blob Storage;
   combina servicios solo si el límite entre responsabilidades es claro.
6. **Diseña conexión:** biblioteca Python, endpoint, pool o cliente, timeout e
   identificación de errores.
7. **Protege acceso:** identidad, permisos mínimos, red, cifrado y secretos.
8. **Prueba recuperación:** operación normal, fallo parcial, reintento,
   configuración incorrecta y limpieza de datos huérfanos.

## Cierre

Azure SQL Database y Azure Database for PostgreSQL ofrecen persistencia
relacional; Cosmos DB ofrece un modelo NoSQL distribuido; Blob Storage conserva
objetos y archivos. La selección correcta depende de consultas, garantías,
crecimiento, conectividad y seguridad. En el siguiente capítulo estas
capacidades se integrarán con colas, topics y eventos para desacoplar el
procesamiento de la aplicación.
