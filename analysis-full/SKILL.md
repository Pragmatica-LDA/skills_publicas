---
name: analysis-full
description: >
  Auditoria adversarial de fundo ao projecto actual, lançando uma EQUIPA DE PERITOS em
  paralelo cujo objectivo é ENCONTRAR FALHAS, bugs e situações em que ninguém pensou durante
  o desenvolvimento — não documentar (isso é a document-codebase). Cobre integridade de dados
  (PKs/FKs/índices, schema drift), backend/modelos, segurança/cyber (SQLi/XSS/CSRF/IDOR/segredos),
  config/deploy, frontend/UI-UX e performance. Cada finding passa por verificação adversarial
  (agente céptico tenta refutar) antes de entrar no relatório, para baixo ruído. Output em
  docs/audits/YYYY-MM-DD/. Parametrizável por dimensão e por app/tenant. NÃO commita.
  Triggers: "analisa o projeto", "analysis full", "/analysis-full", "auditoria completa",
  "audita tudo", "encontra falhas", "encontra bugs", "análise de segurança", "code audit",
  "equipa de peritos", "audita o backend", "auditoria de integridade".
---

# analysis-full

**Objectivo: atacar o projecto, não descrevê-lo.** Uma equipa de peritos varre o código à
procura de defeitos — falhas de integridade de dados, bugs de backend, buracos de segurança,
problemas de config/deploy, UI/UX abaixo das boas práticas, e problemas de performance — com
ênfase nas **situações em que ninguém pensou** durante o desenvolvimento.

Diferença face à `document-codebase`/`document-progress`: essas **descrevem** o que existe;
esta **procura o que está errado**. Só reporta findings que **sobrevivem a uma ronda de
refutação** — o objectivo é um relatório de sinal alto, não uma lista de suspeitas.

O trabalho é feito por **fan-out de agentes** (padrão da `document-codebase`) + uma **ronda
adversarial**: os peritos encontram, um céptico tenta refutar, só os sobreviventes entram no
dossier. Como cada agente escreve um ficheiro de saída diferente, não há conflitos.

**Read-only sobre o código.** A skill NUNCA altera código; só escreve em `docs/audits/`.
E **não faz commit/push** — auditoria é sensível, o utilizador revê primeiro.

## Dimensões

Catálogo completo (checklist de ataque + comandos) em
`~/.claude/skills/analysis-full/references/dimensions.md`. Schema de finding e rubricas em
`references/finding-schema.md`. Templates de output em `references/report-template.md`.

| id | foco | default full |
|----|------|:---:|
| `data-integrity` | PKs/FKs/índices em falta, tipos inconsistentes, colunas órfãs, **drift de schema entre DBs**, integridade referencial não-imposta | ✅ |
| `backend-models` | associações erradas/em falta, validações/rules ausentes, N+1, lógica duplicada, transacções em falta | ✅ |
| `security` | SQLi, XSS, CSRF, auth/autorização (IDOR cross-app/cross-user), uploads, segredos, headers HTTP, sessões, webhooks | ✅ |
| `config-deploy` | app config vs default, credenciais em git, permissões, `.gitignore`, difs de config entre apps, deploy | ✅ |
| `frontend-uiux` | acessibilidade, responsividade, estados erro/loading, validação client vs server, boas práticas | ✅ |
| `performance` | queries lentas, falta de índices, N+1, processamento pesado (vídeo/uploads), caching | ✅ |
| `cross-app-drift` | divergências não-intencionais entre apps equivalentes (fix num sítio mas não noutro) | opt-in |
| `payments-checkout` | consistência de preços, idempotência de webhooks, races, orders órfãs, reconciliação | opt-in |

`cross-app-drift` e `payments-checkout` **não** entram no full por defeito; corre-as
explicitamente (`/analysis-full payments-checkout`). Se o projecto for pesado em multi-app ou
pagamentos, sugere ao utilizador incluí-las.

## Invocação (parametrizável)

- `/analysis-full` → corre as **6 dimensões por defeito**, todas as apps.
- `/analysis-full <dimensão> [<dimensão>...]` → só essas dimensões.
- `/analysis-full <dimensão> <app/tenant>` → só essa dimensão, restrita a essa app.
- `/analysis-full all` → as 8 dimensões (inclui as opt-in).

Sem args → conjunto por defeito, âmbito global.

## Fase A — Orquestrador: preparar e repartir (série, rápido)

1. **Data do dossier**: `date +%F` uma vez → `<data>`.
2. **Parsear args**: resolver dimensões pedidas (default = as 6 ✅) e app/tenant opcional.
3. **Auto-detectar layout** (a skill é global — não assumas o layout de nenhum projecto):
   - `ls` na raiz + procurar `composer.json`/`package.json`/`manage.py` para saber a stack.
   - Inventariar **unidades de aplicação**: monólito único (`src/`), multi-app (várias apps na
     raiz, ex. uma por tenant/cliente), backend+frontend separados, etc.
   - Localizar **schema/migrations** (ex.: `schemas/*.sql`, `config/schema/`, `*/Migrations/`,
     `db/migrate/`) — precisas deles para `data-integrity`.
   - Cruzar com docs existentes (`docs/README.md`, `docs/features/`, `docs/apps/`) como
     ponto de partida — sobretudo para `cross-app-drift` (matriz de divergências).
