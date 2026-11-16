> Traducido de [telvax/telvax@1e3cb31](https://github.com/telvax/telvax/commit/1e3cb318a8ca48be7bb78dea5ac1e8a1db835cc0) (`CONTRIBUTING.md`).

# Cómo contribuir a Telvax

Gracias por probar Telvax. En la versión 0.0.x, la contribución más útil es un informe de bug o un programa que
no se comportó como esperabas.

## Informar de problemas

Abre una issue e incluye:

* la salida de `telvax --version` y tu sistema operativo;
* el programa `.tvx` más pequeño que muestre el problema;
* lo que ejecutaste (`telvax run ...`, `telvax serve ...`) y la salida completa.

Las ideas para el lenguaje también son bienvenidas, idealmente con un breve ejemplo del código que te gustaría poder escribir.

Un cambio en la sintaxis, en el sistema de tipos o en la biblioteca estándar empieza como un RFC en
[telvax/rfcs](https://github.com/telvax/rfcs): consulta su README para conocer el proceso. El pull request aquí
enlaza el RFC aceptado. Las traducciones de la documentación están en `telvax/docs-pt-br`, `docs-es` y
`docs-zh-cn`; los archivos en inglés de `docs/` siguen siendo la fuente.

## Compilar

Se necesita Rust nightly; `rust-toolchain.toml` lo selecciona.

```sh
cargo build --release -p telvax_cli      # target/release/telvax
cargo test --workspace                  # default build, no LLVM needed
```

Piezas opcionales:

| Característica | Qué añade | Necesita |
|---|---|---|
| `--features telvax_cli/llvm` | código nativo, `--emit ptx` | LLVM 22 con MLIR (`. scripts/llvm-env.sh`) |
| `--features telvax_cli/gpu` | `--device gpu` | una GPU con Metal, Vulkan o DX12 |

Los bindings de Python (`python/`), la extensión de VS Code (`editors/vscode`), el playground en Wasm
(`playground/`) y el sitio de documentación (`site/`, publicado por versión, consulta su README) tienen su propio README y sus tests.

## Cómo llegan los cambios a `main`

`main` está protegida: nada se sube directamente a ella, ni siquiera el mantenedor. El trabajo se hace en una
rama y llega a `main` mediante un pull request, y el pull request tiene que pasar todas las comprobaciones
(formato, clippy, tests en Linux y Windows, los builds de LLVM y GPU, el build lite, los
bindings de Python, el playground en un navegador real, la extensión en un VS Code real, la imagen
Docker, `cargo deny` y el análisis de secretos) antes de poder fusionarse. Las fusiones son squash, así que el
título del pull request pasa a ser el mensaje del commit: mantenlo corto y en imperativo (`feat: add ...`).

```sh
git switch -c fix/short-name
# ...edit, commit...
git push -u origin fix/short-name
gh pr create --fill
gh pr merge --auto --squash      # merges by itself once every check is green
```

Puedes seguir ejecutando las comprobaciones en local mientras trabajas (véase abajo), pero lo que cuenta es un pull request
en verde.

## Antes de enviar un pull request

```sh
cargo fmt --all
cargo clippy --workspace --all-targets
cargo test --workspace
cargo deny check
```

* El código de los documentos se prueba (`cli/tests/docs.rs`): un bloque ```` ```telvax ```` tiene que pasar `telvax check` y
  `telvax fmt --check`, y ejecutarse cuando tiene `fn main`. Marca un trozo de programa con ```` ```telvax ignore ````.
  Los archivos de `examples/` pasan por las mismas comprobaciones.
* Añade un test con el cambio. Los tests del compilador viven junto al crate (`compiler/*/tests`), los tests del
  runtime en `runtime/tests`.
* Mantén los commits cortos y en imperativo, con los prefijos existentes: `feat:`, `fix:`, `docs:`,
  `test:`, `refactor:`, `style:`, `ci:`, `build:`, `chore:`.
* Si un cambio altera el lenguaje, actualiza `docs/guide/tour.md` y `CHANGELOG.md`.
* Los programas de `tests/editions/` están congelados (véase `docs/guide/stability.md`). Un cambio no debe hacer
  que falle ninguno, y no debe editar ninguno. Si no puede evitarlo, es un cambio incompatible: una nueva edición y
  un título `feat!:`. La sintaxis nueva recibe un nuevo programa congelado y una línea en el `CHECKSUMS` de ese directorio.
* `docs/design/status.md` dice lo que se ha ejecutado y lo que no en hardware real. Manténlo honesto: no
  marques algo como funcionando solo porque compila.

## Dónde está cada cosa

Consulta la tabla del README. Un mapa rápido del lenguaje en sí: `compiler/lexer` (tokens),
`compiler/parser` (sintaxis), `compiler/semantic` (tipos y comprobaciones), `runtime` (intérprete),
`compiler/ir` y `compiler/codegen` (nativo), `compiler/accel` (PTX y StableHLO).

Una nueva función de la biblioteca estándar va en `compiler/semantic/src/stdlib.rs` (su tipo) y
`runtime/src/std_impl.rs` (lo que hace). Descríbela también en `compiler/semantic/src/stdlib_docs.rs`: un test
falla sin la descripción, y `docs/reference/std.md` se escribe a partir de ella con
`UPDATE_REFERENCE=1 cargo test -p telvax_hir --test reference`.

## Lanzamientos

`main` se lanza automáticamente **una vez por semana** (los lunes, 07:00 UTC, 04:00 en Brasilia), y el **título** de cada pull request fusionado desde el último lanzamiento (un
[Conventional Commit](https://www.conventionalcommits.org)) elige la versión, así que no hay que editar
ningún archivo. Un lanzamiento tarda una hora, y un lanzamiento lleva todas las fusiones de la semana: cuenta el título de **cada**
commit desde la última etiqueta, y gana el mayor incremento (un `feat:` y luego un `fix:` es un lanzamiento
minor). Para lanzar antes, ejecuta a mano el workflow Release:

| Título | Versión | Ejemplo |
|---|---|---|
| `feat!: ...` o una línea `BREAKING CHANGE:` en el cuerpo | major (`1.2.3` a `2.0.0`; mientras el major es 0, minor) | `feat!: rename the pipe operator` |
| `feat: ...` | minor (`0.1.2` a `0.2.0`) | `feat: add traits` |
| `fix:`, `perf:`, `refactor:`, `build:`, `revert:` o sin prefijo | patch (`0.1.2` a `0.1.3`) | `fix: empty table in sql` |
| `docs:`, `ci:`, `test:`, `style:`, `chore:` | sin lanzamiento | `docs: add a use case` |

Añade `[skip release]` a un título y esa fusión no cuenta para nada; no cancela las otras. Para elegir tú una versión, sube
`version` en el `Cargo.toml` de la raíz por encima de la última etiqueta. Las notas del lanzamiento son la sección del `CHANGELOG.md`
de esa versión, si escribiste una, seguida de la lista de pull requests fusionados.

## Seguridad

Consulta [SECURITY.md](security.md).
