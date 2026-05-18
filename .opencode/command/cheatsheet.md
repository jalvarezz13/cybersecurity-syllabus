---
description: Generate a teaching cheat sheet for a 4Geeks Cybersecurity class day from syllabus content (markdown + PDF)
agent: build
---

# /cheatsheet — Generate a class-day cheat sheet (markdown + PDF)

Argument: `$ARGUMENTS` — day number + list of contents (file names, lesson titles, or keywords).

Examples:
- `/cheatsheet 11 web-security web-application-security-solutions application-security enduser-network-security antivirus-spyware`
- `/cheatsheet 11 "Seguridad web" "Soluciones AppSec" "Usuario final" "Antivirus"`
- `/cheatsheet 12 application-security threats-vulnerabilities-data-security incident-management`

---

## 1. Paths (hardcoded — DO NOT prompt user for these)

```bash
SYLLABUS_ROOT="/Users/jalvarezz13/Proyectos/cybersecurity-syllabus"
OUTPUT_ROOT="/Users/jalvarezz13/Proyectos/cheatsheets-4geeks"
TEMPLATE_DAY_10="${OUTPUT_ROOT}/day_10.md"
TEMPLATE_DAY_11="${OUTPUT_ROOT}/day_11.md"
MD2PDF="/Users/jalvarezz13/Documents/Dev/Scripts/md2pdf.py"
```

If `OUTPUT_ROOT` does not exist, create it. Never overwrite an existing day file without explicit user confirmation.

---

## 2. Argument parsing

`$ARGUMENTS` must contain:

1. **Day number** (first integer found, 1–99). Output file: `day_<NN>.md` (zero-padded to 2 digits if < 10).
2. **Content list** (rest of arguments). Items can be:
   - **File stems** (`web-security`, `enduser-network-security`) — match against `*.es.md` in `SYLLABUS_ROOT/**`.
   - **Quoted titles** (`"Seguridad web"`) — match against `title:` in frontmatter.
   - **Keywords** (`web`, `antivirus`) — fuzzy match against filename + title.

If parsing fails or the list is empty, abort and ask the user for clarification.

---

## 3. Preflight

1. Verify `SYLLABUS_ROOT` exists. Abort with clear error if not.
2. Verify `OUTPUT_ROOT` exists. Create with `mkdir -p` if missing.
3. Verify `MD2PDF` exists and is executable. If not, warn the user that the PDF step will be skipped (do NOT abort — markdown can still be generated).
4. Check whether `day_<NN>.md` already exists in `OUTPUT_ROOT`. If yes:
   - Show the user the existing file's first 20 lines.
   - Ask: **overwrite / append `_v2` / abort**.
5. Read `TEMPLATE_DAY_10` and `TEMPLATE_DAY_11` (if both exist) — these are the **style ground truth**. The new cheat sheet MUST match their structure, tone, and formatting.

---

## 4. Content resolution (parallel reads)

For each content item in the argument list:

1. Glob `SYLLABUS_ROOT/**/<item>.es.md` and `SYLLABUS_ROOT/**/<item>*.es.md`.
2. If no hit, fuzzy-match against all `*.es.md` filenames in the syllabus.
3. If still no hit, search frontmatter `title:` fields.
4. If multiple matches, prefer:
   - Files inside the directory matching the day's module (use the `README.md` of `04-seguridad-redes-1`, `05-seguridad-redes-2`, etc. to map day → module).
   - Then `.es.md` over `.md`.
5. If zero matches for an item, **abort** and report unresolved items to the user.

**Always read all resolved files in parallel** with the `read` tool. Do NOT read sequentially.

