# Scripts de evento do mesmo processo compartilham escopo Rhino

> **Plataforma:** Fluig · eventos de processo · Rhino
> **Status:** provado em servidor — e o comentário no código dizia o contrário

## Sintoma

Você define `logInfo()` em `afterStateLeave.js` com o prefixo `[Fluxo]`, roda, e no log do
servidor a linha sai com `[Fluxo:st73]` — o prefixo do **service task**, que é outro arquivo.

## Causa

Todos os `workflow/scripts/*.js` de um processo são carregados **no mesmo escopo Rhino**. É
justamente por isso que o motor consegue chamar `servicetask73()` pelo nome.

Consequência: duas funções com o **mesmo nome** em arquivos diferentes se sobrescrevem, e **a
última carregada vence** — silenciosamente, sem erro nenhum.

> ⚠️ O comentário no topo de um dos arquivos afirmava o oposto: *"Eventos de processo do Fluig
> NÃO compartilham escopo (cada .js roda isolado)"*. Estava errado, e o log provou. Comentário
> não é fonte.

## A prova

```
21:48:41 [Fluxo:st73] anexos antes do termo: 1
```

A função `congelarAnexos()` vive no `afterStateLeave.js`, cujo `logInfo` usa `[Fluxo]`. A linha
saiu com `:st73` porque o `logInfo` do `servicetask73.js` sobrescreveu o dele.

## O tamanho do problema

Levantamento num único processo:

| Helper | Definido em | Implementações distintas |
| --- | ---: | ---: |
| `setCard` | 5 arquivos | **5** |
| `card` | 5 | 2 |
| `cfg` | 2 | 2 (uma com cache, outra sem) |
| `logInfo` / `logWarn` / `logError` | 3 | — |
| `ATIV`, `urlPortal`, `gerarToken`, `numero` | 2 | iguais |

Nesse caso não quebrou nada porque **nenhum código dependia do retorno** do `setCard` (a versão
do service task não devolve valor, as outras devolvem booleano). Foi sorte, não desenho.

## Correção

**Prefixar os helpers por arquivo:** `_ase*` (afterStateLeave), `_bts*` (beforeTaskSave),
`_apc*` (afterProcessCreate), `_bcp*` (beforeCancelProcess), `_st73*` (service task 73).

A função que o **motor chama** (`beforeTaskSave`, `afterStateLeave`, `servicetask73`…) mantém o
nome oficial — só os auxiliares ganham prefixo.

## Como detectar antes

```bash
grep -h "^function " workflow/scripts/*.js | sort | uniq -d
```

Qualquer linha que aparecer é uma colisão esperando acontecer.

E se um log sair com o prefixo de **outro arquivo**, já aconteceu — não é erro de digitação no
prefixo, é sobrescrita.

## Relacionadas

- [`ArrayList` do Java não tem `.length` no Rhino](rhino-arraylist-length.md)
