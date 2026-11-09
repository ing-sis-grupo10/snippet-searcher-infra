# Contratos

Endpoints HTTP y eventos de cada servicio. Los endpoints públicos (Snippet Service) requieren un JWT de Auth0
en el header `Authorization: Bearer <token>`; el `userId` se obtiene del token (`sub`), nunca del body.
Permission Service y PrintScript Service no se exponen por nginx: solo se acceden desde la red interna,
y reciben el `userId` como parámetro desde Snippet Service.

## Formato de error común

Cuando un snippet es inválido, los endpoints responden `422 Unprocessable Entity`:

```json
{
  "errors": [
    { "rule": "missing-semicolon", "line": 3, "column": 14, "message": "Se esperaba ';'" }
  ]
}
```

---

## Snippet Service (público, vía nginx bajo `/api`)

### Snippets

| Método | Ruta | Descripción | US |
|---|---|---|---|
| POST | `/snippets` | Crear snippet (archivo o editor) | #1, #3 |
| PUT | `/snippets/{id}` | Actualizar contenido y/o datos (solo owner) | #2, #4 |
| GET | `/snippets` | Listar snippets accesibles con filtros y orden | #5 |
| GET | `/snippets/{id}` | Detalle del snippet | #6 |
| GET | `/snippets/{id}/download?formatted=true\|false` | Descargar original o formateado | #13 |
| DELETE | `/snippets/{id}` | Borrar snippet, sus tests, su contenido y sus permisos (solo owner) | — |
| POST | `/snippets/{id}/share` | Compartir con otro usuario (solo owner) | #7 |
| GET | `/users?search={texto}` | Buscar usuarios para compartir | #7 |

**POST `/snippets`**

```json
{
  "name": "Hola mundo",
  "description": "Imprime un saludo",
  "language": "printscript",
  "version": "1.1",
  "content": "println(\"Hola\");"
}
```

Respuesta `201 Created`: el snippet creado. Respuesta `422`: lista de errores (ver formato común).

**GET `/snippets`** — query params:

| Param | Valores | Default |
|---|---|---|
| `relation` | `owned`, `shared`, `all` | `all` |
| `name` | texto (búsqueda parcial) | — |
| `language` | `printscript` | — |
| `valid` | `true`, `false` | — |
| `sort` | `name`, `language`, `valid` (+ `,asc` / `,desc`) | `name,asc` |
| `page`, `size` | números | `0`, `20` |

**GET `/snippets/{id}`** — respuesta:

```json
{
  "id": "6f1c2a9e-3b7d-4c1e-9a52-0d8e7f4b1c23",
  "name": "Hola mundo",
  "description": "Imprime un saludo",
  "language": "printscript",
  "version": "1.1",
  "owner": { "id": "auth0|abc", "name": "Martina" },
  "content": "println(\"Hola\");",
  "lintStatus": "INVALID",
  "formatStatus": "DONE",
  "violations": [
    { "rule": "identifier-case", "line": 1, "column": 5, "message": "Se esperaba camelCase" }
  ]
}
```

`lintStatus`: `PENDING` (en proceso), `VALID`, `INVALID`, `FAILED` (no se pudo procesar).
La validez se calcula con las reglas de lint del **owner** del snippet.

`formatStatus`: `PENDING`, `DONE`, `FAILED`.

**GET `/snippets/{id}/download`** — devuelve el archivo como `text/plain` con
`Content-Disposition: attachment`. Con `formatted=true` devuelve la versión formateada; si todavía no está
lista (`formatStatus` distinto de `DONE`), Snippet Service la formatea en el momento con `POST /format`.

Contenido en Azure Blob Storage, contenedor `snippets`:

| Blob | Contenido |
|---|---|
| `snippets/{id}/original` | Lo que subió el usuario |
| `snippets/{id}/formatted` | Última versión formateada con las reglas del owner |

### Tests

| Método | Ruta | Descripción | US |
|---|---|---|---|
| GET | `/snippets/{id}/tests` | Listar tests del snippet | #6, #9 |
| POST | `/snippets/{id}/tests` | Crear test (solo owner) | #8 |
| PUT | `/snippets/{id}/tests/{testId}` | Editar test (solo owner) | #8 |
| DELETE | `/snippets/{id}/tests/{testId}` | Borrar test (solo owner) | #8 |
| POST | `/snippets/{id}/tests/{testId}/run` | Correr un test | #9 |

```json
{
  "name": "Saluda",
  "inputs": ["Juan"],
  "expectedOutputs": ["Hola Juan"]
}
```

Respuesta de `/run`: `{ "passed": false, "outputs": ["Hola"], "error": null }`

