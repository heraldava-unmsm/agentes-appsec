# Resultados: tres niveles de revisión SAST con agentes sobre crAPI

Evaluación del 24/10/2026. Servicio analizado: `community` (Go) de OWASP crAPI.

## Qué se comparó

| Nivel | Herramienta | Qué hace |
|---|---|---|
| 1 | `/security-review` (oficial de Anthropic) | Revisión del diff con un LLM y un filtro de falsos positivos (confianza ≥ 8) |
| 2 | `/security-review-api` (este repositorio) | Nivel 1 con categorías OWASP API Security Top 10:2023, contexto de autorización y exclusiones ajustadas |
| 3 | `/auditoria-sast` (este repositorio) | Semgrep + Gitleaks + tres agentes de solo lectura (triage, autorización, entradas y flujos), sin descartar hallazgos dudosos |

## Entorno

- Código: crAPI, commit `b5fc307`. Solo `services/community` (28 archivos, sin `vendor/`).
- Diseño: configuración en la rama `main`; el código en una rama separada, para que el diff contenga solo el servicio analizado.
- Modelo: `claude-sonnet-5-5` en todas las corridas (los agentes usan `model: inherit`).
- Herramientas: Semgrep 1.178.0 (`p/default`), Gitleaks 8.26.0 (paquete de Kali).
- Tras la medición se corrigió en `SKILL.md` la forma de pasar la lista de archivos a Semgrep en zsh; en las corridas, el agente lo resolvió solo en el segundo intento.
- Dos corridas por nivel, cada una en una sesión nueva.
- En los niveles 1 y 2, los agentes del nivel 3 se ocultaron con `git sparse-checkout` para que no pudieran usarse.

## Cómo se validó

1. Los hallazgos de las 6 corridas se unificaron en una lista canónica de 21 (regla: mismo hallazgo si comparten la causa raíz).
2. Un agente verificador, en una sesión independiente y de solo lectura, comprobó cada hallazgo contra el código y citó la evidencia (`archivo:línea`).
3. Un humano auditó esa evidencia fila por fila y la confirmó o corrigió.
4. El humano asignó la relevancia: `sí` (lo corregiría ya), `no` o `depende` (requiere contexto de despliegue o de otro servicio).

## Resultados

**Validación:** 20 reales, 0 falsos positivos y 1 no concluyente (depende del frontend).
**Auditoría humana:** 0 correcciones en 21 filas; 51 minutos en total (≈2,4 min por hallazgo).
**Relevancia:** 9 `sí`, 12 `depende`, 0 `no`.

| Corrida | Reportados | Precisión | Recall relativo (20 reales) | Relevantes (de 9) | Costo | Costo por relevante |
|---|---|---|---|---|---|---|
| Nivel 1 · r1 | 1 | 100% | 5% | 1 | $0,92 | $0,92 |
| Nivel 1 · r2 | 3 | 100% | 15% | 2 | $0,72 | $0,36 |
| Nivel 2 · r1 | 3 | 100% | 15% | 3 | $1,51 | $0,50 |
| Nivel 2 · r2 | 2 | 100% | 10% | 1 | $1,36 | $1,36 |
| Nivel 3 · r1 | 21 | 100% | 100% | 9 | $0,72 | $0,08 |
| Nivel 3 · r2 | 20 | 100% | 95% | 9 | $0,78 | $0,09 |

**Reproducibilidad** (coincidencia de hallazgos entre las dos corridas, índice de Jaccard):

| Nivel | Coincidencia |
|---|---|
| 1 | 0,33 |
| 2 | 0,25 |
| 3 | ≈0,95 |

**Recall contra la línea base oficial** (`docs/challenges.md`, desafíos asignables al servicio `community`: 4, 12 y 13):

| Nivel | Desafíos detectados |
|---|---|
| 1 | 1 de 3 |
| 2 | 1 a 2 de 3 |
| 3 | 3 de 3 |

## Conclusiones

1. **Ningún nivel reportó falsos positivos.** El problema de los niveles 1 y 2 fueron los falsos negativos.
2. **El filtro de confianza ≥ 8 explica la diferencia.** De los hallazgos que descartó, todos los que se pudieron validar resultaron reales (uno quedó como no concluyente); además, el filtro es la parte más costosa en tokens y la que vuelve inestables los resultados.
3. **Ampliar las categorías (nivel 2) no basta** si se mantiene ese filtro: las categorías nuevas aparecieron antes del filtro y se perdieron después.
4. **Las herramientas deterministas resolvieron lo que el LLM dudaba.** Por ejemplo, un secreto versionado que el nivel 1 descartó por "no poder confirmarlo" lo detectó Gitleaks y se verificó con un comando de Git.
5. **El costo del nivel 3 es humano, no de tokens:** más hallazgos que revisar. Auditar la evidencia del agente tomó unos 2,4 minutos por hallazgo.
6. **12 de 21 hallazgos dependen del contexto** (despliegue, otro servicio o ejecución). Son los candidatos para pruebas dinámicas (DAST).

## Limitaciones

- Un solo servicio, en un solo lenguaje (Go).
- crAPI es público y el modelo lo conoce: algunas corridas citaron "cómo funciona crAPI". Sobre código propio, no visto, el resultado puede ser peor.
- El alcance por servicio corta el contexto entre servicios; varios descartes de los niveles 1 y 2 se debieron a eso. Afecta por igual a los tres niveles.
- La validación audita la evidencia de un agente; no es una validación humana independiente. El validador conocía el resumen del agente antes de auditar.
- La línea base oficial (`docs/challenges.md`) es pequeña para este servicio y usa la nomenclatura de 2019, por eso la métrica principal es el recall relativo.
- La instancia de validación usó las imágenes oficiales `latest` del 24/10/2026, no el commit exacto.

## Hallazgos por categoría

Por ser un repositorio público, aquí no se publican ubicaciones ni pasos de explotación. Los 21 hallazgos se distribuyen así:

| Categoría (OWASP API Security Top 10:2023) | Cantidad |
|---|---|
| API1 / API3: autorización a nivel de objeto y exposición de propiedades | 3 |
| API2: autenticación | 4 |
| API4: consumo de recursos sin límite | 3 |
| API5: autorización a nivel de función | 1 |
| API6: flujos de negocio sensibles | 1 |
| API8: configuración insegura (secretos, contenedores, CORS, depuración) | 4 |
| Sin categoría en API 2023 (inyección) | 3 |
| Robustez e integridad (sin categoría OWASP) | 2 |
| **Total** | **21** |

La clasificación es del autor; algunos hallazgos encajan en más de una categoría y se asignaron a la principal.
