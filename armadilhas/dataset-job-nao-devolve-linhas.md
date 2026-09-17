# Dataset sincronizado que é job não pode devolver linhas

> **Plataforma:** Fluig · Dataset sincronizado · `onSync`
> **Status:** confirmado · o job funcionou, a persistência é que não

## Sintoma

O `onSync` roda inteiro e faz o que tinha que fazer — abriu os processos, mandou os e-mails.
Mas a sincronização termina com uma parede de erro:

```
Cannot insert duplicate row - Parameters: <filial>, <matricula>, <data>
Cannot insert duplicate row - Parameters: <filial>, <matricula>, <data>
... (101 linhas) ...

0 of delete scripts ran succesfully
0 of update scripts ran succesfully
0 of insert scripts ran succesfully
```

Parece que o dataset falhou. **Não falhou.** O efeito colateral já aconteceu dentro do
`onSync`; o que quebrou foi só a gravação das linhas devolvidas, que acontece **depois** do
`return`.

## Causa

A tentação natural é usar as linhas do dataset como relatório: uma linha por registro
processado, com `setKey(["FILIAL","MATRICULA","DATAINICIO"])`.

O problema é que **essa chave é estável entre execuções**. Se a consulta tem janela móvel — do
tipo "quem inicia nos próximos 30 dias" — o mesmo registro volta com a mesma chave todo dia,
por até 30 dias. Na segunda execução o sync trata como INSERT, não como UPDATE, e o índice
único rejeita.

Duas coisas que descartam as explicações fáceis:

- **Não é dado sujo.** As chaves do lote eram distintas entre si (`sort -u` bate com `wc -l`).
  A colisão é contra o que **já estava gravado**.
- **Não é mudança de estrutura.** O erro traz exatamente os parâmetros da chave **nova**, ou
  seja, a estrutura foi aplicada.

## Correção

**Separar as duas naturezas.** Um dataset sincronizado pode ser uma coisa ou outra:

| Natureza | O produto dele | Devolve linhas? |
| --- | --- | --- |
| Fonte de dados | a tabela sincronizada | sim |
| **Job agendado** | **o efeito colateral** (processo aberto, e-mail, integração) | **não** |

Se é job:

```js
// Este dataset NAO devolve linhas de proposito.
return dataset;   // so as colunas, zero addRow
```

E o relatório vai para o log, num formato de uma linha por pendência, feito para grep:

```js
log.warn("meuJob => PENDENCIA | " + status + " | " + chave + " | " + motivo);
```

```bash
grep "meuJob => PENDENCIA |" server.log
```

Vale reportar só o que precisa de ação (PULADO / ERRO / pendência). O caminho feliz não entra:
a existência do processo já é a prova.

> Se o relatório **precisar** virar tabela, a chave tem que carregar um carimbo da execução
> (`yyyyMMddHHmmss`, não só a data — o job roda mais de uma vez no mesmo dia quando alguém
> sincroniza na mão). Mas aí a tabela cresce sem limite e alguém tem que limpar. Log é mais
> barato.

## Como detectar antes

- Perguntar **qual é o produto do dataset**. Se a resposta for "abrir processo", "mandar
  e-mail" ou "integrar", ele é job: não devolve linha.
- Olhar a chave e perguntar **se ela se repete na execução seguinte**. Consulta com janela
  móvel (`BETWEEN hoje AND hoje+30`) repete quase tudo no dia seguinte — colisão garantida.
- `0 of insert scripts ran succesfully` **junto com** o efeito colateral tendo funcionado é a
  assinatura exata: o job deu certo, a persistência é que não.
- Não confundir com estrutura: se o erro lista os parâmetros da chave **nova**, a estrutura foi
  aplicada e o problema é a repetição, não o `defineStructure`.
