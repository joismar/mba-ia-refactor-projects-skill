---
name: refactor-arch
description: Audita projeto legado contra anti-patterns e refatora para o padrão MVC em qualquer stack. Use ao revisar arquitetura, procurar code smells, gerar relatório de auditoria ou separar um projeto em camadas.
---

# refactor-arch — Refatoração Arquitetural Automatizada

Audita um projeto legado, produz um relatório de achados com `arquivo:linha` e
refatora o código para o padrão MVC. É **agnóstica de stack**: nada aqui assume
Python, Flask, Express ou SQLite — a detecção acontece na Fase 1 e o restante do
fluxo se adapta ao que for encontrado.

As 3 fases são **sequenciais e não podem ser fundidas**. A Fase 2 termina com uma
pausa obrigatória: nenhum arquivo é modificado sem confirmação humana.

## Quando usar

- "analise a arquitetura desse projeto", "procure code smells / anti-patterns", "audite este repo"
- "refatore para MVC", "separe em camadas", "tire a regra de negócio das rotas"
- Revisão de código legado herdado, antes de escrever testes ou novas features

Não use para: corrigir um bug pontual, revisar um diff/PR pequeno, formatar código.

## Fase 1 — Análise

Objetivo: saber **o que** é o projeto antes de opinar sobre ele.

1. Mapear a árvore de arquivos, ignorando `node_modules/`, `venv/`, `.venv/`,
   `__pycache__/`, `dist/`, `build/`, `.git/`.
2. Detectar **linguagem, framework, banco e ponto de entrada** pelas heurísticas
   de `references/analise-projeto.md`.
3. Mapear a arquitetura atual: onde vivem rotas, regra de negócio, acesso a dados
   e configuração. Classificar em: arquivo onisciente / MVC parcial / camadas.
4. Localizar os candidatos a anti-pattern: os maiores arquivos, os que concentram
   mais de uma responsabilidade e os pontos de entrada HTTP.
5. **Imprimir o resumo** e encerrar a fase.

```
STACK:       <linguagem> + <framework> + <banco>
ENTRADA:     <arquivo> (porta <n>)
ARQUIVOS:    <n> analisados | ~<n> linhas
ARQUITETURA: <uma frase: como as responsabilidades estão distribuídas hoje>
SUSPEITOS:   <n> arquivos com indício de anti-pattern
```

**Concluída quando:** o resumo foi impresso e cada campo vem de arquivo lido —
nenhum campo preenchido por suposição.

## Fase 2 — Auditoria

1. Cruzar o código contra **todos** os anti-patterns de
   `references/catalogo-anti-patterns.md`, incluindo a seção de APIs deprecated.
2. Registrar **arquivo e linha exatos** em cada achado. A linha é obrigatória e
   precisa ser conferida no arquivo — não inferida, não estimada.
3. Classificar a severidade pela escala do catálogo, justificando pela definição
   (CRITICAL / HIGH / MEDIUM / LOW), não por intuição.
4. Montar o relatório no formato exato de `references/template-relatorio.md`, com
   a contagem por severidade no topo.
5. **Pausar.** Apresentar o relatório e pedir confirmação explícita antes de tocar
   em qualquer arquivo. Se o usuário recusar ou pedir ajustes, revisar o relatório
   e perguntar novamente.

**Concluída quando:** o relatório foi apresentado com `arquivo:linha` em 100% dos
achados e a confirmação do usuário foi obtida.

## Fase 3 — Refatoração

1. Reestruturar para MVC conforme `references/guidelines-mvc.md`, aplicando em cada
   anti-pattern confirmado a transformação correspondente de
   `references/playbook-refatoracao.md`.
2. **Preservar o contrato HTTP:** as rotas existentes continuam existindo, com os
   mesmos paths e métodos. Refatoração não é redesign de API — se um path precisa
   mudar, isso é decisão do usuário, não da skill.
3. Validar de verdade: instalar dependências, **subir a aplicação** e chamar os
   endpoints principais, incluindo pelo menos um caso de erro. Guardar a saída real.
4. Reportar: o que mudou por arquivo, o que foi validado com a saída do boot e das
   requisições, e o que ficou pendente.

**Concluída quando:** a aplicação sobe e os endpoints respondem após a refatoração,
com a saída real do teste anexada ao relatório.

## Referências

Em `references/`, carregadas sob demanda — não presuma o conteúdo:

- `analise-projeto.md` — heurísticas de detecção de linguagem, framework, banco,
  comando de boot e mapeamento de arquitetura
- `catalogo-anti-patterns.md` — catálogo de anti-patterns com sinais de detecção,
  severidade e APIs deprecated/obsoletas
- `template-relatorio.md` — formato padronizado do relatório de auditoria (Fase 2)
- `guidelines-mvc.md` — regras do alvo: camadas, responsabilidade de cada uma e o
  que não pode vazar entre elas
- `playbook-refatoracao.md` — padrões de transformação por anti-pattern, com
  exemplos de código antes/depois

## Regras invioláveis

- **Agnóstica de stack.** Nenhuma fase pode assumir uma linguagem ou framework. Se
  o projeto não é de nenhuma das stacks de exemplo, o fluxo continua o mesmo.
- **A Fase 2 sempre pausa.** Não existe "o usuário já autorizou antes" — a
  confirmação vale por execução.
- **Sem `arquivo:linha` não é achado.** Sem a linha exata o item vira opinião, e
  opinião não entra no relatório.
- **A Fase 3 não quebra contrato.** Se a correção exigir alterar rota ou resposta
  pública, pare e pergunte.

## Pitfalls

- **Confundir "arquivo grande" com God Class.** O anti-pattern é acúmulo de
  responsabilidades diferentes (HTTP + regra + SQL), não o tamanho em linhas.
- **Inflar severidade.** Marcar MEDIUM como CRITICAL torna o relatório inútil e
  queima a confiança no resultado. Use a definição de cada nível.
- **Validar com "deve funcionar".** A Fase 3 exige boot real e requisição real;
  sem isso a fase não terminou.
- **Refatorar o relatório.** A Fase 3 corrige código. A auditoria não é reescrita
  para justificar o que foi feito.
