# CC do notifier recebe o mesmo corpo do e-mail

> **Plataforma:** Fluig · e-mail server-side · segurança
> **Status:** confirmado em auditoria de código

## Sintoma

"Vou mandar cópia para o solicitante saber que o e-mail saiu." Parece cortesia.

O solicitante interno passa a receber, na íntegra, **a credencial de acesso de um terceiro**.

## Causa

O notifier padrão do Fluig **não tem CC de verdade**. A implementação comum contorna assim —
e o comentário no próprio código entrega o problema:

```javascript
// CC anexado como destinatário normal (limitação do notifier padrão)
destinatarios.push(emailCC);
```

O "CC" entra na **mesma lista de destinatários, com o mesmo corpo renderizado**. Se o corpo
tem `${token}`, o interno recebe o token.

Agrava quando o endereço de destino vem de **campo livre do formulário**: aí o próprio
solicitante pode digitar o e-mail dele no campo do fornecedor e emitir credencial para si
mesmo — e o validador nunca compara o e-mail gravado com quem está autenticando. Não há
binding entre o token e o endereço para o qual ele foi emitido.

## Por que é grave

- Credencial de terceiro repousando em caixa de funcionário, replayável por todo o TTL.
- **Destrói o não-repúdio.** O log diz "o fornecedor submeteu", mas qualquer detentor do
  token pode ter submetido. Num fluxo que altera dados bancários, esse é o controle inteiro.

## Correção

**Separar as duas mensagens.** São coisas diferentes:

| Para | Conteúdo |
| --- | --- |
| Destinatário externo | link + token |
| Solicitante interno | *"E-mail de acesso enviado para `f***@dominio.com` em 19/08"* — **sem token** |

E mais:

- token só para endereço **validado contra o cadastro** (ERP), nunca campo livre;
- para auditoria, gravar `tokenId + timestamp + destinatário` — **nunca o valor do token**;
- se precisar de CC real, usar SMTP direto com header `Cc` — mas isso resolve o header,
  **não** resolve o corpo ser o mesmo.

## Como detectar antes

Antes de pôr qualquer CC/BCC num disparo automático, a pergunta:

> **"Este corpo contém algo que só o destinatário principal pode ver?"**

Se contém segredo, o CC não é cópia — é vazamento. Dois destinatários diferentes exigem
**dois templates diferentes**.

E conferir de onde vem o endereço: campo livre de formulário nunca deve receber credencial.

## Relacionadas

- [`notifier.notify` engole e-mail em silêncio](notifier-email-silencioso.md)
- [Status BLOCKED nunca volta para ACTIVE](token-status-blocked-permanente.md) — outro
  defeito da mesma família de login por token
