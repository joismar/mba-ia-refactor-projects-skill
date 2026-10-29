# Análise de Projeto — heurísticas da Fase 1

Referência da **Fase 1**. O objetivo é responder cinco perguntas com evidência de
arquivo: qual a stack, onde a aplicação começa, como as responsabilidades estão
distribuídas hoje, quais arquivos concentram problema e quanto código existe.

Nada aqui pode ser respondido "de cabeça". Cada campo do resumo sai de um arquivo
lido ou de um comando executado. Se não deu para confirmar, o campo recebe
`desconhecido` — nunca um palpite.

## Saída obrigatória da fase

```
STACK:       <linguagem> + <framework> + <banco>
ENTRADA:     <arquivo> (porta <n>)
ARQUIVOS:    <n> analisados | ~<n> linhas
ARQUITETURA: <uma frase: como as responsabilidades estão distribuídas hoje>
SUSPEITOS:   <n> arquivos com indício de anti-pattern
```

---

## 1. Inventário

Liste o código-fonte real, não o repositório inteiro:

```bash
find . -type f \( -name "*.py" -o -name "*.js" -o -name "*.ts" -o -name "*.rb" \
  -o -name "*.php" -o -name "*.java" -o -name "*.go" \) \
  -not -path "*/node_modules/*" -not -path "*/venv/*" -not -path "*/.venv/*" \
  -not -path "*/__pycache__/*" -not -path "*/dist/*" -not -path "*/build/*" \
  -not -path "*/.git/*" | xargs wc -l | sort -n
```

Regras de contagem:

- **Analisado** = arquivo de código do domínio (entrada, rotas, modelos, serviços,
  utilitários, config). Não conta: `package-lock.json`, `*.min.js`, migrations
  geradas, `seed.py` de dados fixos, fixtures, testes, `.claude/`.
- Monorepo/multi-projeto: conte por subprojeto e diga qual foi analisado. O mesmo
  vale para `repo/` contendo 3 aplicações — cada uma é uma auditoria.
- Se a raiz tiver só `README` e uma pasta de app, o projeto é a pasta.

## 2. Linguagem

Ordem de confiança: manifest → extensão → shebang.

| Evidência | Conclusão |
|---|---|
| `requirements.txt`, `pyproject.toml`, `setup.py`, `Pipfile` | Python |
| `package.json` | JavaScript/TypeScript (`tsconfig.json` ou `*.ts` confirma TS) |
| `go.mod`, `composer.json`, `Gemfile`, `pom.xml`, `build.gradle` | Go, PHP, Ruby, Java |
| Só `*.py` sem manifest | Python (diga "sem manifest — dependências não declaradas") |
| Só `*.js` sem `package.json` | JavaScript sem gerenciador |

Divergência entre manifest e extensões mais frequentes é achado por si só
(manifest de Node num projeto Python indica boilerplate copiado): registre em
"Fora de escopo" no relatório, não como anti-pattern.

## 3. Framework

Confirme pelo **import no código**, não pela dependência declarada — dependência
instalada e não usada não define a stack.

| Sinal lido no arquivo | Framework |
|---|---|
| `from flask import Flask` / `Flask(__name__)` | Flask |
| `from fastapi import FastAPI` / `FastAPI()` | FastAPI |
| `django.core`, `settings.py` com `INSTALLED_APPS` | Django |
| `import express` / `express()` | Express |
| `NestFactory.create`, `@Controller()` | NestJS |
| `Rails.application`, `config/routes.rb` | Rails |
| `use Illuminate`, `artisan` | Laravel |
| Nenhum dos acima | "sem framework" (HTTP nativo, CLI, lib) — registre o que é |

Heurísticas de apoio: como as rotas são declaradas (`@app.route`, `app.get(`,
`app.add_url_rule`, `router.get(`, decorators `@Controller`) e onde ficam os
middlewares.

## 4. Banco de dados

| Evidência | Banco |
|---|---|
| `sqlite3.connect(...)`, `new sqlite3.Database`, arquivo `*.db`/`*.sqlite` | SQLite |
| `psycopg2`, `asyncpg`, `postgres://`, `pg` no `package.json` | PostgreSQL |
| `mysqlclient`, `mysql2`, `pymysql` | MySQL/MariaDB |
| `pymongo`, `mongoose`, `MongoClient` | MongoDB |
| `sequelize`, `prisma`, `knex`, `sqlalchemy`, `peewee`, `mongoose` | ORM/Query builder — diga qual |
| Só `CREATE TABLE` em string | SQL cru (sem ORM) — relevante para AP-03 |

Não execute migrations nem suba o banco para descobrir. Se o arquivo do banco
existir, `sqlite3 <arquivo> ".tables"` é leitura, não alteração — permitido.

## 5. Ponto de entrada e comando de boot

