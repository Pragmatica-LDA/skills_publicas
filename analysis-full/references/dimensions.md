# Catálogo de dimensões — checklists de ataque

Cada secção é o briefing de ataque de uma dimensão. O agente lê a **sua** secção, corre os
comandos, lê o código, e devolve findings (schema em `finding-schema.md`). Os comandos de exemplo assumem
stack PHP/CakePHP multi-app (adapta caminhos ao layout detectado na Fase A: monólito `src/`,
multi-app na raiz, ou backend/frontend separados).

Regra transversal: **default a não reportar**. Cada finding precisa de cenário de falha concreto.

---

## data-integrity

Foco: o **schema real** (SQL/migrations), não só os modelos. Em CakePHP a integridade
referencial vive muitas vezes na app, não na DB — procura FKs que existem em lógica mas não em
constraint.

Ataca:
- **PKs em falta ou fracas**: tabelas sem PRIMARY KEY; PKs não-`AUTO_INCREMENT` onde deviam;
  chaves compostas suspeitas.
- **FKs não-impostas**: colunas `*_id` sem `FOREIGN KEY` na DB → órfãos possíveis. Confirma se a
  app impõe (belongsTo + rules) ou se fica ao acaso.
- **Índices em falta**: colunas usadas em `WHERE`/`JOIN`/`ORDER BY` (sobretudo `*_id`, `email`,
  `slug`, `status`) sem índice → full scan.
- **Tipos inconsistentes**: mesma entidade lógica com tipos diferentes entre tabelas/DBs (ex.:
  `order_id INT` numa, `BIGINT` noutra; `price DECIMAL` vs `FLOAT` — perigoso em dinheiro);
  `VARCHAR` curto demais; datas como `VARCHAR`.
- **Colunas órfãs / mortas**: colunas no schema sem uso no código, ou referidas no código mas
  ausentes do schema.
- **DRIFT DE SCHEMA entre DBs** (crítico em multi-DB): comparar a mesma tabela entre as várias
  DBs — coluna que existe numa e falta noutra, tipo divergente, índice só nalgumas. Diverge por
  esquecimento, não por design.
- **NULLability e defaults**: colunas que deviam ser `NOT NULL` mas aceitam NULL; defaults
  ausentes que forçam a app a adivinhar.
- **charset/collation** inconsistentes entre tabelas → bugs de comparação/ordenação.

Comandos úteis:
```
ls schemas/full_schema/ schemas/migrations/ 2>/dev/null
grep -riE 'CREATE TABLE|PRIMARY KEY|FOREIGN KEY|KEY |UNIQUE' schemas/full_schema/*.sql
# drift: extrair colunas de uma tabela em cada dump e comparar
for f in schemas/full_schema/*.sql; do echo "== $f"; grep -iA40 'CREATE TABLE `orders`' "$f"; done
grep -rn 'belongsTo\|hasMany\|hasOne\|belongsToMany' */src/Model/Table/ 2>/dev/null   # FKs em lógica
```

---

## backend-models

Foco: modelos, controllers, serviços. Correcção e robustez da lógica.

Ataca:
- **Associações erradas/em falta**: `belongsTo`/`hasMany` com `foreignKey` errada, ou associação
  usada em queries (`contain`) mas não declarada.
- **Validação/rules ausentes**: `validationDefault()` sem regras para campos obrigatórios;
  `buildRules()` sem `isUnique`/`existsIn` onde a integridade depende disso. Uniqueness só em
  validação (race-prone) sem índice UNIQUE na DB.
- **Transacções em falta**: sequências de writes que deviam ser atómicas (criar order + itens +
  pagamento) sem `Connection::transactional()`/`getConnection()->begin()`. Escrita parcial em
  falha → estado inconsistente.
- **N+1**: loops que fazem query por iteração; `find()` dentro de `foreach`; ausência de
  `contain()` onde há acesso a associações num loop.
- **Lógica de negócio duplicada**: mesma regra copiada em vários controllers/apps → diverge.
- **Mass-assignment**: `newEntity($data)` / `patchEntity` sem `_accessible`/`fields` restritos →
  campos sensíveis (`role`, `owner_id`, `is_admin`, `price`) settáveis via request.
- **Erros engolidos**: `try/catch` que apanha e ignora; `save()` cujo `false` não é tratado.
- **Datas/timezones**: mistura de UTC/local; comparações de datas como string.

Comandos úteis:
```
grep -rn 'patchEntity\|newEntity' */src/Controller/ | head -50
grep -rn '_accessible' */src/Model/Entity/
grep -rn 'transactional\|->begin()\|->commit()' */src/
grep -rn 'foreach' */src/ | grep -i 'find\|->get('   # candidatos a N+1
grep -rnL 'buildRules' */src/Model/Table/*.php 2>/dev/null   # tables sem rules
```

---

## security

Foco: exploração. Pensa como atacante autenticado E anónimo.

