# Template do Relatório de Auditoria

Referência da **Fase 2**. Este é o formato exato do relatório — não é sugestão de
estilo. Desvio de estrutura faz a auditoria parecer opinião.

## Regras que valem mais que o formato

1. **100% dos achados têm `arquivo:linha`.** Sem linha conferida no arquivo, o item
   não entra. Se você suspeita mas não achou a linha, não é achado — é hipótese.
2. **A linha é conferida, não estimada.** Abra o arquivo (`grep -n`, leitura) e
   copie o número. Achado com linha errada é pior que achado ausente.
3. **Ordenação fixa:** CRITICAL → HIGH → MEDIUM → LOW. Dentro do mesmo nível, por
   arquivo e depois por linha.
4. **A contagem vem no topo.** Quem lê precisa saber o tamanho do problema antes
   do primeiro item.
5. **Severidade pela definição** (`catalogo-anti-patterns.md`), não por susto.
   Na dúvida entre dois níveis, o menor — com uma linha de justificativa.
6. **Evidência é cópia literal** do código, com no máximo 5 linhas. Não reescreva,
   não "melhore" o trecho, não use `...` no meio de uma linha.
7. **Um achado por problema**, não por ocorrência. Se o mesmo padrão aparece em 6
   rotas, é um achado com 6 locais listados.
8. **Não invente correção que quebra contrato.** Se a correção exige mudar path ou
   resposta pública, escreva isso explicitamente na recomendação.

## Estrutura

```markdown
# Auditoria de Arquitetura — <nome do projeto>

> <AAAA-MM-DD> · <linguagem> + <framework> + <banco> · <n> arquivos / ~<n> linhas
> Escopo: <o que foi analisado> · Fora: <o que não foi>

## Resumo

| Severidade | Achados |
|---|---|
| CRITICAL | <n> |
| HIGH | <n> |
| MEDIUM | <n> |
| LOW | <n> |
| **Total** | **<n>** |

<Uma frase: qual é o problema estrutural dominante.>

## Achados

### F-01 · <título curto e específico> — CRITICAL

- **Anti-pattern:** AP-04 · God Class / arquivo onisciente
- **Local:** `src/AppManager.js:25-138`; `src/AppManager.js:80-129`
- **Evidência:**
  ```js
  <até 5 linhas literais do arquivo>
  ```
- **Impacto:** <o que quebra ou dificulta, em termos concretos: teste, manutenção,
  segurança, performance>
- **Correção:** <o que fazer, em 1-3 frases, citando a transformação do playbook>
  → `playbook-refatoracao.md` T-04

### F-02 · ... — HIGH
...

## APIs deprecated

| Local | API | Substituta | Severidade |
|---|---|---|---|
| `models/user.py:29` | `hashlib.md5()` | `bcrypt` | HIGH |

<Se não houver nenhuma: "Nenhuma API deprecated encontrada — verificado
`datetime.utcnow`, `.query.get()`, `print` em rota, `new Buffer`, `url.parse`.>

## Fora de escopo / não avaliado

- <ex.: `seed.py` com dados de exemplo não foi tratado como achado>
- <ex.: não foi possível subir a aplicação — dependências ausentes>

## Próximo passo — confirmação

A Fase 3 (refatoração) altera arquivos e **só começa com confirmação explícita**.
Responda:

1. **Aprovar tudo** e seguir para a refatoração; ou
2. **Aprovar só parte** (diga quais IDs: F-01, F-03...); ou
3. **Ajustar o relatório** (diga o que está errado ou faltando).

Nenhum arquivo foi modificado até aqui.
```

## Exemplo preenchido (recorte)

```markdown
# Auditoria de Arquitetura — code-smells-project

> 2026-10-29 · Python + Flask + SQLite · 4 arquivos / ~780 linhas
> Escopo: app.py, controllers.py, models.py, database.py · Fora: seed, testes

## Resumo

| Severidade | Achados |
|---|---|
| CRITICAL | 3 |
| HIGH | 1 |
| MEDIUM | 2 |
| LOW | 2 |
| **Total** | **8** |

A aplicação funciona, mas concentra HTTP, regra e SQL no mesmo par de arquivos e
expõe execução arbitrária de SQL em rota sem autenticação.

## Achados

### F-01 · Endpoint `/admin/query` executa SQL enviado pelo cliente — CRITICAL

- **Anti-pattern:** AP-02 · Superfície administrativa exposta
- **Local:** `app.py:59-78`
- **Evidência:**
  ```python
  query = dados.get("sql", "")
  ...
  cursor.execute(query)
  ```
- **Impacto:** qualquer cliente lê, altera ou apaga o banco inteiro sem
  autenticação. É execução arbitrária remota, não "ferramenta interna".
- **Correção:** remover a rota. Se houver necessidade de suporte, mover para um
  script de CLI fora do processo web, que nunca aceite SQL do cliente.
  → `playbook-refatoracao.md` T-02
```

## Como conferir antes de enviar

- [ ] Cada `F-xx` tem `arquivo:linha` que eu abri e confirmei
- [ ] A soma da tabela de resumo bate com o número de seções `### F-`
- [ ] Evidências são cópias literais, ≤5 linhas
- [ ] Ordem CRITICAL → HIGH → MEDIUM → LOW
- [ ] Nenhum achado sem impacto **e** correção
- [ ] APIs deprecated têm seção própria, mesmo que vazia ("nenhuma encontrada")
- [ ] A pausa de confirmação está no fim, com as 3 opções
- [ ] Eu não modifiquei nenhum arquivo do projeto

## Erros que invalidam o relatório

- **Linha aproximada** ("por volta de 40") ou faixa inventada.
- **Achado genérico** ("código não segue boas práticas") sem evidência.
- **Severidade inflada** para impressionar: MEDIUM tratado como CRITICAL.
- **Recomendação que muda o contrato HTTP** sem avisar que muda.
- **Relatório que já anuncia a refatoração feita** — a Fase 2 termina em pausa.
- **Repetir o mesmo problema como 6 achados** para aumentar a contagem.
