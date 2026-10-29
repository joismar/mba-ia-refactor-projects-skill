# Playbook de Refatoração

Referência da **Fase 3**. Cada transformação resolve um anti-pattern do catálogo,
com código antes/depois. Aplique **uma por vez**, rode a aplicação e faça um commit
— refatoração em bloco único não tem como ser validada nem revertida.

## Regras de execução

1. **Ordem:** segurança → estrutura → performance → padronização. Corrigir estilo
   antes de tirar SQL da rota é trabalho que será refeito.
2. **Contrato HTTP é intocável:** mesmos paths, mesmos métodos, mesmo formato de
   resposta. Se a transformação exigir mudar contrato, **pare e pergunte**.
3. **Um commit por transformação**, com mensagem `refactor(<AP>): <o que mudou>`.
4. **Valide depois de cada uma:** a aplicação sobe e o endpoint afetado responde.
   Transformação sem boot validado não está concluída.
5. **Não crie camada vazia.** Se o service só repassa a chamada, não crie service.

## Ordem recomendada

| # | Transformações | Por quê |
|---|---|---|
| 1 | T-01, T-02, T-03, T-06, T-07 | segurança primeiro: segredo, RCE, injeção, senha, auth |
| 2 | T-04, T-05, T-08, T-09 (estrutura), T-13 | separar camadas antes de otimizar |
| 3 | T-09, T-10, T-11 | performance e exposição |
| 4 | T-12, T-14, T-15, T-16 | padronização e APIs modernas |

---

## T-01 · Segredos hardcoded → configuração por ambiente

**Corrige:** AP-01 · **Sinal:** `SECRET_KEY = "..."`, senha/URL de banco literal.

Antes:
```python
app = Flask(__name__)
app.config["SECRET_KEY"] = "minha-chave-super-secreta-123"
app.config["DEBUG"] = True
```

Depois — `app/config.py`:
```python
import os

class Config:
    # KeyError no boot é intencional: falhar cedo > subir com segredo default
    SECRET_KEY = os.environ["SECRET_KEY"]
    DATABASE_URL = os.environ.get("DATABASE_URL", "sqlite:///loja.db")
    DEBUG = os.getenv("FLASK_DEBUG", "0") == "1"
    CORS_ORIGINS = [o for o in os.getenv("CORS_ORIGINS", "").split(",") if o]
```
`.env.example` (versionado, sem valores reais):
```
SECRET_KEY=
DATABASE_URL=
FLASK_DEBUG=0
CORS_ORIGINS=
```

Node — `src/config/index.js`:
```js
require('dotenv').config();
const required = ['PAYMENT_GATEWAY_KEY', 'DATABASE_URL'];
for (const k of required) {
  if (!process.env[k]) throw new Error(`Variável de ambiente ausente: ${k}`);
}
module.exports = {
  paymentGatewayKey: process.env.PAYMENT_GATEWAY_KEY,
  databaseUrl: process.env.DATABASE_URL,
  corsOrigins: (process.env.CORS_ORIGINS || '').split(',').filter(Boolean),
};
```

**Atenção:** a chave antiga está no histórico do Git — **rotacione**. Trocar o
código sem rotacionar não resolve nada.

## T-02 · Rota administrativa / execução arbitrária → remover

**Corrige:** AP-02 · **Sinal:** `/admin/query` executando SQL recebido; `eval`,
`exec`, `child_process` com entrada do cliente.

Antes:
```python
@app.route("/admin/query", methods=["POST"])
def executar_query():
    query = request.get_json().get("sql", "")
    cursor.execute(query)          # cliente escolhe o SQL
```

Depois — rota apagada; operação vira CLI fora do processo web:
```python
# scripts/admin.py
import argparse
from app import create_app
from app.repositories import db

COMMANDS = {
    "stats": "SELECT COUNT(*) FROM pedidos",
    "reset": "DELETE FROM itens_pedido",   # SQL fixo, nunca do argumento
}

def main():
    p = argparse.ArgumentParser()
    p.add_argument("comando", choices=COMMANDS)
    args = p.parse_args()
    app = create_app()
    with app.app_context():
        db.session.execute(COMMANDS[args.comando])
        db.session.commit()
```