Also read the **module's `README.md`** to detect:
- Which day this module corresponds to.
- The official lab/exercise link for that day (look for `🧪` markers).
- The official lesson order for that day (to confirm the user's content list aligns).

If the user's content list mismatches the official README order for that day, **flag it as a "Discrepancia con README oficial" note** at the end of the generated cheat sheet (do not block — the user knows what they are teaching).

---

## 5. Generation — strict format

The generated file MUST follow this structure (study `day_10.md` and `day_11.md` and copy the pattern exactly):

### 5.1 Header

```markdown
# Clase Día <NN> — <Class Title> (Cheat Sheet)

> **Duración**: 90 min · **Nivel**: principiantes · **<N> bloques + <M> demos**
> **Lectura previa**: `<file1>.es.md` · `<file2>.es.md` · …
> **Lab final**: [<Lab name>](<lab URL from README>)
```

`<Class Title>` is inferred from the dominant theme of the readings. If the user provided a custom title in arguments, use it.

### 5.2 Mapa de la clase

A markdown table with columns: `Min | Bloque | Objetivo`. Total minutes must add up to 90. Distribute as:
- 5 min apertura
- 4–5 content blocks (15–20 min each)
- 2–3 demos (3–5 min each, embedded inside relevant blocks)
- 3 min cierre

### 5.3 Apertura (5 min)

Must contain:
- **Hook**: a striking statistic or question that grabs attention. Use real data from the readings if available.
- **Pregunta ancla**: an open question to throw at the class.
- **Objetivos**: 3–5 numbered learning outcomes ("Al acabar la clase, los alumnos podrán…").

### 5.4 Content blocks (one per major reading or theme)

Each block MUST include:

1. `## <N>. <Block title> (<minutes> min)` — heading with timing.
2. **Idea clave** in bold — single sentence summarising the block.
3. **Analogía maestra** — concrete metaphor from everyday life. Reuse it throughout the block.
4. At least one **comparison table** (top threats, tool comparison, etc.).
5. **Bombas de profundidad** — 2-3 deep-dives on the most important sub-topics.
6. **Frase para grabar** — a quotable one-liner students should remember.
7. **Pregunta para clase** — at least one Socratic question.
8. **DEMO** (optional, in 2–3 of the blocks) — runnable commands or a clear script for live demonstration. Prefer demos with minimal setup (built-in tools, public web services).
9. **Cierre del bloque** — a quotable summary line.

### 5.5 Cierre + tarea (3 min)

- **Resumen de 30 segundos** — verbatim closing speech.
- **Tarea para casa** — link to the official lab from the module README.
- **Q&A** mention.

### 5.6 Tail sections (mandatory)

In this exact order:

1. `## Glosario rápido` — alphabetical-ish list of acronyms used in the class with one-line definitions.
2. `## Preguntas para lanzar en clase` — 8–10 numbered questions with expected answers (`→ <answer>`).
3. `## Errores comunes a evitar al explicar` — bullet list with `❌` prefix; mistakes the teacher should NOT make.
4. `## Material de respaldo si te sobra tiempo o quieren más` — real-world cases and external resources (link them).
5. `## Notas técnicas` (only if applicable) — discrepancies with the official syllabus README, fused readings, etc.

---

## 6. Style rules (non-negotiable)

- **Language**: Spanish from Spain. Use "vosotros", not "ustedes".
- **Tone**: practical, business-oriented, slightly informal but technical. Match `day_10.md` and `day_11.md` exactly.
- **No fluff**. Every sentence earns its place.
- **Tables** for comparisons. Lists for actions. Quotes for memorable lines.
- **Code blocks** must declare language fence (`bash`, `python`, `text`, …).
- **Emojis**: only in tail sections (`🧪` for labs, `📖` for reading, `🛠️` for hands-on, `❌` for errors, `⚠️` for warnings). Do NOT sprinkle emojis in body content.
- **Bold** is for genuinely critical concepts, not decoration.
- **Analogies must be concrete**: a house, a shop, a postman, a submarine — not abstract.
- **No AI mentions**, no "as an AI", no generation disclaimers anywhere in the output.

---

## 7. Length target

- Total file: **300–500 lines**.
- If the topic is very dense and goes over 500, that's acceptable. Do NOT trim depth to hit a number.
- If under 300, reread the source material and add more depth to the bombas de profundidad and Material de respaldo sections.

---

## 8. Write markdown

```bash
OUTPUT_MD="${OUTPUT_ROOT}/day_<NN>.md"
# Use the write tool, NOT shell redirection.
```

After writing the markdown:

1. `wc -l "$OUTPUT_MD"` — confirm line count is in range.
2. Confirm the file shows up in `ls -la "$OUTPUT_ROOT"`.

---

## 9. Generate PDF (mandatory step)

Once the markdown is verified, generate the PDF using the local `md2pdf` script:

```bash
OUTPUT_PDF="${OUTPUT_ROOT}/day_<NN>.pdf"
"$MD2PDF" "$OUTPUT_MD" "$OUTPUT_PDF"
```

Notes:
- The script's shebang is `uv run`, so it runs without an explicit interpreter.
- It auto-installs Chromium on first run (cached afterwards). Allow up to 60 seconds for the first invocation.
- If `MD2PDF` is missing or fails, **report the error** but consider the markdown step a success. Suggest the user run `md2pdf "$OUTPUT_MD"` manually.

After generation:

1. `ls -lh "$OUTPUT_PDF"` — confirm the PDF exists and report its size.
2. Verify the file is non-empty (`stat -f%z "$OUTPUT_PDF"` > 1000 bytes).

---

## 9.5 Commit + push al repo de cheatsheets (mandatory)

Inmediatamente después de verificar que el PDF existe y no está vacío, hacer commit + push de **ambos** ficheros al repositorio `cheatsheets-4geeks`.

```bash
cd "$OUTPUT_ROOT"

# Sanity: confirmar que es un repo git
git -C "$OUTPUT_ROOT" rev-parse --is-inside-work-tree

# Stage SOLO los dos ficheros del día
git -C "$OUTPUT_ROOT" add "day_<NN>.md" "day_<NN>.pdf"

# Conventional commit, lowercase, imperativo, una sola línea (sin body)
git -C "$OUTPUT_ROOT" commit -m "feat(day-<NN>): add cheat sheet for <slug-corto-del-tema>"

# Push a la rama actual (sin --force, sin --no-verify)
git -C "$OUTPUT_ROOT" push
```

Reglas:
- `<slug-corto-del-tema>` debe ser kebab-case y derivar del título de la clase (ej: `acl-firewall-wifi`, `web-security`, `incident-management`). Máximo 4 palabras.
- **Una única commit** que incluye `.md` y `.pdf`. No separarlos.
- **Nunca** usar `--no-verify`, `--amend`, `--force`, `-i`, ni firmar con co-author / AI attribution. Cumplir las anti-spam rules de GitHub.
- Si `git push` falla por falta de upstream o de remoto, **reportarlo** y dejar el commit local hecho — no inventar remotos.
- Si hay cambios pre-existentes sin commitear en `OUTPUT_ROOT` ajenos al día actual, **NO** los toques: `git add` debe limitar el stage a `day_<NN>.md` y `day_<NN>.pdf` exclusivamente.
- Si el commit falla por hook (lint, formato, etc.): reportar el fallo, no intentar `--no-verify`, no intentar `--amend`. Devolver control al usuario.

Reportar en el resumen final el hash corto del commit (`git -C "$OUTPUT_ROOT" rev-parse --short HEAD`) y si el push fue exitoso.

---

## 10. Final summary to user

Short, factual. No flattery. Match the `day_10.md` / `day_11.md` style.

```
Cheat sheet creada:
- `day_<NN>.md` (<N> líneas)
- `day_<NN>.pdf` (<size> KB)
- Commit: `<short-hash>` · push: <ok | failed (motivo)>

Estructura en 90 min:
- 0–5    Apertura
- 5–25   1. <Bloque 1> + DEMO 1
- 25–50  2. <Bloque 2>
- ...
- 87–90  Cierre + tarea

Notas: <discrepancias / fusiones / decisiones tomadas>.
```

---

## 11. Hardblocks

- **Never** modify files inside `SYLLABUS_ROOT`. Read-only.
- **Never** invent a lab link. If no lab is found in the README, write `_(no lab oficial encontrado en el README del módulo)_`.
- **Never** invent statistics or quotes. Pull them from the readings or omit.
- **Never** mention AI, agents, generation, or the source of the content. The cheat sheet must read as if a human teacher wrote it.
- **Never** include attribution trailers, "generated by" footers, or similar.
- **Never** use `cat <<EOF` or shell redirection for the markdown output. Use the `write` tool only.
- **Never** skip the PDF step silently. If the PDF generation fails, report the failure prominently in the final summary.
- **Never** skip the commit+push step silently. If git fails, report it prominently and leave the working tree clean (no half-staged changes).
- **Never** stage files in `OUTPUT_ROOT` that don't belong to the current day. Only `day_<NN>.md` and `day_<NN>.pdf`.
- **Never** use `--no-verify`, `--amend`, `--force`, or add `Co-authored-by` / AI attribution trailers in the commit.

---

## 12. Failure handling

- **Unresolved content item**: list unresolved items and ask the user to confirm filenames or correct typos. Do not generate a partial cheat sheet.
- **Day already exists**: ask overwrite / `_v2` / abort. Default = abort.
- **Source files conflict** (multiple matches with no clear winner): list candidates, ask the user to pick.
- **Glob/read fails**: report the exact path and error. Do not guess.
- **PDF generation fails**: keep the markdown, report the error, suggest the manual command.
- **Git commit fails (hook, lint, etc.)**: report the exact error, do NOT retry with `--no-verify` or `--amend`. Leave the markdown + PDF in place and return control to the user.
- **Git push fails (no upstream, auth, network)**: report the error, leave the local commit in place, suggest the manual `git push` the user should run.
- **Not a git repo**: report it clearly; the markdown + PDF stay, no commit attempted.
