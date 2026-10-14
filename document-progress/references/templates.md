# Templates e mapa de áreas — document-progress

Usa estes templates literalmente (substituindo os `<…>`). Mantém a estrutura igual
entre execuções para os docs serem previsíveis.

---

## Template — entrada de ROADMAP (`docs/ROADMAP.md`)

Cabeçalho do ficheiro (só quando o crias pela 1.ª vez):

```markdown
# Roadmap / Timeline

Registo cronológico do que foi sendo feito. Mais recente primeiro.
Cada entrada resume a sessão; o detalhe vive em `docs/features/<area>.md`.
```

Entrada (acrescentar **no topo**, logo abaixo do cabeçalho):

```markdown
## YYYY-MM-DD — <título curto da sessão>
<descrição genérica do que foi feito nesta sessão, 1–3 linhas>
- Áreas: [tasks](features/tasks.md) · [emails](features/emails.md) · [calendar-blocks](features/calendar-blocks.md) · [calendar](features/calendar.md)
- Commits: `hash1`, `hash2`, …
```

(Inclui apenas os links das áreas realmente tocadas.)

---

## Template — doc de funcionalidade (`docs/features/<area>.md`)

Quando o ficheiro **não existe**, cria com este esqueleto:

```markdown
# <Área> — Documentação

> Estado: ✅ feito | ⚠️ parcial | ⛔ por fazer | 🗓️ planeado  ·  Última actualização: YYYY-MM-DD

## Visão geral
<para que serve esta área do produto, em 2–4 linhas>

## Funcionalidades

### <Nome da funcionalidade>
- **O quê**: <o que faz, do ponto de vista do utilizador>
- **Como funciona**: <fluxo; modelo de dados / endpoints / comportamento no frontend>
- **Ficheiros-chave**: `caminho/ficheiro.py:linha`, `caminho/Componente.tsx`
- **Adicionado/alterado em**: YYYY-MM-DD (commits `hash`, …)

## Modelo de dados
<modelos/campos relevantes desta área, se aplicável>

## Endpoints
<endpoints REST relevantes, se aplicável>

## Por fazer
<gaps conhecidos; linkar a fonte: MISSING_FRONTEND_ANALYSIS.md, orphaned-menus-analysis.md,
docs/implementations/todo/, PLAN.md, ou TODOs no código. Omitir a secção se nada pendente.>

## Notas e limitações
<edge cases, decisões, pendências>
```

Escolhe **um** símbolo no cabeçalho de estado: `⚠️ parcial` quando há funcionalidades feitas
mas também gaps relevantes em "Por fazer".

Quando o ficheiro **já existe**: adiciona uma nova `### <Nome da funcionalidade>`
em "Funcionalidades", ou edita a existente se a funcionalidade mudou; actualiza
"Modelo de dados"/"Endpoints"/"Por fazer"/"Notas" se for caso disso; revê o símbolo de
estado; e actualiza a linha `> Estado: … · Última actualização:`. Não reescrevas o ficheiro inteiro.

---

## Mapa de áreas (caminho → área)

Extensível. Acrescenta linhas quando aparecerem áreas novas.

| Caminhos (backend / frontend)                                                                 | Área              |
|-----------------------------------------------------------------------------------------------|-------------------|
| `backend/apps/tasks/models.py` (Task, estados, checklists), `frontend/src/features/tasks/TaskDetail.tsx`, `KanbanBoard.tsx`, `GroupedBoardView.tsx` | `tasks`           |
| `backend/apps/emails/`, `frontend/src/features/emails/`                                       | `emails`          |
| `backend/apps/tasks/scheduling.py`, `TaskCalendarEvent` (modelo), `place_today_block`/`ensure_auto_block`/`find_block_slot`, flag `is_done`, signals/celery de blocos | `calendar-blocks` |
| `frontend/src/features/tasks/ProjectCalendarPage.tsx`, `CalendarView.tsx`, `TaskQuickView.tsx` | `calendar`        |

Regras de desempate:
- Mudanças em `scheduling.py`/`TaskCalendarEvent`/`is_done` → **calendar-blocks** (não `tasks`).
- Mudanças de UI do calendário (FullCalendar, hover/quick view) → **calendar**.
- Lógica de associação email↔tarefa que vive em `apps/emails/` → **emails** (mesmo que crie blocos; menciona o cruzamento e linka `calendar-blocks`).
- Fallback: `backend/apps/<x>/` → área `<x>`; `frontend/src/features/<x>/` → área `<x>`.