4. **Criar** `docs/audits/<data>/` (`mkdir -p`).

## Fase B — Equipa de peritos: um agente por (dimensão × âmbito), em paralelo

Lança em **lotes de ~4 em simultâneo** (vários numa só mensagem; espera o lote antes do
próximo). Tipo de agente: **`general-purpose`** (precisa de Bash/grep/read amplos — o `Explore`
não corre os comandos de schema/config necessários).

Reparte assim:
- Dimensões **globais/transversais** — `data-integrity` (drift de schema entre DBs), `config-deploy`,
  `cross-app-drift` — → **um agente único** que compara transversalmente as apps.
- Dimensões **por-app** — `backend-models`, `security`, `frontend-uiux`, `performance`,
  `payments-checkout` — → **um agente por app/tenant** quando há várias apps e o custo justifica;
  senão, um agente que varre todas as apps da dimensão.

Dá a cada agente um prompt **auto-suficiente** com este molde:

```
És um perito adversarial em <DIMENSÃO> a auditar <ÂMBITO: app X | todo o projecto>.
O teu objectivo é ENCONTRAR FALHAS, bugs e situações em que ninguém pensou — não documentar.

Lê o CHECKLIST DE ATAQUE da tua dimensão em
~/.claude/skills/analysis-full/references/dimensions.md (secção "<DIMENSÃO>") e segue-o.
Lê o código a fundo; corre os comandos grep/find/schema indicados. Ponteiros do projecto:
<caminhos de código relevantes> · <ficheiros de schema/config> · <docs a cruzar>.

Para CADA problema, devolve um finding no schema de
~/.claude/skills/analysis-full/references/finding-schema.md:
id, dimensão, severidade (crítico|alto|médio|baixo), ficheiro:linha, título, descrição,
cenário-de-falha (INPUT CONCRETO → RESULTADO ERRADO), correcção-sugerida, confiança-inicial.

REGRA ANTI-RUÍDO: default a NÃO reportar. Sem cenário de falha concreto e plausível → não é
finding. Não reportes estilo/preferências; só defeitos com impacto real.

Devolve SÓ (é o teu return value) um array JSON de findings. Array vazio se não encontraste nada real.
```

Passa a data de hoje no prompt. Para escala máxima (auditoria muito grande, muitas apps),
podes em alternativa orquestrar Fases B+C com a ferramenta **Workflow** (um `pipeline` com
estágio find → estágio verify por finding) — mas só se o utilizador optou por workflows; por
defeito, o fan-out acima chega.

## Fase C — Verificação adversarial (paralela, por finding)

Recolhe todos os findings dos peritos. Para cada finding de severidade **≥ média** (críticos e
altos **obrigatório**; médios por amostragem; baixos passam com etiqueta `não-verificado`),
lança **um agente céptico** cujo único objectivo é **REFUTAR**:

```
Tenta REFUTAR este finding. Lê o código real e procura a mitigação que o perito pode ter falhado:
<finding em JSON>.
Lentes conforme a dimensão: segurança → "há mitigação a montante?" (CSRF middleware activo?
query afinal parametrizada? autorização num beforeFilter/policy?); dados → "a constraint é
imposta noutro sítio, ao nível da app?"; performance → "isto está num caminho quente real?".
Default a REFUTADO se não confirmas com evidência.
Devolve SÓ: { veredicto: confirmado|refutado|parcial, justificação, evidência (ficheiro:linha) }.
```

Podes agrupar vários findings da mesma área num só céptico para poupar. Só findings
**confirmados/parciais** entram no corpo do relatório; os **refutados** vão para um anexo
"descartados na verificação" (transparência — mostra o que foi testado e caiu).

## Fase D — Orquestrador: consolidar e publicar

1. **Um ficheiro por dimensão**: `docs/audits/<data>/<dimensão>.md` — findings confirmados
   ordenados por severidade, seguindo `references/report-template.md`; anexo de descartados no fim.
2. **Sumário executivo**: `docs/audits/<data>/README.md` — matriz de contagem (severidade ×
   dimensão), top findings críticos, e uma secção **"Situações em que ninguém pensou"** em
   destaque (os findings mais surpreendentes/não-óbvios).
3. **NÃO commitar.** Termina com um resumo em texto ao utilizador (contagens + top críticos) e
   sugere rever o dossier antes de qualquer commit ou fix.
4. **Idempotente**: re-correr no mesmo dia sobrescreve `docs/audits/<data>/`; noutro dia cria
   novo dossier → auditorias comparáveis ao longo do tempo.

## Regras

- **Encontrar, não descrever.** Se um agente devolve prosa descritiva em vez de findings com
  cenário-de-falha, rejeita e relança.
- **Baixo ruído.** Só entra no relatório o que sobrevive à refutação. Anexo mostra o resto.
- **Read-only.** Nunca alterar código; escrever só em `docs/audits/`. **Sem commit/push.**
- **Paralelizar em segurança**: um agente por ficheiro de saída; o orquestrador só escreve
  depois de os agentes acabarem.
- **Cenário concreto obrigatório** em cada finding crítico/alto: input real → resultado errado.
- Português de Portugal, directo.
