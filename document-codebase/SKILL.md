---
name: document-codebase
description: >
  Documenta de raiz o estado do codebase do projecto actual, lançando uma EQUIPA DE
  AGENTES em paralelo (um por área funcional) para ser rápido. Audita backend/apps e
  frontend/src/features, cruza com a documentação existente (README, implementations,
  PLAN, MISSING/orphaned), e gera/normaliza docs/features/<area>.md por área, com estado
  (feito/parcial/por fazer) e secção "Por fazer". Usar para criar a base de documentação
  inicial ou re-auditar uma ou mais áreas. Triggers: "documenta o codebase", "documentação
  inicial", "baseline de docs", "audita a documentação", "auditar docs", "o que está feito
  e por fazer", "document codebase", "/document-codebase".
---

# document-codebase

Cria/normaliza a **base de documentação** do projecto: um `docs/features/<area>.md` por área
funcional, cada um com **estado** e secção **"Por fazer"**, mantendo `docs/README.md` como
índice-mestre. Complementa a `document-progress` (essa actualiza incrementalmente por sessão;
esta faz o varrimento/audit de fundo).

O codebase é grande (dezenas de apps + features). Documentar tudo em série seria lento e
encheria o contexto. Por isso o trabalho é feito por uma **equipa de agentes**: o orquestrador
(tu) prepara o terreno e reparte, e **um subagente por área** lê o código dessa área e escreve
o respectivo doc — em paralelo. Como cada agente escreve um ficheiro diferente
(`docs/features/<area>.md`), não há conflitos.

Partilha o formato com a `document-progress`: usa os templates em
`~/.claude/skills/document-progress/references/templates.md` (doc de funcionalidade, mapa de
áreas) e o cabeçalho de estado abaixo.

Se o utilizador indicar áreas (ex.: "documenta o codebase: projects budgets"), processa só essas.

## Fase A — Orquestrador: preparar e repartir (em série, rápido)

1. **Ler o panorama** (não duplicar — vais referenciar): `docs/README.md` (índice-mestre +
   estado), `docs/ROADMAP.md`, lista de `docs/features/*`, e os índices de
   `docs/implementations/{done,todo,structure,use_cases}/` e do backlog (`PLAN.md`,
   `MISSING_FRONTEND_ANALYSIS.md`, `docs/orphaned-menus-analysis.md`). Não leias tudo a fundo —
   só o suficiente para saber que fontes existem e a que área pertencem.
2. **Inventariar áreas**: `ls backend/apps/` e `ls frontend/src/features/`. Mapeia para áreas
   funcionais (consolida app+feature relacionados, ex.: app `tasks` + feature `tasks` → `tasks`;
   respeita as áreas já existentes em `docs/features/`). Apresenta ao utilizador a **lista de
   áreas** a documentar e confirma antes de lançar a equipa.
3. **Construir o briefing por área**: para cada área, junta os ponteiros que o agente precisa —
   caminhos de código (`backend/apps/<x>/`, `frontend/src/features/<y>/`), docs/fontes existentes
   relevantes a cruzar, e o ficheiro de saída (`docs/features/<area>.md`).

## Fase B — Equipa de agentes: um por área, em paralelo

Lança um subagente **por área**, em **lotes de ~4 em simultâneo** (lança vários numa só
mensagem para correrem ao mesmo tempo; espera o lote acabar antes do próximo). Usa o tipo de
agente **`documentation-maintainer`** (tem ferramentas de escrita e já segue a disciplina de
actualização cirúrgica). Dá a cada um um prompt **auto-suficiente** com este molde:

```
Documenta a área "<area>" deste projecto Django+React. Escreve/actualiza UM ficheiro:
docs/features/<area>.md.

Lê o código da área para documentar com profundidade (não só docs antigos):
- Backend: backend/apps/<app>/ (models.py, views.py, serializers.py, signals.py, tasks.py, urls.py)
- Frontend: frontend/src/features/<feature>/ (páginas, componentes, chamadas à API)

Cruza com a documentação/estado que JÁ existe (referencia e linka, NÃO copies):
- docs/README.md (estado do sistema), docs/implementations/{done,structure,use_cases,todo}/<relevantes>
- backlog: MISSING_FRONTEND_ANALYSIS.md, docs/orphaned-menus-analysis.md, PLAN.md

Segue exactamente o template de "doc de funcionalidade" em
~/.claude/skills/document-progress/references/templates.md. O ficheiro tem de ter:
- cabeçalho `> Estado: ✅ feito | ⚠️ parcial | ⛔ por fazer | 🗓️ planeado  ·  Última actualização: <hoje>`
- `## Visão geral`, `## Funcionalidades` (o quê / como / ficheiros-chave / modelo·endpoints·UI),
  `## Modelo de dados`, `## Endpoints`, `## Por fazer` (gaps com link à fonte), `## Notas e limitações`.
Se o ficheiro já existir, actualiza cirurgicamente (não reescrevas tudo).

Devolve SÓ (como texto final, é o teu return value): o nome da área, o símbolo de estado
escolhido, e uma frase de resumo. Ex.: "projects | ✅ feito | gestão de projectos, membros e
despesas; API completa".
```

Passa a data de hoje no prompt (obtém-na uma vez com `date +%F`).

Quando precisares de máxima robustez/escala (audit muito grande, várias fases determinísticas),
podes em alternativa orquestrar isto com a ferramenta **Workflow** (um `pipeline`/`parallel`
com um estágio por área) — mas só se o utilizador tiver optado por workflows; por defeito, a
fan-out com o Agent tool acima é suficiente e mais simples.

## Fase C — Orquestrador: consolidar e publicar

1. **Recolher** os returns dos agentes (área | estado | resumo). Se algum agente falhou ou
   devolveu vazio, relança só essa área.
2. **Actualizar `docs/README.md`** (índice-mestre): garante que cada área tem uma linha que
   **liga** a `docs/features/<area>.md` com o símbolo de estado corrente. Sem ficheiro STATUS paralelo.
3. **Garantir `CLAUDE.md`** na raiz com a instrução de ler os docs antes de planear; se faltarem
   áreas novas nos atalhos, actualiza.
4. **Commit + push**: `git add docs/ CLAUDE.md` + commit (`docs: baseline/audit (<áreas>)`) +
   push na branch actual (commit e push sempre juntos; `-u origin <branch>` se não houver upstream).

## Template — cabeçalho de estado (topo de cada docs/features/<area>.md)
```markdown
# <Área> — Documentação

> Estado: ✅ feito | ⚠️ parcial | ⛔ por fazer | 🗓️ planeado  ·  Última actualização: YYYY-MM-DD
```
Escolhe **um** símbolo. `⚠️ parcial` quando há funcionalidades feitas mas também gaps relevantes.

## Regras
- **Não duplicar** o conteúdo das fontes existentes — referenciar e linkar.
- **Profundidade real**: os agentes lêem o código (modelos, views, serializers, componentes),
  não só os docs antigos.
- **Paralelizar de forma segura**: um agente por ficheiro de saída; o orquestrador só escreve
  no README/CLAUDE.md depois de os agentes acabarem.
- **Idempotente**: re-correr actualiza, não duplica áreas nem secções.
- Português de Portugal, directo. Não tocar em código nem em ficheiros fora de `docs/` e `CLAUDE.md`.
