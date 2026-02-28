# CLAUDE.md - MedOS Obsidian Vault

> Instrucciones para Claude al trabajar dentro del vault de MedOS.
> Este archivo es educativo: explica conceptos de Obsidian para que el usuario aprenda.

---

## QUE ES ESTE VAULT

MedOS es el proyecto de Healthcare OS -- un sistema operativo AI-native para healthcare en USA.
Este Obsidian vault es la base de conocimiento, gestion de proyectos, y documentacion tecnica.

**Vault root:** `Z:\medos\` -- todo el contenido vive aqui.

---

## ESTRUCTURA DE CARPETAS

```
Z:\medos\                     # VAULT ROOT (abrir esta carpeta en Obsidian)
  .obsidian/                  # Configuracion de Obsidian (no tocar manualmente)
  00-inbox/                   # Captura rapida -- notas nuevas caen aqui
  01-daily/                   # Daily notes organizadas por YYYY/MM/
  02-meetings/                # Notas de reuniones con stakeholders
  03-projects/                # Epics, features activas, sprints
  04-architecture/            # Decisiones tecnicas y system design
    adr/                      # Architecture Decision Records
    diagrams/                 # Diagramas (Excalidraw, Mermaid)
    system-design/            # Documentos de diseno de sistema
  05-domain/                  # Conocimiento de healthcare (PKM)
    clinical/                 # Workflows clinicos, terminologia medica
    regulatory/               # HIPAA, FDA, state laws
    standards/                # FHIR, HL7, X12, ICD-10, CPT
    billing/                  # Revenue cycle, claims, denials
  06-engineering/             # Standards de ingenieria
    api-specs/                # Especificaciones de API
    infrastructure/           # AWS, Terraform, CI/CD
    security/                 # Security policies, threat models
    testing/                  # Testing strategies, QA
  07-business/                # GTM, pricing, hiring, investors
  08-retrospectives/          # Retros y post-mortems
  09-archive/                 # Notas completadas o deprecated
  _MOCs/                      # Maps of Content (indice de navegacion)
  assets/
    attachments/              # Imagenes, PDFs pegados en notas
    templates/                # Templates para nuevas notas
    excalidraw/               # Diagramas Excalidraw
