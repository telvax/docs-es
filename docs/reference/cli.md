> Traducido de [telvax/telvax@4c5a64d](https://github.com/telvax/telvax/commit/4c5a64ddd71f1b78a06434638f6d7ef77a6be530) (`docs/reference/cli.md`).

# Línea de comandos

`telvax` es un único programa con varios comandos: `telvax <comando> [opciones]`. `telvax --version` imprime la
versión y `telvax --help` imprime la lista de abajo. `telvax doctor` dice qué puede hacer el build que tienes. El gestor de paquetes, `tvpm`, está [al final](#tvpm).

## Sinopsis

```text
usage:
  telvax check [--json] [file.tvx...]                       no file: the package in this directory
  telvax lint <file.tvx> [--allow|--warn|--deny rule]... [--deny-warnings] [--list-rules]
                                                            unused variables and functions, unreachable cases
  telvax fmt [--check] [--diff] [file.tvx|dir]...             format in place (`-` reads stdin; no path: this directory)
  telvax schema <file.csv|.parquet|.arrow> [--name Row]      the `type` for the columns of a data file
  telvax doc [--public] [-o out.md] [file.tvx|dir]...      Markdown reference from the types, signatures and `#` comments
  telvax test [file.tvx|dir] [name] [--allow list] [--coverage] [--fail-under N]
                                                            run the `test_*` functions (a directory: `test_*.tvx`, `*_test.tvx`)
  telvax run <file.tvx> [function] [args...] [--allow list] [--quantize int8|int8-channel|int4-channel] [--device gpu]
  telvax profile <file.tvx> [function] [args...] [--allow list] [--top N] [--json f] [--collapsed f] [--speedscope f]
                                                            time per function, line and std operation, and memory
  telvax doctor                                             what this build can do and how it is set up
  telvax repl [file.tvx...] [--allow list]                  type definitions, variables and expressions and see the result
  telvax dap                                                debug adapter over stdio (breakpoints, stepping, variables)
  telvax lsp                                                language server over stdio
  telvax worker [--listen host:port] [--token T]            run calls for a coordinator
  telvax serve <file.tvx> [--port N] [--host H] [--seed docs.jsonl] [--live] [--allow list] [--quantize int8|int8-channel|int4-channel] [--device gpu] [--metrics] [--log-format json|text] [--log-level debug|info|warn|error] [--cors origins] [--rate-limit N/min] [--security-headers] [--process-time-header] [--shutdown-wait N] [--error-format telvax|detail] [--watch]
  telvax build --serve <file.tvx> [-o out] [--dockerfile]   single-binary server (+ Dockerfile)
  telvax jit <file.tvx> <function> [args...]                 (native, needs --features llvm)
  telvax build <file.tvx> [-o out] [--emit exe|obj|llvm-ir|mlir|ptx|stablehlo] [-O 0-3] [--sm sm_80]
                                                    (all but stablehlo need --features llvm)

--allow takes a list of fs.read, fs.write, fs, env, net, all, none. Without it everything is allowed.
```

Este bloque es la salida de `telvax --help`, y un test comprueba que lo sigue siendo.

## Estado de salida

| Estado | Significado |
|---|---|
| 0 | El comando hizo lo que se le pidió. |
| 1 | Falló: un programa con errores, un test que falla, un archivo que necesita formato, un archivo que falta. El motivo se imprime después de `error:`. |
| 2 | Se llamó mal: un comando desconocido (con una sugerencia cuando hay uno parecido) u opciones que no encajan. |

## Opciones que comparten varios comandos

`--allow lista`
: Los permisos que tiene el programa: `fs.read`, `fs.write`, `fs` (los dos), `env`, `net`, `all` o `none`, separados por comas
  (`--allow fs.read,net`). Una función que necesita un permiso que no se dio se detiene con un error. Sin `--allow`
  todo está permitido. La aceptan `run`, `test`, `profile`, `repl` y `serve`. (`lint` tiene su propio
  `--allow regla`, véase más abajo.)

`--quantize int8 | int8-channel | int4-channel | none`
: Ejecuta los productos de matrices `f32` con aritmética de enteros. Véase la sección Cuantización del [recorrido](../guide/tour.md). La aceptan `run`, `serve` y `build`.

`--device cpu | gpu`
: Dónde se ejecutan las operaciones de tensores; `gpu` necesita un build con la característica `gpu` y una GPU (`telvax doctor` dice si el build la tiene). La aceptan `run` y `serve`.

`--workers host:puerto,host:puerto`
: Envía las llamadas de `std.dist.map` a máquinas que ejecutan `telvax worker`. La variable de entorno `TELVAX_WORKERS` es el
  valor por defecto. La aceptan `run` y `serve`.

## Comandos

### `telvax check`

Analiza la sintaxis y comprueba los tipos de los archivos indicados o, sin ninguno, del paquete del directorio actual (el que tiene un
`telvax.toml`). Imprime cada error con su lugar y falla si hay alguno.
`--json` imprime los diagnósticos como JSON, un objeto por cada uno, con `file`, `line`, `column`, `end_line`, `end_column`,
`severity`, `message` y `notes`, para editores y scripts.

### `telvax lint`

Comprueba un archivo y luego informa del código que no se usa: variables, funciones, casos que nunca pueden alcanzarse.
`--list-rules` imprime cada regla con lo que encuentra. El nivel de una regla (`allow`, `warn`, `deny`) viene de la
tabla `[lint]` de `telvax.toml` y después de las opciones `--allow regla`, `--warn regla` y `--deny regla`.
`--deny-warnings` hace que un aviso haga fallar el comando.

### `telvax fmt`

Reescribe archivos en el único formato que tiene Telvax e imprime `formatted <archivo>` por cada uno que cambió. Una carpeta significa
todos los archivos `.tvx` que hay dentro; ninguna ruta significa el directorio actual; `-` lee el código de la entrada estándar y
escribe el resultado en la salida estándar. `--check` no cambia nada y falla si un archivo cambiaría, que es lo que quiere un job de
CI. `--diff` muestra lo que cambiaría.

### `telvax schema`

Lee un archivo `.csv`, `.parquet` o `.arrow` e imprime el `type` de sus columnas, para pegarlo encima de
`let t: Table[Sales] = read_csv("sales.csv")`. Un CSV se lee entero: la cabecera da los nombres y los valores dan los
tipos (`Int`, `Float`, `Bool`, `Date`, `Timestamp`, si no `Text`), y una columna con un valor vacío se escribe `T?`. Para
Parquet y Arrow se usa el esquema del propio archivo, incluido qué columnas pueden estar vacías. El tipo recibe el nombre del archivo
(`sales_2024.csv` da `Sales2024`); `--name Row` elige otro nombre. Una columna cuyo nombre no es un nombre de campo válido
(`unit price`) se informa, no se renombra.

### `telvax doc`

Escribe una referencia en Markdown de las funciones, tipos, enums y traits de los archivos indicados (una carpeta significa todos los archivos `.tvx`
que hay dentro), con las líneas de comentario `#` escritas justo encima de cada uno. `--public` deja fuera los nombres que
empiezan por `_`. `-o out.md` escribe un archivo en lugar de usar la salida estándar.

### `telvax test`

Ejecuta las funciones llamadas `test_...` de un archivo; un test pasa cuando retorna y falla cuando se detiene con un error
(`assert`, `assert_eq`, `fail`, cualquier error de ejecución). Una carpeta ejecuta todos los archivos `test_*.tvx` y `*_test.tvx` que hay dentro. Una
segunda palabra, `name`, ejecuta solo los tests cuyo nombre la contiene. Imprime `N passed, M failed` y falla si algún test
falló. `--coverage` informa además de qué líneas del programa ejecutaron los tests; `--fail-under N` (de 0 a 100) lo convierte en una
barrera: el comando falla si se ejecutó menos del `N` por ciento de las líneas. Las funciones de `std.test`, como
`mock`, están en la [referencia de la biblioteca](https://github.com/telvax/telvax/blob/main/docs/reference/std.md).

### `telvax run`

Llama a una función de un programa: `telvax run app.tvx main`, o solo `telvax run app.tvx` para llamar a `main`. Las palabras después del
nombre de la función son sus argumentos, en orden. Un parámetro `Text` toma la palabra tal cual; cualquier otro tipo toma JSON (`3`,
`true`, `[1, 2]`, `{"a": 1}`). El valor que retorna la función se imprime, a menos que no retorne nada.

### `telvax profile`

Ejecuta una función como `run` e informa de adónde fueron el tiempo y la memoria: por función, por línea y por operación de `std`.
`--top N` limita el informe a las `N` sentencias que más tardaron. `--json archivo`, `--collapsed archivo` (para herramientas de flame graph) y
`--speedscope archivo` también escriben el informe en esos formatos.

### `telvax doctor`

Imprime la versión, las ediciones de sintaxis que lee, lo que puede hacer este build (`tables`, servicios live, GPU, backend
nativo) y las variables de entorno que importan (`TELVAX_LIVE`, `OLLAMA_HOST`, `OPENAI_API_KEY`,
`TELVAX_WORKERS`, `TELVAX_CLUSTER_TOKEN`). Una clave o un token se muestra como `set` o `not set`, nunca su valor. Pon la
salida en un informe de bug.

### `telvax repl`

Inicia una sesión interactiva: escribes definiciones de tipos, variables y expresiones, y ves el resultado. Los archivos indicados en la
línea de comandos se cargan primero. La misma sesión está en el playground. Dentro de la sesión, `:help` lista sus comandos.

### `telvax dap`

El adaptador de depuración (puntos de interrupción, ejecución paso a paso, variables), que habla el Debug Adapter Protocol por la entrada y la salida
estándar. Los editores lo inician; la extensión de VS Code lo hace. No lo ejecutas a mano.

### `telvax lsp`

El servidor de lenguaje, que habla el Language Server Protocol por la entrada y la salida estándar. Los editores lo inician; la
extensión de VS Code lo hace. No lo ejecutas a mano.

### `telvax worker`

Ejecuta llamadas para un coordinador, de modo que `--workers` pueda repartir `std.dist.map` entre máquinas. Escucha en
`--listen host:puerto` (por defecto `127.0.0.1:7077`) y, cuando `--token T` o la variable de entorno
`TELVAX_CLUSTER_TOKEN` está definida, responde solo a un coordinador que tenga el mismo token.

### `telvax serve`

Sirve las funciones `@endpoint` de un programa por HTTP, con la petición validada y la respuesta escrita como JSON,
hasta que se detiene (Ctrl-C o SIGTERM). Escucha en `--host` (por defecto `127.0.0.1`) y `--port` (por defecto `8080`). Un programa
sin funciones `@endpoint` es un error. `--live` usa los servicios reales de Ollama, OpenAI y PostgreSQL en lugar de los
sustitutos sin conexión (y necesita `net`). `--seed docs.jsonl` rellena el almacén de vectores antes de la primera petición: un objeto JSON
por línea, con los campos de texto `collection`, `id` y `text`; cada texto se convierte en embedding y se guarda.

`--watch` es para el desarrollo. Vigila los archivos del programa (el archivo de entrada, los módulos que importa, los archivos que
incrusta con `std.fs.embed`, y `telvax.toml` y `telvax.lock` de su proyecto) y, cuando uno cambia, comprueba el
programa e inicia el servidor otra vez en el mismo host y puerto. Si la comprobación falla, los errores se imprimen con sus
líneas y se mantiene el servidor que está funcionando, así que una errata no tumba el servicio. El nuevo servidor ejecuta `on_start`
otra vez, y lo que el antiguo tenía en memoria se pierde. Si el programa tiene errores cuando empieza `--watch`, espera a un cambio en lugar de salir.
Ctrl-C y SIGTERM detienen el servidor y `--watch` a la vez. Las demás opciones se pasan al servidor tal cual.

`--metrics` sirve `GET /metrics` en el formato de Prometheus: `http_requests_total{method,path,status}`, el
histograma `http_request_duration_seconds{method,path}`, los contadores que el programa crea con
`std.metrics.count(name)` y los medidores que fija con `std.metrics.set(name, value)` (gana el último valor), cada uno con su propio nombre
(`name`, o `name{label="value"}` con etiquetas) y su propio `# TYPE`. Un nombre es un contador o un medidor, no los dos. `path` es la ruta tal como está escrita en el programa, así que el número
de series no crece con los ids de las URL. `--log-format json` (o `text`) escribe una línea por petición en la
salida de error estándar: hora, nivel, id de la petición, método, ruta sin la query string, estado y duración. El nivel es
`info` para un estado por debajo de 400, `warn` para 4xx y `error` para 5xx; `--log-level` (o `TELVAX_LOG_LEVEL`) es el más bajo
que se escribe, `info` por defecto, así que `--log-level warn` conserva solo las peticiones que fallaron. Las líneas de
`std.log` siguen el mismo formato y nivel. Con cualquiera de las dos opciones, cada
respuesta lleva una cabecera `x-request-id`, la que envió el cliente si es corta y sencilla, y si no una nueva; la
función puede leerla con `req.header("x-request-id")`. Un programa iniciado con `telvax build --serve` acepta las mismas
opciones, o `TELVAX_METRICS=1` y `TELVAX_LOG_FORMAT=json`; la opción gana a la variable.

Una función que falla en tiempo de ejecución (un índice fuera de rango, una división por cero) responde `500` con `Internal server error`, en los dos
formatos de error, porque el mensaje del intérprete nombra el interior del programa. El mensaje va a la salida de error estándar
con el id de la petición, como una línea `error GET /path 500 request_id=...: runtime error: ...` (un objeto JSON con `--log-format json`).
Una función que retorna `Err(Status.Internal(message))` sigue enviando su `message`, porque el programa lo eligió.

Otras tres opciones actúan sobre todas las respuestas. `--cors ORÍGENES` (`*`, o orígenes separados por comas) permite que una página de navegador de
un origen permitido lea la respuesta: añade `access-control-allow-origin` y responde un `OPTIONS` de preflight con `204`;
las credenciales no están soportadas. `--rate-limit N/unidad` (`second`, `minute` o `hour`) permite `N` peticiones de un mismo cliente en
cada ventana y, después, responde `429` con `retry-after`; el cliente es la dirección de la conexión, o la primera
dirección de `x-forwarded-for` con `--trust-proxy`. `/healthz` nunca se limita. `--shutdown-wait N` (o `TELVAX_SHUTDOWN_WAIT`) es el número de segundos que un servidor que se detuvo
espera por sus peticiones y tareas en segundo plano (30 por defecto). `--security-headers` añade
`x-content-type-options: nosniff`, `x-frame-options: DENY`, `referrer-policy: no-referrer` y `cache-control: no-store`
a las respuestas que no los fijan. Las variables son `TELVAX_CORS`, `TELVAX_RATE_LIMIT`, `TELVAX_SECURITY_HEADERS`
y `TELVAX_TRUST_PROXY`. Las cuentas del límite de peticiones se guardan en memoria, por proceso.

Estas opciones también se pueden escribir en el programa, en un bloque `serve:` del archivo de entrada:

```
serve:
    metrics: True
    log_format: "json"
    log_level: "info"
    cors: "*"
    rate_limit: "100/min"
    security_headers: True
    process_time_header: True
    trust_proxy: False
    shutdown_wait: 30
    error_format: "detail"
    on_start: [init_db]
```

Cada línea es una de las opciones de arriba (`--log-format` es `log_format`), con un valor fijo: `True` o `False`, un número entero
de segundos o un texto. Una línea puede, en cambio, leer una variable cuando arranca el servidor, `env("NAME")` o
`env("NAME", "default")`, como en `cors: env("CORS_ORIGINS", "*")`: el valor es el texto de la variable, o el valor por defecto cuando
la variable no está definida o está vacía (sin valor por defecto, la línea se deja fuera), leído como lo lee la opción. Una opción `True or False`
está activada con `1` o `true`; el valor por defecto de una es `1`, `true`, `0` o `false`. `telvax check` rechaza un valor por defecto que la opción
rechazaría; una variable que la opción rechaza (`LOG_LEVEL=loud`) detiene el arranque con
`serve: LOG_LEVEL="loud" is not a value of `log_level`: ...`. No se convierte nada: `RATE_LIMIT_PER_MINUTE=100` no es
`100/min`. `telvax check` informa de un nombre que no está en la lista, un valor del tipo equivocado o que la
opción rechazaría, un nombre escrito dos veces, un segundo bloque `serve:` y un bloque en un archivo que no es el archivo de entrada. Una
variable (`TELVAX_METRICS`, ...) gana al bloque, y una opción gana a la variable. `--port`, `--host`, `--live`,
`--allow`, `--seed`, `--quantize` y `--device` no están en el bloque: dependen de dónde se ejecute el programa. Un binario
de `telvax build --serve` conserva el bloque.

`middleware: [nombres]` lista funciones marcadas con `@middleware`, de la forma `fn name(req: Request, next: Next) -> Response`
(`Request`, `Response` y `Next` de `std.http`), la más externa primero. Se ejecutan por dentro de las opciones integradas y envuelven cada
petición, las rutas y también `/health`, `/metrics`, `/openapi.json` y `/docs`. Un middleware que no llama a `next`
responde por sí mismo; uno que lo llama recibe el `Response` del resto de la cadena, cuyo estado y cabeceras puede cambiar
(el cuerpo se pasa tal como estaba), y `next` se puede llamar una vez. Un middleware que falla responde con `500`. `telvax check`
rechaza un nombre que no es una función marcada con `@middleware`, una firma equivocada y un nombre escrito dos veces, y avisa de un
`@middleware` del archivo de entrada que la lista no menciona. Se ejecutan en el intérprete, lo que cuesta más que una opción
integrada.

`on_start: [nombres]` es la única línea que no es un literal: una lista de funciones de nivel superior sin parámetros, cada una sin devolver
nada o con `Result[(), Text]`, que se ejecutan una vez, en el orden escrito, antes de abrir el puerto (así que `/health` no
responde hasta que terminan). Se ejecutan con los permisos del programa (`--live`, `--allow`). Una función que retorna
un `Err`, o que falla, detiene el arranque: el mensaje va a la salida de error estándar como `on_start: NAME failed: MESSAGE` y el
estado de salida es 1. `telvax check` informa de un nombre que no es una función, una función con parámetros u otro tipo de retorno,
y un nombre escrito dos veces. Una función puede venir de otro archivo (`from db import init_db`). No se ejecutan en `telvax run`
ni en `telvax test`.

### `telvax build --serve`

Crea un único ejecutable que contiene el programa y el servidor: `telvax build --serve app.tvx -o app`. Copia el
`telvax` que ejecuta el comando y le añade el programa, así que es tan grande como `telvax`. Cuando se inicia,
sirve el programa: escucha en `0.0.0.0`, en `PORT` si está definida, y `--port` y `--host` siguen ganando. `HOST` fija
la dirección, y `TELVAX_LIVE=1` activa `--live`. `app --healthcheck [/ruta]` le pide al servidor de esta máquina
`/ruta` (`/healthz` si se omite; el puerto es `--port`, `PORT` u 8080) y sale con 0 ante una respuesta 2xx, y con 1 en los demás casos: un
contenedor no tiene `curl` para eso. `--dockerfile` escribe además un `Dockerfile` para él junto al programa: copia el
directorio entero (los otros módulos y los archivos leídos por `std.fs.embed` tienen que estar ahí; un `.dockerignore` deja fuera
el resto), construye a partir de `ghcr.io/telvax/telvax` (o de la imagen en `--build-arg TELVAX_IMAGE=...`) y tiene un `HEALTHCHECK`
que llama a `--healthcheck`.

### `telvax build`

Compila un programa con el backend nativo (LLVM y MLIR), así que necesita un `telvax` compilado con `--features llvm`; los
binarios publicados no lo están, y `telvax doctor` lo dice. `--emit` elige la salida: `exe` (por defecto, un programa con el nombre del
archivo), `obj` (un archivo objeto), `llvm-ir` y `mlir` (impresos), `ptx` (impreso; `--sm sm_80` elige la generación
de la GPU) y `stablehlo` (impreso). `-o` da el nombre del archivo de salida. `-O 0` a `-O 3` es el nivel de optimización
(por defecto 2). `--emit ptx` y `--emit stablehlo` no necesitan el backend nativo.

### `telvax jit`

Compila una función con el backend nativo y la llama de inmediato: `telvax jit app.tvx norm 3.0 4.0`. Necesita `--features llvm`.

## `tvpm`

`tvpm` es el gestor de paquetes. Lee `telvax.toml`, escribe `telvax.lock` y ejecuta el comando `telvax` correspondiente con las
dependencias instaladas. `tvpm --help` imprime la lista de comandos. `tvpm` encuentra el proyecto en el directorio actual o en el
directorio padre más cercano que tenga un `telvax.toml`. Estado de salida: 0 cuando el comando hizo lo que se le pidió, 1 cuando falló (el motivo
se imprime después de `error:`), 2 para un comando desconocido.

### Registro

Un registro es un directorio o una URL `http(s)://` con la misma estructura: `index/<nombre>.json` tiene una línea JSON por cada versión
publicada (`name`, `vers`, `deps`, `cksum`, `yanked`) y `packages/<nombre>-<versión>.nvpkg` es un tar comprimido con gzip que contiene
`telvax.toml`, `src/**/*.tvx` y los archivos `README.md` y `LICENSE*`, si los hay. Sirve cualquier alojamiento de archivos estáticos.

`--registry <dónde>`, antes o después del comando, o la variable de entorno `TVPM_REGISTRY` dice cuál usar. Sin
uno, una dependencia del registro es un error que lo dice. Los paquetes descargados se descomprimen en `$TVPM_HOME`, o
`~/.tvpm`, y su checksum se comprueba con el índice.

`tvpm search`, `publish` y `yank` escriben en un registro de **directorio** o lo leen; no funcionan con una URL.

### Comandos

`tvpm new <nombre> [--lib]`
: Crea la carpeta `<nombre>` con `telvax.toml`, `src/main.tvx` (`src/lib.tvx` con `--lib`) y un `.gitignore`. El nombre son
  letras minúsculas, dígitos y `_` sueltos, empieza por una letra y no es `std`, `crate`, `self` ni `super`.

`tvpm init`
: Hace lo mismo en el directorio actual, con su nombre. Falla si `telvax.toml` ya está ahí.

`tvpm add <nombre>[@req] [--path DIR]`
: Añade una dependencia a `telvax.toml`. `name@1.2` escribe ese requisito; sin uno, `tvpm` busca en el registro la versión más nueva que no está
  retirada (yanked) y escribe `^<versión>`. `--path DIR` depende de un directorio (relativo al proyecto) cuyo paquete
  tiene ese nombre. No bloquea ni descarga: ejecuta `tvpm install`.

`tvpm remove <nombre>`
: Quita una dependencia de `telvax.toml`.

`tvpm lock`
: Resuelve las versiones y escribe `telvax.lock`. Imprime cada paquete bloqueado.

`tvpm install`
: Bloquea si falta `telvax.lock` o no coincide con `telvax.toml`, y luego descarga los paquetes del registro que no están instalados.

`tvpm update [nombre...]`
: Mueve las versiones bloqueadas a las más nuevas que encajan. Con nombres, solo se mueven esos paquetes del registro, y los otros se quedan donde
  el lock los tiene, a menos que un requisito los obligue. Imprime `updated name 1.0.0 -> 1.1.0` por cada cambio.

`tvpm tree`
: Imprime el árbol de dependencias.

`tvpm search <texto>`
: Lista los paquetes del registro de directorio cuyo nombre contiene `texto`, cada uno con su versión más nueva que no está retirada.

`tvpm publish`
: Comprueba el paquete como `telvax check`, lo empaqueta y lo añade al registro de directorio. Falla si el paquete no pasa
  la comprobación, si esa versión ya está publicada (sube `version` antes) o si tiene una dependencia de ruta: un paquete publicado
  solo puede depender de paquetes del registro.

`tvpm yank <nombre> <versión> [--undo]`
: Marca una versión publicada como retirada (yanked). Una nueva resolución la omite, y un `telvax.lock` que ya la nombra todavía la instala.
  `--undo` la marca otra vez como disponible.

`tvpm check`, `run [fn]`, `serve [opciones]`, `build [opciones]`
: Bloquean e instalan si hace falta y luego ejecutan `telvax check`, `run`, `serve` o `build` sobre el archivo de entrada con el resto de las palabras
  como argumentos. `run` sin función llama a `main`. `tvpm` inicia el `telvax` que está a su lado, o el que está en `TELVAX_BIN`, o
  el que está en `PATH`.

### `telvax.toml`

```toml
[package]
name = "app"                  # required
version = "0.1.0"             # required, semantic version
edition = "2026"              # the syntax the package is written in; "2026" when absent
description = "..."           # optional
entry = "src/app.tvx"         # optional; default src/main.tvx, then src/lib.tvx

[dependencies]
mathx = "^1.2"                         # from the registry
shared = { path = "../shared" }        # from a directory
pinned = { version = "=1.4.2" }        # the same as a string; not together with `path`

[lint]                                 # what `telvax lint` does for each rule: allow, warn or deny
unused-variable = "deny"
unused-function = "allow"
```

Un requisito de versión es el de Cargo: `1.2` y `^1.2` significan `>=1.2.0, <2.0.0`, `~1.2` significa `>=1.2.0, <1.3.0`, `=1.2.3`
significa solo esa versión, y `>=1, <3` combina dos. Cada nombre de paquete se resuelve a una sola versión; cuando dos paquetes necesitan
rangos distintos de un tercero, `tvpm` elige la versión más nueva que sirve para los dos y falla si no hay ninguna. Una `edition` que este `telvax` no lee es un error que dice cuáles lee.

### `telvax.lock`

`tvpm lock` lo escribe; no lo edites. Tiene un `[[package]]` para el proyecto y para cada paquete del que depende, con
`name`, `version`, `source` (`registry` o `path+<dir>`; ninguno para el propio proyecto), `checksum` para un paquete del registro y
`dependencies`, una lista de nombres. Haz commit de él: así todas las máquinas usan las mismas versiones. Queda desactualizado cuando `telvax.toml` tiene
una dependencia o un requisito que no cubre, y `install`, `check`, `run`, `serve` y `build` bloquean de nuevo en ese caso.
