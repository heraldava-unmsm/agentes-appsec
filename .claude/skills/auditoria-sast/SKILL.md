---
name: auditoria-sast
description: Análisis estático asistido por agentes sobre el código modificado respecto a una rama base
disable-model-invocation: true
argument-hint: [rama-base]
allowed-tools: Bash(git diff *) Bash(git log *) Bash(semgrep *) Bash(gitleaks *) Bash(mkdir *) Edit(.audit/**)
---

Ejecuta una auditoría de seguridad sobre los cambios de esta rama.
Rama base: $ARGUMENTS (si está vacío, usa main).

## Paso 1 — Alcance
Ejecuta `git diff --name-only <rama-base>...HEAD`.
Si no hay archivos modificados, informa y termina.
Excluye del alcance dependencias de terceros (vendor/, node_modules/) y archivos
generados.

## Paso 2 — Herramientas deterministas
Crea la carpeta `.audit/` y ejecuta, solo sobre el alcance:
- `semgrep scan --config p/default --json --output .audit/semgrep.json <archivos>`
- `gitleaks git --redact --log-opts="<rama-base>..HEAD" --report-format json --report-path .audit/gitleaks.json`

Pasa cada archivo como un argumento separado. No uses una variable con toda la
lista: en zsh no se divide en palabras y Semgrep falla con "Invalid scanning root".

`gitleaks git` con `--log-opts="<rama-base>..HEAD"` revisa solo los commits nuevos
(modo pull request). Para auditar todo el historial, usa `gitleaks git` sin
`--log-opts` e indícalo en el reporte.

Si alguna herramienta no está instalada o falla, indícalo en el reporte y continúa.

## Paso 3 — Triage
Delega al agente triage-sast con las rutas de los dos reportes.

## Paso 4 — Revisión de lógica
Identifica los endpoints y puntos de entrada de los archivos del alcance.
Delega en paralelo a los agentes revisor-autorizacion y revisor-entradas-flujos,
pasando a cada uno la lista de archivos y endpoints.

## Paso 5 — Reporte consolidado
- Elimina duplicados y ordena por severidad.
- Encabeza con: alcance, herramientas ejecutadas, totales por severidad y
  hallazgos que requieren revisión humana.
- Guarda el reporte en `.audit/reporte-sast.md` y muéstralo.
- No modifiques ningún archivo de código.

## Paso 6 — Insumos para pruebas dinámicas
Lista por separado los hallazgos de lógica con confianza Media o Baja:
son candidatos a confirmarse con pruebas dinámicas (DAST).
