# Snippet Searcher — Arquitectura

## Resumen

Snippet Searcher es una plataforma web para guardar, compartir, validar, formatear, lintear, ejecutar y testear
snippets de código. Por ahora el único lenguaje es PrintScript, pero la arquitectura está pensada para sumar
otros lenguajes sin cambiar los servicios existentes.

La plataforma se compone de una UI web, un proxy (nginx), tres servicios, dos bases de datos, un almacenamiento
de archivos y un broker de mensajes. La autenticación se delega en Auth0.

| Documento | Contenido |
|---|---|
| [user-stories.md](user-stories.md) | Historias de usuario con criterios de aceptación |
| [components/components.puml](components/components.puml) | Diagrama de componentes |
| [sequence/](sequence/) | Diagramas de secuencia de los workflows principales |
| [data-model.puml](data-model.puml) | Modelo de datos |
| [contracts.md](contracts.md) | Endpoints HTTP y eventos de cada servicio |

## Componentes

| Componente | Responsabilidad |
|---|---|
| **UI (Web)** | Interfaz provista por la cátedra; se configura para usar nuestros servicios. Hace el login con Auth0 y envía el JWT en cada request. |
| **nginx** | Único punto de entrada. Sirve la UI y reenvía `/api` a Snippet Service. |
| **Snippet Service** | Servicio principal y único expuesto. Valida el JWT; administra snippets, tests y reglas por usuario; guarda el contenido en Blob Storage; coordina a los demás servicios y publica los trabajos asíncronos. |
| **Permission Service** | Guarda quién tiene acceso a cada snippet (`OWNER` / `READ`) y busca usuarios en Auth0 para compartir. |
| **PrintScript Service** | Sin estado. Expone el parser, linter, formatter e intérprete de la librería PrintScript, tanto por REST como consumiendo pedidos de Redis. |
| **Snippet DB** (PostgreSQL) | Metadata de snippets, violaciones de lint, tests y sus resultados, reglas por usuario y versión del parser registrada. |
| **Permission DB** (PostgreSQL) | Permisos por snippet y usuario. |
| **Azure Blob Storage** | Contenido de los snippets: versión original y formateada. |
| **Redis Streams** | Cola de trabajos asíncronos (lint, formato, tests) y de sus resultados. |
| **Auth0** | Login de usuarios (OAuth) y Management API para buscar usuarios. |

## Comunicación

| Interacción | Tipo | Motivo |
|---|---|---|
| UI → nginx → Snippet Service | REST síncrono | El usuario espera la respuesta. |
| Snippet → Permission | REST síncrono | Cada operación necesita saber el rol antes de seguir. |
| Snippet → PrintScript (`/validate`, `/test`, `/format` al descargar) | REST síncrono | El resultado se muestra en el momento: un snippet inválido no se guarda. |
| Snippet → PrintScript (lint, formato, tests automáticos) | Redis Streams, asíncrono | Pueden ser muchos snippets; el usuario no debe esperar (#12, #15, #16). |
| PrintScript → Snippet (resultados, versión del parser) | Redis Streams, asíncrono | Desacopla a los workers de Snippet Service. |

## Decisiones de diseño

### Servicios separados por responsabilidad
Los permisos están en su propio servicio y base porque son un concepto transversal: hoy aplican a snippets y
tests, y es lo que se va a extender si en el futuro se habilitan funcionalidades por rol. La ejecución del
lenguaje está aislada en PrintScript Service para que un snippet que consume mucha CPU o falla no afecte al
servicio que atiende al usuario, y para poder escalar los workers por separado.

### PrintScript Service sin estado y uno por lenguaje
Toda la información necesaria (contenido, versión, reglas, inputs) viaja en el request o en el mensaje, y el
servicio no guarda nada. Así se pueden levantar varias réplicas que compiten por los mismos mensajes. Para sumar
un lenguaje se agrega otro servicio con la misma interfaz; Snippet Service elige a cuál llamar según el
lenguaje del snippet.

### Procesamiento asíncrono con Redis Streams
Cambiar una regla puede implicar reprocesar todos los snippets de un usuario. Por eso `PUT /rules/*` responde
`202 Accepted` enseguida y el trabajo se encola. Se eligió Redis Streams porque ofrece *consumer groups*
(varios workers se reparten los mensajes) y una lista de mensajes pendientes sin `ACK`, que es la base de la
tolerancia a fallos.

### Tolerancia a fallos (#12, #15)
- El `ACK` se hace recién después de procesar: si un worker se cae, el mensaje no se pierde y otro lo retoma
  (`XAUTOCLAIM`). Así el proceso continúa desde donde falló.
- Si un mismo mensaje se entregó más de 3 veces, se asume que el snippet hace fallar al worker: se marca
  `FAILED` y se saltea, sin frenar al resto.
- Cada pedido lleva una `revision` del snippet. Si llega un resultado viejo (por ejemplo, el usuario cambió las
  reglas dos veces seguidas), se descarta y no pisa al más nuevo.

### PostgreSQL para metadata, Blob Storage para contenido
Listar snippets requiere filtrar, ordenar y paginar por nombre, lenguaje y validez, y cruzar con permisos: eso
lo resuelve una base relacional. El contenido, en cambio, puede ser grande, se lee entero y nunca se consulta
por partes, así que se guarda en Blob Storage y la base solo guarda su metadata. Se usa Azure Blob Storage
porque la infraestructura de la materia corre en Azure.

Para crear un snippet primero se sube el contenido y después se inserta la fila: si algo falla en el medio,
queda a lo sumo un archivo huérfano, nunca una fila sin contenido. Si falla la creación del permiso de owner,
se borran la fila y el archivo.

### Original y formateado
Se guardan las dos versiones (`snippets/{id}/original` y `snippets/{id}/formatted`). El formateo automático
(#12) solo escribe la formateada, así el usuario no pierde lo que subió y puede descargar cualquiera de las dos
(#13).

### Autenticación y autorización
Auth0 maneja el login. Solo Snippet Service valida el JWT, porque es el único servicio expuesto por nginx;
Permission y PrintScript solo son accesibles desde la red interna y reciben el `userId` desde Snippet Service.
Esto evita propagar el token a los mensajes de Redis.

La autorización es por snippet: `OWNER` puede editar, borrar, compartir y administrar tests; `READ` puede ver,
descargar y correr tests.

### Reglas por usuario
Las reglas de lint y formato se configuran por usuario y se aplican a los snippets de los que es owner. La
validez que ve un usuario con quien se compartió un snippet es la calculada con las reglas del owner.

### Nueva versión del parser (#17)
Al arrancar, PrintScript Service publica la versión del parser que usa. Snippet Service la compara con la
última registrada y, si cambió, encola el re-linteo de todos los snippets de ese lenguaje. PrintScript Service
sigue sin estado: solo informa su versión, y reiniciar o escalar el servicio no dispara reprocesamientos.

## Pendientes para próximas iteraciones
- **Ejecución en vivo (#9, #10):** hoy correr un test es una llamada REST que devuelve todos los outputs al
  final. Para verlos a medida que se evalúan y pedir inputs en el momento se usará un WebSocket
  (`/snippets/{id}/run`).
- **Deploy:** cada servicio y base de datos en su contenedor Docker, orquestados con Docker Compose en
  desarrollo y Docker Swarm en las máquinas virtuales.
