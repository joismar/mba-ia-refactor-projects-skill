# Catálogo de Anti-Patterns

Referência da **Fase 2**. Para cada achado: procurar os sinais, confirmar lendo o
arquivo, registrar `arquivo:linha` e classificar pela escala abaixo. Os exemplos
marcados com *calibração* vêm dos 3 projetos do desafio e servem para aferir o
nível de severidade — o catálogo em si é genérico.

## Escala de severidade

- **CRITICAL** — falha grave de arquitetura ou segurança: funciona errado, expõe
  dados sensíveis (credenciais, SQL Injection, execução arbitrária) ou viola
  completamente a separação de responsabilidades (God Class com banco + regra +
  roteamento no mesmo arquivo).
- **HIGH** — violação forte de MVC/SOLID que dificulta muito manter e testar:
  regra de negócio pesada dentro de controller/rota, acoplamento sem injeção de
  dependência, estado global mutável em toda a aplicação.
- **MEDIUM** — padronização, duplicação ou gargalo moderado: queries N+1,
  middleware mal configurado, validação ausente nas rotas.
- **LOW** — legibilidade: nomenclatura ruim, magic numbers, concatenação de string.

> A severidade é decidida pela **definição**, não pelo susto. Na dúvida entre dois
> níveis, escolha o menor e justifique — inflar o relatório destrói a confiança nele.

---

## Índice

| ID | Anti-pattern | Severidade | Sinal rápido |
|---|---|---|---|
| AP-01 | Segredos hardcoded | CRITICAL | `SECRET_KEY =`, senha/chave literal |
| AP-02 | Superfície administrativa exposta / execução arbitrária | CRITICAL | rota `/admin/*` sem auth, `execute(query)` do request |
| AP-03 | SQL Injection por concatenação | CRITICAL | `execute("..." + var)` |
| AP-04 | God Class / arquivo onisciente | CRITICAL | mesmo arquivo com HTTP + regra + SQL |
| AP-05 | Regra de negócio em rota/controller | HIGH | validação + query + decisão no mesmo handler |
| AP-06 | Senha com hash fraco e/ou exposta | HIGH | `md5`/`sha1` para senha; `to_dict()` com `password` |
| AP-07 | Autenticação fake / token não verificado | HIGH | token montado por concatenação; nenhuma rota valida |
| AP-08 | Estado global mutável / acoplamento sem DI | HIGH | variável de módulo mutada; singleton importado direto |
| AP-09 | N+1 / consulta dentro de loop | MEDIUM | query dentro de `for` |
| AP-10 | Ausência de paginação | MEDIUM | `SELECT *`/`query.all()` sem `limit` |
| AP-11 | Middleware sem restrição (CORS, debug) | MEDIUM | `CORS(app)`; `debug=True` |
| AP-12 | Validação de entrada ausente ou duplicada | MEDIUM | mesma checagem reescrita em N rotas |
| AP-13 | Operação de escrita sem transação | MEDIUM | N `delete()` em loop + commit avulso |
| AP-14 | Resposta sem envelope padronizado | LOW | cada rota devolve um formato diferente |
| AP-15 | Nomenclatura ruim / magic numbers | LOW | `p`, `x`, `d1`, `12345` no código |
| AP-16 | Concatenação de string em vez de interpolação | LOW | `"a" + var + "b"` |
| AP-17 | Log por `print`/`console.log` | LOW | sem logger, sem nível, sem correlação |

---

## Detalhamento

### AP-01 · Segredos hardcoded — CRITICAL
- **Sinais:** `SECRET_KEY`, `API_KEY`, `password =`, `token =` com valor literal;
  string longa aleatória em arquivo versionado; chave de gateway de pagamento.
- **Por que importa:** o segredo está no histórico do Git para sempre; rotacionar
  passa a ser obrigatório. Impede configuração por ambiente.
- **Correção (MVC):** mover para variáveis de ambiente, lidas num módulo de
  configuração (`config.py` / `config.js`), com validação de presença no boot.
- *Calibração:* `code-smells-project/app.py:7`, `task-manager-api/app.py:13`.

### AP-02 · Superfície administrativa exposta / execução arbitrária — CRITICAL
- **Sinais:** rotas `/admin/*` sem autenticação; endpoint que executa SQL, comando
  de shell ou expressão recebida no corpo da requisição; `eval`, `exec`, `os.system`,
  `child_process`, `shell=True`.
- **Por que importa:** é execução arbitrária remota — qualquer cliente lê, altera ou
  apaga o banco inteiro. Não existe "só em dev" quando a rota sobe junto.
- **Correção (MVC):** remover a rota. Se for ferramenta de suporte, mover para CLI
  separada, fora do processo web, e nunca aceitar SQL do cliente.
- *Calibração:* `code-smells-project/app.py:59` (`POST /admin/query` executa o SQL
  recebido) e `app.py:47` (`POST /admin/reset-db` apaga todas as tabelas), ambos sem auth.