**Atenção:** se a rota for usada por um painel existente, removê-la quebra o
painel — isso é mudança de contrato: pergunte antes. O SQL recebido do cliente
não pode sobreviver em nenhuma versão.

## T-03 · SQL Injection → parâmetros vinculados

**Corrige:** AP-03 · **Sinal:** `execute("..." + var)`, f-string em SQL.

Antes:
```python
cursor.execute("SELECT * FROM produtos WHERE categoria = '" + categoria + "'")
```

Depois:
```python
cursor.execute(
    "SELECT id, nome, preco FROM produtos WHERE categoria = ? AND ativo = 1",
    (categoria,),
)
```

ORM:
```python
stmt = select(Produto).where(Produto.categoria == categoria, Produto.ativo.is_(True))
produtos = db.session.scalars(stmt).all()
```

Node:
```js
db.all("SELECT id, title FROM courses WHERE id = ? AND active = 1", [cid], cb);
```

**Atenção:** parâmetro só protege **valor**. Nome de tabela/coluna dinâmico exige
allowlist explícita (`{"nome", "preco"}`), nunca interpolação.

## T-04 · God Class → quebrar em camadas

**Corrige:** AP-04 · **Sinal:** um arquivo com rotas + regra + SQL + config.

Antes (`src/AppManager.js`, 141 linhas): `initDb()` cria tabelas e insere seed,
`setupRoutes()` declara todos os endpoints, e cada handler contém validação,
consulta, regra de pagamento, persistência, log e resposta.

Depois:
```
src/
  app.js                    # express(), middlewares, monta routers
  server.js                 # app.listen
  routes/checkout.routes.js # POST /api/checkout -> controller
  controllers/checkout.controller.js
  services/checkout.service.js   # regra: pagamento, matrícula, auditoria
  repositories/course.repo.js
  repositories/user.repo.js
  repositories/enrollment.repo.js
  models/course.js
  config/index.js
```

Esqueleto do controller (só tradução):
```js
exports.checkout = async (req, res, next) => {
  try {
    const input = checkoutSchema.parse(req.body);
    const result = await checkoutService.checkout(input);
    return res.status(200).json({ msg: 'Sucesso', enrollment_id: result.enrollmentId });
  } catch (err) { return next(err); }
};
```

**Atenção:** o `res.send("Usuário deletado, mas as matrículas ficaram sujas")`
revela um bug de domínio (AP-13), não só de arquitetura — corrija os dois
(T-13) ou registre como pendência explícita.

## T-05 · Regra de negócio na rota → service

**Corrige:** AP-05 · **Sinal:** handler com `if` de domínio, cálculo e query.

Antes:
```python
@tasks_bp.route("/tasks", methods=["POST"])
def criar_tarefa():
    data = request.get_json()
    if not data.get("title"):
        return jsonify({"erro": "titulo obrigatorio"}), 400
    if data.get("due_date") and datetime.fromisoformat(data["due_date"]) < datetime.utcnow():
        return jsonify({"erro": "data no passado"}), 400
    task = Task(title=data["title"], user_id=data["user_id"], priority=calcular_prioridade(data))
    db.session.add(task)
    db.session.commit()
    return jsonify(task.to_dict()), 201
```

Depois:
```python
# services/task_service.py
class TaskService:
    def __init__(self, repo: TaskRepository): self.repo = repo

    def criar(self, cmd: CriarTarefa) -> Task:
        if cmd.due_date and cmd.due_date < utcnow():
            raise ValidationError("due_date não pode estar no passado")
        task = Task(title=cmd.title, user_id=cmd.user_id,
                    priority=prioridade_por_prazo(cmd.due_date))
        return self.repo.add(task)

# controllers/task_controller.py
def criar_tarefa():
    cmd = CriarTarefaSchema().load(request.get_json())
    task = service.criar(cmd)
    return jsonify(task.to_dict()), 201
```

