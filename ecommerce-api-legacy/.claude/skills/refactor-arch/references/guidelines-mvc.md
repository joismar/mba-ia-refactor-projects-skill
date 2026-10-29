# Guidelines MVC — o alvo da refatoração

Referência da **Fase 3**. Define o que é "estar em MVC" para esta skill: as
camadas, o que cada uma faz, e principalmente **o que não pode atravessar de uma
para outra**.

A regra que resume tudo: **a dependência aponta em uma direção só**.

```
rotas/views  →  controllers  →  services  →  repositories  →  models
   (HTTP)         (tradução)     (regra)       (dados)        (domínio)
```

Nenhuma seta volta. Um `service` não importa `request`; um `repository` não
devolve `Response`; um `model` não abre conexão.

---

## Camadas

| Camada | Responsabilidade | Pode importar | **Não pode** |
|---|---|---|---|
| **rotas / views** | mapear URL+método para um controller, registrar middleware | controllers, middlewares | SQL, regra de negócio, cálculo de domínio |
| **controllers** | traduzir HTTP ↔ domínio: ler payload, chamar service, escolher status e corpo | services, schemas, erros de domínio | SQL, `if` de regra de negócio, acesso direto ao banco |
| **services** | caso de uso: regra de negócio, orquestração, transação, efeitos colaterais | repositories, models, erros de domínio | `request`/`req`/`res`, status HTTP, JSON de resposta |
| **repositories** | acesso a dados: query, persistência, mapeamento linha→objeto | models, driver/ORM | regra de negócio, HTTP, validação de payload |
| **models** | entidade e invariantes do domínio, serialização segura | biblioteca padrão | abrir conexão, chamar outro service, conhecer HTTP |
| **config** | leitura de ambiente, validação de presença, defaults por ambiente | stdlib/os | importar qualquer camada da aplicação |
| **middlewares** | autenticação, validação de schema, correlação, erro | config, erros, serviços de auth | regra de negócio do domínio |
| **errors** | erros de domínio e o mapa erro→status HTTP | nada | framework web (idealmente) |

Camadas podem ser pastas (`services/`, `repositories/`) ou módulos
(`services.py`). A pasta não é o critério — o conteúdo é.

## Estrutura de diretórios

Python/Flask:

```
app/
  __init__.py        # create_app(): app factory, registra rotas e handlers
  config.py          # leitura e validação de env, fail-fast no boot
  routes/            # blueprints: só URL + chamada ao controller
  controllers/       # ou handlers/ — tradução HTTP
  services/          # regra de negócio por caso de uso
  repositories/      # SQL/ORM
  models/            # entidades
  schemas/           # validação de entrada (Pydantic/Marshmallow)
  middlewares/       # auth, erro, log
  errors.py          # exceções de domínio
run.py               # entry point: create_app().run()
```

Node/Express:

```
src/
  app.js             # express(), middlewares, monta routers
  server.js          # entry point: app.listen(PORT)
  config/index.js
  routes/
  controllers/
  services/
  repositories/
  models/
  schemas/
  middlewares/
  errors.js
```

Projeto pequeno pode achatar em `app/` sem subpastas por camada — desde que os
arquivos respeitem a tabela acima. O que não pode é um arquivo acumular camadas.

## O que não pode vazar

| Vazamento | Por que é problema | Como corrigir |
|---|---|---|
| SQL em controller/rota | regra e dados presos ao transporte | mover para repository, chamar via service |
| `request`/`req` em service | service deixa de ser reutilizável fora do HTTP | passar dados já extraídos (parâmetros) |
| `Response`/`jsonify` em service ou repository | camada de domínio vira dependente do framework | devolver objeto/entidade; controller formata |
| status HTTP decidido no service | regra de transporte misturada com domínio | levantar erro de domínio; middleware mapeia |
| regra de negócio em model de ORM "gordo" | entidade vira service disfarçado | mover orquestração para service |
| `os.environ` espalhado | configuração sem fonte única nem validação | centralizar em `config` |
| conexão/`cursor` aberto por request sem fechamento | vazamento de recurso | repository com escopo de conexão |
| segredo em código | vaza no histórico do Git | `config` + env (ver AP-01 / T-01) |
| exceção crua do driver chegando ao cliente | vaza stack/SQL | capturar no repository, traduzir em erro de domínio |

## Onde cada coisa vive

