# Intent del proyecto

## Objetivo

Implementar desde cero el entorno base de desarrollo (`dev`) diseñado en el
capítulo 1 para una futura API de documentos en Azure. El resultado inmediato
es un laboratorio reproducible y de bajo costo con un grupo de recursos,
Storage Blob privado, red virtual, subnets, NSG, RBAC acotado, etiquetas y
control presupuestario. La API y sus servicios de aplicación se implementarán
en capítulos posteriores.

## Condiciones reales del ejercicio

El proyecto dispone únicamente de una cuenta personal de Azure y una sola
suscripción, aprovechando la capa gratuita o el crédito disponible. No existen
otras suscripciones, tenants, usuarios, grupos de Microsoft Entra ID, grupos
de recursos, identidades administradas ni recursos creados previamente. La
implementación debe comenzar verificando esa cuenta, tenant y suscripción, y
crear cada elemento en orden lógico, sin asumir infraestructura previa ni
crear identidades ficticias para simular una organización.

La cuenta personal será la identidad de arranque y puede conservar su rol
administrativo heredado, porque es la única cuenta y retirarlo podría impedir
la operación del laboratorio. Aun así, se aplican buenas prácticas: no se
crean nuevos `Owner`, el acceso de datos se limita al contenedor necesario, no
se usan claves ni secretos en código, y se evita abrir Storage a Internet.
Cuando aparezcan más colaboradores se usará un grupo de Entra; cuando exista
una API desplegada se creará su identidad administrada con permisos mínimos.

## Resultado esperado

La guía de implementación debe ser paso a paso y cubrir: comprobación de
acceso y costos; selección explícita de la única suscripción; creación del
grupo de recursos; Storage privado y firewall; VNet, subnets y NSG; asignación
RBAC; presupuesto; validación; actualización de IP; y eliminación segura.
Debe explicar errores solo después de que el recurso correspondiente pueda
existir y no debe crear recursos futuros fuera del alcance.