**Atenção:** o service não pode receber `request` nem devolver `jsonify`. Se ele
precisa saber o status HTTP, a regra ainda está no lugar errado.

## T-06 · Hash fraco e senha exposta → bcrypt + serializador seguro

**Corrige:** AP-06 · **Sinal:** `md5`/`sha1`, `to_dict()` com `password`.

Antes:
```python
def set_password(self, senha): self.password = hashlib.md5(senha.encode()).hexdigest()
def to_dict(self): return {"id": self.id, "email": self.email, "password": self.password}
```

Depois:
```python
import bcrypt

def set_password(self, senha: str) -> None:
    self.password_hash = bcrypt.hashpw(senha.encode(), bcrypt.gensalt()).decode()

def check_password(self, senha: str) -> bool:
    return bcrypt.checkpw(senha.encode(), self.password_hash.encode())

def to_dict(self) -> dict:
    return {"id": self.id, "email": self.email}   # sem hash, sem token
```

**Atenção:** hash antigo não vira bcrypt sozinho. Se houver dados, faça migração
no login (verifica MD5, regrava bcrypt) — e registre isso como pendência.

## T-07 · Autenticação fake → token assinado + verificação

**Corrige:** AP-07 · **Sinal:** `f"fake-jwt-token-{id}"`, nenhuma rota valida.

Antes:
```python
return jsonify({"token": f"fake-jwt-token-{user.id}"}), 200
```

Depois:
```python
import jwt
from datetime import datetime, timedelta, timezone

def emitir_token(user) -> str:
    return jwt.encode(
        {"sub": str(user.id), "exp": datetime.now(timezone.utc) + timedelta(hours=2)},
        current_app.config["SECRET_KEY"], algorithm="HS256",
    )

def requer_auth(fn):
    @wraps(fn)
    def wrapper(*a, **kw):
        header = request.headers.get("Authorization", "")
        if not header.startswith("Bearer "):
            raise Unauthorized("token ausente")
        try:
            payload = jwt.decode(header[7:], current_app.config["SECRET_KEY"], algorithms=["HS256"])
        except jwt.PyJWTError:
            raise Unauthorized("token inválido ou expirado")
        g.user_id = int(payload["sub"])
        return fn(*a, **kw)
    return wrapper
```

**Atenção:** aplicar o decorator **muda o comportamento** de rotas que hoje são
abertas. Se o contrato prevê acesso anônimo, pergunte quais rotas proteger.

## T-08 · Estado global mutável → app factory + injeção

**Corrige:** AP-08 · **Sinal:** `global db_connection`, singleton de módulo mutado.

Antes:
```python
db_connection = None
def get_db():
    global db_connection
    if db_connection is None:
        db_connection = sqlite3.connect(db_path, check_same_thread=False)
    return db_connection
```

Depois:
```python
# app/__init__.py
def create_app(config=Config):
    app = Flask(__name__)
    app.config.from_object(config)
    db.init_app(app)
    with app.app_context():
        db.create_all()
    register_blueprints(app)
    register_error_handlers(app)
    return app

# repositories/base.py
def get_session():
    return current_app.extensions["sqlalchemy"].session
```

**Atenção:** conexão por request precisa de `teardown_appcontext` para fechar —
trocar global por "global com nome novo" não resolve.

## T-09 · N+1 → uma consulta

**Corrige:** AP-09 · **Sinal:** query dentro de `for`/`map`, relação lazy na
serialização.

Antes:
```python
tarefas = db.session.scalars(select(Task)).all()
for t in tarefas:
    print(t.user.name)        # 1 query por tarefa
```

Depois:
```python
stmt = select(Task).options(joinedload(Task.user), selectinload(Task.category))
tarefas = db.session.scalars(stmt).unique().all()
```

Node — trocar N `db.get` aninhados por um `JOIN`:
```js
db.all(`SELECT e.id, u.name AS student, p.amount AS paid
        FROM enrollments e
        JOIN users u ON u.id = e.user_id
        LEFT JOIN payments p ON p.enrollment_id = e.id
        WHERE e.course_id = ?`, [courseId], cb);
```

