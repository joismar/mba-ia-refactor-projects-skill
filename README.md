# Criação de Skills - Refatoração Arquitetural Automatizada

## Análise Manual

- **CRITICAL:** Falhas graves de arquitetura ou segurança que impedem o funcionamento correto, expõem dados sensíveis (ex: credenciais hardcoded, SQL Injection) ou violam completamente a separação de responsabilidades (ex: "God Class" contendo banco de dados, lógicas complexas e roteamento no mesmo arquivo).
- **HIGH:** Fortes violações do padrão MVC ou princípios SOLID que dificultam muito a manutenção e testes (ex: lógicas de negócio pesadas presas dentro de Controllers, forte acoplamento sem Injeção de Dependência, ou uso de estado global mutável em toda a aplicação).
- **MEDIUM:** Problemas de padronização, duplicação de código ou gargalos de performance moderada (ex: Queries N+1 no banco de dados, uso inadequado de middlewares, validações ausentes nas rotas).
- **LOW:** Melhorias de legibilidade, nomenclatura de variáveis ruins, ou "magic numbers" soltos pelo código.

### Code Smells Project

#### Secret Key Hardcoded
- **Severity:** CRITICAL
- **Description:** A chave secreta está codificada diretamente no código.
- **File(s):** `app.py:7`, `controllers.py:289`.
- **Impact:** A exposição da chave secreta pode permitir que atacantes comprometam a segurança da aplicação, acessem dados sensíveis ou realizem ações maliciosas.
- **Recommendation:** Remover a chave secreta do código e utilizar variáveis de ambiente ou serviços de gerenciamento de segredos para armazená-la de forma segura. Rotacionar a chave secreta vazada.

#### Falta de paginação nas listagens
- **Severity:** MEDIUM
- **Description:** As listagens de produtos não possuem paginação.
- **File(s):** `controllers.py` e `models.py`.
- **Impact:** A ausência de paginação pode resultar em tempos de resposta lentos e sobrecarga no servidor, além de dificultar a navegação para os usuários.
- **Recommendation:** Implementar paginação nas listagens de produtos para melhorar a performance e a experiência do usuário.

#### get_db Fazendo Consulta de Todos os produtos
- **Severity:** MEDIUM
- **Description:** A função `get_db` está realizando uma consulta que retorna todos os produtos sempre que é chamada.
- **File(s):** `database.py`.
- **Impact:** Consultas que retornam todos os produtos podem causar lentidão e sobrecarga no banco de dados, especialmente em aplicações com grande volume de dados.
- **Recommendation:** Implementar método de seed para dados iniciais.

#### Uso do + ao invés de f-string
- **Severity:** LOW
- **Description:** O código está utilizando a concatenação de strings com o operador `+` ao invés de f-strings.
- **File(s):** `models.py`.
- **Impact:** A concatenação de strings com `+` pode resultar em código menos legível e mais propenso a erros, além de ser menos eficiente em termos de performance.
- **Recommendation:** Substituir a concatenação de strings com `+` pelo uso de f-strings para melhorar a legibilidade e eficiência do código.

#### Respostas de Erro e Sucesso sem Padronização
- **Severity:** LOW
- **Description:** As respostas da API não utilizam objetos para padronizar o retorno.
- **File(s):** `controllers.py`.
- **Impact:** A falta de padronização nas respostas da API pode dificultar a integração com clientes e aumentar a complexidade do código, tornando-o menos previsível e mais difícil de manter.
- **Recommendation:** Implementar uma padronização para as respostas de erro e sucesso, utilizando objetos ou classes específicas para encapsular as respostas da API.

### Ecommerce Api Legacy

#### God Class AppManager
- **Severity:** HIGH
- **Description:** A classe `AppManager` está assumindo muitas responsabilidades, o que a torna uma "God Class".
- **File(s):** `AppManager.js`.
- **Impact:** A presença de uma "God Class" pode levar a um código difícil de entender, testar e manter, aumentando o risco de introdução de bugs e dificultando a evolução da aplicação.
- **Recommendation:** Refatorar o design da aplicação para se adequadar ao padrão de arquitetura MVC, separando as responsabilidades em classes e módulos distintos.

#### Falta de Padrões de Request e Response
- **Severity:** MEDIUM
- **Description:** A API não segue padrões consistentes e tipos definidos para requisições e respostas.
- **File(s):** `AppManager.js`.
- **Impact:** A ausência de padrões consistentes para requisições e respostas pode dificultar a integração com clientes e aumentar a complexidade do código, tornando-o menos previsível e mais difícil de manter.
- **Recommendation:** Implementar padrões consistentes para requisições e respostas, utilizando tipos definidos.

#### Consultas e Inserts Aninhados
- **Severity:** MEDIUM
- **Description:** A aplicação está realizando consultas e inserts aninhados.
- **File(s):** `AppManager.js`.
- **Impact:** Consultas e inserts aninhados podem levar a problemas de performance e dificultar a manutenção do código, além de aumentar o risco de introdução de bugs.
- **Recommendation:** Refatorar o código para evitar consultas e inserts aninhados, utilizando técnicas como joins, transações ou consultas separadas para melhorar a performance e a legibilidade do código.

#### Variáveis com Nomenclatura Ruim
- **Severity:** LOW
- **Description:** Algumas variáveis no código possuem nomes pouco descritivos ou confusos.
- **File(s):** `AppManager.js`.
- **Impact:** Nomes de variáveis pouco descritivos podem dificultar a compreensão do código, tornando-o menos legível e aumentando o risco de introdução de bugs.
- **Recommendation:** Renomear variáveis para nomes mais descritivos e claros, seguindo boas práticas de nomenclatura e convenções de codificação.