| Sinal | Interpretação |
|---|---|
| `if __name__ == "__main__":` + `app.run(...)` | entry point Python; a porta está no `app.run` |
| `uvicorn app:app`, `gunicorn app:app` | entry point via ASGI/WSGI |
| `manage.py runserver` | Django |
| `"start"` / `"dev"` em `package.json` → `scripts` | entry point Node |
| `app.listen(PORT)` | porta no código ou em `process.env.PORT` |
| `Dockerfile` com `CMD`, `Procfile` com `web:` | comando de produção — cite a linha |
| Função `create_app()` | *app factory*: o boot é `create_app().run()` ou `uvicorn create_app` |

Registre a **porta** com a fonte (`app.py:88`) e o comando exato para subir. Se
dependências não estão instaladas, o boot é `pip install -r requirements.txt` /
`npm install` antes. Sem comando de boot identificável, a Fase 3 não pode ser
validada — trate como bloqueio, não como detalhe.

## 6. Mapa de arquitetura

Monte uma matriz simples: para cada arquivo relevante, marque o que ele faz.

| Responsabilidade | Como reconhecer |
|---|---|
| Rota/HTTP | declaração de path, `request`/`req`, `jsonify`/`res.send`, status code |
| Regra de negócio | `if` de domínio, cálculo de preço/status, decisão de fluxo |
| Acesso a dados | SQL, `cursor`, `db.query`, `Model.query`, `this.db.run` |
| Configuração | chave, senha, URL, flag de ambiente |
| Serialização | `to_dict`, montagem do JSON de resposta |
| Efeito colateral | envio de e-mail, cache, log, chamada externa |

Classifique o conjunto em uma das três:

1. **Arquivo onisciente (God Class)** — um ou dois arquivos concentram 4+ das
   responsabilidades acima. Critério objetivo: arquivo com rotas **e** SQL **e**
   regra no mesmo módulo, tipicamente >150 linhas ou >10 handlers.
2. **MVC parcial** — há separação de pastas (`models/`, `routes/`, `controllers/`),
   mas a regra vaza para a rota ou o model faz SQL direto.
3. **Camadas** — cada responsabilidade tem casa própria e a dependência aponta
   numa direção só (ver `guidelines-mvc.md`).

Diga **onde** cada responsabilidade vive hoje: "rotas em `task_routes.py`, regra
nas rotas, dados via SQLAlchemy dentro das rotas, config hardcoded em `app.py:13`".

## 7. Suspeitos

Ordene candidatos por três sinais, não por tamanho isolado:

1. arquivo com maior número de responsabilidades da matriz (peso maior);
2. arquivo que importa framework **e** driver de banco **e** módulo de domínio;
3. maior arquivo em linhas (desempate).

Liste os 3-5 primeiros. "Arquivo grande e coeso" (ex.: catálogo de constantes) não
é suspeito — o critério é mistura de responsabilidade, não volume.

## Casos especiais

- **Sem manifest e sem framework:** o projeto ainda tem stack (Python + SQLite,
  por exemplo). Não invente dependência: registre o que o código importa.
- **Vários entry points** (API + script de seed + CLI): o entry point da auditoria
  é o que sobe o serviço HTTP; os outros entram como observação.
- **Código gerado/vendorizado:** exclua da contagem e diga que excluiu.
- **Projeto que não sobe** (dependência quebrada): a Fase 1 continua; registre o
  erro real do boot na saída, porque a Fase 3 vai precisar dele.
- **Skill/artefato dentro do projeto** (`.claude/`, `reports/`): não faz parte da
  aplicação analisada.

## Pitfalls

- **Pasta `models/` não é camada de modelo.** Confirme o conteúdo: se o arquivo
  tem `cursor.execute`, é acesso a dados com nome enganoso.
- **Contar linhas do repositório inteiro** e inflar `ARQUIVOS`. A contagem é do
  código analisado, com lockfile fora.
- **Deduzir porta ou comando de boot** de um README desatualizado. README orienta,
  código decide.
- **Parar em "é Flask".** A fase só termina com o mapa de responsabilidades: o que
  importa para a Fase 2 é onde a responsabilidade vaza.
- **Marcar suspeito por tamanho.** Arquivo de 400 linhas com uma responsabilidade
  só (ex.: schema de validação) não é candidato a God Class.

## Checklist da fase

- [ ] Lista de arquivos analisados com contagem de linhas (excluindo lockfiles e gerados)
- [ ] Linguagem confirmada por manifest **ou** import lido
- [ ] Framework confirmado por import, não por dependência declarada
- [ ] Banco identificado (driver/ORM/arquivo)
- [ ] Entry point e porta com `arquivo:linha`
- [ ] Comando de boot escrito e executável
- [ ] Matriz de responsabilidades preenchida por arquivo
- [ ] Classificação: onisciente / MVC parcial / camadas, com justificativa
- [ ] Suspeitos ordenados por responsabilidade, não por tamanho
- [ ] Bloco de saída impresso, sem nenhum campo por suposição
