> Traducido de [telvax/telvax@2134c83](https://github.com/telvax/telvax/commit/2134c838344b99b31e44f7ba0bb2bde2ea0017e5) (`SECURITY.md`).

# Política de seguridad

## Versiones soportadas

Telvax es una alfa temprana. Solo la última versión recibe correcciones de seguridad; las versiones más antiguas no se
corrigen, así que actualiza antes de informar.

## Informar de una vulnerabilidad

Informa de las vulnerabilidades **en privado** a través de GitHub: abre la pestaña Security del repositorio y elige
"Report a vulnerability". Por favor, no abras una issue, un pull request ni una discusión públicos para un
problema de seguridad.

Incluye lo que puedas de lo siguiente: la versión afectada (`telvax --version`), tu sistema operativo, un programa
`.tvx` mínimo o una petición que reproduzca el problema, y lo que esperabas que ocurriera.

## Qué esperar

* Nuestro objetivo es acusar recibo de un informe en **7 días** y decirte si lo consideramos una
  vulnerabilidad en **14 días**. Este es un proyecto pequeño, así que esos plazos son objetivos, no garantías.
* Una vulnerabilidad confirmada se corrige en un nuevo lanzamiento, y el aviso se publica una vez que el
  lanzamiento está disponible. Damos crédito a quienes informan y quieren que se les mencione.
* Te pedimos que nos des un tiempo razonable para corregir el problema antes de divulgarlo públicamente.

## Alcance

Dentro del alcance: los binarios `telvax` y `tvpm`, el servidor de lenguaje, el servidor HTTP que producen `telvax serve`
y `telvax build --serve` (manejo de peticiones, validación), el gestor de paquetes y su
cliente de registro, los scripts de instalación y la extensión de VS Code.

Fuera del alcance: el contenido de los programas que decides ejecutar (el intérprete ejecuta el código que recibe),
la denegación de servicio mediante entradas deliberadamente enormes a un servidor de desarrollo, y los problemas en servicios
de terceros a los que Telvax puede llamar (Ollama, OpenAI, PostgreSQL).
