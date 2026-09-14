# Tabela pai-filho sem `tablename` não persiste as linhas

> **Plataforma:** Fluig · formulário · tabela pai × filho
> **Status:** confirmado em servidor · custou três semanas

## Sintoma

`saveAndSendTask` (ou `startProcess`) envia `campo___1`, `campo___2`… no `cardData`; o engine
**aceita**, grava o card (`Saving card ... Sequence: 26`, com os campos presentes nos dados)
e, ao reler, **a tabela filha está vazia**.

Nenhum erro. Em lugar nenhum.

O custo real: três semanas. O `move` foi dado como falho, e o projeto chegou a pivotar para
um widget Java que nunca precisou existir.

## Causa

A `<table>` estava assim:

```html
<table class="table table-bordered" id="tbItens">
```

Sem o atributo **`tablename`**. A TDN (pageId 668198714, *Personalização de formulários*,
seção Pai x Filho) diz:

> `<table tablename="teste">` — A propriedade *tablename* determina que abaixo dessa tabela
> será implementado um sistema de pai filho dentro da definição de formulário.

Sem ele o parser **não cria a estrutura pai-filho**, e `campo___N` não tem onde persistir.

O que mascara o problema: **`wdkAddChild()` continua funcionando na tela** — é JavaScript, não
depende do parser. A tabela parece uma pai-filho, se comporta como uma pai-filho na interface,
e não é uma.

## Correção

```html
<table class="table table-bordered" id="tbItens" tablename="tbItens" noaddbutton="true">
```

Publicar o formulário e testar numa solicitação **nova** — solicitação em andamento fica presa
na versão do form em que nasceu (ver [nota relacionada](form-versao-nao-retroage.md)).

## A prova

Mesmo código, mesma chamada, só com o `tablename` adicionado:

```
21:35:10 [dataset] itens: 0 ja existiam, +2 nesta rodada
21:35:10 saveAndSendTask pid=NNNNNNN -> atividade 26 | itens=2 | anexos=0
21:35:10 retorno bruto: WDNrDocto=NNNNNNN | iTask=26 | cDestino=[Pool:Role:admin]
```

E na tela: **as duas linhas gravadas**, com todos os campos. A rotina de conferência que relê
o card e lança erro se `campo___1` vier vazio **passou** — e era exatamente esse passo que
falhava antes.

Como é comportamento de parser, vale para qualquer tenant.

## Como detectar antes

```bash
grep -i tablename form.html
```

Antes de **qualquer** teste de gravação de tabela filha. Se a tabela é "pai-filho" só no
JavaScript, o servidor não sabe disso — e não vai te contar.

## Relacionadas

- [Nova versão de formulário não alcança solicitação em andamento](form-versao-nao-retroage.md)
- [Campo `disabled` não é gravado](form-disabled-nao-grava.md) — outra forma de o POST chegar
  incompleto sem ninguém avisar
