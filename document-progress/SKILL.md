---
name: document-progress
description: >
  Documenta o progresso de uma sessão de desenvolvimento no projecto actual:
  actualiza um ROADMAP/timeline resumido (entrada datada) e cria/actualiza
  documentação detalhada por funcionalidade em docs/features/. Usar quando o
  utilizador quer registar o que foi feito, actualizar a documentação ou manter
  o roadmap. Triggers: "documenta o que fizemos", "documentar sessão",
  "documentar a sessão", "atualiza a documentação", "actualiza a documentação",
  "regista no roadmap", "documenta o progresso", "document progress",
  "/document-progress".
---

# document-progress

Regista o trabalho de uma sessão em duas camadas:

1. **`docs/ROADMAP.md`** — timeline resumida: uma entrada datada por sessão, com
   uma descrição genérica do que foi feito + links para os docs detalhados +
   hashes dos commits.
2. **`docs/features/<area>.md`** — documentação completa por área funcional
   (tasks, emails, calendar-blocks, calendar, …). Criar se não existir,
   actualizar cirurgicamente se já existir.

Os templates e o mapa de áreas estão em `references/templates.md` — **lê esse
ficheiro antes de escrever** para manter a estrutura consistente.

## Workflow

### 1. Determinar o âmbito (o que documentar)
- Lê `docs/ROADMAP.md` (na raiz do `docs/` do projecto actual). Se existir, encontra
  a data da entrada mais recente (`## YYYY-MM-DD …`).
- Lista o que mudou desde aí:
  ```bash
  git log --since="<data-da-última-entrada>" --oneline
  # se não houver ROADMAP, usa os commits recentes:
  git log --oneline -20
  git diff --stat <ref-anterior>..HEAD
  ```
- Cruza os ficheiros/commits com a **conversa actual** para extrair o racional e as
  decisões (o git diz *o quê*; a conversa diz *porquê*).
- Se o intervalo for ambíguo (sem ROADMAP e muitos commits antigos), **pergunta ao
  utilizador** que commits/datas cobrir antes de avançar.

### 2. Agrupar por área funcional
Mapeia cada ficheiro alterado para uma área usando o mapa em
`references/templates.md` (secção "Mapa de áreas"). Áreas conhecidas neste projecto:
`tasks`, `emails`, `calendar-blocks`, `calendar`. Se algo cair fora do mapa, deriva
o nome da área a partir da app backend (`backend/apps/<x>/`) ou da feature frontend
(`frontend/src/features/<x>/`) e regista-a.

### 3. Criar/actualizar os docs por área
Para cada área tocada, abre `docs/features/<area>.md`:
- **Não existe** → cria a partir do template de funcionalidade (ver references).
- **Existe** → **actualização cirúrgica**: adiciona ou edita apenas a subsecção da
  funcionalidade afectada; não dupliques secções; mantém o resto intacto e coerente.
  Segue a mesma disciplina do agent `~/.claude/agents/documentation-maintainer.md`
  (adicionar o que é novo, actualizar o que mudou, remover o que deixou de ser
  verdade — nunca reescrever tudo do zero).
- Actualiza sempre o cabeçalho `> Estado: … · Última actualização: YYYY-MM-DD` no topo do doc
  (revê o símbolo de estado ✅/⚠️/⛔/🗓️ e mexe na secção `## Por fazer` se a mudança fechou/abriu gaps).
- Usa a data de hoje (do contexto `currentDate`/`date +%F`), em Português de Portugal.
- Se `docs/README.md` (índice-mestre) tiver uma linha de estado para esta área, mantém-na em
  sincronia (link para o doc + símbolo de estado).

### 4. Actualizar o ROADMAP
Acrescenta uma entrada **no topo** (mais recente primeiro) de `docs/ROADMAP.md`,
seguindo o template. A descrição é **genérica e curta** (1–3 linhas) — o detalhe
vive nos docs por área. Inclui os links para as áreas tocadas e os hashes dos commits.
Se `docs/ROADMAP.md` não existir, cria-o com um cabeçalho e a primeira entrada.

### 5. Commit + push
No fim, versiona só a documentação:
```bash
git add docs/
git commit -m "docs: roadmap + features (<áreas>)"
git push
```
(Preferência do utilizador: commit e push sempre juntos.) Se a branch actual não
tiver upstream, faz `git push -u origin <branch>`.

## Regras
- **Idempotência**: só cria nova entrada de ROADMAP quando há commits novos desde a
  última entrada. Não dupliques entradas nem subsecções.
- **Português de Portugal**, directo e conciso. Termos técnicos em inglês quando
  apropriado.
- Documenta a **funcionalidade** (o que faz, como, ficheiros-chave, modelo/endpoints/UI),
  não um diário de cada commit.
- Não toques em código nem em docs fora de `docs/`.