Ataca:
- **SQLi**: interpolação de input em queries — `query("... $x")`, `where("col = $v")` string,
  `->find('all', ['conditions' => "name = '$name'"])`. Confirma que não é bind/array condition.
- **XSS**: output sem escape — `echo $x` em templates, `->raw()`, `{{{ }}}`, `h()` em falta,
  dados de user em atributos HTML/JS inline.
- **CSRF**: forms POST sem token; confirma se `CsrfProtectionMiddleware`/`SecurityComponent` está
  activo globalmente ou se há rotas fora dele (webhooks, APIs).
- **Auth/autorização (IDOR)**: recursos carregados por `$id` da URL sem verificar owner/tenant.
  Em multi-app: acesso cross-tenant (utilizador de um tenant vê dados de outro). Acções
  admin sem check de role. Endpoints sensíveis fora do `beforeFilter`/allow list.
- **Uploads**: sem validação de extensão/MIME/tamanho; nome de ficheiro do user usado no path
  (path traversal); ficheiros graváveis dentro de webroot e executáveis.
- **Segredos hardcoded**: API keys, passwords DB, chaves Stripe, tokens SMTP em código ou em
  ficheiros versionados.
- **Headers HTTP**: ausência de HSTS, CSP, X-Frame-Options, X-Content-Type-Options; cookies sem
  `Secure`/`HttpOnly`/`SameSite`.
- **Sessões**: fixation (sem `renew()` no login); timeout ausente; sessão partilhada entre apps.
- **Webhooks/Stripe**: assinatura não verificada; endpoint aceita eventos sem validar origem.
- **Password/crypto**: hashing fraco (md5/sha1); reset tokens previsíveis; comparação não
  constant-time.

Comandos úteis:
```
grep -rnE "->query\(|conditions.*=>.*\\\$|where\(['\"].*\\\$" */src/ | head -50   # SQLi candidatos
grep -rn '->raw(\|{{{' */src/Template/ ; grep -rnL 'h(' # XSS candidatos (afinar)
grep -rniE 'sk_live|sk_test|password.*=.*['\''\"]|api_key|secret.*=.*['\''\"]' --include=*.php .
grep -rn 'CsrfProtection\|SecurityComponent\|Authentication\|Authorization' */src/
grep -rn 'move_uploaded_file\|->file\|UploadedFile' */src/
git log --all --full-history -- '*app.php' '*.env' 2>/dev/null | head   # segredos no histórico
```

---

## config-deploy

Foco: configuração, segredos, deploy. Diferenças entre apps e entre config real vs default.

Ataca:
- **Segredos versionados**: `config/app.php` (ou equivalente) com credenciais reais commitado;
  confirmar `.gitignore` cobre `app.php`/`.env`/dumps. `git ls-files` mostra o que está mesmo no repo.
- **debug ligado em prod**: `'debug' => true`, `Configure::write('debug', ...)`, DebugKit activo.
- **Config divergente entre apps**: cada app tem a sua `app.php` — comparar chaves de segurança
  (`Security.salt`), config de email, de sessão. Salt igual/fraco reutilizado; salt default do skeleton.
- **Permissões**: `tmp/`, `logs/`, uploads com permissões erradas; ficheiros graváveis a mais.
- **.gitignore lacunas**: dumps de DB, `debug_kit.sqlite`, `logs/` versionados.
- **Deploy**: diferenças entre config de deploy (Docker/Git releases) e o que a app espera;
  paths hardcoded; falta de separação build/runtime; segredos passados em plaintext.
- **Dependências**: `composer.lock`/`package-lock` ausente ou dessincronizado; versões com CVEs
  conhecidos (runtime/framework EOL — assinalar libs sem suporte).

Comandos úteis:
```
git ls-files | grep -iE 'app\.php$|\.env|dump|\.sql$|\.sqlite$'
grep -rn "'debug' => true\|Configure::write('debug'" */config/ config/ 2>/dev/null
grep -rn "'salt'" */config/*.php config/*.php | sort   # salts iguais/default?
cat .gitignore
for d in */config/app.php; do echo "== $d"; grep -iE 'salt|password|host|debug' "$d"; done
```

---

## frontend-uiux

Foco: templates, JS, CSS. Acessibilidade, robustez de UX, validação.

Ataca:
- **Validação só client-side**: forms validados em JS mas sem validação equivalente no servidor
  (cruza com backend-models) → bypass trivial.
- **Estados em falta**: sem estado de loading/erro/vazio; submits sem feedback → duplo-submit
  (cruza com races de pagamento).
- **Acessibilidade**: imagens sem `alt`, forms sem `label`/`for`, contraste insuficiente, foco de
  teclado partido, `<div>` clicável sem role, headings fora de ordem.
- **Responsividade**: layouts fixos que partem em mobile; tabelas sem scroll; viewport meta ausente.
- **jQuery/Bootstrap**: handlers duplicados; listeners não removidos; XSS via `.html()` com dados
  de user; versões desactualizadas com CVEs.