| Preocupação | Lugar certo | Lugar errado (típico) |
|---|---|---|
| validação de formato do payload | schema na fronteira (middleware/controller) | dentro da regra, repetida em N rotas |
| autorização | middleware/decorator antes do controller | `if user.role != 'admin'` no meio da regra |
| transação | service que delimita o caso de uso | commit avulso dentro de loop no repository |
| serialização da resposta | controller (ou `to_dict` do model, sem campo sensível) | montar JSON no repository |
| paginação | repository (defaults e teto), parâmetro vindo do controller | `LIMIT` hardcoded na rota |
| log | middleware de request + logger injetado | `print`/`console.log` espalhado |
| erro → status HTTP | handler central | `try/except` repetido em cada rota |
| efeito colateral externo (e-mail, cache) | service, **depois** do commit | dentro do repository, antes de persistir |

## Model não é tabela, DTO não é model

- **Model de domínio:** carrega invariantes (`pedido.total == soma dos itens`).
  Se ele só tem campos e getters, é um saco de dados — e a regra vai vazar para
  o service ou para a rota.
- **DTO/schema:** existe para a fronteira (entrada/saída). Não deve ser o mesmo
  objeto que o ORM usa.
- **Serialização:** `to_dict()` no model é aceitável desde que **nunca** inclua
  segredo, hash de senha, token ou campo interno (AP-06).

## Erros

1. Erros de domínio são classes próprias (`NotFound`, `ValidationError`,
   `Conflict`, `Unauthorized`), sem dependência de framework.
2. Um handler central traduz cada classe para status + envelope:
   `404`, `422`, `409`, `401/403`, `500` (inesperado, com log e sem stack no corpo).
3. Repository **não** decide status. Ele levanta `NotFound` ou deixa o erro de
   infraestrutura subir para o service traduzir.
4. `500` é sempre logado com contexto; o cliente recebe mensagem genérica.

## Configuração

- Um único módulo lê o ambiente.
- **Fail-fast:** se falta variável obrigatória, a aplicação não sobe e diz qual
  falta. Subir com default silencioso (`SECRET_KEY="changeme"`) é o mesmo defeito
  do hardcoded.
- Nada de segredo em repositório, em `.env` versionado ou em valor default.
- Debug desligado fora de desenvolvimento (AP-11).

## Entry point

- **Python:** `create_app()` (app factory) em `app/__init__.py`; `run.py` só chama.
  Permite testar sem subir servidor e sem estado global de módulo.
- **Node:** `app.js` exporta o app; `server.js` faz `listen`. Teste importa `app`.
- Estado global mutável (conexão singleton, cache em módulo) é o que a app factory
  existe para evitar (AP-08).

## Tamanho e coesão (referência, não dogma)

| Métrica | Alvo | Sinal de alerta |
|---|---|---|
| arquivo | ≤ 200 linhas | > 300 com mais de uma responsabilidade |
| função/handler | ≤ 30 linhas | > 50 ou mais de 3 níveis de `if` |
| parâmetros | ≤ 4 | objeto de contexto começando a virar "Deus" |
| responsabilidades por arquivo | 1 | 2+ da matriz da Fase 1 |

Número é desempate, não meta. O critério real continua sendo responsabilidade.

## Anti-padrões do próprio alvo

- **Service anêmico:** só repassa para o repository. Se não há regra, o service
  pode não existir — não crie camada decorativa.
- **Repository que devolve `Response`/`jsonify`:** o domínio passou a depender do
  framework.
- **Controller com `if` de negócio** que sobreviveu à refatoração: a Fase 3 não
  terminou.
- **`utils.py` como novo depósito:** utilitário genérico é aceitável; regra de
  domínio escondida em `utils` é o mesmo problema com outro nome.
- **Camada criada só para o relatório:** estrutura que ninguém usa é dívida nova.

## Checklist de conformidade (usado na Fase 3)

- [ ] Existe um módulo de config único, sem segredo literal (AP-01)
- [ ] Rotas só mapeiam URL+método; nenhuma tem SQL
- [ ] Controllers sem `if` de regra de negócio
- [ ] Services sem `request`/`req` e sem status HTTP
- [ ] Acesso a dados apenas em repositories
- [ ] Models sem conexão e sem campo sensível na serialização
- [ ] Validação de entrada em schema único por recurso
- [ ] Erros de domínio + handler central mapeando para status
- [ ] Transações delimitadas no service, efeito externo após commit
- [ ] Entry point com app factory (ou equivalente sem estado global)
- [ ] Dependência aponta numa direção só (nenhum import de fora para dentro)
- [ ] Contrato HTTP preservado: mesmos paths, métodos e formato de resposta
