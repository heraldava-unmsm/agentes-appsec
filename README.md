# agentes-appsec

Configuración de Claude Code para revisar la seguridad de tu código combinando
**herramientas deterministas** (Semgrep, Gitleaks) con **agentes de IA de solo
lectura**.

> **El agente propone, la herramienta verifica y el humano decide.**

Acompaña a la serie *Agentes de IA y pruebas de seguridad*. Los resultados de la
prueba sobre OWASP crAPI están en [`docs/resultados-crapi.md`](docs/resultados-crapi.md).

## Qué incluye

| Archivo | Nivel | Qué hace |
|---|---|---|
| `.claude/commands/security-review-api.md` | 2 | El comando `/security-review` de Anthropic, ajustado para APIs (OWASP API Security Top 10:2023, contexto de autorización, defensa ante prompt injection) |
| `.claude/skills/auditoria-sast/SKILL.md` | 3 | `/auditoria-sast`: ejecuta Semgrep y Gitleaks, delega en tres agentes y consolida un reporte sin descartar lo dudoso |
| `.claude/agents/triage-sast.md` | 3 | Clasifica los hallazgos de las herramientas: confirmado, falso positivo o revisión humana |
| `.claude/agents/revisor-autorizacion.md` | 3 | Revisa autenticación y control de acceso (BOLA, BFLA, BOPLA) |
| `.claude/agents/revisor-entradas-flujos.md` | 3 | Revisa inyecciones, SSRF, consumo de recursos y flujos de negocio |
| `.claude/settings.json` | — | Bloquea la lectura de `.env`, llaves y la carpeta `secrets/` |

El nivel 1 es el comando oficial [`/security-review`](https://github.com/anthropics/claude-code-security-review), sin cambios.

## Empieza en 15 minutos

1. Instala [Semgrep](https://semgrep.dev/docs/getting-started/) y [Gitleaks](https://github.com/gitleaks/gitleaks).
2. Copia la carpeta `.claude/` a la raíz de tu proyecto. Si ya tienes una, combina los archivos.
3. Agrega `.audit/` a tu `.gitignore`.
4. Crea una rama con tus cambios y, en Claude Code, ejecuta:
   ```
   /auditoria-sast main
   ```
5. Revisa `.audit/reporte-sast.md`. Empieza por "Requieren revisión humana".
6. Pregúntale al agente: **"¿Qué no pudiste verificar?"**

Para `/security-review-api`, completa antes el bloque **"CONTEXTO DE AUTORIZACIÓN DEL PROYECTO"** del comando. Con él, el agente sabe dónde se valida el acceso en tu proyecto.

## Seguridad

- **Los agentes son de solo lectura** (`Read`, `Grep`, `Glob`): no ejecutan comandos ni modifican código. La skill solo escribe en `.audit/`.
- **Las reglas `deny` filtran herramientas, no datos.** Si un `.env` entra en el diff, el agente lo verá igual. No versiones secretos y ejecuta Gitleaks antes.
- **Úsalo solo con código de confianza.** Los agentes tienen instrucciones de tratar el código como datos, pero ningún LLM es inmune al prompt injection.
- **Nada de esto reemplaza una revisión humana.** Los reportes citan `archivo:línea` para que puedas auditar la evidencia rápido.

## Licencia

MIT. El comando `security-review-api.md` deriva de `security-review.md` de Anthropic (MIT); ver [`LICENSE`](LICENSE).
