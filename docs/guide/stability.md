> Traducido de [telvax/telvax@0524f5d](https://github.com/telvax/telvax/commit/0524f5db55c7dfdef801ffd31e3feb8eb2f30719) (`docs/guide/stability.md`).

# Estabilidad de la sintaxis

Un programa que funciona hoy sigue funcionando. La sintaxis se agrupa en **ediciones**. La única edición hasta ahora
es `2026`, y su sintaxis está congelada: a partir de la versión que contiene esta página, un programa escrito en la
edición 2026 es aceptado por todo Telvax posterior que lea la edición 2026.

## Qué se promete

* **Los programas no se rompen.** Todo programa en `tests/editions/2026/` debe seguir pasando
  `telvax check`, `telvax fmt --check` y, cuando tiene funciones `test_...`, `telvax test`. El test
  que los ejecuta es `cli/tests/editions.rs`.
* **Los archivos están congelados.** `tests/editions/2026/CHECKSUMS` guarda un hash de cada programa. Cambiar un
  programa, o añadir uno sin su línea, hace fallar el test. Un pull request que tiene que editar un programa
  congelado es un cambio incompatible y necesita una nueva edición (más abajo).
* **Se puede añadir sintaxis nueva.** Una palabra, una forma o una función de la biblioteca estándar que no hace que
  ningún programa antiguo deje de funcionar es un cambio `feat:` normal. Los programas nuevos también entran en el
  directorio congelado, así que la promesa crece.
* **Los mensajes de error no forman parte de ella.** El texto de un diagnóstico, los avisos del lint y la salida de
  `telvax fmt` para código que el formateador rechazaba antes pueden cambiar. El formateador no cambia el
  diseño de un programa que ya está formateado.

## Qué no se promete

* La biblioteca `std` puede ganar funciones, y una función puede ganar una nueva sobrecarga. Quitar o cambiar una
  sigue la misma regla que la sintaxis: una nueva edición.
* El backend nativo, la ejecución en GPU y las salidas para aceleradores están listados en
  `docs/design/status.md`; lo que corre ahí puede crecer, y un programa que no compilaba de forma nativa puede llegar a hacerlo.
* La corrección de un bug puede rechazar un programa que solo funcionaba por culpa del bug (un error de tipos que el verificador no detectó).
  El cambio lo dice en `CHANGELOG.md`.

## Elegir una edición

`telvax.toml` da el nombre de la edición de un paquete:

```toml
[package]
name = "app"
version = "0.1.0"
edition = "2026"
```

* Sin la línea, el paquete se lee como la edición más reciente que conoce el compilador.
  `tvpm new` la escribe, así que un paquete nuevo queda fijado desde el principio.
* Una edición que el compilador no conoce es un error que lista las que sí conoce y dice que se actualice
  Telvax si el paquete es más nuevo.
* Un archivo suelto, sin `telvax.toml`, se lee como la edición más reciente.

## Una nueva edición

Un cambio que hace que un programa antiguo deje de compilar, o que cambia lo que significa, va en una nueva edición
y no en la existente:

1. La edición antigua sigue compilando como antes. El compilador lee las dos, elegidas por `edition`.
2. El nuevo directorio `tests/editions/<año>/` recibe sus propios programas congelados y su `CHECKSUMS`, y el
   año se añade a `EDITIONS` en `tools/package-manager/src/manifest.rs`.
3. `CHANGELOG.md` lista lo que cambió, y `telvax-migrate` (`tools/migrate`) recibe una regla que reescribe el
   código antiguo a la nueva edición donde eso se pueda hacer de forma mecánica.
4. Cuánto tiempo sigue soportada una edición antigua se decide cuando se propone la segunda edición; retirar
   una es un cambio `feat!:` aparte.

## Qué existe hoy

Hecho: la línea `edition` y su comprobación, los programas congelados y el test que los protege, y esta
página. Todavía no hecho: una segunda edición (así que la lectura de dos ediciones no está probada), avisos para formas
obsoletas, y un comando `telvax migrate` dentro del binario `telvax`. La herramienta `telvax-migrate`
convierte la antigua sintaxis al estilo Rust y sigue siendo un programa aparte, que se eliminará.
