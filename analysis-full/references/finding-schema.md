# Schema de finding + rubricas

Cada perito devolve um **array JSON** de findings. Um finding tem esta forma:

```json
{
  "id": "SEC-001",
  "dimensao": "security",
  "severidade": "critico",
  "local": "app_a/src/Controller/OrdersController.php:142",
  "titulo": "IDOR: entry de outro utilizador editável via id na URL",
  "descricao": "editEntry() carrega a entry por $id sem verificar owner_id contra o utilizador autenticado.",
  "cenario_de_falha": "Utilizador A autenticado abre /entries/edit/999 (entry do Utilizador B). O controller carrega e permite gravar. → A edita/apaga submissão de B.",
  "correccao_sugerida": "Filtrar por owner_id = $this->Auth->user('id') na query, ou aplicar policy antes do save.",
  "confianca_inicial": "alta"
}
```

## Campos

- **id** — prefixo da dimensão + número sequencial: `DATA-`, `MODEL-`, `SEC-`, `CFG-`, `UI-`, `PERF-`, `DRIFT-`, `PAY-`.
- **dimensao** — id da dimensão (ver `dimensions.md`).
- **severidade** — `critico | alto | medio | baixo` (rubrica abaixo).
- **local** — `ficheiro:linha` (o mais preciso possível). Vários locais → lista o principal e menciona os restantes na descrição.
- **titulo** — uma linha, específico. Mau: "problema de segurança". Bom: "webhook Stripe processa evento repetido sem idempotência".
- **descricao** — o defeito em 1-3 frases. O que está errado no código, não uma aula.
- **cenario_de_falha** — **obrigatório para crítico/alto**. INPUT CONCRETO → RESULTADO ERRADO. É o que distingue um finding de uma suspeita. Sem isto, não reportar.
- **correccao_sugerida** — a direcção do fix (não é preciso o patch completo).
- **confianca_inicial** — `alta | media | baixa`. Honesto: baixa = "cheira mal mas não confirmei o caminho todo".

Após a Fase C, o orquestrador acrescenta:
- **veredicto** — `confirmado | parcial | refutado`.
- **verificacao** — justificação + evidência (`ficheiro:linha`) do céptico.

## Rubrica de severidade

- **crítico** — exploração remota sem auth, perda/corrupção de dados, cobrança errada, bypass total de autorização, credencial exposta em git público. Actuar já.
- **alto** — IDOR/escalada com auth, injecção mitigável por input específico, dados inconsistentes em caminho comum, race que perde escritas, N+1 que derruba página real.
- **médio** — validação em falta que só falha em edge case, header de segurança ausente, índice em falta em tabela média, inconsistência de UX que confunde mas não bloqueia.
- **baixo** — hygiene: código morto, tipo sub-óptimo, warning de acessibilidade menor, drift cosmético. Reportar só se barato e claro.

## Rubrica de confiança (pós-verificação)

- **confirmado** — o céptico reproduziu o caminho ou confirmou a ausência de mitigação. Entra no corpo.
- **parcial** — problema real mas com atenuantes (só sob certa config, ou impacto menor do que o título sugere). Entra no corpo, com a nuance.
- **refutado** — existe mitigação a montante que o perito falhou (CSRF middleware, query afinal parametrizada, constraint app-level, caminho morto). Vai para o anexo "descartados", com a razão.

## Regra anti-ruído (repetir aos agentes)

Default a **não reportar**. Preferível falhar um finding real do que encher o relatório de
falsos positivos — a verificação adversarial existe precisamente para isso. Nada de estilo,
preferências pessoais, ou "podia ser melhor" sem impacto demonstrável.