### AP-03 · SQL Injection por concatenação — CRITICAL
- **Sinais:** `execute("... " + var)`, `f"...{var}"` dentro de SQL, `%` em query,
  template string em `db.query`.
- **Por que importa:** o parâmetro vira código SQL. Um `' OR '1'='1` derruba a
  autenticação; `; DROP TABLE` apaga dados.
- **Correção (MVC):** parâmetros vinculados (`?`/named params) em todas as queries;
  idealmente substituir SQL cru por ORM na camada de repositório.
- *Calibração:* `code-smells-project/models.py:28`, `:68`, `:92`, `:110`.

### AP-04 · God Class / arquivo onisciente — CRITICAL
- **Sinais:** um arquivo/classe que registra rotas, decide regra de negócio, monta
  SQL e formata resposta. Detectores de apoio: contagem alta de imports mistos
  (framework + driver de banco + domínio), métodos com nomes de domínios diferentes.
- **Por que importa:** impossível testar em isolamento; qualquer mudança afeta tudo.
- **Correção (MVC):** quebrar em models, controllers e camada de acesso a dados,
  uma unidade por domínio.
- *Calibração:* `ecommerce-api-legacy/src/AppManager.js` (141 linhas com HTTP, cache,
  pagamento e SQL no mesmo arquivo).

### AP-05 · Regra de negócio em rota/controller — HIGH
- **Sinais:** handler HTTP com `if` de regra de negócio, cálculo, decisão de status
  de domínio ou acesso direto ao banco.
- **Por que importa:** a regra fica presa no transporte; não há reuso entre HTTP e
  job/CLI, e testar exige subir o servidor.
- **Correção (MVC):** extrair para camada de serviço; o controller passa a só
  traduzir HTTP ↔ domínio.
- *Calibração:* `task-manager-api/routes/task_routes.py`, `user_routes.py`, `report_routes.py`.

### AP-06 · Senha com hash fraco e/ou exposta — HIGH
- **Sinais:** `md5`, `sha1`, hash sem salt; campo de senha presente no dicionário de
  serialização retornado pela API.
- **Por que importa:** MD5/SHA1 quebram por força bruta e rainbow table; expor o hash
  vaza credencial reutilizável em outros sistemas.
- **Correção (MVC):** `bcrypt`/`argon2` com salt; serializador sem o campo `password`.
- *Calibração:* `task-manager-api/models/user.py:29`.

### AP-07 · Autenticação fake / token não verificado — HIGH
- **Sinais:** token concatenado (`'token-' + id`), sem assinatura nem expiração; nenhum
  decorator/middleware checando credencial nas rotas; login que devolve string fixa.
- **Por que importa:** não há autenticação real — qualquer cliente acessa qualquer
  recurso, e a existência do "token" cria falsa sensação de segurança.
- **Correção (MVC):** JWT assinado com expiração + middleware/decorator de autorização
  aplicado em todas as rotas sensíveis.
- *Calibração:* `task-manager-api/routes/user_routes.py:210`.

### AP-08 · Estado global mutável / acoplamento sem DI — HIGH
- **Sinais:** variáveis de módulo mutadas em runtime; configuração "hardcoded" no topo
  do arquivo; módulo importado direto pelo consumidor em vez de injetado.
- **Por que importa:** o estado escapa entre requisições, impossibilita teste isolado
  e cria dependência oculta.
- **Correção (MVC):** injeção de dependência e configuração por ambiente.
- *Calibração:* `ecommerce-api-legacy/src/utils.js`.

### AP-09 · N+1 / consulta dentro de loop — MEDIUM
- **Sinais:** `query`/`SELECT` dentro de `for`/`map`; acesso a relação lazy dentro de
  loop de serialização; `len(obj.relacao)` por item.
- **Por que importa:** o custo cresce com o volume — 1 requisição vira N+1 no banco.
- **Correção (MVC):** eager loading (`joinedload`/`include`/`JOIN`), agregação em uma
  consulta, `include` do ORM na camada de repositório.
- *Calibração:* `task-manager-api/routes/task_routes.py:42,51`.

### AP-10 · Ausência de paginação — MEDIUM
- **Sinais:** listagem com `query.all()` / `SELECT *` sem `limit`/`offset`/cursor.
- **Por que importa:** a resposta degrada conforme a base cresce até derrubar o processo.
- **Correção (MVC):** paginação por `limit`/`offset` com defaults e teto no repositório.
- *Calibração:* `task-manager-api/routes/user_routes.py:12`; `code-smells-project/models.py:7`.

### AP-11 · Middleware sem restrição (CORS, debug) — MEDIUM
- **Sinais:** `CORS(app)` sem `origins`; `DEBUG = True`; `app.run(debug=True)` com
  `host='0.0.0.0'`; `Access-Control-Allow-Origin: *`.
- **Por que importa:** qualquer origem consome a API em nome do usuário; debug exposto
  em rede entrega console interativo e stack traces.
