# Skills públicas — Pragmática

Skills para [Claude Code](https://claude.com/claude-code), em Português de Portugal.

| Skill | O que faz |
|-------|-----------|
| [`document-codebase`](document-codebase/SKILL.md) | Documenta de raiz o estado do codebase com uma equipa de agentes em paralelo (um por área funcional). Gera `docs/features/<area>.md` com estado feito/parcial/por fazer. |
| [`document-progress`](document-progress/SKILL.md) | Regista o progresso de uma sessão de desenvolvimento: entrada datada no `docs/ROADMAP.md` + docs por funcionalidade em `docs/features/`. |
| [`analysis-full`](analysis-full/SKILL.md) | Auditoria adversarial ao projecto: uma equipa de peritos procura falhas (integridade de dados, backend, segurança, config/deploy, UI/UX, performance) e cada finding passa por verificação céptica antes de entrar no relatório em `docs/audits/YYYY-MM-DD/`. |

## Instalação

Copiar a pasta da skill para `~/.claude/skills/` (global) ou `.claude/skills/` (projecto):

```bash
git clone https://github.com/Pragmatica-LDA/skills_publicas.git
cp -R skills_publicas/analysis-full ~/.claude/skills/
```

Depois invocar com `/analysis-full`, `/document-codebase` ou `/document-progress`, ou pedir em linguagem natural (ver os triggers na `description` de cada `SKILL.md`).

## Notas

- `document-codebase` usa os templates de `document-progress/references/templates.md` — instalar as duas juntas.
- `document-codebase` assume por defeito um layout Django + React (`backend/apps`, `frontend/src/features`); adaptar ao projecto.
- `document-progress` menciona um agent opcional `documentation-maintainer` (não incluído).
- Os comandos de exemplo da `analysis-full` assumem PHP/CakePHP multi-app; a skill auto-detecta o layout e adapta.
