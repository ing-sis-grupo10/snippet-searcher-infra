# Contratos

Endpoints HTTP y eventos de cada servicio. Todos los endpoints requieren un JWT de Auth0
en el header `Authorization: Bearer <token>`; el `userId` se obtiene del token (`sub`), nunca del body.

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
  "id": "42",
  "name": "Hola mundo",
  "description": "Imprime un saludo",
  "language": "printscript",
  "version": "1.1",
  "owner": { "id": "auth0|abc", "name": "Martina" },
  "content": "println(\"Hola\");",
  "lintStatus": "INVALID",
  "violations": [
    { "rule": "identifier-case", "line": 1, "column": 5, "message": "Se esperaba camelCase" }
  ]
}
```

`lintStatus`: `PENDING` (en proceso), `VALID`, `INVALID`, `FAILED` (no se pudo procesar).

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

### Ejecución interactiva (más adelante)

| Protocolo | Ruta | Descripción | US |
|---|---|---|---|
| WebSocket | `/snippets/{id}/run` | El servidor envía outputs y pedidos de input; el cliente responde inputs | #10 |

---

## Permission Service (interno)

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/permissions` | Crear permiso `{ userId, snippetId, role: OWNER \| READ }` |
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
| `lint-requests` | Snippet | PrintScript | `{ snippetId, content, version, rules }` |
| `lint-results` | PrintScript | Snippet | `{ snippetId, valid, violations }` |
| `format-requests` | Snippet | PrintScript | `{ snippetId, content, version, rules }` |
| `format-results` | PrintScript | Snippet | `{ snippetId, content }` |

**Tolerancia a fallos (#12, #15):**
- Cada consumidor usa un *consumer group* y hace `ACK` recién después de procesar el mensaje.
- Si un worker se cae, los mensajes sin `ACK` quedan pendientes y otro worker los retoma.
- Después de 3 intentos fallidos, el snippet se marca `FAILED` y se saltea.

**Re-validación por nueva versión del parser (#18):** se publica un `lint-requests` por cada snippet.