- **Correção (MVC):** origens explícitas por ambiente; debug desligado fora de dev.
- *Calibração:* `task-manager-api/app.py:15` e `:34`; `code-smells-project/app.py:9` e `:88`.

### AP-12 · Validação de entrada ausente ou duplicada — MEDIUM
- **Sinais:** o mesmo `if not campo` repetido em N handlers; ausência de validação de
  tipo/faixa; validação de e-mail própria em vez de schema.
- **Por que importa:** cada rota pode divergir; corrigir uma regra exige editar todas.
- **Correção (MVC):** schema de validação (Pydantic/Marshmallow/Zod/Joi) por recurso,
  na fronteira da aplicação.
- *Calibração:* `task-manager-api/routes/user_routes.py:54` (repetido em `:102`).

### AP-13 · Operação de escrita sem transação — MEDIUM
- **Sinais:** N `delete()`/`insert()` em loop seguido de commit único; efeito colateral
  externo (e-mail, cache) antes do commit; nenhum `rollback` no caminho de erro.
- **Por que importa:** falha no meio deixa dados inconsistentes; o banco fica "sujo"
  sem forma de reconstruir o estado.
- **Correção (MVC):** transação explícita delimitando o caso de uso, efeitos externos
  após o commit.
- *Calibração:* `task-manager-api/routes/user_routes.py:140` (deleta tarefas em loop
  antes de remover o usuário).

### AP-14 · Resposta sem envelope padronizado — LOW
- **Sinais:** cada rota devolve chaves diferentes (`erro`, `error`, `mensagem`,
  `message`); status HTTP implícito; às vezes lista crua, às vezes objeto.
- **Por que importa:** o cliente precisa de tratamento ad-hoc por endpoint.
- **Correção (MVC):** envelope único (`{success, data, error}`) e códigos HTTP corretos.
- *Calibração:* `ecommerce-api-legacy/src/AppManager.js:135` (`res.send` com texto solto).

### AP-15 · Nomenclatura ruim / magic numbers — LOW
- **Sinais:** `p`, `x`, `tmp`, `d1`; números soltos com significado de negócio.
- **Por que importa:** legibilidade e risco de alteração equivocada.
- **Correção (MVC):** nomes descritivos e constantes nomeadas.

### AP-16 · Concatenação de string em vez de interpolação — LOW
- **Sinais:** `"a" + var + "b"` em mensagens/logs.
- **Por que importa:** menos legível e propenso a erro de tipo.
- **Correção (MVC):** f-string (Python) ou template literal (JS).
- *Calibração:* `code-smells-project/models.py` (montagem de mensagens).

### AP-17 · Log por `print`/`console.log` — LOW
- **Sinais:** `print(...)` / `console.log(...)` como única observabilidade.
- **Por que importa:** sem nível, sem contexto, sem destino configurável.
- **Correção (MVC):** logger com níveis e configuração por ambiente.
- *Calibração:* `task-manager-api/routes/user_routes.py:83`.

---

## APIs deprecated / obsoletas

Exigência explícita do desafio: a auditoria precisa detectar **uso de API obsoleta e
recomendar o equivalente moderno**. Procurar por estes padrões e, ao achar, classificar
em **MEDIUM** (ou **HIGH** se for de segurança, como hash de senha).

| Sinal no código | Por que é obsoleto | Equivalente moderno |
|---|---|---|
| `datetime.utcnow()` | Deprecado no Python 3.12+; retorna datetime *naive* | `datetime.now(timezone.utc)` |
| `Model.query.get(id)` / `.query.get()` | API legada do SQLAlchemy 1.x | `db.session.get(Model, id)` |
| `hashlib.md5()` / `sha1()` para senha | Algoritmo quebrado para credencial | `bcrypt` / `argon2` |
| `@app.before_first_request` | Removido no Flask 2.3 | inicialização no boot da app |
| `flask_cors.CORS(app)` sem escopo | Configuração padrão permissiva | `CORS(app, resources={...})` |
| `print()` no caminho de requisição | Sem nível/destino; vaza dado em stdout | `logging` com `getLogger` |
| `new Buffer(...)` (Node) | Deprecado desde Node 6 | `Buffer.from(...)` |
| `url.parse()` (Node) | Legado e sem validação | `new URL(...)` |
| `body-parser` (Node) | Embutido no Express ≥4.16 | `express.json()` |
| `require('request')` (Node) | Pacote deprecado | `fetch` nativo / `axios` |
| Callbacks aninhados de I/O (Node) | Sem propagação de erro estruturada | `async/await` |
| `console.log` em produção | Sem nível nem correlação | logger estruturado (`pino`/`winston`) |

- *Calibração:* `task-manager-api/models/user.py:29` (MD5) e `utils/helpers.py:38`
  (`datetime.utcnow()`); `task-manager-api/routes/task_routes.py:67` (`Task.query.get`).
