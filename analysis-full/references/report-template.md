# Templates de output

Todos os ficheiros vão para `docs/audits/<data>/`. Português de Portugal.

---

## `<dimensão>.md` (um por dimensão)

```markdown
# Auditoria — <Dimensão>

> Data: YYYY-MM-DD · Âmbito: <todas as apps | app X> · Findings confirmados: N (🔴 c / 🟠 a / 🟡 m / ⚪ b)

Resumo de 1-2 frases do estado da dimensão.

## Findings

### 🔴 [SEC-001] Título específico do finding
- **Severidade:** crítico · **Confiança:** confirmado
- **Local:** `app_a/src/Controller/OrdersController.php:142`
- **Problema:** o que está errado, 1-3 frases.
- **Cenário de falha:** input concreto → resultado errado.
- **Correcção sugerida:** direcção do fix.
- **Verificação:** o que o céptico confirmou + evidência (`ficheiro:linha`).

### 🟠 [SEC-002] ...
(repetir, ordenado por severidade: 🔴 crítico → 🟠 alto → 🟡 médio → ⚪ baixo)

## Anexo — descartados na verificação
Findings que a ronda adversarial refutou (transparência do que foi testado):

- ~~[SEC-009] Título~~ — **refutado**: razão (ex.: CSRF middleware activo globalmente em `config/...`).
```

Emojis de severidade: 🔴 crítico · 🟠 alto · 🟡 médio · ⚪ baixo.

---

## `README.md` (sumário executivo do dossier)

```markdown
# Auditoria adversarial — YYYY-MM-DD

> Âmbito: <dimensões corridas> · Apps: <lista> · Gerado por skill `analysis-full`.
> ⚠️ Read-only: nenhum código foi alterado. Rever antes de commit/fix.

## Contagem por severidade × dimensão

| Dimensão | 🔴 | 🟠 | 🟡 | ⚪ | Total |
|----------|---:|---:|---:|---:|------:|
| data-integrity   | 0 | 0 | 0 | 0 | 0 |
| backend-models   | 0 | 0 | 0 | 0 | 0 |
| security         | 0 | 0 | 0 | 0 | 0 |
| config-deploy    | 0 | 0 | 0 | 0 | 0 |
| frontend-uiux    | 0 | 0 | 0 | 0 | 0 |
| performance      | 0 | 0 | 0 | 0 | 0 |
| **Total**        | 0 | 0 | 0 | 0 | 0 |

## Top críticos (actuar já)
1. [SEC-001] Título → `ficheiro:linha` — uma linha de impacto. Ver [security.md](./security.md).
2. ...

## 💡 Situações em que ninguém pensou
Os findings mais não-óbvios/surpreendentes — os que escaparam ao desenvolvimento:
- [DRIFT-003] Fix de X aplicado na app A mas não nas restantes → comportamento divergente.
- [PAY-002] Webhook reprocessa evento repetido → order duplicada em reenvio.

## Detalhe por dimensão
- [data-integrity](./data-integrity.md)
- [backend-models](./backend-models.md)
- [security](./security.md)
- [config-deploy](./config-deploy.md)
- [frontend-uiux](./frontend-uiux.md)
- [performance](./performance.md)

## Descartados
N findings refutados na verificação adversarial (ver anexos de cada dimensão).
```

Regras de preenchimento:
- Só dimensões corridas entram na tabela e na lista de detalhe.
- "Situações em que ninguém pensou" é a secção-estrela — destaca 3-6 findings genuinamente
  não-óbvios (drift silencioso, edge case, interacção entre componentes), não os óbvios.
- Números da tabela têm de bater com os findings confirmados nos ficheiros de dimensão.