```

### Donde va cada tipo de archivo

| Tipo de nota | Carpeta | Template |
|-------------|---------|----------|
| Ideas sueltas, notas rapidas | `00-inbox/` | (ninguno) |
| Work log diario | `01-daily/YYYY/MM/` | `tpl-daily` |
| Notas de reunion | `02-meetings/` | `tpl-meeting` |
| Feature o epic | `03-projects/` | `tpl-epic` |
| Decision record | `04-architecture/adr/` | `tpl-adr` |
| Concepto de healthcare | `05-domain/<subcarpeta>/` | `tpl-domain` |
| Bug investigation | Donde corresponda | `tpl-bug` |
| Retro o post-mortem | `08-retrospectives/` | (libre) |
| Nota obsoleta | `09-archive/` | (mover ahi) |

---

## OBSIDIAN CONCEPTS (GUIA EDUCATIVA)

### Wikilinks [[]]

Obsidian usa `[[wikilinks]]` para conectar notas entre si.
Esto es lo que hace a Obsidian poderoso -- crea un grafo de conocimiento.

```markdown
Ver el [[HEALTHCARE_OS_MASTERPLAN]] para detalles.
La arquitectura esta en [[MOC-Architecture]].
Relacionado con [[FHIR R4]] y [[Prior Authorization]].
```

**Reglas:**
- SIEMPRE usar `[[wikilinks]]` en vez de `[links](markdown)` tradicionales
- Obsidian resuelve el nombre automaticamente, no necesitas paths completos
- Si la nota no existe aun, el link se muestra en gris -- y al hacer click, la crea
- Para linkear a una seccion especifica: `[[nota#seccion]]`
- Para mostrar texto diferente: `[[nota|texto visible]]`

### Frontmatter (YAML)

El bloque al inicio de cada nota entre `---` es metadata estructurada.
Obsidian lo muestra como "Properties" en la UI.

```yaml
---
type: epic              # Tipo de nota (para filtrar con Dataview)
date: "2026-02-27"      # Fecha de creacion
status: active          # Estado actual
tags:                   # Tags para categorizar
  - project
  - phase-1
priority: high          # Prioridad
owner: "lxhxr"          # Responsable
---
```

**Reglas:**
- TODA nota creada por Claude DEBE tener frontmatter con al menos `type`, `date`, y `tags`
- Las fechas van entre comillas: `"2026-02-27"` (YAML las interpreta mal sin comillas)
- Tags van como lista YAML, no inline

### Tags

Los tags categorizan notas y permiten buscarlas rapidamente.

**Taxonomia de tags para MedOS:**

```
# Por tipo de contenido
#daily, #meeting, #adr, #epic, #bug, #domain, #retro

# Por area del sistema
#module-a, #module-b, #module-c, ... #module-h
#frontend, #backend, #infra, #ai, #data

# Por fase del proyecto
#phase-1, #phase-2, #phase-3

# Por dominio de healthcare
#clinical, #regulatory, #billing, #standards
#fhir, #hl7, #x12, #hipaa

# Por prioridad/estado
#urgent, #blocked, #review-needed

# Especiales
#decision (para decisiones importantes)
#learning (para cosas que aprendimos)
#risk (para riesgos identificados)
```

**Reglas:**
- Tags van en el frontmatter, NO inline en el texto (para consistencia)
- Usar kebab-case: `#phase-1` no `#Phase1`
- No inventar tags nuevos sin necesidad -- buscar primero si ya existe uno

### Maps of Content (MOCs)

Los MOCs son notas-indice que agrupan y conectan otras notas por tema.
Viven en `_MOCs/` y usan Dataview queries para auto-actualizarse.

- `Home.md` -- Dashboard principal, punto de entrada
- `MOC-Architecture.md` -- Todo sobre arquitectura
- `MOC-Domain-Knowledge.md` -- Conocimiento de healthcare
- `MOC-Projects.md` -- Proyectos activos y roadmap
- `MOC-Business.md` -- Estrategia de negocio

**Para que los queries funcionen**, instalar el plugin community **Dataview**.

### Templates

En `assets/templates/` hay templates para cada tipo de nota.
En Obsidian: `Ctrl+T` o Command Palette > "Insert template".

Templates disponibles:
- `tpl-daily` -- Para daily notes
- `tpl-meeting` -- Para notas de reunion
- `tpl-adr` -- Para Architecture Decision Records
- `tpl-epic` -- Para proyectos/epics
- `tpl-domain` -- Para conocimiento de healthcare
- `tpl-bug` -- Para investigacion de bugs

---

## REGLAS PARA CLAUDE

### Al crear notas

1. **SIEMPRE** incluir frontmatter YAML con `type`, `date`, y `tags`
2. **SIEMPRE** usar `[[wikilinks]]` para referencias a otras notas
3. **SIEMPRE** poner la nota en la carpeta correcta segun el tipo
4. **NUNCA** crear notas sueltas en el root del vault (excepto CLAUDE.md y HEALTHCARE_OS_MASTERPLAN.md)
5. **NUNCA** usar markdown links `[text](url)` para notas internas (usar `[[wikilinks]]`)
6. Los links externos (URLs web) SI usan markdown links `[text](https://...)`

### Al editar notas

1. **NUNCA** borrar frontmatter existente -- solo agregar o actualizar campos
2. **NUNCA** cambiar el `type` de una nota sin razon
3. Si una nota esta en la carpeta equivocada, moverla a la correcta
4. Actualizar `status` cuando cambie el estado de algo

### Naming conventions

- Daily notes: `YYYY-MM-DD.md` (ej: `2026-02-27.md`)
- ADRs: `ADR-NNN-titulo-corto.md` (ej: `ADR-001-fhir-native-data-model.md`)
- Otros: titulo descriptivo en kebab-case o titulo natural

### Healthcare compliance

- **NUNCA** poner PHI (Protected Health Information) en el vault
- **NUNCA** poner datos reales de pacientes, ni de prueba
- Usar datos sinteticos o placeholders para ejemplos
- Marcar notas con info sensible con tag `#confidential`

---

## PLUGINS RECOMENDADOS (Community)

Estos se instalan desde Settings > Community Plugins > Browse:

| Plugin | Para que | Prioridad |
|--------|----------|-----------|
| **Dataview** | Queries automaticos en MOCs | CRITICO |
| **Templater** | Templates avanzados con variables | Alto |
| **Calendar** | Navegacion visual de daily notes | Alto |
| **Excalidraw** | Diagramas de arquitectura | Medio |
| **Git** | Backup automatico con Git | Alto |
| **Kanban** | Boards para project management | Medio |
| **Mermaid** | Diagramas en markdown | Medio |

---

## QUICK REFERENCE

```
Ctrl+O          Abrir nota por nombre (Quick Switcher)
Ctrl+P          Command Palette (buscar cualquier comando)
Ctrl+T          Insertar template
Ctrl+N          Nueva nota
Ctrl+E          Toggle edit/preview mode
Ctrl+Click      Abrir link en nueva tab
[[              Iniciar wikilink (autocomplete)
#               Iniciar tag (autocomplete)
```
