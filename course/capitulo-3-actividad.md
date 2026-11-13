# Actividad del Capítulo 3. Diseño de persistencia para documentos

## Propósito

Seleccionar y justificar servicios de persistencia para la API de documentos
del capítulo anterior. La actividad exige relacionar datos, consultas,
garantías, conectividad, seguridad y ciclo de vida, sin crear recursos reales.

## Caso de aplicación

La API debe administrar documentos con estas necesidades:

- cada documento tiene propietario, nombre, tipo, tamaño, estado y fecha;
- el contenido puede ser grande y debe poder descargarse por separado;
- se necesita listar documentos de un propietario y consultar su estado;
- una operación debe evitar registrar dos veces el mismo documento;
- los archivos temporales deben eliminarse después de un periodo definido;
- el backend se ejecuta en `dev` y `test` con recursos separados;
- el equipo evalúa Azure SQL Database, Azure Database for PostgreSQL, Cosmos
  DB y Blob Storage.

No se proporciona volumen, región, requisitos legales ni credenciales reales.
Declara supuestos y señala qué decisiones requieren validación.

## Entregable

Crea un documento Markdown en `course/` llamado
`capitulo-3-entrega.md`. Incluye:

1. **Inventario de datos:** tabla con dato, tamaño aproximado, propietario,
   frecuencia de acceso, retención y sensibilidad.
2. **Decisión de servicios:** selección de base de datos y Blob Storage, con
   comparación de las alternativas descartadas.
3. **Modelo de persistencia:** esquema relacional o documentos, identificadores,
   relaciones, clave de partición e índices cuando correspondan.
4. **Flujos de operación:** carga, registro, consulta, descarga, actualización,
   eliminación y limpieza de archivos temporales.
5. **Conectividad Python:** biblioteca o SDK previsto, configuración, pool o
   cliente, timeouts y tratamiento de errores.
6. **Seguridad:** identidad, permisos mínimos, red, secretos, cifrado y
   separación entre `dev` y `test`.
7. **Consistencia y fallos:** análisis de qué ocurre si se guarda el blob pero
   falla el metadato, o si ocurre lo contrario.
8. **Ciclo de vida:** metadata, nivel de acceso, retención y política para
   mover o eliminar blobs.
9. **Supuestos y validaciones pendientes.**

## Pasos

1. Separa metadatos consultables del contenido binario y explica por qué no
   almacenarías automáticamente ambos en el mismo lugar.
2. Compara SQL Database, PostgreSQL y Cosmos DB con al menos cinco criterios:
   relaciones, transacciones, consultas, crecimiento, compatibilidad Python y
   operación.
3. Elige una base de datos y define el modelo mínimo para documento,
   propietario, estado y referencia al blob. Si eliges Cosmos DB, especifica
   la clave de partición y los patrones de consulta que la justifican.
4. Define el contenedor de Blob Storage, convención de nombres, metadata,
   nivel de acceso y reglas de retención. Indica cómo impedirías acceso público
   accidental.
5. Describe la operación de carga de un documento y el orden de pasos. Explica
   cómo detectarías o compensarías un blob huérfano o un registro sin blob.
6. Diseña la configuración de conexión para `dev` y `test`. Incluye endpoint,
   nombre lógico del recurso y valores de timeout, pero nunca secretos reales.
7. Asigna permisos a la identidad del backend y distingue administrar recursos
   de leer o escribir datos. Indica qué controles de red deben estar activos.
8. Enumera pruebas de aceptación: consulta normal, duplicado, archivo grande,
   error de autorización, timeout, configuración ausente, fallo parcial y
   eliminación por ciclo de vida.

## Criterios de aceptación

La entrega es aceptable cuando:

- distingue correctamente base relacional, base NoSQL y almacenamiento de
  objetos;
- justifica la selección entre SQL Database, PostgreSQL y Cosmos DB con
  requisitos del caso;
- separa metadatos de documentos y contenido binario;
- define relaciones o documentos, identificadores y partición o índices cuando
  correspondan;
- describe conexión desde Python con configuración por ambiente, timeout y
  manejo de errores;
- establece permisos mínimos y distingue plano de administración de acceso a
  datos;
- contempla cifrado, secretos, red y separación de `dev` y `test`;
- analiza consistencia y recuperación ante fallos parciales;
- incluye operaciones Blob Storage, metadata, nivel de acceso y ciclo de vida;
- contiene supuestos, límites y criterios de validación observables;
- no incluye credenciales, datos personales ni despliegues reales.

## Resultado esperado

Obtendrás un diseño de persistencia que puede ser implementado por el backend
en una etapa posterior. La propuesta debe dejar claro qué servicio responde a
cada necesidad y qué riesgos aparecen al combinar base de datos y Blob Storage.

## Preguntas abiertas para la revisión

- ¿El volumen y patrón de consultas justifican una base relacional o una clave
  de partición NoSQL?
- ¿Qué operación garantiza que un documento no se registre dos veces?
- ¿Qué pasa con un archivo si la transacción de metadatos falla?
- ¿Qué permisos exactos necesita la aplicación para cargar, leer y eliminar?
- ¿Cuándo un blob debe pasar a Cool o Archive, y qué latencia de recuperación
  es aceptable?
- ¿Qué requisitos de residencia, retención o auditoría aún no se conocen?

## Fuera de alcance

No implementes tablas, contenedores, migraciones ni recursos reales. No
construyas colas, procesamiento asíncrono, CI/CD, contenedores ni autenticación
de usuarios finales. Esos temas se abordarán en capítulos posteriores.