**Atenção:** `joinedload` em relação 1:N duplica linhas — use `unique()` ou
`selectinload`. Meça antes/depois se possível; otimização sem medição é chute.

## T-10 · Listagem sem paginação → limit/offset com teto

**Corrige:** AP-10 · **Sinal:** `query.all()`, `SELECT *` sem `LIMIT`.

Antes:
```python
produtos = db.session.scalars(select(Produto)).all()
return jsonify([p.to_dict() for p in produtos])
```

Depois:
```python
page = max(int(request.args.get("page", 1)), 1)
per_page = min(max(int(request.args.get("per_page", 20)), 1), 100)   # teto 100

stmt = select(Produto).limit(per_page).offset((page - 1) * per_page)
itens = db.session.scalars(stmt).all()
total = db.session.scalar(select(func.count()).select_from(Produto))
return jsonify({"data": [p.to_dict() for p in itens],
                "meta": {"page": page, "per_page": per_page, "total": total}})
```

**Atenção:** mudar a resposta de lista para objeto **quebra contrato**. Se houver
cliente existente, mantenha a lista e informe paginação por headers
(`X-Total-Count`, `X-Page`) até haver aval para mudar o corpo.

## T-11 · Middleware sem restrição → escopo explícito

**Corrige:** AP-11 · **Sinal:** `CORS(app)`, `debug=True` com `host='0.0.0.0'`.

Antes:
```python
CORS(app)
app.run(host="0.0.0.0", port=5000, debug=True)
```

Depois:
```python
CORS(app, resources={r"/api/*": {"origins": app.config["CORS_ORIGINS"]}})
app.run(host="127.0.0.1", port=5000, debug=app.config["DEBUG"])
```

**Atenção:** com `debug=True` e `0.0.0.0` o console interativo fica exposto na
rede. Em produção use servidor WSGI/ASGI (gunicorn/uvicorn), não `app.run`.

## T-12 · Validação duplicada → schema único

**Corrige:** AP-12 · **Sinal:** o mesmo `if not campo` em N handlers.

Antes:
```python
if not data.get("email") or "@" not in data["email"]:
    return jsonify({"erro": "email invalido"}), 400
if not data.get("nome"):
    return jsonify({"erro": "nome obrigatorio"}), 400
```

Depois:
```python
# schemas/user_schema.py
class CriarUsuarioSchema(Schema):
    nome = fields.Str(required=True, validate=validate.Length(min=1, max=120))
    email = fields.Email(required=True)
    senha = fields.Str(required=True, validate=validate.Length(min=8))

# controller
try:
    dados = CriarUsuarioSchema().load(request.get_json() or {})
except ValidationError as e:
    raise ValidationError(e.messages)      # handler central -> 422
```

Node: `zod`/`joi` no middleware, com o mesmo schema para todas as rotas do recurso.

**Atenção:** validação de formato não substitui autorização. São camadas
diferentes (T-07).

## T-13 · Escrita sem transação → caso de uso atômico

**Corrige:** AP-13 · **Sinal:** N `delete()` em loop + commit avulso; efeito
externo antes do commit.

Antes:
```python
for tarefa in Tarefa.query.filter_by(user_id=user.id).all():
    db.session.delete(tarefa)
db.session.delete(user)
db.session.commit()
```

Depois:
```python
# services/user_service.py
def remover(self, user_id: int) -> None:
    with db.session.begin():                       # commit ou rollback
        self.tasks.delete_by_user(user_id)         # DELETE único, sem loop
        self.users.delete(user_id)
    # efeito externo só depois do commit
    self.notifier.usuario_removido(user_id)
```

**Atenção:** `with db.session.begin()` já abre transação; não chame `commit()`
dentro do bloco. Em infra que não suporta, use `try/commit/except rollback`.

## T-14 · Respostas sem padrão → envelope + handler central

**Corrige:** AP-14 · **Sinal:** `erro`, `error`, `mensagem`, `message`, lista crua.

