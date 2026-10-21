> Traducido de [telvax/telvax@664d389](https://github.com/telvax/telvax/commit/664d3890800279185aac3fa7624632fd396b53fd) (`docs/guide/use-cases.md`).

# Casos de uso

Para qué sirve Telvax, con un programa para cada caso que puedes ejecutar desde la raíz del repositorio.
Todos los programas de abajo están en [`examples/`](https://github.com/telvax/telvax/tree/main/examples) y se ejecutan con los comandos que se muestran.

| Quieres | Mira | Se ejecuta con |
|---|---|---|
| Resumir un archivo CSV o Parquet | [`dataframe/sales.tvx`](https://github.com/telvax/telvax/blob/main/examples/dataframe/sales.tvx) | `telvax run` |
| Comprobar las formas de un modelo antes de ejecutarlo | [`ml/forward.tvx`](https://github.com/telvax/telvax/blob/main/examples/ml/forward.tvx) | `telvax run` |
| Servir una función por HTTP | [`service/sentiment.tvx`](https://github.com/telvax/telvax/blob/main/examples/service/sentiment.tvx) | `telvax serve` |
| Montar un servicio de RAG | [`rag-service.tvx`](https://github.com/telvax/telvax/blob/main/examples/rag-service.tvx) | `telvax serve` |
| Procesar texto | [`text/word-count.tvx`](https://github.com/telvax/telvax/blob/main/examples/text/word-count.tvx) | `telvax run` |
| Manejar errores sin excepciones | [`errors/parse-config.tvx`](https://github.com/telvax/telvax/blob/main/examples/errors/parse-config.tvx) | `telvax run` |
| Escribir código numérico sencillo | [`basics/primes.tvx`](https://github.com/telvax/telvax/blob/main/examples/basics/primes.tvx) | `telvax run`, o nativo con `telvax build` |

## 1. Análisis de datos en tablas

Una `Table[Row]` es una tabla Arrow cuyas columnas son los campos del tipo `Row`. Los nombres y los tipos de las
columnas se comprueban antes de que el programa se ejecute, y una columna se lee como un tensor sin copiarla.

```telvax
type Sale:
    id: Int
    region: Text
    units: Int
    price: Float

fn main():
    let sales: Table[Sale] = read_csv("examples/dataframe/sales.csv")
    print(f"mean price: {sales.price.mean()}")

    let by_region: Table[{region: Text, revenue: Float}] = sales.sql("SELECT region, SUM(units * price) AS revenue FROM t GROUP BY region ORDER BY revenue DESC")
    for row in by_region:
        print(f"{row.region}: {row.revenue}")
```

```sh
telvax run examples/dataframe/sales.tvx main
```

Úsalo para informes, preparación de features y comprobaciones rápidas de un archivo de datos. Una columna mal escrita
(`sales.prise`) es un error de compilación (`no column prise on type Table[Sale]`) en lugar de un fallo
a mitad de la ejecución. SQL (y Parquet) necesitan la compilación completa; `telvax-lite` mantiene las tablas en memoria y lee CSV.

## 2. Modelos con formas comprobadas

Las formas de los tensores forman parte del tipo, así que un producto de matrices con formas que no coinciden no compila:

```telvax
fn layer1(x: Tensor[f32, 2, 3], w: Tensor[f32, 3, 4]) -> Tensor[f32, 2, 4]:
    x.matmul(w).relu()
```

```sh
telvax run examples/ml/forward.tvx main
telvax run examples/ml/forward.tvx main --device gpu      # needs a build with --features gpu
telvax run examples/ml/forward.tvx main --quantize int8   # int8 matrix products
```

Cambia los pesos de `layer1` a `Tensor[f32, 4, 4]` y `telvax check` informa de
``matmul inner dimensions differ: 3 and 4`` en la llamada. Usa dimensiones simbólicas (`Tensor[f32, N, 3]`) cuando un tamaño solo se conoce en tiempo de ejecución.
Aquí los pesos vienen de `std.tensor.from_vec`; un modelo real los cargaría de un archivo o de una tabla.

## 3. Servir una función por HTTP

`@endpoint` convierte una función en una ruta. El tipo del parámetro es el cuerpo JSON y se valida
por ti; `Err(Status.BadRequest(...))` se convierte en un 400.

```telvax ignore
@endpoint(POST, "/v1/sentiment")
fn sentiment(input: Input) -> Result[Output, Status]:
    if input.text.is_empty():
        return Err(Status.BadRequest("empty text"))
```

```sh
telvax serve examples/service/sentiment.tvx
curl -XPOST localhost:8080/v1/sentiment -H 'content-type: application/json' -d '{"text":"great and good"}'
# {"label":"positive","score":2}
curl -XPOST localhost:8080/v1/sentiment -H 'content-type: application/json' -d '{"text":""}'
# {"error":"empty text"}   (HTTP 400)
```

Para publicarlo, `telvax build --serve examples/service/sentiment.tvx -o service --dockerfile` produce un solo
binario con el programa incrustado, más un Dockerfile.

## 4. Generación aumentada por recuperación

[`examples/rag-service.tvx`](https://github.com/telvax/telvax/blob/main/examples/rag-service.tvx) genera el embedding de una pregunta, busca en un almacén de
vectores, construye un prompt con los resultados y consulta un modelo, todo en un solo `@endpoint`. Las etapas se encadenan con
`|>`:

```telvax ignore
let answer = prompt |> llm.complete() |> std.text.trim()
```

```sh
telvax serve examples/rag-service.tvx          # offline stand-ins for the model and the database
telvax serve examples/rag-service.tvx --live   # Ollama, OpenAI and PostgreSQL with pgvector
```

El modo sin conexión permite desarrollar y probar el servicio sin que ninguno de ellos esté en marcha. `--live`
habla con los reales.

## 5. Procesamiento de texto

`Text`, `List` y `Map` tienen los métodos que esperas, y `List` tiene métodos de iterador (`map`,
`filter`, `fold`, `sum`, `enumerate`, `zip`, ...):

```telvax
fn count_words(text: Text) -> Map[Text, Int]:
    let counts: Map[Text, Int] = std.map.new()
    for word in text.to_lowercase().split(" "):
        let seen = counts.get(word).unwrap_or(0)
        counts.insert(word, seen + 1)
    counts
```

```sh
telvax run examples/text/word-count.tvx main
```

## 6. Errores como valores

No hay excepciones. Una función que puede fallar devuelve `Result[T, E]` o `T?`, el `match`
debe cubrir todos los casos, y `?` pasa un error a quien llamó.

```telvax ignore
fn port_of(line: Text) -> Result[Int, Text]:
    let key, value = split_pair(line)?
```

```sh
telvax run examples/errors/parse-config.tvx main
```

## 7. Código numérico sencillo, interpretado o nativo

Los enteros, los floats, los bucles y los tipos propios también funcionan como en un lenguaje corriente. El intérprete ejecuta
cualquier cosa; el backend nativo (`--features llvm`) compila el subconjunto descrito en el
[README](https://github.com/telvax/telvax#language-notes).

```sh
telvax run examples/basics/primes.tvx main
telvax build examples/basics/primes.tvx -o primes     # native, needs --features llvm
```

## Más allá de un archivo

* **Varios archivos y paquetes:** `from util import helper` y `tvpm`, mira el [recorrido](tour.md#módulos-y-paquetes).
* **Muchas máquinas:** `std.dist.map` con `telvax worker`, mira el [recorrido](tour.md#distribución).
* **Desde Python:** el módulo `python/` intercambia tablas Arrow y arrays de NumPy sin copiar.
* **En el navegador:** el playground ejecuta el comprobador y el intérprete como WebAssembly.
* **Editor:** `telvax lsp` y la extensión de VS Code dan diagnósticos, hover y autocompletado.

## Cuándo Telvax todavía no es la herramienta adecuada

Está en alfa. El backend nativo cubre un subconjunto pequeño, la salida PTX y StableHLO no se ha ejecutado
en GPU de NVIDIA ni en TPU, y la sintaxis todavía puede cambiar. [`docs/design/status.md`](https://github.com/telvax/telvax/blob/main/docs/design/status.md)
lista lo que se verificó y cómo.