`GET /snippets/{id}/tests` incluye el resultado del último run de cada test (`lastStatus`: `NOT_RUN`,
`PENDING`, `PASSED`, `FAILED`, `ERROR`), que también se actualiza con los tests automáticos (#16).

### Reglas

| Método | Ruta | Descripción | US |
|---|---|---|---|
| GET | `/rules/lint` | Reglas de lint del usuario (catálogo + config) | #14 |
| PUT | `/rules/lint` | Cambiar reglas → dispara re-linteo async | #14, #15 |
| GET | `/rules/format` | Reglas de formato del usuario | #11 |
| PUT | `/rules/format` | Cambiar reglas → dispara re-formateo async | #11, #12 |

```json
[
  { "name": "identifier-case", "enabled": true, "value": "camelCase" },
  { "name": "println-expression-only", "enabled": false, "value": null }
]
```

`PUT` responde `202 Accepted`: los cambios se guardan y el reprocesamiento continúa en segundo plano.
Las reglas son por usuario y se aplican a los snippets de los que es owner.

### Ejecución interactiva (más adelante)

| Protocolo | Ruta | Descripción | US |
|---|---|---|---|
| WebSocket | `/snippets/{id}/run` | El servidor envía outputs y pedidos de input; el cliente responde inputs | #9, #10 |

nginx debe reenviar los headers `Upgrade` y `Connection` para esta ruta.

---

## Permission Service (interno)

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/permissions` | Crear permiso `{ userId, snippetId, role: OWNER \| READ }`; si ya existe, no hace nada |
| GET | `/permissions/check?userId=&snippetId=` | Rol del usuario sobre el snippet; `404` si no tiene |
| GET | `/permissions/snippets?userId=&relation=` | IDs de snippets accesibles |
| DELETE | `/permissions/snippets/{snippetId}` | Borrar permisos de un snippet |
| GET | `/users?search=` | Buscar usuarios en Auth0 (Management API) |

---

## PrintScript Service (interno, stateless)

Toda la información viaja en el request; el servicio no persiste nada.
Cada lenguaje futuro expone esta misma interfaz en su propio servicio.

| Método | Ruta | Request | Response |
|---|---|---|---|
| POST | `/validate` | `{ content, version }` | `200 { valid: true }` / `422 { errors }` |
| POST | `/lint` | `{ content, version, rules }` | `{ violations: [...] }` |
| POST | `/format` | `{ content, version, rules }` | `{ content: "..." }` |
| POST | `/execute` | `{ content, version, inputs }` | `{ outputs: [...], error }` |
| POST | `/test` | `{ content, version, inputs, expectedOutputs }` | `{ passed, outputs, error }` |
| GET | `/rules/lint` | — | Catálogo de reglas con defaults |
| GET | `/rules/format` | — | Catálogo de reglas con defaults |

---

## Eventos (Redis Streams)

| Stream | Publica | Consume | Payload |
|---|---|---|---|
| `lint-requests` | Snippet | PrintScript | `{ snippetId, revision, content, version, rules }` |
| `lint-results` | PrintScript | Snippet | `{ snippetId, revision, status: VALID \| INVALID \| FAILED, violations }` |
| `format-requests` | Snippet | PrintScript | `{ snippetId, revision, content, version, rules }` |
| `format-results` | PrintScript | Snippet | `{ snippetId, revision, status: DONE \| FAILED, content }` |
| `test-requests` | Snippet | PrintScript | `{ snippetId, testId, content, version, inputs, expectedOutputs }` |
| `test-results` | PrintScript | Snippet | `{ snippetId, testId, passed, outputs, error }` |
| `parser-versions` | PrintScript | Snippet | `{ language, version }` (al arrancar) |

**Tolerancia a fallos (#12, #15):**
- Cada consumidor usa un *consumer group* y hace `ACK` recién después de procesar el mensaje.
- Si un worker se cae, los mensajes sin `ACK` quedan pendientes; otro worker los retoma con `XAUTOCLAIM`
  cuando llevan más de N segundos inactivos.
- Si un mensaje ya se entregó más de 3 veces, el worker publica el resultado con `status: FAILED` y hace `ACK`:
  el snippet queda marcado y el proceso sigue con el resto.
- Cada pedido lleva la `revision` del snippet. Si llega un resultado con una revisión vieja (las reglas o el
  contenido cambiaron después), Snippet Service lo descarta.

**Tests automáticos (#16):** al actualizar un snippet se publica un `test-requests` por cada uno de sus tests.
El resultado no afecta al update.

**Re-validación por nueva versión del parser (#17):** al arrancar, PrintScript Service publica su versión en
`parser-versions`. Snippet Service la compara con la última registrada en `parser_version`; si cambió, publica
un `lint-requests` por cada snippet de ese lenguaje.