- **Consistência**: mensagens de erro inconsistentes entre apps; i18n em falta (strings hardcoded
  onde há multi-idioma).
- **Double-submit / idempotência de UI**: botão de pagamento sem disable no submit.

Comandos úteis:
```
grep -rn '\.html(' */webroot/js */src/Template/ | grep -i 'val\|data\|response'   # XSS via jQuery
grep -rLn 'alt=' # imgs sem alt (afinar por template)
grep -rn 'required' */src/Template/ | head   # validação client — cruzar com server
grep -rniE 'bootstrap|jquery' */webroot/ | grep -iE 'v[0-9]' | head   # versões
```

---

## performance

Foco: caminhos quentes reais. Não micro-optimizações.

Ataca:
- **N+1** (cruza com backend-models): queries em loop em páginas de listagem.
- **Índices em falta** (cruza com data-integrity) em colunas de filtro/ordenação de páginas muito usadas.
- **Queries pesadas**: `SELECT *` em tabelas largas; ausência de paginação; `contain` que puxa tudo.
- **Processamento pesado síncrono**: transcode de vídeo (ex.: FFmpeg), geração de PDF, envio
  de email a correr no request em vez de fila/shell → timeout/memória.
- **Memória em uploads/dumps**: ler ficheiro inteiro para memória; sem streaming.
- **Caching ausente**: dados estáticos (categorias, config de tenant) sem cache; queries
  repetidas por request.
- **Assets**: sem minificação/concat; imagens não optimizadas.

Comandos úteis:
```
grep -rn 'SELECT \*\|->find(' */src/Controller/ | grep -i 'index\|list' | head
grep -rn 'exec\|shell_exec\|proc_open\|ffmpeg' */src/
grep -rn 'file_get_contents\|fread\|->read()' */src/ | grep -i 'upload\|dump\|video'
grep -rn 'Cache::\|->cache(' */src/ | head   # onde HÁ cache — ausência nos hot paths é o problema
```

---

## cross-app-drift (opt-in)

Foco: divergências **não-intencionais** entre apps que deviam ser equivalentes. Só faz sentido
em projectos multi-app com código paralelo (uma app por tenant/cliente).

Ataca:
- **Fix aplicado num sítio, não nos outros**: mesma função/controller com correcção numa app e a
  versão bugada nas restantes. Diff entre apps do mesmo ficheiro lógico.
- **Lógica copy-paste dessincronizada**: cálculo de preço, regras de validação, permissões que
  divergiram silenciosamente.
- **Schema drift** (cruza com data-integrity): tabela equivalente diferente entre DBs.
- **Config drift** (cruza com config-deploy): comportamento diferente por config divergente não-intencional.

Ponto de partida: `docs/apps/README.md` (matriz de divergências documentada) — o que
**não** está lá listado como divergência intencional e diverge, é suspeito.

Comandos úteis:
```
# comparar o mesmo ficheiro lógico entre apps
diff <(cat app_a/src/Controller/CheckoutController.php) <(cat app_b/src/Controller/CheckoutController.php)
for a in app_a app_b app_c; do \
  echo "== $a"; md5 "$a/src/Controller/OrdersController.php" 2>/dev/null; done   # hashes divergentes
```

---

## payments-checkout (opt-in)

Foco: dinheiro. Onde um bug custa/perde dinheiro ou corrompe orders. Área tipicamente ⚠️.

Ataca:
- **Consistência de preços**: preço calculado em sítios diferentes (client, checkout, webhook) que
  podem divergir; preço confiado do request em vez de recalculado no servidor.
- **Idempotência de webhooks**: evento Stripe processado sem deduplicação → cobrança/order dupla
  ao reenviar. Confirma tabela de eventos processados / chave de idempotência.
- **Race conditions**: duplo-submit cria duas orders; webhook chega antes do redirect e ambos criam
  estado; sem lock/unique constraint.
- **Orders órfãs**: pagamento sem order, order sem pagamento, order presa em `pending` sem
  reconciliação. Gate de criação que falha silenciosamente.
- **Reconciliação**: sem processo que cruze Stripe ↔ DB; estados divergentes não detectados.
- **Transacções** (cruza com backend-models): criar order+itens+pagamento não-atómico.
- **Segurança** (cruza com security): assinatura de webhook não verificada; valores manipuláveis.

Comandos úteis:
```
grep -rn 'stripe\|Stripe\|webhook\|payment_intent\|charge' */src/ | grep -i 'controller\|service' | head
grep -rn 'idempot\|event_id\|->exists(\|processed' */src/ | grep -i 'payment\|webhook\|stripe'
grep -rn 'transactional\|->begin()' */src/ | grep -i 'order\|payment\|checkout'
ls schemas/migrations/ | grep -iE 'payment|checkout|order'
```
