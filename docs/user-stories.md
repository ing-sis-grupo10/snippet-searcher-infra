# User Stories

Basadas en el documento de casos de uso de la cátedra (#1–#16), más la #17, que surge de la consigna
("con cada versión nueva del parser se deben re-evaluar y validar todos los snippets").

Los roles (desarrollador, administrador, investigador) describen puntos de vista, no permisos distintos.
Todos los usuarios están autenticados con Auth0. Por snippet hay dos permisos: **OWNER** y **READ**.

> Las historias #9, #10, #12 y #15 se implementan más adelante en la cursada, pero se incluyen en el diseño.

| # | Historia | Diagrama de secuencia |
|---|---|---|
| 1, 3 | Crear snippet (archivo / editor) | [01-create-snippet](sequence/01-create-snippet.puml) |
| 2, 4, 16 | Actualizar snippet y correr sus tests | [02-update-snippet](sequence/02-update-snippet.puml) |
| 14, 15 (11, 12) | Cambiar reglas y reprocesar en segundo plano | [03-change-lint-rules](sequence/03-change-lint-rules.puml) |
| 7 | Compartir snippet | [04-share-snippet](sequence/04-share-snippet.puml) |
| 5 | Listar snippets | [05-list-snippets](sequence/05-list-snippets.puml) |
| 9 | Correr un test | [06-run-test](sequence/06-run-test.puml) |
| 17 | Re-validar por nueva versión del parser | [07-new-parser-version](sequence/07-new-parser-version.puml) |

---

## Gestión de snippets

### #1 — Crear snippet mediante archivo
**Como** desarrollador **quiero** crear un snippet subiendo un archivo **para** tenerlo disponible para verlo, operarlo o modificarlo.

**Criterios de aceptación**
- Se indican nombre, descripción, lenguaje y versión del lenguaje.
- Si el snippet no es válido según el parser, no se guarda y se informa la regla incumplida, la línea y la columna.
- Quien lo crea queda como OWNER.
- Al crearse se encolan su linteo y su formateo; mientras tanto el estado es `PENDING`.

### #2 — Actualizar snippet mediante archivo
**Como** desarrollador **quiero** actualizar un snippet subiendo un archivo **para** replicar los cambios que hice localmente.

**Criterios de aceptación**
- Solo el OWNER puede actualizar.
- Se puede actualizar el contenido y todos los datos (nombre, descripción, lenguaje, versión).
- El nuevo contenido debe ser válido; si no, se informa regla, línea y columna y no se guarda.
- Al actualizar se re-lintea y re-formatea el snippet y se corren sus tests (#16).

### #3 — Crear snippet en el editor
**Como** desarrollador **quiero** crear un snippet en un editor embebido **para** no tener que escribirlo primero en mi IDE.

**Criterios de aceptación:** los mismos que #1. Solo cambia el origen del contenido (el editor en vez del archivo).

### #4 — Actualizar snippet en el editor
**Como** desarrollador **quiero** editar un snippet en el editor embebido **para** hacer cambios rápidos.

**Criterios de aceptación:** los mismos que #2.

### #5 — Listar snippets
**Como** administrador de snippets **quiero** ver todos los snippets a los que tengo acceso **para** tener una visión general.

**Criterios de aceptación**
- Se puede filtrar por relación: propios, compartidos conmigo o ambos.
- Cada snippet muestra si es válido según las reglas de linteo que le aplican (las de su owner).
- Se puede filtrar y ordenar por nombre, lenguaje y validez.
- La lista está paginada.

### #6 — Ver un snippet
**Como** administrador de snippets **quiero** ver un snippet en particular **para** ver su contenido y las reglas de linteo que no cumple.

**Criterios de aceptación**
- Se ven nombre, descripción, lenguaje, versión, contenido y owner.
- Se ven las violaciones de linteo con regla, línea y columna.
- Se ven los tests del snippet y se pueden ejecutar (#9).
- Requiere permiso OWNER o READ.

### #7 — Compartir snippet
**Como** owner **quiero** compartir mis snippets con otros usuarios **para** que puedan verlos.

**Criterios de aceptación**
- Solo el OWNER puede compartir.
- El usuario se elige de un listado que se filtra mientras se escribe (búsqueda en Auth0).
- El usuario con el que se comparte obtiene permiso READ sobre el snippet y sus tests.

### #13 — Descargar snippet
**Como** investigador **quiero** descargar un snippet **para** no tener que copiar y pegar.

**Criterios de aceptación**
- Se puede descargar la versión original o la formateada.
- Si la versión formateada todavía no está lista, se formatea en el momento.
- Requiere permiso OWNER o READ.

---

## Tests

### #8 — Crear tests
**Como** desarrollador **quiero** crear tests para mis snippets **para** verificar que funcionan y documentar su comportamiento.

**Criterios de aceptación**
- Un test tiene nombre, una lista ordenada de inputs y una lista ordenada de outputs esperados.
- Los inputs se consumen en el orden en que el snippet los pide; los outputs se comparan en el orden de los `println`.
- Se pueden guardar tests que no pasan.
- Solo el OWNER puede crear, editar o borrar tests.

### #9 — Correr un test
**Como** investigador **quiero** correr un test de un snippet al que tengo acceso **para** ver su ejecución.

**Criterios de aceptación**
- Los tests se ven al visualizar el snippet.
- Se corre un test a la vez.
- El output se muestra a medida que se evalúa.
- Se informa si pasó o no.

### #16 — Testing automático
**Como** desarrollador **quiero** que al modificar un snippet se corran todos sus tests **para** saber qué tests pasan y cuáles fallan.

**Criterios de aceptación**
- Los tests se corren de forma asíncrona después del update.
- El resultado de los tests no afecta al update: el snippet se guarda igual.
- Se guarda el resultado del último run de cada test.

---

## Ejecución

### #10 — Ejecución interactiva
**Como** investigador **quiero** ejecutar un snippet dando los inputs a medida que se piden **para** probar cómo corre.

**Criterios de aceptación**
- Los inputs se piden en el momento en que el snippet los necesita.
- Los outputs se muestran a medida que se imprimen.
- Hay una ejecución a la vez.

---

## Reglas de formateo y linteo

### #11 — Configurar reglas de formateo
**Como** administrador **quiero** habilitar y deshabilitar reglas de formateo **para** que mis snippets cumplan el estilo acordado.

**Criterios de aceptación**
- Por ahora solo se ofrecen las reglas predeterminadas de PrintScript.
- La configuración es por usuario y aplica a los snippets de los que es owner.

### #12 — Formateo automático
**Como** administrador **quiero** que al cambiar las reglas de formateo se formateen todos mis snippets **para** que siempre cumplan el estilo.

**Criterios de aceptación**
- El cambio responde enseguida; el formateo corre en segundo plano.
- El proceso es tolerante a fallos: si un worker se cae, otro retoma lo pendiente; un snippet que falla repetidamente se marca `FAILED` y se saltea.
- Se guarda la versión formateada sin perder la original.

### #14 — Configurar reglas de linteo
**Como** administrador **quiero** habilitar y deshabilitar reglas de linteo **para** que mis snippets cumplan las reglas acordadas.

**Criterios de aceptación**
- Por ahora solo se ofrecen las reglas predeterminadas de PrintScript.
- La configuración es por usuario y aplica a los snippets de los que es owner.

### #15 — Linteo automático
**Como** administrador **quiero** que al cambiar las reglas de linteo se re-evalúen todos mis snippets **para** ver cuáles cumplen y cuáles no.

**Criterios de aceptación**
- El cambio responde enseguida; el linteo corre en segundo plano.
- Mismo esquema de tolerancia a fallos que #12.
- Durante el proceso el estado del snippet es `PENDING`; al terminar es `VALID`, `INVALID` o `FAILED`.

---

## Plataforma

### #17 — Re-validación por nueva versión del parser
**Como** administrador de la plataforma **quiero** que al publicarse una versión nueva del parser se re-validen todos los snippets **para** que su estado refleje el comportamiento actual del lenguaje.

**Criterios de aceptación**
- Al arrancar, PrintScript Service informa la versión del parser que usa.
- Si es distinta de la última registrada, se encola el re-linteo de todos los snippets de ese lenguaje.
- Usa el mismo procesamiento asíncrono y tolerante a fallos que #15.
