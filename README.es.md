<h1 align="center">book-to-principles</h1>

<p align="center">
  <b>No le des el libro a tu agente. Enséñale las palabras del autor.</b><br>
  Destila un libro en el vocabulario compartido con el que tú y tu agente de código planificáis, construís y revisáis.
</p>

<p align="center">
  <a href="README.md">English</a> · <b>Español</b>
</p>

<p align="center">
  <a href="LICENSE.md"><img src="https://img.shields.io/badge/license-MIT-blue" alt="MIT"></a>
  <img src="https://img.shields.io/badge/status-pre--release-orange" alt="pre-release">
  <a href="https://github.com/virgiliojr94/book-to-skill"><img src="https://img.shields.io/badge/fork%20of-book--to--skill-lightgrey" alt="fork de book-to-skill"></a>
</p>

---

## Dos palabras que cambian el plan

Pídele a un agente de código "añade exportación a CSV en la página de informes" y lo normal es que el plan vuelva por capas:

```
1. Diseñar el esquema de exportación y la consulta a la BD
2. Construir el servicio de exportación
3. Añadir el endpoint de la API
4. Construir el botón en la UI y el flujo de descarga
5. Integrar y probar de punta a punta
```

Ahora añade una sola línea al prompt, *"Start with a tracer bullet"*, y el plan suele volver con otra forma:

```
Tracer bullet: un informe fijo → un CSV → un botón, funcionando de punta a punta hoy.
Después ensánchalo: consulta real, todas las columnas, streaming de ficheros grandes, errores.
```

No has explicado nada. No has escrito un párrafo sobre entrega incremental, bucles de feedback o riesgo de integración. Has usado **dos palabras** que *The Pragmatic Programmer* hizo famosas en 1999 y que el modelo ya ha leído miles de veces. Las palabras llevaban el método entero consigo.

