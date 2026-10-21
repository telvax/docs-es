> Traducido de [telvax/telvax@2253a54](https://github.com/telvax/telvax/commit/2253a54139c057b14d91eeded2259ee66e3a2148) (`README.md`).

<div align="center">

# Telvax

**Un lenguaje compilado y de tipado estático para datos, pipelines de ML y servicios de IA.**

[![CI](https://github.com/telvax/telvax/actions/workflows/ci.yml/badge.svg)](https://github.com/telvax/telvax/actions/workflows/ci.yml)
[![Release](https://img.shields.io/github/v/release/telvax/telvax?include_prereleases&label=release)](https://github.com/telvax/telvax/releases)
[![License](https://img.shields.io/badge/license-MIT%20%2F%20Apache--2.0-blue)](#licencia)

[Documentación](https://telvax.github.io/telvax/) ·
[Recorrido](guide/tour.md) ·
[Biblioteca estándar](https://github.com/telvax/telvax/blob/main/docs/reference/std.md) ·
[Línea de comandos](https://github.com/telvax/telvax/blob/main/docs/reference/cli.md) ·
[Discusiones](https://github.com/telvax/telvax/discussions)

</div>

Telvax es un lenguaje de programación. Escribes programas en archivos de texto que terminan en `.tvx` y los ejecutas con
el comando `telvax`, que instalas una sola vez. Está hecho para programas que trabajan con datos (tablas, CSV,
Parquet), aprendizaje automático (tensores) y servicios de IA (un servidor web que responde a peticiones JSON).
Si nunca lo has usado, empieza por el [Inicio rápido](#inicio-rápido): no necesita ningún otro software.

Telvax se lee como Python (bloques por sangría, sin llaves ni punto y coma) y comprueba como un lenguaje
compilado. Las formas de los tensores forman parte del sistema de tipos, un operador de pipeline (`|>`) compone etapas, y una
función marcada con `@endpoint` se convierte en una ruta HTTP, con la validación y el manejo de JSON generados para ella.

```telvax
import std.ai.embeddings.OllamaEmbedder
import std.http.Status

let embedder = OllamaEmbedder.new("nomic-embed-text")

type QueryPayload:
    query: Text
    top_k: i32

fn embed(text: Text) -> Tensor[f32, 768]:
    text |> std.text.clean_whitespace() |> std.text.truncate(512) |> embedder.embed()

@endpoint(POST, "/v1/embed")
fn handle(payload: QueryPayload) -> Result[Tensor[f32, 768], Status]:
    if payload.query.is_empty():
        return Err(Status.BadRequest("empty query"))
    Ok(embed(payload.query))
```

Esto es un servicio: necesita un [Ollama](https://ollama.com) en marcha para responder. El servicio de RAG completo está en
[`examples/rag-service.tvx`](https://github.com/telvax/telvax/blob/main/examples/rag-service.tvx), y se ejecuta sin conexión con sustitutos de los modelos.
Un programa que no necesita ninguna preparación está en el [Inicio rápido](#inicio-rápido).

> **Alfa temprana.** Bueno para probar el lenguaje y dar feedback. Espera bugs y cambios incompatibles
> en la biblioteca y en las herramientas; la sintaxis de la edición 2026 está congelada ([estabilidad](https://github.com/telvax/telvax/blob/main/docs/guide/stability.md)).
> Lo que funciona, y lo que todavía no se ha ejecutado en hardware real, está listado en [`docs/design/status.md`](https://github.com/telvax/telvax/blob/main/docs/design/status.md).

## Destacados

* **Formas de tensores en los tipos.** `Tensor[f32, 2, 3]` lleva su forma, así que un `matmul` con formas que no coinciden es un error de compilación.
* **Tablas tipadas.** `Table[Row]` es una tabla Arrow cuyas columnas son los campos de `Row`; lee CSV, Parquet y Arrow, y ejecuta SQL.
* **Servicios sin pegamento.** `@endpoint` convierte una función en una ruta JSON validada; `telvax build --serve` genera un solo binario y un Dockerfile.
* **Los errores son valores.** `Result`, `Option`, `T?`, el operador `?` y `if x is Some(v)`.
* **Herramientas incluidas.** Formateador, comprobador de tipos con tramos exactos, REPL, ejecutor de tests con cobertura, profiler, depurador (DAP), servidor de lenguaje, extensión de VS Code, gestor de paquetes (`tvpm`) y un kernel de Jupyter.
* **Intérprete y backend nativo.** Ejecuta un programa de inmediato, o compílalo con MLIR y LLVM; hay generadores de StableHLO y PTX para aceleradores.

## Qué puedes construir

| Caso de uso | Ejemplo |
|---|---|
| Resumir un archivo CSV / Parquet con columnas tipadas y SQL | [`examples/dataframe/sales.tvx`](https://github.com/telvax/telvax/blob/main/examples/dataframe/sales.tvx) |
| Modelos cuyas formas de tensor se comprueban antes de ejecutarse | [`examples/ml/forward.tvx`](https://github.com/telvax/telvax/blob/main/examples/ml/forward.tvx) |
| Convertir una función en una ruta HTTP JSON validada | [`examples/service/sentiment.tvx`](https://github.com/telvax/telvax/blob/main/examples/service/sentiment.tvx) |
| Un servicio de RAG: embeddings, búsqueda vectorial, un LLM | [`examples/rag-service.tvx`](https://github.com/telvax/telvax/blob/main/examples/rag-service.tvx) |
| Procesamiento de texto con `Map`, `List` y métodos de iterador | [`examples/text/word-count.tvx`](https://github.com/telvax/telvax/blob/main/examples/text/word-count.tvx) |
| Errores como valores: `Result`, `Option`, `?`, `if x is Some(v)` | [`examples/errors/parse-config.tvx`](https://github.com/telvax/telvax/blob/main/examples/errors/parse-config.tvx) |

Cada uno viene con los comandos para ejecutarlo en [`guide/use-cases.md`](guide/use-cases.md).

## Instalación

```sh
curl -fsSL https://raw.githubusercontent.com/telvax/telvax/main/scripts/install.sh | sh
```

No se abre nada al terminar: Telvax es una herramienta de línea de comandos, sin ventana y sin app. El script
imprime una línea y coloca dos programas, `telvax` y `tvpm` (el gestor de paquetes), en `~/.telvax/bin`.
En macOS y Linux no modifica tu `PATH`, así que añade la carpeta tú mismo (pon la línea en `~/.zshrc` o `~/.bashrc`
para conservarla) y comprueba:

```sh
export PATH="$HOME/.telvax/bin:$PATH"
telvax --version        # prints: telvax, then the version you installed
```

Si el shell responde `command not found: telvax`, falta la línea del `PATH`. Si no hay un binario
para tu sistema, el script compila desde el código fuente (necesita [Rust](https://rustup.rs)) y `cargo` coloca los
programas en `~/.cargo/bin`. Para compilar a mano: `cargo install --locked --path cli`.

Se publican dos compilaciones para cada sistema: el `telvax` completo (unos 50 MB, con SQL y Parquet)
y `telvax-lite` (unos 10 MB). La compilación lite tiene todo lo demás, tablas en memoria y CSV
incluidos; `std.table.sql` y Parquet informan de un error claro en ella. `TELVAX_LITE=1` hace que el script de instalación elija la lite.

Cada release también publica una imagen de Docker para `linux/amd64` y `linux/arm64`, con `telvax` y `tvpm`:

```sh
docker run --rm -v "$PWD:/work" -w /work ghcr.io/telvax/telvax run app.tvx main
```

Las etiquetas son la versión del release (mira los [releases](https://github.com/telvax/telvax/releases)) y `latest`.

En Windows (PowerShell):

```powershell
irm https://raw.githubusercontent.com/telvax/telvax/main/scripts/install.ps1 | iex
```

En Windows el script añade la carpeta a tu `PATH` (abre una terminal nueva). Las compilaciones para Windows son nuevas y solo las ha compilado y probado el CI, no se han probado a mano. El backend
nativo (`--features llvm`) todavía no está disponible en Windows.

## Inicio rápido

Instala Telvax (mira [Instalación](#instalación)) y guarda esto como `hello.tvx`:

```telvax
fn main():
    let names = ["Ada", "Alan", "Grace"]
    for name in names:
        print(f"hello, {name}")
```

```sh
telvax run hello.tvx main
```

Imprime:

```text
hello, Ada
hello, Alan
hello, Grace
```

`main` es la función que se ejecuta. `telvax check hello.tvx` solo lee y comprueba los tipos del archivo e imprime
`` `hello.tvx` ok ``; un error se informa con la línea y la columna. Más comandos:

```sh
telvax check hello.tvx          # parse and type-check; diagnostics point at the exact span
telvax repl                     # type definitions and expressions and see the result
telvax fmt --check              # every .tvx file under this directory
telvax test tests/ --coverage   # run the `test_*` functions
telvax serve examples/rag-service.tvx   # HTTP server on :8080 (offline model stand-ins)
```

Todos los comandos y flags están en [`docs/reference/cli.md`](https://github.com/telvax/telvax/blob/main/docs/reference/cli.md). Para notebooks,
`pip install "telvax[kernel]"` añade un kernel de Jupyter y la magic `%%telvax` ([`python/README.md`](https://github.com/telvax/telvax/blob/main/python/README.md)). Para compilar desde un clon,
hace falta Rust nightly (`rust-toolchain.toml` lo selecciona): `cargo build --release -p telvax_cli`.

## Documentación

| Lee | Para |
|---|---|
| [Recorrido](guide/tour.md) | el lenguaje, desde el primer programa hasta los módulos y los tests |
| [Casos de uso](guide/use-cases.md) | dataframes, modelos, servicios y RAG, con los comandos para ejecutar cada uno |
| [Biblioteca estándar](https://github.com/telvax/telvax/blob/main/docs/reference/std.md) | cada función y método de `std` |
| [Línea de comandos](https://github.com/telvax/telvax/blob/main/docs/reference/cli.md) | cada comando de `telvax` y `tvpm` |
| [Estabilidad](https://github.com/telvax/telvax/blob/main/docs/guide/stability.md) | qué está congelado y qué puede cambiar |
| [Estado](https://github.com/telvax/telvax/blob/main/docs/design/status.md) | qué existe, qué se verificó y cómo, y qué queda abierto |
| [Sintaxis](https://github.com/telvax/telvax/blob/main/docs/design/sintaxe-pythonica.md) | el diseño de la sintaxis |

La documentación también es un sitio web: <https://telvax.github.io/telvax/>.

## Notas sobre el lenguaje

* **Valores.** Los tipos que defines y `Text` son valores; una `List` es una referencia (las copias comparten elementos).
  Los tensores son valores inmutables. Los literales numéricos sin tipo toman su tipo de cómo se usan.
* **Tensores.** `Tensor[f32, 2, 3]` lleva su forma: `std.tensor.matmul` con formas
  que no coinciden es un error de compilación. Las dimensiones simbólicas (`[N]`) aceptan cualquier tamaño.
* **Tablas.** `Table[Row]` es una tabla Arrow cuyas columnas son los campos del tipo `Row`;
  `t.score` es la columna como un tensor sin copia. `head`, `tail`, `sort_by`, `sort_by_desc`, `select`, `group_by`, `filter`, `concat` y `split`
  son métodos; `sql` ejecuta una consulta. `read_ipc` / `write_ipc` usan archivos Arrow IPC (Feather), mapeados en memoria al
  leer; `read_flight` / `write_flight` hablan con un servidor Arrow Flight (compila con `--features flight`). Una columna de tipo `T?` puede contener nulos, y las columnas `Date` / `Timestamp` son tipos temporales de Arrow cuyos
  valores son `Text` ISO-8601.
* **Módulos.** `import util.helper`, `from crate.geometry import Point, norm`,
  `from mathx.vec import dot` (una dependencia). Un nombre que empieza con `_` es privado y no puede
  importarse.
* **Subconjunto nativo.** Enteros, floats, bools, cadenas, tipos que defines, listas de números, tensores
  `f32`/`f64` de forma estática, bucles y f-strings (ints, bools, cadenas). Las funciones flecha (`x => ...`), `Table`, SQL y la mayor parte
  de `std` se ejecutan solo en el intérprete.

## Estructura del repositorio

| Área | Dónde | Notas |
|---|---|---|
| Front end del compilador | `compiler/{lexer,parser,ast,semantic}` | `logos` + `chumsky` con recuperación de errores; HIR tipada; inferencia de tipo de literales; formas de tensor comprobadas estáticamente |
| Intérprete y runtime | `runtime/` | Tensores y tablas sobre Arrow, CSV / Parquet / SQL (DataFusion), clientes reales de Ollama / OpenAI / pgvector, SQL en PostgreSQL |
| Backend nativo | `compiler/{ir,codegen}` | MLIR (`melior`) y LLVM (`inkwell`) 22; JIT y ejecutables; cuantización int8 de `matmul`. Opcional: `--features llvm` |
| Servicio HTTP | `server/`, `cli/` | `@endpoint` se convierte en una ruta Axum; `telvax build --serve` incrusta el programa en un solo binario |
| Módulos y paquetes | `compiler/modules`, `tools/package-manager` | `import` entre archivos, `tvpm` con registries, un resolvedor y un lockfile |
| Soporte de editores | `tools/lsp`, `editors/vscode` | diagnósticos, hover, autocompletado contextual, ir a la definición, pistas de tipos inferidos |
| Playground web | `playground/` | el comprobador y el intérprete compilados a WebAssembly, con una consola (un REPL) |
| Aceleradores | `compiler/accel/` | generadores de StableHLO y PTX (no ejecutados en hardware TPU / NVIDIA); la ejecución en GPU vive en `runtime` (feature `gpu`) |
| Python | `python/` | módulo PyO3; intercambio sin copia de Arrow, NumPy y arrays CUDA; ejecutor de Ray |
| Distribución | `tools/cluster` | `telvax worker` + `--workers` para `std.dist.map` |
| Benchmarks | `benchmarks/` | objetivos de la alfa: sobrecarga de RAG, uso de memoria, rendimiento de ingesta |

[`docs/design/status.md`](https://github.com/telvax/telvax/blob/main/docs/design/status.md) asigna cada elemento de la especificación de ingeniería original
a lo que existe, lo que se verificó y cómo, y lo que sigue abierto.

## Pruebas

Los programas Telvax tienen su propio ejecutor de tests: una función llamada `test_...` es un test, las funciones `fixture_...`
le dan los parámetros, `cases_...` lo ejecuta con varios valores, `std.test.mock` reemplaza una
función, y `--coverage` y `--fail-under N` informan y mantienen las líneas que ejecutaron los tests. El
[recorrido](guide/tour.md#tests) tiene los detalles. Para probar este repositorio:

```sh
cargo test --workspace                       # no LLVM needed
. scripts/llvm-env.sh && cargo test --workspace --features llvm
cd python && maturin develop && python -m pytest tests
scripts/build-playground.sh --node && node --test playground/test
cd editors/vscode && npm ci && npm test
```

## Contribuir

Lee primero [CONTRIBUTING.md](https://github.com/telvax/telvax/blob/main/CONTRIBUTING.md). Un cambio en la sintaxis, en el sistema de tipos o en la biblioteca
estándar empieza como un RFC en [`telvax/rfcs`](https://github.com/telvax/rfcs).

## Licencia

Con doble licencia, MIT (`LICENSE-MIT`) o Apache 2.0 (`LICENSE-APACHE`): puedes usar Telvax bajo
cualquiera de las dos, a tu elección.

## Comunidad

Las preguntas y las ideas van a las [Discusiones](https://github.com/telvax/telvax/discussions); los bugs y
las peticiones de funcionalidades a las [issues](https://github.com/telvax/telvax/issues). Lee antes el [código de conducta](https://github.com/telvax/telvax/blob/main/CODE_OF_CONDUCT.md) e informa de
los problemas de seguridad en privado, como se describe en [SECURITY.md](https://github.com/telvax/telvax/blob/main/SECURITY.md).

## Otros repositorios

| Repositorio | Qué contiene |
|---|---|
| [`telvax/rfcs`](https://github.com/telvax/rfcs) | propuestas de cambios en el lenguaje y en las herramientas |
| [`telvax/docs-pt-br`](https://github.com/telvax/docs-pt-br), [`docs-es`](https://github.com/telvax/docs-es), [`docs-zh-cn`](https://github.com/telvax/docs-zh-cn) | traducciones del recorrido, de los casos de uso y de este README |
| [`telvax/website`](https://github.com/telvax/website) | el sitio público (todavía no construido; el sitio de la documentación es `site/` aquí) |
| [`telvax/workflows`](https://github.com/telvax/workflows) | workflows compartidos de GitHub Actions (todavía ninguno) |
| [`telvax/telvax-grammar`](https://github.com/telvax/telvax-grammar) | una copia de la gramática TextMate, para herramientas que la toman de un repositorio (GitHub Linguist) |
| [`telvax/.github`](https://github.com/telvax/.github) | perfil de la organización y archivos de comunidad por defecto |