#### Variáveis Globais
- **Severity:** LOW
- **Description:** A aplicação está utilizando variáveis globais hardcoded.
- **File(s):** `utils.js`.
- **Impact:** O uso de variáveis globais hardcoded pode levar a problemas de manutenção, dificultar a testabilidade do código e aumentar o risco de introdução de bugs.
- **Recommendation:** Evitar o uso de variáveis globais hardcoded, utilizando injeção de dependência, variáveis de ambiente ou serviços de configuração para gerenciar valores globais de forma mais segura e flexível.

### Task Manager Api

#### Secret Key e Credenciais Hardcoded
- **Severity:** CRITICAL
- **Description:** A aplicação armazena a `SECRET_KEY` diretamente no código e o serviço de e-mail também usa usuário/senha fixos no código-fonte.
- **File(s):** `app.py`.
- **Impact:** Isso expõe informações sensíveis em repositórios e facilita comprometimento de segurança em produção, além de impedir configuração adequada por ambiente.
- **Recommendation:** Extrair as configurações para variáveis de ambiente e utilizar um gerenciador de segredos com arquivos de configuração separados por ambiente.

#### Senhas com Hash Fraco e Exposição em Respostas da API
- **Severity:** HIGH
- **Description:** As senhas são geradas por `hashlib.md5()`, que é considerado inseguro para armazenamento de credenciais, e o modelo `User.to_dict()` retorna `password` em respostas públicas.
- **File(s):** `models/user.py`.
- **Impact:** Criptografia fraca torna força bruta e ataques mais viáveis, enquanto a exposição do hash pode causar vazamento de dados e risco de reutilização em outros sistemas.
- **Recommendation:** Migrar para `bcrypt` ou `argon2`, remover o campo de senha do serializador e aplicar políticas de hashing com salt.

#### Lógica de Negócio Misturada com Rotas
- **Severity:** HIGH
- **Description:** As rotas de usuários, tarefas e relatórios misturam validação, regras de negócio e acesso ao banco de dados no mesmo arquivo, gerando código duplicado e difícil de evoluir.
- **File(s):** `routes/task_routes.py`, `routes/user_routes.py`, `routes/report_routes.py`.
- **Impact:** Qualquer mudança de regra exige editar várias rotas, aumenta o risco de inconsistência e dificulta testes automatizados e manutenção.
- **Recommendation:** Criar camadas de serviço e validadores reutilizáveis, deixando as rotas responsáveis apenas por transporte HTTP e autorização.

#### Autenticação Fake e JWT Não Real
- **Severity:** HIGH
- **Description:** O endpoint de login retorna uma string do tipo `fake-jwt-token-<id>`, sem assinatura, expiração, verificação ou armazenamento seguro de sessão.
- **File(s):** `routes/user_routes.py`.
- **Impact:** Essa abordagem não oferece autenticação real e pode permitir acesso indevido a endpoints protegidos, além de criar uma falsa sensação de segurança.
- **Recommendation:** Implementar autenticação com JWT real, expiração controlada e middleware para validar tokens em todas as rotas sensíveis.

#### Consultas em Loop (N+1) e Carregamento Não Paginado
- **Severity:** MEDIUM
- **Description:** O código executa múltiplas consultas dentro de loops para obter usuários, categorias e tarefas relacionadas, além de listar todos os registros sem paginação.
- **File(s):** `routes/task_routes.py`, `routes/user_routes.py`, `routes/report_routes.py`.
- **Impact:** Isso degrada a performance à medida que o volume de dados cresce, gerando maior latência e maior carga no banco de dados.
- **Recommendation:** Utilizar `joinedload`/`relationship` quando necessário, aplicar paginação com `limit` e `offset`, e consolidar consultas agregadas em relatórios.

#### CORS Liberado para Qualquer Origem
- **Severity:** MEDIUM
- **Description:** O middleware de CORS é registrado sem qualquer restrição de origem (`CORS(app)`), liberando a API para requisições vindas de qualquer domínio.
- **File(s):** `app.py:15`.
- **Impact:** Qualquer site pode consumir a API em nome dos usuários e contornar a política de mesma origem do navegador; além disso, não há como restringir a API ao front-end oficial quando ela for publicada em produção.
- **Recommendation:** Declarar explicitamente as origens permitidas por ambiente (`CORS(app, resources={r"/api/*": {"origins": [...]}})`) e remover o middleware onde ele não for necessário.

#### Respostas da API sem Padrão e Validações Duplicadas
- **Severity:** LOW
- **Description:** Cada rota retorna estruturas diferentes e reimplementa regras de validação repetidamente, em vez de centralizar respostas e regras de contrato da API.
- **File(s):** `routes/*.py`.
- **Impact:** Isso aumenta a chance de erros de integração, dificulta a documentação e gera comportamento inconsistente entre endpoints.
- **Recommendation:** Padronizar respostas em envelopes `{success, data, error}` e criar helpers/DTOs para validações compartilhadas.

#### Status e Priorização com Lógica Repetitiva
- **Severity:** LOW
- **Description:** A lógica de verificar `due_date`, `overdue`, status e prioridade aparece repetida em múltiplos endpoints, gerando manutenção manual e risco de divergência.
- **File(s):** `routes/task_routes.py`, `routes/user_routes.py`, `routes/report_routes.py`.
- **Impact:** Pequenas inconsistências em uma rota podem divergir das outras, causando comportamento imprevisível para os clientes.
- **Recommendation:** Extrair utilitários ou métodos de domínio (`Task.is_overdue()`, validadores de modelo) e reaproveitá-los em toda a aplicação.

