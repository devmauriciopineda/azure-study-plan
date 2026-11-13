# Actividad del Capítulo 1. Diseño de un entorno base en Azure

## Propósito

Aplicar los fundamentos de organización, identidad, red y almacenamiento a un
entorno de desarrollo para una API backend en Python. La actividad no exige
crear recursos reales: su objetivo es producir un diseño revisable antes de
automatizar o desplegar.

## Caso de aplicación

Una aplicación backend expone una API REST para administrar documentos. En esta
etapa solo se necesita preparar el entorno `dev` con:

- una cuenta de almacenamiento para documentos y archivos de prueba;
- una red virtual para organizar la conectividad futura;
- acceso diferenciado para una persona desarrolladora y una identidad de
  aplicación;
- recursos agrupados de forma que el entorno pueda eliminarse sin afectar
  producción.

No se proporciona una suscripción real, una región obligatoria ni datos de
clientes. Debes declarar supuestos y elegir valores razonables.

## Entregable

Crea un documento Markdown en `course/` llamado
`capitulo-1-entrega.md`. Incluye las siguientes secciones:

1. **Resumen de decisiones:** tabla con entorno, suscripción, región, grupo de
   recursos y justificación.
2. **Inventario de recursos:** tabla con nombre lógico, tipo, grupo, ciclo de
   vida, etiquetas y propósito.
3. **Identidades y RBAC:** tabla con principal, rol, alcance, operaciones
   permitidas y razón por la que el rol es mínimo.
4. **Diseño de red:** diagrama sencillo en Mermaid o ASCII con VNet, subnets,
   recursos y flujos permitidos. Indica puertos y protocolos solo cuando sean
   necesarios.
5. **Almacenamiento:** tipo de datos, contenedor, nivel de acceso, redundancia,
   autorización y política de retención propuesta.
6. **Validación y riesgos:** lista de comprobaciones, supuestos, límites y
   riesgos pendientes.

## Pasos

1. Define un estándar de nombres legible para el entorno `dev` y al menos estas
   etiquetas: `environment`, `application`, `owner` y `costCenter`.
2. Selecciona una región y justifica la decisión considerando latencia,
   disponibilidad del servicio, residencia de datos y costo. Si no puedes
   verificar un dato, decláralo como supuesto.
3. Decide si la API y la cuenta de almacenamiento deben compartir grupo de
   recursos. Justifica la decisión con base en ciclo de vida, no solo en
   proximidad.
4. Asigna permisos a una persona desarrolladora y a una identidad de
   aplicación. Explica por qué cada asignación no tiene un alcance mayor del
   necesario y distingue administración del recurso de acceso a los datos.
5. Propón una VNet con subnets suficientes para el escenario actual. Describe
   los flujos requeridos y las reglas NSG que permitirías o bloquearías.
6. Diseña la Storage Account y un contenedor para documentos. Indica cómo
   impedirías el acceso público accidental y cómo tratarías las credenciales.
7. Revisa el diseño con el método de ocho pasos de la lectura y registra dos
   decisiones que necesitarían confirmación antes de producción.

## Criterios de aceptación

La entrega es aceptable cuando:

- distingue correctamente tenant, suscripción, grupo de recursos y región;
- incluye una justificación explícita para la región y el ciclo de vida del
  grupo de recursos;
- usa las cuatro etiquetas solicitadas y explica su finalidad;
- asigna roles RBAC con principal, alcance y razón de mínimo privilegio;
- diferencia autenticación, autorización, RBAC y acceso a datos;
- muestra VNet y subnets, y documenta tráfico permitido y bloqueado mediante
  NSG;
- describe Blob Storage dentro de una Storage Account sin tratarlo como una
  base de datos;
- identifica al menos un supuesto, un límite y un riesgo de seguridad o costo;
- no incluye secretos, claves reales ni datos personales;
- permite que otra persona revise las decisiones sin ejecutar comandos.

## Resultado esperado

Obtendrás un diseño pequeño pero completo del entorno `dev`, suficientemente
detallado para convertirse después en una implementación reproducible. El
resultado no busca acertar una única arquitectura: debe demostrar que las
decisiones están relacionadas con alcance, ciclo de vida, mínimo privilegio,
conectividad y necesidades de almacenamiento.

## Preguntas abiertas para la revisión

- ¿La región elegida cumple los requisitos de residencia y disponibilidad del
  servicio?
- ¿La identidad de aplicación necesita administrar la cuenta o solo acceder a
  blobs concretos?
- ¿Qué flujo de red se requerirá cuando la API se despliegue en el capítulo 2?
- ¿Qué política de retención y redundancia corresponde a documentos reales?
- ¿Qué controles adicionales serían necesarios antes de usar este diseño en
  producción?

## Fuera de alcance

No implementes recursos, no configures una suscripción, no uses credenciales
reales y no diseñes todavía el despliegue de la API, bases de datos,
mensajería, CI/CD o autenticación de usuarios finales. Esos temas se abordarán
en capítulos posteriores.
