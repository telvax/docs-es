> Traducido de [telvax/telvax@1c88355](https://github.com/telvax/telvax/commit/1c883558b98a024149553d2ba5265f075fc553be) (`docs/guide/tour.md`).

# Un recorrido por Telvax

Telvax es fácil de leer para quien conoce Python: los bloques se hacen con sangría, un bloque se abre
con `:`, y no hay llaves ni punto y coma. Tiene palabras propias (`fn`, `let` / `var`,
`type`, `x => x + 1`, `Int?`) y comprueba antes de que el programa se ejecute lo que Python descubre
a mitad de la ejecución: todo nombre tiene un tipo, escrito o inferido.

El código de esta guía está probado: cada bloque que no está marcado con `ignore` se comprueba, se formatea y,
si tiene `fn main`, se ejecuta. Ejecútalos desde la raíz del repositorio.

## Funciones, tipos e inferencia

```telvax
type Point:
    x: Float
    y: Float

fn norm(p: Point) -> Float:
    p.x * p.x + p.y * p.y

fn main() -> i32:
    let p = Point(x: 3.0, y: 4.0)
    var total = 0  # an untyped literal: its type comes from the uses
    for i in range(10):
        total += i
    print(f"norm = {norm(p)}, total = {total}")
    total
```

* Un `let` fuera de una función puede escribirse en mayúsculas (`MAX_TEXTS`), y sus métodos y campos funcionan como en cualquier nombre.
* `let x = 1` crea un nombre que no puede cambiar; `var x = 1` crea uno que sí puede (`x += 1`).
  `let x: Int = 1` escribe el tipo. La última expresión de una función es su valor, así que
  `return` solo hace falta para salir antes.
* Los tipos empiezan con mayúscula, como los tipos que escribes tú: `Int` (64 bits), `Float`
  (64 bits), `Bool`, `Text`, `List[T]`, `Map[K, V]`, `T?` (un valor o `None`), `Result[T, E]`,
  `Tensor[T, dims]` y `Table[Row]` (las columnas de una tabla también pueden ser `Date` y `Timestamp`). Cuando el tamaño importa: `i8`, `i16`, `i32`, `u8` a `u64`,
  `f32`, `f64`.
* Una función sin `->` devuelve el tipo de su última expresión: `fn double(x: Int): x * 2` devuelve
  un `Int`. Vale también para los métodos (`fn area(self): self.r * self.r`), y un método que implementa un
  trait devuelve lo que el trait declara para él. Escribe el `->` en `main`, en los tests, en los endpoints y en las funciones
  genéricas, y cuando la función tiene un error.
* Un nombre que empieza con `_` es privado de su archivo; todo lo demás puede importarse.
* Control de flujo: `if` / `elif` / `else`, `while`, `for x in items`, `for i in range(n)`,
  `for i in range(a, b)`, `break`, `continue`, `pass` y `return`. `a if cond else b` es una
  expresión. `and`, `or`, `not`, `in` y `not in` son los operadores.
* Los comentarios empiezan con `#`. Las cadenas usan `"` o `'`, y `f"..."` interpola `{expr}`.

## Argumentos con nombre y valores por defecto

Una llamada puede nombrar sus argumentos, y un parámetro o un campo puede tener un valor por defecto, de modo que una configuración
se lee como una frase y la llamada que solo necesita el caso común sigue siendo corta:

```telvax
type Query:
    text: Text
    top_k: Int = 5

fn search(q: Query, limit: Int = 10, rerank: Bool = False) -> List[Text]:
    []

let q = Query(text: "tensor shapes")  # top_k is 5
let hits = search(q, rerank: True)  # limit is 10
```

Los nombres se comprueban contra la función: un nombre mal escrito o repetido es un error de compilación.

## Pipelines

`x |> f(a)` significa `f(x, a)`, así que las etapas se leen de izquierda a derecha. Una línea que empieza con `|>` continúa
la línea anterior:

```telvax ignore
let answer = prompt |> llm.complete() |> std.text.trim()

let docs = query
    |> embed()
    |> store.search(3)
```

La regla es una sola: `x |> stage` es la llamada a `stage` con `x` como primer argumento, y a partir de ahí es una llamada
como cualquier otra.

* La etapa es una función (`x |> f`, `x |> std.text.trim`), una llamada (`x |> f(a)`) o una closure
  (`x |> (n => n + 1)`).
* Los nombres y los valores por defecto cuentan el valor que aporta el pipe: con `fn scale(x: Int, k: Int = 2)`, tanto
  `1 |> scale` como `1 |> scale(k: 3)` funcionan.
* Un sufijo después de la etapa se aplica al resultado de la llamada: `s |> parse()?` es `parse(s)?`,
  `n |> make().field` es `make(n).field`, y `n |> items()[0]` es `items(n)[0]`.
* `|>` tiene menos precedencia que cualquier operador: `a + b |> f` pasa la suma por el pipe, y `x |> f == 1` pasa a
  `f == 1`, lo cual es un error. Escribe `(x |> f()) == 1`.

## Tensores y tablas

```telvax
fn forward(x: Tensor[f32, 2, 3], w: Tensor[f32, 3, 2]) -> Tensor[f32, 2, 2]:
    x |> std.tensor.matmul(w) |> std.tensor.relu()  # [2,3] x [3,2] -> [2,2], checked

type Row:
    id: Int
    score: f32

fn best(t: Table[Row]) -> Table[Row]:
    t |> std.table.sql("SELECT * FROM t WHERE score > 0.5 ORDER BY score DESC")

fn mean_score(t: Table[Row]) -> f32:
    std.tensor.mean(t.score)  # a zero-copy column
```

`std.table.read_csv(path)`, `read_parquet` y `read_ipc` necesitan que el tipo del resultado esté escrito
(`let t: Table[Row] = ...`), igual que `std.tensor.zeros()`, `from_vec` y `reshape`.

Una tabla tiene métodos que conservan el tipo de sus filas. No necesitan SQL, así que también funcionan en `telvax-lite`:

```telvax
type Row:
    id: Int
    score: f32

fn top_two(t: Table[Row]) -> Table[Row]:
    t.sort_by_desc("score").head(2)

fn passed(t: Table[Row]) -> Table[Row]:
    t.filter(r => r.score > 0.5)
```

`head(n)` y `tail(n)` toman las primeras o las últimas `n` filas, `sort_by(column)` y `sort_by_desc(column)` ordenan por una
columna, `filter(keep)` conserva las filas para las que `keep` es verdadero, y `a.concat(b)` añade las filas de `b`
(las dos tablas necesitan el mismo tipo de fila). Un nombre de columna que no existe es un error de ejecución que lista las columnas.

`t.select(["name", "id"])` conserva solo esas columnas, en ese orden. Los nombres se comprueban mientras se
comprueba el programa, y el tipo de fila del resultado se calcula a partir de ellos, así que no hace falta ninguna anotación:

```telvax
type Person:
    id: Int
    name: Text
    age: Int

fn names(t: Table[Person]) -> List[Text]:
    var out: List[Text] = []
    for r in t.select(["name", "id"]):
        out.push(f"{r.id}: {r.name}")
    out
```

Los nombres deben escribirse como texto en una lista literal (no en variables), y una columna que no existe, o que se
nombra dos veces, es un error de compilación.

`t.group_by(keys, aggregations)` crea una fila por cada valor distinto de las columnas clave, en el orden de la primera aparición.
Cada agregación se escribe `column:function` y se convierte en una columna llamada `<column>_<function>`:

```telvax
type Sale:
    region: Text
    units: Int
    price: f64

fn report(t: Table[Sale]) -> List[Text]:
    var out: List[Text] = []
    for r in t.group_by(["region"], ["units:sum", "price:mean"]).sort_by_desc("units_sum"):
        out.push(f"{r.region}: {r.units_sum} units, {r.price_mean} on average")
    out
```

Las funciones son `sum` (un `Int` para una columna de enteros, un `Float` para una decimal), `mean` (un `Float`), `min` y
`max` (el tipo de la columna) y `count` (un `Int`, para cualquier columna). El tipo de fila del resultado se calcula a partir de los
nombres, como en `select`. Una columna o función desconocida, una columna de tipo `T?`, una suma o media de texto y una columna de
resultado con nombre repetido son errores de compilación. La misma consulta en SQL sigue funcionando; `group_by` también se ejecuta en `telvax-lite`.

Una columna de tipo `T?` puede tener valores vacíos: `temp: f64?` lee las celdas vacías de un CSV, los nulos de Parquet y
el `None` de `from_rows` como `None`, y `t.temp` pasa a ser una `List[f64?]` (no un tensor). Una columna sin `?`
que encuentra un valor vacío es un error que nombra la columna y dice que escribas `?`.

```telvax
type Reading:
    id: Int
    temp: f64?

fn warm(t: Table[Reading]) -> Table[Reading]:
    t.filter(r => r.temp.unwrap_or(0.0) > 20.0)
```

`Date` y `Timestamp` son tipos de columna para fechas y horas. En la tabla son `Date32` y
`Timestamp(µs, UTC)` de Arrow, así que Parquet, SQL y PyArrow ven fechas reales. En el programa un valor es un `Text` en ISO-8601:
`"2026-10-17"` y `"2026-10-17T16:53:03Z"`. `from_rows` y el CSV aceptan cualquier desfase (`+02:00`) y guardan en UTC; un
texto que no es una fecha es un error que dice que escribas `YYYY-MM-DD`. Todavía no hay aritmética de fechas: compara
el texto, ordena la tabla o usa SQL.

```telvax
type Event:
    id: Int
    day: Date
    at: Timestamp?

fn latest(t: Table[Event]) -> Table[Event]:
    t.sort_by_desc("day").head(1)
```

`std.table.read_ipc(path)` y `write_ipc(table, path)` leen y escriben un archivo Arrow IPC (Feather v2), que PyArrow,
Polars y pandas también leen y escriben. `read_ipc` mapea el archivo en memoria y las columnas apuntan dentro de él, así que un
archivo con un solo lote no se copia. No reescribas ni trunques un archivo mientras se esté usando una tabla leída de él. Los archivos
comprimidos con LZ4 o Zstd se decodifican (con copia) en el `telvax` completo; `telvax-lite` lee archivos sin comprimir.

Una tabla puede cortarse en trozos. `t.split(n)` da una `List[Table[Row]]` de tablas con como máximo `n` filas cada una;
comparten los buffers de `t`, así que no se copia nada. `std.table.read_csv_batches(path, n)`,
`read_parquet_batches` y `read_ipc_batches` leen un archivo directamente en esos trozos (no se construye una copia unida del
archivo entero), y necesitan que el tipo del resultado esté escrito: `let parts: List[Table[Row]] = ...`. Los trozos están todos en
memoria a la vez; todavía no existe un lector que los entregue de uno en uno.

```telvax
type Row:
    id: Int
    score: f32

fn total_score(path: Text) -> f32:
    let parts: List[Table[Row]] = std.table.read_parquet_batches(path, 100000)
    var sum = 0.0
    for part in parts:
        sum += part.score.sum()
    sum
```

`std.table.read_flight(url, ticket)` obtiene una tabla de un servidor Arrow Flight (`DoGet`), y
`std.table.write_flight(table, url, path)` envía una (`DoPut`). La dirección es `grpc://host:port`, o
`grpc+tls://host:port` para TLS. Ambas necesitan el permiso `net` y una compilación con la feature `flight`
(`cargo install telvax --features flight`); los binarios publicados no la incluyen, y sin ella lo
indican. Hay solo un cliente, no un servidor Flight.

## Endpoints HTTP

```telvax ignore
@endpoint(POST, "/v1/score")
fn score(input: In) -> Result[Out, Status]:
    ...
```

El tipo del parámetro se convierte en el cuerpo JSON validado (GET/DELETE lo leen de la query string).
Devolver `Err(Status.BadRequest("..."))` se convierte en un HTTP 400.

Un campo con un valor por defecto escrito como literal (`language: Text = "en"`) o de tipo `T?` puede omitirse en el
cuerpo; toma el valor por defecto, o `None`. Cualquier otro campo es obligatorio.

Un segmento `{name}` en la ruta se lee de la URL hacia el parámetro con ese nombre:

```telvax ignore
@endpoint(GET, "/v1/results/{job_id}")
fn result(job_id: Text) -> Result[Job, Status]:
    ...

@endpoint(PUT, "/v1/jobs/{id}/steps/{step}")
fn update(id: Int, step: Text, input: Patch) -> Result[Job, Status]:
    ...
```

Un parámetro de ruta es `Text`, un tipo numérico o `Bool`. Un valor que no se convierte es un HTTP 400:
`invalid path parameter id: expected i64, found `x``. El parámetro que no está en la ruta es el
payload, y hay como máximo uno. `/a/{x}` y `/a/{y}` son la misma ruta.

Un parámetro de tipo `Request` recibe la petición. No forma parte del payload, y hay como máximo uno:

```telvax ignore
import std.http.Request

@endpoint(DELETE, "/v1/cache")
fn clear(req: Request) -> Result[Cleared, Status]:
    if req.header("x-admin-token") != Some(token):
        return Err(Status.Unauthorized("invalid token"))
    ...
```

`req.header(name)` es el primer valor de la cabecera, o `None`; el nombre no distingue mayúsculas de minúsculas.
`req.method()` y `req.path()` son `Text`.

Los valores que viven fuera de una función se comparten entre todas las peticiones. Un `let` de nivel superior que guarda una `List` o un
`Map` puede cambiarse desde los endpoints, y cada `push`, `insert` o `remove` es atómico, así que una caché o una tabla de
jobs no necesita nada más:

```telvax ignore
let cache: Map[Text, Scores] = {}
let stored_at: Map[Text, Float] = {}

fn lookup(text: Text) -> Scores?:
    match stored_at.get(text):
        case Some(t):
            if std.time.now() - t < 300.0:
                cache.get(text)
            else:
                None
        case None:
            None
```

Un `var` de nivel superior no está permitido, así que un contador es un `Map` con una clave. Dos llamadas seguidas sobre el mismo valor
no son atómicas en conjunto: otra petición puede llegar entre el `get` y el `insert`. No hay límite de tamaño ni caducidad; elimina
tú mismo las claves antiguas.

Una caché con límite y caducidad es `std.cache.new(max, ttl_seconds)`. Descarta la entrada usada hace más tiempo cuando está
llena, olvida una entrada `ttl_seconds` después de guardarla, y se comparte entre todas las peticiones como el `Map` de arriba, así que la tabla de
`stored_at` y el borrado a mano no hacen falta:

```telvax ignore
let cache = std.cache.new(10000, 300.0)

fn lookup(text: Text) -> Scores?:
    cache.get(text)

fn remember(text: Text, scores: Scores):
    cache.set(text, scores)
```

`get`, `set`, `remove(key)`, `clear()` (cuántas entradas tenía) y `len()`. Las claves son `Text`. Los valores todavía no tienen el tipo comprobado:
`get` da una opción de lo que se guardó. El límite se mantiene por shard (una caché de 10.000 entradas tiene 16 shards), así que la
entrada que sale es la menos usada recientemente de su shard.

Un endpoint responde `200` con el valor como JSON. Para elegir el estado o añadir cabeceras, devuelve una `Response`:

```telvax ignore
import std.http.Response

@endpoint(POST, "/v1/analyze_batch")
fn submit(input: Batch) -> Result[Response, Status]:
    let job = start(input)
    Ok(Response.new(Accepted(job_id: job.id))
        .status(202)
        .header("location", "/v1/results/" + job.id))
```

`Response.new(body)` es un `200`; `.status(code)` toma un número de 100 a 599 (cualquier otro es un HTTP 500) y
`.header(name, value)` define una cabecera, reemplazando una del mismo nombre, `content-type` incluido.
`Err(Status...)` sigue dando los estados de error.

### Trabajo después de la respuesta

`std.task.spawn(task)` inicia una `async fn` y vuelve de inmediato; la tarea sigue después de que el endpoint haya respondido, de modo que
un servicio puede responder `202` y terminar más tarde mientras el cliente consulta:

```telvax ignore
let jobs: Map[Text, Text] = {}

async fn work(id: Text, texts: List[Text]):
    # ... the slow part ...
    jobs.insert(id, "done")

@endpoint(POST, "/v1/jobs")
fn start(input: Batch) -> Result[Response, Status]:
    let id = new_id()
    jobs.insert(id, "running")
    std.task.spawn(work(id, input.texts))
    Ok(Response.new(Accepted(job_id: id)).status(202))
```

* El valor de la tarea se descarta. Un error en ella se escribe en la salida de error estándar como `background task failed: ...`,
  a menos que un segundo argumento diga qué hacer con él: `std.task.spawn(work(id), failed)` llama a `failed(message)`, una
  función de un `Text`, para que el job pueda marcarse como fallido en lugar de quedarse en `running`. Una tarea que debe informar de un
  resultado lo escribe donde el programa lo lee, como `jobs` arriba.
* Como máximo 256 tareas se ejecutan a la vez; una más es un error en el endpoint que la inició.
* La tarea comparte los `List` y `Map` de nivel superior del programa (mira la nota sobre el estado compartido más arriba). Termina con el
  proceso: detener el servidor la detiene.
* Solo un servidor mantiene la tarea en marcha. En `telvax run` y `telvax test`, `spawn` espera a la tarea, como hace `std.task.run`.

### Server-sent events

`Response.events(producer)` responde con `text/event-stream`. El servidor llama a `producer` con un `Sink` mientras el
cliente lee, y cada `sink.send(...)` es un evento, enviado de inmediato:

```telvax ignore
import std.http.Response
import std.http.Sink

@endpoint(POST, "/v1/analyze_stream")
fn stream(input: Batch) -> Result[Response, Status]:
    fn produce(sink: Sink):
        for text in input.texts:
            sink.send(score(text))
        sink.event("done", "{}")
    Ok(Response.events(produce))
```

* `send(data)` toma un `Text`, enviado tal cual, o cualquier otro valor, enviado como JSON. `event(name, data)` añade una
  línea `event:` y `comment(text)` escribe una línea que los clientes ignoran, para mantener abierta la conexión.
* El productor es una función escrita dentro del endpoint (una closure es una línea), así que puede usar los parámetros.
* El estado y las cabeceras se envían primero, así que un error en el productor no puede cambiarlos: el stream termina con
  `event: error` y el mensaje. Un cliente que se desconecta hace fallar el siguiente `send`, lo que termina el productor.
* `telvax serve` y `build --serve` hacen stream; el backend nativo no sirve HTTP.

## SQL en PostgreSQL

`std.db.postgres.PostgresClient` ejecuta SQL en un servidor PostgreSQL. `execute` da el número de filas que una sentencia
cambió, y `query` da las filas como una lista de un `type` que escribes tú:

```telvax ignore
import std.db.postgres.PostgresClient

type Job:
    id: Text
    status: Text
    attempts: Int
    result: Text?

let db = PostgresClient.connect(std.env.get("DATABASE_URL").unwrap())

fn finish(id: Text, result: Text) -> Result[Int, Text]:
    db.execute("UPDATE jobs SET status = $1, result = $2 WHERE id = $3", ("done", result, id))

fn queued() -> Result[List[Job], Text]:
    let rows: List[Job] = db.query("SELECT id, status, attempts, result FROM jobs WHERE status = $1", ["queued"])?
    Ok(rows)
```

* Los valores de `$1`, `$2` se pasan aparte del SQL, como lista cuando son de un mismo tipo y como tupla cuando
  no lo son. Los enlaza el servidor, así que un valor no puede cambiar la sentencia. `None` es `NULL`.
* Un campo se lee de la columna del mismo nombre, y `Text?` permite `NULL`. Una columna que no es un campo se
  ignora; un campo sin columna es un error. Las columnas `json` y `jsonb` son `Text`. Una columna `timestamptz`, `numeric` o
  `uuid` es un error que te dice que hagas el cast en el SQL (`created_at::text`).
* Un error del servidor es `Err(Text)` con su mensaje, como el nombre de la constraint que falló.
* La conexión la abre la primera sentencia y se mantiene. Hay una por cliente, usada una sentencia a la vez,
  y todavía no hay transacciones.
* Necesita `--live` y el permiso `net`: no hay sustituto sin conexión para SQL.

### Métricas y logs

`telvax serve app.tvx --metrics --log-format json` sirve `GET /metrics` (peticiones y latencia por ruta) y escribe una
línea JSON por petición con un id de petición. Un programa cuenta lo que solo él sabe con `std.metrics.count`, que el
servidor lista con su propio nombre, `cache_hit`:

```telvax ignore
fn lookup(key: Text) -> Text?:
    match cache.get(key):
        case Some(v):
            std.metrics.count("cache_hit")
            Some(v)
        case None:
            std.metrics.count("cache_miss")
            None
```

`--cors "*"`, `--rate-limit 100/min` y `--security-headers` ponen las protecciones habituales alrededor de cada respuesta
sin un proxy inverso. Los detalles están en la [referencia de la línea de comandos](https://github.com/telvax/telvax/blob/main/docs/reference/cli.md).

Un contador puede tener etiquetas, un `Map[Text, Text]`: `std.metrics.count("jobs_total", {"status": "done"})` se lista como
`jobs_total{status="done"}`, con `by` como tercer argumento. El nombre son letras, dígitos, `_` y `:`, un contador tiene como máximo
1.000 conjuntos de etiquetas (así que una etiqueta no debe hacerse con lo que envía un usuario), y `--process-time-header` añade
`x-process-time-ms` a cada respuesta.
Un número que sube y baja, como las entradas de una caché, es un gauge: `std.metrics.set("cache_size", cache.len())` reemplaza su valor cada vez, y el servidor lo lista con `# TYPE cache_size gauge`. Un nombre es un contador o un gauge, no ambos.
`--error-format detail` escribe los errores que produce el servidor como `{"detail": ...}`, y un cuerpo que no encaja en su tipo como `422`,
como hace FastAPI; el valor por defecto es `{"error": ...}` y `400`. Una respuesta que el programa construye con `Response` nunca se cambia.
Una función que falla en tiempo de ejecución responde `500` con `Internal server error`; el motivo va al log con el id de la petición.

Un programa escribe sus propias líneas con `std.log.debug`, `info`, `warn` y `error`, en el formato de `--log-format` y en el
nivel de `--log-level`, con campos opcionales: `std.log.warn("slow query", {"ms": "840"})`. Las líneas de petición también reciben un `level`
(`info` por debajo de 400, `warn` para 4xx, `error` para 5xx), así que `--log-level warn` conserva en ambos lo que falló.

Las opciones del servidor pueden escribirse en el programa en lugar de en la línea de comandos, de modo que un archivo dice cómo
se comporta el servicio:

```
serve:
    metrics: True
    cors: "*"
    rate_limit: "100/min"
    security_headers: True
    error_format: "detail"
```

Los valores son fijos, o se leen del entorno cuando el servidor arranca: `cors: env("CORS_ORIGINS", "*")` es la variable
`CORS_ORIGINS`, o `*` cuando no está definida o está vacía (el valor por defecto es opcional; sin él, la línea es como si no estuviera).
El texto de la variable se lee como lo lee la flag, así que `log_format: env("LOG_FORMAT", "json")` acepta `json` o `text`,
y `metrics: env("METRICS")` se activa con `1` o `true`. Una variable `TELVAX_*` o una flag prevalece sobre el bloque, y `--port`, `--host` y las demás opciones
sobre dónde se ejecuta el programa siguen siendo flags. La lista está en la [referencia de la línea de comandos](https://github.com/telvax/telvax/blob/main/docs/reference/cli.md).

Tu propio middleware envuelve cada petición. Una función marcada con `@middleware` recibe la petición y `next`, el resto de la
cadena, y da una `Response`; la línea `middleware:` del bloque los lista, el más externo primero:

```telvax ignore
import std.http.Next
import std.http.Request
import std.http.Response

@middleware
fn version_header(req: Request, next: Next) -> Response:
    next(req).header("x-api-version", "1")

@middleware
fn admin_only(req: Request, next: Next) -> Response:
    if req.path() == "/admin" and req.header("x-admin-token") != Some(admin_token):
        return Response.new(Denied(detail: "Invalid admin token")).status(401)
    next(req)

serve:
    middleware: [admin_only, version_header]
```

`next(req)` ejecuta el siguiente middleware y luego la ruta, y da su `Response`, cuyo estado y cabeceras puedes cambiar.
Un middleware que no llama a `next` responde por sí mismo, y la ruta no se ejecuta. Los integrados (`metrics`, `cors`,
`rate_limit`, ...) quedan por fuera y conservan sus opciones. Un middleware se ejecuta en el intérprete en cada petición, `/health` y
`/metrics` incluidos (comprueba `req.path()` para dejar una ruta fuera), y no puede leer ni cambiar el cuerpo.

La única línea del bloque que no es un valor es `on_start`, una lista de funciones que se ejecutan una vez, en orden, antes de que el servidor
escuche: crear una tabla, calentar una caché, comprobar un ajuste. Una función no recibe parámetros y no devuelve nada, o devuelve un
`Result[(), Text]`; un `Err` detiene el arranque, con su mensaje en la salida de error estándar y estado de salida 1.

```telvax ignore
fn init_db():
    db.execute("CREATE TABLE IF NOT EXISTS jobs (id TEXT PRIMARY KEY)", [])

serve:
    on_start: [init_db]
```

### El documento de la API

Los tipos de los endpoints ya describen la API, así que el servidor la publica. `GET /openapi.json` es un documento OpenAPI 3.0:
cada ruta con sus parámetros de ruta, los parámetros de consulta o el cuerpo JSON, el tipo de la respuesta y
la respuesta de error (`{"error": "..."}`). `GET /docs` es una página que los lista. Ambos están en todos los servidores, y si el
programa tiene un endpoint propio en una de esas rutas, se sirve ese en su lugar. `GET /healthz` responde `ok`, o `503` mientras el servidor se está deteniendo.

Cuando el servidor recibe `SIGTERM` o `Ctrl-C`, deja de aceptar conexiones, termina las peticiones que tiene y espera a las tareas
iniciadas con `std.task.spawn`, como máximo `--shutdown-wait N` segundos (30 por defecto, o `TELVAX_SHUTDOWN_WAIT`). Una segunda
señal sale de inmediato. Si quedó trabajo sin terminar, el estado de salida es 1.

Una función que devuelve una `Response` no tiene respuesta tipada, así que su `200` no tiene schema. Un campo de tipo `T?` o
`Date` se describe como se comprueba, con no más que su tipo JSON.

## Módulos y paquetes

```telvax ignore
from util import clamp  # ./util.tvx
from mathx.vec import dot  # dependency `mathx`, file vec.tvx
```

Un `let` de nivel superior se importa del mismo modo (`from settings import LIMIT`); es un único valor para todo el programa, así que una
`List` o un `Map` cambiado en un módulo cambia en todos. `import util.clamp` trae el mismo nombre. Un nombre puede renombrarse con `as`:
`from util import clamp as limit`, `import util.clamp as limit`, `from std.text import trim as strip`.
Los nombres de `std` no necesitan import: `std.text.trim(s)`. Un módulo no es un nombre: `import util` es un error
que dice que escribas `from util import name`.

```sh
tvpm new app && cd app
tvpm add mathx --registry ~/registry     # or --path ../mathx
tvpm install --registry ~/registry       # resolves, writes telvax.lock, downloads
tvpm run                                  # runs src/main.tvx with the dependencies
tvpm update                               # newest versions that fit; `tvpm update mathx` moves only mathx
```

## Distribución

```telvax ignore
fn work(item: Job) -> Done:
    ...

fn run_all(jobs: List[Job]) -> List[Done]:
    std.dist.map(jobs, work)
```

`std.dist.map` toma el *nombre* de una función de nivel superior. Se ejecuta en hilos locales por defecto;
`telvax run app.tvx run_all --workers host1:7077,host2:7077` envía las llamadas a máquinas que ejecutan
`telvax worker`, y `telvax.ray.RayExecutor` (Python) las envía a Ray.

## Cuantización

`--quantize int8` (en `run`, `serve`, `jit`, `build`) hace que los productos de matrices `f32` usen aritmética int8 simétrica
por tensor con acumulación en `i32`. El intérprete y el backend nativo dan
resultados idénticos.

`--quantize int8-channel` y `--quantize int4-channel` (`run` y `serve`; el backend nativo
los rechaza) dan a cada fila de las activaciones y a cada columna de los pesos su propia escala, de modo que
una columna mucho mayor o menor que el resto ya no pierde su precisión. El modo int4
guarda los pesos en 4 bits. Las escalas se calculan en cada llamada; no hay paso de calibración.

## Aceleradores

* `--device gpu` (en `run` y `serve`, necesita `cargo build --features gpu`) ejecuta operaciones de tensor `f32` en la GPU mediante `wgpu`: Metal en macOS, Vulkan en Linux, DX12 en Windows.
* `telvax build file.tvx --emit stablehlo` imprime StableHLO para las funciones de tensor del archivo, para XLA / PJRT.
* `telvax build file.tvx --emit ptx [--sm sm_80]` imprime kernels PTX para las operaciones de tensor que usa el archivo (necesita `--features llvm`).

PTX y StableHLO se generan y se comprueban, no se ejecutan: mira `docs/design/status.md`.

## Enums, `match`, `break` y `continue`

```telvax
enum Shape:
    Circle(Float)
    Rect(Float, Float)
    Empty

fn area(s: Shape) -> Float:
    match s:
        case Shape.Circle(r):
            r * r * 3.14159
        case Shape.Rect(w, h):
            w * h
        case Shape.Empty:
            0.0

fn first_over(limit: i32) -> i32:
    var found = -1
    for i in range(1, 1000):
        if i * i > limit:
            found = i
            break
    found
```

`match` debe cubrir todos los casos (el compilador nombra las variantes que faltan) y también funciona con
`Result` (`Ok(v)` / `Err(e)`), enteros, cadenas, booleanos y `_`. Los enums viajan como JSON: una variante
sin valores es su nombre (`"Empty"`), una con valores es `{"Circle": 2.0}` o `{"Rect": [2.0, 3.0]}`.
Los enums se ejecutan en el intérprete; el backend nativo todavía no los compila.

## Limitar lo que un programa puede hacer

Un programa puede leer y escribir archivos, leer variables de entorno y acceder a la red. `--allow`
en `run`, `test` y `serve` lista lo que puede hacer; sin la flag, todo está permitido.

```sh
telvax run job.tvx main --allow fs.read          # may read files, nothing else
telvax test app.tvx --allow none                 # may touch nothing outside itself
telvax serve app.tvx --live --allow net,env
```

| Palabra | Permite |
|---|---|
| `fs.read` | `std.fs.read`, `std.fs.exists`, `read_csv`, `read_parquet`, `read_ipc` y sus formas `_batches` |
| `fs.write` | `std.fs.write`, `write_csv`, `write_parquet`, `write_ipc` |
| `fs` | `fs.read` y `fs.write` |
| `env` | `std.env.get` |
| `net` | los clientes de Ollama, OpenAI y PostgreSQL, con `--live`, y `read_flight` / `write_flight` |
| `all`, `none` | todo, nada |

Una llamada no permitida detiene el programa y dice qué palabra añadir:
`` `std.fs.read` needs the `fs.read` capability; run with `--allow fs.read` ``. El límite se comprueba cuando
el programa se ejecuta, no cuando se compila, y `std.io.print` y `std.time` están siempre permitidos.

### Archivos dentro del binario

`std.fs.embed("path")` es un `Text` con el contenido de un archivo, leído cuando se construye el programa y guardado dentro de él.
La ruta es un literal de texto, relativa al directorio del archivo principal, así que `telvax build --serve` hace un solo binario
que no necesita archivos de datos a su lado:

```telvax
let lexicon = std.fs.embed("lexicons/vader.txt")
```

No necesita ningún permiso, porque el archivo se lee al construir, no por el programa en ejecución. Un archivo que no
existe, o una ruta que no es un literal (`"data/" + name`), es un error cuando se comprueba el programa.

## Tests

Una función llamada `test_...` es un test. `assert(cond)` y `assert_eq(a, b)` lo detienen
con un mensaje cuando fallan:

```telvax
fn double(x: Int) -> Int:
    x * 2

fn test_double():
    assert_eq(double(2), 4)
```

```sh
telvax test file.tvx           # run them all, exit 1 if one fails
telvax test file.tvx double    # only the tests whose name contains `double`
telvax test tests/             # every `test_*.tvx` and `*_test.tvx` file under a directory
telvax test                    # the same, in this directory
```

### Fixtures

Un test que recibe parámetros los obtiene de fixtures. `fn test_total(order: Order)` llama a
`fixture_order()` y pasa el resultado; una fixture puede recibir otras fixtures del mismo modo. Cada test
recibe valores nuevos. Una función `teardown_order(order: Order)`, si existe, se ejecuta después del test,
haya pasado o no.

```telvax
fn fixture_base() -> Int:
    10

fn fixture_total(base: Int) -> Int:
    base * 2

fn teardown_total(total: Int):
    print(f"done with {total}")

fn test_total(base: Int, total: Int):
    assert_eq(total, base * 2)
```

Un parámetro sin función `fixture_...`, o una fixture que devuelve otro tipo, hace fallar el test
y dice qué escribir.

### Casos

Para ejecutar un test con varios valores, escribe una función `cases_...` con el nombre del test sin
`test_`. Devuelve una lista; el test se ejecuta una vez por cada elemento. Una tupla da los primeros parámetros del
test, y los parámetros que sobran siguen siendo fixtures.

```telvax ignore
fn cases_double() -> List[(Int, Int)]:
    [(1, 2), (2, 4), (3, 6)]

fn test_double(x: Int, want: Int):
    assert_eq(double(x), want)
```

Cada ejecución se lista como `test_double[(2, 4)]`. Una función `cases_...` que está vacía, o cuyos elementos no
encajan con los parámetros, hace fallar el test y dice por qué.

### Mocks

`std.test.mock(target, replacement)` hace que un test llame a otra función en lugar de una. El objetivo es
una función tuya, escrita sin comillas, o una función de std por su ruta en texto. El objetivo que es una función
debe tener el mismo tipo que su sustituta; un objetivo de std se comprueba cuando el test se ejecuta.
`std.test.calls(target)` dice cuántas veces se llamó a la sustituta. Todo mock se elimina cuando
el test termina.

```telvax
fn price(item: Text) -> Int:
    100

fn total(item: Text) -> Int:
    price(item) * 2

fn fake_read(path: Text) -> Result[Text, Text]:
    Ok("from the test")

fn test_total():
    std.test.mock(price, item => 7)
    assert_eq(total("a"), 14)
    assert_eq(std.test.calls(price), 1)

fn test_read():
    std.test.mock("std.fs.read", fake_read)
    assert_eq(std.fs.read("any.txt"), Ok("from the test"))
```

Una sustituta no puede llamar a la función que reemplaza. Las llamadas a través de métodos (`x.len()`) no pueden
mockearse, y el propio `std.test.*` tampoco.

### Cobertura

`--coverage` lista, para cada archivo del proyecto, las líneas con código y cuántas de ellas ejecutaron los tests,
y los números de las líneas que no se ejecutaron. `--fail-under 90` también hace fallar la ejecución, con código de
salida 1, cuando se ejecutaron menos del 90% de las líneas, de modo que un pipeline puede mantener el nivel.

```sh
telvax test tests/ --coverage
telvax test tests/ --fail-under 90
```

```text
coverage (lines that ran / lines with code):
  src/price.tvx  18/20  90.0%   missing 14-15
  total  18/20  90.0%
```

Una línea cuenta cuando tiene una sentencia, la última expresión de un bloque o el cuerpo de un `case`. Las funciones
`test_`, `fixture_`, `teardown_` y `cases_` no se cuentan, ni tampoco los archivos fuera del
proyecto (el directorio con `telvax.toml`, o el que pruebas). El recuento es de líneas, no de
ramas: una línea con un `if` y sus dos resultados escritos en ella cuenta una vez.

## Macros

Una macro se escribe `name!(...)`. El compilador tiene cinco y no puedes definir más. Cada una se
reescribe como una llamada a una función de `std` antes de que se compruebe el programa.

| Macro | Qué hace |
|---|---|
| `assert!(cond)` | como `assert`, y el mensaje cita la condición: `assertion failed: x > 3` |
| `assert!(cond, "message")` | se detiene con tu mensaje cuando `cond` es falso |
| `dbg!(value)` | imprime `value_as_written = result` en stderr y devuelve `value` |
| `todo!()` | se detiene con `not yet implemented`; puede estar donde se espere cualquier tipo |
| `unreachable!()` | se detiene con `unreachable code was reached` |
| `panic!("message")` | se detiene con el mensaje |

```telvax
fn pick(n: Int) -> Int:
    if n > 0:
        n
    else:
        todo!()

fn test_pick():
    let x = dbg!(pick(4))  # stderr: pick(4) = 4
    assert!(x == 4)
```

Las macros se ejecutan en el intérprete, como `assert`; el backend nativo no las compila.

## El REPL

```sh
telvax repl                    # an interactive session
telvax repl app.tvx            # start with the definitions and variables of a file
```

```text
>>> let xs = [3, 1, 2]
>>> xs.len()
3
>>> fn total(v: List[Int]) -> Int:
...     v.sum()
...
>>> total(xs)
6
>>> var n = 0
>>> for x in xs:
...     n = n + x
...
>>> n
6
>>> :type xs
List[Int]
```

Puedes escribir una definición (`fn`, `type`, `enum`, `trait`, `extend`, `import`), una variable (`let`, `var`), una
sentencia o una expresión; se muestra el valor de una expresión. Un bloque termina con una línea vacía. Los nombres duran
hasta `:reset`: un nombre puede definirse de nuevo (`let x = x + 1`, o una nueva `fn` con el mismo nombre), y a un `let`
no se le puede asignar, como en un archivo. Cada entrada se comprueba con todo lo definido antes, así que un error de tipo
muestra la línea y deja la sesión como estaba. Un número escrito solo (`let n = 4`) es un `Int`, para que las
funciones que reciben un `Int` lo acepten. Una closure necesita que se indique el tipo de sus parámetros para conservarla
(`let inc = (n: Int) => n + 1`).

Los comandos empiezan con dos puntos: `:type <expression>`, `:vars`, `:defs`, `:load <file>`, `:reset`, `:help` y
`:quit`. Tab completa nombres, campos y métodos, y las flechas recorren las entradas anteriores (guardadas en
`~/.telvax_history`; `TELVAX_HISTORY` indica otro archivo). `--allow` limita lo que la sesión puede hacer, como en
`telvax run`. Cada entrada se ejecuta una vez: lo que imprime o escribe ocurre una vez, y los valores no se calculan de nuevo
cuando se introduce la siguiente entrada.

## Depuración

```sh
telvax dap       # a debug adapter on standard input and output
```

En VS Code, F5 sobre un archivo Telvax lo depura (mira el README de la extensión). `telvax dap` habla el [Debug Adapter Protocol](https://microsoft.github.io/debug-adapter-protocol/), así que un
editor que tenga un cliente para él puede ejecutar un programa en el intérprete, detenerlo en breakpoints y avanzar paso a paso
por él. Su petición de launch recibe `program` (el archivo `.tvx`), `function` (por defecto `main`), `args` (las palabras
que darías a `telvax run`), `stopOnEntry`, `noDebug` y `allow` (la misma lista que `--allow`). Se detiene
antes de cada sentencia, así que un breakpoint va en una línea que empieza una sentencia; en cualquier otra línea se mueve
a la siguiente que la empieza. Mientras está detenido lista las llamadas en curso y sus variables, y un nombre o
un campo de una de ellas (`p.x`, `xs.0`) puede evaluarse; otras expresiones no. Depura el intérprete, no los
programas de `telvax build`, y no tiene breakpoints condicionales.

## Profiling

```sh
telvax profile app.tvx                       # runs `main` and reports where the time and memory went
telvax profile app.tvx handler 3 "text"      # another function, with its arguments
telvax profile app.tvx --top 30 --json r.json --collapsed stacks.txt --speedscope p.json
```

El informe tiene cuatro partes: el tiempo total y la memoria (el pico, los bytes asignados y cuántas
asignaciones); las funciones, con llamadas, el tiempo en la propia función (`self`) y con las llamadas que
hace (`total`), y los bytes que asignó; las sentencias, con cuántas veces se ejecutó cada una y cuánto tardó
hasta que empezó la siguiente de la misma llamada; y las funciones de `std` (`std.tensor.matmul`,
`std.table.sql`, ...), con llamadas, tiempo, las filas o elementos que entraron y salieron, y los bytes del
resultado. `--json` escribe todo eso en un archivo, `--collapsed` escribe pilas de llamadas para `flamegraph.pl` o
`inferno`, y `--speedscope` escribe un archivo para <https://www.speedscope.app>.

Los tiempos los mide el intérprete alrededor de lo que hace el programa, así que muestran dónde gasta su tiempo
el *programa* e incluyen el tiempo del propio intérprete. La memoria cuenta el proceso entero, así que con
varios hilos una función también cuenta lo que los otros hilos asignan mientras tanto. Los programas construidos con
`telvax build` no se perfilan.

## Documentación

`telvax schema data.csv` imprime el `type` de las columnas de un archivo CSV, Parquet o Arrow, para que no
lo escribas a mano: una columna con un valor vacío sale como `T?`, y `--name Row` elige el nombre del tipo.

`telvax doc` escribe una referencia en Markdown a partir del código: cada función con su firma, cada `type` con
una tabla de campos (y valores por defecto), cada `enum` con sus variantes, cada `trait`, y los métodos de `extend`.
Las líneas de comentario `#` escritas justo encima de un elemento (sin línea en blanco entre ellas) se convierten en su descripción.

```sh
telvax doc file.tvx                # print the reference of one file
telvax doc src/ -o REFERENCE.md    # every `.tvx` file under the directory, written to a file
telvax doc --public src/           # leave out names that start with `_`
```

```tvx
# Sum of the first `n` squares.
fn squares(n: Int) -> Int:
    n * (n + 1) * (2 * n + 1) / 6
```

## Formato y lint

```sh
telvax fmt file.tvx            # rewrite in place
telvax fmt src/ tests/         # every `.tvx` file under the directories (no path: this directory)
telvax fmt --check             # list files that would change (exit 1), for CI
telvax fmt --diff src/         # show the lines that would change, write nothing (exit 1)
telvax lint file.tvx           # unused variables, unused private functions, unreachable cases
telvax lint --list-rules       # the rules and what they find
```

Cada regla de lint tiene un nombre (`unused-variable`, `unused-function`, `unreachable-case`) y un nivel:
`allow` (no decir nada), `warn` (el valor por defecto) o `deny` (un error, de modo que `telvax lint` falla). Define los
niveles de un paquete en `telvax.toml`, o para una ejecución con `--allow rule`, `--warn rule` y
`--deny rule`; una flag prevalece sobre el archivo. `--deny-warnings` hace que cualquier aviso falle la ejecución.

```toml
[lint]
unused-variable = "deny"
unused-function = "allow"
```

La sangría forma parte de la sintaxis, así que `telvax fmt` nunca mueve una línea entre bloques: fija cuatro
espacios por nivel, los espacios dentro de una línea, las líneas de continuación y las líneas en blanco. Los directorios omiten las carpetas
ocultas, `target`, `node_modules` y `.tvpm`.

El servidor de lenguaje completa nombres mientras escribes: tipos después de `:` y `->`, los campos, columnas y
métodos después de un punto (también después de llamadas como `s.trim().`), las variantes después de `Shape.`, el parámetro de
una closure que estás escribiendo (`xs.map(p => p.`) y las columnas de una tabla dentro de `t.sql("..`. Lo que encaja con el
tipo que el código espera va primero.

El servidor de lenguaje también formatea documentos, así que VS Code puede formatear al guardar. También encuentra todas las
referencias a una variable, función, global, tipo, variante de enum, campo o método y las renombra (un
renombrado se rechaza cuando el nuevo nombre chocaría con otro del ámbito, cuando un archivo tiene un error de
sintaxis, o cuando el nombre pertenece a una dependencia). Siguen los imports hacia los demás archivos del proyecto.

## Cadenas y listas

```telvax
fn demo(s: Text) -> u64:
    let words = s.split(" ")  # List[Text]
    words.push("extra")
    if s.contains("telvax") and not s.starts_with("#"):
        return words.len()
    0
```

`Text`: `len`, `is_empty`, `trim`, `to_lowercase`, `to_uppercase`, `contains`, `starts_with`,
`ends_with`, `replace`, `split`. `List`: `len`, `is_empty`, `push`, `contains`. `len` es un `u64`, y junto a un número con signo (`xs.len() > limit`, `xs.len() - 1`) ambos se leen como `Int`. `x in xs` es
`xs.contains(x)`. Un método que no existe es un error de compilación con una pista de "quizá quisiste decir".
Estos métodos se ejecutan en el intérprete; el backend nativo todavía solo tiene `len` e `is_empty`.

## Valores aleatorios

`std.random.uuid()` es un identificador nuevo cada vez, como `"3f2b8c1e9a4d4b7f8e6a0c5d1b2a3f4e"`, y
`std.random.int(1, 6)` un número entero de 1 a 6, ambos incluidos. Vienen del sistema operativo, así que un id sirve
para nombrar un job cuyo resultado solo debe leer su dueño. No necesitan permiso, y no hay semilla.

## Valores opcionales, maps y funciones pequeñas

```telvax
fn demo(text: Text) -> Int:
    let counts: Map[Text, Int] = std.map.new()
    for word in text.split(" "):
        let seen = counts.get(word).unwrap_or(0)  # Int?
        counts.insert(word, seen + 1)
    match counts.get("telvax"):
        case Some(n):
            n
        case None:
            0
```

* `T?` es `Some(value)` o `None`; `match` debe cubrir ambos. Métodos: `is_some`, `is_none`,
  `unwrap` (un error con `None`), `unwrap_or`, `unwrap_or_else(() => v)`, `expect("message")`,
  `map`, `and_then`, `filter` y `ok_or(error)`, que lo convierte en un `Result`. Varios métodos devuelven uno: `List.first`, `last`,
  `pop`, `get`, `Text.find`, `to_int`, `to_float`, `Map.get`, `remove`. En JSON, `None` es
  `null` y `Some(x)` es simplemente `x`.
* `Map[K, V]` tiene claves `Text`, enteras o `Bool`. Se crea con `std.map.new()` y una
  anotación de tipo, y tiene `insert`, `get`, `remove`, `contains_key`, `len`, `is_empty`, `keys` y
  `values`. Como `List`, las copias comparten las mismas entradas. Las claves se mantienen en orden; los objetos JSON usan
  claves de cadena.
* `List` también tiene `sort` (números, cadenas, bools) y `reverse`; `Text` tiene `chars` y `lines`.
* Las funciones son valores. El tipo de una se escribe como una cabecera sin el nombre:
  `(Int, Int) -> Int`, `() -> Text`, `(Int) -> ()` para una que no devuelve nada. Un nombre puede guardar una
  función (`let f = twice`), y un parámetro, un campo, un elemento de lista o un valor de retorno también:
  `fn apply(f: (Int) -> Int, x: Int) -> Int: f(x)`, `fn adder(n: Int) -> (Int) -> Int: x => x + n`.
  El parámetro de una closure puede indicar su tipo: `(x: Int) => x + 1`. Llamar a una función con un número o unos tipos
  de argumentos incorrectos es un error de compilación que muestra ambos tipos.
* `x => x + 1` y `(a, b) => a + b` son funciones de una línea. Para más, escribe una `fn` dentro de la
  función: `fn add(a: Int): a + k`. Ambas capturan las variables por valor, así que asignar a una variable
  de fuera es un error de compilación; los valores `List` y `Map` compartidos todavía pueden cambiarse desde
  dentro de una de ellas.
* Esto se ejecuta en el intérprete; el backend nativo todavía no lo compila.

## Tablas y tensores, a la manera corta

```telvax
type Sale:
    id: Int
    region: Text
    units: Int
    price: Float

fn main():
    let sales: Table[Sale] = read_csv("examples/dataframe/sales.csv")
    print(f"mean price: {sales.price.mean()}")

    let by_region: Table[{region: Text, revenue: Float}] = sales.sql("SELECT region, SUM(units * price) AS revenue FROM t GROUP BY region")
    for row in by_region:
        print(f"{row.region}: {row.revenue}")
```

* Cada función y método de la biblioteca estándar, con su tipo, está en [la referencia](https://github.com/telvax/telvax/blob/main/docs/reference/std.md).
* `print`, `read_csv`, `read_parquet` y `read_ipc` no necesitan el prefijo `std.`. `print` termina la línea;
  `std.io.print` escribe sin terminarla.
* Los métodos de los tensores son las funciones de `std.tensor`: `x.matmul(w).relu().softmax()`,
  `sales.price.mean()`. Una tabla tiene `sql`. Son las mismas llamadas por debajo, así que
  `std.tensor.relu(x)` sigue funcionando.
* Operadores sobre tensores: `a + b`, `a - b`, `a * b`, `a / b` (misma forma) y `a * k`, `k * a` para
  escalar por un número.
* `Table[{region: Text, revenue: Float}]` escribe el tipo de la fila en el sitio; sin un tipo desechable para
  el resultado de una consulta. Las mismas columnas significan el mismo tipo en todas partes.
* `fn main():` puede no devolver nada.

Mira `examples/dataframe/`. La versión en pandas del mismo programa tiene más o menos la misma longitud, pero la
de Telvax comprueba los nombres y los tipos de las columnas antes de ejecutarse.

## Errores, tuplas e iteradores

```telvax
fn split_pair(line: Text) -> Result[(Text, Text), Text]:
    let parts = line.split("=")
    if parts.len() != 2:
        return Err(f"bad line: {line}")
    Ok((parts[0].trim(), parts[1].trim()))

fn port_of(line: Text) -> Result[Int, Text]:
    let key, value = split_pair(line)?  # `?` returns the Err early
    match value.to_int():
        case Some(n):
            Ok(n)
        case None:
            Err(f"not a number: {value}")

fn main():
    let xs = [3, 1, 4, 1, 5]
    let big = xs.filter(x => x > 2).map(x => x * 10)  # [30, 40, 50]
    for i, x in xs.enumerate():
        print(f"{i}: {x}")
    let stack = [1, 2, 3]
    while stack.pop() is Some(top):
        print(f"{top}")
    if xs.first() is Some(first):
        print(f"first = {first}")
```

* `Result[T, E]` tiene `is_ok`, `is_err`, `unwrap` y `expect("message")` (ambos detienen el programa
  con el error), `unwrap_or(v)`, `unwrap_or_else(e => v)`, `map`, `map_err`, `and_then`, y `ok()` /
  `err()`, que dan un `Option`. `and_then` debe devolver un `Result` con el mismo tipo de error; usa
  `map_err` para cambiarlo antes. Un método que no existe es un error de compilación.
* `expr?` sobre un `Result` o un `Option` devuelve el error (o `None`) de la función actual; el
  tipo de retorno de la función también debe ser un `Result` o `T?`, y los tipos de error deben coincidir.
* `if x is Pattern:` y `while x is Pattern:` comparan un patrón y enlazan sus nombres; funcionan
  con `break` y `continue`, y `else:` se ejecuta cuando el patrón no coincide.
* Tuplas: el tipo `(A, B)`, valores `(1, "a")`, campos `.0` y `.1`, y desestructuración en
  `let a, b = pair`, bucles `for` y `((a, b)) => ...` (un argumento de tupla desarmado).
* Métodos de `List`: `map`, `filter`, `any`, `all`, `find`, `enumerate`, `zip`, `sum`, `min`, `max`,
  `fold`, `for_each`, `skip` y `join`, y `Map.entries`. `min`, `max` y `find` devuelven un
  `Option`. Esto se ejecuta en el intérprete; el backend nativo todavía no lo compila.

## Tipos, métodos, traits y genéricos

```telvax
trait Shape:
    fn area(self) -> Float: ...

    fn describe(self) -> Text:
        f"area {self.area()}"

type Circle(Shape):
    r: Float

    fn area(self) -> Float:
        self.r * self.r * 3.14159

type Stack[T]:
    items: List[T]

    fn push(self, x: T):
        self.items.push(x)
```

Un `type` lista sus campos y después sus métodos. `type Circle(Shape)` dice que el tipo implementa el
trait; un método sin cuerpo (`: ...`) es obligatorio, uno con cuerpo es un valor por defecto. `extend Circle:`
añade métodos a un tipo declarado en otro sitio. Llama a una función que no tiene `self` como `Circle.new(1.0)`.
`[T]` después de un nombre declara un parámetro de tipo; `T: Shape` lo limita a los tipos que implementan `Shape`.
Un método puede tener parámetros de tipo propios, además de los de su tipo:
`fn map[U](self, f: (T) -> U) -> Stack[U]`. Se infieren de la llamada (`s.map(x => x * 2)`), y
cada elección distinta crea su propia instancia. Un método de un trait no puede tenerlos.
Una función genérica se llama con sus tipos inferidos a partir de los argumentos. Cuando no se pueden inferir (una
función que no recibe nada), o para ser explícito, escríbelos antes de los paréntesis:
`empty[Text]()`, `pair[Int, Text](1, "a")`. Los tipos son nombres como `Int` o `List[Text]`.
Tres límites no necesitan un trait propio: `T: Num` (`Int`, `Float` y los números con tamaño), `T: Ord`
(números y `Text`, los valores que `<` y `>` comparan) y `T: Eq` (todo menos funciones). Más de
uno: `T: (Num, Ord)`. Un tipo que no cumple un límite es un error en la llamada que lo usa, y un
trait tuyo llamado `Num`, `Ord` o `Eq` tiene prioridad.

Mira los [casos de uso](use-cases.md) para programas completos.
