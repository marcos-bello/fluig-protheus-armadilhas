# `notifier.notify` engole e-mail em silêncio

> **Plataforma:** Fluig · e-mail server-side · `notifier`
> **Status:** confirmado · o log mente e diz que enviou

## Sintoma

O log diz **"aviso enviado a 5 destinatário(s)"**. O `notifier.notify` não lançou erro.
Nenhuma mensagem de SMTP, nenhuma de template.

**E o e-mail não chegou para ninguém.**

## Causa

> **`notifier.notify` retorna OK sem lançar erro mesmo quando não entrega.** Ele é
> assíncrono: aceita, enfileira e loga "enviado". A entrega real morre depois, em outra
> thread. Por isso o log do script nunca acusa.

Duas causas reais de descarte silencioso:

1. **Template não cadastrado.** O 2º argumento tem que ser o código de um template
   **registrado** — case-sensitive. Um nome inventado (`"notificacao.generica"`) é aceito
   pelo script e descartado depois.
   ⚠️ Em alguns tenants o identificador **não aceita ponto**.
2. **Remetente inválido, ou igual ao destinatário.** Se remetente == destinatário, o
   `AlertServiceBean` descarta com `Same sender and receiver`. E o login do remetente
   precisa existir *naquele* tenant — ver [nota relacionada](notifier-remetente-inexistente.md).

## O falso culpado que custou um dia

O log estava cheio de:

```
ERROR ...send_email.js - Configurações SMTP do e-mail não estão preenchidas no WCMAdmin
```

Parecia a causa raiz óbvia. **Não era.** Vinha de **outro script** do mesmo ambiente, que
usa SMTP direto/legado. O caminho do `notifier` não depende daquela config e funcionava
normalmente.

> **Um erro gritante no log de outro componente não é prova de causa.** Antes de agir sobre
> uma linha de erro, conferir de qual script ela vem.

## Correção

```javascript
var TEMPLATE_EMAIL  = "MeuTemplateRegistrado";  // existe no painel, case-sensitive
var REMETENTE_EMAIL = "<login que existe neste tenant>";
props.mensagem = ...;   // alias, além de subject/content
```

## Como detectar antes

- **Nunca confiar no "enviou" do log** do `notifier`. Confirmar na caixa de entrada.
- Testar primeiro com **um** destinatário conhecido, fixo no código.
- Conferir o template na tela (Personalização → Templates de e-mails) **antes** de codar.
- Incluir **um destinatário interno de controle** no teste: distingue "não entrega nada"
  de "não entrega para fora".

## Relacionadas

- [Remetente inexistente derruba o e-mail inteiro](notifier-remetente-inexistente.md)
- [CC do notifier recebe o mesmo corpo do e-mail](notifier-cc-vaza-corpo.md)
- [`ArrayList` do Java não tem `.length` no Rhino](rhino-arraylist-length.md) — a outra
  metade do problema: às vezes o `notify` nem chega a ser chamado