Antes:
```python
return jsonify({"erro": "não encontrado"}), 404
return jsonify({"mensagem": "ok"}), 200
return jsonify(produtos), 200
```

Depois:
```python
# errors.py
class AppError(Exception):
    status = 500
    code = "internal_error"
class NotFound(AppError): status, code = 404, "not_found"
class ValidationError(AppError): status, code = 422, "validation_error"

# middlewares/error_handler.py
@app.errorhandler(AppError)
def handle(err):
    return jsonify({"success": False, "data": None,
                    "error": {"code": err.code, "message": str(err)}}), err.status
```

**Atenção:** este é o item com **maior risco de contrato**. Se o cliente atual lê
`{"erro": ...}`, padronizar o envelope quebra a integração. Aplique só com aval
explícito, ou mantenha o formato nas rotas existentes e padronize as novas.

## T-15 · Legibilidade: nome, interpolação, log

**Corrige:** AP-15, AP-16, AP-17.

```python
# antes
p = d["preco"] * 0.9
msg = "Total: " + str(p)
print(msg)

# depois
PRECO_COM_DESCONTO = 0.90
total_com_desconto = preco_base * PRECO_COM_DESCONTO
logger.info("total calculado", extra={"total": total_com_desconto})
```

```js
// antes
console.log("Processando cartão " + cc);
// depois
logger.info({ cardLast4: cc.slice(-4) }, 'processando pagamento');
```

**Atenção:** nunca logar dado sensível. `console.log` do cartão completo é achado
de segurança, não de estilo.

## T-16 · APIs deprecated → equivalentes modernas

| Antes | Depois |
|---|---|
| `datetime.utcnow()` | `datetime.now(timezone.utc)` |
| `Model.query.get(id)` | `db.session.get(Model, id)` |
| `@app.before_first_request` | inicialização dentro de `create_app()` |
| `print(...)` em rota | `logger = logging.getLogger(__name__)` |
| `new Buffer(x)` | `Buffer.from(x)` |
| `url.parse(u)` | `new URL(u)` |
| `body-parser` | `express.json()` |
| `require('request')` | `fetch` nativo ou `axios` |
| callbacks aninhados | `async/await` |
| `hashlib.md5` para senha | `bcrypt` / `argon2` |

```python
# antes
criado = datetime.utcnow()
task = Task.query.get(task_id)

# depois
criado = datetime.now(timezone.utc)
task = db.session.get(Task, task_id)
```

**Atenção:** `utcnow()` retorna datetime *naive*; trocar por `now(timezone.utc)`
muda comparações com valores antigos gravados sem tz. Verifique persistência antes.

---

## Validação obrigatória após cada transformação

```bash
# 1. sobe
python run.py            # ou npm start
# 2. responde
curl -s localhost:5000/health
# 3. caso de erro
curl -s -o /dev/null -w "%{http_code}\n" localhost:5000/produtos/999999
```

- [ ] Boot sem exceção (saída real anexada ao relatório)
- [ ] Endpoints originais respondem com o mesmo path/método
- [ ] Formato de resposta preservado (ou mudança aprovada)
- [ ] Caso de erro retorna status coerente, sem stack trace no corpo
- [ ] `git diff` do commit contém só o escopo daquela transformação

## Erros comuns

- **Big bang:** reescrever tudo e testar no fim. Sem commit por transformação não
  há como isolar a causa quando algo quebra.
- **Mudar contrato de brinde:** renomear campo, trocar envelope, alterar status.
  Refatoração não é redesign de API.
- **Camada decorativa:** criar `services/` com 5 linhas de repasse. Se não há
  regra, não há service.
- **Corrigir só o sintoma:** trocar `md5` por `sha256` (continua inadequado para
  senha) ou pôr `LIMIT 100` fixo em vez de paginação.
- **Esquecer o efeito colateral:** mover a regra para o service mas manter o envio
  de e-mail dentro do repository.
- **Não rodar a aplicação:** "deve funcionar" não é validação. A Fase 3 só fecha
  com boot e requisição reais.