*(Los planes de arriba son ilustrativos. El [echo test](#echo-test) es cómo mides el efecto en tu propio agente.)*

**book-to-principles hace que esas palabras sean deliberadas.** Le das un libro y te devuelve el vocabulario de trabajo del autor. Cada término viene con el comportamiento que debe provocar, el hábito por defecto que debe sustituir y una línea que puedes pegar en `CLAUDE.md` para que cada sesión empiece hablando ya ese idioma.

---

## De dónde viene la idea

Este proyecto nace de las ideas de **Matt Pocock**. Pocock es el autor de la colección [`mattpocock/skills`](https://github.com/mattpocock/skills) (`grill-me`, `domain-modeling`, `improve-codebase-architecture`, …). Varias de sus ideas, juntas, pedían una herramienta:

1. **Es un problema de comunicación, no de inteligencia.** El agente es lo bastante capaz. Lo que no puede es leer tus valores. Cada sesión empieza de cero (*Memento-driven development*), así que todo lo que te importa hay que decirlo otra vez, de forma explícita y lo más barata posible.
2. **Leading words (*Leitwörter*, palabras guía).** Algunos términos (*tracer bullet*, *deep module*, *vertical slice*, *software entropy*, *ubiquitous language*) ya están muy dentro del prior del modelo. Úsalos en un prompt o en una skill y el agente empieza a repetirlos en su razonamiento. Y, sobre todo, **cambia lo que hace**: deja de construir por capas horizontales y abre una ruta fina de punta a punta.
3. **El lenguaje de dominio como interfaz (DDD, Eric Evans).** Construye un glosario mientras diseñas y mételo en el propio código. Los prompts se acortan, las respuestas son menos verbosas y el agente se orienta en el repo con un `grep` del término.
4. **Hablar el mismo idioma.** Lo valioso no es el glosario. Lo valioso es que **tú, tu repo y tu agente uséis las mismas palabras para las mismas decisiones**.
5. **El contexto es el presupuesto.** El modelo razona mejor dentro de una "smart zone" (unos 150k tokens). Todo lo que se carga en cada sesión tiene que ser mínimo. Por eso el bloque para `CLAUDE.md` que genera esta herramienta tiene un tope de **300 tokens**.

book-to-principles convierte esas cinco ideas en un pipeline.

---

## El puente: de *leer* un libro a *pensar* con él

Hay dos formas de meter un libro en un agente, y resuelven problemas distintos.

```
                        ┌──────────────────────────────┐
                        │   tu libro / docs / papers   │
                        └──────────────┬───────────────┘
                                       │  el mismo extractor determinista
                                       ▼
               ┌───────────────────────┴───────────────────────┐
               │                                               │
     book-to-skill                                    book-to-principles
     el agente CONSULTA el libro                      el agente PIENSA con el libro
               │                                               │
   capítulos · glosario · patrones                leading words · principios · smells
   ~10–40K tokens, carga bajo demanda             ~5–6K en total, 300 siempre cargados
               │                                               │
   "¿Qué dice Ousterhout sobre                    "Keep this a deep module."
    la profundidad de los módulos?"               → el agente empuja la complejidad hacia dentro,
   → lo busca y responde                            rechaza el wrapper que solo reenvía,
                                                    y lo dice en su plan
```

| | [book-to-skill](https://github.com/virgiliojr94/book-to-skill) | **book-to-principles** |
|---|---|---|
| Pregunta que responde | *"¿Qué dice el libro sobre X?"* | *"¿Cómo hago que mi agente actúe como el autor, con las mínimas palabras?"* |
| El libro se convierte en | una **biblioteca** que el agente consulta | un **idioma** que el agente habla |
| Salida | resúmenes por capítulo + glosario + patrones + chuleta | 25–40 leading words + principios + smells + snippets |
| Cuándo está en contexto | cuando preguntas por él | siempre, en un bloque mínimo, y más bajo demanda |
| El éxito se ve en | tokens ahorrados frente a pegar el PDF | el agente **repite las palabras y cambia el plan** |

Funcionan bien juntos. Usa book-to-skill cuando quieras *estudiar* un libro. Usa book-to-principles cuando quieras que tu agente *trabaje* como lo haría el autor. Los dos comparten el mismo motor de extracción; la diferencia está en qué se destila.

---

## Cómo es una leading word

Un término solo se gana su sitio en el léxico si cambia lo siguiente que hace el agente. Cada entrada tiene campos que aplican esa regla (se generan en inglés, porque el prior del modelo está indexado por la frase original):

```markdown
## Tracer bullet  `phase: plan` · score 8/8

- **Canonical**: build one thin, real path through every layer first; aim by watching where it lands.
- **Trigger**: a feature touches more than one layer or service.
- **Behaviour**: ship the thinnest end-to-end path first, then widen it.
- **Displaces**: scaffold every layer in full, integrate at the end.
- **Say it as**: "Start with a tracer bullet: one path, end to end, running today."
- **Echo test**: the plan's first step names a single path across all layers.
- **Not to be confused with**: *prototype* (thrown away); a tracer bullet is kept and grown.
```

La línea **`Displaces`** es el filtro. Si no puedes nombrar el comportamiento por defecto que una palabra sustituye, es una entrada de glosario y no una leading word, así que se descarta. Los candidatos se puntúan en cuatro ejes (nombrado · denso · probable en el prior · cambia el comportamiento, de 0 a 2 cada uno). Solo entran los que sacan **6/8 o más**, y el léxico tiene un tope de 40, porque cada palabra de más compite por la atención del agente.

---

## Qué obtienes

```
~/.agents/skills/<autor>-principles/
├── SKILL.md            ≤1.5K tokens · las 8–12 palabras principales, cuándo cargar más, cómo hablar
├── leading-words.md    25–40 entradas como la de arriba
├── principles.md       10–25 reglas de decisión: cuándo · haz · en lugar de · porque · Check
├── smells.md           anti-patrones como señales rápidas para la revisión
├── snippets/
│   ├── claude-md.md    ≤300 tokens · pégalo en CLAUDE.md / AGENTS.md → las palabras siempre activas
│   ├── grill.md        8–15 preguntas de decisión en el idioma del autor, antes de un cambio grande
│   └── review.md       un prompt de revisión: ejecuta cada Check, da juicios, el repo manda
└── bindings.md         (opcional) las palabras del autor ↔ las de tu repo
```

Y después:

```
pega snippets/claude-md.md en CLAUDE.md        → cada sesión habla el idioma del autor
"grill me with ousterhout-principles"          → una entrevista en los términos del autor antes de un cambio grande
"review this with ousterhout-principles"       → principios + smells sobre un diff
"ousterhout-principles bind ./mi-repo"         → mapea las palabras sobre tu código
```

---

## Vincúlalo a tu repo: el movimiento del lenguaje ubicuo

El vocabulario de un libro es genérico. El de tu repo no. El modo **Bind** lee tu `CLAUDE.md`, `AGENTS.md`, `CONTEXT.md` y tus ADRs (solo lectura), busca con `grep` dónde vive ya cada concepto y te hace **como mucho 10 preguntas, de una en una**. Son decisiones, no un examen:

> *"En este repo, ¿*deep module *son los paquetes de `pipelines/`, la capa `services/` o algo distinto?"*

El resultado es `bindings.md`: el término del autor, tu término y dónde vive en el código. Cuando no coinciden, **gana el término de tu repo**; el del libro pasa a ser un alias. No se escribe nada en tu `CLAUDE.md` salvo que respondas literalmente `yes`.

A partir de ahí, el libro ya no es algo que tú y tu agente leéis. Son palabras que los dos usáis.

---

<a id="echo-test"></a>
## Demuéstralo: el echo test

Un léxico que nadie repite es solo un glosario. Antes de fiarte de uno, haz la prueba en una sesión nueva:

1. Pide un plan para una funcionalidad mediana **con** `snippets/claude-md.md` en el contexto.
2. Pide el mismo plan, en el mismo repo, **sin** él.
3. Cuenta qué leading words aparecen solo en el primer plan.

**Pasa** = al menos 3 palabras repetidas **y** la *forma* del plan se ha movido en la dirección de alguna línea `Behaviour` (rebanadas en vez de capas, un deep module en vez de tres superficiales). Si aparecen las palabras pero el plan no cambia, las líneas `Behaviour`/`Displaces` son demasiado vagas: reescríbelas antes de fiarte del léxico.

---

## Instalación

```bash
# Como skill de agente (Claude Code, Copilot CLI, Amp, Codex, Hermes, OpenClaw)
npx skills add edumesones/book-to-principles

# O a mano: la carpeta TIENE que llamarse book-to-principles
git clone https://github.com/edumesones/book-to-principles.git ~/.claude/skills/book-to-principles
```

Comprueba qué extractores tienes: `python scripts/extract.py --check`. Texto plano, Markdown, HTML y la mayoría de PDFs funcionan con la librería estándar. Calibre solo hace falta para MOBI/AZW.

## Uso

```
/book-to-principles <ruta|carpeta|glob>... [slug] [--bind <raíz-del-proyecto>]
```

| Modo | Se activa con | Hace |
|---|---|---|
| **Destilado completo** | una ruta | extraer → puntuar → escribir la skill → escanear → echo test |
| **Solo léxico** | "lexicon only" / "just the words" | muestra la lista de candidatos puntuada; no escribe nada |
| **Bind** | `--bind <repo>` sobre una skill existente | entrevista + `bindings.md` + bloque opcional en `CLAUDE.md` |
| **Fold-in** | fuentes nuevas + un slug existente | vuelve a puntuar la unión, mantiene el tope de 40, regenera los snippets |

## Libros que mejor funcionan

Libros cuyo vocabulario ya es canónico, para que el prior del modelo haga la mitad del trabajo:

- Hunt & Thomas, *The Pragmatic Programmer*: tracer bullets, DRY, orthogonality, broken windows
- John Ousterhout, *A Philosophy of Software Design*: deep modules, information hiding, define errors out of existence
- Eric Evans, *Domain-Driven Design* (capítulos 1–3): ubiquitous language, bounded context, model-driven design
- Martin Fowler, *Refactoring*: los code smells, con nombre y catalogados

También funciona con los documentos de diseño de tu propio equipo. Ahí el prior no ayuda, así que los términos tienen que ganarse el sitio solo por densidad y por comportamiento.

---

## Copyright

Este repositorio **no incluye contenido de ningún libro**. Lo apuntas a ficheros que ya son tuyos.

- **El procesamiento es local.** La extracción se ejecuta en tu máquina. El texto que envíes a un modelo en la nube sigue los términos habituales de ese proveedor.
- **La salida son tus notas.** Un vocabulario y reglas parafraseadas, con cada formulación limitada a 25 palabras o menos. Nunca copia pasajes literales.
- **Las skills de libros de terceros, privadas.** Al publicar, el generador crea repos privados por defecto y solo hace uno público si respondes a la pregunta de una palabra con `public` y la fuente lo permite.

## Estado

Fork en pre-release. La spec del generador (`SKILL.md`) está completa. El escáner de seguridad y los tests se están adaptando al nuevo formato de salida, y lo siguiente es la primera ejecución completa y el echo test con un libro real. Los resultados se publicarán aquí cuando estén medidos.

## Créditos

- [**book-to-skill**](https://github.com/virgiliojr94/book-to-skill) de [@virgiliojr94](https://github.com/virgiliojr94) (MIT): el motor de extracción, la lógica de instalación multi-host y el escáner de seguridad sobre los que se construye este fork.
- **Matt Pocock**, entrevistado en *The Pragmatic Engineer Podcast*: leading words, lenguaje compartido y el presupuesto de la smart zone. Ver [`mattpocock/skills`](https://github.com/mattpocock/skills).
- Andrew Hunt & David Thomas, John Ousterhout, Eric Evans, Martin Fowler: el vocabulario que esta herramienta existe para transmitir.

## Licencia

MIT. Cubre el código y la spec del generador de este repositorio, **no** los libros ni documentos que proceses con él.
