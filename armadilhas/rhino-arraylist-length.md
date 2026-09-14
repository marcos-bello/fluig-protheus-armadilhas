# `ArrayList` do Java não tem `.length` no Rhino

> **Plataforma:** Fluig · eventos de processo / service task · Rhino ES5
> **Status:** confirmado em servidor · reincidiu 4 dias depois em outro projeto

## Sintoma

O e-mail **nunca sai**, e o log mostra exatamente a mensagem de "não tem destinatário" —
mesmo com o e-mail preenchido no formulário.

Em um fluxo isso teve duas caras: o e-mail de aprovação simplesmente não era enviado, e a
service task do e-mail de pendência lançava erro em toda execução, mandando toda
solicitação para "Tratar falha".

```
ERROR JavaScriptException: Não foi possível notificar o candidato:
      e-mail não informado na solicitação NNNNNN.  (<Unknown source>#81)
A solicitação NNNNNN não conseguiu movimentar — movendo para a captura de erro
```

**A linha 81 era o `throw`, dentro do `if (!destinos.length)`.** O `notifier.notify` nunca
chegou a ser chamado — e por isso o log não tem uma palavra sobre e-mail ou template, o que
faz parecer problema de configuração de e-mail quando não é.

## Causa

O `notifier.notify` exige uma **`java.util.ArrayList`** como lista de destinatários. Quem
monta a lista devolve esse ArrayList, e a guarda foi escrita como se fosse array JS:

```js
var destinos = resolverDestinos();   // -> new java.util.ArrayList()
if (!destinos.length) { return; }    // ⛔ SEMPRE verdadeiro
```

No Rhino, `.length` existe para **array JS** e para **array Java** (`String[]`), mas **não**
para as coleções (`ArrayList`, `HashMap`, `List`). O LiveConnect resolve `obj.foo` como o
campo público `foo` ou o getter `getFoo()` — e `ArrayList` não tem nenhum dos dois para
`length`, só o método `size()`.

Resultado: `destinos.length` é `undefined`, `!undefined` é `true`, e a função retorna sempre
pelo caminho de erro.

| Objeto | Tamanho |
| --- | --- |
| array JS `[]` | `.length` |
| array Java `String[]` | `.length` |
| `java.util.ArrayList` / `List` | `.size()` |
| `java.util.HashMap` | `.size()` |
| `java.lang.String` | `.length()` — com parênteses |

## Correção

```js
if (destinos.size() === 0) { return; }   // ou destinos.isEmpty()
```

⚠️ Cuidado para não trocar demais: no mesmo arquivo costuma haver arrays JS de verdade
(`EMAILS_TESTE.length`, `brutos.length`) — nesses o `.length` está **certo**. A regra é
olhar **de onde a variável veio**, não o nome dela.

## Como detectar antes

```bash
grep -n "new java.util" workflow/scripts/*.js forms/**/*.js
# para cada variável que recebe uma coleção Java,
# conferir se em algum ponto ela é lida com .length
```

Dois sinais que denunciam o deslize numa revisão:

- A linha de log **logo abaixo** costuma usar `destinos.size()` corretamente. Quando
  `.length` e `.size()` aparecem sobre a **mesma variável** no mesmo arquivo, a que usa
  `.length` está errada.
- **Clonar processo antigo reimporta bugs já corrigidos em outro lugar.** A reincidência
  aqui veio exatamente disso: código copiado de um fluxo escrito antes da descoberta. Ao
  clonar, rodar a varredura no projeto novo *antes* de publicar.

## Relacionadas

- [Template literal engole a tradução i18n](i18n-template-literal.md) — outro choque de
  duas linguagens na mesma sintaxe
- [Scripts de evento compartilham escopo Rhino](eventos-escopo-rhino.md)
